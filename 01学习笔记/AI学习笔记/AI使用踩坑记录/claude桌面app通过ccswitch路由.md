```
claude code的windows桌面版app现在可以用开发者模式接入第三方模型，比如我接入了deepseek

但是配置完打开claude app后报错Your provider setup needs a fix  Some required fields are missing or malformed. Open Setup to finish configuring it.

于是我找到了这篇文章并成功按照步骤完成了修复（这篇文章里还有其他报错修复，先不关注，先关注我提到的这个）

https://www.pmtemple.com/artificial-intelligence/18413/

但是我没搞懂原因，稀里糊涂的就在claude code里用上了deepseek v4

请结合目前最新的windows系统使用claude桌面app接入第三方模型的新闻、开发者社区讨论等，帮我梳理一下本次问题
```

## 结论

你这次不是“瞎配成功了”，而是踩到了 **Claude Desktop / Claude Code 第三方推理接入的三个兼容层问题**：

1. **Claude Code 内部仍然按 Claude/Anthropic 的模型体系工作**：`opus`、`sonnet`、`haiku`、`claude-xxx` 这些是它认识的路由名。
    
2. **DeepSeek 虽然提供 Anthropic-compatible endpoint，但模型名、鉴权方式、工具调用格式、模型发现接口不一定完全符合 Claude Desktop 期望。**
    
3. 你文章里用的 `ccswitch` 本质上是一个 **本地 LLM Gateway / 兼容代理**：它把 Claude Desktop 想看到的“Claude 模型名/Claude 路由”映射到真实的 DeepSeek 模型，同时补齐协议差异。
    

所以报错：

```text
Your provider setup needs a fix
Some required fields are missing or malformed.
Open Setup to finish configuring it.
```

本质不是 DeepSeek API Key 一定错了，而是 **Claude Desktop 认为你的第三方 Provider 配置不符合它当前版本的 schema / model routing / gateway discovery 规则**。

---

# 1. 先区分几个概念

你现在用到的东西容易混在一起：

|名称|作用|
|---|---|
|Claude Desktop App|Windows/macOS 桌面端，里面现在有 Chat、Cowork、Code 等入口|
|Claude Code|面向代码仓库的 agentic coding 工具，原来主要是 CLI，现在也集成进 Desktop 的 Code tab|
|Developer / Third-Party Inference|允许把推理请求路由到非 Anthropic 官方后端|
|DeepSeek Anthropic API|DeepSeek 提供的 Anthropic Messages API 兼容接口|
|ccswitch / local gateway|本地代理层，负责模型映射、协议转换、鉴权转发|

Anthropic 官方文档确认：Claude Desktop 的 Code tab 是 Claude Code 的桌面端入口；Windows 版 Desktop App 已提供下载，Windows 首次使用 Code tab 还要求安装 Git for Windows。([Claude](https://code.claude.com/docs/en/desktop "Use Claude Code Desktop - Claude Code Docs"))

DeepSeek 官方也给了 Claude Code 接入方式：Windows 需要 Git for Windows，并通过环境变量设置 `ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic`、`ANTHROPIC_AUTH_TOKEN=<DeepSeek API Key>`、`ANTHROPIC_MODEL=deepseek-v4-pro[1m]` 等。([DeepSeek API 文档](https://api-docs.deepseek.com/quick_start/agent_integrations/claude_code "Integrate with Claude Code | DeepSeek API Docs"))

---

# 2. 为什么会报 “Some required fields are missing or malformed”

结合你贴的文章和官方文档，最可能的原因是：

## 2.1 新版 Claude 对模型路由更严格了

你看的文章给出的判断是：新版 Claude Desktop 加强了模型名路由校验；旧版 `inferenceModels` 可以写 `qwen`、`deepseek` 之类自定义名，新版则要求 Model list 里使用 Anthropic provider catalog 里的 Claude 路由名。([PmTemple](https://www.pmtemple.com/artificial-intelligence/18413/ "保姆级教程修复Claude desktop（claude桌面版）第三方开发者模式无法使用国内模型的问题，另附Claude desktop（claude桌面版）第三方开发者模式开启“Bypass permissions”疯狗模式教程 | 产品研究院PmTemple"))

这解释了为什么你之前可能直接填：

```text
deepseek-v4-pro
deepseek-v4-flash
qwen3.6-plus
```

然后 Desktop 认为字段“不合法”或“格式不对”。

换句话说，**Claude Desktop 的模型选择器不只是一个字符串输入框**。它背后有模型 alias、能力识别、上下文窗口、工具调用、thinking、权限模式等逻辑。Claude Code 官方文档里也说明了模型 alias，例如：

```text
opus
sonnet
haiku
default
best
```

以及 `ANTHROPIC_DEFAULT_OPUS_MODEL`、`ANTHROPIC_DEFAULT_SONNET_MODEL`、`ANTHROPIC_DEFAULT_HAIKU_MODEL` 这些环境变量用于把 Claude 的模型家族 alias 映射到具体 provider 模型。([Claude](https://code.claude.com/docs/en/model-config "Model configuration - Claude Code Docs"))

所以，Claude Desktop 想看到的是：

```text
我选择的是 opus / sonnet / haiku / claude-xxx
```

而不是：

```text
我选择的是 deepseek-v4-pro
```

后者对 DeepSeek 来说合法，但对 Claude Desktop 的某些配置校验逻辑来说可能不合法。

---

## 2.2 Gateway 模式要求接口形态符合 Claude Code 预期

Claude Code 官方 LLM Gateway 文档说得很明确：网关至少要暴露 Anthropic Messages API 形式的 `/v1/messages`、`/v1/messages/count_tokens`，并且需要正确转发 `anthropic-beta`、`anthropic-version` 等 header。否则可能导致功能降级或无法使用。([Claude](https://code.claude.com/docs/en/llm-gateway "LLM gateway configuration - Claude Code Docs"))

同时，Claude Code 在 Gateway 模式下还可以尝试访问 `/v1/models` 做模型发现，但这个功能有严格过滤：只有 ID 以 `claude` 或 `anthropic` 开头的模型才会加入 picker；如果网关的模型名不是这种格式，就需要用模型配置手动添加或映射。([Claude](https://code.claude.com/docs/en/llm-gateway "LLM gateway configuration - Claude Code Docs"))

这就解释了一个关键点：

```text
DeepSeek 的模型名是 deepseek-v4-pro
但 Claude Desktop / Claude Code 的 gateway discovery 可能只接受 claude* / anthropic* 形态的模型进入模型选择器
```

所以直接接 DeepSeek 时，模型可能“API 上存在”，但 Desktop 的 Setup 校验层不认。

---

## 2.3 鉴权方式也可能不一致

Claude Code 的环境变量文档说明：

```text
ANTHROPIC_API_KEY  -> X-Api-Key header
ANTHROPIC_AUTH_TOKEN -> Authorization: Bearer <token>
ANTHROPIC_BASE_URL -> 改写 API endpoint
```

也就是说，Claude Code 可以用两种不同的鉴权头。([Claude](https://code.claude.com/docs/en/env-vars "Environment variables - Claude Code Docs"))

但不同第三方平台可能要求：

```text
Authorization: Bearer xxx
```

或：

```text
x-api-key: xxx
```

社区里也有人讨论过 Claude Desktop 3P Gateway + DeepSeek 时存在 bearer / x-api-key / Anthropic-compatible protocol 的兼容问题；其中一个 workaround 是本地 bridge 暴露 `/v1/models`，把 Claude Desktop 的 `/v1/messages` 请求转换成 DeepSeek 或其他模型需要的格式。([GitHub](https://github.com/anomalyco/opencode/issues/25144 "go订阅：Claude Desktop win 3p 模式 用deepseek报错不支持 bearer模式 · Issue #25144 · anomalyco/opencode · GitHub"))

所以你看到的错误虽然显示为 “required fields missing or malformed”，但底层可能包括：

```text
模型名不合规
base URL 不合规
鉴权 header 不合规
/v1/models 不符合预期
/v1/messages 返回格式不符合预期
工具调用格式不兼容
```

---

# 3. ccswitch 为什么能修好

你文章里的核心步骤是：

1. 在 ccswitch 里填真实模型供应商 API Key 和 Anthropic-compatible 请求地址；
    
2. 设置模型映射，例如把真实模型映射到 Opus / Sonnet；
    
3. 在 Claude Desktop 里选择 Gateway 模式；
    
4. Gateway base URL 填 `http://127.0.0.1:15721`；
    
5. Model list 不再填真实第三方模型名，而是填 Claude 自己的模型名；
    
6. Apply locally 后重启。([PmTemple](https://www.pmtemple.com/artificial-intelligence/18413/ "保姆级教程修复Claude desktop（claude桌面版）第三方开发者模式无法使用国内模型的问题，另附Claude desktop（claude桌面版）第三方开发者模式开启“Bypass permissions”疯狗模式教程 | 产品研究院PmTemple"))
    

这背后的原理是：

```text
Claude Desktop
  以为自己在调用一个 Claude-compatible gateway
        ↓
ccswitch 本地网关
  接收 Claude Desktop 的模型名：opus / sonnet / claude-xxx
  映射成真实模型名：deepseek-v4-pro / deepseek-v4-flash
  修正请求路径、鉴权、模型字段、部分返回格式
        ↓
DeepSeek API
```

也就是：

```text
Claude Desktop 看到的是“合法 Claude 路由名”
DeepSeek 收到的是“真实 DeepSeek 模型名”
```

所以它能绕过 Desktop 的模型名校验，同时让真实请求发到 DeepSeek。

---

# 4. 你这次问题可以这样理解

你的原始配置大概率是这种逻辑：

```text
Claude Desktop Gateway
base_url = DeepSeek Anthropic endpoint
model = deepseek-v4-pro
api_key = DeepSeek key
```

看起来没错，因为 DeepSeek 官方确实提供 Anthropic-compatible endpoint。

但 Desktop 新版第三方推理配置可能还要求：

```text
model list = Claude provider catalog 里的路由名
gateway = 能按 Claude Code 的预期返回 /v1/models
headers = 符合 Claude Code 的鉴权逻辑
tool schema = 符合 Claude Code 的 Anthropic tool format
```

于是直接写 `deepseek-v4-pro` 被判定为 malformed。

修复后的结构变成：

```text
Claude Desktop Gateway
base_url = http://127.0.0.1:15721
model list = claude/opus/sonnet/haiku 路由名
api_key = 你的 key
        ↓
ccswitch
claude route -> deepseek-v4-pro
        ↓
DeepSeek
```

这就通过了 Desktop 的本地校验，也完成了真实模型调用。

---

# 5. 为什么 “DeepSeek 官方文档说可以接 Claude Code”，但 Desktop 仍可能报错

这是重点。

DeepSeek 官方文档主要讲的是 **Claude Code CLI 接入**：

```powershell
$env:ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
$env:ANTHROPIC_AUTH_TOKEN="<your DeepSeek API Key>"
$env:ANTHROPIC_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_OPUS_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_SONNET_MODEL="deepseek-v4-pro[1m]"
$env:ANTHROPIC_DEFAULT_HAIKU_MODEL="deepseek-v4-flash"
```

这是通过环境变量强制 Claude Code CLI 使用 DeepSeek 模型。([DeepSeek API 文档](https://api-docs.deepseek.com/quick_start/agent_integrations/claude_code "Integrate with Claude Code | DeepSeek API Docs"))

但你现在说的是：

```text
Windows Claude Desktop App
Developer -> Configure Third-Party Inference
Gateway 模式
```

这多了一层 Desktop UI 配置校验和模型选择器逻辑。CLI 环境变量能跑，不代表 Desktop 里的 Setup UI 一定接受同样的模型名。

可以类比成后端开发：

```text
DeepSeek API 本身能用
≠ Claude Desktop Provider Setup 表单校验能过
≠ Claude Code 工具调用/模型发现/权限模式全都兼容
```

---

# 6. 社区讨论里还有一个更深层问题：tool call 协议

Claude Code 不是普通聊天工具，它会大量使用工具调用：

```text
读文件
改文件
跑命令
搜索代码
调用 MCP
子任务 agent
```

社区 issue 里有人分析过 DeepSeek V4 + Claude Code 代理场景的工具调用问题：Claude Code 发的是 Anthropic 工具格式：

```json
{
  "name": "...",
  "description": "...",
  "input_schema": {}
}
```

而某些代理会把它转换成 OpenAI function calling 格式：

```json
{
  "type": "function",
  "function": {
    "name": "...",
    "description": "...",
    "parameters": {}
  }
}
```

如果转换过程中 `function.name` 丢失，DeepSeek 会直接返回 400。([GitHub](https://github.com/anomalyco/opencode/issues/24224 "[OpenCode Go] deepseek-v4-pro Anthropic proxy tool format broken: tools[0].function missing name · Issue #24224 · anomalyco/opencode · GitHub"))

这说明：**第三方模型接入 Claude Code，不只是“能聊天”就够了，还要能稳定处理工具调用。**

对 Claude Code 这种编程 agent 来说，工具调用兼容性比普通 chat completion 更重要。

---

# 7. 官方现在推荐的配置思路是什么

官方文档的方向其实很清楚：

## 方式一：用环境变量 pin alias

例如把 Claude 的 alias 映射到第三方模型：

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "你的 DeepSeek Key",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "deepseek-v4-pro[1m]",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "deepseek-v4-flash",
    "CLAUDE_CODE_SUBAGENT_MODEL": "deepseek-v4-flash"
  }
}
```

Claude Code 官方文档说这些环境变量必须是完整模型名，或者是 API provider 能识别的等价模型名。([Claude](https://code.claude.com/docs/en/model-config "Model configuration - Claude Code Docs"))

## 方式二：用 Gateway

Gateway 适合企业或复杂路由场景，可以统一鉴权、审计、限流、成本控制、模型路由。官方文档明确把 LLM gateway 定义为 Claude Code 与模型供应商之间的集中代理层。([Claude](https://code.claude.com/docs/en/llm-gateway "LLM gateway configuration - Claude Code Docs"))

你现在用的 ccswitch 就属于这个思路，只是更偏个人本地代理。

## 方式三：用 `modelOverrides`

Claude Code 新版支持 `modelOverrides`，可以把模型选择器里的 Anthropic 模型 ID 映射到 provider-specific model ID。官方文档说，`modelOverrides` 的 key 必须是 Anthropic model ID，value 是发送给 provider API 的真实模型 ID；未知 key 会被忽略。([Claude](https://code.claude.com/docs/en/model-config "Model configuration - Claude Code Docs"))

这和你文章里的“不要在 Model list 里填真实第三方模型名，而是填 Claude 自己的模型名”是同一类思路。

---

# 8. 你的问题根因，一句话版

你之前失败，是因为：

```text
你把 DeepSeek 当成“Claude Desktop 能直接识别的 Provider”来配了；
但新版 Claude Desktop 的 Third-Party Inference 更像是要求一个“Claude-compatible Gateway”。
```

文章里的修复成功，是因为：

```text
ccswitch 把 DeepSeek 包装成了 Claude Desktop 能识别的 Claude-compatible Gateway，
并把 Claude 模型路由名映射到了真实 DeepSeek 模型。
```

---

# 9. 建议你以后怎么判断类似问题

以后看到第三方模型接入 Claude Code / Claude Desktop，按这张表排查：

|检查项|你要确认什么|
|---|---|
|Base URL|是不是 Anthropic Messages API 格式，而不是 OpenAI `/v1/chat/completions`|
|模型名|Desktop 里是否需要填 Claude alias / Claude model ID，而不是真实模型名|
|映射|是否通过 env、`modelOverrides`、gateway 做了 alias → provider model 的映射|
|鉴权|用的是 `Authorization: Bearer` 还是 `x-api-key`|
|`/v1/models`|Gateway 是否实现，返回模型 ID 是否能被 Claude Code picker 接受|
|工具调用|是否支持 Claude Code 的 tool use 格式|
|长上下文|`[1m]` 后缀只在底层模型真的支持 1M 上下文时才应该加|
|Desktop vs CLI|CLI 能跑，不代表 Desktop Setup UI 能过|

---

# 10. 对你当前配置的保守建议

如果你现在已经能用 DeepSeek V4，建议暂时保持：

```text
Claude Desktop → ccswitch → DeepSeek Anthropic endpoint
```

不要频繁切回直连。原因是：

1. Desktop 的第三方推理配置还在快速变化；
    
2. Claude Code 近期 changelog 里 Windows、模型、认证、MCP、tool call 相关修复很多，说明这一块仍然在高频迭代；例如 2026 年 5 月 8 日的 2.1.136 就包含 MCP、登录、tool call、Windows/终端显示等大量修复。([Claude](https://code.claude.com/docs/en/changelog "Changelog - Claude Code Docs"))
    
3. 本地 gateway 至少能把模型名、鉴权、协议兼容问题集中到一层解决。
    

另外，最近还有假冒 Claude Windows 安装包传播恶意软件的新闻。Windows 上折腾 Claude Desktop / Claude Code 时，安装包务必只从 Anthropic 官方入口或可信包管理器获取。([IT Pro](https://www.itpro.com/security/cyber-attacks/claude-users-beware-hackers-are-using-a-fake-website-to-dupe-developers-and-deliver-malware?utm_source=chatgpt.com "Claude users beware, hackers are using a fake website to dupe developers and deliver malware"))

---

## 最后给你一个后端视角类比

你可以把它理解成：

```text
Claude Desktop = 前端 + SDK Client
Claude Code = Agent Runtime
Third-Party Inference = SPI 插件点
DeepSeek = 真实后端服务
ccswitch = Adapter / Gateway / Protocol Translator
```

你的错误不是“DeepSeek 不能用”，而是：

```text
SPI 插件点要求的接口契约变严格了；
DeepSeek 的真实接口虽然兼容 Anthropic Messages，
但没有完全满足 Claude Desktop 当前 Provider Setup 的模型路由契约；
所以需要 Adapter 层。
```

这就是你“稀里糊涂修好了”的真实技术原因。