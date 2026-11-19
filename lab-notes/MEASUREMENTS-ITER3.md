# MEASUREMENTS-ITER3.md — PostgreSQL/Patroni failover lab, iteration 3 (17–18 Nov 2026)

Build and re-run: LAB.md "ITERATION 3". Data: `results/iter3-config-{D1..D5}.csv`, log-verified phases `results/iter3-phases-D*.csv`, raw per-run files `runs/iter3-Dx-rNN/`, per-cell Patroni/etcd/Consul logs `runs/iter3-logs/<cell>/`. Same host as before: one Mac, loopback, one clock. PG 17.12 (D5: 16.16), Patroni 4.1.5, etcd 3.7.3 (D4: Consul 1.16.4). All five cells ran with full N (30/15/15/30/30). Iteration 2 outputs were complete; nothing redone.

## Summary (answers to Q1–Q5)
| Q | Answer | Strength |
|---|---|---|
| Q1 sync-STRICT vs sync vs async (lag cell) | **Strict looks the same as plain sync on this fault.** D1: 0/30 acked txns lost (C5 sync 0/30, C4 async 6/30), client outage **6.02 s median [4.55–11.46]** vs C5 6.13 [4.83–6.72] (permutation test on medians p = 0.60) vs C4 async 4.08 [3.27–7.75]. Strict's distinguishing behaviour (blocking writes when no sync standby exists) was **not exercised**: this fault never removes the sync standby. | Established for "strict = sync under a primary-postmaster kill"; strict's own mechanism not tested |
| Q2 candidate dies mid-promotion (D2) | **Cluster converged in 15/15 runs, always on the third node.** Client outage **6.03 s [5.76–8.69]**, i.e. about the pilot's no-second-fault async PST=0 outage (A-fo 5.93 [5.44–7.86], n=10); the second kill costs ≈0.1–0.3 s. | Established at N=15 for this trigger |
| Q3 replica dies during failover (D3) | **No measurable effect.** 15/15 converged on the promoting candidate, outage **5.75 s [5.52–7.94]** (A-fo baseline 5.93). | Established at N=15 (the killed replica was the non-promoting one) |
| Q4 DCS matters? (D4 Consul vs C1 etcd) | **Small, statistically clear difference.** Agent-death outage **21.04 s [20.12–30.15]** on Consul vs **22.38 s [20.67–27.64]** on etcd (C1); ≈1.3 s faster, permutation p = 0.0001. 30/30 promotions each. Mechanism not verified. | Established for this setup; confounded by run time (see limits) |
| Q5 PG version matters? (D5 PG16 vs C1 PG17) | **No difference detected.** 22.35 s [20.92–28.31] vs 22.38 [20.67–27.64], p = 0.67. | Established as "not detectable at N=30" |

## Diagram: what each iteration-3 fault touches
```mermaid
flowchart LR
  H[harness] -->|1 SIGKILL| PM1[old primary postmaster]
  PM1 -.->|its agent notices ~6 s later, releases key| DCS[(DCS: etcd or Consul)]
  DCS -->|key free: replicas race| CAND[candidate agent]
  H -->|poll key every 5 ms| DCS
  H -->|D2: 2nd SIGKILL| CPM[candidate postmaster]
  H -->|D3: 2nd SIGKILL| RPM[other replica postmaster]
  CAND -->|promote| CPM
  CL[client probe] --> CPM
  CPM -.->|D2: its agent releases key, 3rd node takes over (inferred from timing + logs)| DCS
```
Walkthrough: the harness kills the primary's postmaster (arrow 1). Its still-living agent detects that and releases the DCS leader key; that detection is the ~6 s. The harness polls the key; the instant another node holds it (candidate has the lock, promotion starting) it fires the second kill: D2 hits the candidate's postmaster, D3 hits the other replica's. For D2 the candidate's agent then also sees a dead postmaster and gives the key up, and the remaining replica promotes (dotted arrow: inferred; log lines "Leader key released" appear on all nodes in the D2 logs, per-run ordering not individually checked).

## Per-cell results (seconds after the first fault; median [min–max])
| Metric | D1 sync-strict, lag | D2 kill candidate | D3 kill replica | D4 Consul, agent kill | D5 PG16, agent kill |
|---|---|---|---|---|---|
| N / outcomes | 30 promotion | 15 multi-fault convergence | 15 multi-fault convergence | 30 promotion | 30 promotion |
| detect_s (client sees failure) | 0.004 [0.002–0.012] | 0.006 [0.003–0.009] | 0.005 [0.003–0.007] | 0.005 [0.003–0.008] | 0.005 [0.003–0.009] |
| leader_change_s | 6.05 [4.48–9.25] | 5.87 [5.54–8.02] | 5.83 [5.61–7.91] | 21.12 [20.11–30.27] | 22.36 [21.01–28.18] |
| second kill fired (trigger_s) | n/a | 5.73 [5.49–7.83] | 5.66 [5.46–7.89] | n/a | n/a |
| **recover_s (client)** | **6.02 [4.55–11.46]** | **6.03 [5.76–8.69]** | **5.75 [5.52–7.94]** | **21.04 [20.12–30.15]** | **22.35 [20.92–28.31]** |
| acked_lost | 0 in 30/30 | 3 txns in 3/15 (1 each) | 1 txn in 1/15 | 2 txns in 2/30 | 1 txn in 1/30 |
| acked_lost after leader change | 0 | 0 | 0 | 0 | 0 |
| unacked_present (applied, ack unseen) | 6 txns in 6/30 | 0 | 0 | 0 | 0 |
| settle_s (cluster fully healthy) | 21.1 [19.6–26.7] | 21.5 [21.2–23.9] | 21.2 [21.0–23.4] | 41.4 [40.3–50.5] | 41.9 [40.4–47.5] |
| converged on | new primary pg1/2/3 = 10/10/10 | pg2 6, pg1 4, pg3 5 (never the killed candidate, never the old primary) | candidate survived 15/15 | pg2 11, pg1 9, pg3 10 | pg2 11, pg3 11, pg1 8 |
| trigger-time state of the killed candidate | n/a | `in_recovery` 15/15 | `in_recovery` 15/15 (candidate) | n/a | n/a |

Raw per-run numbers are in the CSVs; "settle" for D4/D5 includes the harness restarting the killed agent (same as C1: 41.9 s).

## Q1 detail: strict vs sync vs async (lag cell, heavy writes, 5 s apply delay on the sync standby, PST=0)
| | C4 async | C5 sync | D1 sync-strict |
|---|---|---|---|
| recover_s | 4.08 [3.27–7.75] | 6.13 [4.83–6.72] | 6.02 [4.55–11.46] |
| acked_lost | 6 in 6/30 | 0/30 | 0/30 |
| unacked_present | 0 | 7 in 7/30 | 6 in 6/30 |
| replay lag at kill (bytes, median) | ≈29.1 MB | ≈28.9 MB | ≈29.0 MB [22.7–29.3 MB] |
| flush gap at kill (bytes) | median 0, max 9,280 | median 0, max 4,616 | median 0, max 5,776 |
- Lag was recorded for every D1 run (`lag_*` columns, raw `lag_json`). As in iteration 2 the 5 s apply-delay produces **replay** lag (~3.6 s), not receive/flush lag, so it does not create a shipping gap.
- Strict vs plain sync: identical loss (0/0), indistinguishable recovery time, same ambiguous-commit rate (6/30 vs 7/30). C4→D1 inherits iteration 2's +2 s sync cost (Patroni's crash-failover decision logs at 5.84 s median in D1 vs 3.96 s in C4); why sync delays that decision is still **not established**.
- D1 had two slow runs: 9.14 s (run 4) and 11.46 s (run 3). Run 3: the new primary was promoted and logged "ready" at ≈5.9 s but the client's first commit came at 11.46 s; its probe errors in that window include "session is read-only" from port 5441. I did not find the cause (unexplained; not filtered out of the medians). Strict mode holds commits until a sync standby is attached (documented Patroni behaviour); the log shows the sync standby assigned ≈1 s after promotion, which does not account for 5.6 s, so this explanation is not supported for run 3.
- **Not tested:** strict's actual purpose, i.e. the sync standby dying while the primary lives (strict blocks writes, plain sync degrades to async). D3 killed a replica but in async mode.

## Q2/Q3 detail: mid-failover faults
- The second kill was injected 2–3 ms after the DCS leader key changed hands (`second_kill_s − trigger_s`). In 15/15 D2 runs the candidate was **still in recovery** at that instant (state read just before the kill), so the kill landed inside the promotion window as intended. Same for the candidate in D3's trigger sample.
- **D2:** old primary (pg1/2/3 in rotation) + candidate killed → the third node promoted in every run; Patroni on the candidate released the key itself (log: "Primary crashed. Failing over / Demoting self (immediate) / Leader key released", seen in the first smoke run's logs and consistent with 15/15 "crash_failover" events in the D2 phase file, median 5.67 s). Added outage versus a single-fault run is small because the second election is a second ≈0.1 s promotion, not a second 6 s detection: the killed candidate's agent notices and gives up the key within tens of ms of the kill (inferred from log timestamps in the smoke run).
- D2 vs D3 recover medians: 6.03 vs 5.75 s, permutation p = 0.001 (excluding each cell's cold first run: 6.00 vs 5.74). There is no within-iteration single-fault baseline; the pilot A-fo (5.93, n=10, different day-hour) sits between them. So the **cost of the second fault is ≈0.3 s at most and I cannot separate it from run-to-run drift**.
- First run of each of D2/D3 was slow (8.69 s, 7.94 s): cold cluster start. They are included.
- **Data:** D2 3/15 runs and D3 1/15 each lost exactly 1 acked txn, all before the leader change (`acked_lost_after_leader_change` = 0). That is the same last-milliseconds async window seen in iteration 2 (C1: 2/30; C4: 6/30 heavy). D2's 3/15 is nominally higher than C1's 2/30 and A-fo's 0/10 but n is tiny; **no claim that a second fault raises loss** (Fisher exact, two-sided 3/15 vs 2/30: p = 0.31).
- No run failed to converge; settle ≈21 s, same as single-fault runs.

## Q4/Q5 detail: DCS and PG version (agent death: postmaster + Patroni SIGKILL, async, default ttl30/loop10/retry10, PST 300)
| | C1 etcd, PG17 (iter2) | D4 Consul, PG17 | D5 etcd, PG16 |
|---|---|---|---|
| recover_s median [min–max] | 22.38 [20.67–27.64] | **21.04 [20.12–30.15]** | 22.35 [20.92–28.31] |
| quartiles (p25/p50/p75) | 22.22/22.38/22.53 | 20.81/21.04/21.26 | 22.14/22.35/22.48 |
| log: "promoted self…" (survivor) | 22.29 | 21.00 | 22.25 |
| acked_lost | 2 in 2/30 | 2 in 2/30 | 1 in 1/30 |
- The outage here is the leader key's remaining lifetime after the agent died (as in iteration 2). With Consul, Patroni creates a session whose TTL it sets to `ttl/2` (Patroni's consul.py comment: "Consul multiplies the TTL by 2x"), so the session should expire between 15 and 30 s after the last renewal. That the median is ≈1.3 s lower than etcd is consistent with a different expiry/renewal mechanism but **I did not verify the cause** (no Consul session-expiry timestamps extracted).
- D4 has one 30.15 s tail run; its IQR is tight (0.45 s). Per-cell spread in C1/D5 is similar (20.7–28.3).
- **Confound:** C1 ran roughly 5 h before D4 and 5.5 h before D5 on the same Mac; run order was fixed (not randomised across DCS/version). A 1.3 s shift could in principle come from background load differences, though D5, run later than D4 under the same conditions as C1, matches C1 to 0.03 s.
- Consul was a **single dev-mode agent** (`consul agent -dev`), matching the single-member etcd: neither cell tests multi-node consensus. Consul 1.16.4 was chosen because it is the last tag whose repository LICENSE is MPL-2.0 (1.16.5 and later are BUSL; 1.16.6 was downloaded first and replaced). PG16 is 16.16 vs the 17.12 used elsewhere.

## Updated cross-iteration claims table
| Claim | Evidence | Iterations supporting | Status |
|---|---|---|---|
| Under defaults, killing only the primary's postmaster causes no failover; the agent restarts it in place (~6 s outage) | A, B: 20/20 restart-in-place, 5.96 s | Pilot only | Established (n=10 each); not re-run |
| With `primary_start_timeout=0` the same fault becomes a promotion at ≈6 s | A-fo 5.93; D1/D2/D3 ≈5.8–6.0 | Pilot, 3 | Established (n=10 + 60) |
| Outage is dominated by the failed node's agent decision (≈5.7–6 s), not promotion (≈0.1 s) | Phase logs: promote→ready 0.02–0.05 s; crash_failover logged at 5.6–5.8 s | Pilot, 2, 3 | Established |
| Killing agent too moves recovery to TTL expiry (~22 s, ttl 30) | C1 22.38; D5 22.35; D4 21.04 | 2, 3 | Established (n=90) |
| ...and that holds on PG16 | D5 vs C1, p=0.67 | 3 | Established, no difference detected |
| ...and is ≈1.3 s shorter on Consul | D4 vs C1, p=0.0001 | 3 | Established for this setup; cause unverified; time-confounded |
| Tighter timings cut detection (5.93 → 1.89 s with ttl10/loop2/retry3) | C3 | 2 | Established; false-failover cost untested |
| Async can lose the last acked txn in a kill; sync prevented it in the heavy lag cell | C4 6/30 vs C5 0/30 (p=0.012); D1 0/30 | 2, 3 | Established at this rate; loss is the last-ms window, not apply-delay |
| `synchronous_mode_strict` changes the picture | D1 = C5 on loss and time | 3 | **Not established** (no difference found; strict's blocking behaviour untested) |
| Sync costs ≈2 s extra outage vs async | C5−C4 +2.0 s; D1−C4 +1.9 s; pilot B-fo 7.66 vs A-fo 5.93 | Pilot, 2, 3 | Established; cause unexplained; (B vs A-fo pilot gap 1.7 s) |
| A second fault during failover (candidate or replica) does not stop convergence | D2 15/15, D3 15/15 | 3 | Established for these triggers, this host |
| ...and adds little outage | D2 6.03, D3 5.75 vs A-fo 5.93 | 3 | Weak: ≤≈0.3 s, not separable from drift |
| Postmaster-only freeze is invisible to Patroni and clients | C2 15/15 no effect | 2 | Established |
| Agent+postmaster freeze → zombie primary acking thousands of writes during TTL path (~38 s) | C2a n=10 | 2 | Established (labelled variant) |
| Mid-failover faults increase data loss | D2 3/15, D3 1/15 vs C1 2/30, A-fo 0/10 | 3 | **Not established** (n too small, no baseline) |

## Limitations and anomalies
- Same single-host, loopback, process-fault-only design as before; no network faults, no host loss, no clock skew. All conclusions are about process kills.
- **Not randomised:** D1–D5 ran in fixed order; within D1 the random element is only which replica gets the delay (seeded). Brief asked for randomisation "where the harness allows"; run order within a cell was not shuffled.
- D2/D3 have no concurrent single-fault control; the comparison uses the pilot A-fo cell (n=10, light load, earlier in the day).
- Tests are permutation tests on medians, uncorrected for multiple comparisons.
- D1 outliers (9.1 s, 11.5 s) unexplained; D2/D3 run 1 slow (cold start).
- "Second kill timing": trigger = 5 ms poll of the etcd key `/service/lab/leader`; the candidate's state at kill was read with a separate connection a few ms before the SIGKILL. The 15/15 `in_recovery` state shows the kill landed before the promotion completed (timeline switch) but not exactly which sub-step of promote.
- Smoke runs for D2/D4/D5 (one run each) were discarded before the real cells (their data are in no CSV). During D1–D3 I only wrote/edited files and downloaded packages; I did not start any other cluster.
- Consul/PG16 runs share `cluster/` paths with etcd runs; logs preserved per cell in `runs/iter3-logs/`.
- Novelty versus literature/Patroni docs: still unchecked.
- No sudo used. Nothing published, pushed or posted. Open-source software only (Consul 1.16.4 MPL-2.0 tag; PG16 via Postgres.app 2.9.7); no employer material.

## Completion note
- **Did:** extended the harness (mid-failover kill triggered by DCS key change, strict mode, Consul and PG16 knobs); ran D1 (30), D2 (15), D3 (15), D4 (30), D5 (30) with logs preserved per cell and phases extracted; updated LAB.md.
- **Found:** strict ≈ plain sync here; second faults during failover converge in 30/30 with ≈0–0.3 s extra; Consul ≈1.3 s faster than etcd on the TTL path; PG16 = PG17.
- **Blocked / not established:** why Consul is faster; why sync adds ≈2 s; strict's write-blocking behaviour; D1 outliers; whether mid-failover faults change data loss; anything about real networks.

ITERATION 3 READY
