# Dataset Initializer and Current F77 Product Layout

## Scope

Use this reference when the task involves `init_program.py`, `.fits.fz` extraction, exposure lists, DQ naming, dataset roots, product directories, or compatibility between the initializer and `f77` / `f77_Lite`.

The user's actual source tree remains authoritative. The layout below describes the current reference contract used by the modernized Legacy F77 implementation.

## 1. Initializer entry point

`init_program/init_program.py` replaces the former split workflow based on `gencat.py`, `decom_mask.py`, and a separate initializer Slurm script. One Python entry point supports:

- local serial execution;
- MPI execution under `mpirun`/compatible launcher;
- direct `sbatch init_program.py ...`, where the script starts an `srun` step unless `--no-srun` is used.

Runtime requirements are Python, Astropy/NumPy, optional `mpi4py`, and Slurm `srun` only for direct batch relaunch.

Typical serial run:

```bash
python init_program/init_program.py \
  --science-root /data/archive/science \
  --dq-root /data/archive/dq \
  --output-root /data/work \
  --target gband \
  --prefix c4d_ \
  --contains v1 \
  --existing fail
```

Typical MPI resume run:

```bash
mpirun -np 16 python init_program/init_program.py \
  --science-root /data/archive/science \
  --dq-root /data/archive/dq \
  --output-root /data/work \
  --target gband \
  --prefix c4d_ \
  --contains v1 \
  --existing resume
```

## 2. Published layout

```text
<output-root>/
├── <target>/
│   ├── science/<exposure>/<exposure>_<sequence>.fits
│   ├── dqmask/<exposure>/<exposure>_<CCDNUM>.fits
│   ├── expolists/<exposure>.list
│   ├── result/
│   ├── stamps/<product>/[<exposure>/]
│   └── astrometry/<product>/[<exposure>/]
├── expo_<target>.list
├── fits_<target>.list
└── init_<target>_manifest.json
```

The target directory is the dataset root seen by the F77 product-layout helpers. The top-level `expo_<target>.list` lives one level above that target directory and is the natural positional input to `Fourier_Quad_Pipe`.

## 3. Science and DQ naming

Science extraction and DQ extraction intentionally use different numbering rules:

- Science output basename suffix = occurrence number among extractable two-dimensional HDUs in the source archive.
- Science FITS headers retain their physical `CCDNUM` when present.
- DQ output basename suffix = physical `CCDNUM` from the DQ HDU header.
- DQ exposure stems map the archive naming convention from `ood` to the corresponding science `ooi` stem.

Therefore do not assume `science/<exposure>/<exposure>_N.fits` uses physical CCD identity. When code needs the physical mask identity, current F77 uses the chip/CCD identity and constructs `dqmask/<exposure>/<exposure>_<CCDNUM>.fits`.

## 4. Exposure-list contract

The top list contains one per-exposure list path plus chip count per record, for example:

```text
"/data/work/gband/expolists/exposure_001.list" 2
```

Each per-exposure list contains one science FITS path per nonblank line:

```text
/data/work/gband/science/exposure_001/exposure_001_1.fits
/data/work/gband/science/exposure_001/exposure_001_2.fits
```

Current F77 readers are defensive about this contract:

- the CLI positional exposure-list path is checked for missing/overlong input;
- blank records are skipped;
- malformed top-list records are rejected;
- per-chip paths longer than the current fixed string capacity are rejected rather than silently truncated;
- `NMAX_EXPO` and `NMAX_CHIP` bounds are enforced.

When using containers, every path written into the lists must be valid **inside the container**, not merely on the host.

## 5. Product directory contract

Both Full and Lite currently define the product directories in `path_layout.inc`:

```text
DIR_DQ              dqmask
DIR_NORM            stamps/Norm
DIR_CAT_ORIG        stamps/cat_Orig
DIR_STAR_CAN_INFO   stamps/dat_StarCanInfo
DIR_STAR_CAN        stamps/fits_StarCan
DIR_STAR_CAN_N      stamps/fits_StarCanN
DIR_STAR_CAN_P      stamps/fits_StarCanP
DIR_SRC_INFO        stamps/dat_SrcInfo
DIR_SRC             stamps/fits_Src
DIR_NOISE           stamps/fits_Noise
DIR_SRC_P           stamps/fits_SrcP
DIR_PSF_FIT         stamps/dat_PsfFit
DIR_PSF_LOCAL       stamps/fits_PsfLocal
DIR_SHEAR           stamps/dat_Shear
DIR_STAR_XY         stamps/dat_StarXY
DIR_PSF_RESI        stamps/fits_PsfResi
DIR_ASTRO_DATA      astrometry/dat_Astro
DIR_ASTRO_HEAD      astrometry/Head
DIR_ASTRO_CHECK     astrometry/dat_Chk
DIR_STAR_INFO       stamps/dat_StarInfo
DIR_STAR_P          stamps/fits_StarP
DIR_PSF_SRC         stamps/fits_PsfSrc
DIR_EXPO_INFO       stamps/dat_ExpoInfo
DIR_STAR_COMP       stamps/dat_StarComp
DIR_RESCALE         stamps/dat_Rescale
DIR_PCS             stamps/dat_Pcs
DIR_STAR_COMP_V2    stamps/dat_StarCompV2
DIR_RESULT          result
```

Do not duplicate these strings in new stage code unless there is a deliberate compatibility reason.

## 6. Path-construction helpers

Current `universal.f` centralizes product names with four helper shapes:

- `fq_chip_product_path(image_file, dir_output, product_dir, suffix, filename)` → `<dataset>/<product>/<exposure>/<chip-prefix><suffix>`
- `fq_expo_product_path(image_file, dir_output, product_dir, suffix, filename)` → `<dataset>/<product>/<exposure><suffix>`
- `fq_expo_ccd_product_path(image_file, dir_output, product_dir, ccd_id, suffix, filename)` → `<dataset>/<product>/<exposure>/<exposure>_<CCDNUM><suffix>`
- `fq_base_product_path(dir_output, product_dir, basename, filename)` → dataset-global product path

`fq_assign_path` rejects empty/overlong constructed paths before assignment. For implementation changes, extend the layout/helper layer first instead of restoring ad-hoc `trim(DIR_OUTPUT)//'/...'` concatenation across stage files.

## 7. Existing-output policies and atomic publication

Initializer modes:

- `fail`: reject an archive when any planned output already exists.
- `resume`: validate and reuse good outputs; regenerate missing or invalid outputs.

Chip outputs are first written below `<target>/.fq_init_tmp/<run>/` and then atomically moved into place on the same filesystem. Exposure lists, top lists, and the manifest are atomically replaced as well.

A failed source archive causes a nonzero final status and is recorded in `init_<target>_manifest.json`; successful exposures remain available for diagnosis and a later `--existing resume` run.

## 8. Manifest semantics

The current initializer writes a schema-versioned JSON manifest recording at least:

- status (`success`, `partial`, or `failed`);
- source roots and output root;
- target and filename filters;
- existing-output policy;
- science/DQ source and image counts;
- failed/partial sources and skipped HDUs;
- whether exposure lists and product directories were published;
- implementation/provenance details.

Treat the manifest as the initializer provenance record, not as an input consumed by F77 stages.

## 9. Full/Lite compatibility

The current reference `f77/` and `f77_Lite/` use the same dataset/product layout. The major Full/Lite differences are algorithmic/feature branches, not the filesystem contract. When porting a path-related fix between variants, inspect both trees and normally keep the layout/helper changes synchronized.
