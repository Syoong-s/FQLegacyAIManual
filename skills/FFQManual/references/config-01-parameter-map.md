# F77 Configuration and Parameter Map

## How to use this reference

These values describe the current Legacy F77 reference layout. Treat them as orientation values: before editing, locate and read the user's current include/configuration file. Every `parameter(...)` value is compile-time; rebuild after changing it.

For a symbol not documented below, trace all uses first. Fixed-form Fortran dimensions, paths, and record indices often form cross-file contracts.

## 1. Execution and branch selectors (`para.inc`)

| Parameter | Full default | Lite | Meaning / change impact |
|---|---:|---|---|
| `PROCESS_stage` | `2*3*5*7*11*13*17*19*23` | editable | Prime-product stage selector. Skipping an upstream stage requires compatible intermediates to already exist. |
| `ASTROMETRY_trivial` | `0` | frozen `0` | Full may select header-WCS trivial mode; Lite keeps Gaia/PU path only. |
| `include_FLAT` | `0` | frozen `0` | Full flat-field branch. If enabled, `FLAT_PATH` and runtime bind must be valid. |
| `include_Mask` | `2` | frozen `2` | Full mask mode; current production value applies DQ. |
| `ext_cat` | `1` | frozen `1` | External source-catalog mode. |
| `ext_PSF` | `0` | frozen `0` | Full may use external PSF; Lite always builds PSF from frame stars. |
| `deblending` | `1` | frozen `1` | Full deblending switch; Lite always deblends. |
| `PSF_type` | `1` | frozen `1` | Full: local polynomial vs alternate very-local mode. |
| `PSF_Ms` | `0` | frozen `0` | Full optional multi-scale/PCA PSF reconstruction; removed from Lite. |
| `CCD_split` | `2` | editable | CCD/amplifier split used by background/F6 processing. |

Stage primes: `2=pre_process`, `3=astrometry`, `5=source`, `7=FFT-1`, `11=PSF`, `13=FFT-2`, `17=shear`, `19=info`, `23=combine`.

## 2. Paths and catalog naming (`para.inc`)

| Parameter | Current reference value / role | Runtime coupling |
|---|---|---|
| `ASTROMETRY_CAT` | Gaia/reference catalog root | Container-visible path must match compiled string |
| `SOURCE_CAT` | External source catalog root | Container-visible path must match compiled string |
| `FLAT_PATH` | Flat/calibration root when used | Container-visible path must match compiled string |
| `PSF_PATH` | External PSF root when `ext_PSF=1` | Add a matching runtime bind when used |
| `SOURCE_CAT_TILE_PREFIX` | `extern_` | Prefix before `RA_<RA0>_<RA1>_Dec_<Dec0>_<Dec1>.dat`; prefix itself does not include `RA_` |

Changing only a host-side env variable does not change compiled Fortran paths. If the container destination changes, update the include value and rebuild.

## 3. Fixed capacities and image geometry

| Parameter | Current reference | Meaning / caution |
|---|---:|---|
| `npx`, `npy` | `3000`, `5000` | Maximum image dimensions used by fixed arrays. |
| `strl` | `512` | Fixed path/exposure string length in both current Full and Lite. Long inputs are checked instead of silently truncated in the current list readers. |
| `chipnx`, `chipny` | `2046`, `4094` | Camera CCD geometry. |
| `Camera_ccd_num` | `62` | Camera CCD count. |
| `NMAX_CHIP` | `62` | Maximum chip paths per exposure. |
| `NMAX_EXPO` | `25000` | Maximum top-list exposure count. |
| `ns` | `64` | Stamp/Fourier-grid side; high-impact array and file-layout setting. |
| `ngal_max` | `4000` | Maximum galaxies per chip in fixed storage. |
| `nstar_max` | `2000` | Maximum stars/candidates per chip in fixed storage. |

Before changing fixed dimensions, audit declarations, MPI counts, FITS shapes, and all derived constants.

## 4. Dataset/product paths

Product-directory constants are no longer expected to be duplicated in stage code. Current Full and Lite both use `path_layout.inc`; the current contract is documented in [dataset-01-initializer-layout.md](dataset-01-initializer-layout.md).

When changing a path contract, inspect:

1. `path_layout.inc`;
2. `universal.f` helpers `fq_chip_product_path`, `fq_expo_product_path`, `fq_expo_ccd_product_path`, `fq_base_product_path`, `fq_assign_path`;
3. initializer output layout;
4. all external scripts/containers that write list paths.

## 5. Preprocessing / F6

Important current relationship:

- Stage 1 loads the DQ image before `set_background` / `set_sig` when the active mask mode requires DQ.
- Nonzero DQ pixels set `weight=0`.
- `set_sig` itself does **not** open/read a DQ FITS file, but it receives `weight` and skips nonpositive-weight pixels/triples.

Therefore the current F6 estimate can be affected by DQ through its input weight mask. Do not describe current Stage 1 as DQ-independent merely because `set_sig` performs no DQ I/O.

Key parameters include `blocksize=200`, `saturation_thresh=25000`, `source_thresh=2.0`, `area_thresh=6`, plus the numerical controls in `sig_para.inc` (`sig_blocksize`, `sig_clip_k`, `sig_rdil`, `sig_clip_niter`, `sig_min_fit_triples`, `sig_scale`, etc.). Treat `sig_scale` as a calibrated convention, not a generic tuning knob.

## 6. PSF and Fourier controls

Current orientation values include `psf_order=8`, `npo=64`, `npl=10`, `nstar_min_local=16`, `step_psf=100`, `gal_smooth=2`, `star_smooth=2`, and `chi2_thresh=0.01`.

Full-only PCA controls live in `cust_para.inc`, including `rescale_size=1.2`, `procs_pn=40`, `work_pn=10`, `nblocks=2`, `n_pcs=100`, `npp6th=28`, and `nmax_star_pchip=1000000`. Lite removes these because the `PSF_Ms=1` branch is absent.

## 7. Astrometry / record ABI

`npd=33` is the PU astrometric parameter count. The shear record uses fixed column indices in `para.inc`; treat these as a file-format ABI and change all writers/readers/headers together.

## Parameter-change workflow

1. Identify owning subsystem and variant.
2. Read the current definition and direct uses.
3. Check derived dimensions, path helpers, and file contracts.
4. Make the smallest change; do not replace a derived expression with a duplicated literal.
5. Rebuild from clean state for include/branch/dimension changes.
6. Re-run the earliest affected stage and downstream consumers.
7. Compare intermediate products and logs, not only final `*_all.cat`.
