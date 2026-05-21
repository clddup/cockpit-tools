# Local Changes for `develop/clddup`

This file records fork-specific behavior maintained on `develop/clddup` so upstream syncs can check whether new upstream changes affect local customizations.

## Branch policy

- Long-lived fork branch: `develop/clddup`
- Upstream baseline: latest upstream release tag commit on `main`
- Main branch policy: keep `main` aligned with latest upstream `main`
- Development branch sync policy: merge the latest upstream release tag commit into `develop/clddup`, not commits after that tag on `upstream/main`
- Upstream remote URL: `https://github.com/jlcodes99/cockpit-tools.git`
- Fork release tag policy: use user-owned tags such as `v0.24.3-clddup.1`; do not push upstream's original tag names to `origin`

## Current custom commits

These commits describe the current local customization set relative to `origin/main` at the time this file was created:

- `c9d71fc feat: codex导入不阻塞`
- `eff496d fix: 修复构建错误 - 对齐 tauri 版本并修复变量名`
- `79b8b6a feat: 优化导入性能、添加导入进度显示、拆分异常筛选`

## Watched files

When syncing upstream, compare upstream-changed files against this list first:

- `Cargo.lock`
- `docs/PROJECT_ANALYSIS.md`
- `package.json`
- `pnpm-lock.yaml`
- `src-tauri/src/commands/codex.rs`
- `src-tauri/src/modules/codex_account.rs`
- `src/pages/CodexAccountsPage.tsx`
- `src/stores/useCodexAccountStore.ts`

## Custom behavior areas

### Codex import performance, partial success, and progress

Watched files:

- `src-tauri/src/modules/codex_account.rs`
- `src-tauri/src/commands/codex.rs`
- `src/pages/CodexAccountsPage.tsx`
- `src/stores/useCodexAccountStore.ts`

Behavior to preserve:

- Codex JSON/token batch import should not fail the entire batch when one item fails.
- Failed import items should be skipped and collected into user-visible failure messages.
- Import progress is emitted through `codex:json-import-progress`.
- Frontend import UI shows `current/total` progress.
- Refresh phase shows quota refresh progress.
- Codex profile hydration runs concurrently with a concurrency limit.
- Codex profile hydrate updates are buffered and flushed in batches to reduce frequent state updates.

Symbols and strings to watch:

- `import_codex_candidate`
- `import_accounts_from_token_lines`
- `import_from_json`
- `refresh_imported_codex_accounts`
- `codex:json-import-progress`
- `hydrateMissingProfiles`
- `CODEX_PROFILE_HYDRATE_BUFFER`
- `CODEX_PROFILE_HYDRATE_FLUSH_INTERVAL_MS`

### Codex abnormal account filters

Watched files:

- `src/pages/CodexAccountsPage.tsx`

Behavior to preserve:

- The previous broad `ERROR` filter is split into more specific filter categories:
  - `AUTH_ERROR`
  - `QUOTA_ERROR`
  - `REFRESH_FAILED`
- `requires_reauth` should count as authorization error.
- Quota errors caused by plain network refresh failure should count as refresh failure, not quota error.
- API HTTP errors should not be misclassified as plain network refresh failures.

Symbols and strings to watch:

- `isAbnormalAccount`
- `AUTH_ERROR`
- `QUOTA_ERROR`
- `REFRESH_FAILED`
- `requires_reauth`
- `quota_error`
- `error sending request`
- `API 返回错误`

### Tauri dependency alignment

Watched files:

- `package.json`
- `pnpm-lock.yaml`
- `Cargo.lock`

Behavior to preserve:

- `@tauri-apps/api` is aligned to `^2.11.0`.
- Lockfiles should remain consistent with dependency changes.

## Merge impact checklist

When syncing upstream:

1. Read this file before merging.
2. Identify files changed by upstream since the pre-merge commit.
3. Intersect upstream-changed files with `Watched files` above.
4. If the intersection is non-empty, inspect those diffs before declaring the merge safe.
5. Even if there is no file intersection, consider indirect impact from dependency, API, or type changes.
6. Never treat “merge completed without conflicts” as proof that local behavior is unaffected.

## Commit maintenance rule

When Claude creates a commit for this repository:

1. Inspect whether the commit changes fork-specific behavior, watched files, or merge-risk areas.
2. If yes, update this file in the same workflow before committing.
3. Do not rely on the user to request this maintenance explicitly.
