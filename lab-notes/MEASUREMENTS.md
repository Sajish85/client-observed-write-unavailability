# MEASUREMENTS.md — failover pilot, 17 Nov 2026

## Summary
- **Under Patroni defaults, `kill -9` of the primary's postmaster does NOT cause a failover.** The Patroni agent on that node survives, restarts Postgres in place, and keeps the leader key. All 20 runs of sets A and B were restart-in-place (timeline unchanged). The brief's "time to promotion" does not exist for this fault under defaults, so I added two sets with `primary_start_timeout=0`, which forces a real promotion (A-fo, B-fo; 20 promotions, new timeline each time).
- Client-visible outage is **~6 s (median) in A, A-fo, and B; ~7.7 s in B-fo**. Ranges: 4.9–8.3 s.
- **Acknowledged-but-lost transactions: 0 across all 40 runs** (~9,000 acked txns per run), in both async and sync configs. This pilot cannot separate A from B on data loss.
- **Where the time goes:** the client sees failure within ~10 ms; Postgres is dead within ~0.1 s; then ~5–7.5 s passes before Patroni acts. The promotion itself is ≈0.1 s (verified in logs for one B-fo run).

## Data (n=10 per set; seconds after kill unless noted; median [min–max])
Raw: `results/config-{A,B,A-fo,B-fo}.csv`.

| Metric | A (async, default) | B (sync, default) | A-fo (async, PST=0) | B-fo (sync, PST=0) |
|---|---|---|---|---|
| Outcome | 10× restart in place | 10× restart in place | 10× promotion | 10× promotion |
| write_stop_s | 0.000 [−0.003–0.003] | −0.001 [−0.002–0.002] | 0.001 [−0.002–0.003] | 0.002 [−0.003–0.003] |
| detect_s (client sees failure) | 0.012 [0.007–0.015] | 0.013 [0.007–0.021] | 0.011 [0.009–0.020] | 0.013 [0.008–0.017] |
| pg_down_s (monitor) | 0.078 [0.038–0.106] | 0.057 [0.013–0.103] | 0.057 [0.004–0.101] | 0.085 [0.008–0.103] |
| leader_change_s (Patroni) | n/a | n/a | 6.03 [5.46–7.83] | 7.77 [7.11–8.04] |
| primary_ready_s | 6.07 [5.07–8.60] | 5.51 [5.38–7.48] | 5.96 [5.44–7.85] | 7.66 [7.16–7.99] |
| **recover_s (client)** | **5.96 [4.92–8.28]** | **5.96 [5.79–8.05]** | **5.93 [5.44–7.86]** | **7.66 [7.12–7.93]** |
| acked_lost | 0 in 10/10 | 0 in 10/10 | 0 in 10/10 | 0 in 10/10 |
| unacked_present | 0 | 1 in 4 of 10 runs | 0 | 0 |
| error rows (probe) | 85 [70–118] | 77 [74–105] | 83 [77–110] | 108 [99–112] |
| timeout rows | 0 | 0 | 0 | 0 |
| settle_s (cluster fully healthy again) | 21.2 [20.1–24.3] | 21.2 [21.2–23.2] | 21.2 [21.2–23.3] | 23.3 [22.2–23.3] |

Notes: `primary_ready_s` is monitor-based (±0.1 s poll) and can differ from `recover_s` by a few hundred ms in either direction (A: median 6.07 vs 5.96). `recover_s` is the data of record. `unacked_present` = a commit was in flight at kill, applied, but the client got an error (the classic ambiguous-commit case); not data loss.

## Where recovery time goes
Phase breakdown, median, async-failover set (A-fo): kill → client error 0.01 s → Postgres port dead 0.06 s → **Patroni decision ≈ 5.9 s** → new primary accepting 5.96 s → client first commit 5.93 s.
Verified detail for one B-fo run (run 9, from Patroni logs, node pg3 killed at 18:13:35.95, pg1 promoted):
| t after kill | event (source: cluster/logs/pg*.log) |
|---|---|
| +7.6 s | killed node's Patroni: "Primary crashed. Failing over." |
| +7.7 s | leader key released; pg1 "promoted self to leader by acquiring session lock" |
| +7.7 s | pg1 "database system is ready to accept connections" |
| +8.8 s | pg1 assigns pg2 as synchronous standby |

So ~98% of the outage is Patroni's detection/decision latency on the killed node, not election, promotion, or client reconnect. The log shows the agent retrying its connection to the dead Postgres for several seconds before declaring the crash. **Why it takes 5–8 s rather than a fixed interval, and why B-fo is ~1.7 s slower than A-fo, is not established by this pilot** (hypotheses, unverified: HA-loop phase vs `retry_timeout`; extra sync-mode handling). For sets A and B (restart in place) I did not read the Patroni logs; the sub-split of the ~6 s between "decide to restart" and "crash recovery + startup" is unverified (monitor shows the node in `starting` state only in the last ~0.6 s).

## A vs B contrast
- Recovery time: with defaults, identical (5.96 s medians). With forced failover, sync costs about +1.7 s median (7.66 vs 5.93; ranges 7.12–7.93 vs 5.44–7.86 overlap only at the top; n=10, one host, no significance test run).
- Data loss: none in either. This is expected for this fault and should not be read as "async is safe": `kill -9` of a process leaves its WAL in the OS page cache, the walsenders had already shipped everything (lag 0 before each kill, enforced by the health gate, and loopback is instant). Async loss needs replication lag at the moment of a *host/network* failure, which this fault does not create. So the pilot shows the cost side of sync (some extra recovery time) without the benefit side.

## Verdict: is there a paper here?
**Weak as a measurement paper; one modest, real, practitioner-relevant finding.**
- Non-obvious and verified: process-crash of the primary under Patroni defaults yields restart-in-place, not failover, and `primary_start_timeout` (default 300 s) governs it; the outage is dominated by agent-side detection latency (~6–8 s) rather than promotion (~0.1 s). Many operators assume "kill -9 the primary" tests failover; it tests crash restart. Whether this is already documented: Patroni's docs describe `primary_start_timeout`, so the *behaviour* is documented; the *measured* breakdown from the client's point of view is the contribution, and I haven't done a literature search (unverified novelty).
- Candidate claim for a fuller study: "Client-observed failover time under Patroni is set by the failed node's agent's detection path, not by consensus or promotion, and depends on fault type; the sync-vs-async trade-off only shows up under faults that create replication lag (host/network loss), not process crashes." Needs those other faults to be a claim at all.
- A full study needs:
  1. Fault types: kill -9 Patroni too, SIGSTOP (freeze) of PG and/or Patroni, network partition (macOS `pf`, or better real VMs), host power-off, disk stall. The ttl-expiry path (~30 s by default) is untested here.
  2. Real replication lag: heavier write load, WAN-like latency (netem on Linux), so async loss becomes non-zero and sync's cost is measurable.
  3. Parameter sweep: `loop_wait`, `ttl`, `retry_timeout`, `primary_start_timeout`, `synchronous_mode_strict`, and a second DCS/PG version.
  4. N ≥ 30 per cell (spread is 3 s wide at n=10), randomised order, confidence intervals.
  5. **A second machine** (or at least separate VMs with NTP/PTP-synced clocks) so the client and cluster don't share a host. The one-host limitation, plainly: probe, harness, Patroni, etcd and three Postgres instances share one CPU set, one kernel, and one clock; loopback has no network delay or failure modes; the client-vs-cluster clock-skew question is not even posed here. Results are valid as relative, mechanism-level evidence only.
  6. Patroni-side instrumentation (log timestamps per phase) merged with client logs.

## Limitations of this pilot (honest list)
- One host, loopback, Rosetta-free arm64 but a single OS; pg_rewind/trust auth are lab settings.
- Kill target is the postmaster only; backends were not signalled separately.
- Probe rate 200 tps with 50 ms retry back-off; pgbench load stops at the kill.
- Sets ran sequentially (A, B, A-fo, B-fo), not randomised; time-of-day/thermal drift not controlled.
- Failover sets deviate from "defaults" by `primary_start_timeout=0` — a deliberate, labelled change.
- No statistical tests were run.

## Completion note
- **Did:** native stack built (PG 17.12, etcd 3.7.3, Patroni 4.1.5; no sudo, no Docker); probe + harness written; 4 sets × 10 runs executed; CSVs and per-run raw logs saved; LAB.md and this file written.
- **Found:** no failover under defaults for a postmaster kill; ~6 s restart-in-place; ~6–7.7 s forced failover; 0 lost acked txns in 40 runs; recovery time dominated by Patroni detection.
- **Blocked:** nothing. Open: Patroni logs for sets A, B, A-fo were overwritten by later sets (only B-fo logs remain), so the phase breakdown is log-verified for B-fo only.
- Nothing was published, pushed, or posted. Open-source software only; no employer material.

PILOT LAB READY
