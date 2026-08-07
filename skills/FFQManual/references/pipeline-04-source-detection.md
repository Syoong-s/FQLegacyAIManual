# Pipeline Stage 3: Source Detection

> **Lite scope:** `f77_Lite` freezes `ext_cat=1`, `ext_PSF=0`, and `deblending=1`; internal detection, external-PSF selection, and deblending-off branches are not available as switches.

## Purpose

Detect or import all astronomical sources (stars + galaxies) in the pre-processed image, extract image stamps, reconstruct the noise map, deblend overlapping sources, and select star candidates for PSF modeling.

This stage supports two operating modes controlled by the `ext_cat` parameter in `para.inc`:

- **Internal detection** (`ext_cat = 0`): Sources are detected directly from the image using SNR thresholding.
- **External catalog** (`ext_cat = 1`, **default**): Sources are read from a pre-built external galaxy catalog (e.g., DES Y6, DECaLS), with astrometric registration via the WCS solution.

## Key Parameters (`para.inc`)

| Parameter | Value | Meaning |
|:---|:---:|:---|
| `ext_cat` | 1 | Source catalog mode (0=internal, 1=external) |
| `ext_PSF` | 0 | External PSF mode (0=build from stars, 1=use external PSF image) |
| `deblending` | 1 | Enable deblending (1=on, 0=off) |
| `ns` | 64 | Stamp size (pixels) |
| `source_thresh` | 2.0 | Detection threshold (σ) |
| `core_thresh` | 4.0 | Core detection threshold (σ) |
| `n_neighbor` | 5 | Deblending neighbor count |
| `CCD_split` | 2 | Amplifier split (for noise map reconstruction) |
| `ngal_max` | 4000 | Max galaxies per chip |
| `nstar_max` | 2000 | Max stars per chip |
| `len_g` | 40 | Galaxy stamp I/O blocking factor |
| `len_s` | 15 | Star stamp I/O blocking factor |

## Processing Steps

### 1. Read Pre-processed Image and Noise Coefficients

- Read the normalized image (`*_norm.fits`) from Stage 1.
- **Validate**: `normap(1,1) < 0` indicates successful preprocessing; otherwise the chip is rejected.
- **Reconstruct noise map** from the serialized `sigabc` coefficients:
  - When `CCD_split = 2`: Two amplifier planes are used:
    - Left half: `sigmap(i,j) = √(0.5 × (sigabc(1,1) + sigabc(1,2)·i + sigabc(1,3)·j))`
    - Right half: `sigmap(i,j) = √(0.5 × (sigabc(2,1) + sigabc(2,2)·i + sigabc(2,3)·j))`
  - When `CCD_split = 1`: Single plane: `sigmap(i,j) = √(0.5 × (sigabc(1,1) + sigabc(1,2)·i + sigabc(1,3)·j))`
- Mask pixels with `normap < −900` (defects from Stage 1).
- Apply flat-field correction if `include_FLAT = 1`.

### 2. Exposure Catalog Setup (`get_expo_catalog`)

Generate or read the exposure-level catalog structure and noise/weight maps for source extraction.

### 3a. Internal Detection Mode (`ext_cat = 0`)

#### Source Detection (`gen_source_catalog`)

- Detect peaks with signal-to-noise ratio SNR ≥ `core_thresh` (= 4.0).
- For each detected source, extract a `64×64` pixel stamp and corresponding noise stamp.
- Write all source stamps and parameters.

#### Star Candidate Selection (`gen_star_candidate`)

- Apply a size-vs-sharpness criterion to separate point-like sources (stars) from extended sources (galaxies).
- Write star candidate stamps for Stage 4 (FFT-1) and Stage 5 (PSF modeling).

### 3b. External Catalog Mode (`ext_cat = 1`, **default**)

#### Astrometric Registration

- Read the `.head` astrometry file (from Stage 2) for the current chip.
- Generate the galaxy catalog filename from the WCS reference position (CRVAL) via `generate_gal_cat_file_name`.
- The external catalog path is set by `SOURCE_CAT` in `para.inc`.

#### Deblending (`de_blending`)

When `deblending = 1`, overlapping sources from the external catalog are split using the astrometric solution (CRPIX, CD, CRVAL, PU). This prevents blended sources from being treated as a single object.

#### Source Stamp Extraction (`gen_source_ext_catalog`)

- For each galaxy in the external catalog that falls within the chip's footprint, extract the image stamp and noise stamp at the catalog position.
- Store source parameters (position, size, peak, SNR, etc.).

#### Star Candidate Selection (`gen_star_candidate_direct`)

- Select star candidates directly from the image using a size/sharpness criterion.
- Write star candidate stamps for Stage 4 (FFT-1) and Stage 5 (PSF modeling).

### 4. External PSF Mode (`ext_PSF = 1`)

When `ext_PSF = 1`, an external PSF image is read from `PSF_PATH/PSF.fits` instead of being modeled from stellar stamps. In this mode:
- Stage 4 (FFT-1 for stars) is **skipped** entirely.
- Stage 5 (PSF modeling) uses the external PSF directly.

## Output

| File | Content |
|:---|:---|
| `*_source.fits` | All galaxy stamps |
| `*_noise.fits` | Corresponding noise stamps |
| `*_source_info.dat` | Per-source parameter table (positions, size, peak, SNR, etc.) |
| `*_star_can.fits` | Star candidate stamps (only when `ext_PSF = 0`) |
| `*_star_can_noise.fits` | Star candidate noise stamps |
| `*_star_can_info.dat` | Star candidate parameter table |
| `*_orig.cat` | External catalog original fields (when `ext_cat = 1`) |

## Complexity Assessment

Source detection, stamp extraction, and deblending are common in astronomical image processing. The external catalog mode (`ext_cat = 1`) is a significant architectural feature - it allows the pipeline to process pre-defined source lists from external surveys (DES, DECaLS) rather than performing internal detection, which improves completeness and reduces detection bias. The noise map reconstruction from serialized Stage 1 coefficients ensures the noise information is preserved across stages without requiring a separate noise file.


> **Source-of-truth note:** before a code edit or parameter change, verify the current symbol/value in the user's actual source tree. Filenames and paths in this reference describe the Legacy F77 reference layout; they are navigation hints, not required repository paths or a pinned source snapshot.
