# Direct HPC: Minimal Apptainer/Singularity + Slurm

## Scope

Use this reference when the user wants a directly runnable production Slurm job, not the complete runner/audit/smoke-test framework. Generate only the paths, binds, compile step, and launch command needed for the requested job.

For raw archive preparation and the current dataset layout, also load `dataset-01-initializer-layout.md`.

## 1. Inputs to resolve

A direct job normally needs:

- selected source variant: `f77/` or `f77_Lite/`;
- SIF path or OCI/GHCR image used to create it;
- processing-data host path and container destination;
- top `EXPO_LIST` container path;
- astrometry/source catalog binds;
- flat/external-PSF bind only when active;
- task/node count and site MPI launch mode;
- any site modules needed for Apptainer/Slurm.

Do not add unrelated runner variables.

## 2. Current F77 entry point

```bash
Fourier_Quad_Pipe <EXPO_LIST>
```

Do not emit C++-style flags for the F77 executable.

If the current initializer prepared target `gband` under `/data/work`, the host list is normally:

```text
/data/work/expo_gband.list
```

Inside a container, use the equivalent container-visible path.

## 3. Source bind

The host source may point to either variant and can be mounted at a common container build location such as `/workspace/f77`:

```bash
--bind "$F77_SOURCE_HOST:/workspace/f77"
```

The container destination does not imply that the source is Full; it is just a runtime mount point.

## 4. Data and catalog binds

Example conceptual binds:

```text
HOST processing tree  -> /data/DataProcess
HOST Gaia catalog     -> /data/Gaia:ro
HOST source catalog   -> /data/SourceCat:ro
```

The catalog destinations must exactly match the strings compiled into the selected `para.inc`.

Every path embedded inside exposure-list files must also be valid at the mounted container prefix.

## 5. Compile-once step

Before the MPI launch, compile once inside the container:

```bash
apptainer exec \
  --bind "$F77_SOURCE_HOST:/workspace/f77" \
  "$SIF" \
  bash -lc '
    cd /workspace/f77
    make clean
    make LAPACK_LIB_DIR=/opt/f77stack/lib \
         CFITSIO_LIB_DIR=/opt/f77stack/lib \
         CFITSIO_LIB=/opt/f77stack/lib/libcfitsio.so
  '
```

For a different toolchain, adapt the Makefile overrides to the actual image. Current Makefiles depend on `path_layout.inc`; layout changes therefore trigger rebuilds.

## 6. Direct launch pattern

A site using Slurm's container-compatible launch might conceptually run:

```bash
srun -n "$SLURM_NTASKS" \
  apptainer exec \
    --bind "$F77_SOURCE_HOST:/workspace/f77" \
    --bind "$PROCESS_DATA_HOST:/data/DataProcess" \
    --bind "$ASTROMETRY_CAT_HOST:/data/Gaia:ro" \
    --bind "$SOURCE_CAT_HOST:/data/SourceCat:ro" \
    "$SIF" \
    /workspace/f77/Fourier_Quad_Pipe \
    "$F77_EXPO_LIST_CONTAINER"
```

This is a pattern, not a universal MPI recipe. Some sites require host MPI integration, `--mpi` modes, PMI/PMIx configuration, or an outer `mpiexec`. Use the site's supported mode.

## 7. Optional initializer phase

When a single Slurm allocation should initialize data before running F77, keep initializer and pipeline steps separate and fail fast between them. Example structure:

```bash
srun -n "$INIT_TASKS" python /workspace/pipeline/init_program/init_program.py \
  --science-root "$SCIENCE_ARCHIVE" \
  --dq-root "$DQ_ARCHIVE" \
  --output-root "$PROCESS_DATA_HOST" \
  --target "$TARGET" \
  --existing resume \
  --no-srun

# then compile once and run Fourier_Quad_Pipe on expo_${TARGET}.list
```

Use `--no-srun` when the initializer is already launched inside an explicit Slurm step and the current script's CLI supports that mode; otherwise inspect the current initializer behavior before composing nested launchers.

## 8. Full/Lite cautions

For Lite, do not generate instructions to edit selectors that no longer exist (`include_FLAT`, `ext_cat`, `ext_PSF`, etc.). Full and Lite now share the same dataset/path layout, so input/bind layout guidance is common unless the current tree says otherwise.

## 9. What Direct mode should not add automatically

Unless requested, do not add:

- runner audit scripts;
- `run-apptainer.sh --check`;
- MPI smoke-test jobs;
- unused runner configuration fields;
- complex wrapper layers around a simple compile-and-run request.

Keep the generated job small enough that the user can compare every bind and command with the cluster configuration.
