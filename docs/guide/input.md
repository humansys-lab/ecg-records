# Preparing input signals

Every measurement starts from four facts about the signal: the samples, the
sampling rate, the name of every row and the amplitude unit. `ecg-records`
never guesses any of them, so a mislabelled signal fails loudly instead of
producing plausible but wrong numbers.

## The contract

| Requirement | Rule | Error if violated (`code`) |
|---|---|---|
| Array shape | 2-D, **channel-major**: `(n_leads, n_samples)` | `SignalShapeError` (`invalid_signal_shape`, `signal_lead_count_mismatch`) |
| Values | numeric and finite (no NaN or ±inf); converted to `float64` | `SignalShapeError` (`invalid_signal_type`, `nonfinite_signal`) |
| Duration | at least 1 second | `SignalShapeError` (`record_too_short`) |
| Sampling rate | finite, at least 100 Hz; need not be an integer | `SamplingRateError` (`invalid_sampling_rate`) |
| Lead names | one non-empty string per row, unique, without `|`; surrounding whitespace is stripped; `lead_names="standard_12"` names twelve rows in the order of `STANDARD_12_LEADS` | `LeadNameError` (`invalid_lead_names`, `missing_lead_names`) |
| Standard mode | exactly the twelve canonical names, each once, **in any row order** | `LeadNameError` (`standard_lead_contract`) |
| Limited mode | 1 to 8 explicitly named channels, with `input_mode="limited"` (never inferred) | `LeadNameError` (`limited_lead_count`) |
| Amplitude unit | `"mV"` (default) or `"uV"`; case-insensitive, `µV` and `μV` accepted | `AmplitudeUnitError` (`unsupported_amplitude_unit`) |

All of these errors subclass [`ECGInputError`](errors.md), which subclasses
`ValueError`.

The canonical names are `ecgfeat.STANDARD_12_LEADS`:

```python
from ecgfeat import STANDARD_12_LEADS

print(STANDARD_12_LEADS)
# ('I', 'II', 'III', 'aVR', 'aVL', 'aVF', 'V1', 'V2', 'V3', 'V4', 'V5', 'V6')
```

Standard names are matched case-insensitively since 0.2.0: `"AVR"`, `"avr"`
and `"lead II"` become `aVR` and `II` (`ecgfeat.normalize_lead_name`). The file
readers also drop `ECG `/`EKG ` prefixes. Other names are kept as given.

```python
import numpy as np
from ecgfeat import ecg_record

signal = np.load("ecg.npy")
upper = ecg_record(signal, sampling_rate=500, lead_names=[name.upper() for name in STANDARD_12_LEADS])
short = ecg_record(signal, sampling_rate=500, lead_names="standard_12")
print(upper.axes["leads"][3], upper.record_id == short.record_id)   # aVR True
```

## Orientation: channel-major

Row `i` of the array is the lead named `lead_names[i]`. Many file formats store
samples in rows instead (`(n_samples, n_leads)`); transpose those:

```python
import numpy as np

sample_major = np.load("ecg.npy").T          # pretend we received (5000, 12)
signal = np.ascontiguousarray(sample_major.T)
print(signal.shape)                          # (12, 5000)
```

A `(5000, 12)` array passed with twelve lead names fails with
`signal_lead_count_mismatch`, because it has 5000 rows.

## Lead order does not matter

In standard mode the rows may come in any order, as long as each canonical lead
appears exactly once. The record always lists leads in the order you gave them
(`record.acquisition["leads"]`, `record.axes["leads"]`), and every per-lead
matrix uses that order.

```python
import numpy as np
from ecgfeat import ecg_record

signal = np.load("ecg.npy")
leads = ["I", "II", "III", "aVR", "aVL", "aVF", "V1", "V2", "V3", "V4", "V5", "V6"]
order = [6, 7, 8, 9, 10, 11, 0, 1, 2, 3, 4, 5]          # precordial leads first
shuffled = ecg_record(signal[order], sampling_rate=500, lead_names=[leads[i] for i in order])
print(shuffled.axes["leads"][:3])                        # ('V1', 'V2', 'V3')
```

## Amplitude units

Declare the unit your samples are in. Microvolt input is accepted and recorded
as such (`record.acquisition["amplitude_unit"]`); published amplitudes are
always in microvolts (`uV`) regardless of the input unit.

```python
import numpy as np
from ecgfeat import ecg_record, query_measurement

signal_mv = np.load("ecg.npy")
leads = ["I", "II", "III", "aVR", "aVL", "aVF", "V1", "V2", "V3", "V4", "V5", "V6"]
from_uv = ecg_record(signal_mv * 1000.0, sampling_rate=500, lead_names=leads, amplitude_unit="uV")
print(from_uv.acquisition["amplitude_unit"])                                         # uV
print(query_measurement(from_uv, "r_amplitude_uv", lead="II", beat=0).unit)          # uV
```

Digital ADC counts are not accepted directly: convert them to physical units
first (for WFDB: `(counts - baseline) / gain`).

## Sampling rate

Any rate of at least 100 Hz works. The pipeline analyses internally at the
native rate rounded to an integer, unless you set
[`ECGConfig.fs_internal`](configuration.md). Published fiducials are always
zero-based sample indices **at the native rate** of the array you passed, so
they index your signal directly.

## Fewer than 12 leads: limited mode

For 1–8 channels, pass `input_mode="limited"`. The names are free-form but must
be explicit. Nothing is synthesized: there is no derived Einthoven or Goldberger
lead and no reconstructed twelve-lead report.

```python
import numpy as np
from ecgfeat import ecg_record, query_measurement

twelve = np.load("ecg.npy")
two = twelve[[1, 6]]                                       # leads II and V1
record = ecg_record(two, sampling_rate=500, lead_names=["II", "V1"], input_mode="limited")

print(query_measurement(record, "heart_rate_bpm").value)   # measured
axis = query_measurement(record, "frontal_qrs_axis_deg")
print(axis.value, axis.absence)                            # None NotApplicableAbsence(reason='input_mode_limited', ...)
```

In limited mode, quantities that need the full lead set are published as
`not_applicable` with reason `input_mode_limited`: the frontal QRS axis and
QTc (Bazett). `record.acquisition["capabilities"]` states this for the whole
record (`axis` and `formal_qt`). Raw per-lead measurements stay available.

## Patient metadata

`patient` is optional. Pass a mapping or an `ecgfeat.PatientMeta`:

```python
import numpy as np
from ecgfeat import ecg_record

signal = np.load("ecg.npy")
leads = ["I", "II", "III", "aVR", "aVL", "aVF", "V1", "V2", "V3", "V4", "V5", "V6"]
record = ecg_record(signal, sampling_rate=500, lead_names=leads,
                    patient={"age": 54, "sex": "F", "patient_id": "subject-017"})
print(dict(record.metadata["patient"]))     # {'age': 54, 'sex': 'F', 'patient_id': 'subject-017'}
```

- Accepted keys: `age`, `age_days`, `sex`, `patient_id`, `meds`,
  `clinical_classifications`, `device`, `acquisition_time`, `amplitude_unit`,
  `gain_uv_per_lsb`, `adc_full_scale_mv`, `filter_highpass_hz`,
  `filter_lowpass_hz`, `muscle_filter_enabled`. Unknown keys raise
  `ConfigurationError`.
- Only `age`, `age_days`, `sex` and `patient_id` are copied into the record
  (`/metadata/patient`). The rest is used internally and never published.
- All patient metadata enters the `record_id`. The same signal with different
  metadata gets a different id, because metadata can change the result.
- Records are not anonymized for you: do not pass identifiers you are not
  allowed to store next to the measurements.

## Reading files

`ecgfeat.read_signal(path)` returns a `SignalData`: channel-major float64
samples in millivolts, the sampling rate and one name per row, taken from the
file. Pass it straight to `ecg_record`. Nothing is filtered or resampled.

| Suffix | Format | Notes |
|---|---|---|
| `.hea`, `.dat` | WFDB | formats 16, 212, 24, 32, 61 and 80 decoded natively; other layouts (multi-frequency, skew, multi-segment) through the optional `wfdb` package |
| `.mat` + `.hea` | PhysioNet/CinC MATLAB | the `val` array with the header's gains and baselines |
| `.edf`, `.bdf` | EDF, EDF+, BDF | annotation channels are skipped; `channels=` selects signals by label |
| `.csv`, `.tsv`, `.txt` | delimited text | header row names the leads; a first column `time`/`t` (s) or `time_ms` gives the rate; files with one row per lead are transposed |
| `.npy`, `.npz` | NumPy | `.npz` may store `signal`, `sampling_rate`, `lead_names`, `amplitude_unit`; twelve unnamed rows take the standard order |

A path without a suffix is read as a WFDB record when `<path>.hea` exists.
Explicit `sampling_rate=` and `lead_names=` win over the file. The WFDB
decoder is checked sample for sample against the `wfdb` package on LUDB,
PTB-XL, QTDB, EDB, LTSTDB, BUT-PDB and the PhysioNet 2021 MATLAB records.

```python
import ecgfeat

data = ecgfeat.read_signal("ecg.npz", sampling_rate=500)
print(data)                                  # SignalData(12 leads [...], 10 s @ 500 Hz, mV)
record = ecgfeat.ecg_record(data)            # rate, names and unit come from data
window = data.window(2.0, 8.0)               # 6 s of the recording
v_leads = data.select(["V1", "V2", "V3"])    # rows by name, in this order
```

<!-- doc-check: skip (needs files on disk) -->
```python
signal = ecgfeat.read_signal("ptbxl/records500/00000/00001_hr.hea")   # WFDB
signal = ecgfeat.read_signal("study.edf", channels=["II", "V5"])      # EDF/BDF
signal = ecgfeat.read_signal("export.csv")                            # header + time column
record = ecgfeat.record_from_file("JS00001.hea")    # read + measure; WFDB Age/Sex become patient metadata
```

`ecgfeat.io.read_wfdb` (MATLAB pairs only) remains for 0.1 callers.

### Example ECGs

Three human-validated PTB-XL 1.0.1 records ship with the package (CC BY 4.0;
cite PTB-XL when you use them): `sinus` (normal ECG), `af` (atrial
fibrillation) and `lbbb` (sinus rhythm with complete LBBB).

```python
from ecgfeat.datasets import describe_example

signal = ecgfeat.load_example("lbbb")
print(describe_example("lbbb")["source"])    # PTB-XL 1.0.1 ecg_id 618 (records500/00000/00618_hr)
```

### Long recordings

The engine measures resting recordings of about 10 s. Measure a window, or
a sequence of windows with `ecg_record_segments`; each record carries
`/metadata/segment` (`index`, `start_sample`, `stop_sample`, ...), so a
sample index maps back to the source as `start_sample + sample`.

```python
import numpy as np

long = np.concatenate([np.load("ecg.npy")] * 3, axis=1)          # 30 s
records = list(ecgfeat.ecg_record_segments(long, sampling_rate=500, lead_names="standard_12"))
print([r.as_dict()["metadata"]["segment"]["start_sample"] for r in records])   # [0, 5000, 10000]
```

## Validating input without measuring

`ecg_prepare` runs exactly the checks above and returns an immutable
`ECGInput` (a private copy of the samples). Use it to reject bad files early,
or as the first step of the [staged API](measuring.md#staged-api).

```python
import numpy as np
from ecgfeat import ecg_prepare, ECGInputError

try:
    ecg_prepare(np.zeros((12, 200)), sampling_rate=500, lead_names=["I"] * 12)
except ECGInputError as exc:
    print(exc.to_dict())   # {'code': 'invalid_lead_names', 'message': ..., 'details': {}}
```
