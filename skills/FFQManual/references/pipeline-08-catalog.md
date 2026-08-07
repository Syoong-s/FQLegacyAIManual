# Pipeline Stages 8 & 9: Catalog Assembly & Calibration

## Stage 8: Information Aggregation

### Purpose

Aggregate per-chip diagnostics to the exposure level for quality control and downstream filtering.

### Processing (`get_expo_info`)

1. Read per-chip star info from `_star_info_expo.dat` (columns: ichip, nstar, FWHM, e1, e2, chi_d).
2. Read the `.head` astrometry file for WCS reference values (CRVAL1, CRVAL2).
3. For each chip with `nstar > 0`: accumulate FWHM, chi_d (χ²), nstar; record CRVAL.
4. Compute averages over valid chips: FWHM_AVE, chi_d_AVE, nstar_AVE.
5. Write per-chip diagnostics to `_expo_info.dat` (columns: ichip, nstar, FWHM, e1, e2, chi_d, cRPIX, cD).

After all chips are processed, `MPI_AllReduce` (sum) combines the per-node results across all MPI ranks. Rank 0 writes `expo_info.dat` with the header: `N-valid-chip PSF-FWHM(arcsec) chi_d-stars nstar-per-chip cRVAL1 cRVAL2 expo_name`.

The exposure-level 6 parameters stored in `expo_para`:
1. `nvalid` - number of valid chips
2. `FWHM_AVE` - mean PSF FWHM (arcsec)
3. `chi_d_AVE` - mean star PSF χ²
4. `nstar_AVE` - mean star count per chip
5. `cRVAL1` - WCS reference RA
6. `cRVAL2` - WCS reference Dec

The exposure-level PSF χ² value (`chi_d_AVE`, stored as `expo_para(3)`) is used as the primary quality gate in Stage 9.

## Stage 9: Catalog Merge & Distortion Calibration

### Quality Cuts

- **Exposure-level**: If the exposure PSF χ² > `chi2_thresh = 0.01`, the entire exposure is discarded (no sources written).
- **Per-source**: Sources whose peak falls outside the stamp boundary (`imax ≥ ns` or `jmax ≥ ns`), or with `poly_chi2 < −900` (invalid PSF model placeholder), are dropped.

### Distortion Shear Calibration

The calibration formula differs by operating mode:

#### External Catalog Mode (`ext_cat = 1`, **default**)

The calibration is an **identity transform** - both additive bias terms are set to zero:

$$g_1^{\text{calib}} = g_1^{\text{rot}} - 0 \cdot de + 0 \cdot h_1 + 0 \cdot h_2 = g_1^{\text{rot}}$$

$$g_2^{\text{calib}} = g_2^{\text{rot}} - 0 \cdot de + 0 \cdot h_2 - 0 \cdot h_1 = g_2^{\text{rot}}$$

Full calibration is deferred to the external measurement program. The field distortion shear (gf1, gf2) is still recorded for later use.

#### Internal Catalog Mode (`ext_cat = 0`)

The calibration uses field distortion as the "known input shear" plus constant additive bias:

$$g_{1c} = gf_1 + g_{1c}^{\text{const}}, \qquad g_{2c} = gf_2 + g_{2c}^{\text{const}}$$

where $g_{1c}^{\text{const}} = g1\_c = -0.001$ and $g_{2c}^{\text{const}} = g2\_c = -0.0003$ (g-band defaults).

$$g_1^{\text{calib}} = g_1^{\text{rot}} - g_{1c} \cdot de + g_{1c} \cdot h_1 + g_{2c} \cdot h_2$$

$$g_2^{\text{calib}} = g_2^{\text{rot}} - g_{2c} \cdot de + g_{1c} \cdot h_2 - g_{2c} \cdot h_1$$

### Output Format

#### `ext_cat = 1` (external catalog)

Each row = external catalog original fields (`_orig.cat` content) + `CCD_NUM` + 24 shear columns + exposure χ².

Header: `<ext_cat_header> CCD_NUM <shear_header> Chi2`

#### `ext_cat = 0` (internal catalog)

Each row = `CCD_NUM` + 24 shear columns.

Header: `CCD_NUM <shear_header>`

### `*_shear.dat` / `*_all.cat` Column Order (24 columns)

| # | Field | Description |
|:---:|:---|:---|
| 1 | poly_chi2 | Normalized PSF fit χ² |
| 2 | xc | Source x pixel coordinate |
| 3 | yc | Source y pixel coordinate |
| 4 | sigma | Source size |
| 5 | nstar | Number of true stars in chip |
| 6–7 | imax, jmax | Peak pixel row, column |
| 8 | half_light_flux | Half-light flux |
| 9 | half_light_area | Half-light area |
| 10 | flag | Status flag |
| 11 | psf_FWHM | Local PSF FWHM (arcsec) |
| 12 | SNR_F | Signal-to-noise ratio |
| 13 | ra | Right ascension |
| 14 | dec | Declination |
| 15 | gf1 | Field distortion shear 1 |
| 16 | gf2 | Field distortion shear 2 |
| 17 | g1 | Rotated shear estimator 1 |
| 18 | g2 | Rotated shear estimator 2 |
| 19 | de | Isotropic response |
| 20 | h1 | Rotated correction term 1 |
| 21 | h2 | Rotated correction term 2 |
| 22 | cos2 | cos 2φ |
| 23 | sin2 | sin 2φ |
| 24 | parity | Coordinate parity (±1) |

## Post-Pipeline: PDF Self-Calibration (External Module)

The `*_all.cat` product can be consumed by an external PDF-symmetry / field-distortion calibration workflow. The auxiliary methodology is documented under the `measurement-*` references, but the auxiliary program source was not part of the reference material used to build this manual. Load `measurement-00-scope.md` before using those references, and inspect the user's actual auxiliary source before making code-level claims.


> **Source-of-truth note:** before a code edit or parameter change, verify the current symbol/value in the user's actual source tree. Filenames and paths in this reference describe the Legacy F77 reference layout; they are navigation hints, not required repository paths or a pinned source snapshot.
