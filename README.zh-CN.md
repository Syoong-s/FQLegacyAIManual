# FFQManual

**FFQManual** 是面向 **Fourier_Quad Pipeline Legacy (F77)** 的低上下文、按需加载 AI 操作手册。插件/手册自身维护在 **Syoong-s/FQLegacyAIManual**，但这个 skill **不绑定任何特定的 Pipeline Git 仓库、owner、branch、remote URL 或检出路径**。`f77/main.f`、`f77_Lite/para.inc` 等文件名仅描述 Legacy F77 的参考布局并用于导航；AI 在修改或运行代码前必须检查用户实际提供的源码树。

仓库：https://github.com/Syoong-s/FQLegacyAIManual  
作者：[Syoong-s](https://github.com/Syoong-s)

## 设计原则

`skills/FFQManual/SKILL.md` 只保留精简的任务路由索引，不把完整算法、参数表、运行环境和开发细节一次性装入上下文。AI 首先判断任务属于哪个 stage、参数、variant、部署模式或源码模块，然后只读取对应 reference。

详细知识仍然完整保留，包括：

- `f77` 与生产冻结版 `f77_Lite` 的精确差异；
- 九个 F77 Pipeline stage 的数据流和实现；
- `para.inc` / `cust_para.inc` / `sig_para.inc` 参数含义、耦合关系和修改影响；
- Docker Compose 构建与路径映射；
- Apptainer/Singularity + Slurm 的 Direct HPC 生产作业生成；
- 用户显式要求时生成完整 `f77_docker/runner` 工作流；
- MPI launcher / PMI 约束；
- “开发需求 → 源码文件/符号 → 验证步骤”的精确定位；
- 与主 F77 源码范围隔离的 PDF-symmetry 辅助测量说明。

## 安装

### Claude Code marketplace

```text
/plugin marketplace add Syoong-s/FQLegacyAIManual
/plugin install FFQManual@ffqmanual-plugin
```

解压 Release 后也可以直接测试：

```bash
claude --plugin-dir ./FFQManual-plugin-1.3.0
```

### Codex marketplace

```bash
codex plugin marketplace add Syoong-s/FQLegacyAIManual
codex plugin add FFQManual@ffqmanual-plugin
```

使用本地解压目录：

```bash
codex plugin marketplace add /path/to/FFQManual-plugin-1.3.0
codex plugin add FFQManual@ffqmanual-plugin
```

插件沿用与 Superplan 相同的双宿主结构：Codex 使用 `.codex-plugin/` 和 `.agents/plugins/`，Claude Code 使用 `.claude-plugin/`，两者共享 `skills/`。

## 调用

Codex 中可显式调用：

```text
$FFQManual
```

Claude Code 中可调用安装后的 `FFQManual` skill/plugin，或者直接要求处理 legacy Fourier_Quad F77 Pipeline 相关任务。

插件允许隐式触发，但 description 被限制在具体实现范围内，因此普通弱透镜理论问题不会无必要地加载整套手册。

## Direct HPC 模式

当用户明确要求“直接生成 HPC 上可以运行的 Slurm”，而不是完整 runner 校验框架时，FFQManual 会单独加载 Direct HPC reference，并按需求生成：

1. OCI/GHCR 镜像拉取与 Apptainer/Singularity SIF 构建命令；
2. 精简 env，或者单个完全自包含的 Slurm 文件；
3. source、catalog、calibration、输入和输出目录的 host/container bind；
4. MPI 启动前仅编译一次；
5. 最终 `srun` 或兼容 `mpiexec` 启动 `Fourier_Quad_Pipe <EXPO_LIST>` 的命令。

除非用户明确要求，否则不会自动加入 MPI audit、smoke test、`run-apptainer.sh --check` 或完整 runner 的防御性检查层。

## 仓库结构

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

> Codex 实际使用的是 `.agents/`（复数），这与 Superplan 当前结构一致；因此自动 Release 打包 `.agents/`，而不是不存在的 `.agent/`。

## 自动 Release

推送任何符合 `v*` 的 tag 会触发 `.github/workflows/release.yml`。

例如：

```bash
git tag -a v1.3.0 -m "Release v1.3.0"
git push origin v1.3.0
```

工作流会先校验 manifest JSON 和 tag/manifest 版本一致性，然后完整打包：

```text
.agents/
.claude-plugin/
.codex-plugin/
skills/
```

同时附带 `README.md`、`README.zh-CN.md`、`CHANGELOG.md`、`AUDIT.md`、`LICENSE`，自动生成并发布：

```text
FFQManual-plugin-<version>.zip
FFQManual-plugin-<version>.tar.gz
checksums-sha256.txt
```

到 GitHub Release。

## Source-of-truth 规则

FFQManual 是知识路由/操作手册，不内置一份固定 Pipeline 源码快照。涉及精确代码修改时，应以用户实际提供或当前工作目录中的 Pipeline 源码为准；reference 中的默认参数只用于定位，真正修改前必须读取实际 include/source 文件确认。

## License

MIT，见 [LICENSE](LICENSE)。
