
模型参数举例

1. **基础连接参数**：`model`, `base_url`, `api_key`（模型标识，基础 URL，API 密钥）
    
2. **AIGC 行为控制参数**：`temperature`, `max_tokens`;
    
    - **温度 (`temperature`)**：控制在 0 到 2 之间，影响生成文本的随机性。值越低（如 0.1），输出越确定；值越高，输出越具有创造性和随机性。我一般都是根据官网的建议来调整。比如 Deepseek-v3.2 做 Agent 开发，最好设置 1.0 到 1.3。
    - **最大令牌数 (`max_tokens`)**：限制模型单次响应所能生成的最大令牌 (Token) 数量，有助于控制响应长度和成本
3. **请求可靠性控制参数**：超时时间 (`timeout`)，最大重试次数 (`max_retries`)：当 API 请求因网络等问题失败时，自动重试的最大次数，提高程序的容错能力。
    
4. **流式使用量统计 (`stream_usage`)**：当设置为 True 时，即使在流式传输响应的情况下，也能实时获取 Token 消耗统计信息


# LangChain / LangGraph / DeepAgents 参数速查表

> 本速查表基于以下版本源码整理（所有参数均经过源码核验）：
> - `langchain` **1.3.11** / `langchain-core` **1.4.8** / `langchain-openai` **1.3.3**
> - `langgraph` **1.2.8** / `langgraph-prebuilt` **1.1.0**
> - `deepagents` **0.6.12**
>
> 适用 Python **3.10+**。

---

## 目录

- [一、模型创建参数](#一模型创建参数)
  - [1.1 init_chat_model 统一工厂（推荐）](#11-init_chat_model-统一工厂推荐)
  - [1.2 ChatOpenAI 参数表](#12-chatopenai-参数表)
  - [1.3 with_structured_output 结构化输出参数](#13-with_structured_output-结构化输出参数)
- [二、create_agent 参数](#二create_agent-参数)
- [三、create_deep_agent 参数](#三create_deep_agent-参数)
- [四、关键约束与坑点速记](#四关键约束与坑点速记)

---

## 一、模型创建参数

### 1.1 init_chat_model 统一工厂（推荐）

**来源**：`langchain.chat_models.init_chat_model`（langchain 1.3.11）

**为什么推荐**：通过 `"provider:model"` 字符串格式透明切换 provider，无需为每个 provider 导入不同的类，便于配置化与多 provider 容灾。是企业级应用创建模型的标准入口。

#### 函数签名

```python
def init_chat_model(
    model: str | None = None,
    *,
    model_provider: str | None = None,
    configurable_fields: Literal["any"] | list[str] | tuple[str, ...] | None = None,
    config_prefix: str | None = None,
    **kwargs: Any,
) -> BaseChatModel | _ConfigurableModel: ...
```

#### 参数表

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `model` | `str \| None` | `None` | 模型名，**推荐** `"provider:model"` 格式（如 `"openai:gpt-5"`）；bare 名会尝试前缀推断 |
| `model_provider` | `str \| None` | `None` | provider 名（与 `model` 前缀二选一）；适合 provider 动态读取场景 |
| `configurable_fields` | `Literal["any"] \| list[str] \| tuple[str, ...] \| None` | `None`（`model=None` 时默认 `("model", "model_provider")`） | 运行时可配置字段；**`"any"` 有安全风险** |
| `config_prefix` | `str \| None` | `None` | 配置前缀（多模型共存时隔离 `config["configurable"]` 命名空间） |
| `**kwargs` | `Any` | — | 透传给具体 provider 的 ChatModel（如 `temperature` / `max_tokens` / `timeout` / `max_retries` / `base_url` / `api_key` / `rate_limiter` / `streaming`） |

#### 支持的 provider（26 个）

`model` 加 `"provider:"` 前缀或通过 `model_provider` 显式指定均可。前缀推断按大小写不敏感匹配。

| `model_provider` | 集成包 | `model` 前缀自动推断 |
|---|---|---|
| `openai` | `langchain-openai` | `gpt-...` / `o1...` / `o3...` / `chatgpt...` / `text-davinci...` |
| `anthropic` | `langchain-anthropic` | `claude...` |
| `azure_openai` | `langchain-openai` | — |
| `azure_ai` | `langchain-azure-ai` | — |
| `google_vertexai` | `langchain-google-vertexai` | `gemini...`（下个大版本默认将切换） |
| `google_genai` | `langchain-google-genai` | — |
| `google_anthropic_vertex` | `langchain-google-vertexai` | — |
| `bedrock` / `anthropic_bedrock` / `bedrock_converse` | `langchain-aws` | `amazon....` / `anthropic....` / `meta....` |
| `cohere` | `langchain-cohere` | `command...` |
| `fireworks` | `langchain-fireworks` | `accounts/fireworks...` |
| `together` | `langchain-together` | — |
| `mistralai` | `langchain-mistralai` | `mistral...` / `mixtral...` |
| `huggingface` | `langchain-huggingface` | — |
| `groq` | `langchain-groq` | — |
| `ollama` | `langchain-ollama` | — |
| `deepseek` | `langchain-deepseek` | `deepseek...` |
| `ibm` | `langchain-ibm` | — |
| `nvidia` | `langchain-nvidia-ai-endpoints` | — |
| `xai` | `langchain-xai` | `grok...` |
| `openrouter` | `langchain-openrouter` | — |
| `perplexity` | `langchain-perplexity` | `sonar...` |
| `upstage` | `langchain-upstage` | `solar...` |
| `baseten` | `langchain-baseten` | — |
| `litellm` | `langchain-litellm` | — |

> 💡 **provider 选择策略**：优先使用 provider 专属集成包（如 `deepseek` / `anthropic`）；若 provider 没有 LangChain 集成包但提供 OpenAI 兼容接口，用 `openai` provider + `base_url` 切换（见模板 3）。

#### 代码模板

**模板 1：固定模型（最常见）**

```python
from langchain.chat_models import init_chat_model

# 推荐 "provider:model" 前缀格式（前缀推断虽支持但不可靠）
llm = init_chat_model(
    "openai:gpt-4o-mini",
    temperature=0,
    max_retries=10,   # 默认 6 偏小，生产建议 10-15
    timeout=120,      # 默认无超时，慢请求会卡死
)
llm.invoke("hello")
```

**模板 2：切换 provider（有专属集成包）**

```python
import os
from langchain.chat_models import init_chat_model

# DeepSeek（需 pip install langchain-deepseek）
llm = init_chat_model(
    "deepseek:deepseek-v4-flash",
    temperature=0.6,
    api_key=os.getenv("DEEPSEEK_API_KEY"),
    max_retries=10,
    timeout=120,
)

# Anthropic Claude（需 pip install langchain-anthropic）
llm = init_chat_model("anthropic:claude-sonnet-4-5", temperature=0, max_retries=10)

# 本地 Ollama（需 pip install langchain-ollama）
llm = init_chat_model("ollama:llama3.2", temperature=0)
```

**模板 3：OpenAI 兼容接口（无专属集成包时）**

provider 没有 LangChain 集成包但提供 OpenAI 兼容接口时，用 `openai` provider + `base_url` 切换。

```python
import os
from langchain.chat_models import init_chat_model

# 智谱 GLM（base_url 末尾不带 /，否则 embedding 等接口会 404）
llm = init_chat_model(
    "openai:glm-4.6",
    temperature=0.6,
    api_key=os.getenv("ZHIPU_API_KEY"),
    base_url="https://open.bigmodel.cn/api/paas",
    max_retries=10,
    timeout=120,
)

# 通义千问 DashScope（OpenAI 兼容模式）
llm = init_chat_model(
    "openai:qwen-plus",
    api_key=os.getenv("DASHSCOPE_API_KEY"),
    base_url="https://dashscope.aliyuncs.com/compatible-mode/v1",
)
```

**模板 4：运行时可配置（configurable_fields）**

允许通过 `RunnableConfig` 在运行时切换模型/provider，无需重建实例。适用于 A/B 测试、灰度发布、按用户路由。

```python
from langchain.chat_models import init_chat_model

# 不传 model，model 和 model_provider 默认可配置
llm = init_chat_model(configurable_fields=("model", "model_provider", "temperature"))

# 运行时通过 config 切换
config = {
    "configurable": {
        "model": "anthropic:claude-sonnet-4-5",
        "temperature": 0.2,
    }
}
llm.invoke("hello", config=config)
```

**模板 5：多模型共存（config_prefix）**

同一应用中配置多个可切换模型，通过前缀隔离 `config["configurable"]` 命名空间。

```python
from langchain.chat_models import init_chat_model

primary = init_chat_model(
    "openai:gpt-4o",
    configurable_fields=("model", "temperature"),
    config_prefix="primary",
)
fallback = init_chat_model(
    "anthropic:claude-sonnet-4-5",
    configurable_fields=("model", "temperature"),
    config_prefix="fallback",
)

# 运行时通过前缀独立配置
config = {
    "configurable": {
        "primary_model": "openai:gpt-4o-mini",   # primary 降级
        "fallback_temperature": 0,                # fallback 严格
    }
}
```

**模板 6：结构化输出**

```python
from pydantic import BaseModel, Field
from langchain.chat_models import init_chat_model

class UserIntent(BaseModel):
    intent: str = Field(description="意图标签")
    confidence: float = Field(description="置信度 0-1")

# 结构化输出场景 temperature 必须 = 0
llm = init_chat_model("openai:gpt-4o-mini", temperature=0)
structured = llm.with_structured_output(
    UserIntent,
    method="json_schema",  # OpenAI 原生；非 OpenAI provider 改用 "function_calling"
    strict=True,
)
result = structured.invoke("我想订一张明天去北京的机票")
# result: UserIntent(intent="book_flight", confidence=0.95)
```

**模板 7：主备容灾（with_fallbacks）**

```python
from langchain.chat_models import init_chat_model

primary = init_chat_model("openai:gpt-4o", max_retries=3, timeout=60)
fallback = init_chat_model("anthropic:claude-sonnet-4-5", max_retries=5, timeout=120)
model = primary.with_fallbacks([fallback])
# primary 异常时自动切换到 fallback，不会主动负载均衡
```

#### 安全注意

> ⚠️ **`configurable_fields="any"` 有安全风险**：允许运行时通过 `config` 修改 `api_key` / `base_url` 等字段，可能被恶意重定向到其他服务。**接受不可信配置时必须显式枚举 `configurable_fields=("model", "temperature", ...)`**。

#### 企业级最佳实践

- API Key 永远走环境变量或密钥管理服务，**禁止硬编码**
- `max_retries` 与 `timeout` 必须显式设置（默认值偏小）
- 流式优先（`streaming=True`）以降低首字延迟
- 优先使用 `"provider:model"` 前缀格式，而非 bare model 名（避免推断失败）
- 优先使用 pinned model ID（如 `claude-haiku-4-5-20251001`），避免使用 moving alias（如 `claude-haiku-4-5`），防止上游重定向导致行为漂移
- 多 provider 容灾用 `with_fallbacks`，但注意它**仅在异常时触发**，不负载均衡

---

### 1.2 ChatOpenAI 参数表

**来源**：`langchain_openai.chat_models.base.BaseChatOpenAI`（langchain-openai 1.3.3）

> 💡 **使用建议**：新代码推荐使用 [1.1 init_chat_model](#11-init_chat_model-统一工厂推荐)，仅在需要 OpenAI 专属参数（如 `reasoning_effort` / `use_responses_api` / `verbosity`）或直接操作 `ChatOpenAI` 实例时使用本类。下表仅作参数速查，**不再提供代码模板**。

#### 1.2.1 模型本体参数

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `model` / `model_name` | `str` | `"gpt-3.5-turbo"` | 模型名（`model` 为别名） |
| `temperature` | `float \| None` | `None` | 采样温度，结构化输出场景固定为 0 |
| `max_tokens` | `int \| None` | `None` | 单次生成最大 token 数；`None` 由模型决定 |
| `top_p` | `float \| None` | `None` | nucleus sampling，与 `temperature` 二选一 |
| `n` | `int \| None` | `None` | 每次提示生成的候选数 |
| `seed` | `int \| None` | `None` | 生成种子（可复现） |
| `stop` | `list[str] \| str \| None` | `None` | 停止序列（别名 `stop_sequences`） |
| `presence_penalty` | `float \| None` | `None` | 主题 novelty 惩罚（-2.0 ~ 2.0） |
| `frequency_penalty` | `float \| None` | `None` | 频率惩罚（-2.0 ~ 2.0） |
| `logprobs` | `bool \| None` | `None` | 是否返回 logprobs |
| `top_logprobs` | `int \| None` | `None` | 每位置返回 top-N token（0-20） |
| `logit_bias` | `dict[int, int] \| None` | `None` | token ID 偏置（-100 ~ 100） |

#### 1.2.2 流式与推理控制（OpenAI 专属）

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `streaming` | `bool` | `False` | 是否流式输出 |
| `stream_usage` | `bool \| None` | `None` | 流式输出中是否包含 usage（v0.3.9+） |
| `disable_streaming` | `bool \| "tool_calling"` | `False` | 禁用流式；`"tool_calling"` 仅在工具调用时禁用 |
| `stream_chunk_timeout` | `float \| None` | `120.0` | 异步流式 chunk 超时（秒） |
| `reasoning_effort` | `str \| None` | `None` | 推理强度（Chat Completions API）：`minimal/low/medium/high` |
| `reasoning` | `dict \| None` | `None` | 推理参数（Responses API，v0.3.24+）：`{"effort":..., "summary":"auto/concise/detailed"}` |
| `verbosity` | `str \| None` | `None` | 响应详细度（Responses API，v0.3.28+）：`low/medium/high` |
| `use_responses_api` | `bool \| None` | `None` | 是否使用 Responses API（v0.3.9+） |

#### 1.2.3 客户端与网络

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `api_key` / `openai_api_key` | `SecretStr \| Callable` | env | API Key，支持 sync/async callable（v0.3+） |
| `base_url` / `openai_api_base` | `str \| None` | env | API 基地址（代理/网关） |
| `organization` | `str \| None` | env | OpenAI 组织 ID |
| `openai_proxy` | `str \| None` | env | HTTP 代理 |
| `timeout` / `request_timeout` | `float \| tuple \| Any` | `None` | 请求超时（**生产建议 ≥ 120s**） |
| `max_retries` | `int \| None` | `None`(实际默认 6) | 最大重试次数（**生产建议 10-15**） |
| `default_headers` | `Mapping[str, str] \| None` | `None` | 默认请求头 |
| `default_query` | `Mapping[str, object] \| None` | `None` | 默认查询参数 |
| `http_client` | `Any \| None` | `None` | 自定义 `httpx.Client`（同步） |
| `http_async_client` | `Any \| None` | `None` | 自定义 `httpx.AsyncClient`（异步） |
| `http_socket_options` | `Sequence[tuple] \| None` | `None` | TCP socket 选项（keepalive 等） |

#### 1.2.4 其他

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `model_kwargs` | `dict[str, Any]` | `{}` | 透传给底层 API 的额外参数 |
| `tiktoken_model_name` | `str \| None` | `None` | tiktoken 计费模型名（兼容非标准模型） |
| `rate_limiter` | `BaseRateLimiter \| None` | `None` | 速率限制器 |
| `cache` | `BaseCache \| None` | `None` | 实例级缓存 |
| `verbose` | `bool` | `False` | 详细日志 |

---

### 1.3 with_structured_output 结构化输出参数

**来源**：`BaseChatModel.with_structured_output`（OpenAI 系重载默认 `method="json_schema"`）

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `schema` | `dict[str, Any] \| type` | 必填 | Pydantic 类或 JSON Schema dict |
| `method` | `Literal["function_calling", "json_mode", "json_schema"]` | `"json_schema"`（OpenAI 重载） | 输出策略 |
| `include_raw` | `bool` | `False` | 是否返回原始响应（调试用） |
| `strict` | `bool \| None` | `None` | 严格模式（schema 完全匹配） |
| `tools` | `list \| None` | `None` | 工具绑定（高级用法） |

> ⚠️ **provider 兼容性**：`method="json_schema"` 是 OpenAI 原生支持；非 OpenAI provider 应改用 `method="function_calling"`（兼容性最广）或 `method="json_mode"`（无 schema 约束）。

---

## 二、create_agent 参数

**来源**：`langchain.agents.factory.create_agent`（langchain 1.3.11，**推荐用法**）

> ⚠️ `langgraph.prebuilt.create_react_agent` 已废弃，新代码请使用本函数。

### 函数签名

```python
def create_agent(
    model: str | BaseChatModel,
    tools: Sequence[BaseTool | Callable[..., Any] | dict[str, Any]] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware[StateT_co, ContextT]] = (),
    response_format: ResponseFormat[ResponseT] | type[ResponseT] | dict[str, Any] | None = None,
    state_schema: type[AgentState[ResponseT]] | None = None,
    context_schema: type[ContextT] | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    interrupt_before: list[str] | None = None,
    interrupt_after: list[str] | None = None,
    debug: bool = False,
    name: str | None = None,
    cache: BaseCache[Any] | None = None,
    transformers: Sequence[TransformerFactory] | None = None,
) -> CompiledStateGraph: ...
```

### 参数详解

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `model` | `str \| BaseChatModel` | 必填 | 模型实例或 `"openai:gpt-5"` 字符串 |
| `tools` | `Sequence[BaseTool \| Callable \| dict] \| None` | `None` | 工具列表（推荐 `@tool` 装饰器） |
| `system_prompt` | `str \| SystemMessage \| None` | `None` | 系统提示词（v1.0 改名自 `prompt`，仍支持 `SystemMessage`） |
| `middleware` | `Sequence[AgentMiddleware]` | `()` | 中间件链（替代废弃的 `pre_model_hook` / `post_model_hook`） |
| `response_format` | `ResponseFormat \| type \| dict \| None` | `None` | 结构化输出 schema |
| `state_schema` | `type[AgentState] \| None` | `None` | 状态 schema（**仅 TypedDict**，不再支持 Pydantic/dataclass） |
| `context_schema` | `type \| None` | `None` | 运行时不可变上下文 schema |
| `checkpointer` | `Checkpointer \| None` | `None` | 持久化 checkpoint（生产必填） |
| `store` | `BaseStore \| None` | `None` | 跨 thread 长期存储 |
| `interrupt_before` | `list[str] \| None` | `None` | 节点前中断 |
| `interrupt_after` | `list[str] \| None` | `None` | 节点后中断 |
| `debug` | `bool` | `False` | 调试模式 |
| `name` | `str \| None` | `None` | 图名称 |
| `cache` | `BaseCache \| None` | `None` | 缓存 |
| `transformers` | `Sequence[TransformerFactory] \| None` | `None` | 流式输出 transformer |

### AgentMiddleware 钩子方法

**来源**：`langchain.agents.middleware.types.AgentMiddleware`

中间件通过实现以下钩子方法介入代理执行流程。

| 钩子方法 | 同步 / 异步 | 触发时机 | 典型用途 |
|---|---|---|---|
| `before_agent` / `abefore_agent` | ✓ | 代理执行开始前 | 初始化状态、注入上下文 |
| `before_model` / `abefore_model` | ✓ | 每次模型调用前 | 消息裁剪、PII 脱敏、动态提示 |
| `after_model` / `aafter_model` | ✓ | 每次模型调用后 | 输出守卫、审批、遥测 |
| `after_agent` / `aafter_agent` | ✓ | 代理执行完成后 | 终态审计、资源清理、结果落库 |
| `wrap_model_call` / `awrap_model_call` | ✓ | 包裹整个模型调用 | 重试、缓存、降级、可观测性 |
| `wrap_tool_call` / `awrap_tool_call` | ✓ | 包裹工具调用 | 工具错误处理、审计 |

**中间件类属性：**

| 属性 | 类型 | 用途 |
|---|---|---|
| `state_schema` | `type[StateT]` | 中间件贡献的状态字段 |
| `tools` | `Sequence[BaseTool]` | 中间件注入的工具 |
| `transformers` | `Sequence[TransformerFactory]` | 流式输出 transformer |
| `name` | `str`（property） | 默认为类名 |

**`wrap_model_call` 签名：**

```python
def wrap_model_call(
    self,
    request: ModelRequest[ContextT],          # 不可变，修改用 request.override(**changes)
    handler: Callable[[ModelRequest], ModelResponse[ResponseT]],
) -> ModelResponse[ResponseT] | AIMessage | ExtendedModelResponse[ResponseT]:
    # handler 可多次调用（重试）、可跳过（短路）、可包装响应
    ...
```

**`wrap_tool_call` 签名：**

```python
def wrap_tool_call(
    self,
    request: ToolCallRequest,                 # 字段: tool_call(dict), tool, state, runtime
    handler: Callable[[ToolCallRequest], ToolMessage | Command[Any]],
) -> ToolMessage | Command[Any]:
    ...
```

> ⚠️ **`ToolCallRequest` 字段访问**：工具名与参数应通过 `request.tool_call["name"]` 与 `request.tool_call["args"]` 访问，**没有** `request.name` / `request.args` 字段。

---

## 三、create_deep_agent 参数

**来源**：`deepagents.graph.create_deep_agent`（deepagents 0.6.12）

### 函数签名

```python
def create_deep_agent(
    model: str | BaseChatModel | None = None,         # 1.0 起必填
    tools: Sequence[BaseTool | Callable | dict] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware] = (),
    subagents: Sequence[SubAgent | CompiledSubAgent | AsyncSubAgent] | None = None,
    skills: list[str] | None = None,
    memory: list[str] | None = None,
    permissions: list[FilesystemPermission] | None = None,
    backend: BackendProtocol | BackendFactory | None = None,
    interrupt_on: dict[str, bool | InterruptOnConfig] | None = None,
    response_format: ResponseFormat[ResponseT] | type[ResponseT] | dict | None = None,
    state_schema: type[DeepAgentState] | None = None,
    context_schema: type[ContextT] | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    debug: bool = False,
    name: str | None = None,
    cache: BaseCache | None = None,
) -> CompiledStateGraph: ...
```

### 参数详解

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `model` | `str \| BaseChatModel \| None` | `None`(已废弃) | 模型或 `"provider:model"`；**1.0 起必填** |
| `tools` | `Sequence \| None` | `None` | **追加**到内置工具集（write_todos/fs/execute/task） |
| `system_prompt` | `str \| SystemMessage \| None` | `None` | 自定义系统提示，置于 SDK 默认提示前 |
| `middleware` | `Sequence[AgentMiddleware]` | `()` | 用户中间件（插入到 base 与 tail 栈之间） |
| `subagents` | `Sequence[SubAgent \| CompiledSubAgent \| AsyncSubAgent] \| None` | `None` | 子代理 |
| `skills` | `list[str] \| None` | `None` | 技能目录路径（POSIX 风格） |
| `memory` | `list[str] \| None` | `None` | `AGENTS.md` 文件路径列表 |
| `permissions` | `list[FilesystemPermission] \| None` | `None` | 文件系统权限规则（`allow/deny/interrupt`） |
| `backend` | `BackendProtocol \| BackendFactory \| None` | `StateBackend()` | 文件/执行后端 |
| `interrupt_on` | `dict[str, bool \| InterruptOnConfig] \| None` | `None` | 工具级 HITL 审批 |
| `response_format` | `ResponseFormat \| type \| dict \| None` | `None` | 结构化输出 schema |
| `state_schema` | `type[DeepAgentState] \| None` | `None` | 自定义状态（**必须继承 `DeepAgentState`**） |
| `context_schema` | `type \| None` | `None` | 运行时上下文 schema |
| `checkpointer` | `Checkpointer \| None` | `None` | 持久化（生产必填） |
| `store` | `BaseStore \| None` | `None` | 长期存储 |
| `debug` | `bool` | `False` | 调试模式 |
| `name` | `str \| None` | `None` | 代理名称 |
| `cache` | `BaseCache \| None` | `None` | 缓存 |

### DeepAgents 默认中间件栈（顺序）

```
Base stack（SDK 内置，不可排除）:
  1. TodoListMiddleware                      任务规划
  2. SkillsMiddleware                        若 skills 提供
  3. FilesystemMiddleware                    虚拟 FS + 权限（不可排除）
  4. SubAgentMiddleware                      若有同步子代理（不可排除）
  5. SummarizationMiddleware                 上下文压缩
  6. PatchToolCallsMiddleware                悬空工具调用修复
  7. AsyncSubAgentMiddleware                 若有异步子代理

[用户 middleware 插入位置]

Tail stack:
  - Harness profile extra_middleware
  - _ToolExclusionMiddleware                 若有 excluded_tools
  - AnthropicPromptCachingMiddleware         无操作对非 Anthropic 模型
  - BedrockPromptCachingMiddleware           若安装 langchain-aws
  - MemoryMiddleware                         若 memory 提供
  - HumanInTheLoopMiddleware                 若 interrupt_on 提供
```

### FilesystemPermission 字段

**来源**：`deepagents.middleware.filesystem.FilesystemPermission`（`@dataclass`）

| 字段 | 类型 | 用途 |
|---|---|---|
| `operations` | `list[FilesystemOperation]` | 操作类型，`"read"` 和/或 `"write"`（`FilesystemOperation = Literal["read", "write"]`） |
| `paths` | `list[str]` | 路径 glob 列表（必须以 `/` 开头） |
| `mode` | `"allow" \| "deny" \| "interrupt"` | 匹配时的处置（默认 `"allow"`） |

**工具到操作的映射**（由 FilesystemMiddleware 自动应用）：
- `read` → `ls`, `read_file`, `glob`, `grep`
- `write` → `write_file`, `edit_file`

**规则评估**：按声明顺序评估，第一个匹配生效；无规则匹配时默认 `allow`。

### 后端类型对照

| kind | Backend | 适用场景 |
|---|---|---|
| `state` | `StateBackend` | 默认；通过 `invoke(files=...)` 提供 |
| `fs` | `FilesystemBackend` | 直接读写磁盘（需权限严格控制） |
| `shell` | `LocalShellBackend` | 执行 shell 命令（**高危**！） |
| composite | 组合 | 多后端组合 |

> ⚠️ **`LocalShellBackend` 参数**：继承自 `FilesystemBackend`，构造参数是 `root_dir`（**无 `workdir` 参数**），额外支持 `timeout` / `max_output_bytes` / `env` / `inherit_env`。

---

## 四、关键约束与坑点速记

### 4.1 版本约束

| 约束 | 说明 |
|---|---|
| `state_schema` 仅 TypedDict | LangChain v1.0 起，**不再支持** Pydantic BaseModel / dataclass |
| `create_react_agent` 已废弃 | 改用 `langchain.agents.create_agent` |
| `pre_model_hook` / `post_model_hook` 已废弃 | 迁移到 middleware 的 `before_model` / `after_model` / `wrap_model_call` |
| `prompt` 参数已改名 | v1.0 改名为 `system_prompt`，类型 `str \| SystemMessage \| None` |
| `model=None` 默认值已废弃 | deepagents 0.5.3 起，1.0 将强制要求显式指定模型 |

### 4.2 必填参数陷阱

| API | 参数陷阱 | 正确用法 |
|---|---|---|
| `HumanInTheLoopMiddleware` | `interrupt_on` 必填 | `HumanInTheLoopMiddleware(interrupt_on={"send_email": True})` |
| `SummarizationMiddleware` | `model` 必填 | `SummarizationMiddleware(model=model)` |
| `create_deep_agent` 的 `state_schema` | 必须继承 `DeepAgentState` | `class MyState(DeepAgentState, total=False): ...` |

### 4.3 PostgresSaver 用法

**`from_conn_string` 是 `@contextmanager`**，返回 `Iterator[PostgresSaver]`，**不能直接作为 checkpointer 返回**。

```python
# ❌ 错误：工厂模式下不能用 context manager
def build_checkpointer():
    return PostgresSaver.from_conn_string(dsn)  # 返回的是 generator

# ✅ 正确：直接构造并显式 setup
import psycopg
from psycopg.rows import dict_row
from langgraph.checkpoint.postgres import PostgresSaver

conn = psycopg.connect(dsn, autocommit=True, prepare_threshold=0, row_factory=dict_row)
checkpointer = PostgresSaver(conn)
checkpointer.setup()  # 首次必须调用，创建表与迁移
```

**异步版本导入路径**：
```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
```

### 4.4 生产环境推荐值

| 参数 | 推荐值 | 原因 |
|---|---|---|
| `max_retries` | `10-15` | 默认 6 偏小，生产需要更多容错 |
| `timeout` | `≥ 120` 秒 | 慢网络与复杂推理需要余量 |
| `temperature` | `0`（结构化输出） | 高温会导致校验失败重试增加 35x |
| `recursion_limit` | `25`（默认）或按业务上限调 | 控制 LangGraph 图执行深度，防止死循环 |
| `streaming` | `True` | 降低首字延迟，改善用户体验 |

### 4.5 调用规范

每次调用代理必须传入 `thread_id` 与 `context`：

```python
config = {
    "configurable": {"thread_id": str(uuid.uuid4())},  # ≤ 255 字符
    "context": EnterpriseContext(...),
}
result = await agent.ainvoke({"messages": [...]}, config=config)
```

中断恢复用 `Command(resume=...)`：
```python
async for event in agent.astream_events(Command(resume="approved"), config=config, version="v3"):
    ...
```

---

**参考源码文件**（相对路径）：
- `langchain_openai/chat_models/base.py` — ChatOpenAI 实现
- `langchain/agents/factory.py` — create_agent 实现
- `langchain/agents/middleware/types.py` — AgentMiddleware / ModelRequest / ModelResponse
- `langgraph/prebuilt/tool_node.py` — ToolCallRequest
- `langgraph/checkpoint/postgres/__init__.py` — PostgresSaver
- `deepagents/graph.py` — create_deep_agent 实现
- `deepagents/middleware/filesystem.py` — FilesystemPermission
- `deepagents/backends/local_shell.py` — LocalShellBackend
- `langchain/agents/middleware/human_in_the_loop.py` — HumanInTheLoopMiddleware
- `langchain/agents/middleware/summarization.py` — SummarizationMiddleware
