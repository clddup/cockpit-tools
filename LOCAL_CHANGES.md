# Local Changes

## 2026-06-24 Codex 账号配额刷新空数据问题

### 现象

- 账号 `50****8@q*.com` 刷新后一直显示“暂无配额数据”。
- 其他 Codex 账号刷新正常。
- 该账号本地状态没有 `quota`，也没有 `quota_error`，导致 UI 看起来像“无数据”而不是“刷新失败”。

### 根因

- 日志中该账号刷新 token 失败，错误码为 `invalid_refresh_token`。
- 旧逻辑在 `prepare_account_for_injection(account_id)` 阶段失败后直接返回错误，没有把错误写入账号的 `quota_error`。
- 前端刷新入口捕获异常后只 `console.error`，但由于后端未落库 `quota_error`，页面没有可展示的账号级错误状态。

### 修改

- `src-tauri/src/modules/codex_account.rs`
  - 将 `invalid_refresh_token` / `invalid refresh_token` 归类为需要重新登录的授权错误。
  - 增加单测覆盖 `invalid_refresh_token` 的用户可读错误提示。
- `src-tauri/src/modules/codex_quota.rs`
  - 在配额刷新准备阶段失败时，重新加载账号并写入 `quota_error`，再返回错误。
- `src/utils/codexQuotaError.ts`
  - 将 `invalid_refresh_token` 识别为阻塞型 Codex 配额错误。
- `src/pages/CodexAccountsPage.tsx`
  - `invalid_refresh_token` 时提供重新授权动作。
  - Codex 配额刷新全失败时不再把长错误渲染到顶部全局红条。

### 验证

- `npm run typecheck` 通过。
- `cargo check` 通过。
- `cargo test -p cockpit-tools formats_refresh_errors_with_actionable_reason --lib` 通过。

### 结论

- 这次没有写入 `.claude/` 或 `.claude/settings.local.json`。
- 该账号的 `refresh_token` 已失效，代码修复只能让 UI 正确显示授权异常和重新登录入口，不能恢复已失效 token。
- 需要对该 Codex 账号重新 OAuth 登录后才能重新拉取配额数据。
