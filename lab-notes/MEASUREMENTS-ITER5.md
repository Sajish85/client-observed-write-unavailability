# MEASUREMENTS — PG Lab Iteration 5: CockroachDB contrast slice (18 Nov 2026)

Internal only. Contrast slice: ONE system (CockroachDB v26.3.6, defaults), process-level faults, single host, loopback. It is not a second full matrix and not a like-for-like benchmark against the PG cells (see CONTRAST-NOTES.md). `--insecure` local lab only.

## 1. Summary (all figures recompute from `results/iter5-config-F*.csv`, script `bin/mkdoc.py`)

| Cell | Fault | N | Outcomes | Client write outage `recover_s`, median [min–max] s | retryable (40001) attempts | failed attempts | acked total | acked-lost (ledger) | runs with loss |
|---|---|---|---|---|---|---|---|---|---|
| F1 | kill -9 leaseholder, probe via surviving gateway | 30 | {'lease_transfer': 30} | **4.27 [3.11–6.08]** | 0 | 30 | 43688 | **0** | 0/30 |
| F2 | kill -9 non-leaseholder (control) | 15 | {'no_client_impact': 15} | none (no client-visible outage; `recover_s` blank). Largest inter-commit gap median [min–max]: 0.04 [0.03–0.07] | 0 | 0 | 22819 | **0** | 0/15 |
| F3 | SIGSTOP leaseholder 20 s, then SIGKILL | 15 | {'lease_transfer': 15} | 6.13 [3.12–6.15] (measured from SIGSTOP) | 0 | 26 | 42207 | **0** | 0/15 |

Headlines (each traceable to the table above and to §3):
- F1: with the leaseholder killed, the client's write path stalled for a median of 4.27 s (range 3.11–6.08 s; 90th percentile 6.06 s, N=30 so no meaningful P99). Zero acknowledged writes were lost in 43688 acknowledged writes (full ledger reconciliation each run).
- F2: killing a non-leaseholder produced no failed or retryable attempt in 15/15 runs; the largest gap between consecutive commits was ≤ 0.07 s (normal 200 tps pacing noise). This is the control behaving as expected.
- F3: the client recovered while the leaseholder was still frozen (3.1–6.2 s after SIGSTOP); the SIGKILL at +20 s caused no further outage (largest commit gap after SIGKILL ≤ 0.07 s in all 15 runs, computed from the raw probe logs).
- The retryable (SQLSTATE 40001) class stayed **empty in every run of every cell** (0 attempts). Single-row autocommit INSERTs did not surface serialization errors under these faults. This is a statement about this workload, not about CockroachDB in general.
- Each F1 run had exactly one `failed` attempt (an in-flight INSERT), and in 29 of 30 runs that "failed" write was present in the final ledger: it had committed but the client was never told (ambiguous result). These are counted as `failed` (never acked), not as acked-lost. See §3.

## 2. Methods

**Setup.** CockroachDB CCL v26.3.6 (darwin arm64, build 2026/11/11, commit 4b93acd33ca641384ed50bfb6fdf168482113e17), downloaded natively to `tools/crdb/`, sha256 verified against the published `.sha256sum`. 3 nodes on 127.0.0.1, `--insecure`, SQL ports 26257/26258/26259, default settings (nothing tuned). Fresh cluster per cell; the cluster persists across runs inside a cell; the killed node is restarted by the harness (same store) after the measured window. Host: macOS 15.5 arm64, 10 cores, 16 GB (as stated in the director brief).

**Defaults as read from the binary** (`logs/defaults.txt`): `kv.leases.leader_leases.enabled = true`; `kv.lease.expiration_leases_only.enabled = false`; `kv.store_liveness.enabled = true`; `kv.raft.leader_fortification.fraction_enabled = 1`; `server.time_after_store_suspect_in_store_liveness = 30s`; `server.time_until_store_dead = 5m`. Replication factor: default zone `num_replicas = 3`, and `SHOW RANGES FROM TABLE ... WITH DETAILS` on a new table returned `replicas {1,2,3}` (verified, not assumed). So the lease type in force is leader leases, inferred from those setting values.

**Probe** (`bin/probe5.py`, adapted from the PG lab `probe.py`). One writer, autocommit single-row `INSERT INTO t(seq, client_ts)`, unique `seq`, target 200 tps, one CSV row per *attempt*: `seq,attempt,t_start,t_end,outcome,err`. Outcome classes, never merged: `committed` (server acked), `failed` (any non-40001 error, including connection loss, `statement timeout`, ambiguous-result errors, or a 6 s client-side hard deadline), `retryable` (SQLSTATE 40001; same seq retried up to 20 attempts with 10–100 ms backoff; a seq that then commits is `committed`, not lost). Session `statement_timeout` 3000 ms (same as PG probe), plus a 6 s hard per-operation client deadline added because, in a discarded smoke attempt, a stalled request outlived `statement_timeout`.

**Fixed target rule.** The probe connects to exactly ONE gateway node per run via a single-host DSN (no multi-host failover string). The gateway is always a surviving node: the lowest-numbered node that is not the run's victim, chosen before the fault and recorded as `gateway_node`. The strategy is fixed for the iteration; the identity differs per run (mostly n1; n2 when n1 is the victim).

**Leaseholder identification** (per run, before injecting): `SELECT lease_holder, replicas, voting_replicas FROM [SHOW RANGES FROM TABLE lab.<probe_table> WITH DETAILS]` via node1; `lease_holder` is a CRDB node ID, mapped to node1/2/3 through the `sql_address` column of `cockroach node status` (IDs follow join order and do not match directory order: id2 = port 26259). Checked once after a 4.5 s warm-up; if it changed the probe restarted. New leaseholder is read the same way via the gateway after the recovery window, with the victim still dead.

**Faults.** F1: `kill -9` the leaseholder's `cockroach` PID. F2: `kill -9` the lowest-numbered node that is not the leaseholder (gateway = lowest-numbered node other than the victim, so it was the leaseholder itself in 12/15 runs). F3: SIGSTOP the leaseholder PID, wait 20 s, SIGKILL. `kill_epoch` = the instant before the first signal.

**Metrics.** Over committed attempts sorted by `t_end`, the largest gap that ends after `kill_epoch` is `max_gap_s`; `write_stop_s` = start of that gap − kill; `recover_s` = end of that gap − kill, reported only when the run had a non-committed attempt or a gap ≥ 1 s (otherwise blank and outcome `no_client_impact`). `detect_s` = `t_end` of the first non-committed attempt after the kill − kill. `acked_lost` = acked seqs absent from `SELECT seq` on the gateway after the cluster settled (full reconciliation, every run). `settle_s` = kill → health gate passing again, which includes the fixed measurement window (≥17 s) and is therefore **not** a recovery metric; ~19.8 s for F1/F2 and ~39.8 s for F3 reflects the harness timing plus about 1–2 s for restart and gate. Health gate: 3 PIDs alive, `node status` all live/available, every range ≥ 3 replicas, UPSERT works through each SQL port.

## 3. Per-cell results

### F1 — kill -9 leaseholder (N=30)
- `recover_s` median [min–max]: 4.27 [3.11–6.08] s; mean 4.57; 90th percentile 6.06. Sorted: 3.11, 3.64, 3.76, 3.77, 3.79, 3.80, 3.80, 3.81, 3.81, 3.85, 3.86, 4.01, 4.09, 4.14, 4.15, 4.39, 4.52, 4.59, 4.62, 4.79, 4.80, 4.84, 5.60, 5.61, 5.75, 5.94, 6.06, 6.07, 6.07, 6.08.
- `detect_s` median [min–max]: 4.20 [3.01–6.01] s. `max_gap_s`: 4.28 [3.13–6.08] s.
- Outcomes: 30× `lease_transfer` (the lease was on a different node afterwards in every run; 0 runs where it stayed).
- Retries: 0 retryable attempts. Failed attempts: 30 (1 per run): 25 `result is ambiguous` errors, 1 ambiguous error naming the gRPC connection to the dead node, 4 `client_hard_timeout 6s` (my 6 s deadline fired).
- Acked-lost: 0 of 43688 acked. Present-but-unacked: 29 writes among the 30 failed attempts were found in the ledger (committed, client not told).
- The values cluster in three bands (3.1–3.9 s, 4.0–4.8 s, 5.6–6.1 s). I have no explanation for the bands; unverified.
- Run 22 was replaced: the original run-22 row was invalid (harness bug, see §6) and is preserved in `results/iter5-excluded-F1-r22.csv`; the replacement was run on a fresh cluster afterwards, so it is not from the same cluster instance as runs 1–21 and 23–30.

### F2 — kill -9 non-leaseholder control (N=15)
- 15× `no_client_impact`: 0 failed, 0 retryable, 0 acked-lost, 22819 acked. Largest inter-commit gap 0.04 [0.03–0.07] s.
- Lease behaviour: leaseholder unchanged in 13 runs, changed in 1 (run 1: n1→n3 with a non-leaseholder dead, so not caused by the kill; cause not investigated), unknown in 1 (run 13: new-leaseholder query hit its 5 s statement timeout; left blank).

### F3 — SIGSTOP leaseholder 20 s, then SIGKILL (N=15, run because time remained)
- `recover_s` (from SIGSTOP) median [min–max]: 6.13 [3.12–6.15] s; two bands: 4 runs at 3.12–3.39 s, 11 runs at 6.11–6.15 s.
- `detect_s` is 2.99–3.01 s in all 15 runs: the first non-committed attempt is the probe's own 3000 ms `statement_timeout`, so `detect_s` here reflects my probe setting, not CockroachDB.
- 26 failed attempts (all `statement timeout`), 14 of which were present in the ledger; 0 retryable; 0 acked-lost in 42207 acked.
- Labelled separately from F1; the 20 s freeze does not extend client outage in this setup (inferred: leases moved while the process was stopped).

## 4. Contrast table (PG numbers copied from `~/postgres-failover-lab/STUDY.md` §2, read on this machine)

| System / cell | Config | Fault | N | Median client `recover_s` [min–max] s | acked-lost |
|---|---|---|---|---|---|
| PG A-fo | async, PST=0, 3 nodes | kill_pm | 10 | 5.93 [5.44–7.86] | 0 |
| PG B-fo | sync, PST=0, 3 nodes | kill_pm | 10 | 7.66 [7.12–7.93] | 0 |
| PG C1 | async, Patroni defaults, 3 nodes | kill_pm_agent | 30 | 22.38 [20.67–27.64] | 2 in 2/30 runs |
| PG C3 | async, PST0 ttl10/loop2/retry3, 3 nodes | kill_pm | 30 | 1.89 [0.19–2.13] | 0 |
| PG A (restart-in-place, NOT a failover) | async, Patroni defaults (PST 300) | kill_pm | 10 | 5.96 [4.92–8.28] | 0 |
| PG B (restart-in-place, NOT a failover) | sync, Patroni defaults (PST 300) | kill_pm | 10 | 5.96 [5.79–8.05] | 0 |
| **CRDB F1** | v26.3.6 defaults, 3 nodes | kill -9 leaseholder node process | 30 | **4.27 [3.11–6.08]** | **0 of 43688** |
| **CRDB F2** (control) | v26.3.6 defaults, 3 nodes | kill -9 non-leaseholder node process | 15 | no client outage (no failed/retryable attempt) | 0 of 22819 |
| CRDB F3 (separate) | v26.3.6 defaults, 3 nodes | SIGSTOP leaseholder 20 s, SIGKILL | 15 | 6.13 [3.12–6.15] | 0 of 42207 |

Footnotes (caveats, not optional):
1. Different failure units: PG `kill_pm` kills the postmaster only (the Patroni agent survives); `kill_pm_agent` kills postmaster and agent; CRDB kills the whole node process (SQL, KV, Raft in one process). The faults are not equivalent.
2. PG numbers are also single-host loopback; both systems share that limitation.
3. The PG probe used a multi-host libpq string with `target_session_attrs=read-write` (client follows the primary); the CRDB probe used one fixed surviving gateway.
4. The PG pilot cells ran pgbench alongside the probe; CRDB is probe-only.
5. PG's `recover_s` waits for a new primary to accept writes; CRDB's measures the first commit after the stall through a surviving gateway. Both are client-observed write outages, defined in each lab's own harness.
6. PG C1's loss (acked-lost 2) is a PG asynchronous-replication property; CRDB commits require a Raft quorum, and 0 losses here is a result for this fault only, N=30.

## 5. External reference (published; not ours)
As reported in a research-report extraction of Kettaneh et al., "Scalable Leader Leases for Multi Consensus Groups in CockroachDB," SIGMOD Companion 2026 (the PDF itself was not read; figures are as reported secondhand, unverified against the paper): on a 6-node GCP cluster, each experiment repeated 300 times, client-observed range unavailability had a median of 3.0–3.9 s with expiration leases and 4.0–4.7 s with leader leases (P99 about 6.5 s), under node crash, full/partial partition, and disk stall, with lease expiry the dominant phase. Our F1 (kill -9 of the leaseholder node, leader leases on by default, v26.3.6) gave a median of 4.27 s with range 3.11–6.08 s (N=30), which sits inside the reported leader-lease median band, and our maximum (6.08 s) is below the reported ~6.5 s P99. That agreement is a sanity check, not a replication: theirs is cloud VMs, a real network, 6 nodes, 300 repetitions and several fault types; ours is one host, loopback, 3 nodes, N=30, one fault type, a single-row-insert probe through a fixed gateway, and a different software version (their version is not stated in the report extraction I read). Our outage metric is the client's inter-commit gap and may not equal their range-unavailability definition. With N=30 I cannot estimate a P99.

## 6. Limitations and honest notes
- Single host, loopback (no network latency or partition), process-level faults only (no disk stall, partition, or whole-machine fault), 3 nodes, one range of interest (a new table lands in the shared tail range), one table per run, probe-only load at 200 tps single writer. Results may not transfer to real networks or heavier load.
- `--insecure`, local only.
- N=30/15/15: medians are stable, tails are not estimated.
- Gateway was the leaseholder in 12/15 F2 runs and in none of the F1/F3 runs by construction; a different gateway rule would change what the client sees.
- `statement_timeout` (3 s) and the 6 s hard deadline shape the `failed` counts and `detect_s`.
- Excluded run: F1 run 22 (original). Cause: the leaseholder moved during warm-up, my harness restarted the probe, and the new probe restarted `seq` at 1 against the same table, producing 227 duplicate-key failures and no real measurement. Fixed (table cleared on warm-up restart), row moved to `results/iter5-excluded-F1-r22.csv`, replacement run done. In F2/F3 and the other F1 rows, no warm-up restart occurred (checked in the notes column).
- A first smoke attempt was discarded: the harness killed a node about 30 s after init while system ranges still had 2 replicas, so quorum was lost and the cluster stalled until I restarted it. The gate now requires every range ≥ 3 replicas. Smoke rows for all three cells are in `results/iter5-smoke-F*.csv` and are not counted.
- F2 `write_stop_s` was blanked post hoc for no-impact rows (it held the start of a sub-0.1 s sampling gap). Newlines were stripped from `notes` cells.
- Disk: free space ~13 GiB during runs (18 GiB at start); the 3 stores took about 3.1 GB. No pruning was needed.
- Not run / not done by design: VM mirroring, Space update, debate session (director brief assigns these elsewhere).

PATENT-FLAGS: none identified

## Completion note
- **Did:** native install of v26.3.6 (checksum verified); 3-node loopback cluster at defaults; adapted probe and harness; F1 N=30, F2 N=15, F3 N=15, every run with full ledger reconciliation; CSVs, CONTRAST-NOTES.md, LAB-ITER5.md, STEPLOG.md.
- **Found:** F1 outage median 4.27 [3.11–6.08] s, 0 acked-lost; F2 no client impact; F3 recovered during the freeze; 0 retryable errors anywhere; ambiguous in-flight writes that committed but were not acked (29 in F1).
- **Blocked / deviations:** F1 run 22 replaced after a harness bug (see §6); one F2 `new_leaseholder_node` blank (query timeout); external-paper figures are secondhand; "latest stable" resolved via the docs release table plus CDN probing (v26.4.x exists on the CDN but is not in the release table, treated as pre-release, unverified). Acceptance counts: F1 CSV 30 data rows, F2 15, F3 15.

## Anti-publication debate
**Debate (adversarial review session, 18 Nov 2026; full text `DEBATE-ITER5-AGAINST.md`):** verdict — include the contrast only with specified downgrades. Central attacks: (1) CRDB `recover_s` is the lifetime of one blocked in-flight request, not a sampled service outage — `recover_s ≈ detect_s + 0.07 s` in all 30 F1 runs (one failed attempt per run, the next attempt commits ~0.06 s later); availability during the stall was never sampled, so 4.27 s is an upper bound whose length is set by when the in-flight RPC errors out. (2) The top of the F1 distribution is censored by the probe's own 6 s hard deadline (runs 14, 15, 16, 18); the maximum and 90th percentile are instrument ceiling, so "max below the SIGMOD P99" is invalid. (3) F3 is timer-quantised: its two bands are 1× vs 2× the probe's 3 s `statement_timeout`. (4) The 40001/retry class is empty in all 60 runs — the CRDB-specific part of "the method travels" is untested. (5) The gateway is chosen knowing the victim (conditional on gateway survival) and never reconnects, unlike the PG probe's multi-host failover — different client experiences in one table column. (6) The dominant failure mode is ambiguity: 29/30 F1 "failed" writes had committed; "0 acked-lost" selects the class that cannot fail under quorum acks. (7) Replacement run 22 came from a different cluster instance (median without it: 4.391 s). (8) F2's gateway was the leaseholder in 12/15 runs. (9) "Sits inside the SIGMOD band" overclaims: our median's bootstrap 95% CI (3.86–4.80) extends outside the reported 4.0–4.7 band, the metric differs, and the figures are secondhand (PDF not read). (10) "Latest stable" is an unverified inference; the leader-lease label is inferred from settings.

**Adjudication (Muse, director):** I recomputed the debate's checkable claims from the raw CSVs before adjudicating: recover−detect mean difference 0.071 s (max 0.104); F1 median without run 22 = 4.391 s; by-gateway medians n1 4.391 (n=27) / n2 4.014 (n=3); deadline-censored runs are exactly 14, 15, 16, 18; bootstrap 95% CI of the F1 median 3.86–4.79 s — all match the debate exactly. ACCEPTED in full: the verdict (include only with downgrades) and all nine required changes, including the debate's exact paper wording (metric renamed to the stall of the in-flight write through a pre-selected surviving gateway; F3 reduced to one timer-quantised sentence and removed from any table; F1 max/P90 marked censored and the max-vs-P99 sentence deleted; SIGMOD phrasing downgraded to "consistent with a secondhand report, not a replication", and the comparison dropped entirely unless the PDF is read; CRDB reported in a separate block from PG `recover_s` with an `unacked_but_committed` column for both systems; version given as v26.3.6 + build only, lease type "inferred from settings"; replacement-run and F2-composition footnotes). One piece of context, not a rebuttal: the PG probe is also a single closed-loop writer, but PG attempts fail fast (hundreds of failed attempts per agent-death run) while CRDB's gateway blocks the one in-flight request — so the two `recover_s` values are produced by different client failure behaviour (fail-fast retry vs block-and-wait), which is itself worth one sentence in the paper and is a further reason the numbers are not one metric. As with iteration 4, the measurements document had already disclosed nearly every limitation; the failure is headline labelling, not honesty. DEBATED VERDICT for iteration 5: the contrast survives as a short portability observation in the debate's wording — the probe + ledger discipline ports to a second PG-wire system, a leaseholder kill stalls the in-flight write a median 4.27 s with zero acknowledged loss and near-universal ambiguous commits, a follower kill is client-invisible — and NOT as a co-equal PG-vs-CRDB comparison table. What would upgrade it: an open-loop observer sampling availability during the stall, and one contended cell exercising 40001 with the retry path working. Lab stays private; nothing published.

ITERATION 5 READY
