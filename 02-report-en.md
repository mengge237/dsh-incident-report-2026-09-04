# DSH Harness incident report: after an update / restore, plugins "fail to install" and dsh web won't start

Date: 2026-09-04 (UTC). Machine: Windows 10.0.26200, DSH 0.1.2-rc.1 (installed under E:/Visual Studio/nodejs/node_global), pnpm 11.23.0,
plus third-party plugins from the dsh-web family (@linxin666/dsh-web-all 0.3.13, dsh-doctor 0.3.13, dsh-session-archive 0.3.13).

## Summary

Two independent bugs made the GUI unbootable after a backup -> clean -> restore cycle:

1. A session log whose FIRST zstd frame is no longer the header line makes dsh web crash at boot
   (corrupt Zstandard session log: first frame is not exactly one header line). The whole
   plugin tree fails to load.
2. Local plugins declared with link: deps (dsh-whale-widget, whalehub-market) live in node_modules as
   junctions. A plain zip backup does not preserve them, so after restoring from the zip the bundles
   are missing and dsh web cannot start. Worse: 'dsh plugin --profile web install' then prints
   pnpm "Already up to date" (exit 0) WITHOUT re-creating the missing links, and dsh's own reconcile
   step silently REMOVES those bundles from dsh.profile.bundles and rewrites package.json.

## Root cause A: corrupt first zstd frame vs strict boot validation

- At boot, @deepseek-ai/dsh-workspace lists sessions via the JSONL persistence backend
  (readFirstZstdLine -> listArtifacts -> list). For every session.jsonl.zstd it decompresses the
  FIRST zstd frame and requires it to be exactly ONE header line ending in newline
  (assertZstdHeaderFrame). Any other content throws, and the throw is fatal for the whole boot
  ("dsh: plugin tree failed to load ... corrupt Zstandard session log ...").
- How it happened: a running session's log got trimmed/archived (rolling trim; dsh-session-archive
  is installed) in a way that dropped the header frame. The file that remained was fully decodable,
  but its first frame was a mid-log tool/result event. Every "restore" that copied the sessions
  folder back re-introduced this file and crashed boot again -> restore/crash loop.

## Root cause B: zip backup loses junctions; pnpm "up to date"; reconcile silently downgrades

- Profiles/web/package.json had link:E:/S_Software/... deps. pnpm installs link: deps as junctions
  inside node_modules. The backup zip contains neither the junction semantics nor the content of the
  two linked local plugins.
- After restore: dsh web boot failed to resolve those two bundles.
- dsh plugin (lib/bin.js -> lib/plugin-*.js, runPlugin) spawns pnpm in the profile dir and, when pnpm
  exits 0, calls reconcilePlugins: for each dependency it resolves the package and checks whether it
  declares dsh.bundle.patch; packages that were bundles but are no longer resolvable are removed from
  dsh.profile.bundles and the manifest is rewritten.
- In this state pnpm reported "Already up to date" without creating the missing junctions, exited 0,
  and reconcile silently dropped the two bundles from the layer stack. The GUI then started with a
  degraded (3-bundle) profile and no warning to the user.

## Fixes applied (and reusable restore checklist)

1. Validate every session log with the app's own rule (first zstd frame == one header line);
   quarantine invalid files instead of deleting them (original bytes kept), and never write them
   back under ~/.dsh/sessions.
2. Full restore of ~/.dsh from the second backup zip (41,313 entries, ~4.8 GB, 0 errors),
   excluding: the currently active session, .credentials.yaml (merged instead), and
   profiles/node_modules (a junction farm pointing at the global install on this machine).
3. Restore 30 historical sessions (29 from a 15:08 export + 1 from the backup), all header-valid.
4. Recreate the two missing junctions in profiles/web/node_modules:
   - dsh-whale-widget -> E:/S_Software/deepseek-harness/plugins/DeepSeek-Balance-Whale-Widget
   - whalehub-market -> E:/S_Software/deepseek-harness/plugins/whalehub-dsh/plugin
   (Both local plugins have no third-party deps, so junctions are sufficient.)
5. Restore the 5-bundle package.json (base, web-app, dsh-whale-widget, whalehub-market, dsh-web-all),
   then re-run 'dsh plugin --profile web install' and verify reconcile keeps all 5 bundles.

## Suggested upstream changes (please consider)

1. Boot resilience: when a session log fails the first-frame header check, skip + quarantine it and
   report the path instead of failing the whole plugin tree. Ship a dsh doctor command that scans
   and fixes this (quarantine + profile bundle preflight).
2. Session trim/archive must never drop the first (header) frame; self-check after any rewrite.
3. Backup/export tooling should either preserve junction/symlink semantics or emit a list of links
   that were not archived; restore flows should run a consistency check (bundles resolvable).
4. dsh plugin reconcile should not silently edit the user's bundle layer: distinguish "user removed"
   from "dependency missing / not installed"; in the latter case warn loudly with a suggested fix
   and/or keep an undo copy instead of deleting the entry.
5. Before trusting pnpm's "Already up to date", verify the actual node_modules state for link:
   dependencies; missing top-level links should be recreated or reported as an error.

## Reproduction

A) Boot crash: take any session.jsonl.zstd, drop its first zstd frame, keep the rest, put it under
   sessions/<ws>/<id>/, run dsh web -> boot fails with the corrupt-header error.
B) Restore-loss + silent downgrade: profile with link: deps installed as junctions -> zip backup ->
   wipe -> restore from zip -> dsh web fails on the missing bundles -> 'dsh plugin --profile web
   install' reports up to date but reconcile removes the bundles from dsh.profile.bundles.

## Supporting data (this machine)

- Backups: C:/Users/Lenovo/.dsh.zip (1st), C:/Users/Lenovo/.dsh_backup.zip (2nd)
- Session export snapshot: C:/Users/Lenovo/dsh-git-backup/
- Quarantine: C:/Users/Lenovo/dsh-git-backup/quarantine/
- Restore worker log: C:/Users/Lenovo/.dsh-restore-work/worker.log
- Relevant code in the 0.1.2-rc.1 install:
  - @deepseek-ai/dsh-session-persistence-jsonl/lib/index.js (assertZstdHeaderFrame, readFirstZstdLine, listArtifacts)
  - @deepseek-ai/dsh-workspace/lib/index.js:324 (boot scan trigger)
  - @deepseek-ai/dsh/bin.js -> lib/plugin-*.js (runPlugin / reconcilePlugins / exportsPatch / writeProfileManifest)


---

## 9. Addendum: second root cause — 0.1.2-rc.1 removed Session#events, breaking a third-party agent preset (fixed on this machine)

At 2026-09-04 13:19 UTC the user ran `npm install --global @deepseek-ai/dsh@0.1.2-rc.1` (all 222 @deepseek-ai packages replaced). Before that (13:08) sessions running the custom 'liangshen' agent preset worked; right after the update, every turn in a liangshen-preset session failed immediately with:

    Cannot read properties of undefined (reading 'length')

Findings (no stack available; established by code comparison + experiments):
- The new Session class (@deepseek-ai/dsh-session) removed the old `events` array member; the API is now eventAt(seq) / snapshotEvents() / ownEvents() / seq.
- The user preset ~/.dsh/.agent-presets/liangshen/tool-bootstrap.mjs (written 2026-08-25) still runs `const events = session.events; ... events.length` inside scanEvents() during every system-prompt assembly (every turn start). Under rc.1 session.events is undefined -> TypeError. Built-in presets (standard) do not load this file, so they are unaffected.
- Extra compatibility note: rc.1 requires a newer Node (node:zlib createZstdDecompress and node:module stripTypeScriptTypes are missing on Node 22.12; Node >= 23/24 required). Running with an old Node fails at plugin-tree load with a cryptic SyntaxError.

Fix applied on this machine (preset-side compatibility patch, import-verified):
- scanEvents now uses `Array.isArray(session.events) ? session.events : (typeof session.snapshotEvents === 'function' ? session.snapshotEvents() : [])`.
- Original preserved as tool-bootstrap.mjs.bak-pre-rc1.
- A full restart of `dsh web` is required (in-process ESM cache keeps old code).

Suggested upstream changes:
1. When removing a public member, keep a deprecated alias (e.g. session.events getter delegating to snapshotEvents()) with a warning, so ecosystem presets do not break instantly.
2. turn/end error records store only the message, not the stack; storing the stack (or offering a feedback channel) would make such issues far easier to diagnose.
3. For upgrades that raise the Node floor, add an explicit runtime version check with a clear error instead of failing at load time with a bare SyntaxError.
