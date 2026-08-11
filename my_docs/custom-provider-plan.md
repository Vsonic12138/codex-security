# 自定义 OpenAI-compatible Provider 改造计划

## 目标

增加一个可配置的第三方推理渠道（`--provider custom`），指向任意兼容
OpenAI **Responses API** 的网关，用于安全扫描推理。

## 已确认的事实（2026-08-11）

- 当前随包 `@openai/codex` v0.144.6 原生运行时**仅支持 Responses 协议**：
  `wire_api = "chat" is no longer supported`（官方已废弃 Chat Completions）。
- 目标网关 newapi（`https://newapi.vsonic12138.shop/v1`）已实测**完整支持
  Responses 协议**：`/v1/responses` 返回标准 `response` 对象、SSE 流式、
  函数工具调用均正常。
- 可用模型：`gpt-5.6-sol`、`gpt-5.6-terra`、`gpt-5.6-luna`。

## 方案（A：Responses-only + fail-fast 拒绝 chat）

```bash
export CUSTOM_PROVIDER_BASE_URL="https://newapi.vsonic12138.shop/v1"
export CUSTOM_PROVIDER_API_KEY="<key>"
codex-security scan . --provider custom --model gpt-5.6-sol --effort xhigh
```

运行时投影：

```toml
[model_providers.custom]
name = "Custom OpenAI-compatible provider"
base_url = "https://newapi.vsonic12138.shop/v1"
env_key = "CUSTOM_PROVIDER_API_KEY"
wire_api = "responses"
```

- 若用户显式设置 `CUSTOM_PROVIDER_WIRE_API="chat"`，在 preflight 阶段
  fail-fast，提示改用 Responses 兼容端点或协议转换代理（如 LiteLLM）。
- 密钥仅存环境变量；扫描子进程会清理其他 provider 的 API Key，防止串泄露。

## 涉及文件（按现有代码风格）

| 文件 | 改动 |
|---|---|
| `sdk/typescript/src/config.ts` | 新增 custom provider 工厂 + base_url/协议校验 |
| `sdk/typescript/src/cli.ts` | `--provider` 增加 `custom`；强制 `--model` |
| `sdk/typescript/src/api.ts` | 认证来源、环境隔离、运行时 TOML 投影 |
| `tests-ts/*.test.ts` | 正反向用例（缺 key、URL 校验、协议拒绝、投影） |
| `sdk/typescript/README.md` | 使用示例 + Responses 说明 |

## 验收

- `--provider custom` 可解析；缺 key/base_url 在 preflight 报清晰错误
- preflight 投影含 `base_url`/`env_key`/`wire_api="responses"`
- 运行时环境只含 `CUSTOM_PROVIDER_API_KEY`，不泄露其他凭据
- 新增测试通过；`pnpm test`、`pnpm run types`、`pnpm run format` 全绿
