# Pipeline: Architecture & Execution Model

## Overview

The Legacy F77 pipeline is an MPI-parallel astronomical CCD reduction and weak-lensing shear pipeline. It processes initialized science chips through nine prime-selected stages, from preprocessing and astrometry to source extraction, PSF modeling, Fourier_Quad shear estimation, diagnostics, and final catalogs.

`f77_Lite` preserves the same stage architecture and current filesystem contract, but freezes eight Full-pipeline feature selectors and removes their dead branches. Read [variant-01-full-vs-lite.md](variant-01-full-vs-lite.md) before applying Full feature advice to Lite.

For raw `.fits.fz` preparation and the current WFST-compatible directory contract, load [dataset-01-initializer-layout.md](dataset-01-initializer-layout.md).

## Source Structure

### Core include files

| File | Role |
|---|---|
| `para.inc` | Master compile-time parameters: dimensions, `PROCESS_stage`, catalog paths, selector branches, catalog indices, thresholds, PSF controls |
| `path_layout.inc` | Current product-directory contract shared by Full and Lite |
| `cust_para.inc` | CCD geometry and, in Full, PCA/multi-scale PSF parameters |
| `sig_para.inc` | F6 mode-bar noise-plane numerical controls |

### Main source files

| File | Stage / role |
|---|---|
| `main.f` | MPI lifecycle, CLI top exposure-list input, stage dispatch, Stage-8 global reduction |
| `pre_process.f` | Stage 1: image read, DQ/flat behavior, background, F6, astrometry-source preparation, defects |
| `proc_astrometry.f` | Stage 2: Gaia/PU astrometric solution |
| `proc_source.f` | Stage 3: source import/detection, deblending, stamp extraction, star candidates |
| `proc_FFT_st1.f` | Stage 4: star-candidate power spectra |
| `proc_PSF.f` | Stage 5: star selection and PSF modeling |
| `proc_psfreconsV2.f` | Full-only optional PCA/multi-scale reconstruction support |
| `00_psf_module.f` | Full-only PSF PCA storage module |
| `proc_FFT_st2.f` | Stage 6: galaxy/source power spectra |
| `proc_shear.f` | Stage 7: Fourier_Quad shear estimation |
| `proc_info.f` | Stage 8: exposure diagnostics |
| `proc_combine_shear_catalog.f` | Stage 9: catalog merge and calibration |
| `astrometry_calib.f` | Astrometric utility routines |
| `rw_fits_image.f` | FITS I/O through CFITSIO |
| `universal.f` | General utilities, list reading, catalog naming, centralized product-path helpers |

## Execution model

The executable takes one positional argument:

```bash
Fourier_Quad_Pipe <EXPO_LIST>
```

The current reference implementation uses `get_command_argument` plus explicit length/status checks. The top list is parsed by `initialize`; per-exposure science lists are read by `get_image_list`.

MPI work is exposure-parallel: ranks receive exposures dynamically while chips within one exposure are processed sequentially.

## Stage scheduling

`PROCESS_stage` is a product of unique primes. A stage runs when `mod(PROCESS_stage, prime) == 0`.

| Stage | Prime | Function |
|---:|---:|---|
| 1 | 2 | Preprocessing / background / F6 / DQ / defect handling |
| 2 | 3 | Astrometry |
| 3 | 5 | Source extraction/import and star candidates |
| 4 | 7 | Star FFT/power spectrum |
| 5 | 11 | PSF modeling |
| 6 | 13 | Galaxy FFT/power spectrum |
| 7 | 17 | Shear measurement |
| 8 | 19 | Exposure diagnostics |
| 9 | 23 | Catalog assembly/calibration |

Default Full `PROCESS_stage` currently includes all nine primes.

## Dataset and path model

With the current initializer contract, a target resembles:

```text
<output-root>/
├── <target>/
│   ├── science/<exposure>/<exposure>_<sequence>.fits
│   ├── dqmask/<exposure>/<exposure>_<CCDNUM>.fits
│   ├── expolists/<exposure>.list
│   ├── astrometry/...
│   ├── stamps/...
│   └── result/...
└── expo_<target>.list
```

`get_image_list` derives the dataset root from the science path layout. Stage code should use the centralized path API rather than hard-code old product locations:

- `fq_chip_product_path`
- `fq_expo_product_path`
- `fq_expo_ccd_product_path`
- `fq_base_product_path`

Product directory names come from `path_layout.inc`. This layout is currently shared by `f77` and `f77_Lite`.

## Data flow

```text
expo_<target>.list
  -> expolists/<exposure>.list
  -> science/<exposure>/<chip-sequence>.fits
  -> Stage 1 preprocessing
  -> Stage 2 astrometry
  -> Stage 3 source/stamp extraction
  -> Stage 4 star FFT
  -> Stage 5 PSF
  -> Stage 6 galaxy FFT
  -> Stage 7 shear
  -> Stage 8 diagnostics
  -> Stage 9 result/<exposure>_all.cat
```

Stage-8 `expo_para` values are reduced across ranks with `MPI_AllReduce`; rank 0 publishes the global exposure diagnostics beside the top exposure-list path according to the current source behavior.

## Important current parameters

Orientation values only; verify the actual tree before editing:

- `strl = 512`
- `NMAX_EXPO = 25000`
- `NMAX_CHIP = 62`
- `ext_cat = 1` in current Full default; frozen in Lite
- `include_Mask = 2` in current Full default; frozen in Lite
- `SOURCE_CAT_TILE_PREFIX = 'extern_'`
- `PSF_type = 1`, `PSF_Ms = 0` in current Full default; frozen in Lite

See [config-01-parameter-map.md](config-01-parameter-map.md) for the change map.

## Compilation

Both variants currently depend on `path_layout.inc` in addition to the traditional includes. Typical build:

```bash
make -C f77 \
  LAPACK_LIB_DIR="$SCIENCE_PREFIX/lib" \
  CFITSIO_LIB_DIR="$SCIENCE_PREFIX/lib" \
  FFLAGS='-mcmodel=medium -w -fallow-argument-mismatch'
```

Replace `f77` with `f77_Lite` for Lite. The `-fallow-argument-mismatch` flag is a modern GNU Fortran compatibility aid for the legacy implicit-interface code; the recorded legacy container toolchain may not need the same modern workaround.

## Editing guidance

For parameter changes, load `config-01-parameter-map.md`. For source changes, load `development-01-change-map.md` and inspect the current source. For filesystem or initializer questions, load `dataset-01-initializer-layout.md`.

> Source-of-truth note: this manual describes a current reference implementation but is not a vendored source snapshot. Before editing, verify the current symbol, call order, and file layout in the user's tree.
