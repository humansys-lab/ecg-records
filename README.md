# ecg-records: documentation, schemas and examples

Public home of the **documentation site**, the **ECG Record JSON schemas** with
their validation registries, the **changelog** and the **examples** of
[`ecg-records`](https://pypi.org/project/ecg-records/), the Python library
that measures 12-lead ECGs into versioned, queryable ECG Records.

- Documentation: <https://humansys-lab.github.io/ecg-records/>
- Install: `pip install "ecg-records[engine]"` (Python 3.10–3.14; Linux, macOS, Windows)
- Issues and questions: this repository's issue tracker

> This software is research and engineering software. It is not a medical
> device and is not intended to diagnose, treat, cure, or prevent disease.
> Outputs require independent validation for the intended use and must not be
> used as a substitute for professional medical judgment.

## Packages and licences

| Package | Contents | Licence |
|---|---|---|
| `ecg-records` | ECG Record format, readers, queries, plots, CLI | Apache-2.0 |
| `ecg-records-engine` | the compiled measurement engine | PolyForm Noncommercial 1.0.0, or a commercial licence |

The engine's source code is not published; see
[Licensing](https://humansys-lab.github.io/ecg-records/licensing/). Everything
in this repository (documentation, schemas, registries, examples) is
Apache-2.0; the example signal is synthetic.

## Layout

| Path | What |
|---|---|
| `docs/` | the MkDocs site (`mkdocs serve` to preview) |
| `schemas/ecg-record/<family>/` | JSON Schema, validation-evidence registry and legacy crosswalk per schema family |
| `examples/` | the reference signal and records every documentation example runs on |
| `CHANGELOG.md` | package changelog (mirrors the released `ecg-records`) |
| `tools/check_doc_examples.py` | runs every Python block of the site against the installed package |

## Building the site

```bash
python -m pip install "mkdocs>=1.6,<2" "ecg-records[engine,viz]"
python tools/check_doc_examples.py     # every example runs
mkdocs build --strict
```

The `docs` workflow deploys `main` to GitHub Pages.
