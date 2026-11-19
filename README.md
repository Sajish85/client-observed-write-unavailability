# client-observed-write-unavailability

Client-side measurements of write unavailability after a primary process kill in five replicated database systems at shipped settings, with per-system failure-detection decomposition and acknowledged-write loss accounting.

Sajish Krishnan, independent researcher. Paper: [paper.md](paper.md).

## Summary

A single-writer probe issued uniquely numbered writes to a three-node cluster. The primary was killed with `kill -9` (N=30 per cell), and every write was then reconciled against the surviving data. Five systems ran on one machine over loopback at their shipped settings: PostgreSQL 17.12 with Patroni 4.1.5, MongoDB 9.0.8 (replica set), MySQL 8.4.13 Group Replication, Redis 8.10.2 with Sentinel, and CockroachDB v26.3.6.

Headline findings, as stated in the paper's abstract:

- Median client write outage was 22.38 s for PostgreSQL/Patroni at defaults when the Patroni agent dies with the database, 10.45 s for MongoDB, and 21.49 s for MySQL Group Replication.
- Redis with Sentinel (30.92 s) and CockroachDB (4.27 s stall of the in-flight write) are reported separately because their architectures and the meaning of the measurement differ.
- In the four primary/replica systems the outage tracks failure-detection or election timing rather than promotion work.
- For PostgreSQL, a cell with tighter timing settings and a narrower fault recovered in 1.89 s.
- No acknowledged write was lost in the MongoDB, MySQL, CockroachDB or Redis cells. The PostgreSQL defaults cell lost one acknowledged transaction in each of 2 of 30 runs.
- All results are single-host, loopback, process-kill results. Read the paper's limitations before drawing conclusions.

## What is here

| Path | What |
|---|---|
| `paper.md` | the paper |
| `data/` | per-run result CSVs the paper's tables draw on (one row per run) |
| `lab-notes/` | the study plan (`STUDY.md`) and the measurement notes for each iteration, so the numbers can be audited |
| `LICENSE` | MIT licence text |
| `CITATION.cff` | citation metadata |

`data/` file names: `config-*.csv` (PostgreSQL pilot cells A, B, A-fo, B-fo); `iter2-config-C1.csv`, `iter2-config-C3.csv` (PostgreSQL defaults and tuned-timing cells); `iter3-config-D4.csv`, `iter3-config-D5.csv`, `iter4-config-E1.csv`, `iter4-config-E1c.csv`, `iter4-config-R1.csv` (PostgreSQL sibling cells cited in Section 3.1); `iter5-*` CockroachDB; `iter6-*` Redis; `iter7-*` MongoDB; `iter8-*` MySQL Group Replication. Each `lab-notes/MEASUREMENTS-*.md` lists the columns and cells of the matching iteration.

## Reproduce

This repo holds the results and notes, not the probe or harness code. What you can do from here:

1. Recompute any median in the paper from the `recover_s` column of the matching CSV, for example:

   ```
   python3 -I -c "import csv,statistics as s; print(s.median(float(r['recover_s']) for r in csv.DictReader(open('data/iter2-config-C1.csv')) if r['recover_s']))"
   ```

2. Compare against the matching `lab-notes/MEASUREMENTS-*.md` file and the paper's tables.

Re-running the measurements needs the software versions listed in Table 1 of the paper and a machine of the kind described in Section 2.2.

## Links

- DOI: to be added at deposit

## Licence

- Paper (`paper.md`) and lab notes (`lab-notes/`): CC-BY-4.0, https://creativecommons.org/licenses/by/4.0/.
- Data (`data/`) and any code: MIT, see [LICENSE](LICENSE). This repo contains no code yet.
