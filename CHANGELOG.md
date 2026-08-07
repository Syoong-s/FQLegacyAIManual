# Changelog

## 1.3.0 - 2026-08-08

- Removed all binding between the skill knowledge and any specific external pipeline repository.
- Reframed `f77/...`, `f77_Lite/...`, and `f77_docker/...` paths as reference Legacy F77 layout hints rather than required checkout paths.
- Updated source-of-truth rules so the user's actual source tree, symbols, configuration files, and runtime layout are authoritative.
- Reworded runner/deployment references to distinguish reference patterns from repository contracts.
- Updated auxiliary measurement scope so it only states what was or was not included in the manual's reference material.
- Updated README files and plugin metadata to make the repository-independent scope explicit.

## 1.2.0 - 2026-08-08

- Renamed the plugin and skill to `FFQManual`; moved the skill directory to `skills/FFQManual/`.
- Changed plugin author/developer metadata to `Syoong-s` with `https://github.com/Syoong-s`.
- Changed the manual plugin homepage/repository to `https://github.com/Syoong-s/FQLegacyAIManual`.
- Updated Codex and Claude marketplace metadata and installation commands.
- Added `.github/workflows/release.yml`: every `v*` tag validates manifest/tag version consistency, packages `.agents/`, `.claude-plugin/`, `.codex-plugin/`, and the complete `skills/` tree, then publishes ZIP, tar.gz, and SHA-256 release assets.
- Reworked English and Chinese README files for the new naming, repository, install commands, Direct HPC mode, package layout, and automatic release process.

## 1.1.0 - 2026-08-07

- Added `deployment-02-direct-hpc-slurm.md` as a first-class Direct HPC deployment mode.
- Split minimal production Slurm generation from the full runner/audit/smoke-test workflow.
- Added direct OCI/GHCR-to-SIF commands, task-specific env generation, explicit Apptainer binds, compile-once execution, and direct MPI launch patterns.
- Added one-file Slurm mode and rules preventing unused runner validation variables/checks from leaking into direct jobs.
- Updated the top-level router so requests for "directly runnable Slurm" load only the new deployment reference by default.

## 1.0.0 - 2026-08-07

- Reworked `SKILL.md` into a compact task router.
- Added explicit current-source-as-truth rules for code changes.
- Added Full `f77` vs `f77_Lite` frozen-selector reference.
- Added consolidated compile-time parameter map.
- Added Docker/Apptainer/Slurm runner-generation reference.
- Added developer request-to-source-file change map and verification ladder.
- Isolated auxiliary PDF-symmetry measurement documentation from the main Legacy F77 scope.
- Removed missing cross-skill dependencies and unsupported `process_fd` source claim.
- Added dual Codex + Claude Code plugin manifests/marketplace catalogs following the Superplan layout.
