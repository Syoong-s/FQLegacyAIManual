# FFQManual

**FFQManual** 是面向 **Fourier_Quad Pipeline Legacy (F77)** 的低上下文、按需加载 AI 操作手册。插件/手册维护在 **Syoong-s/FQLegacyAIManual**，但不会把用户的实际 Pipeline 源码硬绑定到某个固定 checkout。reference 中的路径描述当前维护版 Legacy F77 的参考布局；进行精确代码修改时，用户实际源码始终优先。

仓库：https://github.com/Syoong-s/FQLegacyAIManual  
作者：[Syoong-s](https://github.com/Syoong-s)

## 1.4.0 适配内容

1.4.0 增加了对当前新版 Legacy F77 I/O/初始化接口的系统支持，包括：

- 统一的 `init_program/init_program.py` 初始化器；
- `science/<exposure>/`、`dqmask/<exposure>/`、`expolists/`、`stamps/`、`astrometry/`、`result/` 数据合同；
- `expo_<target>.list`、`fits_<target>.list` 和 initializer manifest；
- Full/Lite 共享的 `path_layout.inc`；
- `fq_chip_product_path`、`fq_expo_product_path`、`fq_expo_ccd_product_path`、`fq_base_product_path` 路径构造模型；
- 当前 `strl=512` 的路径/列表长度安全规则；
- `SOURCE_CAT_TILE_PREFIX='extern_'`；
- Stage 1 的 DQ/F6 正确调用顺序：DQ 先更新 `weight`，之后才执行 `set_background` / `set_sig`；
- Full/Lite 当前统一文件系统布局，同时保留 Lite 的 8 个冻结功能选择器；
- 与新版数据树匹配的 Docker / Apptainer / Slurm 运行说明。

## 设计原则

`skills/FFQManual/SKILL.md` 只保留任务路由和少量高价值 invariant。具体任务按需加载对应 reference，避免一次性把所有算法、参数和部署说明装入上下文。

新增的核心 reference 是：

```text
skills/FFQManual/references/dataset-01-initializer-layout.md
```

凡是涉及 initializer、曝光列表、DQ 命名、数据目录、产品路径或 Full/Lite I/O 兼容的问题，优先加载它。

## 安装

### Claude Code

```text
/plugin marketplace add Syoong-s/FQLegacyAIManual
/plugin install FFQManual@ffqmanual-plugin
```

测试本地 1.4.0 Release：

```bash
claude --plugin-dir ./FFQManual-plugin-1.4.0
```

### Codex

```bash
codex plugin marketplace add Syoong-s/FQLegacyAIManual
codex plugin add FFQManual@ffqmanual-plugin
```

本地 Release：

```bash
codex plugin marketplace add /path/to/FFQManual-plugin-1.4.0
codex plugin add FFQManual@ffqmanual-plugin
```

## 使用范围

FFQManual 可以用于：

- 从 Science/DQ `.fits.fz` 仓库生成当前 F77 所需数据树；
- 理解和调试九个 F77 stage；
- 选择 Full / Lite；
- 修改编译期参数；
- 修改产品目录/文件路径，同时优先复用当前 path helper；
- 生成 Docker、Apptainer 或 Slurm 运行方案；
- 定位“需求 → 源文件 → 符号 → 验证步骤”。

## 自动 Release

1.4.0 可使用：

```bash
git tag -a v1.4.0 -m "Release v1.4.0"
git push origin v1.4.0
```

Release workflow 会校验 plugin manifest 的版本号并打包完整运行目录。

## Source-of-truth 规则

FFQManual 是知识路由层，不是 Pipeline 源码镜像。reference 中的当前值用于导航和兼容判断；实际修改前仍必须读取用户当前源码中的真实参数、函数、调用顺序和目录结构。

## License

MIT，见 [LICENSE](LICENSE)。
