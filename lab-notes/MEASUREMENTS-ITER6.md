# MEASUREMENTS — Iteration 6: Redis Sentinel slice (reference depth)

Internal only. Single host, loopback, shipped defaults. All numbers below recompute from `results/iter6-config-R1.csv` and `iter6-config-R2.csv` (`bin/stats.py`).

## 1. Summary

| Cell | Fault | N | Outcomes | Client write outage `recover_s`, median [min–max] | Failed attempts (total) | Acked total | Acked-lost | Runs with loss |
|---|---|---|---|---|---|---|---|---|
| R1 | `kill -9` primary | 30 | 30× promotion | **30.92 s [30.42–31.45]** | 185,360 (median 6,181.5/run) | 116,541 | **0** | 0/30 |
| R2 | `kill -9` replica (control) | 15 | 15× no_client_impact | none (no failed attempt); largest inter-commit gap median 0.01 s [0.01–0.03] | 0 | 75,775 | **0** | 0/15 |

- R1 outage is the shipped Sentinel `down-after-milliseconds` = 30000 plus about 0.4–1.5 s of election and promotion. `detect_s` (first failed attempt after kill) is median 0.003 s, because a killed local process refuses connections immediately; the client sees failures at once but cannot write again until Sentinel promotes.
- Acked-lost was 0 of 116,541 acked writes in R1, by full ledger reconciliation (EXISTS on every acked ID against the promoted primary after settle). Redis replication is asynchronous, so loss is possible by design; it was not observed here. Why (inferred, not tested): on loopback, replication lag is far smaller than the 200 writes/s spacing, and the only exposed window is writes acked in the last milliseconds before the kill. This says nothing about loss under network delay, load, or a partition rather than a process kill.
- R1 victim was always the primary; new primary was always a different node (30/30). Primary alternated between ports 6379/6380/6381.
- `settle_s` (median 51.61 s R1, 21.31 s R2) includes the fixed measurement window and restart; it is NOT a recovery metric.

## 2. Methods

**Software.** Redis 8.10.2, built from source, user space, no sudo/Docker/Homebrew. "Latest stable" resolved as: `download.redis.io/redis-stable.tar.gz` contains `REDIS_VERSION "8.10.2"` (version.h). The releases directory also lists 8.10.3, which I did not use because it is not the `redis-stable` pointer (judgement call; unverified whether 8.10.3 is formally "stable"). Downloaded `redis-8.10.2.tar.gz`; sha256 `b9ffee22…c02b3` matches the value published in the redis-hashes repo (verified). Top-level `make` exits non-zero because the bundled modules (redisbloom, redisearch, redisjson, redistimeseries) do not build on this host; the core was built with `make -C src -j8` (rc 0) and modules are not used. `redis-server --version`: `v=8.10.2`. Client: redis-py 7.0.1 (venv, Python 3.9.6). Host: macOS arm64, loopback.

**Topology.** 1 primary (6379) + 2 replicas (6380, 6381, `--replicaof`), 3 Sentinels (26379/26380/26381), monitoring `mymaster` quorum 2. Only the working dir differs per node. Fresh topology per cell; within a cell it persists across runs (each run a fresh key prefix). After a kill the harness restarts the victim (for R1 it comes back with no `--replicaof` and Sentinel demotes it) and waits for the health gate.

**Defaults, read from the running system** (`logs/defaults.txt`): `down-after-milliseconds` 30000; `failover-timeout` 180000; `parallel-syncs` 1; quorum 2; `save 3600 1 300 100 60 10000`; `appendonly no`; `min-replicas-to-write 0`. Nothing tuned.

**Probe.** One writer (thread), redis-py `Sentinel` discovery with `master_for('mymaster')`, so it re-discovers the primary through Sentinel (no hard-coded target). `socket_timeout=1.0 s` (instrument setting; also connect timeout is default). Target 200 SET/s, small value `v`, unique IDs `R<cell>-r<run>-<seq>`. Every attempt logged (`id,t_start,t_end,outcome`) in `runs/iter6-<cell>-rNN/probe.csv`. A failed ID is never retried; the writer moves on. redis-py client-internal retry settings were left at library defaults (not inspected, so unverified how they interact with failed-attempt timing). Warm-up 5 s before kill; after kill the writer runs for at least 45 s (R1) / 20 s (R2) and until 10 s after first recovery, capped at 150 s.

**Metrics.** `kill_epoch` = instant before SIGKILL. `detect_s` = end of first failed attempt after kill − kill. `recover_s` = end of first committed attempt that started after that failed attempt − kill. `write_stop_s` = end of last commit before the first failure − kill. `max_gap_s` = largest gap between consecutive commit ends that ends after kill. `acked_lost` = acked IDs absent from the promoted primary after settle, by full ledger reconciliation.

**Health gate** (before every fault and as "settled"): all 6 processes alive; all 3 Sentinels agree on one primary with `num-slaves` 2, `num-other-sentinels` 2, no s_down/o_down/disconnected on any replica; primary `connected_slaves` 2; a SET/GET through Sentinel discovery succeeds; both replicas' offsets ≥ the primary offset sampled just before (strict equality is impossible because Sentinel hello PUBLISHes keep advancing the primary offset).

## 3. Per-cell results

**R1 (N=30).** All 30 promotions. `recover_s` 30.92 [30.42–31.45]; `max_gap_s` equals it within ~5 ms; `detect_s` 0.003 [0.00–0.01]; `write_stop_s` −0.004 [−0.005–0.001] (the outage begins at the kill). Failed attempts per run median 6,181.5 [6,082–6,287]. Acked per run median 3,884.5. Acked-lost 0/116,541. `settle_s` 51.61 [46.05–52.64].

**R2 (N=15).** Victim always port 6380 (lowest-port non-primary; primary stayed 6379). 15/15 `no_client_impact`: zero failed attempts, 75,775 acked, 0 lost, largest commit gap ≤ 0.03 s. `settle_s` median 21.31 [21.20–25.41].

**Excluded:** N=1 smoke runs per cell (outputs in `results/iter6-smoke-*.csv`, not counted). No `no_recovery` runs, no rebuilds, no pruning (disk 12 GiB free at end).

## 4. Cross-system placement

Redis R1 sits at the slow end: ~31 s client write outage, essentially the shipped 30 s Sentinel down-after plus sub-2 s promotion. That is comparable to Patroni C1 (22.38 s, bound by leader-key TTL after the agent died) and far slower than CockroachDB F1 (4.27 s) and tuned Patroni C3 (1.89 s). Unlike PG C1 (2 acked-lost) Redis showed 0 acked-lost here, despite asynchronous replication, but the sample is process kills on loopback and loss is a legitimate possible outcome. The R2 control produced no client impact, like CRDB F2.

| System / cell | Config | Fault | N | Median client recover_s [min–max] | Acked-lost |
|---|---|---|---|---|---|
| PG C1 | async, Patroni defaults, 3 nodes | kill_pm_agent | 30 | 22.38 [20.67–27.64] | 2 in 2/30 runs |
| PG C3 | async, PST0 ttl10/loop2/retry3, 3 nodes | kill_pm | 30 | 1.89 [0.19–2.13] | 0 |
| CRDB F1 | v26.3.6 defaults, 3 nodes | kill -9 leaseholder | 30 | 4.27 [3.11–6.08] | 0 of 43688 |
| CRDB F2 (control) | v26.3.6 defaults | kill -9 non-leaseholder | 15 | no client impact | 0 of 22819 |
| **Redis R1** | 8.10.2 Sentinel defaults, 1+2+3 | kill -9 primary | 30 | **30.92 [30.42–31.45]** | **0 of 116,541** |
| **Redis R2** (control) | 8.10.2 Sentinel defaults | kill -9 replica | 15 | no client impact | 0 of 75,775 |

PG numbers copied from `~/postgres-failover-lab/STUDY.md` §2 table; CRDB from `~/cockroach-lab/MEASUREMENTS-ITER5.md` §4 (both read on this machine).

Footnotes: (1) Redis is a key-value store; the probe is `SET`, not `INSERT`. (2) Replication is asynchronous by design, so acked-lost > 0 would be expected and legitimate; none was observed. (3) R1's outage is dominated by Sentinel's shipped 30000 ms down-after, which does not map directly onto Patroni's agent-decision latency or CRDB's lease expiry. (4) Failure units and client paths differ (Sentinel discovery vs multi-host libpq vs fixed gateway). (5) Everything is single-host loopback. (6) Acked totals differ across systems because failed attempts are not acked and run lengths differ.

## 5. Limitations

Single host, loopback only, process-level faults only (`kill -9`; no partition, delay, load, disk fault); one writer; small values; defaults only; N=30/15 so no P99; socket_timeout 1.0 s shapes failed-attempt timing; writer paced at 200/s so acked-lost measures only that exposure; victim restart arguments are a harness choice; 8.10.3 exists and was not used; modules not built. Redis-py retry defaults not inspected.

## PATENT-FLAGS

PATENT-FLAGS: none identified

## Anti-publication debate

**Debate (adversarial review session, 18 Nov 2026; full text `DEBATE-ITER6-AGAINST.md`):** verdict — include only with specified downgrades, NOT as a row in the PG/CRDB master table. Central attacks: (1) R1's outage is a configuration constant: recover_s − 30 s = median 0.92 s [0.42–1.45] in 30/30 runs — N=30 measured the shipped down-after timer plus a residue that is never decomposed into phases (per-run Sentinel logs were not kept; data/ is wiped per cell). (2) STRONGEST: "0 acked-lost of 116,541" overstates the exposure by orders of magnitude — 74% of acked writes were acked after recovery and could not be lost; the at-risk set is the 30,035 writes acked before the kill (~1,001/run), and only 28 writes across all 30 runs were acked within 5 ms of the kill (300 within 50 ms). A loopback process kill with ~0 replication lag and 5 ms write spacing could not be expected to produce loss; Redis has no lag-provoking cell comparable to PG C4. The doc's "despite asynchronous replication" / "Unlike PG C1 (2 acked-lost)" sentence invites a contrast the data cannot support. (3) One acked-lost column across three ack meanings is not one quantity: a Redis ack at the shipped defaults (appendonly no, min-replicas-to-write 0) is "applied in one node's memory" — the weakest ack in the table, vs PG's WAL commit and CRDB's Raft quorum. (4) Client behaviour partly uninspected in the doc; the debate read the installed redis-py 7.0.1 source: the client re-resolves the master through Sentinel on every attempt (no stale-connection caching), which supports recover_s being Sentinel-side, and the first post-recovery commit takes ~9–11 ms, bounding the client share of the residue at ~10 ms; but failed attempts return in ~0.8 ms despite redis-py's default 3-retry exponential backoff (base 1 s) — unexplained, unverified — and the failed-attempt count (185,360 ≈ 99.97% of outage × 200/s) is just writer ticks, not independent evidence. (5) R2's victim was always port 6380 with primary always 6379 — the control tests only "no spurious failover when a replica dies at min-replicas-to-write 0". (6) Label: 8.10.3 exists upstream and was passed over on an unverified judgement call; core-only build (modules failed) — low risk for a Sentinel timer, but the label must say so. The debate also found a factual error in MEASUREMENTS §2: the restarted R1 victim is brought back WITH `--replicaof <new primary>` (harness.py line 114), not with no `--replicaof` as the doc states. (7) Load realism: 200 SET/s is a probe rate, not a Redis workload; lag and loss are load-dependent. (8) The tight distribution (SD 0.32 s) is timer determinism, not statistical strength — the same timer weakness as PG C1's TTL-bound 22.38 s, except PG's paper contribution is the mechanism/loss analysis around its timer, which has no Redis counterpart here.

**Adjudication (Muse, director):** I recomputed the debate's checkable claims from the raw CSVs and per-run probe logs on the Mac before adjudicating: R1 median 30.9165 [30.417–31.445], excess over 30 s median 0.9165 [0.417–1.445]; pre-kill acked total exactly 30,035; acked within 5 ms of the kill exactly 28 (within 50 ms: 300); failed-attempt durations in runs 1/12/30 median 0.75–0.81 ms, max 19–28 ms; harness.py line 114 restarts the victim with `--replicaof` set to the current primary — all match the debate exactly, including the Methods error it caught. ACCEPTED: the verdict (downgrades only, no master-table row) and all ten required changes, including the debate's exact one-sentence paper wording, the residue-first metric rename (lead with "down-after 30000 + median 0.92 s residue", not 30.92 s), the at-risk denominator replacing "0 of 116,541" in any comparative use, the ack-meaning footnote, deletion of "despite asynchronous replication" and the PG C1 contrast sentence, the failed-attempts-as-ticks correction, the redis-py facts replacing "not inspected" (with the retry/duration interaction flagged unexplained), the core-only-build label, the R2 fixed-victim wording, and the Methods restart correction (recorded here as an erratum; the body text stands as written, corrected by this section). One piece of context, not a rebuttal: the timer-domination critique applies to the study's own PG C1 cell too — the difference the debate identifies is real, namely that the PG cells carry a mechanism and loss analysis around the timer (agent death, lag cells, freeze taxonomy) while the Redis slice, at reference depth by design, does not; that is exactly why Redis stays a portability observation and PG remains the deep system, as the owner brief framed it. As with iterations 4 and 5, the measurements document had already disclosed nearly every limitation; the failure is headline labelling and one Methods error, not the measurements. DEBATED VERDICT for iteration 6: Redis survives as one portability sentence in the debate's wording — the probe + ledger discipline runs against a non-SQL Sentinel-discovery client; a primary kill stalls writes ~31 s (the configured 30 s down-after plus a median 0.9 s residue, client rediscovery ~10 ms of it); a replica kill is client-invisible; and no loss appeared in a fault that could not be expected to produce any — and NOT as a cross-system table row. What would upgrade it: per-run Sentinel log capture decomposing the residue (+sdown/+odown/+elected-leader/+switch-master), one lag or partition cell where loss is possible by construction reported with the at-risk denominator, one min-replicas-to-write ≥ 1 cell, and a small test explaining the 0.8 ms failed attempts against redis-py's default retry. Lab stays private; nothing published.

## Completion note

- **Did:** built Redis 8.10.2 from source (sha verified), 1+2+3 default topology, Sentinel-discovery probe, R1 N=30 and R2 N=15 with full ledger reconciliation, CSVs, LAB-ITER6.md, STEPLOG.md.
- **Found:** R1 outage 30.92 [30.42–31.45] s, 0 acked-lost of 116,541; R2 no client impact, 0 of 75,775.
- **Blocked / not done:** per director brief, no mirror to `~/workspace/ironclad/lab/`, no Space update, no debate, no cron removal (owner brief item 3 lists these; director brief says the director does them after verification). Module build failure (not needed). 8.10.3 vs 8.10.2 "stable" choice noted above.

ITERATION 6 READY
