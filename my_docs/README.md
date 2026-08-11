# Codex Security 二开说明

本仓库为 OpenAI `openai/codex-security` 的二开维护分支，通过 GitHub Actions
自动同步上游，同时在独立分支上承载本地定制。

## 分支模型

| 分支 | 角色 | 同步方式 |
|---|---|---|
| `official-main` | 官方纯净镜像（仅镜像上游内容） | workflow 每 6 小时自动 `branch -f` + push |
| `main`（默认） | 官方内容 + 同步 workflow 载体 | workflow 自动 merge 上游更新 |
| `custom/openai-compatible-provider` | **二开分支** | 手动 merge `official-main` |

## 同步机制

- 仓库：`Vsonic12138/codex-security`（公开）
- 上游：`https://github.com/openai/codex-security.git`
- workflow：`.github/workflows/sync-upstream.yml`，cron `17 */6 * * *`（UTC）
- 手动触发：`gh workflow run sync-upstream.yml -R Vsonic12138/codex-security`
- 同步使用 GitHub 自动的 `GITHUB_TOKEN`（contents: write），不依赖个人凭据

## 二开工作流

```bash
# 在二开分支开发
git switch custom/openai-compatible-provider

# 需要合并官方更新时（official-main 已被 workflow 自动更新）
git fetch origin official-main
git merge official-main
# 解决冲突 → 提交 → push
```

## 当前二开内容

- 自定义 OpenAI-compatible inference provider（详见 `custom-provider-plan.md`）
