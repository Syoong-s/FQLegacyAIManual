# FFQManual

**FFQManual** is a low-context, on-demand AI operating manual for **Fourier_Quad Pipeline Legacy (F77)**. This plugin/manual is maintained at **Syoong-s/FQLegacyAIManual**, but the skill is **not bound to any particular pipeline Git repository, owner, branch, remote URL, or checkout path**. Filenames such as `f77/main.f` and `f77_Lite/para.inc` describe the reference Legacy F77 layout and are used only as navigation hints; an agent must inspect the user's actual source tree before editing or running it.

Repository: https://github.com/Syoong-s/FQLegacyAIManual  
Author: [Syoong-s](https://github.com/Syoong-s)

> 中文说明：[README.zh-CN.md](README.zh-CN.md)

## Design

`skills/FFQManual/SKILL.md` is intentionally a compact routing index rather than a full manual. It identifies the requested subsystem first, then loads only the matching detailed reference. This keeps the initial context small while preserving complete detail once the task is routed.

The skill covers:

- Full `f77` and production-frozen `f77_Lite` behavior and differences;
- all nine F77 pipeline stages;
- `para.inc`, `cust_para.inc`, and `sig_para.inc` parameter semantics and tuning;
- Docker Compose build/runtime paths;
- direct Apptainer/Singularity + Slurm production-job generation;
- complete reference-style `f77_docker/runner` workflows when explicitly requested;
- MPI launcher and PMI constraints;
- developer request-to-source-file localization and verification;
- auxiliary PDF-symmetry measurement notes kept separate from the main F77 source scope.

## Installation

### Claude Code marketplace

```text
/plugin marketplace add Syoong-s/FQLegacyAIManual
/plugin install FFQManual@ffqmanual-plugin
```

To test an extracted release directly:

```bash
claude --plugin-dir ./FFQManual-plugin-1.3.0
```

### Codex marketplace

```bash
codex plugin marketplace add Syoong-s/FQLegacyAIManual
codex plugin add FFQManual@ffqmanual-plugin
```

For a local extracted release:

```bash
codex plugin marketplace add /path/to/FFQManual-plugin-1.3.0
codex plugin add FFQManual@ffqmanual-plugin
```

The dual-host layout follows the same pattern as Superplan: Codex uses `.codex-plugin/` and `.agents/plugins/`, Claude Code uses `.claude-plugin/`, and both share `skills/`.

## Use

Codex explicit invocation:

```text
$FFQManual
```

Claude Code can invoke the installed `FFQManual` skill/plugin or the user can ask directly for a task involving the legacy Fourier_Quad F77 pipeline.

Implicit invocation is enabled, but the skill description is implementation-specific so unrelated weak-lensing questions should not load the manual unnecessarily.

## Direct HPC mode

When the user requests a directly runnable HPC job rather than a full defensive runner framework, FFQManual routes to the dedicated Direct HPC reference. It can generate:

1. OCI/GHCR image pull and Apptainer/Singularity SIF creation commands;
2. a small environment file or a single self-contained Slurm script;
3. host-to-container bind mappings for source, catalogues, calibration data, input, and output;
4. compile-once execution before MPI launch;
5. the final `srun`/compatible `mpiexec` command for `Fourier_Quad_Pipe <EXPO_LIST>`.

Audit scripts, smoke tests, `run-apptainer.sh --check`, and the full runner validation layer are not added unless explicitly requested.

## Repository layout

```text
FQLegacyAIManual/
├── .agents/
│   └── plugins/marketplace.json
├── .claude-plugin/
│   ├── marketplace.json
│   └── plugin.json
├── .codex-plugin/
│   └── plugin.json
├── .github/
│   └── workflows/release.yml
├── skills/
│   └── FFQManual/
│       ├── SKILL.md
│       ├── agents/openai.yaml
│       └── references/
├── README.md
├── README.zh-CN.md
├── CHANGELOG.md
├── AUDIT.md
└── LICENSE
```

> Codex uses `.agents/` (plural). This is the Superplan-compatible marketplace path; `.agent/` is not used by the current package layout.

## Automatic releases

Pushing a tag matching `v*` triggers `.github/workflows/release.yml`.

Example:

```bash
git tag -a v1.3.0 -m "Release v1.3.0"
git push origin v1.3.0
```

The workflow validates the plugin manifests and tag version, then packages the complete runtime plugin directories:

```text
.agents/
.claude-plugin/
.codex-plugin/
skills/
```

It also includes `README.md`, `README.zh-CN.md`, `CHANGELOG.md`, `AUDIT.md`, and `LICENSE`, and publishes:

```text
FFQManual-plugin-<version>.zip
FFQManual-plugin-<version>.tar.gz
checksums-sha256.txt
```

as GitHub Release assets.

## Source-of-truth policy

FFQManual is a navigation/manual layer, not a vendored snapshot of the pipeline. For exact code modifications, the user's actual pipeline source tree is authoritative. Parameter defaults in references are orientation values and must be verified against the actual source before editing.

## License

MIT. See [LICENSE](LICENSE).
