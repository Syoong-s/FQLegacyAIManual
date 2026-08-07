# Skill Architecture Audit

## Main issues found in the uploaded skill

1. The top-level `SKILL.md` was already reference-based, but still contained formulas, compile environment details, C++ commentary, and an external-skill dependency. This made the always-loaded layer larger and more fragile than necessary.
2. It did not contain a complete parameter-routing layer for precise user-directed tuning.
3. It did not document the real `f77_Lite` contract: eight selectors are frozen and alternate branches physically removed.
4. It mentioned Docker/HPC only briefly, insufficient for reliably generating `.env` / `f77pipeline.env` or reasoning about host/container path coupling and MPI mode.
5. It lacked a developer change map tying requested behavior to exact stage files and cross-file interfaces.
6. Auxiliary measurement documentation was presented too close to the main pipeline even though its source was not part of the reference material used for the audit; it also contained an unsupported `process_fd` source claim.
7. References to `general_rule` / `alatrico` made the skill non-self-contained when packaged alone.

## Resulting architecture

- `SKILL.md`: compact router + invariants only.
- stage references: retain detailed algorithms/formulas.
- `config-01-parameter-map.md`: parameter-oriented lookup.
- `variant-01-full-vs-lite.md`: variant boundaries.
- `deployment-01-build-docker-runner.md`: reproducible runtime/runner generation.
- `development-01-change-map.md`: source edit localization + verification.
- `measurement-00-scope.md`: isolates unverified auxiliary source scope.

This structure optimizes for low initial context while preserving deep detail after routing.

## 1.1.0 follow-up: Direct HPC mode

The 1.0.0 deployment reference still biased an agent toward the full defensive runner workflow even when a user explicitly wanted a short production Slurm script. Version 1.1.0 separates these concerns:

- `deployment-01-build-docker-runner.md` is now explicitly the full runner-style path.
- `deployment-02-direct-hpc-slurm.md` is the minimal production path: acquire SIF, define only needed paths, bind them directly, compile once, launch MPI ranks, and run `Fourier_Quad_Pipe <EXPO_LIST>`.
- Direct mode forbids automatically adding audit/check/smoke-test wrappers or unused runner configuration fields.
- Direct mode supports either a small env + Slurm pair or a single self-contained Slurm file.


## 1.2.0 follow-up: FFQManual packaging and release

- Runtime identity is now `FFQManual` in both host manifests and in `skills/FFQManual/SKILL.md`.
- Plugin repository metadata points to `Syoong-s/FQLegacyAIManual`.
- Automatic release packaging copies the complete `.agents/`, `.claude-plugin/`, `.codex-plugin/`, and `skills/` trees instead of reconstructing the knowledge tree file by file, preventing newly added references from being accidentally omitted.
- The Codex marketplace directory is `.agents/` (plural), matching the Superplan structure.

## 1.3.0 follow-up: repository-independent Legacy F77 knowledge

- The skill no longer identifies any external Git repository as the source of truth for the Legacy F77 pipeline.
- Reference paths such as `f77/main.f`, `f77_Lite/para.inc`, and `f77_docker/runner/...` are explicitly navigation hints describing the reference implementation layout.
- Code changes must first locate the corresponding file/symbol/interface in the user's actual source tree; directory names are not assumed.
- Deployment material is described as reference runner/direct-HPC patterns rather than a contract tied to one repository checkout.
- Auxiliary measurement notes describe limitations of the manual's reference material rather than absence from a named repository.
