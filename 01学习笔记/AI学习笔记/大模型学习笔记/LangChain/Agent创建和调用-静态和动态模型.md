下面按 **LangChain v1 的 Agent 写法**讲。核心结论先说：

> LangChain Agent 本质是：**模型 + 工具 + Prompt + Agent Harness 循环**。模型决定是否调用工具，工具返回结果后再喂回模型，直到模型给出最终答案。LangChain v1 推荐用 `create_agent()` 创建 Agent。官方文档明确把 Agent 定义为“模型循环调用工具直到任务完成”，`create_agent` 是一个可配置的 agent harness。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/agents "Agents - Docs by LangChain"))

---

# 1. LangChain Agent 的工程心智模型

从 Python AI 应用开发看，不要把 Agent 理解成“神秘智能体”，而要理解成一套 **LLM 编排运行时**：

```text
用户输入
  ↓
Agent State / messages
  ↓
模型推理：要不要调用工具？
  ├─ 不调用工具 → 返回最终答案
  └─ 调用工具 → 执行 Python 函数 / API / DB / RAG / MCP
          ↓
      工具结果写回 messages
          ↓
      再次调用模型
```

官方也把 `create_agent` 底层流程描述为：选择模型、工具、Prompt；执行时发送请求给模型，模型要么返回工具调用，要么返回最终答案；如果调用工具，就执行工具并把结果加入对话，然后重复。([LangChain](https://www.langchain.com/blog/langchain-langgraph-1dot0 "LangChain and LangGraph Agent Frameworks Reach v1.0 Milestones"))

---

# 2. Agent 创建的最小结构

LangChain v1 推荐：

```python
from langchain.agents import create_agent

agent = create_agent(
    model="openai:gpt-5.5",
    tools=tools,
    system_prompt="You are a helpful AI assistant."
)
```

官方文档里，`create_agent` 的基础参数就是：

|参数|作用|
|---|---|
|`model`|Agent 使用的模型，可以是模型字符串，也可以是初始化后的模型实例|
|`tools`|Agent 可调用的工具列表|
|`system_prompt`|约束 Agent 行为的系统提示词|
|`middleware`|高级控制点，比如动态模型选择、动态 prompt、工具错误处理、PII 防护等|

LangChain v1 文档明确说，Agent 基础配置可以直接通过 `model=`, `tools=`, `system_prompt=` 完成，更高级能力通过 middleware 扩展。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/agents "Agents - Docs by LangChain"))

---

# 3. 工具：Agent 能力边界的核心

在 AI 应用里，工具不是“附属品”，而是 Agent 接入业务系统的边界。

例如你做一个订单客服 Agent：

```python
from langchain.tools import tool

@tool
def get_order_status(order_id: str) -> str:
    """查询订单状态。输入订单号，返回订单当前物流状态。"""
    # 生产环境中，这里应调用订单服务 / 数据库 / RPC / REST API
    return f"订单 {order_id} 当前状态：已发货，预计 2 天后送达。"
```

官方推荐最简单的工具定义方式是 `@tool` 装饰器，函数的类型注解会成为工具输入 schema，docstring 会成为模型判断何时使用该工具的说明。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/tools "Tools - Docs by LangChain"))

创建 Agent：

```python
from langchain.agents import create_agent

agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_order_status],
    system_prompt="""
你是一个电商客服 Agent。
当用户询问订单状态时，必须调用 get_order_status 工具。
不要编造订单状态。
"""
)
```

调用 Agent：

```python
result = agent.invoke({
    "messages": [
        {"role": "user", "content": "帮我查一下订单 10086 到哪了？"}
    ]
})

print(result["messages"][-1].content)
```

---

# 4. 静态模型 Agent：模型在创建时固定

## 4.1 什么是静态模型？

**静态模型**就是 Agent 创建时已经固定模型，后续每次调用都用同一个模型。

```python
agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_order_status],
    system_prompt="你是一个严谨的订单客服 Agent。"
)
```

这类 Agent 的特点：

|维度|静态模型|
|---|---|
|模型选择|创建时固定|
|成本|可预测|
|行为稳定性|较好|
|适合场景|MVP、内部工具、单一任务 Agent、固定质量要求的 Agent|
|缺点|不能根据任务难度自动切换便宜/高级模型|

---

## 4.2 静态模型 Agent 完整示例

```python
from langchain.agents import create_agent
from langchain.tools import tool


@tool
def search_knowledge_base(query: str) -> str:
    """查询公司知识库。输入用户问题，返回相关知识片段。"""
    # 真实项目中，这里可以接 RAG：Embedding → Vector DB → TopK Recall → Rerank
    return f"知识库检索结果：与 '{query}' 相关的文档片段..."


@tool
def create_ticket(title: str, description: str) -> str:
    """创建工单。输入标题和描述，返回工单编号。"""
    # 真实项目中，这里调用工单系统 API
    return f"工单已创建，编号：TICKET-20260624-001"


agent = create_agent(
    model="openai:gpt-5.5",
    tools=[search_knowledge_base, create_ticket],
    system_prompt="""
你是企业内部 IT 支持 Agent。

规则：
1. 用户问制度、流程、FAQ 时，优先调用 search_knowledge_base。
2. 用户明确要求报修、创建问题、反馈故障时，调用 create_ticket。
3. 不允许编造系统状态。
4. 工具返回的信息不足时，要向用户追问。
"""
)

result = agent.invoke({
    "messages": [
        {"role": "user", "content": "VPN 连不上，帮我看看怎么处理，不行就帮我提工单。"}
    ]
})

print(result["messages"][-1].content)
```

---

# 5. 动态模型 Agent：运行时选择模型

## 5.1 什么是动态模型？

**动态模型**指 Agent 每次调用模型前，可以根据当前上下文、任务复杂度、消息数量、用户等级、成本预算等因素，动态选择不同模型。

LangChain 官方文档对动态模型选择的定义是：动态模型会在运行时基于当前 state 和 context 被选择，用于复杂路由逻辑和成本优化；实现方式是用 `@wrap_model_call` middleware 修改 request 里的 model。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/models "Models - Docs by LangChain"))

典型策略：

|场景|模型选择|
|---|---|
|简单问答|便宜模型 / mini 模型|
|多轮复杂任务|强模型|
|涉及代码生成|代码能力强的模型|
|涉及高风险工具调用|强模型 + human-in-the-loop|
|用户是免费层|便宜模型|
|用户是付费层|高级模型|
|上下文超过阈值|长上下文模型|

---

## 5.2 动态模型选择代码示例

```python
from collections.abc import Callable

from langchain.agents import create_agent
from langchain.agents.middleware import wrap_model_call, ModelRequest, ModelResponse
from langchain.chat_models import init_chat_model
from langchain.tools import tool


@tool
def search_knowledge_base(query: str) -> str:
    """查询企业知识库。"""
    return f"知识库结果：{query} 的相关文档片段..."


# 便宜模型：适合简单问答、短上下文
basic_model = init_chat_model(
    "openai:gpt-5.4-mini",
    temperature=0
)

# 强模型：适合复杂 Agent 推理、多工具调用、长上下文
advanced_model = init_chat_model(
    "openai:gpt-5.5",
    temperature=0
)


@wrap_model_call
def dynamic_model_selection(
    request: ModelRequest,
    handler: Callable[[ModelRequest], ModelResponse],
) -> ModelResponse:
    """
    根据当前会话复杂度动态选择模型。
    request.state["messages"] 是当前 Agent 的消息状态。
    """

    messages = request.state["messages"]
    message_count = len(messages)

    # 简化示例：多轮会话超过 10 条消息，用强模型
    if message_count > 10:
        selected_model = advanced_model
    else:
        selected_model = basic_model

    # override(model=...) 表示本次模型调用换成 selected_model
    return handler(request.override(model=selected_model))


agent = create_agent(
    model=basic_model,  # 默认模型
    tools=[search_knowledge_base],
    system_prompt="你是一个企业知识库助手，回答前必要时调用工具。",
    middleware=[dynamic_model_selection],
)
```

这个写法基本对应 LangChain 官方动态模型示例：初始化 basic / advanced 两个模型，然后通过 `@wrap_model_call` 在每次模型调用前根据消息数量切换模型。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/models "Models - Docs by LangChain"))

调用方式不变：

```python
result = agent.invoke({
    "messages": [
        {"role": "user", "content": "请帮我总结一下公司 VPN 故障排查流程。"}
    ]
})

print(result["messages"][-1].content)
```

重点是：**调用方不需要关心本次到底用了哪个模型**。这是 Agent Harness 内部的路由逻辑。

---

# 6. 静态模型 vs 动态模型：工程对比

|维度|静态模型 Agent|动态模型 Agent|
|---|---|---|
|模型绑定时机|Agent 创建时固定|每次模型调用前动态选择|
|实现复杂度|低|中等|
|成本控制|粗粒度|细粒度|
|质量控制|稳定但不灵活|可按任务升级模型|
|可观测性要求|一般|更高，需要记录每次路由结果|
|适合阶段|Demo / MVP / 单任务场景|生产级 Agent / 多用户 / 多任务复杂系统|
|风险|高级模型成本可能浪费|路由策略错误会导致质量波动|

---

# 7. 动态模型不是越早用越好

从 Java 后端 + AI 应用工程角度，我建议分阶段：

## 阶段 1：先用静态模型跑通闭环

先确保：

```text
用户输入 → Agent → 工具调用 → 工具结果 → 最终答案
```

能稳定跑通。

这一阶段重点不是“智能”，而是：

- 工具 schema 是否清晰；
    
- Agent 是否会正确调用工具；
    
- 工具异常是否可控；
    
- 返回结构是否适合前端展示；
    
- 日志、trace、token 消耗是否可观测。
    

## 阶段 2：再引入动态模型

当你发现以下问题时，再做动态模型：

- 简单问题用强模型太贵；
    
- 复杂问题用小模型经常翻车；
    
- 多轮对话变长后质量下降；
    
- 不同用户等级需要不同模型；
    
- 不同业务场景需要不同模型能力。
    

---

# 8. 动态模型的几种实战路由策略

## 8.1 按消息数量路由

```python
if len(request.state["messages"]) > 10:
    model = advanced_model
else:
    model = basic_model
```

适合长对话、多轮任务。

---

## 8.2 按任务类型路由

```python
last_user_message = request.state["messages"][-1].content

if "生成代码" in last_user_message or "排查 bug" in last_user_message:
    model = coding_model
elif "总结" in last_user_message:
    model = basic_model
else:
    model = advanced_model
```

适合 AI Coding、知识库助手、运维助手。

---

## 8.3 按用户等级路由

```python
user_plan = request.runtime.context.get("user_plan", "free")

if user_plan == "pro":
    model = advanced_model
else:
    model = basic_model
```

适合 SaaS 类 AI 应用。

---

## 8.4 按工具风险路由

例如涉及：

- 发邮件；
    
- 下订单；
    
- 删除数据；
    
- 修改数据库；
    
- 提交工单；
    
- 触发 CI/CD；
    
- 调用支付接口。
    

这类场景应该使用更强模型，并且配合 human-in-the-loop。LangChain v1 的 middleware 能用于 PII 处理、摘要、人工审批等场景，官方也把 middleware 作为 `create_agent` 的核心扩展点，用于动态 prompt、上下文管理、工具权限、状态管理和 guardrails。([LangChain 文档](https://docs.langchain.com/oss/python/releases/langchain-v1 "What's new in LangChain v1 - Docs by LangChain"))

---

# 9. Agent 调用方式：invoke、stream、事件流

## 9.1 普通调用：`invoke`

```python
result = agent.invoke({
    "messages": [
        {"role": "user", "content": "帮我查一下订单状态，订单号 10086"}
    ]
})

answer = result["messages"][-1].content
print(answer)
```

适合：

- 后端 HTTP API；
    
- 同步任务；
    
- 内部管理后台；
    
- 简单问答。
    

---

## 9.2 流式调用：`stream`

面向前端聊天 UI，建议用流式输出。

```python
for chunk in agent.stream({
    "messages": [
        {"role": "user", "content": "请分析一下这个故障的可能原因。"}
    ]
}):
    print(chunk)
```

LangChain 模型层支持 `invoke`、`stream`、`batch` 等调用方式，官方文档说明 `stream()` 会逐步返回输出 chunk，适合长文本提升用户体验。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/models "Models - Docs by LangChain"))

---

# 10. 和普通 Chain 的区别

很多人学 LangChain 时容易混淆：

```text
Chain = 你规定流程
Agent = 模型决定下一步
```

|类型|控制权|适合场景|
|---|---|---|
|Chain|程序员控制流程|固定流程：RAG、分类、抽取、摘要|
|Agent|模型参与决策|多工具、多步骤、不确定流程|
|LangGraph|显式状态机 / 图编排|复杂生产级 Agent、多节点工作流|

如果你的任务是：

```text
用户问题 → 检索知识库 → 生成答案
```

不一定需要 Agent，普通 RAG Chain 就够。

如果你的任务是：

```text
用户说 VPN 连不上
Agent 自己判断：
1. 是否查知识库
2. 是否调用诊断工具
3. 是否创建工单
4. 是否追问用户系统版本
```

这才更适合 Agent。

---

# 11. Python AI 应用里的推荐分层

做生产级 Agent，不建议把所有代码写在一个 Python 文件里。

推荐结构：

```text
app/
  agents/
    support_agent.py          # create_agent 工厂
    model_router.py           # 动态模型选择 middleware
    prompts.py                # system_prompt
  tools/
    kb_tool.py                # RAG / 知识库工具
    ticket_tool.py            # 工单工具
    order_tool.py             # 订单工具
  services/
    rag_service.py
    ticket_service.py
    order_service.py
  api/
    chat_api.py               # FastAPI / Flask 接口
  observability/
    tracing.py
```

核心原则：

```text
Agent 不直接写业务逻辑
Tool 负责把业务能力暴露给 Agent
Service 负责真正调用 DB / API / MQ / RAG
Middleware 负责控制模型、上下文、权限、错误处理
```

---

# 12. 面试/简历表达

可以这样讲：

> 我在 LangChain Agent 里不会只把它当成一个聊天机器人，而是把它理解为一个 Agent Harness：模型负责决策，Tools 负责连接业务系统，State 维护短期上下文，Middleware 负责模型路由、上下文压缩、权限控制和错误处理。简单场景我会使用静态模型保证稳定性；生产场景会引入动态模型路由，比如根据消息长度、任务复杂度、用户等级、工具风险选择不同模型，从而在成本和质量之间做工程权衡。

关键词：

```text
create_agent
tools
system_prompt
agent loop
middleware
wrap_model_call
dynamic model selection
state
tool calling
streaming
guardrails
cost optimization
```

---

# 13. 最小结论

- **静态模型 Agent**：`create_agent(model=固定模型, tools=..., system_prompt=...)`，适合先跑通 MVP。
    
- **动态模型 Agent**：通过 `@wrap_model_call` middleware 在运行时替换 `request.model`，适合生产级成本优化和复杂任务路由。
    
- **Agent 的核心不是 Prompt，而是 Harness**：模型、工具、上下文、状态、错误处理、权限、安全、观测共同组成可运行的 AI 应用系统。
    
- 对你这种 Java 后端转 AI 应用开发的路线，重点不是背 LangChain API，而是理解：**LangChain Agent = Python 版的 LLM 编排层 / Agent Harness / 工具调用运行时**。