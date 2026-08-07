# Pipeline Stage 2: Astrometry - PU Polynomial Coordinate Transformation

> **Lite scope:** `f77_Lite` freezes `ASTROMETRY_trivial=0`; only the Gaia-based astrometry path remains.

## Purpose

Establish the mapping from pixel coordinates (x, y) to celestial coordinates (RA, Dec) for each CCD chip, accounting for optical distortion. The astrometric solution provides the field distortion shear used in Stage 7 coordinate rotation and Stage 9 calibration.

## Operating Modes

### Trivial Mode (`ASTROMETRY_trivial = 1`)

When enabled, the FITS header WCS (CRPIX, CD, CRVAL) is used directly without Gaia matching. The PU coefficients are set to zero. This mode is useful when the header WCS is already accurate.

- Calls `get_astrometry_trivial`: reads per-chip WCS from `_astro.dat` (generated in Stage 1), writes `.head` with zero PU coefficients.

### Full Gaia-Based Mode (`ASTROMETRY_trivial = 0`, **default**)

- Calls `get_astrometry`: performs full astrometric calibration via Gaia pattern matching.

## Method: PU Polynomial Distortion Model

The pixel->sky transformation uses a polynomial distortion model with `npd = 33` parameters, denoted as PU coefficients:

$$\mathrm{RA},\mathrm{Dec} = f_{PU}(x, y;\ \mathrm{CRPIX}, \mathrm{CD}, \mathrm{CRVAL}, \mathrm{PU})$$

The mapping proceeds in two steps:

1. **Pixel -> Intermediate Standard Coordinates** (`mapping_PU`): A polynomial transformation maps pixel coordinates to an intermediate tangent-plane coordinate system using the PU coefficients.
2. **Standard Coordinates -> (RA, Dec)** (`ra_dec_to_xi_eta`): The standard WCS linear transformation (CRPIX, CD matrix, CRVAL) maps intermediate coordinates to the celestial sphere.

## Coefficient Solving

The full set of per-chip CRPIX, CD, and global PU coefficients is solved by pattern-matching against a Gaia reference star catalog (`gen_astrometry_data` in Stage 1, `measure_astrometry_global` in Stage 2):

1. **Per-chip WCS initialization**: Read the FITS header CRPIX, CD, CRVAL for each chip.
2. **Gaia catalog selection** (`generate_gaia_file_name`): Select Gaia stars within the RA/Dec footprint of the exposure.
3. **Source matching** (`get_astrometry_catalog`): Detect point sources in the image and match them to Gaia stars.
4. **Pattern matching** (`pattern_matching`): Match detected sources to Gaia reference stars using a shift-search algorithm.
5. **Global solve** (`measure_astrometry_global`): Fit the global PU coefficients and per-chip CRPIX/CD simultaneously from all matched stars across all chips. Requires at least `(npd + nchip×3)×3` total matched sources.

## Output

| File | Content |
|:---|:---|
| `.head` | Per-exposure astrometry file: CRVAL, PU coefficients, per-chip (k, valid, CRPIX, CD) |
| `_astro.dat` | Gaia-matched star positions (per-chip, from Stage 1) |
| `_check.dat` | Astrometry verification: compares input Gaia (RA, Dec) with PU-mapped (RA, Dec) for each matched star |
| `_norm.fits` (updated) | WCS header parameters updated via `update_para` |

## Verification (`_check.dat`)

After solving, `chip_process_astrometry` verifies the solution by:
1. Reading each chip's astrometry parameters from `.head`.
2. Updating the `_norm.fits` header with the solved CRPIX/CD (`update_para`).
3. For each matched star: comparing the input Gaia (RA, Dec) with the PU-mapped (RA, Dec) from the pixel position.
4. Writing the comparison to `_check.dat` for quality assessment.

## Role in Downstream Stages

- Stage 3 uses the WCS to map external catalog positions to pixel coordinates (when `ext_cat = 1`).
- Stage 7 uses the PU mapping to compute field distortion shear (gf1, gf2) and coordinate rotation (cos2φ, sin2φ) at each galaxy position.
- Stage 9 uses field distortion as a proxy for "input shear" in the distortion calibration.

## Complexity Assessment

The PU polynomial model with 33 parameters is a moderately sophisticated astrometric solution. The core idea (polynomial distortion + WCS) is standard in astronomy, but the specific 33-term formulation and the Gaia-based global fitting procedure are domain-specific implementations. The pattern-matching algorithm and the simultaneous global solve across all chips are computationally non-trivial.


> **Source-of-truth note:** before a code edit or parameter change, verify the current symbol/value in the user's actual source tree. Filenames and paths in this reference describe the Legacy F77 reference layout; they are navigation hints, not required repository paths or a pinned source snapshot.
