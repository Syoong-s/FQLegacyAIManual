# Build, Docker, and Full Runner-Style Deployment

## Scope

Use this reference for local builds, Docker Compose, or the repository-style defensive HPC runner workflow. For a short self-contained Slurm job without runner checks, use `deployment-02-direct-hpc-slurm.md`. For dataset preparation, load `dataset-01-initializer-layout.md`.

## 1. Build contract

Both `f77/` and current `f77_Lite/` build the executable `Fourier_Quad_Pipe`. Their Makefiles currently depend on:

- Fortran source files;
- `para.inc`;
- `cust_para.inc`;
- `sig_para.inc`;
- `path_layout.inc`.

Typical modern GNU build:

```bash
export SCIENCE_PREFIX=/path/to/scientific-stack
make -C f77 \
  FC=mpifort \
  FFLAGS='-mcmodel=medium -w -fallow-argument-mismatch' \
  LAPACK_LIB_DIR="$SCIENCE_PREFIX/lib" \
  CFITSIO_LIB_DIR="$SCIENCE_PREFIX/lib"
```

Replace `f77` with `f77_Lite` as needed. If CFITSIO is not discoverable through the directory override, pass the explicit library path supported by the current Makefile.

The recorded legacy container toolchain uses GNU Fortran/MPICH/CFITSIO/LAPACK versions chosen for reproducibility; use the repository's actual Dockerfile as authority for exact versions.

## 2. Native run

```bash
mpirun -np 4 ./f77/Fourier_Quad_Pipe /data/work/expo_gband.list
```

The first positional argument is the **top exposure list**, not a per-exposure chip list.

If the dataset was built by the current initializer, the natural input is:

```text
<output-root>/expo_<target>.list
```

## 3. Container path rule

Paths compiled into `para.inc` (`ASTROMETRY_CAT`, `SOURCE_CAT`, `FLAT_PATH`, optional external PSF path) must match the paths visible **inside** the container.

Likewise, paths stored inside `expo_<target>.list` and every `expolists/<exposure>.list` must be container-visible paths. A correct host file containing host-only absolute paths will still fail inside the container.

## 4. Docker Compose reference pattern

Typical workflow:

```bash
cd f77_docker
cp .env.example .env
# edit .env

docker compose pull    # or build, according to current repository workflow
docker compose run --rm FourierQuad-F77
```

Set at least:

- image name/tag;
- `F77_SOURCE_HOST` pointing to `f77/` or `f77_Lite/`;
- processing-data host path;
- catalog/calibration host paths;
- matching container destinations;
- host UID/GID when required.

Do not copy an old `.env` blindly: compare it with the current repository example because path/layout variables can evolve.

## 5. Initializer in container workflows

The initializer reads source `.fits.fz` archives and publishes the current F77-compatible layout. You may run it on the host or in an appropriate Python environment before launching the Fortran container.

For example:

```bash
python init_program/init_program.py \
  --science-root /archive/science \
  --dq-root /archive/dq \
  --output-root /data/work \
  --target gband \
  --existing resume
```

Then ensure `/data/work` is bound at the same container-visible prefix encoded in the generated list files, or regenerate lists using container-visible paths.

## 6. Full runner-style HPC workflow

Use the repository's `f77_docker/runner/` framework when the user wants its audit/check/smoke-test behavior rather than a minimal job. A typical current pattern is:

```bash
cd f77_docker/runner
cp f77pipeline.env.example f77pipeline.env
# edit site paths, image/SIF, source and data binds, MPI mode

bash run-apptainer.sh --check
sbatch mpi-smoke-test.slurm
sbatch f77pipeline.slurm
```

Treat these filenames as reference-layout hints; inspect the user's current runner directory before generating exact files.

## 7. MPI compatibility

Container MPI launch is site-specific. Do not assume a host OpenMPI launcher is ABI-compatible with an MPICH application merely because both provide `mpiexec`. Use the repository's runner documentation and the cluster's supported PMI/PMIx/Slurm launch mode.

For multi-node jobs, validate MPI separately before attributing launch failures to F77 stage code.

## 8. Build once, run many ranks

Compile the selected bind-mounted source once before launching the multi-rank executable. Do not let every MPI rank race to rebuild the same executable.

Any edit to compile-time includes—including `path_layout.inc` and path strings in `para.inc`—requires rebuilding before execution.

## 9. Troubleshooting order

1. validate list paths and container-visible prefixes;
2. confirm initializer/product layout when files are missing;
3. confirm catalog/calibration binds match compiled paths;
4. clean-build the selected variant;
5. test a single rank/small exposure set;
6. validate MPI launcher compatibility;
7. only then debug stage numerics.
