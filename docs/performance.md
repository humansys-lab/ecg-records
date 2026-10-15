# Performance

Measured on 2026-10-14 on a 16-core x86_64 Linux host, Python 3.12, NumPy 2,
one BLAS thread per process, with the bundled 10 s, 500 Hz, 12-lead examples
(`python tools/benchmark_throughput.py --records 48 --workers 1 4 8 12`).
Numbers vary with the host and its load; the CI budget
(`benchmarks/performance/budget.json`) allows about 2.5 times these values.

## One record

| Step | 0.1.0 | 0.2.0 |
|---|---|---|
| `import ecgfeat` | — | 0.02 s (lazy; no NumPy until used) |
| first `ecg_record` in a process | — | 4.0 s |
| warm `ecg_record` (median of 5) | 2.44 s | 2.60 s |
| `query_measurement` on a record | 16.7 ms | 0.03 ms |
| `ECGRecord.to_rows()` (≈4,000 cells) | — | about 10 ms |

Extraction is about 6% slower than 0.1.0 because the engine does more (the
measurement corrections since 0.1.0, schema 1.1 fields). Profiling shows no
single hotspot: delineation and the atrial analysis take most of the time,
spread over many small NumPy calls. Reading records is much faster: queries,
tables and plots now share one cached document per record instead of copying
the whole record on every call.

## Many records

`ecg-record batch --jobs N` (and `ecg-record batch DIR`) runs N worker
processes; 0.1 used threads, which the interpreter lock serialised.

| Workers | 48 records | Records per hour |
|---|---|---|
| 1 | 128 s | 1,347 |
| 4 | 34 s | 5,067 |
| 8 | 18 s | 9,424 |
| 12 | 13 s | 13,159 |

On 24 records with 8 workers, processes took 15.6 s and threads 91.0 s.
At 12 workers the 21,799 PTB-XL records take about 1.7 hours.

## Interpretation

With the separate `ecginterpret` distribution, `ecginterpret.analyze(signal)`
measures once and returns the record and the Interpretation document: about
2.7 s per record instead of about 5.5–7 s for `ecg_record` followed by
`interpret_record`.

## Tips

- Use `profile="summary"` (default) unless you need the `all` fields; the
  JSON is about half the size.
- Install the `performance` extra (Numba) to JIT the Kalman smoother; the
  first call in each process pays the compilation.
- For long recordings, `ecg_record_segments` measures 10 s windows; run
  windows in parallel with your own process pool if needed.
