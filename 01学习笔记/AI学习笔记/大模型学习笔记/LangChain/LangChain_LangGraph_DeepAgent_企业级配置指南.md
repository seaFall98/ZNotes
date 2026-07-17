# LangChain / LangGraph / DeepAgents 企业级配置指南

> 本文档基于以下版本源码与官方文档整理：
> - `langchain` **1.3.11** / `langchain-core` **1.4.8** / `langchain-openai` **1.3.3**
> - `langgraph` **1.2.8** / `langgraph-prebuilt` **1.1.0** / `langgraph-supervisor` **0.0.31**
> - `deepagents` **0.6.12**
>
> 适用 Python **3.10+**（推荐 3.12/3.13）。
>
> 重要版本变更提示：
> - LangChain 1.0 起 `langgraph.prebuilt.create_react_agent` **已废弃**，推荐使用 `langchain.agents.create_agent`（基于 middleware 架构）。
> - `pre_model_hook` / `post_model_hook` 已废弃，全部迁移至 middleware 的 `before_model` / `after_model` / `wrap_model_call`。
> - `state_schema` 仅支持 `TypedDict`，不再支持 Pydantic BaseModel 和 dataclass。
> - `deepagents` 0.5.3 起 `model=None` 默认值已废弃，1.0 将强制要求显式指定模型。

---

## 目录

- [LangChain / LangGraph / DeepAgents 企业级配置指南](#langchain--langgraph--deepagents-企业级配置指南)
  - [目录](#目录)
  - [一、核心参数速查表](#一核心参数速查表)
    - [1.1 ChatOpenAI 模型参数](#11-chatopenai-模型参数)
    - [1.2 create\_agent 代理参数](#12-create_agent-代理参数)
    - [1.3 create\_deep\_agent 参数](#13-create_deep_agent-参数)
    - [1.4 AgentMiddleware 钩子方法](#14-agentmiddleware-钩子方法)
  - [二、企业级模型构建模板](#二企业级模型构建模板)
  - [三、企业级代理创建模板（LangGraph）](#三企业级代理创建模板langgraph)
  - [四、企业级 DeepAgent 创建模板](#四企业级-deepagent-创建模板)
  - [五、安全、日志、性能与可观测性](#五安全日志性能与可观测性)
    - [5.1 安全考虑](#51-安全考虑)
    - [5.2 错误处理](#52-错误处理)
    - [5.3 日志记录](#53-日志记录)
    - [5.4 性能优化](#54-性能优化)
    - [5.5 可观测性](#55-可观测性)
    - [5.6 部署](#56-部署)
  - [六、生产环境检查清单](#六生产环境检查清单)
    - [6.1 基础环境](#61-基础环境)
    - [6.2 模型配置](#62-模型配置)
    - [6.3 持久化](#63-持久化)
    - [6.4 安全](#64-安全)
    - [6.5 可观测性](#65-可观测性)
    - [6.6 性能](#66-性能)
    - [6.7 调用规范](#67-调用规范)
    - [6.8 部署](#68-部署)
  - [附录：版本迁移关键点](#附录版本迁移关键点)
    - [LangChain v1.0 重大变更](#langchain-v10-重大变更)
    - [LangGraph v1.0 重大变更](#langgraph-v10-重大变更)
    - [DeepAgents 0.6.x 关键变更](#deepagents-06x-关键变更)

---

## 一、核心参数速查表

### 1.1 ChatOpenAI 模型参数

来源：`langchain_openai.chat_models.base.BaseChatOpenAI`（langchain-openai 1.3.3）

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `model` / `model_name` | `str` | `"gpt-3.5-turbo"` | 模型名（`model` 为别名） |
| `temperature` | `float \| None` | `None` | 采样温度 |
| `max_tokens` | `int \| None` | `None` | 单次生成最大 token 数 |
| `top_p` | `float \| None` | `None` | nucleus sampling 概率质量 |
| `n` | `int \| None` | `None` | 每次提示生成的候选数 |
| `streaming` | `bool` | `False` | 是否流式输出 |
| `stream_usage` | `bool \| None` | `None` | 流式输出中是否包含 usage（v0.3.9+） |
| `seed` | `int \| None` | `None` | 生成种子（可复现） |
| `stop` | `list[str] \| str \| None` | `None` | 停止序列（别名 `stop_sequences`） |
| `presence_penalty` | `float \| None` | `None` | 主题 novelty 惩罚 |
| `frequency_penalty` | `float \| None` | `None` | 频率惩罚 |
| `logprobs` | `bool \| None` | `None` | 是否返回 logprobs |
| `top_logprobs` | `int \| None` | `None` | 每位置返回 top-N token |
| `logit_bias` | `dict[int, int] \| None` | `None` | token 偏置 |
| `reasoning_effort` | `str \| None` | `None` | 推理强度（Chat Completions API）：`minimal/low/medium/high` |
| `reasoning` | `dict \| None` | `None` | 推理参数（Responses API，v0.3.24+）：`{"effort":..., "summary":"auto/concise/detailed"}` |
| `verbosity` | `str \| None` | `None` | 响应详细度（Responses API，v0.3.28+）：`low/medium/high` |
| `use_responses_api` | `bool \| None` | `None` | 是否使用 Responses API（v0.3.9+） |
| `api_key` / `openai_api_key` | `SecretStr \| Callable` | env | API Key，支持 sync/async callable（v0.3+） |
| `base_url` / `openai_api_base` | `str \| None` | env | API 基地址（代理/网关） |
| `organization` | `str \| None` | env | OpenAI 组织 ID |
| `openai_proxy` | `str \| None` | env | HTTP 代理 |
| `timeout` / `request_timeout` | `float \| tuple \| Any` | `None` | 请求超时（建议生产 ≥ 120s） |
| `max_retries` | `int \| None` | `None` | 最大重试次数（默认 6，生产建议 10-15） |
| `model_kwargs` | `dict[str, Any]` | `{}` | 透传给底层 API 的额外参数 |
| `default_headers` | `Mapping[str, str] \| None` | `None` | 默认请求头 |
| `default_query` | `Mapping[str, object] \| None` | `None` | 默认查询参数 |
| `http_client` | `Any \| None` | `None` | 自定义 `httpx.Client`（同步） |
| `http_async_client` | `Any \| None` | `None` | 自定义 `httpx.AsyncClient`（异步） |
| `http_socket_options` | `Sequence[tuple] \| None` | `None` | TCP socket 选项（keepalive 等） |
| `stream_chunk_timeout` | `float \| None` | `120.0` | 异步流式 chunk 超时（秒） |
| `tiktoken_model_name` | `str \| None` | `None` | tiktoken 计费模型名（兼容非标准模型） |
| `disable_streaming` | `bool \| "tool_calling"` | `False` | 禁用流式；`"tool_calling"` 仅在工具调用时禁用 |
| `rate_limiter` | `BaseRateLimiter \| None` | `None` | 速率限制器 |
| `cache` | `BaseCache \| None` | `None` | 实例级缓存 |
| `verbose` | `bool` | `False` | 详细日志 |

**关键方法签名：**

```python
# 绑定工具（v1.0+ 不再推荐在 create_agent 外使用）
def bind_tools(
    tools: Sequence[dict | type | Callable | BaseTool],
    *,
    tool_choice: str | None = None,
    **kwargs: Any,
) -> Runnable[LanguageModelInput, AIMessage]: ...

# 结构化输出（method/strict 在子类重载，ChatOpenAI 支持 json_schema/function_calling/json_mode）
def with_structured_output(
    schema: dict[str, Any] | type,
    *,
    include_raw: bool = False,
    **kwargs: Any,
) -> Runnable[LanguageModelInput, dict | BaseModel]: ...

# 统一工厂：支持 "provider:model" 格式
def init_chat_model(
    model: str | None = None,
    *,
    model_provider: str | None = None,
    configurable_fields: Literal["any"] | list[str] | tuple[str, ...] | None = None,
    config_prefix: str | None = None,
    **kwargs: Any,
) -> BaseChatModel | _ConfigurableModel: ...
```

### 1.2 create_agent 代理参数

来源：`langchain.agents.factory.create_agent`（langchain 1.3.11，**推荐用法**）

```python
def create_agent(
    model: str | BaseChatModel,
    tools: Sequence[BaseTool | Callable[..., Any] | dict[str, Any]] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware[StateT_co, ContextT]] = (),
    response_format: ResponseFormat[ResponseT] | type[ResponseT] | dict[str, Any] | None = None,
    state_schema: type[AgentState[ResponseT]] | None = None,  # 必须 TypedDict
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

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `model` | `str \| BaseChatModel` | 必填 | 模型实例或 `"openai:gpt-5"` 字符串 |
| `tools` | `Sequence[...] \| None` | `None` | 工具列表（推荐 `@tool` 装饰器） |
| `system_prompt` | `str \| SystemMessage \| None` | `None` | 系统提示词（v1.0 改名自 `prompt`） |
| `middleware` | `Sequence[AgentMiddleware]` | `()` | 中间件链（替代 pre/post_model_hook） |
| `response_format` | `ResponseFormat \| type \| dict \| None` | `None` | 结构化输出 schema |
| `state_schema` | `type[AgentState] \| None` | `None` | 状态 schema（**仅 TypedDict**） |
| `context_schema` | `type \| None` | `None` | 运行时不可变上下文 schema |
| `checkpointer` | `Checkpointer \| None` | `None` | 持久化 checkpoint |
| `store` | `BaseStore \| None` | `None` | 跨 thread 长期存储 |
| `interrupt_before` | `list[str] \| None` | `None` | 节点前中断 |
| `interrupt_after` | `list[str] \| None` | `None` | 节点后中断 |
| `debug` | `bool` | `False` | 调试模式 |
| `name` | `str \| None` | `None` | 图名称 |
| `cache` | `BaseCache \| None` | `None` | 缓存 |
| `transformers` | `Sequence[TransformerFactory] \| None` | `None` | 流式输出 transformer |

> ⚠️ `langgraph.prebuilt.create_react_agent` 已废弃，仍接受 `prompt / pre_model_hook / post_model_hook / version: Literal["v1","v2"]` 等参数，新代码请勿使用。

### 1.3 create_deep_agent 参数

来源：`deepagents.graph.create_deep_agent`（deepagents 0.6.12）

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

| 参数 | 类型 | 默认值 | 用途 |
|---|---|---|---|
| `model` | `str \| BaseChatModel` | `None`(已废弃) | 模型或 `"provider:model"` |
| `tools` | `Sequence \| None` | `None` | **追加**到内置工具集（write_todos/fs/execute/task） |
| `system_prompt` | `str \| SystemMessage \| None` | `None` | 自定义系统提示，置于 SDK 默认提示前 |
| `middleware` | `Sequence[AgentMiddleware]` | `()` | 用户中间件（插入到 base 与 tail 栈之间） |
| `subagents` | `Sequence \| None` | `None` | 子代理（SubAgent / CompiledSubAgent / AsyncSubAgent） |
| `skills` | `list[str] \| None` | `None` | 技能目录路径（POSIX 风格） |
| `memory` | `list[str] \| None` | `None` | `AGENTS.md` 文件路径列表 |
| `permissions` | `list[FilesystemPermission] \| None` | `None` | 文件系统权限规则（`allow/deny/interrupt`） |
| `backend` | `BackendProtocol \| BackendFactory \| None` | `StateBackend()` | 文件/执行后端 |
| `interrupt_on` | `dict[str, ...] \| None` | `None` | 工具级 HITL 审批 |
| `response_format` | `ResponseFormat \| type \| dict \| None` | `None` | 结构化输出 schema |
| `state_schema` | `type[DeepAgentState] \| None` | `None` | 自定义状态（必须继承 `DeepAgentState`） |
| `context_schema` | `type \| None` | `None` | 运行时上下文 schema |
| `checkpointer` | `Checkpointer \| None` | `None` | 持久化（生产必填） |
| `store` | `BaseStore \| None` | `None` | 长期存储 |
| `debug` | `bool` | `False` | 调试模式 |
| `name` | `str \| None` | `None` | 代理名称 |
| `cache` | `BaseCache \| None` | `None` | 缓存 |

**DeepAgents 默认中间件栈（顺序）：**

```
Base stack:
  1. TodoListMiddleware                      (任务规划)
  2. SkillsMiddleware                        (若 skills 提供)
  3. FilesystemMiddleware                    (虚拟 FS + 权限，不可排除)
  4. SubAgentMiddleware                      (若有同步子代理，不可排除)
  5. SummarizationMiddleware                 (上下文压缩)
  6. PatchToolCallsMiddleware                (悬空工具调用修复)
  7. AsyncSubAgentMiddleware                 (若有异步子代理)

[用户 middleware 插入位置]

Tail stack:
  - Harness profile extra_middleware
  - _ToolExclusionMiddleware                 (若有 excluded_tools)
  - AnthropicPromptCachingMiddleware         (无操作对非 Anthropic 模型)
  - BedrockPromptCachingMiddleware           (若安装 langchain-aws)
  - MemoryMiddleware                         (若 memory 提供)
  - HumanInTheLoopMiddleware                 (若 interrupt_on 提供)
```

### 1.4 AgentMiddleware 钩子方法

来源：`langchain.agents.middleware.types.AgentMiddleware`

| 钩子方法 | 同步 / 异步 | 触发时机 | 典型用途 |
|---|---|---|---|
| `before_agent` / `abefore_agent` | ✓ | 代理执行开始前 | 初始化状态、注入上下文 |
| `before_model` / `abefore_model` | ✓ | 每次模型调用前 | 消息裁剪、PII 脱敏、动态提示 |
| `after_model` / `aafter_model` | ✓ | 每次模型调用后 | 输出守卫、审批、遥测 |
| `after_agent` / `aafter_agent` | ✓ | 代理执行完成后 | 终态审计、资源清理、结果落库 |
| `wrap_model_call` / `awrap_model_call` | ✓ | 包裹整个模型调用 | 重试、缓存、降级、可观测性 |
| `wrap_tool_call` / `awrap_tool_call` | ✓ | 包裹工具调用 | 工具错误处理、审计 |

**类属性：**
- `state_schema: type[StateT]` — 中间件贡献的状态字段
- `tools: Sequence[BaseTool]` — 中间件注入的工具
- `transformers: Sequence[TransformerFactory]` — 流式输出 transformer
- `name: str`（property）— 默认为类名

**`wrap_model_call` 关键签名：**
```python
def wrap_model_call(
    self,
    request: ModelRequest[ContextT],          # 不可变，修改用 request.override(**changes)
    handler: Callable[[ModelRequest], ModelResponse[ResponseT]],
) -> ModelResponse[ResponseT] | AIMessage | ExtendedModelResponse[ResponseT]:
    # handler 可多次调用（重试）、可跳过（短路）、可包装响应
    ...
```

---

## 二、企业级模型构建模板

```python
"""
企业级 LangChain 模型构建模板
适用：langchain 1.3.x / langchain-openai 1.3.x / langchain-core 1.4.x
作者：基于源码与官方文档整理
"""
from __future__ import annotations

import logging
import os
from functools import lru_cache
from typing import Any

from langchain.chat_models import init_chat_model
from langchain_core.caches import BaseCache
from langchain_core.language_models import BaseChatModel
from langchain_core.rate_limiters import BaseRateLimiter, InMemoryRateLimiter
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field, SecretStr

logger = logging.getLogger(__name__)


# =============================================================================
# 1. 配置层（Pydantic Settings）—— 统一管理环境变量与运行参数
# =============================================================================
class ModelConfig(BaseModel):
    """模型配置：所有可调参数集中管理，避免散落代码各处。

    设计原则：
    - 敏感信息（API key）只从环境变量读取，不接受明文入参
    - 提供合理的企业级默认值（超时、重试）
    - 支持多 provider 切换（openai / anthropic / deepseek / google_genai 等）
    """

    # —— 模型本体参数 ——
    model: str = Field(default="gpt-4o-mini", description="模型名或 'provider:model' 格式")
    temperature: float = Field(default=0.0, description="采样温度，结构化输出场景固定为 0")
    max_tokens: int | None = Field(default=None, description="单次生成上限；None 表示由模型决定")
    top_p: float | None = Field(default=None, description="nucleus sampling，与 temperature 二选一")
    seed: int | None = Field(default=None, description="可复现场景使用")

    # —— 客户端参数（生产关键） ——
    timeout: float = Field(default=120.0, description="请求超时秒数，慢网络建议 ≥ 120")
    max_retries: int = Field(default=10, description="最大重试次数（默认 6 偏小，生产建议 10-15）")
    streaming: bool = Field(default=True, description="默认开启流式以降低首字延迟")

    # —— OpenAI 特有 ——
    reasoning_effort: str | None = Field(
        default=None, description="推理模型强度：minimal/low/medium/high"
    )
    use_responses_api: bool | None = Field(
        default=None, description="OpenAI Responses API 开关"
    )

    # —— 网络 / 代理 ——
    base_url: str | None = Field(default=None, description="API 网关或代理地址")
    organization: str | None = Field(default=None, description="OpenAI 组织 ID")
    openai_proxy: str | None = Field(default=None, description="HTTP 代理")

    # —— 业务标签 ——
    tags: list[str] = Field(default_factory=list, description="追踪标签")
    metadata: dict[str, Any] = Field(default_factory=dict, description="追踪元数据")

    model_config = {"extra": "forbid"}  # 严格模式：拼写错误立即报错


# =============================================================================
# 2. 速率限制器（防止单进程击穿 provider 配额）
# =============================================================================
def build_rate_limiter(
    requests_per_second: float = 2.0,
    check_every_n: int = 1,
    max_bucket_size: float = 10.0,
) -> InMemoryRateLimiter:
    """构造进程内速率限制器。

    生产建议：
    - 单实例：InMemoryRateLimiter 足够
    - 多实例：使用 Redis 后端的分布式限流器
    - 限流值应低于 provider 配额的 80%，留出缓冲
    """
    return InMemoryRateLimiter(
        requests_per_second=requests_per_second,
        check_every_n=check_every_n,
        max_bucket_size=max_bucket_size,
    )


# =============================================================================
# 3. 模型工厂（核心入口）
# =============================================================================
def build_chat_model(
    config: ModelConfig | None = None,
    *,
    rate_limiter: BaseRateLimiter | None = None,
    cache: BaseCache | None = None,
    extra_kwargs: dict[str, Any] | None = None,
) -> BaseChatModel:
    """构造一个企业级 ChatModel 实例。

    工厂统一负责：
    - API Key 校验（必须存在，否则 fail-fast）
    - 重试 / 超时 / 速率限制的合理默认值
    - 多 provider 透明切换（通过 init_chat_model）
    - 流式默认开启
    - 标签 / 元数据注入，便于 LangSmith 追踪

    Args:
        config: 模型配置；为 None 时使用默认配置
        rate_limiter: 速率限制器；为 None 时不启用
        cache: 实例级缓存；为 None 时使用全局缓存
        extra_kwargs: 透传给底层 provider 的额外参数

    Returns:
        BaseChatModel 实例

    Raises:
        ValueError: 缺少必要的 API Key
    """
    cfg = config or ModelConfig()

    # —— 安全：API Key 必须存在，绝不硬编码 ——
    # 仅对 OpenAI 系模型校验 OPENAI_API_KEY；其他 provider 由各自的 init_chat_model 校验
    api_key = os.getenv("OPENAI_API_KEY")
    needs_openai_key = (
        cfg.model.startswith("openai:")
        or cfg.model.startswith("gpt")
        or "gpt" in cfg.model
    )
    if needs_openai_key and not api_key:
        raise ValueError(
            "OPENAI_API_KEY 未设置。请通过环境变量或密钥管理服务注入，"
            "禁止在代码中硬编码。"
        )

    # —— 构造 kwargs ——
    kwargs: dict[str, Any] = {
        "temperature": cfg.temperature,
        "max_tokens": cfg.max_tokens,
        "timeout": cfg.timeout,
        "max_retries": cfg.max_retries,
        "streaming": cfg.streaming,
        "tags": cfg.tags or None,
        "metadata": cfg.metadata or None,
    }

    # 可选参数：None 不传入，使用 provider 默认
    optional_fields = {
        "top_p": cfg.top_p,
        "seed": cfg.seed,
        "base_url": cfg.base_url,
        "organization": cfg.organization,
        "openai_proxy": cfg.openai_proxy,
        "reasoning_effort": cfg.reasoning_effort,
        "use_responses_api": cfg.use_responses_api,
    }
    for k, v in optional_fields.items():
        if v is not None:
            kwargs[k] = v

    # 速率限制与缓存
    if rate_limiter is not None:
        kwargs["rate_limiter"] = rate_limiter
    if cache is not None:
        kwargs["cache"] = cache

    if extra_kwargs:
        kwargs.update(extra_kwargs)

    # —— 通过统一工厂创建，支持 "provider:model" 字符串 ——
    logger.info("初始化 ChatModel: model=%s, streaming=%s", cfg.model, cfg.streaming)
    return init_chat_model(cfg.model, **kwargs)


# =============================================================================
# 4. 结构化输出辅助函数
# =============================================================================
def build_structured_model(
    schema: type[BaseModel],
    config: ModelConfig | None = None,
    *,
    method: str = "json_schema",  # json_schema | function_calling | json_mode
    strict: bool = True,
    include_raw: bool = False,
    rate_limiter: BaseRateLimiter | None = None,
) -> Any:
    """构造一个返回结构化输出的模型包装器。

    生产建议：
    - 关键数据抽取用 strict=True（保证 schema 完全匹配）
    - 创意生成场景用 strict=False
    - 调试阶段 include_raw=True 以审计；生产关闭以简化类型
    - 结构化输出场景 temperature 必须 = 0（高温会导致校验失败重试增加 35x）

    Args:
        schema: Pydantic 模型类
        method: 输出策略
            - 'json_schema'：OpenAI 原生（最可靠，langchain-openai 0.3+ 默认）
            - 'function_calling'：兼容性最广
            - 'json_mode'：仅 JSON 模式，无 schema 约束
        strict: 严格模式（schema 完全匹配，限制见 OpenAI 文档）
        include_raw: 是否返回原始响应（调试用）

    Returns:
        Runnable，invoke 后返回 Pydantic 实例（或 dict）
    """
    # 结构化输出强制 temperature=0
    cfg = config or ModelConfig()
    cfg.temperature = 0.0
    cfg.streaming = False  # 结构化输出通常不需要流式

    model = build_chat_model(cfg, rate_limiter=rate_limiter)
    return model.with_structured_output(
        schema,
        method=method,
        strict=strict,
        include_raw=include_raw,
    )


# =============================================================================
# 5. 多模型 fallback（提升可用性）
# =============================================================================
def build_model_with_fallbacks(
    primary: BaseChatModel,
    fallbacks: list[BaseChatModel],
) -> BaseChatModel:
    """构造带 fallback 的模型：主模型失败时自动切换。

    适用场景：
    - 主模型限流时降级到便宜模型
    - 主模型超时时切换到快速模型
    - 多 provider 容灾

    注意：fallbacks 仅在异常时触发，不会主动负载均衡。
    """
    return primary.with_fallbacks(fallbacks)


# =============================================================================
# 6. 单例缓存（避免重复创建客户端连接池）
# =============================================================================
@lru_cache(maxsize=8)
def get_cached_model(model_name: str, temperature: float) -> BaseChatModel:
    """LRU 缓存模型实例，避免重复创建 httpx 连接池。

    注意：
    - 仅缓存常用配置组合
    - 长期运行的服务建议显式管理生命周期，而非依赖 LRU
    - 多线程/进程下，每个进程持有独立连接池
    """
    cfg = ModelConfig(model=model_name, temperature=temperature)
    return build_chat_model(cfg)


# =============================================================================
# 使用示例
# =============================================================================
if __name__ == "__main__":
    # 配置日志
    logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(name)s] %(levelname)s: %(message)s")

    # 1. 基础用法
    cfg = ModelConfig(
        model="gpt-4o-mini",
        temperature=0,
        max_retries=10,
        timeout=120,
        tags=["production", "v1.0"],
        metadata={"service": "enterprise-agent"},
    )
    model = build_chat_model(cfg, rate_limiter=build_rate_limiter())
    print(f"模型创建成功: {model.name}")

    # 2. 结构化输出
    from pydantic import BaseModel as PydBase

    class UserIntent(PydBase):
        """用户意图分类"""
        intent: str = Field(description="意图标签")
        confidence: float = Field(description="置信度 0-1")

    structured = build_structured_model(UserIntent, method="json_schema", strict=True)
    print("结构化模型创建成功")
```

---

## 三、企业级代理创建模板（LangGraph）

```python
"""
企业级 LangGraph 代理创建模板
适用：langchain 1.3.x / langgraph 1.2.x

注意：langgraph.prebuilt.create_react_agent 已废弃，
      本模板使用 langchain.agents.create_agent（基于 middleware 架构）。
"""
from __future__ import annotations

import logging
import os
import uuid
from collections.abc import Awaitable, Callable
from typing import Any, TypedDict

from langchain.agents import create_agent
from langchain.agents.middleware import (
    AgentMiddleware,
    HumanInTheLoopMiddleware,
    SummarizationMiddleware,
)
from langchain.agents.middleware.types import AgentState, ModelRequest, ModelResponse
from langchain_core.messages import AIMessage, BaseMessage, SystemMessage
from langchain_core.tools import BaseTool, tool
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.memory import MemorySaver  # 仅开发用
from langgraph.checkpoint.postgres import PostgresSaver  # 生产推荐
from langgraph.store.memory import InMemoryStore
from pydantic import BaseModel, Field

logger = logging.getLogger(__name__)


# =============================================================================
# 1. 自定义状态 Schema（必须 TypedDict，v1.0 重大变更）
# =============================================================================
class EnterpriseAgentState(TypedDict, total=False):
    """企业级代理状态扩展。

    v1.0 重要变更：
    - 仅支持 TypedDict，不再支持 Pydantic BaseModel / dataclass
    - 通过 middleware 的 state_schema 属性扩展状态更优
      （状态字段与相关 middleware/工具的作用域一致）

    内置字段（来自 AgentState）：
    - messages: add_messages reducer
    - structured_response: 结构化输出
    """

    # 业务扩展字段
    user_id: str
    session_id: str
    tenant_id: str
    request_id: str
    # 自定义 reducer 字段示例：last_intent: Annotated[str, lambda a, b: b]


# =============================================================================
# 2. 运行时上下文 Schema（不可变，每次调用注入）
# =============================================================================
class EnterpriseContext(TypedDict):
    """运行时上下文：跨节点共享的不可变数据。

    与 state_schema 的区别：
    - state: 可变，节点可写回；持久化到 checkpointer
    - context: 不可变，整个执行周期内只读；不持久化

    典型用途：user_id、API key、feature flags、租户隔离
    """

    user_id: str
    tenant_id: str
    environment: str  # dev / staging / production
    feature_flags: dict[str, bool]


# =============================================================================
# 3. 自定义中间件：可观测性 + 安全
# =============================================================================
class ObservabilityMiddleware(AgentMiddleware):
    """可观测性中间件：统一记录模型调用、token 消耗、耗时。

    生产部署建议：
    - 将指标发送到 Prometheus / Datadog / OpenTelemetry
    - 通过 metadata 关联 trace_id / span_id
    - 监控 P50/P95/P99 延迟与错误率
    """

    def before_agent(
        self, state: AgentState, runtime
    ) -> dict[str, Any] | None:
        logger.info(
            "代理启动: request_id=%s, user_id=%s",
            state.get("request_id"),
            runtime.context.get("user_id") if runtime.context else None,
        )
        return None

    def wrap_model_call(self, request, handler):
        """包裹模型调用，记录耗时与 token。

        handler 可多次调用（重试）、可跳过（短路）、可包装响应。
        ModelRequest 不可变，修改需用 request.override(**changes)。
        """
        import time

        start = time.perf_counter()
        try:
            response = handler(request)
            elapsed = time.perf_counter() - start
            # 从响应中提取 token 使用量
            if isinstance(response, ModelResponse):
                for msg in response.result:
                    if isinstance(msg, AIMessage) and msg.usage_metadata:
                        logger.info(
                            "模型调用成功: elapsed=%.3fs, tokens=%s",
                            elapsed,
                            msg.usage_metadata,
                        )
            return response
        except Exception as e:
            elapsed = time.perf_counter() - start
            logger.exception("模型调用失败: elapsed=%.3fs, error=%s", elapsed, e)
            raise


class PIIFilterMiddleware(AgentMiddleware):
    """PII 脱敏中间件：模型调用前过滤敏感信息。

    规则示例（按需扩展）：
    - SSN: \b\d{3}-?\d{2}-?\d{4}\b
    - 信用卡: \b(?:\d[ -]*?){13,16}\b
    - 邮箱: 替换为 <email>
    - 手机号: 替换为 <phone>

    生产建议：
    - 规则库应可热更新（从配置中心拉取）
    - 脱敏映射表加密存储，便于反查
    """

    def before_model(self, state, runtime) -> dict[str, Any] | None:
        messages = state.get("messages", [])
        # 实际场景在此应用正则脱敏
        # cleaned = [self._filter(msg) for msg in messages]
        # return {"messages": cleaned}
        return None


class AuditLogMiddleware(AgentMiddleware):
    """审计日志中间件：记录所有工具调用，用于合规审计。

    与 ObservabilityMiddleware 的区别：
    - Observability: 关注性能指标
    - Audit: 关注业务行为（who/what/when），需持久化
    """

    def wrap_tool_call(self, request, handler):
        # ToolCallRequest 字段：tool_call(dict), tool(BaseTool|None), state, runtime
        # tool_call 是 dict，含 name/args/id
        tool_name = request.tool_call["name"]
        tool_args = request.tool_call["args"]
        logger.info("工具调用: name=%s, args=%s", tool_name, tool_args)
        try:
            result = handler(request)
            logger.info("工具完成: name=%s, status=success", tool_name)
            return result
        except Exception as e:
            logger.warning("工具失败: name=%s, error=%s", tool_name, e)
            raise


# =============================================================================
# 4. 工具定义规范
# =============================================================================
class QueryDatabaseInput(BaseModel):
    """工具输入 Schema：严格校验用户输入。"""

    sql: str = Field(description="只读 SQL 查询语句，禁止 INSERT/UPDATE/DELETE")
    limit: int = Field(default=100, ge=1, le=1000, description="返回行数上限")


@tool(args_schema=QueryDatabaseInput)
def query_database(sql: str, limit: int = 100) -> str:
    """查询只读数据库副本。

    工具描述规范：
    - 第一行：动词开头，描述"做什么"
    - 第二段：描述"何时使用"
    - 边界条件：明确不支持的场景

    安全实践：
    - 输入参数用 Pydantic 严格校验
    - SQL 走只读副本 + 语句白名单
    - 限制返回行数
    """
    # 实际实现：连接只读副本，执行 SQL
    return f"查询返回 {limit} 行结果（示例）"


@tool
def send_email(to: str, subject: str, body: str) -> str:
    """发送邮件。

    高危操作：本工具应在 interrupt_on 中配置审批。
    """
    # 实际实现：调用邮件服务
    return f"邮件已发送至 {to}"


# =============================================================================
# 5. Checkpointer 工厂（生产 vs 开发）
# =============================================================================
def build_checkpointer(environment: str = "dev"):
    """构造 checkpointer。

    环境             | Checkpointer              | 持久化 | 性能  | 并发
    -----------------|---------------------------|--------|-------|------
    dev              | MemorySaver               | 否     | 最快  | 单进程
    staging          | SqliteSaver               | 是     | 中    | 单进程
    production       | PostgresSaver             | 是     | 高    | 多进程
    production (异步)| AsyncPostgresSaver        | 是     | 最高  | 多进程

    导入路径：
        - MemorySaver: from langgraph.checkpoint.memory import MemorySaver
        - SqliteSaver: from langgraph.checkpoint.sqlite import SqliteSaver
        - PostgresSaver: from langgraph.checkpoint.postgres import PostgresSaver
        - AsyncPostgresSaver: from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

    重要：PostgresSaver/AsyncPostgresSaver 的 from_conn_string 是
    @contextmanager，返回 Iterator，不能直接作为 checkpointer 返回。
    工厂模式下应直接构造（传入已建好的 conn），并显式调用 setup() 建表。

    生产注意：
    - thread_id 长度 ≤ 255 字符（PostgresSaver 限制）
    - 建议用 UUID7（PostgresSaver 可索引优化）
    - 定期 cron 清理过期 checkpoints
    - 首次使用必须调用 checkpointer.setup() 创建表与迁移
    """
    if environment == "production":
        # 生产：直接构造，避免 context manager 生命周期问题
        dsn = os.getenv("POSTGRES_DSN")
        if not dsn:
            raise ValueError("POSTGRES_DSN 未设置")
        # 同步版本
        import psycopg
        from psycopg.rows import dict_row

        conn = psycopg.connect(
            dsn, autocommit=True, prepare_threshold=0, row_factory=dict_row
        )
        checkpointer = PostgresSaver(conn)
        checkpointer.setup()  # 首次必须调用，创建表与迁移
        return checkpointer
        # 异步版本（推荐生产）：
        #   from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
        #   import psycopg_pool
        #   pool = psycopg_pool.AsyncConnectionPool(conninfo=dsn, kwargs={"autocommit": True, "prepare_threshold": 0, "row_factory": dict_row})
        #   checkpointer = AsyncPostgresSaver(pool)
        #   await checkpointer.setup()
    elif environment == "staging":
        # SqliteSaver.from_conn_string 同样是 context manager；
        # 同样建议用同步上下文持有，或直接构造
        from langgraph.checkpoint.sqlite import SqliteSaver
        import sqlite3
        conn = sqlite3.connect("checkpoints.db", check_same_thread=False)
        return SqliteSaver(conn)
    else:
        # 仅开发测试：进程重启后丢失
        return MemorySaver()


# =============================================================================
# 6. 代理工厂（核心入口）
# =============================================================================
def build_enterprise_agent(
    model: ChatOpenAI,
    *,
    environment: str = "dev",
    enable_pii_filter: bool = True,
    enable_audit_log: bool = True,
    enable_human_in_loop: bool = False,
    enable_summarization: bool = True,
    extra_tools: list[BaseTool | Callable] | None = None,
) -> Any:
    """构造企业级 LangGraph 代理。

    通过 middleware 组合实现横切关注点：
    - 可观测性（必选）
    - PII 脱敏（生产必选）
    - 审计日志（合规场景必选）
    - 上下文摘要（长对话必选）
    - 人机协作（高危操作必选）

    Args:
        model: 已配置好的 ChatModel 实例
        environment: 部署环境
        enable_pii_filter: 启用 PII 脱敏
        enable_audit_log: 启用审计日志
        enable_human_in_loop: 启用 HITL（需要 checkpointer）
        enable_summarization: 启用自动摘要
        extra_tools: 额外工具

    Returns:
        CompiledStateGraph
    """
    # —— 组装中间件 ——
    middleware: list[AgentMiddleware] = [ObservabilityMiddleware()]

    if enable_pii_filter:
        middleware.append(PIIFilterMiddleware())

    if enable_audit_log:
        middleware.append(AuditLogMiddleware())

    if enable_summarization:
        # 内置摘要中间件：上下文溢出时自动压缩
        # 注意：SummarizationMiddleware 的 model 是必填参数
        middleware.append(SummarizationMiddleware(model=model))

    if enable_human_in_loop:
        # HumanInTheLoopMiddleware 的 interrupt_on 是必填参数，必须显式指定
        # 哪些工具需要审批。True 表示允许 approve/edit/reject/respond 全部决策。
        middleware.append(HumanInTheLoopMiddleware(
            interrupt_on={"send_email": True},  # 按业务扩展审批工具集
        ))

    # —— 工具集 ——
    tools: list[BaseTool | Callable] = [query_database]
    if extra_tools:
        tools.extend(extra_tools)

    # —— Checkpointer 与 Store ——
    checkpointer = build_checkpointer(environment)
    store = InMemoryStore()  # 生产建议替换为 PostgresStore / RedisStore

    # —— 系统提示词 ——
    system_prompt = """你是一个企业级 AI 助手。请遵循：
1. 严格使用工具完成业务任务，不要臆造数据
2. 涉及敏感操作（发邮件、修改数据）时请求用户确认
3. 回答简洁、准确、可追溯
4. 用户身份信息从 context 中读取，不要询问
"""

    # —— 构造代理 ——
    agent = create_agent(
        model=model,
        tools=tools,
        system_prompt=system_prompt,
        middleware=middleware,
        context_schema=EnterpriseContext,
        checkpointer=checkpointer,
        store=store,
        name="enterprise-agent",
        debug=(environment == "dev"),
    )

    logger.info("企业级代理创建成功: env=%s, middleware=%d", environment, len(middleware))
    return agent


# =============================================================================
# 7. 调用示例
# =============================================================================
async def invoke_agent_example():
    """异步调用代理的标准模式。"""
    from langchain_openai import ChatOpenAI

    model = ChatOpenAI(model="gpt-4o-mini", temperature=0, max_retries=10)
    agent = build_enterprise_agent(model, environment="dev")

    # 配置 thread_id（持久化关键）
    config = {
        "configurable": {"thread_id": str(uuid.uuid4())},
        "context": EnterpriseContext(
            user_id="user-123",
            tenant_id="tenant-abc",
            environment="dev",
            feature_flags={"enable_email": True},
        ),
    }

    # 异步流式输出（生产推荐 stream_events v3）
    async for event in agent.astream_events(
        {"messages": [("user", "查询最近 10 条订单")]},
        config=config,
        version="v3",
    ):
        kind = event["event"]
        if kind == "on_chat_model_stream":
            chunk = event["data"]["chunk"]
            print(chunk.content, end="", flush=True)
        elif kind == "on_tool_start":
            logger.info("工具开始: %s", event["name"])


if __name__ == "__main__":
    import asyncio
    logging.basicConfig(level=logging.INFO)
    asyncio.run(invoke_agent_example())
```

---

## 四、企业级 DeepAgent 创建模板

```python
"""
企业级 DeepAgents 创建模板
适用：deepagents 0.6.x / langchain 1.3.x / langgraph 1.2.x

DeepAgents 内置：
- 任务规划（TodoListMiddleware）
- 虚拟文件系统（FilesystemMiddleware）
- 子代理协作（SubAgentMiddleware）
- 上下文压缩（SummarizationMiddleware）
- 工具调用修复（PatchToolCallsMiddleware）
- Anthropic 提示缓存（AnthropicPromptCachingMiddleware）
"""
from __future__ import annotations

import logging
import os
import uuid
from typing import Any, TypedDict

from deepagents import DeepAgentState, create_deep_agent
from deepagents.backends.filesystem import FilesystemBackend
from deepagents.backends.local_shell import LocalShellBackend
from deepagents.backends.state import StateBackend
from deepagents.middleware import (
    FilesystemMiddleware,
    SubAgentMiddleware,
    SubAgent,
    AsyncSubAgent,
    CompiledSubAgent,
)
from deepagents.middleware.permissions import FilesystemPermission
from langchain.agents.middleware import AgentMiddleware, HumanInTheLoopMiddleware
from langchain.agents.middleware.types import ModelRequest, ModelResponse
from langchain_core.messages import AIMessage
from langchain_core.tools import BaseTool, tool
from langchain_openai import ChatOpenAI
from langgraph.checkpoint.postgres import PostgresSaver
from langgraph.checkpoint.memory import MemorySaver
from pydantic import BaseModel, Field

logger = logging.getLogger(__name__)


# =============================================================================
# 1. 自定义 DeepAgent 状态（必须继承 DeepAgentState）
# =============================================================================
class EnterpriseDeepAgentState(DeepAgentState, total=False):
    """DeepAgent 状态扩展。

    注意：state_schema 必须继承 DeepAgentState，
    以保留 messages 的 DeltaChannel reducer。

    生产建议：优先通过 middleware 的 state_schema 扩展状态，
    仅在多 middleware 共享字段时使用顶层 state_schema。
    """

    project_id: str
    budget_tokens: int  # 预算控制
    audit_trail: list[dict[str, Any]]  # 审计链路


# =============================================================================
# 2. 子代理配置（领域专家分工）
# =============================================================================
# 子代理规范：
# - name 唯一，主代理通过 task(name=...) 调用
# - description 必须清晰、动作导向（决定何时调用）
# - system_prompt 详细且自包含（子代理不继承主代理上下文）
# - tools 最小化（避免选择困惑）
# - 复杂推理用强模型，简单任务用快模型

RESEARCHER_SUBAGENT: SubAgent = {
    "name": "researcher",
    "description": (
        "当需要在网络或文档库中检索信息、综合多源资料、"
        "生成研究报告时调用此子代理。"
    ),
    "system_prompt": (
        "你是一个研究专家。请：\n"
        "1. 使用搜索工具检索权威信息源\n"
        "2. 交叉验证至少 2 个独立来源\n"
        "3. 返回结构化结论，标注引用与置信度\n"
        "4. 不要臆造数据，找不到信息时明确说明"
    ),
    # "model": "openai:gpt-4o",  # 复杂推理用强模型
    # "tools": [search_tool, ...],  # 仅给必要工具
}

CODER_SUBAGENT: SubAgent = {
    "name": "coder",
    "description": (
        "当需要编写、修改、调试代码，运行测试或重构时调用此子代理。"
    ),
    "system_prompt": (
        "你是一个资深工程师。请：\n"
        "1. 严格遵循项目编码规范\n"
        "2. 修改前先阅读相关文件，理解上下文\n"
        "3. 编写后运行测试验证\n"
        "4. 返回 diff 形式的修改摘要"
    ),
}


# =============================================================================
# 3. 权限配置（文件系统安全的关键）
# =============================================================================
def build_permissions() -> list[FilesystemPermission]:
    """文件系统权限规则。

    FilesystemPermission 字段（@dataclass）：
    - operations: list[FilesystemOperation]  操作类型，"read" 和/或 "write"
    - paths: list[str]                        路径 glob 列表（必须以 / 开头）
    - mode: "allow" | "deny" | "interrupt"    匹配时的处置

    工具到操作的映射（由 FilesystemMiddleware 自动应用）：
        read  -> ls, read_file, glob, grep
        write -> write_file, edit_file

    规则按声明顺序评估，第一个匹配生效。
    无规则匹配时默认 allow。

    mode 取值：
    - "allow"（默认）：允许
    - "deny"：拒绝，返回 permission-denied
    - "interrupt"：暂停等待人工审批（需 checkpointer）；
                  建议配合字面前导锚定模式（如 /secrets/**）

    子代理默认继承父代理规则；
    若子代理指定自己的 permissions，则完全替换父规则。
    """
    return [
        # 允许读写项目工作目录
        FilesystemPermission(
            operations=["read", "write"], paths=["/workspace/**"], mode="allow"
        ),
        # 禁止读取系统目录
        FilesystemPermission(
            operations=["read", "write"], paths=["/etc/**"], mode="deny"
        ),
        FilesystemPermission(
            operations=["read", "write"], paths=["/root/**"], mode="deny"
        ),
        FilesystemPermission(
            operations=["read", "write"], paths=["/secrets/**"], mode="deny"
        ),
        # 生产目录的写操作需要审批（read 仍允许）
        FilesystemPermission(
            operations=["write"], paths=["/workspace/production/**"], mode="interrupt"
        ),
    ]


# =============================================================================
# 4. 后端选择
# =============================================================================
def build_backend(kind: str = "state"):
    """构造执行后端。

    kind     | Backend            | 适用场景
    ---------|--------------------|------------------------------
    state    | StateBackend       | 默认；通过 invoke(files=...) 提供
    fs       | FilesystemBackend  | 直接读写磁盘（需权限严格控制）
    shell    | LocalShellBackend  | 执行 shell 命令（高危！）
    composite| 组合              | 多后端组合
    """
    if kind == "fs":
        # 文件系统后端：root_dir 必须严格限制
        return FilesystemBackend(root_dir="/workspace")
    elif kind == "shell":
        # Shell 后端：生产慎用，必须配合 permissions 与 interrupt_on
        # 注意：LocalShellBackend 继承 FilesystemBackend，参数是 root_dir（无 workdir）
        # 额外支持 timeout / max_output_bytes / env / inherit_env
        return LocalShellBackend(root_dir="/workspace")
    else:
        # 默认状态后端：通过 invoke 入参传入文件
        return StateBackend()


# =============================================================================
# 5. 自定义中间件（DeepAgents 兼容 LangChain middleware）
# =============================================================================
class BudgetControlMiddleware(AgentMiddleware):
    """预算控制中间件：限制 token 总消耗。

    实现要点：
    - 通过 wrap_model_call 拦截每次模型调用
    - 累计 token，超额时短路（不调用 handler）
    - 配合 state_schema 累计 budget_tokens
    """

    def wrap_model_call(self, request: ModelRequest, handler):
        # 实际实现：从 state 读取已消耗 token，判断是否超额
        # if state["budget_tokens"] > LIMIT:
        #     return ModelResponse(result=[AIMessage(content="预算已耗尽")])
        return handler(request)


class TraceMiddleware(AgentMiddleware):
    """链路追踪中间件：发送到 OpenTelemetry / LangSmith。"""

    def before_agent(self, state, runtime) -> dict[str, Any] | None:
        logger.info(
            "DeepAgent 启动: project_id=%s",
            state.get("project_id", "unknown"),
        )
        return None

    def wrap_model_call(self, request, handler):
        import time
        start = time.perf_counter()
        response = handler(request)
        elapsed = time.perf_counter() - start
        # 实际场景：发送 span 到 OTel collector
        logger.debug("模型调用 span: elapsed=%.3fs", elapsed)
        return response


# =============================================================================
# 6. 工具定义
# =============================================================================
@tool
def query_knowledge_base(query: str) -> str:
    """查询企业知识库。

    何时使用：用户询问公司产品、政策、流程等内部信息时。
    """
    return f"知识库结果：{query}（示例）"


@tool
def create_jira_ticket(title: str, description: str) -> str:
    """创建 Jira 工单。本工具应配置在 interrupt_on 中审批。"""
    return f"工单 {title} 已创建"


# =============================================================================
# 7. DeepAgent 工厂（核心入口）
# =============================================================================
def build_enterprise_deep_agent(
    model: ChatOpenAI,
    *,
    environment: str = "dev",
    enable_subagents: bool = True,
    enable_shell: bool = False,
    enable_audit: bool = True,
    enable_budget_control: bool = True,
) -> Any:
    """构造企业级 DeepAgent。

    DeepAgents 自动注入的中间件栈：
    1. TodoListMiddleware（任务规划）
    2. FilesystemMiddleware（FS + 权限，不可排除）
    3. SubAgentMiddleware（若有子代理，不可排除）
    4. SummarizationMiddleware（上下文压缩）
    5. PatchToolCallsMiddleware（工具修复）
    [用户 middleware]
    Tail:
    - AnthropicPromptCachingMiddleware（自动启用）
    - MemoryMiddleware（若 memory 提供）
    - HumanInTheLoopMiddleware（若 interrupt_on 提供）

    Args:
        model: ChatModel 实例
        environment: 部署环境
        enable_subagents: 启用子代理
        enable_shell: 启用 shell 执行（高危）
        enable_audit: 启用审计中间件
        enable_budget_control: 启用预算控制

    Returns:
        CompiledStateGraph
    """
    # —— 用户中间件 ——
    user_middleware: list[AgentMiddleware] = [TraceMiddleware()]

    if enable_budget_control:
        user_middleware.append(BudgetControlMiddleware())

    # —— 子代理 ——
    subagents = None
    if enable_subagents:
        subagents = [RESEARCHER_SUBAGENT, CODER_SUBAGENT]

    # —— 权限 ——
    permissions = build_permissions()

    # —— 后端 ——
    backend = build_backend("shell" if enable_shell else "state")

    # —— HITL 审批配置 ——
    interrupt_on: dict[str, Any] = {
        "create_jira_ticket": True,  # 创建工单前审批
        "send_email": True,          # 发邮件前审批
    }
    if enable_shell:
        interrupt_on["execute"] = True  # shell 命令必须审批

    # —— Checkpointer ——
    # 注意：PostgresSaver.from_conn_string 是 @contextmanager，不能直接构造返回。
    # 工厂模式下用 psycopg.Connection 直接构造，并调用 setup() 建表。
    if environment == "production":
        dsn = os.getenv("POSTGRES_DSN")
        if not dsn:
            raise ValueError("POSTGRES_DSN 未设置")
        import psycopg
        from psycopg.rows import dict_row

        conn = psycopg.connect(
            dsn, autocommit=True, prepare_threshold=0, row_factory=dict_row
        )
        checkpointer = PostgresSaver(conn)
        checkpointer.setup()  # 首次必须调用，创建表与迁移
    else:
        checkpointer = MemorySaver()

    # —— 系统提示 ——
    system_prompt = """你是一个企业级 Deep Agent，具备多步骤任务规划与执行能力。

工作准则：
1. 复杂任务先制定 todo list，逐步推进
2. 涉及生产环境修改时，工具会暂停等待人工审批，请理解并配合
3. 子代理任务结果会自动回到你这里，请整合后给用户
4. 严格在 /workspace 目录内操作，禁止访问系统目录
5. 找不到信息时明确说明，不要臆造
"""

    # —— 构造 DeepAgent ——
    agent = create_deep_agent(
        model=model,
        tools=[query_knowledge_base, create_jira_ticket],
        system_prompt=system_prompt,
        middleware=user_middleware,
        subagents=subagents,
        permissions=permissions,
        backend=backend,
        interrupt_on=interrupt_on,
        state_schema=EnterpriseDeepAgentState,
        checkpointer=checkpointer,
        store=None,  # 生产建议配置 PostgresStore
        debug=(environment == "dev"),
        name="enterprise-deep-agent",
    )

    logger.info(
        "企业级 DeepAgent 创建成功: env=%s, subagents=%d, shell=%s",
        environment,
        len(subagents or []),
        enable_shell,
    )
    return agent


# =============================================================================
# 8. 调用示例（含 HITL 恢复）
# =============================================================================
async def invoke_deep_agent_example():
    """调用 DeepAgent 的完整流程，含人工审批恢复。"""
    from langchain_core.messages import HumanMessage
    from langgraph.types import Command

    model = ChatOpenAI(model="gpt-4o", temperature=0, max_retries=10)
    agent = build_enterprise_deep_agent(model, environment="dev")

    thread_id = str(uuid.uuid4())
    config = {"configurable": {"thread_id": thread_id}}

    # 第一次调用：可能因 interrupt_on 暂停
    async for event in agent.astream_events(
        {"messages": [HumanMessage("为 BUG-123 创建一个 Jira 工单")]},
        config=config,
        version="v3",
    ):
        if event["event"] == "on_interrupt":
            # 工具调用被暂停，等待人工审批
            logger.info("等待人工审批: %s", event["data"])
            # 实际场景：通知审批系统，等待人工决策
            # 审批通过后，用 Command(resume=...) 恢复执行
            decision = "approved"  # 从审批系统获取
            async for resume_event in agent.astream_events(
                Command(resume=decision), config=config, version="v3"
            ):
                # 处理恢复后的事件
                pass
        elif event["event"] == "on_chat_model_stream":
            chunk = event["data"]["chunk"]
            print(chunk.content, end="", flush=True)


if __name__ == "__main__":
    import asyncio
    logging.basicConfig(level=logging.INFO)
    asyncio.run(invoke_deep_agent_example())
```

---

## 五、安全、日志、性能与可观测性

### 5.1 安全考虑

| 维度 | 实践 |
|---|---|
| **API Key 管理** | 绝不硬编码；通过环境变量或 Vault / AWS Secrets Manager 注入；LangSmith Workspace Secrets 共享组织密钥 |
| **OAuth 凭据** | 使用 LangSmith Agent Auth 托管 OAuth 2.0 流程；沙箱代码通过 sandbox auth proxy 自动注入 |
| **PII 脱敏** | 部署 LangSmith anonymizers 屏蔽 SSN/信用卡/邮箱；通过 `LangChainTracer` 自动应用到所有 trace |
| **工具权限** | DeepAgents `FilesystemPermission` 路径级控制；`interrupt_on` 配置高危工具审批 |
| **输入校验** | `context_schema` 强类型运行时上下文；工具用 Pydantic 严格校验；middleware 层实现输入守卫 |
| **多租户** | custom auth 建立用户身份；authorization handlers 控制资源访问；ownership 元数据隔离 |
| **SQL 注入** | 数据库工具走只读副本 + 语句白名单；限制返回行数 |
| **Shell 执行** | 慎用 LocalShellBackend；必须配合 permissions 与 interrupt_on |

### 5.2 错误处理

**重试策略：**
- LangChain 内置自动重试：网络错误 / 429 / 5xx，指数退避 + 抖动
- 客户端错误（401/404）不重试
- 生产配置：`max_retries=10-15`（默认 6 偏小），`timeout=120`
- 长时 agent 必须配 checkpointer，失败可从断点恢复

**优雅降级：**
```python
# 通过 with_fallbacks 实现主备切换
primary = ChatOpenAI(model="gpt-4o", max_retries=3)
fallback = ChatOpenAI(model="gpt-4o-mini", max_retries=5)
model = primary.with_fallbacks([fallback])
```

**中间件层错误恢复（DeepAgents 内置）：**
- `SummarizationMiddleware`：检测 `ContextOverflowError`，自动摘要旧消息重试
- `LocalShellBackend`：命令超时返回结构化错误，引导 LLM 用 `timeout` 参数重试
- `FilesystemMiddleware`：工具输出超 `tool_token_limit_before_evict` 时持久化到后端
- `PatchToolCallsMiddleware`：扫描悬空工具调用，注入合成错误让 agent 可感知

### 5.3 日志记录

**LangSmith 集成（核心）：**
```bash
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY=...
export LANGSMITH_PROJECT=my-agent-project
```

**自定义 Callback（v1.0 类型化）：**
```python
from langchain_core.callbacks import AsyncCallbackHandler

class MetricsCallback(AsyncCallbackHandler):
    """通过 metadata 注入业务指标。"""
    async def on_llm_start(self, serialized, prompts, **kwargs):
        # 记录调用开始
        pass
    async def on_llm_end(self, response, **kwargs):
        # 记录 token 消耗、延迟
        pass
```

**通过 middleware 统一记录（推荐）：**
见前文 `ObservabilityMiddleware` 与 `AuditLogMiddleware`。

### 5.4 性能优化

| 优化点 | 实践 |
|---|---|
| **并发** | 异步优先：`ainvoke` / `astream` / `abatch`；创建 native async 工具避免线程开销 |
| **批处理** | `abatch` 一次提交多请求；LangGraph Superstep 内节点并行 |
| **流式** | 生产用 `stream_events(version="v3")`；typed projections 独立消费；`nostream` 标签排除内部 LLM |
| **缓存** | 高并发用 RedisCache；语义缓存用 MomentoCache 节省 30-50%；配置 TTL |
| **Pydantic v2** | 全量使用；`state_schema` 用 TypedDict（v1.0 强制） |
| **连接池复用** | 模型实例 LRU 缓存；httpx 连接池共享 |
| **提示缓存** | DeepAgents 自动启用 `AnthropicPromptCachingMiddleware`，大幅降低重复提示成本 |
| **结构化输出** | `temperature=0` 避免校验失败重试增加 35x 开销 |
| **Python 3.11+** | 异步代码无需显式传 RunnableConfig；context vars 自动传播 |

### 5.5 可观测性

**LangSmith 追踪能力：**
- 自动记录每个 LLM 调用、工具调用、状态转换
- 时间旅行回到历史 checkpoint
- LangSmith Studio 可视化调试
- DeepAgents 子代理支持按子代理名过滤 trace

**指标注入规范：**
```python
config = {
    "tags": ["production", "v1.0", "tenant-abc"],
    "metadata": {
        "user_id": "user-123",
        "session_id": "sess-456",
        "request_id": "req-789",
        "environment": "production",
    },
}
```

**LangSmith Engine（推荐）：**
- 监控 trace 自动检测异常
- 提供修复建议
- 适用于生产环境的智能监控

### 5.6 部署

**LangGraph Platform / LangSmith Deployments（推荐）：**
- 自动提供 threads / runs / store / checkpointer 基础设施
- 支持 MCP 与 A2A 协议
- 自动 trace 发送到以 deployment 命名的项目

**Docker 容器化：**
```json
// langgraph.json
{
  "dependencies": ["."],
  "graphs": {"agent": "./agent.py:agent"},
  "env": ".env"
}
```

```bash
# 本地开发
langgraph dev

# 生产部署
langgraph build  # 构建镜像
langgraph up     # 启动
```

**调用规范（生产必读）：**
```python
# 每次调用必须传 thread_id 与 context
config = {
    "configurable": {"thread_id": str(uuid.uuid4())},  # ≤ 255 字符
    "context": EnterpriseContext(...),
}
result = await agent.ainvoke({"messages": [...]}, config=config)
```

---

## 六、生产环境检查清单

### 6.1 基础环境
- [ ] Python 3.10+（推荐 3.12/3.13）
- [ ] LangChain 1.x / LangGraph 1.x / DeepAgents 0.6.x 最新稳定版
- [ ] 依赖锁定（`requirements.txt` 或 `pyproject.toml` + lock 文件）

### 6.2 模型配置
- [ ] `max_retries=10-15`（默认 6 偏小）
- [ ] `timeout=120` 秒以上
- [ ] API Key 通过环境变量或密钥管理服务
- [ ] 结构化输出场景 `temperature=0`
- [ ] 多 provider 容灾配置 `with_fallbacks`
- [ ] 流式默认开启以降低首字延迟

### 6.3 持久化
- [ ] 生产环境使用 `PostgresSaver`（非 `MemorySaver`）
- [ ] `thread_id` 使用 UUID，长度 ≤ 255 字符
- [ ] 定期 cron 清理过期 checkpoints
- [ ] 跨 thread 长期记忆配置 `Store`

### 6.4 安全
- [ ] LangSmith anonymizer 屏蔽 PII
- [ ] 工具输入用 Pydantic 严格校验
- [ ] 高危工具配置 `interrupt_on` 审批
- [ ] 文件系统权限规则（`allow/deny/interrupt`）
- [ ] 多租户配置 custom auth + authorization handlers

### 6.5 可观测性
- [ ] 启用 LangSmith 追踪，配置项目名
- [ ] 通过 middleware 统一记录耗时与 token
- [ ] metadata 注入 user_id / session_id / request_id
- [ ] tags 标记环境与版本
- [ ] 配置 LangSmith Engine 智能监控

### 6.6 性能
- [ ] 异步优先：`ainvoke` / `astream` / 异步工具
- [ ] 流式输出用 `stream_events(version="v3")`
- [ ] 缓存策略：RedisCache 或语义缓存
- [ ] DeepAgents 子代理按任务选模型（强/快）
- [ ] 通过 `config["recursion_limit"]` 控制图执行深度（默认 25，按业务上限调）

### 6.7 调用规范
- [ ] 每次调用传 `thread_id` + `context`
- [ ] interrupt 恢复用 `Command(resume=...)`
- [ ] 中断前副作用幂等
- [ ] Python 3.11+ 异步代码无需显式传 RunnableConfig

### 6.8 部署
- [ ] `langgraph.json` 配置完整
- [ ] Docker 镜像构建测试
- [ ] 健康检查端点
- [ ] 优雅停机处理
- [ ] 资源限制（CPU / 内存 / 连接数）

---

## 附录：版本迁移关键点

### LangChain v1.0 重大变更
1. Python 3.9 支持移除（2025-10 EOL），要求 3.10+
2. `create_react_agent` 废弃，改用 `langchain.agents.create_agent`
3. `AgentExecutor` / `Agent` 类废弃，新代码用 `create_agent` + LCEL
4. `prompt` 参数改名为 `system_prompt`，类型 `str | SystemMessage | None`
5. `pre_model_hook` / `post_model_hook` 废弃，迁移到 middleware
6. `state_schema` 仅支持 `TypedDict`，Pydantic/dataclass 不再支持
7. 预绑定模型不再支持（`model.bind_tools()`），改用 `create_agent(model, tools)`
8. 流式节点名从 `"agent"` 改为 `"model"`
9. 运行时上下文通过 `context` 参数注入，不再用 `config["configurable"]`
10. 旧代码迁移到 `langchain-classic` 包

### LangGraph v1.0 重大变更
1. `create_react_agent` 废弃，使用 LangChain `create_agent`
2. Typed interrupts：通过 `interrupts` 配置在图构造时严格定义中断类型
3. `toLangGraphEventStream` 移除，使用 `graph.stream` 配合 `encoding`
4. Node.js 18 支持移除，要求 22+
5. Durable execution 默认启用
6. HITL 一等支持：原生 `interrupt()` API

### DeepAgents 0.6.x 关键变更
1. `model=None` 默认值已废弃（0.5.3 起），1.0 强制显式指定
2. 中间件栈重组：base + user + tail 三层结构
3. `_REQUIRED_MIDDLEWARE`：`FilesystemMiddleware` 与 `SubAgentMiddleware` 不可排除
4. 异步子代理 `AsyncSubAgent` 支持
5. `permissions` 支持 `interrupt` 模式自动启用 `HumanInTheLoopMiddleware`

---

**参考文档：**
- [LangChain Models](https://docs.langchain.com/oss/python/langchain/models)
- [LangChain Structured Output](https://docs.langchain.com/oss/python/langchain/structured-output)
- [LangChain v1 迁移指南](https://docs.langchain.com/oss/python/migrate/langchain-v1)
- [LangGraph Persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
- [LangGraph Streaming](https://docs.langchain.com/oss/python/langgraph/streaming)
- [LangGraph Interrupts](https://docs.langchain.com/oss/python/langgraph/interrupts)
- [DeepAgents Customization](https://docs.langchain.com/oss/python/deepagents/customization)
- [DeepAgents Subagents](https://docs.langchain.com/oss/python/deepagents/subagents)
- [DeepAgents Going to Production](https://docs.langchain.com/oss/python/deepagents/going-to-production)
- [DeepAgents GitHub](https://github.com/langchain-ai/deepagents)
