# Direct HPC Slurm Mode — Minimal Production Job Generation

## Scope and intent

Use this reference when the user asks for a **directly runnable HPC job** and explicitly does not want the full reference-style `runner/` validation framework. Typical requests include:

- "直接生成可以 sbatch 的脚本"
- "不要 runner wrapper / smoke test / audit"
- "给我拉镜像、生成 SIF、挂载目录并运行 pipeline"
- "给我一个最小的 production Slurm"
- "根据这些输入输出路径生成 env + Slurm"

This mode is intentionally separate from `deployment-01-build-docker-runner.md`.

**Direct mode optimizes for a short, transparent execution chain, not for reproducing every defensive check in `f77_docker/runner/`.**

## 1. Mode-selection rule

Choose **Direct HPC mode** when the user wants the final runnable commands/files rather than a full runner framework.

Do **not** automatically add any of the following in Direct mode:

- `inspect-cluster-mpi.sh`
- `run-apptainer.sh --check`
- `mpi-smoke-test.slurm`
- wrapper-level `die()`/path-validation frameworks
- compiler/library version audits
- MPI ABI diagnostic scripts
- temporary smoke-test programs
- duplicated runner indirection

Only add diagnostics if the user asks for them or if a known unresolved cluster-compatibility problem makes the requested production command impossible to specify safely.

If the user explicitly asks to reproduce or modify a full runner-style workflow, route to `deployment-01-build-docker-runner.md` instead.

## 2. Required execution chain

A normal Direct-mode deliverable should implement only this chain:

```text
OCI/GHCR image
    -> acquire SIF with Apptainer/Singularity
    -> define host/container paths
    -> bind required directories
    -> compile the selected f77 or f77_Lite source once
    -> launch allocated MPI ranks with the site-appropriate launcher
    -> Fourier_Quad_Pipe <EXPO_LIST>
```

The AI may provide:

1. one SIF acquisition command;
2. one small env file plus one Slurm script; or
3. one self-contained Slurm script with variables embedded at the top.

If the user asks for "one file" or does not want an env file, embed the variables directly in the Slurm script. Do not force the reference `f77pipeline.env` schema on Direct mode.

## 3. Information to resolve

Use values supplied by the user. If a nonessential value is missing, use a conspicuous `<PLACEHOLDER>` rather than loading unrelated pipeline references.

Resolve:

- source variant: `f77` or `f77_Lite`;
- OCI/GHCR image URI, preferably a digest for immutable production use;
- final shared SIF path;
- shared host source path;
- processing/input-output host path(s);
- astrometry catalogue host path when required;
- external source catalogue host path when required;
- flat/calibration host path when required by the selected Full-pipeline configuration;
- any external PSF path when Full `ext_PSF=1` is used;
- exposure-list host/container path;
- container destination paths;
- Slurm partition, nodes, total tasks, tasks/node, memory, walltime, account/QoS if applicable;
- cluster module names and MPI launch mode.

Do not ask for values already present in the user's request or current conversation.

## 4. SIF acquisition

For a published OCI/GHCR image, a normal non-root command is:

```bash
module load <apptainer-module-if-needed>
apptainer pull /shared/path/f77pipeline.sif \
    docker://ghcr.io/OWNER/REPOSITORY:TAG
```

A digest is preferred for production:

```bash
apptainer pull /shared/path/f77pipeline.sif \
    docker://ghcr.io/OWNER/REPOSITORY@sha256:DIGEST
```

`apptainer build output.sif docker://...` is also valid when the user specifically asks to *build* the SIF rather than pull it. Do not add `sudo`; the reference HPC pattern is non-root unless the user's site explicitly requires otherwise.

If the SIF already exists and the user only asked for the Slurm job, do not regenerate it.

## 5. Canonical container paths

When the user's runtime layout does not already define container destinations, the following **reference container paths** are convenient defaults:

```text
source          /workspace/f77
astrometry      /data/catalogs/AstroDir
source catalog  /data/catalogs/ExtSrcDir
flat/calib      /data/calib/FlatDir
processing data /data/DataProcess
```

The source host path may point to either Full `f77/` or `f77_Lite/`; both can be mounted at `/workspace/f77` because that is a runtime destination, not a variant name.

## 6. Compile-time path coupling

The strings compiled into `para.inc` must match the paths visible inside the SIF.

At minimum verify the current selected source before finalizing a job:

- `ASTROMETRY_CAT`
- `SOURCE_CAT`
- `FLAT_PATH` when Full `include_FLAT` uses it
- `PSF_PATH` when Full `ext_PSF=1`

If those values do not match the intended bind destinations, either:

1. change `para.inc` to the intended container paths and rebuild; or
2. bind the host directories to the exact paths already compiled into `para.inc`.

Do not silently bind a catalogue to `/data/catalogs/...` when the executable is compiled to read an unrelated `/lustre/...` path.

For `f77_Lite`, remember that `include_FLAT=0`, `ext_cat=1`, and `ext_PSF=0` are frozen behaviors; do not emit instructions to edit those removed selectors.

## 7. Minimal env format

When a separate env file is useful, keep it small and task-specific. Example:

```bash
F77_SIF=/shared/project/f77pipeline.sif

F77_SOURCE_HOST=/shared/project/f77_Lite
F77_SOURCE_CONTAINER=/workspace/f77

ASTROMETRY_CAT_HOST=/shared/catalogs/gaia
ASTROMETRY_CAT_CONTAINER=/data/catalogs/AstroDir

SOURCE_CAT_HOST=/shared/catalogs/sources
SOURCE_CAT_CONTAINER=/data/catalogs/ExtSrcDir

PROCESS_DATA_HOST=/shared/project/run01
PROCESS_DATA_CONTAINER=/data/DataProcess
F77_EXPO_LIST_CONTAINER=/data/DataProcess/expo_list.list

F77_EXECUTABLE=/workspace/f77/Fourier_Quad_Pipe
```

Only include `FLAT_PATH_*`, external PSF, or other binds when the selected configuration needs them.

Unlike full runner-style mode, Direct mode does **not** require placeholder variables for unused runner checks.

## 8. Bind construction

Build the Apptainer arguments directly in the job. Typical mappings are:

```bash
--bind "${F77_SOURCE_HOST}:${F77_SOURCE_CONTAINER}:rw"
--bind "${ASTROMETRY_CAT_HOST}:${ASTROMETRY_CAT_CONTAINER}:ro"
--bind "${SOURCE_CAT_HOST}:${SOURCE_CAT_CONTAINER}:ro"
--bind "${PROCESS_DATA_HOST}:${PROCESS_DATA_CONTAINER}:rw"
```

Add only the binds actually required by the selected pipeline configuration.

Use read-only mounts for catalogues/calibration where possible. The source mount must be writable if the job compiles into that directory. Processing/output data normally needs `rw`.

If input and output are separate host directories, bind both explicitly to distinct container paths and ensure all pipeline paths/exposure-list entries refer to those container destinations.

## 9. Compile once, not once per MPI rank

Compilation must happen once before the parallel launch:

```bash
apptainer exec \
    <bind arguments> \
    "${F77_SIF}" \
    make -C "${F77_SOURCE_CONTAINER}" -j4
```

Do not put `make` inside the `srun`/`mpiexec` rank launch.

If the user requests a clean rebuild, run `make clean` once before `make`. Otherwise avoid adding `make clean` by default because it can create unnecessary shared-source races and rebuild cost.

## 10. MPI launch choice

### Validated pilogin-style mode

The reference deployment material records one successful site configuration in which the host loads GCC 12.3.0 and OpenMPI 4.1.6, but **Slurm PMI2**, not host OpenMPI `mpirun`, starts the MPICH-linked program in the SIF. Treat these module names and launcher details as site-specific evidence, not universal defaults:

```bash
module load gcc/12.3.0
module load openmpi/4.1.6-gcc-12.3.0

unset OMPI_MCA_mtl
unset OMPI_MCA_osc

srun --mpi=pmi2 --ntasks="${SLURM_NTASKS}" \
    apptainer exec <binds> "${F77_SIF}" \
    "${F77_EXECUTABLE}" "${F77_EXPO_LIST_CONTAINER}"
```

Do not replace this with host OpenMPI `mpirun` for the MPICH-linked image.

### Generic MPICH hybrid mode

On a site with compatible host MPICH, a direct hybrid launch can instead be:

```bash
mpiexec -n "${SLURM_NTASKS}" \
    apptainer exec <binds> "${F77_SIF}" \
    "${F77_EXECUTABLE}" "${F77_EXPO_LIST_CONTAINER}"
```

Do not invent a launcher when the user's cluster contract explicitly provides one. If no cluster MPI information exists, label the launcher line as site-dependent rather than presenting an unverified choice as universal.

## 11. Slurm resource rules

Keep allocation simple and explicit. For this MPI pipeline, the normal model is one CPU per MPI rank:

```bash
#SBATCH --partition=<partition>
#SBATCH --nodes=<nodes>
#SBATCH --ntasks=<total-ranks>
#SBATCH --ntasks-per-node=<ranks-per-node>
#SBATCH --cpus-per-task=1
```

Require or preserve:

```text
ntasks = nodes * ntasks-per-node
```

unless the user intentionally requests uneven placement.

Respect any site limits supplied by the user or another active cluster-specific skill. Site-specific allocation constraints override generic examples in this reference.

Do not add `--exclusive`, memory, account, QoS, mail settings, or long validation prologues unless requested or already required by the known cluster policy.

## 12. Minimal direct Slurm pattern

A typical Direct-mode script should resemble this structure:

```bash
#!/usr/bin/env bash
#SBATCH --job-name=fq-f77
#SBATCH --partition=<PARTITION>
#SBATCH --nodes=<NODES>
#SBATCH --ntasks=<NTASKS>
#SBATCH --ntasks-per-node=<TASKS_PER_NODE>
#SBATCH --cpus-per-task=1
#SBATCH --time=<TIME>
#SBATCH --output=fq-%j.out
#SBATCH --error=fq-%j.err

set -euo pipefail

module purge
module load <compiler/module stack>
module load <apptainer module if needed>

source /shared/project/f77-direct.env

BINDS=(
  --bind "${F77_SOURCE_HOST}:${F77_SOURCE_CONTAINER}:rw"
  --bind "${ASTROMETRY_CAT_HOST}:${ASTROMETRY_CAT_CONTAINER}:ro"
  --bind "${SOURCE_CAT_HOST}:${SOURCE_CAT_CONTAINER}:ro"
  --bind "${PROCESS_DATA_HOST}:${PROCESS_DATA_CONTAINER}:rw"
)

apptainer exec "${BINDS[@]}" "${F77_SIF}" \
  make -C "${F77_SOURCE_CONTAINER}" -j4

unset OMPI_MCA_mtl OMPI_MCA_osc
srun --mpi=pmi2 --ntasks="${SLURM_NTASKS}" \
  apptainer exec "${BINDS[@]}" "${F77_SIF}" \
  "${F77_EXECUTABLE}" "${F77_EXPO_LIST_CONTAINER}"
```

This is a **shape**, not a source of universal site defaults. Replace module names, resource values, optional binds, and launcher mode from the user's actual cluster information.

## 13. Self-contained one-file mode

If the user asks for one script only, put all path variables immediately below `set -euo pipefail` and omit `source ...env` entirely. Example pattern:

```bash
F77_SIF=/shared/project/f77pipeline.sif
F77_SOURCE_HOST=/shared/project/f77_Lite
F77_SOURCE_CONTAINER=/workspace/f77
PROCESS_DATA_HOST=/shared/project/run01
PROCESS_DATA_CONTAINER=/data/DataProcess
F77_EXPO_LIST_CONTAINER=/data/DataProcess/expo_list.list
F77_EXECUTABLE=/workspace/f77/Fourier_Quad_Pipe
```

Direct mode should adapt to the requested packaging instead of insisting on reference runner file boundaries.

## 14. Output contract for the AI

When the user asks to "generate the HPC job", return the actual runnable artifacts, not merely instructions about what they should contain.

Unless the user asks for another format, produce in this order:

1. **SIF acquisition command** — only if a SIF still needs to be created;
2. **env file** — only if useful/requested;
3. **Slurm script** — complete and directly submit-able after replacing any explicit placeholders;
4. **submit command**, usually `sbatch <script>`.

Keep explanatory prose short. Do not append the full runner validation workflow after a Direct-mode deliverable.

## 15. Direct-mode final consistency check

Before emitting the files, verify internally:

- selected source variant is correct;
- F77 uses positional `EXPO_LIST`, not C++ CLI flags;
- all compiled catalogue/calibration paths match bind destinations;
- every host path used by multiple nodes is on shared storage;
- source is `rw` if compiling there;
- output/processing path is `rw`;
- compile happens once before MPI launch;
- total ranks and ranks/node are consistent;
- launcher is compatible with the known site contract;
- no unwanted runner audit/smoke/check wrapper has leaked into Direct mode.
