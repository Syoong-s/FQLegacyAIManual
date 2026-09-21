# Changelog

## 1.4.0 - 2026-09-21

- Added `dataset-01-initializer-layout.md` for the unified `init_program.py`, current WFST-compatible dataset tree, science/DQ numbering rules, exposure lists, manifest, atomic publication, and resume/fail behavior.
- Added routing for initializer, dataset-layout, product-path, and list-contract tasks without turning `SKILL.md` into a full manual.
- Updated current Full/Lite I/O compatibility: both variants now share `path_layout.inc`, the modernized dataset layout, and centralized `fq_*_product_path` helpers.
- Corrected Stage-1 DQ/F6 documentation: DQ is loaded before `set_background`/`set_sig`; `set_sig` does not read DQ directly but receives a DQ-filtered `weight` mask in the current call flow.
- Updated the reference string capacity from `strl=150` to `strl=512` and documented the defensive exposure-list/path-length checks.
- Added `SOURCE_CAT_TILE_PREFIX='extern_'` and the current external-catalog filename contract.
- Added developer routing for `path_layout.inc`, `fq_assign_path`, `fq_chip_product_path`, `fq_expo_product_path`, `fq_expo_ccd_product_path`, `fq_base_product_path`, `initialize`, `get_image_list`, and initializer changes.
- Updated build guidance so current Full/Lite Makefiles are treated as depending on `path_layout.inc` as well as the traditional include files.
- Updated Docker/runner and Direct HPC references to use the current `expo_<target>.list` / initialized dataset model and to keep container-visible paths consistent with list contents.
- Preserved the 1.3.0 source-of-truth policy: the user's actual source tree remains authoritative for exact edits even when a current reference compatibility snapshot is documented.

## 1.3.0 - 2026-08-08

- Removed hard binding between skill knowledge and any specific external pipeline checkout.
- Reframed `f77/...`, `f77_Lite/...`, and `f77_docker/...` paths as reference Legacy F77 layout hints.
- Updated source-of-truth rules so the user's actual source tree, symbols, configuration files, and runtime layout are authoritative.
- Reworded deployment references to distinguish reference patterns from repository contracts.

## 1.2.0 - 2026-08-08

- Renamed the plugin and skill to `FFQManual`; moved the skill directory to `skills/FFQManual/`.
- Added dual Codex + Claude Code manifests and automatic release packaging.

## 1.1.0 - 2026-08-07

- Added a first-class Direct HPC deployment mode separate from the full runner/audit/smoke-test workflow.

## 1.0.0 - 2026-08-07

- Reworked `SKILL.md` into a compact task router.
- Added Full/Lite, parameter, deployment, development, stage, and auxiliary measurement references.
