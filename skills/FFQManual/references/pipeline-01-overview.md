# Pipeline: Architecture & Execution Model

## Overview

The pipeline is an MPI-parallel Fortran 77 data-reduction program that processes astronomical CCD exposures (DECam/DES and similar surveys) through a 9-stage workflow, from raw images to calibrated shear catalogs.

> This reference is F77-specific. `f77_Lite` keeps the same stage architecture but freezes eight Full-pipeline branch selectors and removes their dead branches. Read `variant-01-full-vs-lite.md` before applying Full configuration advice to Lite.

## Source Structure

### Include Files

| File | Content |
|:---|:---|
| `para.inc` | Master parameters: image dimensions, stage control (`PROCESS_stage`), catalog paths, PSF order, stamp size, thresholds, catalog column indices, `ext_cat`/`ext_PSF`/`CCD_split` modes |
| `cust_para.inc` | CCD geometry (`chipnx=2046`, `chipny=4094`, `Camera_ccd_num=62`) and PCA PSF decomposition parameters |
| `sig_para.inc` | F6 mode-bar noise-plane estimator parameters (~30 config values) |

### Source Files

| File | Stage | Description |
|:---|:---:|:---|
| `main.f` | — | Main entry point: MPI init, exposure list read, stage dispatch |
| `pre_process.f` | 1 | Background, F6 noise estimation, DQ mask, defect detection |
| `proc_astrometry.f` | 2 | Astrometric calibration via Gaia reference catalog |
| `proc_source.f` | 3 | Source detection/extraction (internal or external catalog) |
| `proc_FFT_st1.f` | 4 | First-stage Fourier transform (star candidates) |
| `proc_PSF.f` | 5 | PSF modeling from stellar stamps |
| `proc_psfreconsV2.f` | — | PSF PCA reconstruction (when `PSF_Ms=1`) |
| `00_psf_module.f` | — | Fortran 90 module for global PSF PCA storage |
| `proc_FFT_st2.f` | 6 | Second-stage Fourier transform (galaxies) |
| `proc_shear.f` | 7 | Fourier_Quad shear estimation |
| `proc_info.f` | 8 | Per-exposure statistics collection |
| `proc_combine_shear_catalog.f` | 9 | Shear catalog combination and calibration |
| `astrometry_calib.f` | — | Astrometric calibration utility routines |
| `ex_star.f` | — | Star extraction and classification |
| `rw_fits_image.f` | — | FITS image read/write (via CFITSIO) |
| `universal.f` | — | General-purpose utility functions |
| `univ_imag_proc.f` | — | Image processing utilities (FFT, power spectrum, smoothing) |
| `press.f` | — | Numerical Recipes routines (sorting, statistics, interpolation) |
| `mpi_routines.f` | — | MPI distribution helper (`mpi_distribute`) |
| `FFTPACK.f` | — | FFTPACK 5.1 FFT library |

## Parallel Execution Model

**Hybrid load-balancing scheme:**

- **Exposure-level dynamic load balancing**: A manager-worker model (`mpi_distribute`) distributes exposure tasks across MPI nodes. Each node processes one exposure at a time, requesting the next from the manager when done.
- **Chip-level sequential processing**: Within each exposure, up to 62 CCD chips (DECam) are processed sequentially.

## Stage Scheduling via Prime Factor Decomposition

An integer `PROCESS_stage` acts as a bitmask via prime factor decomposition. Each stage corresponds to a unique prime factor:

| Stage | Prime | Function | Core Algorithm |
|:---|:---:|:---|:---|
| 1 | 2 | Pre-processing | F6 noise-plane estimation, background, defect masking |
| 2 | 3 | Astrometry | PU polynomial sky-coordinate mapping |
| 3 | 5 | Source Detection | Internal/external catalog, deblending, star candidates |
| 4 | 7 | FFT-1 | Power spectrum for star candidates (skipped if `ext_PSF=1`) |
| 5 | 11 | PSF Modeling | χ² star selection, spatial polynomial + PCA PSF |
| 6 | 13 | FFT-2 | Power spectrum for galaxies (+ flux/SNR) |
| 7 | 17 | Shear Measurement | Fourier_Quad shear estimation (5 estimators) |
| 8 | 19 | Info Aggregation | Chip→exposure diagnostics |
| 9 | 23 | Catalog Assembly | Quality cuts, distortion calibration, merged catalog |

A stage is executed when `mod(PROCESS_stage, prime) .eq. 0`. The default value is the product of all 9 primes, enabling end-to-end processing. Any combination of stages can be run by setting the appropriate product.

## Data Flow

```
Exposure List -> [Stage 1: Pre-processing (F6 noise)] -> [Stage 2: Astrometry]
    -> [Stage 3: Source Detection (ext_cat)] -> [Stage 4: FFT-1 (stars)]
    -> [Stage 5: PSF Modeling] -> [Stage 6: FFT-2 (galaxies)]
    -> [Stage 7: Shear Measurement] -> [Stage 8: Info Aggregation]
    -> [Stage 9: Catalog Merge & Calibration] -> *_all.cat
```

After Stage 8, exposure-level diagnostics are reduced across all MPI nodes via `MPI_AllReduce` (sum), and the result is written to `expo_info.dat` by rank 0.

## Key Global Parameters

| Parameter | Value | Meaning |
|:---|:---:|:---|
| `npx`, `npy` | 3000, 5000 | Max image dimensions |
| `ns` | 64 | Stamp / power spectrum size (pixels) |
| `npd` | 33 | PU polynomial distortion terms |
| `NMAX_CHIP` | 62 | Max CCDs per exposure |
| `NMAX_EXPO` | 25000 | Max exposures |
| `PROCESS_stage` | 2·3·5·7·11·13·17·19·23 | Full pipeline stage switch |
| `ext_cat` | 1 | Source catalog mode (0=internal, 1=external) |
| `ext_PSF` | 0 | External PSF mode (0=from stars, 1=external image) |
| `CCD_split` | 2 | Amplifier split mode |
| `include_FLAT` | 0 | Flat-field correction (0=off, 1=on) |
| `include_Mask` | 2 | DQ mask mode (0=off, 1=flat, 2=DQ, 3=both) |
| `PSF_type` | 1 | PSF evaluation mode (1=polynomial, 2=very-local) |
| `PSF_Ms` | 0 | PCA reconstruction (0=off, 1=on) |
| `psf_order` | 8 | PSF polynomial order |
| `npo` | 64 | PSF polynomial terms |
| `npl` | 10 | Local PSF polynomial terms |
| `nstar_min_local` | 16 | Minimum true stars per chip |
| `n_pcs` | 100 | PCA principal components |
| `npp6th` | 28 | 6th-order 2D polynomial terms for PCA coefficients |
| `blocksize` | 200 | Background/noise block size (pixels) |
| `pixel_size` | 0.2628 | Pixel scale (arcsec) |
| `chi2_thresh` | 0.01 | Exposure PSF χ² quality threshold |
| `g1_c`, `g2_c` | −0.001, −0.0003 | Additive bias parameters (g-band) |
| `PSFr_ratio` | 0.75 | Window scale factor: k₀ = 0.75·kₛ (defined locally in `proc_shear.f`) |
| `sig_scale` | 1.027786 | F6 noise estimator publication scale (honest σ convention) |

## Compilation & Libraries

- Fortran 77 + MPI, compiled with `mpif77` (GCC 4.8.5 gfortran + MPICH 4.1.2)
- Libraries: CFITSIO 4.3.1 (FITS I/O), LAPACK 3.8.0 + reference BLAS (PCA eigen-decomposition `dsyevd`), FFTPACK 5.1
- Docker environment: Rocky Linux 8.10 base image with reproducible toolchain
- For reproducible Docker/HPC toolchain and runner details, use `deployment-01-build-docker-runner.md`.

## Editing Guidance

For parameter changes load `config-01-parameter-map.md`. For source changes load `development-01-change-map.md` and inspect the current source. The tables above are a navigation map, not a substitute for the user's actual source tree.


> **Source-of-truth note:** before a code edit or parameter change, verify the current symbol/value in the user's actual source tree. Filenames and paths in this reference describe the Legacy F77 reference layout; they are navigation hints, not required repository paths or a pinned source snapshot.
