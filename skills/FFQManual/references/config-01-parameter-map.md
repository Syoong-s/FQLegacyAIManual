# F77 Configuration and Parameter Map

## How to use this reference

These values come from the Legacy F77 reference implementation used to build this manual, including reference files named `f77/para.inc`, `f77/cust_para.inc`, `f77/sig_para.inc`, and corresponding Lite includes. Treat them as an orientation snapshot: **before editing, locate and read the user's current include/configuration file**. The reference paths are navigation hints, not required repository paths. Every `parameter(...)` value is compile-time; rebuild after changing it.

For a parameter whose semantics are not documented below, trace all uses in source before changing it. Fixed-form Fortran dimensions often participate in array shapes and file contracts, so a seemingly local size change can require multiple files.

## 1. Execution and branch selectors (`para.inc`)

| Parameter | Full default | Lite | Meaning / change impact |
|---|---:|---|---|
| `PROCESS_stage` | `2*3*5*7*11*13*17*19*23` | editable | Prime-product stage selector. Removing a prime skips that stage; verify prerequisites/intermediates already exist. |
| `ASTROMETRY_trivial` | `0` | frozen `0` | `0` Gaia/PU solution; Full may select trivial mode. |
| `include_FLAT` | `0` | frozen `0` | Full flat-field branch. If enabled, compiled `FLAT_PATH` and runner bind must be valid. |
| `include_Mask` | `2` | frozen `2` | Full mask mode: `0` off, `1` flat-related, `2` DQ, `3` both (verify exact branch behavior before changing). |
| `ext_cat` | `1` | frozen `1` | `1` external source catalog; `0` internal detection (Full only). |
| `ext_PSF` | `0` | frozen `0` | `1` external PSF image (Full only); also changes Stage 4/5 relevance and runner bind needs. |
| `deblending` | `1` | frozen `1` | Full deblending switch. |
| `PSF_type` | `1` | frozen `1` | Full: `1` local polynomial, `2` very-local/hybrid interpolation. |
| `PSF_Ms` | `0` | frozen `0` | Full: optional multi-scale/PCA reconstruction. Lite removes this path. |
| `CCD_split` | `2` | editable | Number/geometry mode used to split a CCD into amplifier regions for noise handling. |

### Stage prime map

`2=pre_process`, `3=astrometry`, `5=source`, `7=FFT-1`, `11=PSF`, `13=FFT-2`, `17=shear`, `19=info`, `23=combine`.

Do not assume arbitrary isolated stages are self-contained. If skipping an upstream stage, required intermediate products must already exist and match the current configuration.

## 2. Paths (`para.inc`) and container coupling

| Parameter | Purpose | Runner coupling |
|---|---|---|
| `ASTROMETRY_CAT` | Gaia/reference astrometry catalog root | Must match `ASTROMETRY_CAT_CONTAINER` destination |
| `SOURCE_CAT` | External source catalog root (`ext_cat=1`) | Must match `SOURCE_CAT_CONTAINER` destination |
| `FLAT_PATH` | Flat/calibration root when used | Must match `FLAT_PATH_CONTAINER` destination |
| `PSF_PATH` | External PSF root when `ext_PSF=1` | No dedicated standard runner bind; add an extra bind / `HPC_EXTRA_BINDS` and compile the same container path |

The Docker and HPC runners bind host paths into fixed container destinations. **Editing only the host env file does not change compiled Fortran path strings.** When a container destination changes, update the matching include value and rebuild.

## 3. Image geometry and fixed capacities

| Parameter | Default | Meaning / caution |
|---|---:|---|
| `npx`, `npy` | `3000`, `5000` | Maximum image dimensions used by fixed arrays. Increasing can raise memory substantially. |
| `strl` | `150` | Fixed character length for paths/exposure names. |
| `chipnx`, `chipny` (`cust_para.inc`) | `2046`, `4094` | Camera CCD geometry. Change only for a different detector/layout and audit geometry assumptions. |
| `Camera_ccd_num` | `62` | Camera CCD count. |
| `NMAX_CHIP` | `62` | Max chips per exposure; keep consistent with camera/data. |
| `NMAX_EXPO` | `25000` | Max exposure list capacity. |
| `ns` | `64` | Source/star stamp and Fourier-grid side. High-impact: array shapes, output FITS dimensions, FFT behavior, margins, and derived values depend on it. |
| `nsns` | `ns*ns` | Derived stamp pixel count. Do not set independently. |
| `ns_2` | `ns/2` | Derived half-stamp size. |
| `chip_margin` | `8` | Margin used around chip/stamp extraction logic. |
| `nl_2` | `ns_2+chip_margin` | Derived. |
| `nl` | `nl_2*2` | Derived. |
| `chip_edge_margin` | `chip_margin` | Edge exclusion margin. |
| `ngal_max` | `4000` | Maximum galaxies per chip in fixed storage. |
| `nstar_max` | `2000` | Maximum stars/candidates per chip in fixed storage. |
| `npara` | `25` | Source/shear parameter record capacity; changing may alter array/file contracts. |
| `len_g`, `len_s` | `40`, `15` | Galaxy/star blocked I/O dimensions used by FITS processing. |
| `len_sam` | `50` | Internal sample/work-array size; trace uses before changing. |

## 4. Detection, masks, and preprocessing

| Parameter | Default | Role |
|---|---:|---|
| `blocksize` | `200` | Background/preprocessing block side. Separate from `sig_blocksize` although both currently equal 200. |
| `source_thresh` | `2.0` | Source/defect threshold used in detection-related logic. |
| `core_thresh` | `4.0` | Core threshold for source detection. |
| `flat_thresh` | `0.01` | Flat-field validity threshold (Full flat branch). |
| `area_max` | `ns*ns` | Area ceiling derived from stamp size. |
| `area_thresh` | `6` | Minimum connected/merged defect area threshold. |
| `saturation_thresh` | `25000` | Saturation threshold. |
| `flag_thresh` | `3` | Source/status flag threshold used by downstream cuts. Trace consumers before altering. |
| `dz_thresh` | `0.1` | Matching/selection tolerance used by source/astrometry-related logic; verify use site before changing. |
| `n_neighbor` | `5` | Deblending neighbor count (Full selector path; Lite deblending always on). |

## 5. PSF and Fourier processing

| Parameter | Default | Role / dependency |
|---|---:|---|
| `psf_order` | `8` | PSF polynomial order/control used by the full model. |
| `npo` | `64` | PSF fit term/work dimension. `nstar_min=npo*3/2`, so changing `npo` changes minimum candidate requirement. |
| `npox` | `8` | PSF polynomial/work dimension; trace array definitions before changing. |
| `nstar_min` | `npo*3/2` (`96`) | Global minimum candidate count derived from `npo`. |
| `npl` | `10` | Number of terms for local polynomial PSF fit. |
| `nplx` | `2` | Local fit/work dimension; trace exact uses before changing. |
| `nstar_min_local` | `16` | Minimum selected stars for valid local PSF fit. |
| `step_psf` | `100` | Grid/step for very-local PSF mode (Full `PSF_type=2`). |
| `SNR_PSF` | `100` | PSF star S/N-related threshold/control. |
| `gal_smooth` | `2` | Galaxy power-spectrum smoothing mode. |
| `star_smooth` | `2` | Star power-spectrum smoothing mode. |
| `chi2_thresh` | `0.01` | PSF/exposure fit quality cut used in catalog combination. |

### Full-only PCA parameters (`cust_para.inc`)

Lite removes these because `PSF_Ms=0` is frozen.

| Parameter | Default | Role |
|---|---:|---|
| `rescale_size` | `1.2` | PCA residual rescaling control |
| `procs_pn` | `40` | PCA/reconstruction process/work partition parameter |
| `work_pn` | `10` | PCA work partition parameter |
| `nblocks` | `2` | Spatial block partition for PCA coefficient modeling |
| `n_pcs` | `100` | Number of retained principal components |
| `npp6th` | `28` | Terms in 6th-order 2D polynomial for PCA coefficients |
| `nmax_star_pchip` | `1000000` | Maximum per-chip star storage for PCA path |

## 6. Astrometry and catalog record indices

| Parameter | Default | Role |
|---|---:|---|
| `npd` | `33` | PU astrometric polynomial/distortion parameter count. High-impact; do not change without rewriting the model and I/O. |
| `pixel_size` | `0.2628` arcsec | Pixel scale used in physical/FWHM conversion. |
| `isig`...`iparity` | fixed indices | Column positions in the 24-column shear record. Treat as an I/O ABI: changing requires all writers/readers/headers to change together. |

Current shear-record indices are: `isig=4`, `istar=5`, `ipeak=5`, `i_imax=6`, `i_jmax=7`, `ih_flux=8`, `ih_area=9`, `iflag=10`, `iPSF=11`, `iSNR_F=12`, `ira=13`, `idec=14`, `igf1=15`, `igf2=16`, `ig1=17`, `ig2=18`, `ide=19`, `ih1=20`, `ih2=21`, `icos2=22`, `isin2=23`, `iparity=24`.

## 7. Catalog calibration constants

| Parameter | Default | Role |
|---|---:|---|
| `g1_c` | `-0.001` | Compiled additive correction constant currently labeled g-band in source. |
| `g2_c` | `-0.0003` | Same for component 2. |

Changing these changes Stage 9 catalog calibration, not the raw Stage 7 Fourier moments. Keep band/dataset provenance explicit.

## 8. F6 estimator (`sig_para.inc`)

The F6 estimator uses an image-only private symmetric mask, pixel triples, and a fitted per-amplifier plane. It does not read or require the DQ mask inside `set_sig`. Load `pipeline-02-preprocessing.md` for the full algorithm.

| Parameter | Default | Function |
|---|---:|---|
| `sig_blocksize` | `200` | Target seeding block side |
| `sig_block_max` | `sig_blocksize^2` | Derived fixed work-array bound |
| `sig_max_blocks` | `2048` | Maximum block count |
| `sig_min_block_pixels` | `1000` | Min pixels in valid seed block |
| `sig_min_block_triples` | `1000` | Min triples in block |
| `sig_min_blocks` | `4` | Minimum usable blocks |
| `sig_hist_nbin` | `256` | Mode histogram bins |
| `sig_hist_range` | `6.0` | Histogram range in sigma units |
| `sig_min_mode_count` | `500` | Minimum modal-bin support |
| `sig_min_lower_count` | `1000` | Minimum lower-half sample count |
| `sig_lower_quantile` | `0.3173105` | Lower-half quantile used to estimate initial sigma |
| `sig_clip_k` | `3.0` | Symmetric image clip half-width |
| `sig_rdil` | `2` | Private-mask dilation radius |
| `sig_clip_niter` | `2` | Global + local plane refinement passes |
| `sig_min_fit_triples` | `1000` | Minimum triples for final plane fit |
| `sig_min_fit_frac` | `0.20` | Minimum surviving triple fraction |
| `sig_median_ratio` | `1.2678405` | Calibration constant for median(pix)/sigma^2 mixture |
| `sig_plane_min` | `1e-8` | Minimum plane value |
| `sig_max_plane_ratio` | `4.0` | Maximum plane corner ratio (factor-two sigma variation) |
| `sig_pivot_min` | `1d-8` | Numerical pivot floor |
| `sig_scale_s1` | `0.673475` | Historical stage-one scale reference |
| `sig_scale_s2` | `1.027786` | Active stage-two scale |
| `sig_scale` | `sig_scale_s2` | Published active scale |

`sig_scale` is a calibrated convention, not a generic tuning knob. If F6 clipping/selection logic changes, re-validation/re-calibration is required rather than casually compensating with this constant.

## Parameter-change workflow

1. Identify the owning subsystem and variant.
2. Read the current include definition and all direct uses (`rg`/symbol search).
3. Check derived dimensions/thresholds and file contracts.
4. Make the smallest change; do not duplicate a derived expression with a literal.
5. Rebuild from clean state when dimensions/branch selectors changed.
6. Re-run the earliest affected stage plus any downstream consumers.
7. Compare intermediate diagnostics, not only final `*_all.cat`.
