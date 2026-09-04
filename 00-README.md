# DSH Issue Report — 2026-09-04

中文完整报告: 01-report-zh.md
English summary (for GitHub Issues): 02-report-en.md

## Where to submit

1. Official harness (DeepSeek Harness):
   repo https://github.com/deepseek-ai/deepseek-harness (package @deepseek-ai/dsh, directory apps/cli)
   -> open a GitHub issue with the content of 02-report-en.md
2. Third-party plugin family (dsh-web-all / doctor / session-archive by @linxin666):
   repo https://github.com/zhu1090093659/dsh-web -> same issue or a second one scoped to the
   session-archive trim behavior and doctor schtasks failure.
3. Local plugin fork (dsh-whale-widget / whalehub-market) are private local repos
   (E:/S_Software/deepseek-harness/plugins/), no upstream needed.

Suggested titles:
- EN: "[bug] boot aborts on corrupt session log first frame; restore loses link: junctions and
  'dsh plugin install' silently drops bundles"
- ZH: 「dsh web 无法启动」事故报告：坏会话首帧 + zip 恢复丢失 link: 依赖 + reconcile 静默删 bundle

## Timeline of fixes already applied on this machine

- Quarantined the corrupt session log session-5e7515fd (bytes kept in dsh-git-backup/quarantine/).
- Restored ~/.dsh fully from .dsh_backup.zip (0 errors).
- Recreated node_modules junctions dsh-whale-widget / whalehub-market in profiles/web.
- Restored 5-bundle package.json; verified dsh plugin --profile web install keeps all bundles.
- Remaining user action: fully restart dsh web so the GUI boots with the full layer config.
