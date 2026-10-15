# ecg-records

**ecg-records** measures 12-lead (or 1–8 channel) electrocardiograms and emits a
small, versioned JSON document, the **ECG Record**. It contains record-level
intervals, axes and rhythm measures, median-beat values per lead, per-beat
fiducial points, intervals and amplitudes, signal quality and full
provenance, all in raw-sample coordinates. Every value has an explicit unit,
every missing value carries an explicit reason, every value has a stable
address, and every field carries a validation status. It reads WFDB, EDF/BDF,
CSV and NumPy files and draws clinical-format plots.

The Python import name is `ecgfeat`.

> This software is research and engineering software. It is not a medical
> device and is not intended to diagnose, treat, cure, or prevent disease.
> Outputs require independent validation for the intended use and must not be
> used as a substitute for professional medical judgment.
> See [Limitations and intended use](limitations.md).

```bash
pip install ecg-records            # NumPy + SciPy only
pip install "ecg-records[viz]"     # adds Matplotlib for ecgfeat.viz
pip install "ecg-records[all]"     # plots, pandas tables, Plotly, ecginterpret
```

```python
import ecgfeat

signal = ecgfeat.load_example("lbbb")           # bundled PTB-XL record; or ecgfeat.read_signal("x.hea")
record = ecgfeat.ecg_record(signal)
print(record.summary())                         # rate, P/PR/QRS/QT, QTc, axes
print(record.get("qrs_duration_ms"))            # record level: 134 (ms)
print(record.get("median_r_amplitude_uv", lead="V6"))   # median beat of V6: 777 (µV)

qrs = record.query("qrs_duration_ms", lead="V1", beat=0)  # one beat, one lead
print(qrs.value, qrs.unit, qrs.validation.tier)
```

![12-lead ECG paper view of the bundled LBBB example](images/paper_lbbb.png)

## What you get

- **One call from signal to record.** [`ecg_record`](guide/measuring.md) runs
  signal-quality assessment, multilead QRS detection, beat grouping,
  representative beats, delineation, atrial analysis and measurement. The
  result is a validated, immutable record.
- **Files in, records out.** [`read_signal`](guide/input.md#reading-files)
  reads WFDB, EDF/BDF, CSV and NumPy files with their sampling rate and lead
  names; three example ECGs ship with the package.
- **A published, versioned format.** The [ECG Record](guide/record-format.md)
  (schema `1.1.0`) ships with a JSON Schema. Records are canonical JSON: the
  same input always gives the same bytes. The default `summary` profile of a
  10 s, 12-lead record stays under 30,000 bytes.
- **Three levels of measurement**: record level (`qt_interval_ms`,
  `qtc_fridericia_ms`, `p_axis_deg`, ...), median beat per lead
  (`median_st_60ms_uv`, ...) and per beat and lead. Tables in one call:
  `record.to_dataframe("global")`.
- **Honest missing values.** A value is either measured, plain `null`,
  `unmeasurable(reason)` or `not_applicable(reason)`. See
  [Absence states](guide/record-format.md#absence-states).
- **Stable addresses.** Every value has an address such as
  `ecg-record:<id>@1.1.0#/measurements/intervals/qrs_duration_ms/values/3/6`
  that you can cite and resolve later ([Querying](guide/querying.md)).
- **Provenance and validation status on every field**, so no value looks more
  trustworthy than it is. The record-level P duration, PR, QRS and QT are
  benchmark-validated against IEC 60601-2-25 limits ([Validation](validation.md)).
- **Plotting** of `(signal, record)` pairs with [`ecgfeat.viz`](guide/plotting.md):
  ECG paper at true scale, median beats, ST and rhythm views, light and dark
  themes. Matplotlib stays optional: `import ecgfeat` never loads it.
- **A command line** for measuring files, summaries, tables, plots and
  process-parallel batch runs ([CLI](guide/cli.md)).

## Where to go next

| If you want to… | Read |
|---|---|
| install the package | [Installation](getting-started/installation.md) |
| measure your first ECG in five minutes | [Quickstart](getting-started/quickstart.md) |
| know what the signal must look like | [Preparing input signals](guide/input.md) |
| understand every member of a record | [The ECG Record](guide/record-format.md) |
| read values, iterate beats and leads, resolve addresses | [Reading and querying records](guide/querying.md) |
| store records, use NPZ sidecars | [Saving and loading](guide/serialization.md) |
| draw waveforms with annotations | [Plotting](guide/plotting.md) |
| process many files from a shell | [Command line](guide/cli.md) |
| know how fast it is | [Performance](performance.md) |
| tune the pipeline | [Configuration](guide/configuration.md) |
| handle errors programmatically | [Errors and warnings](guide/errors.md) |
| move off the legacy `ECGFeatureExtractor` API | [Migration guide](migration.md) |
| look up a signature | [API reference](reference/api-ecgfeat.md) |
| work on the library itself | [Developer guide](development/architecture.md) |

## Package facts

| | |
|---|---|
| Distribution | `ecg-records` on PyPI |
| Import name | `ecgfeat` |
| Current release | 0.2.0 |
| ECG Record schema | 1.1.0 (reads every `1.*`) |
| Python | 3.10 – 3.14 |
| Runtime dependencies | NumPy ≥ 1.26, SciPy ≥ 1.11 |
| Optional extras | `viz` (Matplotlib), `table` (pandas, pyarrow), `interactive` (Plotly), `performance` (Numba), `wfdb` (exotic WFDB layouts), `interpret` (the separate [`ecginterpret`](https://pypi.org/project/ecginterpret/) distribution), `all` |
| License | `ecg-records` Apache-2.0; `ecg-records-engine` PolyForm Noncommercial 1.0.0 or commercial ([Licensing](licensing.md)) |
