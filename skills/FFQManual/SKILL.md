---
name: FFQManual
description: On-demand background and operating knowledge for the Fourier_Quad Pipeline Legacy (F77): Full/Lite reference layouts, Docker/Apptainer/Slurm deployment, runner patterns, compile-time parameters, stage behavior, outputs, debugging, and source-code modification guidance. Use when the user asks to understand, configure, run, tune, debug, or modify a compatible Legacy Fourier Quad F77 pipeline.
---

# FFQManual — F77 Pipeline Router

This file is a **routing index**, not the pipeline manual. Keep initial context small: load only the references required by the current task.

## Source-of-Truth Rules

1. **The user's actual source tree wins over this documentation.** Before changing code, inspect the user's current files and locate the relevant symbol, parameter, interface, or runtime script. Paths such as `f77/main.f`, `f77_Lite/para.inc`, and `f77_docker/runner/...` describe the reference Legacy F77 layout and are navigation hints, not required repository paths.
2. **Defaults in references are orientation values.** Before changing a parameter, verify its current definition in `para.inc`, `cust_para.inc`, or `sig_para.inc`.
3. **Do not assume Full and Lite are interchangeable.** `f77_Lite` freezes eight Full-pipeline selectors and physically removes their unused branches; read the variant reference before proposing a Lite change involving those features.
4. **Compile-time parameter edits require rebuilding** `Fourier_Quad_Pipe`.
5. **Never load all references at once.** For a normal task, use one routing reference plus one stage reference; add more only when the task crosses subsystem boundaries.

## Task Router

| User task | Load first | Then load only if needed |
|---|---|---|
| Understand overall F77 architecture / data flow | [pipeline-01-overview.md](references/pipeline-01-overview.md) | One stage reference below |
| Choose `f77` vs `f77_Lite` or port a change between them | [variant-01-full-vs-lite.md](references/variant-01-full-vs-lite.md) | Relevant stage reference + actual source files |
| Change/tune a compile-time parameter | [config-01-parameter-map.md](references/config-01-parameter-map.md) | Relevant stage reference; variant reference for Lite |
| Build or run locally | [deployment-01-build-docker-runner.md](references/deployment-01-build-docker-runner.md) | [pipeline-01-overview.md](references/pipeline-01-overview.md) if stage selection matters |
| Generate Docker `.env` or a full reference-style `f77pipeline.env` / runner wrapper | [deployment-01-build-docker-runner.md](references/deployment-01-build-docker-runner.md) | User's current env/runner files when available |
| Generate a minimal directly runnable HPC Slurm job, SIF command, binds, and optional small env without runner checks | [deployment-02-direct-hpc-slurm.md](references/deployment-02-direct-hpc-slurm.md) | Parameter map only if compiled paths/settings must change |
| Modify implementation / add feature / fix bug | [development-01-change-map.md](references/development-01-change-map.md) | Relevant stage reference + current source files |
| F6 noise/background/mask behavior | [pipeline-02-preprocessing.md](references/pipeline-02-preprocessing.md) | [config-01-parameter-map.md](references/config-01-parameter-map.md) |
| Astrometry / Gaia / PU mapping | [pipeline-03-astrometry.md](references/pipeline-03-astrometry.md) | Parameter map |
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

- **Explain one stage:** `pipeline-01-overview` + that stage reference.
- **Tune one parameter:** `config-01-parameter-map` + owning stage reference. For Lite frozen selectors, also load `variant-01-full-vs-lite`.
- **Implement a code change:** `development-01-change-map` + owning stage reference + the actual current source file(s). Do not edit from documentation alone.
- **Generate a full runner-style file:** `deployment-01-build-docker-runner` + the user's current env/runner file when available. If no such files exist, use the reference runner layout only as a template. Load pipeline algorithm references only if compiled path/mode settings must also change.
- **Generate a direct HPC job:** `deployment-02-direct-hpc-slurm` only. Add the parameter map if `para.inc` path/config edits are needed; do not load the full runner reference unless the user asks for runner-compatible wrappers or diagnostics.
- **Debug output:** start from the stage that produced the first bad intermediate file, not from Stage 9.

## Hard Invariants Worth Keeping in Initial Context

- F77 entry point accepts the exposure-list path as its first positional argument: `Fourier_Quad_Pipe <EXPO_LIST>`.
- Full `f77` supports configurable `ASTROMETRY_trivial`, `include_FLAT`, `include_Mask`, `ext_cat`, `ext_PSF`, `deblending`, `PSF_type`, and `PSF_Ms`.
- `f77_Lite` freezes those eight behaviors to `0, 0, 2, 1, 0, 1, 1, 0` respectively and removes the alternate branches; do not tell a Lite user to edit parameters that no longer exist.
- Full `f77` contains `00_psf_module.f` and optional PCA/multi-scale PSF reconstruction; Lite removes that path.
- Both runner mode and Direct HPC mode compile the bind-mounted source once before MPI launch. Host/container catalog paths must match strings compiled into `para.inc`; Direct mode must not inherit runner audit/smoke-test machinery unless requested.
