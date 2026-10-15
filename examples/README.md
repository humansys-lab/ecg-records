# Examples

`data/` holds the synthetic 10 s, 500 Hz, 12-lead reference signal of the
documentation (`signal.npz`, key `signal`, shape 12 × 5000, mV) and the ECG
Records measured from it (`record.summary.json`, `record.all.json`), with the
`manifest.json` that pins them. Every Python example on the documentation
site runs against this signal (`tools/check_doc_examples.py`).

```python
import numpy as np
import ecgfeat
from ecgfeat import viz

signal = np.load("examples/data/signal.npz")["signal"]
record = ecgfeat.ecg_record(signal, sampling_rate=500)      # needs ecg-records[engine]
print(record.summary())
figure, axes = viz.plot_clinical(signal, record)

saved = ecgfeat.load_record("examples/data/record.all.json")  # reading needs no engine
print(saved.get("qt_interval_ms"))
```

More examples: the [Quickstart](https://humansys-lab.github.io/ecg-records/getting-started/quickstart/)
and the user guide on the documentation site.
