# DSH Harness 事故报告：更新/恢复后「插件没装好、无法启动」的根因与修复

> 日期：2026-09-04（UTC）/ 2026-09-05（本地 UTC+8）
> 范围：Windows 10.0.26200 单用户现场；涉及官方 @deepseek-ai/dsh 与第三方 @linxin666/dsh-web-all 生态。
> 状态：已定位、已修复、已恢复；本文档供反馈社区与维护者复现。

---

## 1. 摘要

一次「备份 -> 清理 -> 恢复」操作后，dsh web 反复无法启动，先后出现两类错误：

1. corrupt Zstandard session log: first frame is not exactly one header line（启动即崩，整棵插件树加载失败）；
2. Web profile 中两个以 link: 声明的本地插件（dsh-whale-widget、whalehub-market）在 zip 恢复后丢失（junction 未进备份包），导致启动解析 bundle 失败；随后 dsh plugin --profile web install 报告成功（pnpm "Already up to date"）却没有补装缺失依赖，并把这两个 bundle 从 dsh.profile.bundles 中静默移除（用户层配置被悄悄降级）。

本文给出：完整时间线、代码级根因、可复现步骤、已实施的修复清单、以及对社区/维护者的改进建议。

---

## 2. 环境

- OS: Windows 10.0.26200.9168
- DSH: @deepseek-ai/dsh@0.1.2-rc.1（全局安装于 E:/Visual Studio/nodejs/node_global/node_modules/@deepseek-ai/dsh）
- 运行 Node: v22.12.0（DSH 自带）/ v24.12.0（PATH）
- pnpm: 11.23.0（store: %LOCALAPPDATA%/pnpm/store/v11）
- registry: https://registry.npmmirror.com
- 第三方插件: @linxin666/dsh-web-all@0.3.13、dsh-doctor@0.3.13、dsh-session-archive@0.3.13（repo: github.com/zhu1090093659/dsh-web）
- 本地 link 插件: dsh-whale-widget@0.2.10（本地 fork）、whalehub-market@1.0.0（ZIP 引入），位于 E:/S_Software/deepseek-harness/plugins/
- 数据目录: C:/Users/Lenovo/.dsh（即 $DSH_HOME）

---

## 3. 时间线与现象

（时间均为 UTC；本地 = UTC+8）

- 08-25 ~ 09-02：正常使用。Web profile（~/.dsh/profiles/web）装有 @linxin666/dsh-web-all（08-30 时为 0.3.6）、dsh-whale-widget、whalehub-market，layer 栈 = base + web-app + whale-widget + whalehub-market + web-all；默认 agent preset 为自建 liangshen。
- 09-04 约 13:30：发现 Web profile 被重置：profiles/web/package.json 变回默认（bundles 只剩 base+web-app，236B，插件全部丢失），而 settings.yaml（872B，含 liangshen 预设/llm-pi-ai）仍是自定义。未留下触发重置的日志（疑似与插件升级/重装有关，见第 6 节建议）。
- 13:40：第一次完整备份 .dsh.zip（23,698 项）。
- 13:41 起：会话 session-5e7515fd（"修复/恢复"会话）运行期间，其自身 session.jsonl.zstd 被会话滚动裁剪/归档处理，首帧（含 header 行）被裁掉：文件第一帧变成日志中段的一条 tool/result（seq 139350）。该文件成为「坏首帧」日志。
- 15:01-15:08：该会话内完成 30 个会话的解码与导出（dsh-git-backup/，raw 里含这个已损坏的日志）。
- 15:50：用户运行 dsh web -> 崩溃：

      Error: dsh: plugin tree failed to load: failed to apply loader entry include (cordis:include):
        failed to apply loader entry workspace (@deepseek-ai/dsh-workspace):
        corrupt Zstandard session log: first frame is not exactly one header line
        at assertZstdHeaderFrame (.../@deepseek-ai/dsh-session-persistence-jsonl/lib/index.js:792)
        at readFirstZstdLine (...:1374) -> listArtifacts(...:1143) -> list(...:1103)
        at [cordis.init] (.../@deepseek-ai/dsh-workspace/lib/index.js:324)

- 15:50-15:56：修复会话重建 web profile（安装 web-all 0.3.13 全家桶等，package.json 恢复为 5-bundle 559B）。
- 15:56：第二次完整备份 .dsh_backup.zip（41,346 项）。注意此时 profiles/node_modules 是 junction/symlink 农场（备份包只收录了目录项）；whale-widget/whalehub-market 两个 junction 依赖没有进入 zip；sessions 目录里也只剩 1 个会话。
- 约 15:57-16:00：~/.dsh 被清空并由 DSH 重建为全新默认（新 anonymous-user-id、默认 settings/profiles），随后开启新的恢复会话（本报告由该会话完成）。
- 16:15-16:17：全量恢复（见第 5 节）。
- 16:22：用户重启 dsh web / 运行 dsh plugin --profile web install：
  - 因 zip 恢复丢了两个 junction，这两个 bundle 解析失败，启动失败；
  - pnpm install 输出 Already up to date（exit 0），但没有补建缺失的 link 依赖；
  - dsh plugin 的 reconcile 随后把这两个 bundle 从 dsh.profile.bundles 静默移除并写回 package.json（504B，bundles 只剩 3 个）。
  - GUI 随后以 3 bundles 启动成功（pet.json/skin-center-active.json 于 16:22:52 被写入，说明 dsh-web-all 已加载）。
- 16:2x-16:3x：本会话修复（见第 5 节），验证 dsh plugin --profile web install 不再移除 bundle。

---

## 4. 根因分析（祸首）

### A. 损坏的会话日志让 dsh web 启动即崩（官方 DSH 侧）

- DSH 启动时 @deepseek-ai/dsh-workspace 在 cordis init 阶段调用 session 持久化层的 list/listArtifacts，对每个 sessions/<workspace>/<id>/session.jsonl.zstd 执行 readFirstZstdLine：读取第一个 zstd 帧并解压，然后 assertZstdHeaderFrame 要求它恰好是一行（header 记录 + 结尾换行，长度非 0）。
- 本机坏文件（session-5e7515fd，修复会话自己的日志）首帧解出的是日志中段的 tool/result（多行、不是 header）-> 抛错 -> 整棵插件树加载失败 -> dsh web 起不来。
- 触发机制：会话日志的滚动裁剪/归档（运行时把早期轮次裁走；本机还装有 @linxin666/dsh-session-archive）在处理中把带 header 的首帧裁掉了，留下一个合法可解、但首帧不是 header 的文件。应用对运行中会话是追加/裁剪写入，重启后扫描才暴露问题。
- 后果放大：任何「把 sessions 目录整体拷回」的恢复（包括第一次尝试）都会把该坏文件带回去 -> 启动再次崩溃 -> 陷入 恢复->起不来->再恢复 死循环；期间 ~/.dsh 又被清空重建，配置/会话/上传全部丢失。

### B. zip 备份丢失 junction/symlink 依赖 -> 恢复后 bundle 缺失 -> 启动失败

- Web profile 的 package.json 用 link:E:/S_Software/... 声明两个本地插件；pnpm 安装后在 node_modules 里放的是 junction（目录重解析点）。
- 制作 .dsh_backup.zip 的归档方式不保留 junction 语义（zip 里既没有这两个包的实体内容，也没有链接记录）。
- 恢复后 profiles/web/node_modules 完整但没有这两个包 -> dsh web 启动加载 bundle 失败（解析不到 dsh-whale-widget/whalehub-market）。

### C. dsh plugin --profile web install 的「假成功」+ 静默降级（DSH CLI 侧）

- dsh plugin（lib/bin.js -> lib/plugin-*.js）的工作方式：在 profile 目录 spawnSync("pnpm", args)，pnpm exit 0 后执行 reconcilePlugins：逐个 dependency 调 exportsPatch（resolveBundleDir 解析包 + 检查 package.json 是否声明 dsh.bundle.patch），把能解析成 bundle 的加进 layer 栈，把曾是依赖、但现在解析不到/不再是 bundle 的从 dsh.profile.bundles 里删掉并写回 package.json。
- 本机 pnpm 在缺失 link 依赖时输出 Already up to date 且不补建（pnpm 以自己的状态文件判断最新，而非对照 node_modules 实际内容核对 link 项），exit 0 -> reconcile 照跑 -> 两个「装着但解析不到」的 bundle 被静默移除（stderr 只对新增非 bundle 依赖有 warning，对移除没有任何提示）。
- 用户看到的现象：插件命令显示成功、却把插件从 layer 里删了；GUI 再启动就少了鲸鱼余额小组件和插件市场。

### D. 其它现场噪音（与本次事故无直接因果）

- @linxin666/dsh-doctor 的 supervisor 计划任务注册失败：schtasks /Create ... Access is denied（无管理员权限时必然失败），doctor 每次运行 ok:false，容易把用户注意力带偏。
- 恢复前 ~/.dsh 被整体清空重建（新 anonymous-user-id、默认设置），让「配置丢失」的表现更严重。

---

## 5. 已实施的修复（可复用的恢复清单）

1. 定位并隔离坏会话日志
   - 用与应用完全一致的规则校验每个 session.jsonl.zstd：首个 zstd 帧解压后必须恰好一行、以换行结尾。
   - 将损坏的 session-5e7515fd 隔离到 dsh-git-backup/quarantine/（原字节保留 + 说明文件），不写回 ~/.dsh/sessions；可读文稿另有 dsh-git-backup/conversations/.../session-5e7515fd-....md。
2. 全量恢复 ~/.dsh
   - 以第二次备份 .dsh_backup.zip 为源（41,313 项、约 4.8 GB，0 错误），排除：当前活动会话、.credentials.yaml（改为合并）、profiles/node_modules（本机为指向全局安装的链接农场，无需覆盖）。
   - 恢复 settings.yaml（liangshen 默认预设/暗色主题/llm-pi-ai）、.agent-presets/liangshen、profiles/web（含 @linxin666 全家桶真实文件）、profiles/dsh-tui、storages / task-board / uploads / attachments / skins 等全部用户数据。
3. 恢复会话历史：30 个历史会话（29 个来自 15:08 导出 + 1 个来自第二次备份），全部通过首帧校验；当前会话不受影响。
4. 凭据与身份：合并补回 OPENROUTER_API_KEY ref，保留当前浏览器授权记录；恢复原 .anonymous-user-id。
5. 修复缺失的 link 依赖（本次「插件没装好」的直接修复）
   - 在 profiles/web/node_modules 重建两个 junction：
     - dsh-whale-widget -> E:/S_Software/deepseek-harness/plugins/DeepSeek-Balance-Whale-Widget
     - whalehub-market -> E:/S_Software/deepseek-harness/plugins/whalehub-dsh/plugin
   - 两个本地插件均无第三方依赖（dependencies 为空），junction 即可完整工作。
   - 恢复 5-bundle 的 package.json（base、web-app、dsh-whale-widget、whalehub-market、dsh-web-all）。
   - 重跑 dsh plugin --profile web install 验证：pnpm Already up to date，reconcile 不再移除任何 bundle（5 个 bundle 全部可解析，patch 文件均存在）。
6. 验证：恢复后 ~/.dsh/sessions 共 31 个日志全部通过首帧校验（0 无效）；5 个 bundle 均可从 profile 解析到 package.json 与 cordis.patch.yml。

用户侧最后一步：完全退出并重启 dsh web，让 GUI 以完整 5-bundle layer 配置启动。

---

## 6. 给社区/维护者的建议（按优先级）

1. 启动扫描容错（强烈建议）：dsh-workspace 在 boot 时扫描 session 日志，遇到坏首帧文件应跳过并隔离/标记（stderr 报告路径），而不是让整棵插件树加载失败。至少提供 dsh doctor 一键「扫描 + 隔离坏会话 + 校验 profile bundle」的修复入口。
2. 会话滚动裁剪/归档必须保证 header 帧不被裁掉：任何 trim/archive 都要保留第一个 zstd 帧（header 行），并在写入后自检「首帧 == 恰好一行 header」。
3. 备份/导出应保留链接语义或给出差异清单：对 node_modules 中的 junction/symlink 依赖（link: 依赖很常见），zip 类备份要么记录并提示「以下链接未备份」，要么在恢复流程里跑一致性校验（bundle 可解析性 + 缺失链接补建）。
4. dsh plugin reconcile 不要静默改写用户 layer 配置：移除 dsh.profile.bundles 中的包时应区分「用户移除」与「依赖缺失/未安装」，后者应醒目 warning + 建议命令（如给出 pnpm install 失败原因），或把被移除项写进备份文件而不是直接删。
5. pnpm Already up to date 需要对照真实 node_modules：缺失顶层 link 依赖时 pnpm 仍报最新，建议 dsh 侧在 pnpm 返回后自行核对每个 dependency 是否可解析（reconcile 前加 preflight），缺失则报错或执行修复。
6. 更新/升级流程留痕：profile 被重置（plugins 层丢失）没有日志与审计；建议升级或重装前对 profiles/<name>/package.json 自动备份（*.bak），升级后 diff 并提示。
7. 错误信息可操作化：boot 失败信息应点明哪个 bundle/文件、缺什么、怎么修，并建议官方提供 dsh doctor 全套自检。

---

## 7. 可复现步骤（供维护者）

复现 A（坏首帧导致无法启动）
1. 任取一个正常 session.jsonl.zstd，用 zstd 逐帧切片，把第一个帧删掉（保留后续帧）后放回 sessions/<ws>/<id>/；
2. 启动 dsh web -> boot 阶段即抛 corrupt Zstandard session log: first frame is not exactly one header line，GUI 无法启动。
3. 校验规则源码：@deepseek-ai/dsh-session-persistence-jsonl 的 assertZstdHeaderFrame（plaintext.length===0 || indexOf('\n') !== length-1 即抛错）。注意：真实代码行号以安装包为准。

复现 B（zip 恢复丢 link 依赖 + plugin install 静默降级）
1. profile package.json 含 link: 依赖并已由 pnpm 装成 junction；
2. 用常规 zip 工具备份 ~/.dsh 后清空重建，再从 zip 恢复（junction 丢失）；
3. dsh web 启动失败（bundle 解析不到）；
4. 运行 dsh plugin --profile web install：pnpm 显示 Already up to date，随后 reconcile 把这些 bundle 从 dsh.profile.bundles 中移除（可观察到 package.json 被改写、bundles 变少）。

---

## 8. 附件与数据位置（本机）

- 备份源：C:/Users/Lenovo/.dsh.zip（第 1 次，23,698 项）、C:/Users/Lenovo/.dsh_backup.zip（第 2 次，41,346 项）
- 会话导出/状态快照：C:/Users/Lenovo/dsh-git-backup/（conversations/、raw/、state/）
- 隔离区：C:/Users/Lenovo/dsh-git-backup/quarantine/session-5e7515fd-...（含 WHY-QUARANTINED.txt）
- 恢复日志：C:/Users/Lenovo/.dsh-restore-work/worker.log
- 本报告目录：C:/Users/Lenovo/dsh-issue-report-2026-09-04/
- 相关官方代码（dsh 0.1.2-rc.1 安装内）：
  - @deepseek-ai/dsh-session-persistence-jsonl/lib/index.js（assertZstdHeaderFrame / readFirstZstdLine / listArtifacts）
  - @deepseek-ai/dsh-workspace/lib/index.js:324（boot 扫描触发点）
  - @deepseek-ai/dsh/bin.js -> lib/plugin-*.js（runPlugin / reconcilePlugins / exportsPatch）

---

报告完。


---

## 9. 追加：第二个根因 — 更新 0.1.2-rc.1 移除 Session.events，导致 liangshen 预设续聊崩溃（本报告修复的现场问题）

时间：2026-09-04 13:19 UTC 用户执行 npm install --global @deepseek-ai/dsh@0.1.2-rc.1（222 个 @deepseek-ai 包全量替换）。此前（13:08）liangshen 预设会话运行正常；更新后所有 agentPreset=liangshen 的会话在 turn 开始时立即失败：

    Cannot read properties of undefined (reading 'length')

定位结论（无堆栈，通过代码比对与实验得出）：
- 新版 Session（@deepseek-ai/dsh-session）删除了旧成员 events（数组），改为 eventAt(seq) / snapshotEvents() / ownEvents() / seq。
- 用户自建预设 ~/.dsh/.agent-presets/liangshen/tool-bootstrap.mjs（2026-08-25 编写）的 scanEvents() 仍执行 const events = session.events; ... events.length —— 在 rc.1 下 session.events 为 undefined，每次 system prompt 组装（每个 turn 开始时）都会崩溃；standard 等内置预设不加载该文件，因此不受影响。
- 额外兼容性要求：0.1.2-rc.1 需要较新的 Node（node:zlib 的 createZstdDecompress、node:module 的 stripTypeScriptTypes 在 Node 22.12 不存在；需 Node >= 23/24）。用旧 Node 启动会直接在插件树加载期报 SyntaxError。

已做的修复（预设侧兼容补丁，已验证可加载）：
- tool-bootstrap.mjs 的 scanEvents 改为：Array.isArray(session.events) ? session.events : (typeof session.snapshotEvents === 'function' ? session.snapshotEvents() : [])。
- 原文件备份为 tool-bootstrap.mjs.bak-pre-rc1。
- 需要完全重启 dsh web 后生效（进程内 ESM 缓存旧代码）。

对维护者的补充建议：
1. 移除公开成员时保留一个 deprecated 别名（如 session.events getter -> snapshotEvents()），避免生态插件瞬间全崩，并在日志里 warning。
2. turn/end 的 error 记录只保存 message、不保存 stack，给用户和开发者定位带来困难；建议至少把 stack 存进错误事件或可选的反馈通道。
3. 大版本升级（此处 Node 能力要求）应做运行时版本检查并给出明确报错，而不是在加载期抛出晦涩的 SyntaxError。
