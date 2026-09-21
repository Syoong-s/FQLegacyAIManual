# Pipeline Stage 5: PSF Modeling

> **Lite scope:** current `f77_Lite` freezes `PSF_type=1` and `PSF_Ms=0`. Very-local (`PSF_type=2`) and PCA/multi-scale reconstruction are Full-only unless Lite is deliberately expanded.

## Purpose

Stage 5 reads star-candidate products from the earlier source/FFT stages, selects robust stars, models the spatial PSF, and writes the PSF products consumed by shear estimation.

## Current product locations

The modernized reference implementation routes products through `path_layout.inc` and the `fq_*_product_path` helpers. Important logical products include:

```text
stamps/dat_StarInfo/<exposure>_star_info_expo.dat
stamps/dat_PsfFit/<exposure>/<chip-prefix>_PSF_coe_local.dat
stamps/fits_PsfLocal/<exposure>/<chip-prefix>_PSF_local.fits
stamps/fits_PsfSrc/<exposure>_PSF_source.fits
stamps/dat_StarComp/<exposure>_star_comp_expo.dat
stamps/dat_Rescale/<exposure>_factor.dat
```

Full PCA/multi-scale processing additionally uses products such as `stamps/dat_StarXY`, `stamps/fits_PsfResi`, `stamps/dat_Pcs`, and `stamps/dat_StarCompV2` according to the current layout helpers.

Do not restore old direct `DIR_OUTPUT//'/result/...'` or flat `stamps/...` concatenation when the current source already uses the centralized path API.

## Processing flow

At a high level Stage 5:

1. reads candidate metadata and star power stamps;
2. performs star-quality/shape selection;
3. computes exposure/chip diagnostics;
4. builds the active PSF representation;
5. writes products for Stage 7 and Stage 8.

For exact selection thresholds and fitting equations, inspect the current `proc_PSF.f` before making numerical claims or changes.

## Local polynomial PSF (`PSF_type=1`)

This is the current production path and the only PSF type retained by Lite. The model fits spatial polynomial coefficients from selected stellar power spectra. The primary per-chip coefficient product is logically:

```text
stamps/dat_PsfFit/<exposure>/<chip-prefix>_PSF_coe_local.dat
```

Current orientation parameters include `npl`, `nplx`, `nstar_min_local`, and the broader PSF controls in `para.inc`.

## Very-local PSF (`PSF_type=2`, Full only)

Full may use a gridded/local interpolation path that produces a PSF map such as:

```text
stamps/fits_PsfLocal/<exposure>/<chip-prefix>_PSF_local.fits
```

Lite has no selector for this branch.

## PCA / multi-scale reconstruction (`PSF_Ms=1`, Full only)

Full contains `00_psf_module.f`, `proc_psfreconsV2.f`, and PCA-specific controls in `cust_para.inc`. The path uses residual/star-position products, constructs principal components, models their spatial coefficients, and publishes reconstruction data used later by the Full pipeline.

Current Full reference PCA controls include:

- `rescale_size=1.2`
- `procs_pn=40`
- `work_pn=10`
- `nblocks=2`
- `n_pcs=100`
- `npp6th=28`
- `nmax_star_pchip=1000000`

Lite removes this feature path rather than merely setting a runtime flag.

## External PSF (`ext_PSF=1`, Full only)

Full may select an external PSF path. When active, verify the current `proc_shear.f` and `PSF_PATH` handling instead of assuming Stage-5 products are consumed in the normal way. Lite freezes `ext_PSF=0`.

## Build dependencies

Current Full and Lite Makefiles should be treated as depending on:

- source files;
- `para.inc`;
- `cust_para.inc`;
- `sig_para.inc`;
- `path_layout.inc`.

Full also includes the source/module needed for its optional PCA path. Rebuild after modifying any compile-time include, including product-layout constants.

The reference implementation links CFITSIO and LAPACK/BLAS; Full PCA uses LAPACK eigensolver functionality. Modern GNU Fortran builds may require the legacy-interface compatibility flag documented in the deployment reference.

## Modification guidance

For PSF algorithm changes, load `development-01-change-map.md` and inspect `proc_PSF.f`. For Full PCA changes, also inspect `00_psf_module.f`, `proc_psfreconsV2.f`, and `cust_para.inc`. For any product-path change, inspect `path_layout.inc` and the centralized path helpers before touching stage-local filename logic.

> Source-of-truth note: verify current symbols, values, and call sites in the user's source tree before editing.
