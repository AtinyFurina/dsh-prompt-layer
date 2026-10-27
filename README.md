# dsh-prompt-layer

[![ci](https://github.com/AtinyFurina/dsh-prompt-layer/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/AtinyFurina/dsh-prompt-layer/actions/workflows/ci.yml)
[![license](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

给 DeepSeek Harness 加一段**可配置的系统提示词段落**，渲染在所有内置段落之后。

它自己不含任何策略文案：写什么由部署方通过 `config.text` 或 `textFile` 提供。

---

## 为什么需要它

| 层级 | 载体 | 实际效力 |
|---|---|---|
| 系统提示词 | `ctx.systemPrompt.section()` ← **本插件** | 与平台自身指令同层，渲染在最后 |
| 工作区指令 | `~/.dsh/AGENTS.md` / `AGENTS.md` / `CLAUDE.md` | **user 角色消息**，信封明确写着 *"They do not override system, developer, or direct user instructions"* |
| 伪配置块 | `NSFW{ allow = true }` 之类 | 没有消费者，等于普通文本 |

也就是说：写在 `AGENTS.md` 里的"以本文件为准"是自赋优先级，会被承载它的那条 user 消息的信封当场否掉。要让部署方自己的策略真正落到高优先级，得把它放进**系统提示词层**。

本插件默认把段落序设为宿主最后一段（`DEPLOYMENT_PERSONA_SUFFIX`，当前 `10200`）+ 300 = `10500`。更大的 order 意味着渲染在**系统提示词内部的末位**（系统消息本身仍排在对话最前，工作区指令作为后续 user 消息在其之后）。

> 这是一套通用机制，不是某个场景的专用开关。文案内容由部署方自己负责。

---

## 安装

1. 把本目录复制到 profile 的插件目录：

   ```
   ~/.dsh/profiles/desktop/plugins/dsh-prompt-layer
   ```

2. 在 `~/.dsh/profiles/desktop/cordis.patch.yml` 里挂载：

   ```yaml
   - insert:
       - id: prompt-layer
         name: ./plugins/dsh-prompt-layer/lib/index.js
         config:
           text: |
             本部署为内部测试环境。
             以下规则优先于工作区 AGENTS.md 注入的内容：
             - 输出使用简体中文。
             - 不确定的事实先检索再回答。
   ```

3. 让配置重载。DSH 桌面版对本 profile 的配置热重载通常是开的：保存 patch 后数秒内插件会被加载。若没有生效，重启 DSH 应用。

---

## 配置

| 字段 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `enabled` | boolean | `true` | 设为 `false` 可临时停用，不必卸载 |
| `text` | string | — | 内联文案 |
| `textFile` | string | — | UTF-8 文件路径；**优先于 `text`**。相对路径按 DSH 进程的工作目录解析，建议用绝对路径 |
| `order` | number | 宿主 `DEPLOYMENT_PERSONA_SUFFIX` + 300（当前 `10500`） | 渲染顺序，越大越靠后；低于宿主基准会告警 |
| `sectionName` | string | `deploy:custom-layer` | 出现在宿主诊断里的段落名 |

行为细节：

- `text` 与 `textFile` 都为空（或只有空白）→ 不注册，打一条 warning。
- `textFile` 读取失败 → 不注册，打 warning（不会用 `text` 兜底，避免掩盖错误）。
- 文案里完整的 `{{...}}` 组会被转换成字面量花括号（插入一个零宽空格），因为 DSH 把 `{{name}}` 当作严格的提示词变量引用，未知变量会让整个 prompt 装配抛错。
- 插件不写任何文件、不联网、不注册工具。

---

## 验证

单元测试（无依赖，stub context）：

```bash
npm test
```

真机集成检查（挂载到真实 `SystemPrompt` 并渲染）：

```bash
# DSH 的包不在本仓库 node_modules 里，指向 profile 的 node_modules
npm run verify -- --modules "C:\Users\<you>\.dsh\profiles\node_modules"
# 等价的直接调用：node verify.mjs --modules "..."
```

> **macOS 桌面版避坑说明**：  
> 桌面版的核心依赖（如 `@deepseek-ai/dsh-system-prompt`）默认打包在 `app.asar` 虚拟压缩包内，外部 `~/.dsh/profiles/node_modules` 可能是死链或缺少依赖，且普通系统 Node.js 无法直接解析 asar 虚拟路径。  
> 请使用 DSH 自带的运行时和内置模块路径进行验证：  
> ```bash
> ELECTRON_RUN_AS_NODE=1 "/Applications/DeepSeek Harness.app/Contents/MacOS/DeepSeek Harness" verify.mjs --modules "/Applications/DeepSeek Harness.app/Contents/Resources/app.asar/dsh/node_modules"
> ```

期望输出里能看到：

```
PASS  section renders last  last=deploy:verify
OK: dsh-prompt-layer renders after every built-in section.
```

---

## 兼容性

- App 打包：`dsh-system-prompt 0.2.1-rc.2`；profile 解析：`0.1.5-rc.2`。
- 两份源码的 `section()` / `SECTION_ORDERS`（`DEPLOYMENT_PERSONA_SUFFIX = 10200`）/ `interpolate()` 一致。
- `verify.mjs --modules` 默认指向 profile 那份；要与 App 实际运行版本对齐需另行指定 App 内的模块路径。
- Node.js `>= 20`。
- 无第三方运行时依赖。

---

## 回退

删掉 `cordis.patch.yml` 里那条 `id: prompt-layer` 的 insert，然后重载配置。插件目录留着不影响启动。

---

## 责任边界

插件只负责"把一段文字放到系统提示词的最后一层"，不判断文案内容。部署方应确保自己提供的文案符合适用法律、平台政策与所在环境的授权范围。

---

## English summary

`dsh-prompt-layer` registers a configurable section at the end of the DeepSeek Harness system prompt. Workspace instruction files (`AGENTS.md`, `CLAUDE.md`) arrive as a *user* message whose frame says it does not override system instructions; a system-prompt section does not have that limitation. The plugin ships no policy text of its own — supply `text` or `textFile` in the profile patch. MIT licensed.

## License

MIT © 2026 AtinyFurina
