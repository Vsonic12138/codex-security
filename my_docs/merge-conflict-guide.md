# 上游合并与冲突处理指南

本文档记录 `custom/openai-compatible-provider` 分支与上游官方代码库（`openai/codex-security`）合并时的**冲突机制、成因与标准解决步骤**。

---

## 1. 为什么合并时容易产生冲突？

GitHub Actions 每 6 小时自动把官方最新提交同步到 `official-main` 和 `main`，而二开分支需要手动合并。

```text
┌──────────────────────────────────────────────────────────┐
│                   分支同步与冲突产生位置                 │
└──────────────────────────────────────────────────────────┘

  upstream (openai/codex-security)
     │
     ▼ (Actions 每 6 小时自动同步)
┌──────────────────────────┐
│ origin/official-main     │ ◄── 严格镜像官方（无任何冲突）
└─────────────┬────────────┘
              │
              ▼ (Actions 自动 fast-forward / merge)
┌──────────────────────────┐
│ origin/main              │ ◄── 默认主分支（无业务代码冲突）
└─────────────┬────────────┘
              │
              ▼ ⚡ 手动合并点 (在此处产生定制代码与官方代码的冲突)
┌──────────────────────────┐
│ custom/openai-compatible │ ◄── 本地二开分支 (包含包名/Provider 定制)
└──────────────────────────┘
```

### 必然冲突点：
1. **`sdk/typescript/package.json`（版本与包名）**
   - 官方每次发版会修改 `"version"`（如 `0.1.14` -> `0.1.15`），其包名是 `"@openai/codex-security"`。
   - 二开分支的包名为 `"@bigyellow12138/codex-security"`。
   - Git 在同一行的修改无法自动合并，**只要官方发版，此处必冲突**。

### 潜在冲突点（取决于官方改动范围）：
2. **`sdk/typescript/src/api.ts`（Provider 初始化入口）**
   - 二开分支在 `#prepareSession` 中注入了 `codexProviderConfig`。如果官方重构会话初始化流程，会出现冲突。
3. **`sdk/typescript/tests-ts/*.test.ts`（测试用例末尾追加）**
   - 双方都在测试文件末尾添加新 `test(...)` 块时，Git 可能产生块结尾重叠冲突。

---

## 2. 标准合并步骤

### 步骤 1：拉取最新代码并发起合并

```bash
# 1. 切换到二开分支
git switch custom/openai-compatible-provider

# 2. 拉取远端最新提交
git fetch origin

# 3. 合并 main（或 official-main）
git merge origin/main
```

---

### 步骤 2：解决典型冲突

#### 冲突 A：`package.json`
保留二开定制的包名，但**接受官方升级后的版本号**：

```json
<<<<<<< HEAD
  "name": "@bigyellow12138/codex-security",
  "version": "0.1.14",
=======
  "name": "@openai/codex-security",
  "version": "0.1.15",
>>>>>>> origin/main
```
➡️ **解决后保留**：
```json
  "name": "@bigyellow12138/codex-security",
  "version": "0.1.15",
```

---

#### 冲突 B：`src/api.ts`
检查 `#prepareSession` 内的 `externalProvider` 变量声明：

```typescript
// 确保使用 codexProviderConfig 动态加载（支持 "custom" provider）：
const externalProvider = isExternalModelProvider(modelProvider)
  ? codexProviderConfig(modelProvider, this.#dependencies.environment)
  : null;
```

并确认 `PreparedSession` 接口中 `externalProvider` 的类型定义为：
```typescript
externalProvider: CodexProviderConfig | null;
```

---

#### 冲突 C：`tests-ts/*.test.ts`
若测试文件末尾冲突，通常为双方独立添加的测试用例。**将双方的 `test(...)` 块全部保留**，并确保每个测试用例的 `});` 括号正常闭合。

---

### 步骤 3：本地编译与测试验证

冲突解决后，必须在本地运行类型检查与核心测试，确保构建无破坏：

```bash
cd sdk/typescript

# 1. 类型检查
pnpm run types

# 2. 核心单元测试（针对 Custom Provider 与 CLI）
bun test tests-ts/config.test.ts tests-ts/api-preflight-config.test.ts tests-ts/cli.test.ts tests-ts/update-notice.test.ts

# 3. 产物构建打包
pnpm run build

# 4. CLI 冒烟测试
node ./bin/codex-security.mjs scan . --dry-run
```

---

### 步骤 4：提交并推送到远端

```bash
cd /home/vsonic/workspace/test_proj/codex-security

git add -A
git commit -m "Merge branch 'main' into custom/openai-compatible-provider"
git push origin custom/openai-compatible-provider
```
