# Build, Docker, Apptainer, Slurm, and Runner Generation

## Scope

Use this reference for source builds, local Docker Compose, and the **reference Legacy F77** `f77_docker/runner` pattern: `f77pipeline.env`, wrapper scripts, validation flow, and runner-compatible Slurm allocation. These names describe a reference layout, not a required repository structure. Prefer the user's current runner/env files when they exist, and adapt the reference pattern when they do not.

If the user explicitly wants a **minimal directly runnable Slurm job without runner audits, smoke tests, or wrapper validation**, do not use this as the primary reference; route to `deployment-02-direct-hpc-slurm.md`.

## 1. Native/container build contract

The F77 Makefile uses:

- compiler wrapper: `mpif77`
- target: `Fourier_Quad_Pipe`
- source set: all `*.f` in the selected source directory
- required includes: `para.inc`, `cust_para.inc`, `sig_para.inc`
- flags: `-mcmodel=medium -w`
- libraries: LAPACK, BLAS, and CFITSIO
- overridable Make variables: `LAPACK_LIB_DIR`, `CFITSIO_LIB_DIR`, `CFITSIO_LIB`

Changing any include parameter requires recompilation. Full `f77` may additionally emit/use a module file for `00_psf_module.f`; Lite omits that module path.

## 2. F77 invocation contract

The executable consumes the exposure list as its first positional argument:

```bash
mpirun -np N ./Fourier_Quad_Pipe /path/to/expo_list.list
```

Do not use the C++ pipeline's `--expo-list` / `--run-main` flags for F77.

The exposure list reader expects at least two whitespace-separated fields per line: exposure name/path and chip count. The current `initialize` routine stores the first field as the exposure identifier and reads the second into `nchip`.

## 3. Local Docker Compose `.env`

Current variables:

```text
IMAGE_NAME
BASE_IMAGE
BUILD_JOBS
HOST_UID
HOST_GID
F77_SOURCE_HOST
ASTROMETRY_CAT_HOST
ASTROMETRY_CAT_CONTAINER
SOURCE_CAT_HOST
SOURCE_CAT_CONTAINER
FLAT_PATH_HOST
FLAT_PATH_CONTAINER
PROCESS_DATA_HOST
PROCESS_DATA_CONTAINER
```

Canonical container destinations are:

```text
/workspace/f77
/data/catalogs/AstroDir
/data/catalogs/ExtSrcDir
/data/calib/FlatDir
/data/DataProcess
```

`compose.yaml` binds source and processing data read-write; astrometry/source/flat inputs are read-only.

### Generate a Docker `.env`

When the user gives host paths, preserve the canonical container paths unless there is a concrete reason to change them. Then ensure `para.inc` uses the same container strings for `ASTROMETRY_CAT`, `SOURCE_CAT`, and `FLAT_PATH`.

A safe generated skeleton is:

```bash
IMAGE_NAME=f77pipeline-dev:gnu4.8.5
BASE_IMAGE=quay.io/rockylinux/rockylinux:8.10@sha256:e8a49c5403b687db05d4d67333fa45808fbe74f36e683cec7abb1f7d0f2338c6
BUILD_JOBS=4
HOST_UID=<uid>
HOST_GID=<gid>

F77_SOURCE_HOST=<absolute host path to f77 or f77_Lite>
ASTROMETRY_CAT_HOST=<absolute host astrometry path>
ASTROMETRY_CAT_CONTAINER=/data/catalogs/AstroDir
SOURCE_CAT_HOST=<absolute host source-catalog path>
SOURCE_CAT_CONTAINER=/data/catalogs/ExtSrcDir
FLAT_PATH_HOST=<absolute host flat/calibration path>
FLAT_PATH_CONTAINER=/data/calib/FlatDir
PROCESS_DATA_HOST=<absolute host processing-data path>
PROCESS_DATA_CONTAINER=/data/DataProcess
```

## 4. HPC runtime model

On Slurm HPC, Docker Compose is not used at runtime. The current design is:

1. Build/publish OCI image elsewhere.
2. Pull/convert to immutable SIF as a non-root user.
3. Bind source, catalogs, calibration, and processing data into the SIF.
4. Compile the bind-mounted source once in the batch allocation.
5. Launch one `apptainer exec`/`singularity exec` command per MPI rank using host `mpiexec` or a validated `srun` PMI mode.

The source and all host input paths must be visible at the same absolute location on every allocated compute node.

## 5. `runner/f77pipeline.env` contract

Required/current fields are:

```text
OCI_IMAGE_URI
F77_SIF
F77_SOURCE_HOST
F77_SOURCE_CONTAINER
ASTROMETRY_CAT_HOST
ASTROMETRY_CAT_CONTAINER
SOURCE_CAT_HOST
SOURCE_CAT_CONTAINER
FLAT_PATH_HOST
FLAT_PATH_CONTAINER
PROCESS_DATA_HOST
PROCESS_DATA_CONTAINER
F77_EXPO_LIST_CONTAINER
HPC_SHARED_SCRATCH_HOST
APPTAINER_BIN
HPC_EXTRA_BINDS
FI_PROVIDER
FI_PROVIDER_PATH
HPC_SCRUB_OPENMPI_ENV
HPC_MODULES
SITE_ENV_SCRIPT
MPI_LAUNCH_MODE
MPI_LAUNCHER
SLURM_MPI_TYPE
F77_BUILD_JOBS
F77_MAKE_CLEAN
F77_EXECUTABLE
```

`run-apptainer.sh` currently validates all source/catalog/flat/data host directories even if the selected Full compile-time branch will not use one of them. Do not omit a required env field merely because `include_FLAT=0` or `ext_cat=0`; either provide a valid directory or intentionally modify the runner contract.

### Generic generated HPC env

```bash
OCI_IMAGE_URI=<ghcr image tag or preferably digest>
F77_SIF=<absolute shared path to .sif>

F77_SOURCE_HOST=<absolute shared path to f77 or f77_Lite>
F77_SOURCE_CONTAINER=/workspace/f77

ASTROMETRY_CAT_HOST=<absolute shared astrometry dir>
ASTROMETRY_CAT_CONTAINER=/data/catalogs/AstroDir
SOURCE_CAT_HOST=<absolute shared source catalog dir>
SOURCE_CAT_CONTAINER=/data/catalogs/ExtSrcDir
FLAT_PATH_HOST=<absolute shared flat/calibration dir>
FLAT_PATH_CONTAINER=/data/calib/FlatDir
PROCESS_DATA_HOST=<absolute shared processing dir>
PROCESS_DATA_CONTAINER=/data/DataProcess
F77_EXPO_LIST_CONTAINER="${PROCESS_DATA_CONTAINER%/}/expo_list.list"

HPC_SHARED_SCRATCH_HOST="${PROCESS_DATA_HOST}"
APPTAINER_BIN=
HPC_EXTRA_BINDS=
FI_PROVIDER=
FI_PROVIDER_PATH=
HPC_SCRUB_OPENMPI_ENV=1
HPC_MODULES=()
SITE_ENV_SCRIPT=

MPI_LAUNCH_MODE=mpiexec
MPI_LAUNCHER=mpiexec
SLURM_MPI_TYPE=pmi2

F77_BUILD_JOBS=4
F77_MAKE_CLEAN=1
F77_EXECUTABLE="${F77_SOURCE_CONTAINER%/}/Fourier_Quad_Pipe"
```

### Validated pilogin OpenMPI-module mode

The reference runner material includes a pilogin example that loads:

```bash
HPC_MODULES=(gcc/12.3.0 openmpi/4.1.6-gcc-12.3.0)
MPI_LAUNCH_MODE=srun
MPI_LAUNCHER=
SLURM_MPI_TYPE=pmi2
HPC_SCRUB_OPENMPI_ENV=1
```

This does **not** mean OpenMPI launches the MPICH-linked application. Slurm PMI2 starts the container command per rank; host OpenMPI `mpirun` must not be used for the MPICH-linked executable.

## 6. Bind-path consistency rules

`run-apptainer.sh` makes these binds:

| Host | Container | Access |
|---|---|---|
| `F77_SOURCE_HOST` | `F77_SOURCE_CONTAINER` | rw |
| `ASTROMETRY_CAT_HOST` | `ASTROMETRY_CAT_CONTAINER` | ro |
| `SOURCE_CAT_HOST` | `SOURCE_CAT_CONTAINER` | ro |
| `FLAT_PATH_HOST` | `FLAT_PATH_CONTAINER` | ro |
| `PROCESS_DATA_HOST` | `PROCESS_DATA_CONTAINER` | rw |

Extra mappings can be passed by `--bind` or `HPC_EXTRA_BINDS`.

### External PSF special case

Full `f77` with `ext_PSF=1` reads a PSF path compiled into `PSF_PATH`, but the standard runner has no dedicated `PSF_PATH_HOST/CONTAINER` pair. Generate an explicit additional read-only bind, for example:

```bash
HPC_EXTRA_BINDS=/host/psf:/data/catalogs/PSF:ro
```

and compile `PSF_PATH` to `/data/catalogs/PSF` (or the exact expected subpath). Lite cannot select `ext_PSF=1` without reintroducing removed code.

## 7. Slurm resource allocation

Allocation directives live in `.slurm`, not in `f77pipeline.env`. The generic template currently defaults to `cpu`, 2 nodes, 8 ranks, 4 ranks/node. The pilogin wrapper currently defaults to `cpu`, `--exclusive`, 2 nodes, 4 ranks, 2 ranks/node.

When generating a site-specific Slurm file:

- keep `--ntasks = nodes * ntasks-per-node` unless deliberately uneven;
- keep `--cpus-per-task=1` for this MPI process model unless code/threading changes;
- respect site-specific per-node and total-core limits;
- edit/override partition, account/QoS, memory, walltime, logs together as needed;
- do not put scheduler allocation settings into `f77pipeline.env` and assume Slurm will read them.

## 8. Validation sequence before production

1. `inspect-cluster-mpi.sh` on a new cluster (read-only audit).
2. Pull/transfer SIF.
3. `run-apptainer.sh --check` for compiler/MPI/CFITSIO and bind validation.
4. Single rank.
5. Multiple ranks on one node.
6. One rank on each of two nodes.
7. Multiple ranks across two nodes using `mpi-smoke-test.slurm`.
8. Representative pipeline run and performance/fabric validation.

Do not infer multi-node compatibility from version strings or a local Docker test alone.

## 9. Concurrency hazard

The batch template can run `make clean` and rebuild the **shared bind-mounted source tree**. Two concurrent jobs using the same source directory can race or delete each other's build products. For concurrent development/runs, use separate source/build directories or set a deliberate no-clean policy after verifying the executable.

## Runner-generation checklist for an AI

Before writing the final env/slurm file, resolve or use visible placeholders for:

- Full vs Lite source directory;
- OCI image URI/digest and SIF path;
- five required shared host directories;
- whether compiled container paths already match `para.inc`;
- exposure-list filename/location;
- generic MPICH `mpiexec` vs validated `srun` PMI mode;
- cluster modules/runtime command;
- nodes, ranks, ranks/node, memory, time, partition/account/QoS;
- external PSF or RDMA extra binds.

If only host paths are missing, generate a complete file with explicit `<...>` placeholders rather than loading unrelated algorithm references.
