# Pipeline Stages 4 & 6: Power Spectrum Computation

> **Lite scope:** `f77_Lite` freezes `ext_PSF=0`; star power processing is part of the retained production path. `gal_smooth` and `star_smooth` remain configurable.

## Purpose

Compute the 2D noise-subtracted power spectrum for each source stamp. Stage 4 processes star candidates; Stage 6 processes galaxies. Both use the same FFT and noise-subtraction algorithm, but differ in regularization and post-processing.

## Key Parameters (`para.inc`)

| Parameter | Value | Meaning |
|:---|:---:|:---|
| `ns` | 64 | Stamp / power spectrum size (pixels) |
| `star_smooth` | 2 | Smoothing mode for star power spectra |
| `gal_smooth` | 2 | Smoothing mode for galaxy power spectra |
| `ext_PSF` | 0 | External PSF mode (1=skip Stage 4) |
| `len_g` | 40 | Galaxy stamp I/O blocking factor |
| `len_s` | 15 | Star stamp I/O blocking factor |

## Processing Steps

### 1. Power Spectrum Estimation (`get_power`)

For each source stamp $S$ and its corresponding noise stamp $N$, the 2D power spectrum is computed via FFT (FFTPACK):

$$P_S(\mathbf{k}) = |\mathcal{F}\{S\}(\mathbf{k})|^2, \qquad P_N(\mathbf{k}) = |\mathcal{F}\{N\}(\mathbf{k})|^2$$

The FFT output is shifted so that DC is at the center (`ns/2+1, ns/2+1`).

**Smoothing**: Both stages apply smoothing with `smooth = 2` (stars: `star_smooth`, galaxies: `gal_smooth`), which calls `smooth_image55_hole_ln` - a 5×5 smoothing filter that preserves the central pixel and operates in **log space**. This reduces high-frequency noise in the power spectrum without distorting the DC component.

The central power value `pc = P(ns/2+1, ns/2+1)` is stored for later use.

### 2. Noise Subtraction and DC Correction (`process_powers`)

The noise power spectrum is subtracted, then a residual DC bias is estimated from the four boundary rows/columns and removed:

$$P_S^{*}(\mathbf{k}) = P_S(\mathbf{k}) - P_N(\mathbf{k})$$

$$\bar{b} = \frac{1}{4(n-2)} \sum_{\text{boundary}} P_S^{*}, \qquad P_S(\mathbf{k}) \leftarrow P_S^{*}(\mathbf{k}) - \bar{b}$$

The boundary consists of rows 1 and n, and columns 1 and n (excluding the four corners), giving `4×(n−2)` boundary pixels.

### 3. Regularization (`regularize_power`) - Stars Only

**Stage 4 (stars)**: After noise subtraction and DC correction, `regularize_power` is called with `star_smooth = 2` (≥ 1), which **normalizes by the central power value**:

$$P(\mathbf{k}) \leftarrow \frac{P(\mathbf{k})}{P(\mathbf{0})}$$

This produces a PSF power spectrum normalized to 1 at DC.

**Stage 6 (galaxies)**: `regularize_power` is **NOT called**. The galaxy power spectra undergo only noise subtraction and DC correction - no normalization is applied. The absolute scale of the galaxy power spectrum is preserved for the Fourier_Quad shear estimator.

> **Correction from previous documentation**: The galaxy power spectra are NOT normalized by the mean of the four central-neighbor pixels. This regularization is available in the `regularize_power` subroutine (when called with `star_smooth = 0`), but the current Stage 6 code does not invoke it.

### 4. Additional Stage 6 Processing (Flux and SNR)

Stage 6 (`proc_FFT_st2.f`) also computes additional source parameters during power spectrum processing:

- **Flux estimate**: `source_para(11) = √(max(pc, P(ns/2+1, ns/2+1)))` - the square root of the central power.
- **SNR estimate**: `source_para(12) = source_para(11) / source_para(4) × ns` - flux normalized by source size.

These are written back to `_source_info.dat`, extending the parameter table to include the `flux2` and `SNR_F` columns (columns 11–12) used by downstream stages.

## Stage 4 Skip Condition

When `ext_PSF = 1` (external PSF mode), Stage 4 is **skipped entirely** (`proc_FourierT_st1` returns immediately). The PSF is provided externally and no star power spectra are needed.

## Output

| Stage | Output File | Content |
|:---|:---|:---|
| 4 (FFT-1) | `*_star_can_power.fits` | Noise-subtracted, regularized (DC-normalized) power spectra of star candidates |
| 6 (FFT-2) | `*_source_p.fits` | Noise-subtracted, DC-corrected (NOT regularized) power spectra of galaxies |
| 6 (FFT-2) | `*_source_info.dat` (updated) | Extended with flux2 and SNR_F columns |

The star power spectra feed into Stage 5 (PSF modeling). The galaxy power spectra are the direct input $G(\mathbf{k})$ for Stage 7 (shear measurement).

## Complexity Assessment

FFT power spectrum computation and noise subtraction are standard techniques. The log-space 5×5 smoothing with hole preservation is a domain-specific refinement that reduces noise without distorting the central peak. The key difference from the previous documentation is that galaxy power spectra are NOT regularized - only noise-subtracted and DC-corrected. The additional flux/SNR computation in Stage 6 provides quality metrics used by later stages for source filtering.


> **Source-of-truth note:** before a code edit or parameter change, verify the current symbol/value in the user's actual source tree. Filenames and paths in this reference describe the Legacy F77 reference layout; they are navigation hints, not required repository paths or a pinned source snapshot.
