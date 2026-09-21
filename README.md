# FFQManual

**FFQManual** is a low-context, on-demand AI operating manual for **Fourier_Quad Pipeline Legacy (F77)**. The plugin/manual is maintained at **Syoong-s/FQLegacyAIManual**, but the skill is not hard-bound to one pipeline checkout. Reference paths describe the maintained Legacy F77 layout; the user's actual source tree is authoritative for exact edits and runs.

Repository: https://github.com/Syoong-s/FQLegacyAIManual  
Author: [Syoong-s](https://github.com/Syoong-s)

> 中文说明：[README.zh-CN.md](README.zh-CN.md)

## Design

`skills/FFQManual/SKILL.md` is a compact router. It loads only the reference needed for the user's task instead of placing the whole pipeline manual in the initial context.

Version 1.4.0 adds explicit compatibility knowledge for the current modernized Legacy F77 layout, including:

- the unified `init_program/init_program.py` dataset initializer;
- `science/<exposure>/`, `dqmask/<exposure>/`, `expolists/`, `stamps/`, `astrometry/`, and `result/` contracts;
- `expo_<target>.list`, `fits_<target>.list`, and initializer manifest semantics;
- shared Full/Lite `path_layout.inc` and centralized `fq_*_product_path` helpers;
- the current `strl=512` list/path safety model;
- `SOURCE_CAT_TILE_PREFIX='extern_'`;
- corrected Stage-1 DQ/F6 ordering: DQ updates `weight` before `set_background` / `set_sig`;
- synchronized Full/Lite filesystem behavior while preserving Lite's frozen feature selectors;
- updated Docker/Apptainer/Slurm guidance using the current dataset contract.

The skill also covers the nine F77 stages, compile-time parameters, Full/Lite feature differences, Docker/runner patterns, Direct HPC jobs, and source-change localization.

## Installation

### Claude Code marketplace

```text
/plugin marketplace add Syoong-s/FQLegacyAIManual
/plugin install FFQManual@ffqmanual-plugin
```

Test an extracted 1.4.0 release directly:

```bash
claude --plugin-dir ./FFQManual-plugin-1.4.0
```

### Codex marketplace

```bash
codex plugin marketplace add Syoong-s/FQLegacyAIManual
codex plugin add FFQManual@ffqmanual-plugin
```

Local extracted release:

```bash
codex plugin marketplace add /path/to/FFQManual-plugin-1.4.0
codex plugin add FFQManual@ffqmanual-plugin
```

## Use

Codex explicit invocation:

```text
$FFQManual
```

Typical tasks include:

- initialize a raw science/DQ archive tree for the F77 pipeline;
- explain or debug a pipeline stage;
- choose Full vs Lite;
- change compile-time parameters safely;
- modify product paths without reintroducing hard-coded legacy strings;
- create Docker, Apptainer, or Slurm runs with correct container-visible list paths.

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
│           ├── dataset-01-initializer-layout.md
│           └── ...
├── README.md
├── README.zh-CN.md
├── CHANGELOG.md
├── AUDIT.md
└── LICENSE
```

## Automatic releases

Pushing a `v*` tag triggers the release workflow. For 1.4.0:

```bash
git tag -a v1.4.0 -m "Release v1.4.0"
git push origin v1.4.0
```

The workflow validates manifest versions and packages the runtime plugin directories plus documentation.

## Source-of-truth policy

FFQManual is a navigation/manual layer, not a vendored pipeline snapshot. Current reference facts are useful for routing and compatibility, but exact code changes must first inspect the user's current source, symbols, call order, configuration, and runtime layout.

## License

MIT. See [LICENSE](LICENSE).
