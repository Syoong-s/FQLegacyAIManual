# Pipeline Stage 5: PSF Modeling

> **Lite scope:** `f77_Lite` freezes `PSF_type=1` and `PSF_Ms=0`. Very-local (`PSF_type=2`) and PCA/multi-scale reconstruction are Full-only unless Lite is deliberately expanded.

## Purpose

Select true stars from star candidates, model the PSF as a function of position across each CCD chip, and optionally apply PCA residual correction. The PSF model is used in Stage 7 for deconvolution during shear estimation.

## Processing Flow (`proc_PSF`)

1. **Read candidates** (`read_in_candidates`): Load star candidate stamps and power spectra from Stage 4.
2. **Star selection** (`star_selection`): Identify true stars via χ² clustering.
3. **Star plotting** (`plot_star_expo`, `plot_stars`): Generate diagnostic plots and statistics.
4. **Rescale factor** (`save_rescale_factor`, when `PSF_Ms = 1`): Save the PCA rescale factor.
5. **PSF fitting**: Depending on `PSF_type`:
   - `PSF_type = 1`: `make_PSF_local_fit` (spatial polynomial)
   - `PSF_type = 2`: `make_PSF_hybrid` (very-local interpolation)

## Step 1: Automatic Star Selection via χ² Clustering

Star candidates are compared pairwise in Fourier space. For each pair, the central quarter of the power spectrum is used to compute a χ² distance:

$$\chi^2 = \sum_{\text{central quarter}} \left( P_1(\mathbf{k}) - P_2(\mathbf{k}) \right)^2$$

A hierarchical clustering based on this χ² distance matrix groups similar sources. The largest cluster (most mutually similar) is identified as true stars. Quality thresholds:

- **Per-chip minimum**: If a chip has fewer than `nstar_min_local = 16` true stars, the chip is marked invalid.
- **Global minimum**: If the total candidate count is less than `nstar_min = npo × 3/2 = 96`, the selection may fail.

The star power spectra are characterized by their size and shape (ellipticity e1, e2) in Fourier space, computed via `get_power_shape` prior to clustering.

## Step 2: Spatial PSF Polynomial Fitting (`make_PSF_local_fit`, `PSF_type = 1`)

A per-chip spatial polynomial model is fitted to the selected star power spectra. This model allows evaluation of the PSF $P(\mathbf{k})$ at any position (x, y) on the chip.

- Polynomial order: `npl = 10` terms per pixel, stored as `local_coe(ns, ns, npl+1)`.
- Fit quality: `poly_ave` (mean) and `poly_std` (std) of the per-star χ².
- Output: `*_PSF_coe_local.dat` - per-chip polynomial coefficients with header (nstar, status, poly_ave, poly_std).

## Step 3: Very-Local PSF Interpolation (`make_PSF_hybrid`, `PSF_type = 2`)

Instead of a polynomial fit, the PSF at each galaxy position is interpolated from nearby star PSFs. A PSF map (`*_PSF_local.fits`) stores the star power spectra on a grid (`step_psf = 100` pixel spacing).

- `interpolate_PSF`: Interpolates from the nearest star PSFs at the galaxy position.
- `get_PSF_model_very_local`: Returns the interpolated PSF model.

## Step 4: PCA Residual Reconstruction (Optional, `PSF_Ms = 1`)

When enabled, a hierarchical PSF model augments the polynomial fit with PCA-corrected residuals:

$$P_{\text{psf}}(x, y) = P_{\text{poly}}(x, y) + \sum_{j=1}^{n_{\text{pcs}}} c_j(x, y)\, \mathrm{PC}_j$$

Steps:
1. **Rescale**: A rescale factor (`res_factor`) is applied to normalize the PCA input (`save_rescale_factor`).
2. **Residuals**: Differences between each star's power spectrum and the polynomial model are collected into a covariance matrix.
3. **Eigen-decomposition** (LAPACK `dsyevd`, double precision): Yields the principal components. The top `n_pcs = 100` PCs are retained.
4. **PC coefficient interpolation**: The PC coefficients $c_j$ are spatially interpolated across the chip using a 6th-order 2D polynomial (`npp6th = 28` terms) fitted on a 2×2 grid of sub-regions (`nblocks = 2`).

> **PCA parameters** (`cust_para.inc`): `rescale_size = 1.2`, `procs_pn = 40`, `work_pn = 10`, `n_pcs = 100`, `npp6th = 28`, `nmax_star_pchip = 1000000`.

## External PSF Mode (`ext_PSF = 1`)

When `ext_PSF = 1`, an external PSF image is read directly in Stage 7 (`proc_shear.f`) from `PSF_PATH/PSF.fits`. In this mode:
- Stage 4 (FFT-1) is skipped (no star power spectra).
- Stage 5 still runs but its output may not be used; the external PSF takes precedence in shear estimation.

## PSF Evaluation in Stage 7

Three PSF evaluation modes are available, selected by `PSF_type` and `PSF_Ms`:

| Mode | PSF_type | PSF_Ms | Method |
|:---|:---:|:---:|:---|
| Local polynomial | 1 | 0 | Evaluate the spatial polynomial at the galaxy position |
| Hierarchical (poly + PCA) | 1 | 1 | Evaluate polynomial + PCA-corrected residual |
| Very local interpolation | 2 | - | Interpolate from nearest star PSFs |
| External PSF | - | - | Read from `PSF_PATH/PSF.fits` (when `ext_PSF = 1`) |

## PSF Quality Diagnostics

A normalized χ² metric `poly_chi2` measures the quality of the polynomial PSF fit to the selected stars. The per-source PSF FWHM is estimated from the $1/e$ equivalent area of the PSF power spectrum (see pipeline-07-shear-estimation.md for the FWHM formula).

## Compilation

The Makefile links against LAPACK (`-llapack -lblas`) for the `dsyevd` eigen-decomposition used in PCA, and CFITSIO for FITS I/O. The build depends on `para.inc`, `cust_para.inc`, and `sig_para.inc`.

## Complexity Assessment

The χ² clustering approach to star selection is a moderately sophisticated automated method. The hierarchical PSF model (polynomial + PCA residual) is a more unique aspect - using PCA to capture PSF variation beyond what a smooth polynomial can represent. The `dsyevd` eigen-decomposition on the residual covariance matrix is computationally non-trivial.


> **Source-of-truth note:** before a code edit or parameter change, verify the current symbol/value in the user's actual source tree. Filenames and paths in this reference describe the Legacy F77 reference layout; they are navigation hints, not required repository paths or a pinned source snapshot.
