# Full `f77` vs `f77_Lite`

## Purpose

Use this reference whenever a task may behave differently between the two Fortran trees. The Lite tree is not merely a smaller build: it freezes production choices and removes the dead branches from source.

## Shared architecture

Both variants keep the same 9-stage prime-factor schedule, executable name (`Fourier_Quad_Pipe`), MPI entry pattern, core stage files, Fourier_Quad estimator, F6 estimator, catalog layout, and most numerical parameters.

## Eight frozen selectors

In Full `f77/para.inc`, these are editable compile-time parameters. In `f77_Lite`, they are **absent as selectors** and the listed behavior is compiled directly into the remaining code.

| Full selector | Lite behavior | Consequence for edits |
|---|---:|---|
| `ASTROMETRY_trivial` | `0` | Gaia-based astrometry only; no trivial branch |
| `include_FLAT` | `0` | No super-flat multiplication |
| `include_Mask` | `2` | Per-chip DQ mask mode only |
| `ext_cat` | `1` | External source catalog only |
| `ext_PSF` | `0` | PSF measured from frame stars only |
| `deblending` | `1` | Deblending always applied |
| `PSF_type` | `1` | Local polynomial PSF fit only |
| `PSF_Ms` | `0` | No multi-scale/PCA PSF reconstruction |

Still selectable in Lite include `PROCESS_stage`, `CCD_split`, `gal_smooth`, and `star_smooth`, plus the ordinary numerical thresholds/sizes that remain in `para.inc`.

## File-level differences

- Full contains `00_psf_module.f` and the PCA/multi-scale reconstruction path (`proc_psfreconsV2.f`, module storage, `free_psf_memory`, PCA parameters in `cust_para.inc`).
- Lite omits `00_psf_module.f` and removes PCA-only parameters from `cust_para.inc`.
- Full `main.f` conditionally calls `chip_psf_recons` and `free_psf_memory` when `PSF_Ms=1`; Lite `main.f` has no such calls.
- Many corresponding files have branch deletions in Lite even when filenames match. Never copy a Full patch mechanically into Lite without checking the Lite symbol context.

## Decision rule

Use **Full** if the requested behavior needs any frozen alternative (internal detection, external PSF, trivial astrometry, flat branch, alternate mask mode, PSF_type=2, PCA PSF, or deblending off). Use **Lite** when the production choices above are acceptable and the goal is a smaller, less branch-heavy codebase.

## Porting a change safely

1. Identify whether the edited logic lies inside/depends on one of the eight frozen branches.
2. Inspect the exact symbol in both trees.
3. If logic is shared, patch both implementations if parity is required.
4. If logic is Full-only, do not re-introduce the deleted selector into Lite unless the user explicitly wants to expand Lite's scope.
5. Build each edited tree separately; matching filenames do not guarantee matching source context.
