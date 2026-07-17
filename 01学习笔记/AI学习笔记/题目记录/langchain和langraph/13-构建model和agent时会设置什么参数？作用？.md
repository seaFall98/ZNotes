
模型参数举例

1. **基础连接参数**：`model`, `base_url`, `api_key`（模型标识，基础 URL，API 密钥）
    
2. **AIGC 行为控制参数**：`temperature`, `max_tokens`;
    
    - **温度 (`temperature`)**：控制在 0 到 2 之间，影响生成文本的随机性。值越低（如 0.1），输出越确定；值越高，输出越具有创造性和随机性。我一般都是根据官网的建议来调整。比如 Deepseek-v3.2 做 Agent 开发，最好设置 1.0 到 1.3。
    - **最大令牌数 (`max_tokens`)**：限制模型单次响应所能生成的最大令牌 (Token) 数量，有助于控制响应长度和成本
3. **请求可靠性控制参数**：超时时间 (`timeout`)，最大重试次数 (`max_retries`)：当 API 请求因网络等问题失败时，自动重试的最大次数，提高程序的容错能力。
    
4. **流式使用量统计 (`stream_usage`)**：当设置为 True 时，即使在流式传输响应的情况下，也能实时获取 Token 消耗统计信息

# LangChain / LangGraph 中 Model 与 Agent 构建参数详解

基于官方文档（reference.langchain.com、docs.langchain.com）整理，涵盖 `init_chat_model`（模型层）、`create_agent`（新版 LangChain 智能体工厂，取代旧的 `create_react_agent`）、以及 `deepagents.create_deep_agent`。

---

## 一、模型构建：`init_chat_model`

```python
from langchain.chat_models import init_chat_model
```

|参数|类型|说明|
|---|---|---|
|`model`|`str`|模型名，可用 `"provider:model"` 格式一次指定提供商+模型，如 `"anthropic:claude-sonnet-4-6"`、`"openai:gpt-5.5"`|
|`model_provider`|`str \| None`|若未在 `model` 中用 `provider:` 前缀写明，则用此参数指定（如 `"openai"`、`"anthropic"`、`"google_genai"`），也可根据 model 名前缀自动推断|
|`configurable_fields`|`Literal["any"] \| Sequence[str] \| None`|哪些字段允许在运行时通过 `config["configurable"]` 动态覆盖。`"any"` 表示全部字段可配置（**安全提示**：若接受不可信配置，应显式枚举字段，而非用 `"any"`，否则 `api_key`、`base_url` 等可能被篡改）|
|`config_prefix`|`str \| None`|当 `configurable_fields` 启用时，运行时配置键的前缀，如 `config["configurable"]["{prefix}_temperature"]`|
|`**kwargs`|任意|透传给底层模型构造函数的参数，最常用的有：<br>• `temperature`：随机性/创造性，0 最确定<br>• `max_tokens`：最大输出 token 数<br>• `timeout`：请求超时秒数<br>• `max_retries`：失败自动重试次数（默认约 2，网络不稳定建议 6–15）<br>• `base_url`：自定义 API 端点（本地模型/代理常用）<br>• `api_key`：密钥<br>• `rate_limiter`：`BaseRateLimiter` 实例，控制请求速率<br>其余参数因 provider 而异，需查该 provider 集成文档（如 `langchain_openai.ChatOpenAI` 还支持 `model_kwargs`、`extra_body` 等）|

```python
model = init_chat_model(
    "anthropic:claude-sonnet-4-6",
    temperature=0.3,
    max_tokens=2000,
    timeout=30,
    max_retries=6,
)
```

---

## 二、智能体构建：`create_agent`（新版，取代已弃用的 `langgraph.prebuilt.create_react_agent`）

> ⚠️ `langgraph.prebuilt.create_react_agent` 已弃用，官方建议改用 `from langchain.agents import create_agent`。

```python
from langchain.agents import create_agent
```

|参数|类型|默认值|作用|
|---|---|---|---|
|`model`|`str \| BaseChatModel`|必填|语言模型，字符串标识符或已初始化模型实例；也支持"动态模型"（一个 `(state, runtime) -> BaseChatModel` 的可调用对象，实现按上下文切换模型）|
|`tools`|`Sequence[BaseTool \| Callable \| dict] \| None`|`None`|工具列表；为空则退化为无工具调用循环的纯 LLM 节点|
|`system_prompt`|`str \| SystemMessage \| None`|`None`|静态系统提示词，会加到消息列表开头；需要动态提示词则用 middleware|
|`middleware`|`Sequence[AgentMiddleware]`|`()`|中间件序列，用于在模型调用前后/工具调用前后插入逻辑（人工审核、重试、限流、日志、防护栏等），是 `create_agent` 最核心的扩展机制|
|`response_format`|`ResponseFormat \| type \| dict \| None`|`None`|结构化输出配置，可传 Pydantic 类/TypedDict/JSON Schema，或 `ToolStrategy`/`ProviderStrategy`|
|`state_schema`|`type[AgentState] \| None`|`None`|自定义图状态 schema，用于叠加自定义状态字段|
|`context_schema`|`type \| None`|`None`|运行时上下文（`Runtime.context`）的 schema，用于向动态模型/中间件传递运行期配置（如 user_id）|
|`checkpointer`|`Checkpointer \| None`|`None`|检查点保存器，实现单线程（会话）级别的状态持久化，支撑多轮对话记忆|
|`store`|`BaseStore \| None`|`None`|跨线程/跨会话持久化存储（长期记忆）|
|`interrupt_before`|`list[str] \| None`|`None`|在指定节点前中断，用于人工确认后再执行|
|`interrupt_after`|`list[str] \| None`|`None`|在指定节点后中断|
|`debug`|`bool`|`False`|是否开启调试模式|
|`name`|`str \| None`|`None`|编译后图的名称，作为子图节点加入多智能体系统时会用到|
|`cache`|`BaseCache \| None`|`None`|缓存后端，可对模型调用结果做缓存|
|`transformers`|`Sequence[TransformerFactory] \| None`|`None`|流事件转换器工厂，用于自定义 `.stream()` 输出的事件形态|

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain_core.tools import tool
from langgraph.checkpoint.memory import MemorySaver
from pydantic import BaseModel

@tool
def get_weather(city: str) -> str:
    """Get weather for a given city."""
    return f"It's always sunny in {city}!"

class WeatherResponse(BaseModel):
    conditions: str

model = init_chat_model("anthropic:claude-sonnet-4-6", temperature=0)

agent = create_agent(
    model=model,
    tools=[get_weather],
    system_prompt="You are a helpful weather assistant.",
    response_format=WeatherResponse,   # 需要结构化输出时可选
    checkpointer=MemorySaver(),        # 支持多轮对话记忆
    name="weather_agent",
)

config = {"configurable": {"thread_id": "1"}}
result = agent.invoke(
    {"messages": [{"role": "user", "content": "北京天气怎么样？"}]},
    config=config,
)
print(result["messages"][-1].content)
```

---

## 三、Deep Agent 构建：`deepagents.create_deep_agent`

`deepagents` 是在 `create_agent` 之上封装的"重装харness"，内置文件系统、子智能体、技能、记忆、摘要压缩、提示缓存等能力，专为长程复杂任务（研究、编码等）设计。

```python
from deepagents import create_deep_agent
```

|参数|类型|默认值|作用|
|---|---|---|---|
|`model`|`str \| BaseChatModel \| None`|`None`（已弃用，即将强制必填）|同 `create_agent`；不显式传入将默认用 `claude-sonnet-4-6`，官方建议显式指定|
|`tools`|`Sequence[BaseTool \| Callable \| dict] \| None`|`None`|自定义工具、LangChain 工具或 MCP 工具，与内置文件系统/任务工具并存|
|`system_prompt`|`str \| SystemMessage \| SystemPromptConfig \| None`|`None`|系统提示词，也可传结构化 `SystemPromptConfig`|
|`middleware`|`Sequence[AgentMiddleware]`|`()`|额外自定义中间件，会叠加在 deepagents 默认中间件栈之后|
|`subagents`|`Sequence[SubAgent \| CompiledSubAgent \| AsyncSubAgent] \| None`|`None`|定义可被 `task` 工具调用的子智能体（专用/隔离上下文的子任务执行者），也可传入预编译的 LangGraph 子图或远程异步子智能体|
|`skills`|`list[str] \| None`|`None`|技能目录路径列表，每个技能是含 `SKILL.md` 的目录，按需渐进加载领域知识/工作流|
|`memory`|`list[str] \| None`|`None`|`AGENTS.md` 记忆文件路径列表，启动时始终加载，存放长期偏好/项目约定|
|`permissions`|`list[FilesystemPermission] \| None`|`None`|声明式文件系统权限规则（`operations`/`paths`/`mode`），控制读写范围，按声明顺序首个匹配生效|
|`backend`|`BackendProtocol \| BackendFactory \| None`|`None`|虚拟文件系统后端：内存态 `StateBackend`、持久态 `StoreBackend`、真实磁盘 `FilesystemBackend`、按路径路由的 `CompositeBackend`，或自定义/沙箱后端|
|`interrupt_on`|`dict[str, bool \| InterruptOnConfig] \| None`|`None`|人机协同：指定哪些工具（如 `"edit_file"`）需要人工审批后才能执行|
|`response_format`|`ResponseFormat \| type \| dict \| None`|`None`|同 `create_agent`，结构化输出配置|
|`state_schema`|`type[DeepAgentState] \| None`|`None`|自定义状态 schema（基于 `DeepAgentState`）|
|`context_schema`|`type \| None`|`None`|运行时上下文 schema|
|`checkpointer`|`Checkpointer \| None`|`None`|会话级持久化（同 `create_agent`）|
|`store`|`BaseStore \| None`|`None`|跨会话持久化（同 `create_agent`）|
|`debug`|`bool`|`False`|调试模式|
|`name`|`str \| None`|`None`|图名称|
|`cache`|`BaseCache \| None`|`None`|缓存后端|

```python
from deepagents import create_deep_agent
from langchain_core.tools import tool

@tool
def web_search(query: str) -> str:
    """Search the web for a query."""
    return f"Search results for: {query}"

# 定义一个专职研究的子智能体
research_subagent = {
    "name": "researcher",
    "description": "Delegate in-depth research tasks to this agent.",
    "system_prompt": "You are a meticulous research assistant.",
    "tools": [web_search],
}

agent = create_deep_agent(
    model="anthropic:claude-sonnet-4-6",
    tools=[web_search],
    system_prompt="You are a research assistant that plans, delegates, and writes reports.",
    subagents=[research_subagent],
    skills=["./skills/report-writing"],       # 目录需含 SKILL.md
    memory=["./AGENTS.md"],                   # 项目级长期记忆/约定
    permissions=[
        {"operations": ["write"], "paths": ["/workspace/**"], "mode": "allow"},
        {"operations": ["read", "write"], "paths": [".env", "**/secrets/**"], "mode": "deny"},
    ],
    interrupt_on={"edit_file": True},          # 修改文件前需人工审批
)

result = agent.invoke(
    {"messages": [{"role": "user", "content": "研究 LangGraph 的最新特性并写一份摘要"}]}
)
print(result["messages"][-1].content)
```

---

## 四、三者关系速记

- **LangGraph**：最底层图运行时（状态、检查点、流式、持久化）。
- **`create_agent`（LangChain）**：在 LangGraph 之上的轻量智能体工厂，`model + tools + middleware` 即可组装标准 ReAct 循环。
- **`create_deep_agent`（deepagents）**：在 `create_agent` 之上再封装一层"重装备"harness，默认内置文件系统、子智能体、技能、记忆、摘要压缩与提示缓存，适合长程复杂任务（编码/研究类）。

如果你的场景是"轻量对话/工具调用助手"，用 `create_agent` 足够；如果是"长程多步骤、需要规划-委派-文件读写"的任务，直接上 `create_deep_agent` 会省很多自建中间件的工作。
