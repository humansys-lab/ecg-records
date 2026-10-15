# Command line

Installing the package provides the `ecg-record` command. Every subcommand
reads and writes the same records as the Python API.

```text
ecg-record {measure,summary,table,plot,examples,validate,query,resolve,select,batch} ...
```

Wherever a file argument accepts `-`, the command reads from standard input or
writes to standard output. Results go to standard output as JSON. Errors go to
standard error as one JSON line, `{"error": "<ExceptionType>", "message": "..."}`,
and set a non-zero [exit code](#exit-codes).

## measure

Measure a signal file and write a record.

```bash
ecg-record measure ptbxl/records500/00000/00001_hr.hea -o rec.json   # rate and leads from the file
ecg-record measure ecg.npy --sampling-rate 500 --lead-names standard_12 -o ecg.record.json
ecg-record measure study.edf --start 60 --end 70 -o window.json      # a 10 s window
```

| Option | Default | Meaning |
|---|---|---|
| `signal` (positional) | | any file [`read_signal`](input.md#reading-files) reads (WFDB `.hea`/`.dat`/`.mat`, EDF/BDF, CSV/TSV, NPY/NPZ); `-` reads NPY/NPZ bytes from stdin |
| `--sampling-rate HZ` | from the file | needed only when the file does not say (NPY, CSV without a time column, stdin) |
| `--lead-names NAME ...` | from the file | one name per row, in row order, or `standard_12`; needed only when the file does not say |
| `--amplitude-unit {mV,uV}` | `mV` | unit of CSV/NPY samples |
| `--input-mode {standard_12,limited}` | `standard_12` | use `limited` for 1–8 channels |
| `--start S`, `--end S` | whole file | measure a window (seconds) |
| `--method METHOD` | `default` | `default` or `legacy_dxl_inspired` |
| `--profile {summary,all,debug}` | `summary` | record profile |
| `--config FILE` | | JSON file with [configuration](configuration.md#configuration-files) |
| `--sidecar FILE` | | write dense matrices to this NPZ sidecar; it must be in the same directory as `-o` |
| `-o, --output FILE` | `-` (stdout) | record path |

Files are written atomically (temporary file, then rename). With `--sidecar`,
the sidecar is written first, so the record never points at a missing
sidecar. The record is followed by a newline; otherwise the bytes are the same
as `dumps_record`.

```bash
# two channels, all measurements, dense matrices in a sidecar
ecg-record measure holter.npz --sampling-rate 250 --lead-names II V5 \
    --input-mode limited --profile all \
    --sidecar out/holter.dense.npz -o out/holter.record.json
```

## summary, table, plot, examples

```bash
ecg-record summary rec.json                     # or a signal file: measured on the fly
ecg-record table rec1.json rec2.json -o global.csv          # one row of record-level values per record
ecg-record table rec.json --kind long -o cells.parquet      # one row per published cell (needs pandas+pyarrow)
ecg-record plot ptbxl/00001_hr.hea --kind paper -o ecg.png  # paper | record | beat | median | rhythm | st | quality
ecg-record plot rec.json --signal ptbxl/00001_hr.hea --kind median -o median.png
ecg-record examples                             # list the bundled example ECGs
ecg-record examples lbbb -o lbbb.npz            # export one as NPZ (signal, sampling_rate, lead_names)
```

## validate

```bash
ecg-record validate ecg.record.json            # strict (default)
ecg-record validate ecg.record.json --level schema
```

Prints `{"record_id": "...", "schema_version": "1.1.0", "valid": true}`. An
invalid record exits with code 3 and the validation error on stderr.

## query

```bash
ecg-record query ecg.record.json qrs_duration_ms --lead V1 --beat 3
ecg-record query ecg.record.json heart_rate_bpm
```

Prints one JSON object with the same members as
[`MeasurementResult`](querying.md#one-value-query_measurement): `name`, `lead`,
`beat`, `value`, `unit`, `absence`, `address`, `provenance`, `validation`.

## resolve

```bash
ecg-record resolve ecg.record.json 'ecg-record:rec-…@1.1.0#/quality/record'
```

Prints `{"address": "...", "node_kind": "record", "value": {...}}`. Quote the
address in the shell because it contains `#`.

## select

Export several values in one call. The query file is a JSON array of
`{"name", "lead", "beat"}` objects, or an object with a `queries` array:

```json
[
  {"name": "heart_rate_bpm"},
  {"name": "qrs_duration_ms", "lead": "V1", "beat": 0},
  {"name": "qt_interval_ms", "lead": "V5", "beat": 0}
]
```

```bash
ecg-record select ecg.record.json --queries queries.json            # JSON to stdout
ecg-record select ecg.record.json --queries queries.json --format jsonl -o values.jsonl
```

## batch

Measure many signals from a JSON Lines manifest, a directory or a glob, in
parallel worker processes.

```bash
ecg-record batch manifest.jsonl --output-dir records --jobs 4
ecg-record batch 'ptbxl/records500/**/*.hea' --output-dir records --jobs 12 --table records/global.csv
```

A directory or glob makes one item per signal file (WFDB records by their
header), with the file name as `id`; `--table` also writes one row of
record-level values per successful record (CSV, or Parquet for `.parquet`).

Each manifest line describes one item:

```json
{"id": "p001", "signal": "signals/p001.npy", "sampling_rate": 500, "lead_names": ["I", "II", "III", "aVR", "aVL", "aVF", "V1", "V2", "V3", "V4", "V5", "V6"]}
{"id": "p002", "signal": "signals/p002.npz", "sampling_rate": 250, "lead_names": ["II", "V5"], "input_mode": "limited", "profile": "all", "output": "limited/p002.json"}
```

| Member | Required | Meaning |
|---|---|---|
| `id` | yes | non-empty string, echoed in the status line |
| `signal` | yes | any signal file `measure` accepts, relative to the manifest's directory |
| `sampling_rate`, `lead_names` | when the file does not say | as for `measure` |
| `amplitude_unit`, `input_mode`, `method`, `profile`, `start_s`, `end_s` | no | as for `measure` |
| `config` | no | configuration JSON file, relative to the manifest's directory |
| `output` | no | record path under `--output-dir`; default `<id>.json` |

Unknown members are rejected. Outputs must stay inside `--output-dir` and be
unique. Batch items cannot read stdin.

For every item, in manifest order, one status line goes to stdout. For the
manifest above plus a third item (`p003`) whose signal file does not exist:

```json
{"id": "p001", "output": "/abs/path/records/p001.json"}
{"id": "p002", "output": "/abs/path/records/limited/p002.json"}
{"error": "FileNotFoundError", "id": "p003", "message": "[Errno 2] No such file or directory: 'signals/missing.npy'"}
```

A failed item does not stop the others (unless `--fail-fast`); its error is
reported in its status line.

- `--jobs N` measures up to N items concurrently in worker processes
  (`--threads` uses threads, as 0.1 did; the interpreter lock makes them much
  slower). Statuses stay in manifest order.
- `--fail-fast` stops after the first failed item.
- The exit code is 0 when every item succeeded and 4 when any failed.

## Exit codes

| Code | Meaning |
|---|---|
| 0 | success |
| 2 | invalid configuration or arguments (`ConfigurationError`) |
| 3 | invalid input or record (`ECGInputError` and its subclasses, `ValueError`) |
| 4 | `batch`: at least one item failed |
| 5 | file system error (`OSError`) |
| 70 | internal error, including `ComputationInvariantError` |

Invalid command-line syntax (missing required options, unknown choices) is
reported by `argparse` with exit code 2.
