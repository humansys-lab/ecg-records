# Compatibility and versioning

## Two version numbers

| Version | Where | Current | Changes when |
|---|---|---|---|
| **Package** | `ecgfeat.__version__`, PyPI | 0.2.0 | code changes; follows Semantic Versioning |
| **Schema** | `record.schema_version`, `/provenance/schema_version` | 1.1.0 | the record format changes |

They move independently. Schema rules:

- A **schema minor** (1.0 → 1.1) only adds: new optional fields, new reason
  codes, new validation evidence. It requires at least a package minor
  release.
- A **schema major** (1.x → 2.0) may change meaning, units or absence
  semantics. It requires a package major release.

## Reading and writing

- `ecg-records 0.2.x` writes schema `1.1.0`; 0.1.x wrote `1.0.0`.
- Both read every schema `1.*` record. Readers ignore members they do not
  understand, so a consumer that uses only `1.0` fields can read any later
  `1.*` record, and values keep their meaning, unit and absence semantics.
  Schema 1.1 adds fields only inside the five 1.0 measurement groups and
  optional members of existing objects; the released 0.1.0 reader loads 1.1
  records in strict mode (release-tested).
- One caveat for 0.1 readers of 1.1 records: four record-level names
  (`p_duration_ms`, `pr_interval_ms`, `qrs_duration_ms`, `qt_interval_ms`)
  also exist as per-beat cells, and the 0.1 `query_measurement` resolves an
  unqualified name to the per-beat field. 0.2 resolves by selector and
  accepts group-qualified names (`"global/qt_interval_ms"`).
- Each schema family ships its own validation-evidence registry; a record is
  checked against the registry of the family it declares.
- Records of an unsupported major version are rejected on read.

## What is stable

| Surface | Stability |
|---|---|
| `ecgfeat.__all__` public API (`ecg_*`, record read/query functions, configuration and error types) | stable within a major version, with the pre-1.0 caveat below |
| ECG Record members, fields, units and absence semantics in `summary` and `all` | stable within the schema major |
| Error `code` values | stable; new codes may be added |
| `ecg-record` command line, output formats and exit codes | stable within a major version |
| `/debug` content, the `debug` profile's extra members | **no guarantee**; may change in any release |
| Private modules (`ecgfeat._engine`, `ecgfeat.pipeline.stages`, …) | no guarantee |
| `record_id` for identical input | changes with every library version (see [Record identity](guide/measuring.md#record-identity)) |

Before 1.0, a minor release (0.1 → 0.2) may still change the public Python API
after a deprecation period. Patch releases (0.1.0 → 0.1.1) never break it.

## Deprecation policy

- The legacy API (`ECGFeatureExtractor`, `to_dict`, `interpret`, `plot_*`,
  `ecgfeat.visualize`, the WFDB helpers, the legacy result dataclasses and the
  old private module paths) warns from 0.1.0, the first warning release.
- Deprecated names stay for at least two minor releases and are removed **no
  earlier than 0.3.0**.
- In the release before removal, the old name turns into a *tombstone* that
  raises an `ImportError` naming its replacement.
- `ecgfeat.compat` is the explicit legacy contract. It does not warn and is
  removed as a whole, also no earlier than 0.3.0.
- Deprecated in 0.2.0 and removed no earlier than 0.4.0: `ExtractionConfig`
  (use `ECGConfig`), `extract_record` and `extract_ecg_record` (use `ecg_record`).

## Supported environments

Python 3.10–3.14, with NumPy 1.26 or 2.x and SciPy ≥ 1.11, on Linux, macOS and
Windows. CI runs the test suite on Python 3.10–3.14 with NumPy 1.26 and 2.x, and
the release smoke test pins the oldest supported NumPy, SciPy and Matplotlib
(1.26, 1.11, 3.7).

## Releases are immutable

Released artifacts are never replaced: PyPI does not allow re-uploading a
version. Defects are fixed forward in a new release, and each release's
changes are listed in `feature_extraction/CHANGELOG.md`.
