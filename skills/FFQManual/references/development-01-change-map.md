# Developer Change Map for the F77 Pipeline

## Core rule

This reference tells you where to look. It is not a substitute for reading current source. For any requested implementation change, inspect the exact symbol and callers in the user's current tree before editing.

## 1. Request → likely edit locations

| Requested change | Primary files/symbols | Also inspect |
|---|---|---|
| CLI exposure-list argument / top-list parsing | `main.f`, `initialize`, `fq_record_length` | `strl`, `NMAX_EXPO`, MPI broadcast |
| Per-exposure chip-list parsing / dataset-root derivation | `universal.f::get_image_list` | `strl`, `NMAX_CHIP`, initializer list format |
| Dataset/product directory contract | `path_layout.inc` | initializer, all `fq_*_product_path` call sites |
| Product path construction / path overflow handling | `universal.f::{fq_assign_path,fq_chip_product_path,fq_expo_product_path,fq_expo_ccd_product_path,fq_base_product_path}` | `strl`, local filename buffer lengths |
| Initializer extraction / list publication / resume policy | `init_program/init_program.py` | `init_program/README.md`, F77 list readers |
| Background/F6 estimator | `pre_process.f::set_sig` and helpers, `sig_para.inc` | DQ-updated `weight`, normalization callers |
| DQ mask naming/timing | `pre_process.f`, `path_layout.inc::DIR_DQ`, `fq_expo_ccd_product_path` | initializer DQ `CCDNUM` naming |
| Flat/DQ selector behavior | `pre_process.f`, `para.inc` | Full/Lite variant boundary, container binds |
| External source tile filename prefix | `para.inc::SOURCE_CAT_TILE_PREFIX`, catalog filename builder in `universal.f` | external catalog preparation docs/scripts |
| Camera geometry / CCD count | `cust_para.inc`, `para.inc` | fixed arrays, chip loops, combiner |
| Astrometric model / Gaia match | `proc_astrometry.f`, `astrometry_calib.f` | Stage-1 `_astro.dat`, `npd`, `DIR_ASTRO_*` |
| Internal/external source extraction | `proc_source.f` | astrometry products, `SOURCE_CAT`, `SOURCE_CAT_TILE_PREFIX` |
| FFT-1/FFT-2 | `proc_FFT_st1.f`, `proc_FFT_st2.f`, `FFTPACK.f` | stamp dimensions and product helpers |
| PSF star selection/local fit | `proc_PSF.f` | `para.inc`, Full/Lite branch differences |
| PCA/multi-scale PSF | Full: `00_psf_module.f`, `proc_psfreconsV2.f`, `cust_para.inc` | not present as a feature path in Lite |
| Shear estimator / PSF deconvolution | `proc_shear.f` | PSF products and `para.inc` record indices |
| Exposure diagnostics | `proc_info.f`, `main.f` reduction/write | Stage 9 cuts, `DIR_EXPO_INFO` |
| Final catalog merge | `proc_combine_shear_catalog.f` | source original catalog + shear record ABI |
| Build dependencies / compiler compatibility | `Makefile` | `path_layout.inc`, container toolchain, modern `-fallow-argument-mismatch` when needed |

## 2. Current path API rule

Do not add new code like:

```fortran
filename=trim(DIR_OUTPUT)//'/stamps/...'
```

when the current tree already routes products through the centralized layout/helper layer. First decide which path shape is correct:

- chip product under an exposure directory → `fq_chip_product_path`;
- exposure product → `fq_expo_product_path`;
- exposure + physical CCD product (not science sequence number) → `fq_expo_ccd_product_path`;
- dataset-global product → `fq_base_product_path`.

Add/change the directory token in `path_layout.inc` when the product category itself changes.

## 3. DQ/F6 modification rule

Current Stage 1 ordering matters. DQ is read and nonzero pixels set `weight=0` before `set_background`/`set_sig`. `set_sig` does not open DQ itself, but it skips nonpositive weight when choosing pixels/triples.

Therefore any request such as “make F6 ignore DQ” or “move DQ after noise fitting” is a behavioral/numerical change, not a documentation-only cleanup. Inspect the Stage-1 call order and validate F6 outputs after changing it.

## 4. Full/Lite synchronization rule

For every modified source filename that exists in both variants:

1. diff the current Full and Lite versions;
2. classify the change as shared I/O/numerics vs Full-only feature logic;
3. synchronize shared path/list/error-handling changes;
4. preserve Lite's frozen-selector design unless expanding Lite is the stated goal.

`path_layout.inc`, list parsing, path helpers, and most modern I/O hardening belong to the synchronized category.

## 5. Fixed-form / path safety

- Current reference `strl` is 512, but internal construction helpers may use larger temporary buffers before assigning into caller-owned strings.
- Do not rely on silent Fortran character truncation. Preserve or improve explicit length checks.
- Keep fixed-form continuation and line-length constraints in mind.
- Large arrays are one reason the reference Makefile uses `-mcmodel=medium`.
- On modern GNU Fortran, existing legacy implicit interfaces may require `-fallow-argument-mismatch`; use it as a compatibility build flag rather than rewriting interfaces casually.

## 6. Verification ladder

For a source change, use the cheapest relevant checks first:

1. symbol/path search confirms every intended caller is covered;
2. clean compile of the affected variant;
3. Full/Lite compile when a shared source/layout change was made;
4. initializer/list-format sanity check when filesystem contracts changed;
5. one-exposure or smallest available functional run;
6. inspect the earliest affected intermediate product;
7. compare downstream catalog/diagnostic counts;
8. multi-rank run for MPI-sensitive changes.

For path changes, verify actual produced paths against `dataset-01-initializer-layout.md`, not only that the executable exits successfully.
