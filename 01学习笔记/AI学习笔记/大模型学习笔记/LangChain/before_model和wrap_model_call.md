在 LangChain middleware 里，**`before_model` 是“模型调用前的一个节点钩子”，`wrap_model_call` 是“包住整个模型调用过程的拦截器”**。两者都发生在每次 model call 附近，但控制能力差很多。

## 核心区别

|对比项|`before_model`|`wrap_model_call`|
|---|---|---|
|类型|node-style hook|wrap-style hook|
|运行时机|每次调用模型前运行一次|包住每次模型调用，模型调用前后都能控制|
|能否拿到模型响应|不能|能|
|能否决定是否真的调用模型|基本不能，只能改状态或跳转|可以：调用 `handler(request)`、不调用、调用多次|
|适合用途|日志、校验、裁剪消息、更新 state、提前终止|retry、fallback、缓存、动态切模型、改请求、捕获错误、改响应|
|修改 state 的方式|直接 `return dict`|返回 `ModelResponse` 或 `ExtendedModelResponse + Command`|
|控制粒度|“模型调用前”|“整个模型调用生命周期”|

LangChain 官方文档把它们分成两类：`before_model / after_model / before_agent / after_agent` 属于 node-style hooks，按执行点顺序运行；`wrap_model_call / wrap_tool_call` 属于 wrap-style hooks，会包住每一次模型或工具调用。官方也明确说 wrap-style 适合 retries、caching、transformation，并且你可以决定 handler 调用 0 次、1 次或多次。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/middleware/custom "Custom middleware - Docs by LangChain"))

## `before_model` 更像“进模型前做准备”

例如：

```python
from langchain.agents.middleware import before_model, AgentState
from langgraph.runtime import Runtime
from typing import Any

@before_model
def trim_or_log(state: AgentState, runtime: Runtime) -> dict[str, Any] | None:
    print(f"About to call model with {len(state['messages'])} messages")
    return None
```

它适合做：

```text
检查上下文长度
裁剪 messages
注入状态字段
统计调用次数
根据条件 jump_to="end"
```

LangChain 文档里的例子也展示了 `before_model` 可以在消息过多时返回一条 AIMessage 并 `jump_to: "end"`，从而提前结束 agent。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/middleware/custom "Custom middleware - Docs by LangChain"))

但它有一个关键限制：**它不直接包住实际 LLM 请求**。它运行完之后，框架继续走后续模型节点。你不能在 `before_model` 里自然地写“try 调模型，如果失败换模型再调”。

## `wrap_model_call` 更像“拦截一次 LLM 调用”

典型形态是：

```python
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from typing import Callable

@wrap_model_call
def retry_model(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    for attempt in range(3):
        try:
            return handler(request)
        except Exception:
            if attempt == 2:
                raise
```

这里的 `handler(request)` 才是真正继续执行模型调用的入口。

所以 `wrap_model_call` 可以做这些事：

```text
调用前修改 request
调用 handler(request)
拿到 response 后检查/改写
捕获异常
重试
切换备用模型
缓存命中时直接返回，不调用模型
根据 response usage 更新 state
```

官方文档的 retry 示例就是用 `wrap_model_call`，因为它需要捕获模型调用异常并多次调用 handler。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/middleware/custom "Custom middleware - Docs by LangChain"))

## 最实用的判断方式

你可以按这个规则选：

### 用 `before_model`，当你只需要“模型调用前处理 state”

比如：

```python
@before_model
def check_limit(state, runtime):
    if len(state["messages"]) > 50:
        return {"jump_to": "end"}
    return None
```

这是“调用前检查”。

### 用 `wrap_model_call`，当你需要“控制这次模型调用本身”

比如：

```python
@wrap_model_call
def use_fast_or_smart_model(request, handler):
    if len(request.state["messages"]) > 20:
        request.model = smart_model
    else:
        request.model = fast_model

    return handler(request)
```

这是“决定这次到底怎么调模型”。

## 和你前面 DeepSeek thinking/tool_choice 的问题放一起看

如果你想解决类似：

```text
DeepSeek thinking mode 不支持某些 tool_choice
需要在调用模型前删掉 tool_choice
或者 thinking mode 下切到另一个模型
或者失败后自动 fallback 到 non-thinking 模型
```

那通常应该用 **`wrap_model_call`**，不是 `before_model`。

原因是你需要操作的是 **本次 ModelRequest**，甚至可能需要捕获 400 错误后重试。这个属于 wrap-style 的典型场景：retry、fallback、request transformation。官方最佳实践也建议：node-style 用于顺序逻辑如 logging/validation，wrap-style 用于控制流如 retry/fallback/caching。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/middleware/custom "Custom middleware - Docs by LangChain"))

## 简单记忆

`before_model`：  
**“模型调用前，我先看一眼 state，要不要改 state 或提前结束。”**

`wrap_model_call`：  
**“这次模型调用交给我包办，我可以改请求、调用模型、重试、换模型、改响应，甚至不调用。”**

所以，工程上我会这样用：

```text
日志、上下文裁剪、quota 检查 → before_model
动态模型选择、移除 tool_choice、DeepSeek fallback、retry、缓存 → wrap_model_call
```