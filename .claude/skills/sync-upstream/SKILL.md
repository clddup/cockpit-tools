---
name: sync-upstream
description: 同步上游：读取 LOCAL_CHANGES.md，确认 main 可跟上游最新 main，但 develop/clddup 只合并上游最新 release tag 对应 commit，并检查是否影响本分支自定义修改。
disable-model-invocation: true
allowed-tools:
  - Read(.claude/LOCAL_CHANGES.md)
  - Bash(git status *)
  - Bash(git remote *)
  - Bash(git config *)
  - Bash(git ls-remote *)
  - Bash(git fetch *)
  - Bash(git branch *)
  - Bash(git rev-parse *)
  - Bash(git merge-base *)
  - Bash(git diff *)
  - Bash(git log *)
  - Bash(git merge *)
---

# 同步上游到 `develop/clddup`

仅当用户明确调用 `/sync-upstream`，或明确要求“同步上游 / 更新 fork / 合并 upstream / 更新 develop/clddup”时使用。

## 固定仓库策略

- 个人 fork 远程仓库：`origin` = `https://github.com/clddup/cockpit-tools.git`
- 上游源仓库：`upstream` = `https://github.com/jlcodes99/cockpit-tools.git`
- 长期维护分支：`develop/clddup`
- 上游 `main`：可以同步到本地/个人 fork 的 `main`，用于保持 fork 的主分支最新
- `develop/clddup` 合并目标：上游最新 release tag 指向的 commit，不合并 tag 之后的 `upstream/main` 新提交
- 不批量 fetch 上游所有 tags；优先用 `git ls-remote --tags --refs` 查询远程 tag
- 除非用户明确要求，否则不要 push 分支或 tag
- 如果将来需要打包发布，使用用户自己的 tag，例如 `v0.24.3-clddup.1`，不要 push 上游原始 tag 名

## 必须执行的流程

1. 先读取 `.claude/LOCAL_CHANGES.md`，把它作为本分支自定义行为、watched files、提交维护规则的依据。

2. 简短说明将执行的策略：`main` 可跟上游最新 `main`，但 `develop/clddup` 只合并上游最新 release tag 对应 commit，并会用 `.claude/LOCAL_CHANGES.md` 检查影响范围。

3. 检查工作区是否干净：

   ```bash
   git status --short
   ```

   如果有任何输出，停止。报告变更文件，让用户先提交、stash 或处理。不要继续合并。

4. 检查当前分支：

   ```bash
   git branch --show-current
   ```

   如果不是 `develop/clddup`，停止并询问用户是否切换分支。除非用户明确要求，不要自动 checkout。

5. 确保 `upstream` remote 存在并指向固定 URL：

   ```bash
   git remote get-url upstream
   ```

   - 如果 `upstream` 不存在，添加：

     ```bash
     git remote add upstream https://github.com/jlcodes99/cockpit-tools.git
     ```

   - 如果 `upstream` 已存在但指向其他 URL，停止并报告当前 URL 和期望 URL，询问用户是否更新。不要擅自改已有 remote URL。

6. 获取上游 `main`，但不要批量 fetch tags：

   ```bash
   git fetch upstream main
   ```

7. 查询上游 release tags，排除用户自己的 fork release tag 后缀：

   ```bash
   git ls-remote --tags --refs --sort='v:refname' upstream 'v*'
   ```

   从输出中选择最新的上游 release tag。忽略包含 `clddup` 的 tag。记录：

   - `LATEST_UPSTREAM_TAG`
   - `LATEST_UPSTREAM_TAG_COMMIT`

8. 只 fetch 最新 release tag 对应对象，不批量导入所有上游 tags：

   ```bash
   git fetch upstream tag LATEST_UPSTREAM_TAG
   ```

9. 验证最新 release tag commit 在 `upstream/main` 历史中。必须在 fetch tag 后验证，避免本地没有 tag 对象时误判：

   ```bash
   git merge-base --is-ancestor LATEST_UPSTREAM_TAG_COMMIT upstream/main
   ```

   如果不是 ancestor，停止并报告异常，不要合并。

10. 记录当前提交：

    ```bash
    git rev-parse HEAD
    ```

    记为 `PRE_MERGE_HEAD`。

11. 合并最新 release tag 对应 commit，而不是合并 `upstream/main` 最新提交：

    ```bash
    git merge LATEST_UPSTREAM_TAG_COMMIT
    ```

    如果冲突，立即停止并报告：

    ```bash
    git status --short
    git diff --name-only --diff-filter=U
    ```

    不要自动解决冲突。

12. 合并成功后，列出本次上游引入的文件：

    ```bash
    git diff --name-only PRE_MERGE_HEAD..HEAD
    git diff --stat PRE_MERGE_HEAD..HEAD
    ```

13. 将上一步文件列表与 `.claude/LOCAL_CHANGES.md` 的 `Watched files` 做交集，明确报告：

    - 是否触碰 watched files
    - 触碰了哪些 watched files
    - 是否需要重点 review

    如果交集非空，即使 Git 没冲突，也必须标记为“需要重点检查”。

14. 如果 watched files 交集非空，必须立即对这些文件做重点检查，不要等用户再次提醒：

    ```bash
    git diff PRE_MERGE_HEAD..HEAD -- WATCHED_FILE_1 WATCHED_FILE_2 ...
    ```

    检查重点必须覆盖 `.claude/LOCAL_CHANGES.md` 中对应行为区域，至少包括：

    - Codex JSON/token 批量导入不能因单项失败导致整批失败
    - `codex:json-import-progress` 进度事件与前端 `current/total` 展示
    - Codex profile hydration 的并发限制、buffer、flush 行为
    - Codex 异常筛选拆分：`AUTH_ERROR`、`QUOTA_ERROR`、`REFRESH_FAILED`
    - `requires_reauth`、`quota_error`、`error sending request`、`API 返回错误` 的分类语义
    - 各 provider `remove_accounts` 批量删除路径，避免恢复逐账号 index read/write 循环
    - Codex 删除 `current_account_id` 与 API Key 绑定 OAuth 清理逻辑
    - `@tauri-apps/api`、`Cargo.lock`、`pnpm-lock.yaml` 与 Tauri 版本对齐

    若出现以下文件，必须在最终报告中列为“已重点检查 / 仍需人工复核”之一：

    - `Cargo.lock`
    - `package.json`
    - `pnpm-lock.yaml`
    - `src-tauri/src/commands/codex.rs`
    - `src-tauri/src/modules/codex_account.rs`
    - `src-tauri/src/modules/trae_account.rs`
    - `src/pages/CodexAccountsPage.tsx`
    - `src/stores/useCodexAccountStore.ts`

15. 最后报告：

    - `upstream/main` 是否 fetch 成功
    - 最新上游 release tag 是哪个
    - `develop/clddup` 合并到哪个 tag commit
    - 是否出现冲突
    - 是否触碰 `.claude/LOCAL_CHANGES.md` 中的 watched files
    - watched files 交集非空时，重点检查结论是什么，哪些文件仍需人工复核
    - 是否未 push，以及如需打包应由用户确认后 push 自己的 `*-clddup.*` tag
