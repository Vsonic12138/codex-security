# Custom Provider 验证记录（2026-08-12）

端到端验证：`--provider custom` + 自建 OpenAI-compatible 网关 + Responses 协议。

## 验证环境

| 项 | 值 |
|---|---|
| 网关 | `https://newapi.vsonic12138.shop/v1`（Responses 协议，已实测） |
| API Key | 环境变量 `CUSTOM_PROVIDER_API_KEY`（不落盘、不进 git） |
| runtime | bundled `@openai/codex` v0.144.6（codex-security 自带） |
| 测试目标 | 临时小仓库（故意植入 4 个漏洞：SQL 注入/硬编码 token/命令注入/路径遍历） |

## 测试结果

### 1. 凭据有效性（curl /v1/responses）
- `gpt-5.6-luna` / `gpt-5.6-terra` 均返回标准 `response` 对象，协议链路通。

### 2. CLI dry-run（0 消耗）
```bash
export CUSTOM_PROVIDER_BASE_URL="https://newapi.vsonic12138.shop/v1"
export CUSTOM_PROVIDER_API_KEY="<YOUR_API_KEY>"
codex-security scan . --provider custom --model gpt-5.6-terra --dry-run --json
```
输出确认：`modelProvider: custom`、`authentication: api_key (CUSTOM_PROVIDER_API_KEY)`。

### 3. 端到端真实扫描（gpt-5.6-terra --effort high）
```
FINDINGS  4 (2 high, 2 medium)    COVERAGE complete
ELAPSED   5m 53s                  COST $0.54
```
- SQL injection in username lookup（high, CVSS 7.5）✅
- OS command injection in deployment helper（high, CVSS 7.8）✅
- Hardcoded administrator token（medium, CVSS 5.9）✅
- Path traversal in config reader（medium, CVSS 5.3）✅

4 个植入漏洞全部命中，产物完整（findings.json / report.md / coverage.json / scan-manifest.json）。

## 发现的问题与修复

### 问题 1：WSL2 缺 `codex-linux-sandbox`（环境问题，非代码问题）
codex 0.144.6 权限系统执行工具需要 `codex-linux-sandbox`（多入口二进制），npm 平台包未携带。
现象：扫描卡在 preflight，agent 报 `bwrap: execvp codex-linux-sandbox: No such file or directory`。

**修复**（symlink 指向 codex 二进制，arg0 检测进入 sandbox 模式）：
```bash
ln -sf <path-to-bundled-codex> ~/.local/bin/codex-linux-sandbox
# 例：ln -sf ~/workspace/Test_Proj/codex-security/sdk/typescript/node_modules/.pnpm/@openai+codex@0.144.6-linux-x64/node_modules/@openai/codex/vendor/x86_64-unknown-linux-musl/bin/codex ~/.local/bin/codex-linux-sandbox
```
验证：`codex exec` 的权限沙箱工具执行恢复正常。

### 问题 2：gpt-5.6-luna 不适用安全扫描（模型能力问题）
- luna 在 reviewing 阶段循环不收敛（10 分钟 Files 0/2、$0.12 空耗）。
- **推荐 `gpt-5.6-terra --effort high`**（与日常 ~/.codex/config.toml 一致）；`gpt-5.6-sol` 未测，理论可用。

## 使用示例（推荐配置）

```bash
export CUSTOM_PROVIDER_BASE_URL="https://newapi.vsonic12138.shop/v1"
export CUSTOM_PROVIDER_API_KEY="<YOUR_API_KEY>"
codex-security scan /path/to/repo \
  --provider custom --model gpt-5.6-terra --effort high \
  --max-cost 1.0 --headless
```

## 备注

- 扫描用 git 快照（git_worktree），工作区未提交改动不进入扫描。
- Python 插件需要 3.10+（3.10 需 tomli）；本机用 `--python ~/.local/bin/python3.12`（uv 管理，原生 tomllib）。
