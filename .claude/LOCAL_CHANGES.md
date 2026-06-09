# Local Changes for `develop/clddup`

This file records fork-specific behavior maintained on `develop/clddup` so upstream syncs can check whether new upstream changes affect local customizations.

## Branch policy

- Long-lived fork branch: `develop/clddup`
- Upstream baseline: latest upstream release tag commit on `main`
- Main branch policy: keep `main` aligned with latest upstream `main`
- Development branch sync policy: merge the latest upstream release tag commit into `develop/clddup`, not commits after that tag on `upstream/main`
- Upstream sync merge commits should use a custom message such as `sync upstream release v0.24.13`, not Git's default `Merge commit '...' into develop/clddup` message
- Upstream remote URL: `https://github.com/jlcodes99/cockpit-tools.git`
- Fork release tag policy: use user-owned tags such as `v0.24.3-clddup.1`; do not push upstream's original tag names to `origin`

## Current custom commits

These commits describe the current local customization set relative to the latest upstream release baseline:

- `bdb9d75 optimize account batch deletion`
- Codex JSON/token batch import should skip per-item failures instead of aborting the entire batch.
- Codex abnormal account filters are split into `AUTH_ERROR`, `QUOTA_ERROR`, and `REFRESH_FAILED`.

## Watched files

When syncing upstream, compare upstream-changed files against this list first:

- `src-tauri/src/modules/codex_account.rs`
- `src-tauri/src/modules/codebuddy_account.rs`
- `src-tauri/src/modules/codebuddy_cn_account.rs`
- `src-tauri/src/modules/cursor_account.rs`
- `src-tauri/src/modules/gemini_account.rs`
- `src-tauri/src/modules/github_copilot_account.rs`
- `src-tauri/src/modules/kiro_account.rs`
- `src-tauri/src/modules/qoder_account.rs`
- `src-tauri/src/modules/trae_account.rs`
- `src-tauri/src/modules/windsurf_account.rs`
- `src-tauri/src/modules/workbuddy_account.rs`
- `src-tauri/src/modules/zed_account.rs`
- `src/pages/CodexAccountsPage.tsx`

## Custom behavior areas

### Codex import partial success

Watched files:

- `src-tauri/src/modules/codex_account.rs`

Behavior to preserve:

- Codex JSON/token batch import should not fail the entire batch when one item fails.
- Failed import items should be skipped and collected into user-visible failure messages.
- The persist stage (`import_prepared_codex_candidates_batch`) must also skip per-item failures (e.g. invalid JWT id_token) instead of aborting the whole batch with `?`; it returns `(accounts, failures)` and only saves the index when at least one account succeeded.

Symbols and strings to watch:

- `import_codex_candidate`
- `import_accounts_from_token_lines`
- `import_from_json`
- `import_prepared_codex_candidates_batch`
- `LabeledPreparedCodexJsonImportCandidate`
- `codex:json-import-progress`
- `跳过失败项`
- `跳过落盘失败项`

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

### Account batch deletion performance

Watched files:

- `src-tauri/src/modules/codebuddy_account.rs`
- `src-tauri/src/modules/codebuddy_cn_account.rs`
- `src-tauri/src/modules/codex_account.rs`
- `src-tauri/src/modules/cursor_account.rs`
- `src-tauri/src/modules/gemini_account.rs`
- `src-tauri/src/modules/github_copilot_account.rs`
- `src-tauri/src/modules/kiro_account.rs`
- `src-tauri/src/modules/qoder_account.rs`
- `src-tauri/src/modules/trae_account.rs`
- `src-tauri/src/modules/windsurf_account.rs`
- `src-tauri/src/modules/workbuddy_account.rs`
- `src-tauri/src/modules/zed_account.rs`

Behavior to preserve:

- Account deletion should use one shared batch path per provider.
- Single-account deletion should wrap the id and delegate to batch deletion.
- Batch deletion should normalize and deduplicate account ids before touching storage.
- Batch deletion should update the account index once, then delete account detail files.
- Codex batch deletion should clear `current_account_id` when the current account is removed.
- Codex batch deletion should scan remaining accounts once to clear API Key accounts bound to removed OAuth accounts.
- Avoid reintroducing per-account index read/write loops in provider `remove_accounts` implementations.

Symbols and strings to watch:

- `remove_account`
- `remove_accounts`
- `target_ids`
- `HashSet<String>`
- `bound_oauth_account_id`
- `current_account_id`

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
