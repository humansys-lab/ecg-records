# Migration guide

[From 0.1 to 0.2](#from-ecg-records-01-to-02) is at the end of this page. The
first sections cover the move from the unpublished `ecgfeat-dxl-inspired`
working tree and the legacy `ECGFeatures` object to `ecg-records`.

## In one minute

Before (legacy object; warns, and needs `ecginterpret`):

<!-- doc-check: skip (legacy API; needs the separate ecginterpret distribution) -->
```python
from ecgfeat import ECGFeatureExtractor
features = ECGFeatureExtractor().extract(signal, 500)
```

After:

```python
import numpy as np
from ecgfeat import ecg_record, query_measurement

signal = np.load("ecg.npy")
leads = ["I", "II", "III", "aVR", "aVL", "aVF", "V1", "V2", "V3", "V4", "V5", "V6"]
record = ecg_record(signal, sampling_rate=500, lead_names=leads)
qrs = query_measurement(record, "qrs_duration_ms", lead="V1", beat=0)
```

The new API requires explicit lead names (the legacy extractor defaulted to the
canonical order when `lead_names` was omitted) and returns a versioned,
validated record instead of a mutable object graph.

## Imports

| Old | New | Old path supported until |
|---|---|---|
| `from ecgfeat import ECGFeatureExtractor` / `ecgfeat.api` | `ecgfeat.ecg_record(...)` for records; `ecgfeat.compat.api_v0.ECGFeatureExtractor` for the exact legacy object | warns now; removal no earlier than 0.3.0 |
| `ecgfeat.to_dict`, `ecgfeat.export.to_dict` / `prepare_json_export` | `ecgfeat.dumps_record(record)` / `record_to_dict`; legacy payload: `ecgfeat.compat.export_v0` | warns now; ≥ 0.3.0 |
| `ecgfeat.models.*` dataclasses | `ecgfeat.compat.models_v0.*` (same classes) | silent alias; ≥ 0.3.0 |
| `ecgfeat.interpret`, `ecgfeat.clinical_rules`, `ecgfeat.glasgow_rules`, … | the separate [`ecginterpret`](https://pypi.org/project/ecginterpret/) distribution (0.2.1 on PyPI; it installs `ecg-records[engine]` 0.3): `ecginterpret.analyze(signal)` or `ecginterpret.interpret_record(record, signal=...)` | aliases warn; ≥ 0.3.0 |
| `ecgfeat.plot_beat`, `plot_rep_beat`, … / `ecgfeat.visualize` | `ecgfeat.viz.plot_beat(signal, record, ...)` (`pip install "ecg-records[viz]"`) | warn; ≥ 0.3.0 |
| `ecgfeat.load_wfdb_mat`, `parse_wfdb_header` | `ecgfeat.io.read_wfdb`, `ecgfeat.io.read_wfdb_header` | warn; ≥ 0.3.0 |
| `ecgfeat.delineate`, `ecgfeat.quality`, `ecgfeat.features`, … (private engine) | no public replacement; use record fields | alias warns; ≥ 0.3.0 |

`ecgfeat.compat` is the explicit legacy contract: it does not warn, and the
whole namespace is removed together no earlier than 0.3.0.

## Values

Every published record value maps exactly to a legacy leaf through the
reviewed [crosswalk](reference/legacy-crosswalk.md) (sample indices converted
to native sampling rate, milliseconds and microvolts rounded half-to-even).
Values that violate a published invariant are published as `unmeasurable`
instead. The legacy payload itself is unchanged: `ecgfeat.compat.export_v0`
reproduces the pre-migration bytes on 1,063 golden records.

## Records vs. legacy object

The legacy object carries many intermediate quantities that ECG Record 1.0
does not publish (representative-lead morphology, rhythm inputs, P-wave
assessments). Consumers that need them keep using `ecgfeat.compat` during the
compatibility window; new quantities become record fields only with a unit,
absence semantics, provenance, a validation entry and size accounting.

## Interpretation is no longer part of this package

The legacy extractor and the legacy JSON export include interpretation
sections, and interpretation now lives in the separate `ecginterpret`
distribution. Without it, `ECGFeatureExtractor().extract(...)`, `to_dict(...)`
and `ecgfeat.interpret` raise an `ImportError` that names the missing
distribution. The record API never needs interpretation.

## Turning warnings into errors

To find every remaining legacy call, run your tests with deprecation warnings
as errors:

```bash
python -W error::FutureWarning -m pytest
```

`ECGDeprecationWarning` is a `FutureWarning`, so this catches exactly the
legacy calls. See [Errors and warnings](guide/errors.md#warnings).

## From ecg-records 0.1 to 0.2

Nothing that 0.1 published changes value, unit or absence encoding on the
same input; 0.2 writes schema 1.1.0 (additive) and reads 1.0.0 records
unchanged. The engine's measurement corrections since 0.1.0 are included, so
values can differ from 0.1 on the same signal.

| 0.1 | 0.2 |
|---|---|
| `ecg_record(np.load(...), sampling_rate=..., lead_names=[...twelve names...])` | `ecg_record(read_signal(path))`, or `lead_names="standard_12"` |
| `ecgfeat.io.read_wfdb` (MATLAB pairs only), or the `wfdb` package | `read_signal` (WFDB `.dat`/`.mat`, EDF/BDF, CSV, NumPy) |
| map `AVR`/`avr` to `aVR` yourself | standard names are normalized |
| a loop over `query_measurement` | `record.to_dataframe("global" \| "leads" \| "beats" \| "long")`, `record.to_rows()`, `record.to_csv()` |
| median of per-beat cells for a record-level interval | `record.get("qt_interval_ms")` (the engine's global value) |
| `ecg-record measure x.npy --sampling-rate 500 --lead-names ...` | `ecg-record measure x.hea` (metadata from the file) |
| `ecg-record batch --jobs N` with threads | worker processes; directories and globs; `--table` |
| `plot_record` with per-lead y-scales | one shared millivolt span by default (`y_scale="independent"` for the old look) |
| five fiducial hues | three hues by wave plus marker shape by landmark |
| `ExtractionConfig` | `ECGConfig` (`ExtractionConfig` warns; removed no earlier than 0.4.0) |
| `ecg_record(...)` then `ecginterpret.interpret_record(record, signal=...)` | `ecginterpret.analyze(signal)`: one extraction for both |

Behaviour changes to note: summary records are up to 30,000 bytes (the
reference case is 29,084); `plot_quality_summary` shows a noise heat map for
records that carry quality metrics (`metrics="rms"` for the 0.1 bar chart);
`ecg-record batch --jobs N` uses processes (`--threads` for threads).
