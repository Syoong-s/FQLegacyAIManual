# Auxiliary Measurement / PDF-Symmetry References — Scope Boundary

The `measurement-01` through `measurement-06` references describe a separate field-distortion/PDF-symmetry calibration workflow that consumes pipeline catalogs. They are preserved because they can be useful for interpreting calibration, but **the corresponding Fortran/Python source was not part of the reference material used to build this manual**.

Therefore:

- Do not mix this program's stages with the main F77 pipeline's 9 stages.
- Do not claim that files such as `control.py`, its `para.inc`, or an implementation named `process_fd` exist in the user's current project unless the user's source proves it.
- For conceptual questions, the measurement references can be used as documented methodology.
- For code modifications, exact commands, or parameter changes in this auxiliary program, require/inspect its actual source before editing; the references are not sufficient source truth.

Routing inside this auxiliary section:

| Task | Reference |
|---|---|
| Architecture/execution | `measurement-01-overview.md` |
| Catalog loading and field-distortion spatial bins | `measurement-02-spatial-binning.md` |
| Equal-probability inner bins | `measurement-03-equal-prob-binning.md` |
| PDF sign-test chi-square | `measurement-04-chi2-sign-test.md` |
| Minimization / quadratic fit | `measurement-05-minimization.md` |
| Multiplicative/additive calibration | `measurement-06-calibration.md` |
