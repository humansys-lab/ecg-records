# Changelog — ecg-records

All notable changes to the `ecg-records` distribution (import name `ecgfeat`).
The ECG Record *schema* is versioned separately (see `schema_version` in every
record); this file tracks the package.

## [Unreleased]

## [0.3.1] - 2026-10-16

Shell-only patch: works with `ecg-records-engine` 0.3.0 (any 0.3.x).

### Fixed
- Project links on PyPI point to the documentation site and the public
  repository `humansys-lab/ecg-records` (0.3.0 linked the source repository,
  which is no longer public).
- Documentation: current-version mentions, the `ecginterpret` version
  (0.2.1, which requires ecg-records 0.3 with the engine), the P-wave
  settings known issue (still present in 0.3), and the quickstart notebook's
  install line and outputs.

## [0.3.0] - 2026-10-15

### Added
- Three more per-beat, per-lead landmarks in `/delineation/fiducials`
  (profiles `all` and `debug`; the `summary` byte contract is unchanged):
  `p_peak`, `t_onset` and `t_peak` (native samples, engine `p/peak`, `t/onset`,
  `t/peak`), additive within schema 1.1. A P peak outside its P bounds, a T
  onset after the T offset or a T peak outside its T bounds is withheld with
  `fiducial_order_violation`; the final P/T bounds and every interval keep
  their values.
- `ecgfeat.viz`: `plot_record`, `plot_beat`, `plot_representative_beat`,
  `plot_clinical` and `export_landmarks` draw/export the P peak, T onset and
  T peak when the record carries them (colour names the wave, shape the
  landmark: `^` peak, `>` onset).

### Changed
- **Distribution split.** The numerical engine (`ecgfeat._engine`) now ships
  only as the compiled `ecg-records-engine` wheels (CPython 3.10–3.14; Linux
  x86_64/aarch64, macOS x86_64/arm64, Windows x86_64), licensed PolyForm
  Noncommercial 1.0.0 or commercially. `ecg-records` stays Apache-2.0 and pure
  Python; install both with `pip install "ecg-records[engine]"` (any versions of
  the same minor version work together, so a shell-only or engine-only patch
  release does not need the other package). Without the engine, measuring raises `EngineNotInstalledError`;
  records can still be read, queried and plotted. New: `ecgfeat.license_info()`,
  `EngineNotInstalledError`, `EngineVersionError`; offline licence keys
  (`ECG_RECORDS_LICENSE_KEY`, `ECG_RECORDS_LICENSE`, `~/.config/ecg-records/license.key`).
- The Kalman recursion of the compiled engine is a Cython kernel
  (`_kalman_kernel.pyx`, bit-identical to the NumPy/Numba reference); Numba is
  only used by source checkouts.
- Atrial measurement (engine): the P boundary uses the T-corrected primary
  signal when it agrees better, with a record-quiet noise fallback; one
  conducted P is kept per cardiac cycle and weak extra atrial events are
  dropped in anchored cycles. On PTB-XL, PR availability rises (dev .902→.913,
  hold-out .972→.979) with PR error unchanged; LUDB global measurements are
  byte-identical.

## [0.2.0] - 2026-10-14

Writes ECG Record schema **1.1.0** (additive) and reads 1.0.0 unchanged. Every
value 0.1 published keeps its value, unit and absence encoding on the same
input; the engine's measurement corrections of 2026-09-24..10-14 are included.

### Added
- **Schema 1.1.0** (`ecgfeat/schemas/ecg-record/1.1/`, own validation registry):
  - record-level fields in `/measurements/global`: `p_duration_ms`,
    `pr_interval_ms`, `qrs_duration_ms`, `qt_interval_ms`, `qtc_fridericia_ms`,
    `qtc_framingham_ms`, `qtc_hodges_ms`, `p_axis_deg`, `t_axis_deg`,
    `qrs_t_angle_deg`, `atrial_rate_bpm`, `heart_rate_min_bpm`/`max`, `rr_mean_ms`,
    `rr_sd_ms`, `rmssd_ms`, `pnn50`, `rr_cv`, `qt_dispersion_ms`, `ptf_v1_uv_ms`,
    `sokolow_lyon_uv`, `cornell_voltage_uv`, `paced_beat_fraction`,
    `pacing_spike_count`, `organized_p_fraction`, `atrial_residual_rms_uv`,
    `atrial_dominant_cycle_ms`;
  - median-beat per-lead fields (`median_*`): P/Q/R/S/R′/T amplitudes, ST at J,
    J+40, J+60, J+80, P duration, PR, QRS, QT, Q duration, QRS area;
  - per-beat `rr_interval_ms`; beat `group`, `dominant`, `paced`;
  - `/axes/atrial_events` and `/axes/pacing_spikes` (all/debug);
  - per-lead quality `grade` (every profile) and `reason_codes`,
    `reliable_for`, noise `metrics` (all/debug);
  - per-cell feature states (`observed`, `unassessable`, ...) on every field.
- **Validation**: the four record-level intervals (P duration, PR, QRS, QT)
  are `benchmark_validated` on LUDB 1.0.1 against IEC 60601-2-25 limits (CSE
  trimming); QRS and QT also agree with GE 12SL on the PTB-XL hold-out (indirect
  evidence). Manifests, tools and the non-passing results (per-lead fiducials;
  P duration and PR against 12SL) are in `benchmarks/evidence/`.
- **File readers**: `read_signal` for WFDB (formats 16, 212, 24, 32, 61, 80
  natively; `.mat`), EDF/EDF+/BDF, CSV/TSV and NPY/NPZ, returning
  `SignalData` (now with `metadata`, `select`, `window`); `record_from_file`;
  case-insensitive lead names (`normalize_lead_name`); `lead_names="standard_12"`;
  `ecg_record`/`ecg_prepare` accept `SignalData`.
- **Long recordings**: `ecg_record_segments` with `/metadata/segment`.
- **Examples**: `load_example`, `list_examples`, `ecgfeat.datasets.describe_example`
  -- three human-validated PTB-XL records (CC BY 4.0, see NOTICE).
- **Record conveniences**: `ECGRecord.get`, `query`, `global_values`,
  `summary`, `to_rows`, `to_csv`, `to_dataframe("long"|"global"|"leads"|"beats")`,
  a readable `repr` and `_repr_html_`; group-qualified names
  (`"global/qt_interval_ms"`) and selector-based disambiguation.
- **Plots**: `plot_ecg` (ECG paper in real millimetres, 3x4/6x2/12x1 + rhythm
  strips, calibration, measurement header), `plot_median_beats`,
  `plot_st_segments`, `plot_rhythm`, a noise heat map in
  `plot_quality_summary`, `PlotStyle` with light/dark themes (`set_style`,
  `use_style`, `style=`), `annotations_to_dict`, `plot_record_interactive`
  (Plotly, `interactive` extra).
- **CLI**: `ecg-record summary`, `table` (CSV/Parquet), `plot`, `examples`;
  `measure` reads any supported file and takes rate/leads from it, `--start`/`--end`;
  `batch` accepts directories and globs, runs worker processes and writes
  `--table`.
- Extras `table` (pandas, pyarrow), `interactive` (Plotly), `all`; Python 3.14.
- Release tools: `tools/check_record_crosswalk.py` (live crosswalk over the
  sentinel corpus), `tools/build_gallery.py`, `tools/make_examples.py`.

### Changed
- Summary records are up to **30,000 bytes** for the 10 s / 12-lead / 10-beat
  reference case (28,855 bytes; 24,000 in schema 1.0) and now include
  `qtc_bazett_ms`.
- `plot_record` and `plot_beat_all_leads` use one shared millivolt span by
  default (`y_scale="independent"` for the 0.1 look); markers shrink on long
  windows; fiducials use three hues (P, QRS, T) with marker shape per landmark
  (validated for colour-vision deficiency).
- `ecg-record batch --jobs N` uses processes (`--threads` restores threads).
- Internal readers share one cached document per record (faster queries,
  tables and plots).

### Deprecated
- `ExtractionConfig` (use `ECGConfig`), `extract_record` and
  `extract_ecg_record` (use `ecg_record`): they warn with
  `ECGDeprecationWarning` and are removed no earlier than 0.4.0.

### Fixed
- Schema 1.0 resources are again byte-identical to 0.1.0: the per-cell
  feature states and presence flags added since were emitted under
  `schema_version` 1.0.0 and now belong to 1.1.0.

### Known limitations
- Per-lead fiducials do not meet the CSE delineation tolerances on LUDB and
  stay `unvalidated`; P-wave boundaries are the weakest endpoint.
- `ecginterpret.interpret_record` still needs the raw signal (the rule
  engines read engine state beyond schema 1.1); use `ecginterpret.analyze` to
  measure and interpret in one pass.

## [0.1.0] - 2026-09-24

First release of the redesigned library (previously the unpublished
`ecgfeat-dxl-inspired` 0.1.0 working tree).

### Added
- **ECG Record** (schema `1.0.0`): `ecg_record`, `ecg_prepare` / `ecg_measure` /
  `ecg_emit`, `load_record`, `validate_record`, `dumps_record`, `query_measurement`,
  `query_many`, `select_measurements`, `resolve_address`, `iter_leads`, `iter_beats`,
  profiles `summary` / `all` / `debug`, canonical addresses
  `ecg-record:<record_id>@<schema_version>#<RFC 6901 pointer>`, and three explicit
  absence states (plain null, `unmeasurable(reason)`, `not_applicable(reason)`,
  with a lossless per-field default state).
- Packaged JSON Schema and validation-evidence registry
  (`ecgfeat/schemas/ecg-record/1.0/`); strict reading rejects validation claims
  above the registry. Every published field is `unvalidated` in this release.
- Verified NPZ sidecar for dense matrices (`serialize_record(..., sidecar_uri=...)`,
  CLI `measure --sidecar`), deterministic bytes, SHA-256 and identity bound.
- `ecg-record` command line: `measure`, `validate`, `query`, `resolve`, `select`, `batch`.
- Record-based plotting in `ecgfeat.viz`: `plot_record`, `plot_beat`,
  `plot_beat_all_leads`, `plot_representative_beat`, `plot_quality_summary`, all
  taking `(signal, record)` and returning `(Figure, axes)` through the
  object-oriented Matplotlib API; the signal is checked against the record's
  raw-signal SHA-256. `import ecgfeat` never imports Matplotlib.
- Extras: `viz` (Matplotlib, for `ecgfeat.viz`), `interpret` (installs `ecginterpret`),
  `performance` (Numba), `wfdb`.

### Changed
- Extraction runs as an explicit staged pipeline (input, quality, ventricular,
  beats, delineation, atrial, measurement, finalize) with pacing, QT,
  applicability and lead-integrity policy objects. Measurements are unchanged:
  legacy output is byte-identical to the pre-migration baseline on 1,063 golden
  records from seven public datasets.
- Engine modules are private (`ecgfeat._engine`); interpretation moved to the
  `ecginterpret` distribution; the legacy `ECGFeatures` plots moved to
  `ecgfeat.viz.legacy`.
- Published records never contain an impossible fiducial order or a negative
  interval: such cells are `unmeasurable(fiducial_order_violation)` /
  `unmeasurable(negative_interval)` instead of failing the whole record.

### Deprecated
- `ecgfeat.ECGFeatureExtractor` / `ecgfeat.api`, `ecgfeat.to_dict` and the
  `ecgfeat.export` entry points, top-level `interpret`, `plot_*`,
  `ecgfeat.visualize`, `load_wfdb_mat`, `parse_wfdb_header`, and the old
  private module paths.
  They warn on use and are removed no earlier than 0.3.0; `ecgfeat.compat`
  is the explicit legacy contract meanwhile.

### Known limitations
- No published field has an approved evidence manifest yet (all `unvalidated`).
- P-wave delineation is the weakest endpoint in the repository benchmarks.
- ECG Record 1.0 publishes a compact subset of the engine's measurements;
  `ecginterpret.interpret_record` therefore needs the raw signal.
