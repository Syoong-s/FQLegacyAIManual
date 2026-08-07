# Developer Change Map for the F77 Pipeline

## Core rule

This reference tells you **where to look**. It is not a substitute for reading current source. For any requested implementation change, inspect the exact symbol and nearby callers in the user's current tree before editing.

## 1. Request → likely edit locations

| Requested change | Primary files/symbols | Also inspect |
|---|---|---|
| Stage scheduling / CLI exposure list / MPI lifecycle | `main.f`, `mpi_routines.f` | `para.inc`, downstream stage prerequisites |
| Background/F6 estimator | `pre_process.f` (`set_sig` and helpers), `sig_para.inc` | masks/normalization callers, F6 validation code if present |
| Flat/DQ mask behavior | `pre_process.f`, `para.inc` | Docker/runner path bindings |
| Camera geometry / CCD count | `cust_para.inc`, `para.inc` | all fixed-size arrays, chip loops, catalog combiners |
| Astrometric model / Gaia match | `proc_astrometry.f`, `astrometry_calib.f` | preprocessing `_astro.dat`, `npd`, WCS writers |
| Internal/external source extraction | `proc_source.f`, `ex_star.f`, `para.inc` | astrometry outputs, source-info writers |
| Deblending/connectivity | `proc_source.f` | `univ_imag_proc.f` / utilities that implement flood fill or morphology |
| FFT/power/noise subtraction | `proc_FFT_st1.f`, `proc_FFT_st2.f`, `univ_imag_proc.f`, `FFTPACK.f` | source/star FITS layout, smoothing switches |
| Star selection / local PSF | `proc_PSF.f` | `para.inc`, Stage 4 star powers, Stage 7 model evaluator |
| PCA/multi-scale PSF | Full only: `proc_psfreconsV2.f`, `00_psf_module.f`, `cust_para.inc`, PSF callers | `main.f`, LAPACK linkage, memory/free path |
| External PSF | Full: `proc_shear.f`, `para.inc` (`ext_PSF`, `PSF_PATH`) | Stage 4/5 skip behavior, runner extra bind |
| Fourier_Quad moments / deconvolution / rotation | `proc_shear.f` | PSF model, catalog index ABI, downstream calibration |
| Exposure diagnostics | `proc_info.f`, `main.f` `expo_para` reduction/write | Stage 9 quality cuts |
| Final catalog cuts/calibration/columns | `proc_combine_shear_catalog.f`, `para.inc` indices/constants | all downstream readers |
| Build/link/toolchain | `Makefile`, `f77_docker/Dockerfile` | verification scripts, runner `--check` |
| Local Docker mounts | `f77_docker/.env`, `.env.example`, `compose.yaml` | compiled `para.inc` paths |
| HPC bind/MPI launch | `f77_docker/runner/*` | SIF toolchain, cluster MPI/PMI audit |

## 2. Full/Lite parity rule

For every source-code change in a filename that exists in both `f77` and `f77_Lite`:

1. Check whether the changed block is shared or belongs to one of the eight Full-only branch selectors.
2. Search the same symbol in the other tree.
3. If the behavior is meant to stay equivalent, patch both implementations deliberately.
4. If the feature is one of Lite's deleted alternatives, keep it Full-only unless the user explicitly asks to expand Lite.
5. Compile both trees when parity is claimed.

Never assume a patch applies cleanly just because the filename and subroutine name match.

## 3. Fixed-form Fortran hazards

- Preserve fixed-form continuation/layout conventions used by the file.
- Watch implicit typing in routines without `implicit none`.
- `COMMON` blocks are an ABI: declaration order/type/shape must match in every routine using them.
- Include-file `parameter` values can define array dimensions; changing them can affect stack/static memory, binary layout, and file dimensions.
- Character lengths are fixed; path strings can silently truncate if `strl`/declarations are too short.
- Large arrays are one reason the Makefile uses `-mcmodel=medium`.
- MPI collectives require identical counts/types on all ranks; update both buffers and call counts if shapes change.

## 4. I/O contract hazards

Treat these as interfaces, not incidental files:

- exposure list consumed by `main.f`;
- Stage 1 normalized FITS / astrometry inputs;
- source/star stamp FITS and info tables;
- star/galaxy power-spectrum FITS;
- PSF coefficient/model products;
- 24-column shear records and final `*_all.cat`;
- `expo_info.dat` six-value `expo_para` record.

If a writer changes a column/order/dimension, find every reader before editing. Update headers/documentation at the same time.

## 5. Parameter change vs algorithm change

Prefer a parameter edit only when the existing parameter actually controls the requested behavior. Do not add a new knob to compensate for a logic error. Conversely, do not hard-code a numerical experiment inside a subroutine if it should be a compile-time parameter used consistently across files.

When adding a parameter:

1. Put it in the narrowest appropriate include (`sig_para.inc` for F6, `cust_para.inc` for camera/PCA, otherwise `para.inc`).
2. Use a derived expression rather than duplicating dependent constants.
3. Update Full/Lite intentionally.
4. Document default, units, valid range, and effect in `config-01-parameter-map.md` if this plugin is maintained with the source.

## 6. Verification ladder by change type

### Pure threshold/default change

- clean build;
- run earliest affected stage on a small exposure set;
- compare source/star counts and diagnostic distributions;
- run downstream stages needed to confirm final effect.

### F6/preprocessing change

- validate estimator on controlled/noise test data if available;
- inspect fitted plane validity/fallbacks and normalized image statistics;
- verify source detection/PSF changes are expected consequences, not normalization regressions.

### Source/deblend change

- compare object count, connected-component/deblend behavior, flags, stamp positions;
- ensure no array limits (`ngal_max`, `nstar_max`) are newly exceeded;
- compare Stage 6/7 population after cuts.

### PSF change

- compare selected-star counts per chip, fit chi-square, PSF size/shape diagnostics;
- verify invalid-chip logic and Stage 7 deconvolution remain finite;
- for PCA changes, verify allocation/free and LAPACK dimensions.

### Shear-estimator change

- first verify estimator identities/rotation/parity on controlled inputs;
- compare raw `g1,g2,de,h1,h2` before any Stage 9 calibration;
- only then evaluate final multiplicative/additive calibration.

### Runner/MPI change

- shell syntax check where appropriate;
- image/bind `--check`;
- staged MPI smoke tests across nodes;
- representative data run.

## 7. Debug from the first divergent intermediate

For end-to-end regressions, compare two runs stage by stage and stop at the first product that diverges unexpectedly. A final catalog difference can originate from F6 normalization, star selection, PSF, or source population long before Stage 9; modifying the final calibration until it “looks right” can hide the real fault.

## 8. When current-source inspection is mandatory

Always inspect current source (and not only this skill) for:

- exact line-level code edits;
- parameters marked “trace uses before changing”;
- output format changes;
- Full/Lite parity changes;
- MPI/COMMON-array changes;
- runner changes after the user's local/runtime layout changes;
- any discrepancy between documentation and observed behavior.
