# Limitations and intended use

> This software is research and engineering software. It is not a medical
> device and is not intended to diagnose, treat, cure, or prevent disease.
> Outputs require independent validation for the intended use and must not be
> used as a substitute for professional medical judgment.

Records carry `intended_use = "research_and_engineering_only"`; no record is
medically cleared, approved or clinically validated.

- In schema 1.1.0 only the record-level P duration, PR, QRS and QT are
  `benchmark_validated`, for resting 10 s, 500 Hz, 12-lead ECGs like LUDB;
  every other field is `unvalidated`. Per-lead fiducials do not meet the CSE
  delineation tolerances on LUDB ([Validation](validation.md)).
- The engine is designed for resting recordings of about 10 s. Longer
  recordings are measured window by window (`ecg_record_segments`), without
  beat continuity across windows.
- In atrial fibrillation the record can still report P-wave quantities
  (PR, P axis) found by the atrial detector; check `organized_p_fraction`, the
  feature states and the rhythm plot.
- The bundled example ECGs are PTB-XL records (CC BY 4.0); cite PTB-XL when you
  use them.
- P-wave delineation is the weakest endpoint in the repository benchmarks,
  especially at short RR intervals or when the P wave overlaps the previous T wave.
- LUDB distributes amplitude-normalized signals, so LUDB cannot validate amplitudes.
- The four-class ST morphology was never validated and is anti-correlated with
  ischemia; it is not published and must not be used as ischemia evidence.
- The package measures; it does not diagnose. Every record states `capabilities.diagnosis = not_applicable (measurement_package_boundary)`.
- Measurements come from automated delineation. They can be wrong on noisy signals, paced rhythms, arrhythmias with irregular or absent P waves, and morphologies unlike the benchmark datasets. Check `quality`, the absence states and the plots before relying on a value.
- The pipeline is inspired by publicly documented practice; it is not a
  reproduction of any proprietary product.
