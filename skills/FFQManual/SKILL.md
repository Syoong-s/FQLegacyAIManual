---
name: FFQManual
description: On-demand operating and developer knowledge for Fourier_Quad Pipeline Legacy (F77): current Full/Lite dataset layout, initializer, Docker/Apptainer/Slurm deployment, compile-time parameters, stage behavior, outputs, debugging, and source-code modification guidance. Use when the user asks to understand, configure, initialize, run, tune, debug, or modify a compatible Legacy Fourier Quad F77 pipeline.
---

# FFQManual — F77 Pipeline Router

This file is a routing index, not the full manual. Keep initial context small: load only the references required by the current task.

## Source-of-Truth Rules

1. **The user's actual source tree wins over this documentation.** Before changing code, inspect the user's current files and locate the relevant symbol, parameter, interface, layout helper, or runtime script. Paths such as `f77/main.f`, `f77_Lite/para.inc`, `init_program/init_program.py`, and `f77_docker/runner/...` describe the current reference layout and are navigation hints, not required checkout paths.
2. **Defaults in references are orientation values.** Before changing a parameter, verify its current definition in `para.inc`, `cust_para.inc`, `sig_para.inc`, or the equivalent current file.
3. **Treat dataset layout as an interface.** In the current reference implementation, Full and Lite share the WFST-compatible dataset/product layout through `path_layout.inc` and `fq_*_product_path` helpers. Do not reintroduce hard-coded legacy product paths without checking the current helpers.
4. **Do not assume Full and Lite are feature-equivalent.** `f77_Lite` freezes eight Full selectors and physically removes their unused branches; read the variant reference before proposing a Lite feature change.
5. **Compile-time edits require rebuilding** `Fourier_Quad_Pipe`.
6. **Never load all references at once.** For a normal task, use one routing reference plus one stage reference; add more only when the task crosses subsystem boundaries.

## Task Router

| User task | Load first | Then load only if needed |
|---|---|---|
| Prepare/decompress a dataset, create exposure lists, use `init_program.py`, understand `science/`, `dqmask/`, `expolists/`, `stamps/`, `astrometry/`, `result/` | [dataset-01-initializer-layout.md](references/dataset-01-initializer-layout.md) | Deployment reference if MPI/Slurm/container execution is involved |
| Understand overall F77 architecture / data flow | [pipeline-01-overview.md](references/pipeline-01-overview.md) | Dataset reference + one stage reference |
| Choose `f77` vs `f77_Lite` or port a change between them | [variant-01-full-vs-lite.md](references/variant-01-full-vs-lite.md) | Relevant stage reference + actual source files |
| Change/tune a compile-time parameter | [config-01-parameter-map.md](references/config-01-parameter-map.md) | Relevant stage reference; variant reference for Lite |
| Build or run locally / Docker Compose | [deployment-01-build-docker-runner.md](references/deployment-01-build-docker-runner.md) | Dataset reference if inputs are not already initialized |
| Generate Docker `.env` or a full reference-style `f77pipeline.env` / runner wrapper | [deployment-01-build-docker-runner.md](references/deployment-01-build-docker-runner.md) | User's current env/runner files when available |
| Generate a minimal directly runnable HPC Slurm job, SIF command, binds, and optional small env without runner checks | [deployment-02-direct-hpc-slurm.md](references/deployment-02-direct-hpc-slurm.md) | Dataset reference if initializer and pipeline are combined |
| Modify implementation / add feature / fix bug | [development-01-change-map.md](references/development-01-change-map.md) | Relevant stage reference + current source files |
| F6 noise/background/DQ behavior | [pipeline-02-preprocessing.md](references/pipeline-02-preprocessing.md) | Parameter map + dataset reference for DQ naming |
| Astrometry / Gaia / PU mapping | [pipeline-03-astrometry.md](references/pipeline-03-astrometry.md) | Dataset reference for current output directories |
| Source extraction / external catalog / deblending | [pipeline-04-source-detection.md](references/pipeline-04-source-detection.md) | Parameter map + variant reference |
| FFT / power spectrum / noise subtraction | [pipeline-05-power-spectrum.md](references/pipeline-05-power-spectrum.md) | Shear reference if estimator coupling matters |
| PSF selection/model/reconstruction | [pipeline-06-psf-modeling.md](references/pipeline-06-psf-modeling.md) | Variant reference + parameter map |
| Fourier_Quad estimator / rotations | [pipeline-07-shear-estimation.md](references/pipeline-07-shear-estimation.md) | PSF reference if deconvolution is involved |
| Catalog columns / Stage 8–9 calibration | [pipeline-08-catalog.md](references/pipeline-08-catalog.md) | Auxiliary measurement refs only for PDF-symmetry analysis |
| PDF sign-test / FD calibration auxiliary program | [measurement-00-scope.md](references/measurement-00-scope.md) | Then the matching `measurement-0*.md` file |

## Pipeline Stage Index

`PROCESS_stage` is a product-of-primes selector. A stage runs when `mod(PROCESS_stage, prime) == 0`.

| Stage | Prime | Main source | Detailed reference |
|---:|---:|---|---|
| 1 | 2 | `pre_process.f` | [pipeline-02-preprocessing.md](references/pipeline-02-preprocessing.md) |
| 2 | 3 | `proc_astrometry.f` | [pipeline-03-astrometry.md](references/pipeline-03-astrometry.md) |
| 3 | 5 | `proc_source.f` | [pipeline-04-source-detection.md](references/pipeline-04-source-detection.md) |
| 4 | 7 | `proc_FFT_st1.f` | [pipeline-05-power-spectrum.md](references/pipeline-05-power-spectrum.md) |
| 5 | 11 | `proc_PSF.f` | [pipeline-06-psf-modeling.md](references/pipeline-06-psf-modeling.md) |
| 6 | 13 | `proc_FFT_st2.f` | [pipeline-05-power-spectrum.md](references/pipeline-05-power-spectrum.md) |
| 7 | 17 | `proc_shear.f` | [pipeline-07-shear-estimation.md](references/pipeline-07-shear-estimation.md) |
| 8 | 19 | `proc_info.f` | [pipeline-08-catalog.md](references/pipeline-08-catalog.md) |
| 9 | 23 | `proc_combine_shear_catalog.f` | [pipeline-08-catalog.md](references/pipeline-08-catalog.md) |

## Minimum-Load Recipes

- **Initialize a dataset:** `dataset-01-initializer-layout` only; add a deployment reference for MPI/Slurm/container execution.
- **Explain one stage:** `pipeline-01-overview` + that stage reference; add dataset reference only when paths or products matter.
- **Tune one parameter:** `config-01-parameter-map` + owning stage reference. For Lite frozen selectors, also load `variant-01-full-vs-lite`.
- **Implement a code change:** `development-01-change-map` + owning stage reference + the actual current source file(s). Do not edit from documentation alone.
- **Change product paths:** inspect `path_layout.inc` and `universal.f` path helpers first. Avoid stage-local string concatenation unless the current source intentionally does so.
- **Generate a full runner-style file:** `deployment-01-build-docker-runner` + the user's current env/runner file when available.
- **Generate a direct HPC job:** `deployment-02-direct-hpc-slurm`; add dataset reference when `init_program.py` is part of the job.
- **Debug output:** start from the earliest stage that produced a bad/missing product, not from Stage 9.

## Hard Invariants Worth Keeping in Initial Context

- F77 entry point accepts the top exposure-list path as its first positional argument: `Fourier_Quad_Pipe <EXPO_LIST>`.
- Current reference `main.f` validates the positional argument and exposure-list records rather than silently truncating them; `strl` is currently 512 in both variants.
- The initializer publishes `<output-root>/expo_<target>.list`; this is the natural F77 entry list for the initialized target.
- The current reference layout is `<target>/science/<exposure>/...`, `<target>/dqmask/<exposure>/...`, `<target>/expolists/...`, plus `stamps/`, `astrometry/`, and `result/` products.
- Full and Lite currently share `path_layout.inc` and the `fq_chip_product_path`, `fq_expo_product_path`, `fq_expo_ccd_product_path`, and `fq_base_product_path` construction model.
- Full `f77` supports configurable `ASTROMETRY_trivial`, `include_FLAT`, `include_Mask`, `ext_cat`, `ext_PSF`, `deblending`, `PSF_type`, and `PSF_Ms`.
- `f77_Lite` freezes those eight behaviors to `0, 0, 2, 1, 0, 1, 1, 0` respectively and removes the alternate branches.
- Full `f77` contains `00_psf_module.f` and optional PCA/multi-scale PSF reconstruction; Lite removes that path.
- In current Stage 1, DQ is loaded before `set_background`/`set_sig`; `set_sig` does not perform DQ file I/O itself, but its input `weight` may already exclude DQ pixels.
