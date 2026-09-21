# Full `f77` vs `f77_Lite`

## Shared execution and I/O contract

Current Full and Lite both:

- build `Fourier_Quad_Pipe`;
- accept `Fourier_Quad_Pipe <EXPO_LIST>`;
- use `strl=512` in the current reference configuration;
- validate exposure-list input more defensively than the old `getarg`/list-reader path;
- include `path_layout.inc` as a build dependency;
- use the same WFST-compatible dataset/product layout;
- use the same family of `fq_*_product_path` helpers in `universal.f`.

So a filesystem/layout fix should normally be reviewed and synchronized in both variants.

## Frozen Lite selectors

In Full `f77/para.inc`, these are editable compile-time selectors. In current `f77_Lite`, they are absent as selectors and their production behavior is compiled directly into the remaining code.

| Full selector | Lite behavior | Consequence |
|---|---:|---|
| `ASTROMETRY_trivial` | `0` | Gaia/PU astrometry only |
| `include_FLAT` | `0` | No super-flat branch |
| `include_Mask` | `2` | DQ masking path retained |
| `ext_cat` | `1` | External source catalog only |
| `ext_PSF` | `0` | PSF measured from stars in the frame |
| `deblending` | `1` | Deblending always applied |
| `PSF_type` | `1` | Local polynomial PSF path |
| `PSF_Ms` | `0` | No multi-scale/PCA PSF reconstruction |

Still-selectable controls include `PROCESS_stage`, `CCD_split`, `gal_smooth`, `star_smooth`, geometry/capacity values, catalog paths, and numerical F6 controls that remain present in Lite.

## Full-only source/features

Full includes `00_psf_module.f` and the substantial PCA/multi-scale reconstruction path used when `PSF_Ms=1`, including PCA-specific `cust_para.inc` parameters. Lite removes that feature path rather than merely disabling it at runtime.

## Porting rule

When porting a Full change to Lite, classify it first:

- **Layout/path/list-reader/build dependency:** normally synchronize in both variants.
- **Shared numerical routine:** compare both implementations and synchronize where the routine still exists.
- **One of the eight frozen feature branches:** do not mechanically add the Full selector/branch to Lite; decide whether expanding Lite's scope is explicitly intended.
- **PCA/multi-scale PSF code:** Full-only unless Lite is deliberately redesigned.

Never infer Lite behavior from old documentation saying it uses a separate legacy filesystem layout; the current reference implementation uses the same modernized product layout as Full.
