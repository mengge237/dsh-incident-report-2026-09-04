# DSH Harness 0.1.2-rc.1 更新事故报告（2026-09-04）

DeepSeek Harness 更新/恢复过程中「插件没装好、dsh web 无法启动、liangshen 预设会话全部报 Cannot read properties of undefined (reading 'length')」的完整复盘。

**这是一份真实事故的根因分析 + 修复记录 + 给上游的改进建议，不含任何会话日志、凭据或隐私数据。**

## 内容

| 文件 | 说明 |
|---|---|
| `01-report-zh.md` | 中文完整报告（时间线、代码级根因 A/B/C、修复清单、可复现步骤、7 条上游建议、第 9 节追加第二根因） |
| `02-report-en.md` | English summary（适合直接贴 GitHub Issues） |
| `00-README.md` | 投递指引（提交到哪个仓库） |

## 核心发现（速览）

1. **启动崩溃**：会话日志滚动裁剪/归档把首个 zstd 帧（header 行）裁掉后，`dsh web` 启动扫描会以
   `corrupt Zstandard session log: first frame is not exactly one header line` 让整棵插件树加载失败。
2. **恢复丢失链接依赖**：zip 备份不保留 node_modules 里的 junction/symlink（`link:` 依赖），恢复后 bundle 缺失；
   且 pnpm 会误报 `Already up to date` 不补装，`dsh plugin` 的 reconcile 随后**静默删除** `dsh.profile.bundles` 里的包。
3. **升级回归（追加）**：`0.1.2-rc.1` 移除 `Session.events` 后，依赖旧 API 的第三方 agent 预设（本机 liangshen 预设的
   `tool-bootstrap.mjs`）在每个回合开始时报 `Cannot read properties of undefined (reading 'length')`；同时 rc.1 需要 Node ≥ 24
   （`node:zlib` zstd / `node:module` type-stripping），Node 22 直接加载失败。

## 配套过渡插件（想“先让它能跑起来”的人直接用）

在官方修复合入前，发布了一个纯 server 的兼容垫片插件，装上即可缓解上述 ①②③：

- 仓库：https://github.com/mengge237/dsh-legacy-compat
- 一条命令安装（已验证：自动进 bundle 栈，无需 npm）：

```bash
dsh plugin --profile web add github:mengge237/dsh-legacy-compat
# 完全重启 dsh web；Settings → Plugins 里可看到它
```

- 功能：坏会话日志启动自检/隔离（不删除）、Session.events 兼容别名、Node 预检；
  默认安静，`DSH_LEGACY_COMPAT_VERBOSE=1` 看详情；带 `bin/uninstall.mjs` 一键卸载。

## 社区反馈位置

- 官方 DeepSeek Harness Discussion: https://github.com/deepseek-ai/deepseek-harness/discussions/5655
- 插件作者（dsh-web 家族）Issue: https://github.com/zhu1090093659/dsh-web/issues/1376

## 许可

© 2026 mengge237 · 自由引用请注明出处。本仓库只包含分析与建议，不含用户数据。
