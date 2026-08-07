# Pipeline Stage 7: Fourier_Quad Shear Estimation (Core)

> **Variant note:** the core estimator is shared, but Full may select external/PCA/alternate PSF paths that Lite has removed. Inspect the active PSF path before changing deconvolution logic.

## Purpose

This is the central stage of the pipeline. In Fourier space, it performs PSF deconvolution on the galaxy power spectrum, computes the Fourier_Quad moments, and generates **5 shear estimators** - the raw signal from which weak-lensing shear is ultimately calibrated.

## Processing Flow (`proc_shear` -> `expo_shear`)

For each galaxy in each chip:

1. Read the galaxy noise-subtracted power spectrum $G(\mathbf{k})$ from Stage 6 (`_source_p.fits`).
2. Read the galaxy parameter table (`_source_info.dat`): position, size, peak, SNR, etc.
3. **Evaluate the local PSF model** $P(\mathbf{k})$ at the galaxy position:
   - `ext_PSF = 1`: Read external PSF from `PSF_PATH/PSF.fits`, compute its power spectrum.
   - `PSF_type = 1, PSF_Ms = 0`: Evaluate the spatial polynomial from `_PSF_coe_local.dat`.
   - `PSF_type = 1, PSF_Ms = 1`: Evaluate polynomial + PCA-corrected residual (using rescale factor and chip ID).
   - `PSF_type = 2`: Interpolate from `_PSF_local.fits` (very-local mode).
4. Compute PSF quality diagnostics (poly_chi2, FWHM).
5. Compute celestial coordinates (RA, Dec) and field-distortion quantities (gf1, gf2, rotation angle, parity) via PU mapping.
6. Call the Fourier_Quad estimator to obtain 5 raw estimators.
7. Rotate estimators from pixel coordinates to the celestial coordinate system.
8. Write the 24-column shear catalog row to `*_shear.dat`.

## Fourier_Quad Estimator - Detailed Derivation

### Notation

- Galaxy power spectrum: $G(\mathbf{k})$ (noise-subtracted, DC-corrected, from Stage 6 - **not regularized**)
- PSF model power spectrum: $P(\mathbf{k})$ (from Stage 5, evaluated at galaxy position)
- Stamp size: `ns = 64`, centered at `cc = 1 + ns/2 = 33`
- Wave numbers relative to stamp center: $k_x = i - cc$, $k_y = j - cc$
- $k^2 = k_x^2 + k_y^2$

### Step A: PSF Scale and Gaussian Window Width

First, find the PSF power spectrum peak $P_{\max}$ and the $1/e$ threshold. Count pixels above this threshold to get the PSF equivalent area $A$, then define the PSF characteristic radius $k_s$ and the window width scale $k_0$:

$$A = \#\{\,(i,j): P(\mathbf{k}) \ge e^{-1} P_{\max}\,\}$$

$$k_s = \sqrt{A/\pi}, \qquad k_0 = k_s \cdot \mathtt{PSFr\_ratio},\quad \mathtt{PSFr\_ratio}=0.75$$

> `PSFr_ratio = 0.75` is defined **locally** within the Fourier_Quad estimator subroutine in `proc_shear.f`, not in `para.inc`.

Internally, $k_0^{-2}$ is used directly: $k_s^{-2} \cdot (0.75)^{-2}$.

### Step B: Low-k Cutoff Radius

To suppress high-frequency noise and deconvolution divergence, a cutoff radius $r_{\text{win}}$ is computed by scanning from DC outward until the PSF power falls below $\mathrm{th} = 10^{-4} P_{\max}$:

$$r_{\text{win}} = \min_{P(\mathbf{k}) < \mathrm{th}} \sqrt{k_x^2 + k_y^2}$$

This is implemented by `get_window_min_k`, which finds the smallest radius where the PSF drops below the threshold. An alternative version (`get_window_min_k_ver2`) uses radial binning and log-space standard deviation analysis for more robust cutoff detection.

All subsequent sums are restricted to $k < r_{\text{win}}$.

### Step C: PSF Deconvolution Window

A Gaussian window normalized by the PSF power spectrum defines the deconvolution weight:

$$W(\mathbf{k}) = \frac{\exp(-k^2 / k_0^2)}{P(\mathbf{k})}$$

The deconvolved and windowed galaxy power spectrum is:

$$M(\mathbf{k}) = W(\mathbf{k})\, G(\mathbf{k})$$

This is the Fourier_Quad analog of real-space PSF deconvolution: dividing by $P(\mathbf{k})$ removes the PSF, and the Gaussian window suppresses noise at high $k$ where $P(\mathbf{k})$ is small.

### Step D: The 5 Shear Estimators

Define $f = k^2 / k_0^2$ for brevity. All sums are over $\mathbf{k}$ with $k < r_{\text{win}}$.

**Raw shear estimators** $g_1, g_2$ (corresponding to the anisotropic part of the second moments in real space):

$$g_1 = -\sum_{\mathbf{k}} M(\mathbf{k})\,(k_x^2 - k_y^2)$$

$$g_2 = -\sum_{\mathbf{k}} M(\mathbf{k})\,(2\,k_x k_y)$$

**Isotropic response** $de$ (the normalization denominator, corresponding to the trace of the second-moment matrix):

$$de = \sum_{\mathbf{k}} M(\mathbf{k})\, k^2\,(2 - f)$$

**Anisotropic correction terms** $h_1, h_2$ (carrying fourth-order anisotropy information for calibration):

$$h_1 = \sum_{\mathbf{k}} M(\mathbf{k})\, k_0^{-2}\,(k^4 - 8\,k_x^2 k_y^2)$$

$$h_2 = \sum_{\mathbf{k}} M(\mathbf{k})\, k_0^{-2}\,(4\,k_x k_y\,(k_x^2 - k_y^2))$$

### Physical Interpretation

- **$g_1, g_2$** are the shear signal - they measure the quadrupole anisotropy of the deconvolved galaxy image.
- **$de$** is the isotropic response - used as the denominator when calibrating shear (the estimator $g/de$ recovers the reduced shear).
- **$h_1, h_2$** carry fourth-order shape information - they participate in the distortion calibration (Stage 9) as higher-order correction terms.

The Fourier_Quad method avoids real-space moment-based measurements, which suffer from centroid errors and noise bias, by operating entirely in the Fourier domain where convolution becomes multiplication and noise properties are better characterized.

## PSF FWHM Estimation (`get_PSF_area`)

From the PSF model evaluated at the galaxy position, the $1/e$ equivalent area $A_p$ is measured, and the FWHM in arcseconds is:

$$r = \sqrt{A_p/\pi}, \qquad \beta = \frac{n_s}{2\pi r}, \qquad \mathrm{FWHM} = \beta \cdot 2\sqrt{2\ln 2}\cdot 0.2628$$

where `pixel_size = 0.2628` arcsec.

## Coordinate Rotation to Celestial Frame

The estimators $g_1, g_2$ (rank-2) and $h_1, h_2$ (rank-4) are initially in the pixel coordinate system. The field distortion PU mapping provides the local rotation angle $\phi$ and parity at each galaxy position.

**Rotation angle from the distortion Jacobian** (`field_distortion_PU`):

The Jacobian of the pixel->intermediate-standard-coordinate mapping is decomposed as "rotation + shear". The rotation angle $\phi$ is extracted, giving $\cos 2\phi$ and $\sin 2\phi$.

**Rank-2 rotation (for $g$):**

$$g_1^{\text{rot}} = g_1\cos 2\phi + g_2\sin 2\phi$$
$$g_2^{\text{rot}} = g_2\cos 2\phi - g_1\sin 2\phi$$

**Rank-4 rotation (for $h$):**

Define $\cos 4\phi = \cos^2 2\phi - \sin^2 2\phi$, $\sin 4\phi = 2\sin 2\phi \cos 2\phi$:

$$h_1^{\text{rot}} = h_1\cos 4\phi + h_2\sin 4\phi$$
$$h_2^{\text{rot}} = h_2\cos 4\phi - h_1\sin 4\phi$$

**Parity correction:** If `parity = -1` (coordinate axis flip), the second component signs are flipped: $g_2^{\text{rot}} \leftarrow -g_2^{\text{rot}}$, $h_2^{\text{rot}} \leftarrow -h_2^{\text{rot}}$.

**Field distortion shear** $gf_1, gf_2$ is also computed from the distortion Jacobian - this is the "known input shear" used as a proxy in Stage 9 calibration and in the measurement program's spatial binning.

## Output

`*_shear.dat` - 24 columns per galaxy (see pipeline-08-catalog.md for the complete column order), including celestial coordinates, field distortion shear, the 5 rotated estimators, rotation angle, and parity.

Header: `poly_chi2 xc yc sigma nstar imax jmax half_light_flux half_light_area flag psf_FWHM SNR_F ra dec gf1 gf2 g1 g2 de h1 h2 cos2 sin2 parity`


> **Source-of-truth note:** before a code edit or parameter change, verify the current symbol/value in the user's actual source tree. Filenames and paths in this reference describe the Legacy F77 reference layout; they are navigation hints, not required repository paths or a pinned source snapshot.
