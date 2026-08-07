# Pipeline Stage 1: Pre-processing

> **Lite scope:** `f77_Lite` freezes `include_FLAT=0` and `include_Mask=2`; alternate Full branches are removed. `CCD_split` and F6 numerical parameters remain configurable.

## Purpose

Prepare raw CCD exposures for downstream analysis: estimate and remove sky background, compute a per-amplifier noise plane via the **F6 mode-bar estimator**, apply DQ masks, detect and mask image defects, match Gaia reference stars, and produce a noise-normalized image.

This stage is controlled by `pre_process.f` and the configuration file `sig_para.inc`. The noise estimator was substantially rewritten from the legacy linear model to the current F6 mode-bar noise-plane estimator.

## Configuration Files

### `para.inc` (master parameters)

| Parameter | Value | Meaning |
|:---|:---:|:---|
| `npx`, `npy` | 3000, 5000 | Max image dimensions |
| `CCD_split` | 2 | Split each chip into 2 amplifiers (at `nx/2`) |
| `include_FLAT` | 0 | Flat-field correction (0=off, 1=on) |
| `include_Mask` | 2 | DQ mask mode (0=off, 1=flat, 2=DQ, 3=both) |
| `blocksize` | 200 | Background block side length (pixels) |
| `nct`, `ncx` | 12, 3 | 2D polynomial terms and x-order for background |
| `saturation_thresh` | 25000 | Saturated pixel threshold |
| `source_thresh` | 2.0 | Defect detection threshold (σ) |
| `area_max` | `ns*ns` | Max defect area before masking |
| `area_thresh` | 6 | Min area for defect merging |

### `sig_para.inc` (F6 noise estimator)

Controls the per-amplifier noise-plane estimator. Key parameters:

| Parameter | Value | Meaning |
|:---|:---:|:---|
| `sig_blocksize` | 200 | Block side length for median seeding |
| `sig_max_blocks` | 2048 | Max blocks per amplifier |
| `sig_min_block_pixels` | 1000 | Min valid pixels per block |
| `sig_min_block_triples` | 1000 | Min triples per block |
| `sig_min_blocks` | 4 | Min valid blocks required |
| `sig_hist_nbin` | 256 | Histogram bins for mode finding |
| `sig_hist_range` | 6.0 | Histogram range in σ units |
| `sig_min_mode_count` | 500 | Min counts at mode bin |
| `sig_lower_quantile` | 0.3173105 | Lower-half quantile (= Q at center−1σ) |
| `sig_clip_k` | 3.0 | Symmetric clip half-width (in σ) |
| `sig_rdil` | 2 | Mask dilation radius (pixels) |
| `sig_clip_niter` | 2 | Mask/fit passes (global + local refinement) |
| `sig_min_fit_triples` | 1000 | Min surviving triples for final fit |
| `sig_min_fit_frac` | 0.20 | Min surviving fraction of possible triples |
| `sig_median_ratio` | 1.2678405 | Expected median(pix)/σ² for χ²₁ mixture |
| `sig_plane_min` | 1e−8 | Min valid plane value |
| `sig_max_plane_ratio` | 4.0 | Max corner ratio (permits 2× σ change across amp) |
| `sig_scale` | 1.027786 | Publication scale (C=1 active convention: honest σ) |

## Processing Steps

### 1. Image Read and Initialization (`chip_pre_process`)

- Read the FITS image and its WCS header (CRPIX, CD, CRVAL).
- Initialize the weight map: saturated pixels (`> saturation_thresh`) are masked.
- If `include_FLAT = 1`, apply flat-field correction (multiply by flat, mask invalid flat pixels).

### 2. Background Subtraction (`set_background`)

Performed **per amplifier** (when `CCD_split = 2`, the chip is split at `nxc = nx/2`).

**Coarse flattening** (`flatten_chip`, order 4, step 2): A low-order 2D polynomial is fitted to remove large-scale gradients.

**Fine background** (`set_background`):
1. Divide the amplifier into blocks of `blocksize = 200` pixels.
2. Randomly sample `npp = 1000` pixels per block and compute the block median.
3. Fit a 2D polynomial (`nct = 12` terms, `ncx = 3` x-order) to the block-median map with iterative outlier rejection (3σ clipping).
4. Subtract the fitted polynomial from every pixel.

> **Numerical fix F1**: Each sub-region is affinely mapped onto [−1, 1]×[−1, 1] before fitting. The legacy code used absolute pixel indices, causing the second amplifier's design matrix to have condition number ~2×10⁹ (nearly collinear columns). The affine normalization preserves the fitted surface in exact arithmetic while dropping the condition number to ~530.

### 3. F6 Mode-Bar Noise-Plane Estimation (`set_sig`)

This is the core noise estimation algorithm, replacing the legacy simple linear model. It operates **per amplifier** and proceeds in several phases:

#### Phase A: Block-Median Seeding

1. Partition the amplifier into blocks of `sig_blocksize = 200` pixels.
2. For each block: collect valid pixels, sort, compute the median.
3. For each block: compute pixel triples `pix = 0.5×((I(i,j)−I(i+1,j))² + (I(i,j)−I(i,j+1))²)` for all valid triples, sort, compute the triple median.
4. Require at least `sig_min_blocks = 4` valid blocks.

#### Phase B: Brightness Mode Finding

1. Build a histogram (`sig_hist_nbin = 256` bins, range `±sig_hist_range × σ_seed` around the block-median median) of all valid pixel values.
2. Smooth 5 adjacent bins with weights [1, 2, 3, 2, 1] before selecting the maximum.
3. Apply parabolic interpolation around the peak bin for sub-bin mode center.
4. Recenter a second histogram so the lower half ends at the mode.

#### Phase C: Noise Width (σ₀) Estimation

1. Count pixels below the mode (lower half).
2. Find the quantile `Q(sig_lower_quantile = 0.3173105)` of the lower half — this corresponds to center − 1σ.
3. `σ₀ = mode − Q(0.3173105)`.

#### Phase D: Private Symmetric Mask (`build_sig_private_mask`)

1. **Pass 1 (global)**: Clip pixels where `|I − mode| > sig_clip_k × σ₀` (= 3σ). Dilate masked pixels by `sig_rdil = 2` pixels.
2. **Pass 2 (local refinement)**: Scale the clip width by the prior plane: `σ_local = σ₀ × √(plane(i,j) / plane_center)`, where the plane is from Pass 1's fit. This adapts the mask to spatially varying noise.

The mask is **image-only** — it never reads or requires a DQ mask. It may read a pre-F6 base weight (e.g., flat-field validity) but never modifies it.

#### Phase E: Triple-Based Plane Fitting (`fit_sig_masked_plane`)

For all surviving triples (not masked, all three weight pixels valid):

$$\mathrm{pix}(i,j) = 0.5 \times \left[(I_{i,j} - I_{i+1,j})^2 + (I_{i,j} - I_{i,j+1})^2\right]$$

The expectation of `pix` for independent, equal-variance noise is `2σ²`. A linear plane model `pix ≈ aa + bb·x + cc·y` is fitted via:

1. Affine-normalized basis: `[1, (i−xmid)·xhalf, (j−ymid)·yhalf]` (same F1 fix).
2. Accumulate the 3×3 normal equations and 3-element RHS in double precision.
3. Jacobi scale: divide each row/column by `1/√(diagonal)` to improve conditioning.
4. Solve via checked Cholesky decomposition (`solve_sig_plane3`), rejecting singular or non-finite pivots.

This produces raw coefficients `(aa, bb, cc)` for the `2σ²` plane.

#### Phase F: Validation and Scale Conversion

1. **Double-precision validation** (`validate_sig_plane_d`): Check the plane at all 4 rectangle corners — must be finite, positive (> `sig_plane_min`), and the max/min ratio must not exceed `sig_max_plane_ratio = 4`.
2. **Scale conversion**: Multiply raw coefficients by `sig_scale = 1.027786` to convert to the published `2σ²` convention. The active convention (C=1) makes the reported σ an honest σ.
3. **Real-precision preflight** (`validate_sig_plane`): Promote to double and re-validate before image mutation.

#### Phase G: Noise Normalization (`apply_sig_plane`)

Divide each pixel by the noise standard deviation:

$$I'(i,j) = \frac{I(i,j)}{\sqrt{0.5 \times (\mathrm{aa} + \mathrm{bb} \cdot i + \mathrm{cc} \cdot j)}}$$

After normalization, the background noise has standard deviation ≈ 1.

### 4. DQ Mask Application

When `include_Mask = 2` or `include_Mask = 3`, a DQ (data quality) mask file is read from `dqmask/<prefix>_<chip>.fits`. Every nonzero DQ pixel is rejected (`weight = 0`, `normap = −1000`).

> The DQ mask is applied **after** `set_sig` to keep the noise estimator independent of DQ. This is a deliberate design choice: `set_sig` never reads, constructs, or requires a DQ mask.

### 5. Gaia Star Matching (`gen_astrometry_data`)

- If `ASTROMETRY_trivial = 1`: Use the FITS header WCS directly (no Gaia matching).
- Otherwise: Cross-match Gaia reference stars against the image to produce `_astro.dat` for Stage 2 astrometry.

### 6. Defect Detection and Masking

After noise normalization and DQ masking, several defect detection routines run:

| Subroutine | Function |
|:---|:---|
| `locate_defects` | Masks chip edges (margin=10px) and amplifier boundary; detects gradient outliers (>8σ in x/y differences of log-image); calls sub-detectors below |
| `mask_source_regions` | Masks bright source halos (threshold > 2× defect_halo_thresh) |
| `detect_stripes` | Detects natural stripes (x_smooth=100, y_smooth=200) via median/sigma of row/column profiles |
| `detect_artificial_stripes` | Detects artificial stripes using gradient statistics |
| `detect_stellar_halo` | Detects stellar halos via connected-component analysis |
| `detect_dent` | Detects dents/depressions in the image |
| `merge_defects` | Flood-fill connected components above `source_thresh`; components larger than `area_max` are masked |

All detected defects set `weight = 0` and `normap = −1000`.

### 7. Output Serialization

The noise-plane coefficients `sigabc(2,3)` (up to 2 amplifiers × 3 coefficients) are serialized into the first pixels of the output image:

- `normap(1,1) = −1` (success flag) or `+1` (error flag)
- `normap(1+i, j) = sigabc(i, j)` for `i = 1..CCD_split`, `j = 1..3`

All masked pixels are set to `−1000`.

## Output

| File | Content |
|:---|:---|
| `*_norm.fits` | Background-subtracted, noise-normalized image with serialized noise coefficients and defect mask |
| `*_astro.dat` | Gaia-matched star positions for astrometry (or trivial WCS) |
| `dqmask/*_<chip>.fits` | DQ mask (input, not output) |

## Key Design Principles

1. **Per-amplifier processing**: Each CCD is split at `nx/2` and processed independently, matching the DECam dual-amplifier readout.
2. **F6 estimator independence**: The noise estimator never reads or requires a DQ mask. It constructs its own private symmetric mask from image statistics alone.
3. **Two-pass refinement**: Global σ₀ seeds the first mask; the provisional plane refines the mask locally before the final all-survivor fit.
4. **Numerical robustness**: Affine normalization (F1 fix), Jacobi scaling, checked Cholesky, and IEEE-compliant finite checks throughout.
5. **Honest sigma convention**: The active scale (`sig_scale = 1.027786`, C=1) makes the reported noise width an honest σ rather than a historical effective scale.

## Complexity Assessment

The F6 mode-bar noise-plane estimator is a sophisticated, numerically robust replacement for the legacy linear noise model. The core innovation is using pixel triples `pix = 0.5×((ΔIx)² + (ΔIy)²)` with expectation `2σ²` to fit a spatially varying noise plane without assuming a specific noise model. The two-pass mask refinement adapts to spatially varying noise while maintaining robustness through block-median seeding and histogram-based mode finding. The defect detection suite (stripes, stellar halos, dents) provides comprehensive image quality control beyond simple threshold masking.


> **Source-of-truth note:** before a code edit or parameter change, verify the current symbol/value in the user's actual source tree. Filenames and paths in this reference describe the Legacy F77 reference layout; they are navigation hints, not required repository paths or a pinned source snapshot.
