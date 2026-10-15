# Installation

## Requirements

- Python 3.10, 3.11, 3.12, 3.13 or 3.14
- NumPy ≥ 1.26 (NumPy 1.26 and 2.x are both supported) and SciPy ≥ 1.11,
  installed automatically

`ecg-records` itself is pure Python (a `py3-none-any` wheel). The numerical
engine, `ecg-records-engine`, ships as compiled wheels for CPython 3.10–3.14 on
Linux (x86_64, aarch64), macOS (x86_64, arm64) and Windows (x86_64).

## Install from PyPI

```bash
pip install "ecg-records[engine]"
```

This installs the ECG Record read/write/query API, the `ecg-record` command
line, the plotting module and the measurement engine. Both packages depend
only on NumPy and SciPy, and their versions must match (`pip` enforces it).

`pip install ecg-records` alone installs everything except the engine: records
can be read, queried and plotted, and measuring a signal raises
`EngineNotInstalledError` with the install command. The engine is free for
noncommercial use and needs a licence key for commercial use; see
[Licensing](../licensing.md).

## Optional extras

| Extra | Command | Adds | Needed for |
|---|---|---|---|
| `engine` | `pip install "ecg-records[engine]"` | `ecg-records-engine` (compiled) | measuring signals; see [Licensing](../licensing.md) |
| `viz` | `pip install "ecg-records[viz]"` | Matplotlib ≥ 3.7 | [`ecgfeat.viz`](../guide/plotting.md) plots |
| `table` | `pip install "ecg-records[table]"` | pandas, pyarrow | `record.to_dataframe()`, Parquet output of `ecg-record table` / `batch --table` |
| `interactive` | `pip install "ecg-records[interactive]"` | Plotly | `ecgfeat.viz.plot_record_interactive` |
| `performance` | `pip install "ecg-records[performance]"` | Numba | source checkouts only: the compiled engine already contains the fast kernels |
| `wfdb` | `pip install "ecg-records[wfdb]"` | the `wfdb` package | WFDB layouts the built-in reader does not decode (multi-frequency, skewed, multi-segment); formats 16, 212, 24, 32, 61, 80 and `.mat` need nothing extra |
| `interpret` | `pip install "ecg-records[interpret]"` | the separate [`ecginterpret`](https://pypi.org/project/ecginterpret/) distribution | rule-based interpretation and reports, and the legacy `ECGFeatureExtractor` / `to_dict` path |
| `all` | `pip install "ecg-records[all]"` | `viz`, `table`, `interactive`, `interpret` | everything above except `performance` and `wfdb` |

Extras combine: `pip install "ecg-records[engine,viz]"`. The record path
(`ecg_record`, `load_record`, the queries, `ecgfeat.viz`, the CLI) never needs
`ecginterpret`.

## Verify the installation

```python
import ecgfeat

print(ecgfeat.__version__)               # 0.2.0
print(ecgfeat.STANDARD_12_LEADS)         # ('I', 'II', 'III', 'aVR', ..., 'V6')
print(ecgfeat.list_examples())           # ['af', 'lbbb', 'sinus']: bundled example ECGs
```

```bash
ecg-record --help
```

`import ecgfeat` is cheap: imports are lazy, so reading and querying records
does not load the numerical engine, and nothing in the package imports
Matplotlib until you use `ecgfeat.viz`.

## Install from source

The source repository is private (the engine's source is not published; see
[Licensing](../licensing.md)). Maintainers with access install the package in
editable mode from `feature_extraction/`, where the engine runs as Python
source (optionally accelerated by Numba):

```bash
python -m pip install -e "./feature_extraction[viz,performance]"
```

See the [contributor guide](../development/contributing.md) for the
development tools.
