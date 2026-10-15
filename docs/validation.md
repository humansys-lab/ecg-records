# Validation and benchmarks

## What is (and is not) claimed

A field may claim a better status than **`unvalidated`** only by citing a
passing **evidence manifest** (`benchmarks/evidence/`), and
`tools/check_validation_registry.py` checks every claim mechanically. Schema
1.0.0 fields are all `unvalidated`. In schema **1.1.0** (ecg-records 0.2.0)
four record-level fields are `benchmark_validated`; the other 73 published
fields are `unvalidated`. None of this is a clinical-performance claim.

Each field's status is in the record (`/<field>/validation/status`) and in
every query result (`result.validation.tier`). It comes from the
validation-evidence registry shipped for the record's schema family
(`ecgfeat/schemas/ecg-record/<1.0|1.1>/validation-evidence.json`). The strict
reader rejects any record that claims more than its family's registry allows
(`validation_above_registry`).

| Record status | Query tier (`result.validation.tier`) | Meaning |
|---|---|---|
| `unvalidated` | `unvalidated` | no passing evidence |
| `indirectly_validated` | `partially_validated` | supported by evidence against another algorithm or a related endpoint |
| `benchmark_validated` | `validated` | a manifest shows the field meets a stated threshold against an expert reference on a named dataset |
| `known_problem` | `unvalidated` | known to be unreliable; the debug-only four-class ST morphology is capped here |

The registry also records a **validation ceiling** per field: the best status
the field could ever reach with evidence. See the schema references
([1.1](reference/ecg-record-schema-1.1.md), [1.0](reference/ecg-record-schema-1.0.md))
and the [validation policy](development/validation-policy.md).

## Record-level intervals (schema 1.1.0)

Criterion: IEC 60601-2-25:2011, 201.12.4.102.3.2, global intervals, evaluated
the CSE way: after removing the 8 differences farthest from the initial mean,
the mean difference and its standard deviation must lie within the limits.
Differences are record minus reference, in ms. Every run used the published
entry point (`ecg_record`, default configuration) on a clean commit with the
same engine as the 0.2.0 release.

| Field | Limits (mean / SD) | LUDB 1.0.1 vs experts | PTB-XL hold-out vs GE 12SL | Status |
|---|---|---|---|---|
| `p_duration_ms` | ±10 / 15 | +0.2 / 11.5 (n=164) ✔ | +3.6 / 16.9 (n=2,657) ✘ | benchmark_validated |
| `pr_interval_ms` | ±10 / 10 | +2.9 / 5.5 (n=162) ✔ | +0.7 / 10.7 (n=2,601) ✘ | benchmark_validated |
| `qrs_duration_ms` | ±10 / 10 | +2.7 / 6.6 (n=192) ✔ | −0.1 / 7.9 (n=2,992) ✔ | benchmark_validated |
| `qt_interval_ms` | ±25 / 30 | +9.0 / 10.7 (n=191) ✔ | −0.1 / 13.7 (n=2,983) ✔ | benchmark_validated |

- **LUDB** (200 records, 500 Hz, 12 leads, adults and children, many rhythms)
  is the expert reference: record-level values are reconstructed from the
  per-lead expert annotations. It is a proxy for the CSE database, not a
  conformance test.
- **PTB-XL** (a pinned, patient-disjoint 3,000-record hold-out) is compared
  with GE 12SL from PTB-XL+ 1.0.1: indirect evidence, since the reference is
  another algorithm. P duration and PR fall outside the limits against 12SL.
  For context, Glasgow Uni-G against 12SL on the same records gives P −0.4 /
  14.9 ms and PR −4.4 / 12.5 ms: the two machine algorithms also disagree on
  PR by more than the limit.
- QT values whose feature state is not `observed` (about 30%: the QT
  reportability gate) are included above; restricted to observed values,
  LUDB gives +8.6 / 6.3 ms and PTB-XL −0.6 / 8.2 ms.
- Coverage (published values / records with a reference): LUDB 97–100%,
  PTB-XL 96–100%.

### Not validated

| Endpoint | Result | Criterion |
|---|---|---|
| per-lead fiducials on LUDB (P onset/offset, QRS onset/offset, T offset) | error SD 36.9, 33.8, 12.0, 15.5, 38.0 ms; mean +11.3, +0.5, +0.4, −1.9, +7.1 ms | CSE delineation tolerances on the SD (10.2, 12.7, 6.5, 11.6, 30.6 ms): not met |
| heart rate vs 12SL (PTB-XL) | mean +0.0, SD 2.3 /min | none applied |
| QTc Bazett / Fridericia vs 12SL | mean +1.2 / +0.5, SD 25.4 / 22.3 ms | none applied |
| P / QRS / T frontal axis vs 12SL | mean +0.6 / −0.5 / −0.5, SD 13.8 / 8.7 / 15.1° | none applied |

These stay `unvalidated`. The manifests, raw outputs (`runs/`), tool and
dataset hashes and every number above are in `benchmarks/evidence/`; the
evaluation tools in `benchmarks/evidence/tools/` use the LUDB annotation
harness of the research repository and the mounted datasets.

## Regression gates

The repository guards against unintended changes in what the library
measures. These gates bound regressions from an accepted baseline. They are
**not** claims of clinical accuracy.

- **Golden corpus** (`snapshot_regression.py`, [details](development/golden-corpus.md)):
  156 sentinel and 1,063 full-tier records from seven public datasets (LUDB,
  QTDB, EDB, PTB-XL, BUT-PDB, NSTDB, GUDB). The frozen `ecg-records-0.2.0`
  baseline holds the SHA-256 of every record profile that release produces
  (`ecg-records-0.1.0` keeps the previous release's).
  Any change to a published value shows up as a difference. The older
  `phase0-52a339c` baseline preserves the pre-migration outputs: the legacy
  export reproduces them byte for byte, and every record value equals its
  legacy value through the [crosswalk](reference/legacy-crosswalk.md).
- **Contract, property and fixture tests**: schema and invariant checks,
  Hypothesis property tests of the record builder, the pinned 10 s reference
  record and the summary byte contract (30,000 bytes in schema 1.1; 24,000 in 1.0).
- **Live crosswalk** (`tools/check_record_crosswalk.py`): on the 108 default
  standard and limited sentinel cases, every published value equals the legacy
  export of the same engine run under independently re-implemented rules.
- **Forward compatibility**: the released 0.1.0 reader loads 0.2.0 records in
  strict mode.
- **Performance budget** (`tools/check_performance.py`): import time, first and
  warm extraction time, and peak memory on the reference record.
- **Layer contracts** (`import-linter`): the record API, the engine, the
  pipeline and the plotting package stay separated.

## Known weaknesses

- P-wave delineation is the weakest endpoint in the benchmarks, especially at
  short RR intervals or when the P wave overlaps the preceding T wave.
- LUDB distributes amplitude-normalized signals, so amplitude accuracy cannot
  be assessed on LUDB.
- The four-class ST morphology was never validated and is anti-correlated with
  ischemia. It is not published (debug-only, capped at `known_problem`).

See also [Limitations and intended use](limitations.md).
