
## 先给结论

**LangGraph Workflow 的核心不是“画图”，而是把 LLM 应用从一段不可控的 agent loop，改造成一个显式、有状态、可恢复、可观测的图执行系统。**

你前面说的“像决策树、组合模式”，方向是对的。但更完整的抽象应该是：

```text
LangGraph Workflow
= 有向图编排
+ 状态机流转
+ 决策树式路由
+ 组合模式复用
+ LLM / Tool / Agent 节点
+ Checkpoint / Memory / Human-in-the-loop / Observability
```

官方对 workflow 和 agent 的区分很清楚：**workflow 是按预定代码路径和顺序运行的系统；agent 是动态决定自身流程和工具使用的系统**。LangGraph 同时支持两者，并提供 persistence、streaming、debugging、deployment 等能力。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/workflows-agents "Workflows and agents - Docs by LangChain"))

---

# 1. LangGraph Workflow 到底是什么？

LangGraph 官方把自己定位为：

> **用于构建、管理和部署长运行、有状态 Agent 的低层编排框架和运行时。**

官方文档也明确说：LangGraph 不是必须依赖 LangChain；LangChain 更偏模型、工具、agent loop 抽象，LangGraph 更偏 **durable execution、streaming、human-in-the-loop、persistence** 这类底层编排能力。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/overview "LangGraph overview - Docs by LangChain"))

换成工程语言：

|层次|关注点|典型工具|
|---|---|---|
|Model API|调模型|OpenAI / Anthropic / DeepSeek API|
|Tool Calling|让模型调用工具|Function calling / Tools|
|Agent Loop|模型决定下一步行动|LangChain Agent / OpenAI Agents SDK|
|Workflow Runtime|控制状态、分支、循环、恢复|**LangGraph**|
|Observability / Eval|追踪、评估、优化|LangSmith / 自建 Eval|

所以 LangGraph Workflow 不是简单的“把几个函数连起来”，而是一个 **LLM 应用的控制流运行时**。

---

# 2. 一张总览图：LangGraph Workflow 的知识地图

```mermaid
mindmap
  root((LangGraph Workflow))
    核心定位
      Agent Orchestration Runtime
      Stateful Workflow
      Durable Execution
      Human-in-the-loop
    图模型
      State
      Node
      Edge
      Conditional Edge
      START/END
    执行机制
      Message Passing
      Super-step
      Parallel Branch
      Loop
      Reducer
    工程能力
      Checkpointer
      Store
      Streaming
      Interrupt
      LangSmith Tracing
    设计模式类比
      决策树
      状态机
      责任链
      策略模式
      组合模式
    典型场景
      Multi-step RAG
      Tool Agent
      Multi-agent
      审批流
      长任务执行
```

这张图可以作为整体入口：**LangGraph = 图结构 + 状态管理 + Agent 编排 + 生产运行能力**。

---

# 3. LangGraph 的三个核心对象：State、Node、Edge

官方 Graph API 里把核心概念讲得很直接：

1. **State**：应用当前快照，共享数据结构；
    
2. **Node**：执行逻辑的函数，读取当前 state，执行计算或副作用，然后返回 state 更新；
    
3. **Edge**：决定下一个执行哪个 node，可以是固定流转，也可以是条件分支。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/graph-api "Graph API overview - Docs by LangChain"))
    

可以这样理解：

```mermaid
flowchart TD
    A[State<br/>当前上下文快照] --> B[Node<br/>执行一步任务]
    B --> C[Partial State Update<br/>返回局部状态更新]
    C --> D[Reducer / State Merge<br/>合并状态]
    D --> E[Edge / Router<br/>决定下一个节点]
    E --> F[Next Node]
```

## 3.1 State：不是普通参数，而是“工作流内存”

在普通 Python 函数里，你可能这样写：

```python
result = step3(step2(step1(input)))
```

但复杂 LLM 应用不适合只靠函数入参和返回值，因为中间过程会有：

```text
用户消息
检索结果
工具调用结果
模型草稿
安全检查结果
人工审批结果
重试次数
错误信息
当前执行阶段
```

LangGraph 用 **State** 统一承载这些数据。

官方说 State 通常用 `TypedDict`、`dataclass` 或 Pydantic model 定义；所有 node 都会向 State 发出更新，并通过 reducer 规则合并。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/graph-api "Graph API overview - Docs by LangChain"))

示例：

```python
from typing import TypedDict, Annotated
from operator import add

class ResearchState(TypedDict):
    question: str
    documents: list[str]
    draft: str
    final_answer: str
    retry_count: int
```

更复杂时，列表字段可能需要 reducer：

```python
class ResearchState(TypedDict):
    question: str
    # 多个节点可能同时追加 documents，用 add 合并，而不是覆盖
    documents: Annotated[list[str], add]
    draft: str
```

**State 是 LangGraph 和普通 chain 的关键区别之一。**

---

## 3.2 Node：不是一定等于 Agent

Node 本质是一个函数。

官方明确说：**Nodes and Edges are nothing more than functions**；node 可以包含 LLM，也可以只是普通代码。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/graph-api "Graph API overview - Docs by LangChain"))

所以 Node 可以是：

```text
普通 Python 函数
一次 LLM 调用
一次工具调用
一个 router
一个 evaluator
一个 human approval 节点
一个完整 Agent
一个子图 Subgraph
```

示例：

```python
def retrieve_node(state: ResearchState):
    docs = search_docs(state["question"])
    return {"documents": docs}


def generate_node(state: ResearchState):
    answer = llm.invoke(
        f"基于这些资料回答问题：{state['documents']}\n问题：{state['question']}"
    )
    return {"draft": answer.content}
```

注意：**Node 不是 Agent 的同义词。**

更严谨的关系是：

```text
Node 可以封装 Agent；
Agent 也可以由多个 Node 构成；
整个 Graph 也可以被视为一个有状态 Agent Runtime。
```

---

## 3.3 Edge：普通流程边 + 条件路由边

Edge 决定下一步去哪。

固定边：

```python
builder.add_edge("retrieve", "generate")
```

条件边：

```python
def route_after_eval(state: ResearchState):
    if state["retry_count"] >= 3:
        return "final"
    if not state["draft"]:
        return "retrieve"
    return "review"

builder.add_conditional_edges(
    "evaluate",
    route_after_eval,
    {
        "retrieve": "retrieve",
        "review": "review",
        "final": "final",
    }
)
```

这就是你说的“决策树感”的来源。

---

# 4. 为什么说它像决策树？

因为 LangGraph 有大量这样的模式：

```mermaid
flowchart TD
    A[用户输入] --> B[意图识别 Router]
    B -->|需要检索| C[RAG 检索]
    B -->|需要工具| D[Tool Calling]
    B -->|可直接回答| E[LLM 直接回答]
    C --> F[生成答案]
    D --> F
    E --> G[结束]
    F --> H[质量检查]
    H -->|通过| G
    H -->|不通过| C
```

这确实像决策树：

|决策树|LangGraph|
|---|---|
|判断条件|router / conditional edge|
|分支路径|edge|
|中间节点|node|
|叶子节点|END|
|输入样本|state|
|决策结果|下一个 node|

但 LangGraph **不是普通决策树**，因为普通决策树通常是单向、无环、从根到叶子；LangGraph 是有向图，可以循环、并发、暂停、恢复、持久化。

更准确地说：

> **LangGraph 是带状态的有向图；决策树只是它的一种局部形态。**

---

# 5. 为什么说它像状态机？

状态机关注的是：

```text
当前状态是什么？
收到什么事件？
迁移到哪个状态？
执行什么动作？
```

LangGraph 也类似：

```mermaid
stateDiagram-v2
    [*] --> ReceivedQuestion
    ReceivedQuestion --> Retrieving: need_context
    ReceivedQuestion --> DirectAnswer: simple_question
    Retrieving --> Generating
    Generating --> Evaluating
    Evaluating --> Retrieving: evidence_not_enough
    Evaluating --> HumanReview: high_risk
    Evaluating --> FinalAnswer: pass
    HumanReview --> FinalAnswer: approved
    HumanReview --> Generating: rejected
    FinalAnswer --> [*]
```

对应关系：

|状态机概念|LangGraph 对应|
|---|---|
|State|Graph State|
|Transition|Edge|
|Guard condition|Conditional edge / router|
|Action|Node|
|Terminal state|END|

这也是 LangGraph 比简单 agent loop 更工程化的地方：  
**它把“模型下一步想干什么”变成了一个可观察、可控制、可恢复的状态迁移过程。**

---

# 6. 为什么说它像组合模式？

组合模式的核心是：

> 单个对象和组合对象可以被统一对待。

LangGraph 的 Subgraph 很接近这个思想。官方文档明确说：**subgraph 是作为另一个 graph 的 node 使用的 graph**，适合 multi-agent、复用一组节点、让不同团队独立开发不同部分，只要输入输出 schema 对齐即可。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/use-subgraphs "Subgraphs - Docs by LangChain"))

可以这样画：

```mermaid
flowchart TD
    A[Parent Graph] --> B[Router Node]
    B --> C[Search Node]
    B --> D[Writer Subgraph]
    B --> E[Review Subgraph]

    subgraph D[Writer Subgraph]
      D1[Plan Node] --> D2[Draft Node] --> D3[Self Check Node]
    end

    subgraph E[Review Subgraph]
      E1[Policy Check] --> E2[Human Approval] --> E3[Revision Advice]
    end
```

这就是典型组合模式：

|组合模式|LangGraph|
|---|---|
|Leaf|普通 Node|
|Composite|Graph / Subgraph|
|Component 接口|invoke / stream / input-output schema|
|组合对象可继续组合|Subgraph 作为 Node 加入 Parent Graph|

所以你可以把 LangGraph 理解成：

```text
Node 是叶子节点；
Subgraph 是组合节点；
Graph 是最终组合出来的可执行系统。
```

但还要记住：组合模式只解释了“结构复用”，不能解释 LangGraph 的全部。它还需要状态机解释“流程流转”，需要 checkpoint 解释“恢复能力”，需要 tracing/eval 解释“生产可观测性”。

---

# 7. LangGraph Workflow 和 Agent 的关系

官方区分是：

```text
Workflow：预定代码路径，按特定顺序运行
Agent：动态定义自己的流程和工具使用
```

([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/workflows-agents "Workflows and agents - Docs by LangChain"))

但在真实工程里，二者不是非此即彼，而是经常混合。

## 7.1 纯 Workflow

流程完全由代码控制：

```mermaid
flowchart TD
    A[输入] --> B[分类]
    B --> C[检索]
    C --> D[生成]
    D --> E[审核]
    E --> F[输出]
```

特点：

```text
稳定
可测试
可观测
适合生产系统
模型自由度低
```

## 7.2 纯 Agent

LLM 自己决定下一步：

```mermaid
flowchart TD
    A[用户任务] --> B[LLM 思考]
    B --> C{是否调用工具}
    C -->|是| D[Tool]
    D --> B
    C -->|否| E[最终回答]
```

特点：

```text
灵活
适合开放任务
不可控性更强
更依赖模型能力和 guardrails
```

## 7.3 工程里更常见：Workflow 包住 Agent

真实生产更常见的是：

```mermaid
flowchart TD
    A[用户请求] --> B[Workflow Router]
    B --> C[规则校验]
    C --> D[Agent Node]
    D --> E[结果评估]
    E -->|通过| F[输出]
    E -->|不通过| G[降级策略 / 人工处理]
    G --> F
```

也就是：

> **外层用 Workflow 控制边界，内层用 Agent 处理不确定任务。**

这是工程实践里更稳的结构。

---

# 8. LangGraph 的执行机制：不是普通函数调用链

官方文档提到，LangGraph 底层使用类似 message passing 的方式定义程序，受到 Pregel 启发，按离散的 **super-step** 执行；并行运行的节点处于同一个 super-step，顺序执行的节点属于不同 super-step。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/graph-api "Graph API overview - Docs by LangChain"))

可以简化理解为：

```mermaid
sequenceDiagram
    participant Runtime as LangGraph Runtime
    participant N1 as Node A
    participant N2 as Node B
    participant N3 as Node C
    participant S as State

    Runtime->>N1: 发送当前 State
    N1->>S: 返回 Partial Update
    Runtime->>S: 合并 State
    Runtime->>N2: 根据 Edge 激活 Node B
    Runtime->>N3: 同一 super-step 可激活 Node C
    N2->>S: 返回 Partial Update
    N3->>S: 返回 Partial Update
    Runtime->>S: Reducer 合并多个更新
```

这里有一个非常重要的概念：**Reducer**。

当多个节点都更新同一个 state 字段时，到底是覆盖、追加、合并，必须有规则。官方 StateGraph reference 也说明，每个 state key 可以可选标注 reducer，用于聚合多个节点给同一 key 的值。([LangChain AI](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph.StateGraph.html?utm_source=chatgpt.com "StateGraph | LangGraph.js API Reference"))

例如：

```python
from typing import TypedDict, Annotated
from operator import add

class State(TypedDict):
    # 多个并发节点都返回 logs 时，用 add 追加
    logs: Annotated[list[str], add]
```

这也是为什么 LangGraph 不是“随便把函数连一连”，而是一个真正的 graph runtime。

---

# 9. Persistence：LangGraph 生产价值的核心

很多人学 LangGraph，只学到了 StateGraph、Node、Edge，但真正让它有生产价值的是 **Persistence**。

官方文档说，LangGraph persistence layer 提供：

1. **Checkpointers**：保存 thread 的 graph state checkpoints；
    
2. **Stores**：保存 graph state 外的应用级长期数据。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/persistence "Persistence - Docs by LangChain"))
    

区别：

|能力|Checkpointer|Store|
|---|---|---|
|保存什么|Graph state 快照|应用自定义数据|
|作用范围|单个 thread|跨 thread|
|典型用途|对话连续性、HITL、time travel、容错|用户偏好、事实、共享知识|
|工程类比|工作流实例状态表|业务 KV / Memory 表|

官方也明确说：生产中不要用纯内存 saver，因为进程重启会丢失 checkpoint；生产应使用数据库支持的 checkpointer，例如 PostgresSaver 或 SqliteSaver。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/persistence "Persistence - Docs by LangChain"))

## 9.1 为什么 Checkpoint 很关键？

假设一个 Agent 正在执行：

```text
1. 检索资料
2. 调用外部 API
3. 生成草稿
4. 等待人工审批
5. 审批通过后发送邮件
```

如果执行到第 4 步，用户半小时后才审批，普通函数调用链早就结束了。

LangGraph 可以：

```text
保存当前 graph state
暂停执行
等待外部输入
恢复执行
从暂停点继续跑
```

这就是 workflow runtime 的价值。

---

# 10. Human-in-the-loop：把人类审批纳入图执行

LangGraph 的 interrupts 可以在指定点暂停 graph execution，等待外部输入后继续。官方文档说明：interrupt 触发时，LangGraph 会通过 persistence layer 保存 graph state，并一直等待，直到恢复执行。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/interrupts?utm_source=chatgpt.com "Interrupts - Docs by LangChain"))

这非常适合：

```text
高风险工具调用前审批
SQL 执行前确认
邮件发送前确认
代码修改前确认
订单退款前确认
医疗/法律/金融类建议审核
```

图可以这样设计：

```mermaid
flowchart TD
    A[Agent 生成操作计划] --> B{是否高风险}
    B -->|否| C[直接执行工具]
    B -->|是| D[Interrupt<br/>等待人工审批]
    D --> E{审批结果}
    E -->|通过| C
    E -->|拒绝| F[要求 Agent 修改计划]
    F --> A
    C --> G[输出结果]
```

这就是把 LLM 的不确定性关进一个明确的工程护栏里。

---

# 11. Memory：短期记忆和长期记忆

官方 Memory 文档把 LangGraph 里的 memory 分成两类：

1. **Short-term memory**：作为 agent state 的一部分，用于多轮对话；
    
2. **Long-term memory**：存储跨 session 的用户或应用级数据。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/add-memory "Memory - Docs by LangChain"))
    

这和 Persistence 文档里的 checkpointer/store 是对应的：

```mermaid
flowchart LR
    A[Memory] --> B[Short-term Memory]
    A --> C[Long-term Memory]

    B --> D[Checkpointer<br/>thread-scoped graph state]
    C --> E[Store<br/>cross-thread durable data]

    D --> F[对话上下文<br/>当前任务状态<br/>中断恢复]
    E --> G[用户偏好<br/>长期事实<br/>共享知识]
```

工程上不要把所有东西都塞进 `messages`。更好的方式是：

```text
messages：当前对话必要上下文
state fields：当前 workflow 的结构化状态
store：跨会话、跨任务的长期记忆
vector db：外部知识库 / 文档知识
database：业务事实和交易数据
```

---

# 12. 常见 Workflow 模式

官方 workflows and agents 文档列出了常见模式，包括 prompt chaining、parallelization、routing、orchestrator-worker、evaluator-optimizer、agents 等。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/workflows-agents "Workflows and agents - Docs by LangChain"))

下面按工程价值梳理。

---

## 12.1 Prompt Chaining：顺序流水线

适合确定性流程：

```mermaid
flowchart TD
    A[用户输入] --> B[抽取结构化信息]
    B --> C[检索资料]
    C --> D[生成草稿]
    D --> E[格式化输出]
```

适合：

```text
合同摘要
简历解析
客服工单分类
文章生成流水线
RAG 问答
```

优点是可控；缺点是灵活性一般。

---

## 12.2 Routing：路由分发

适合根据输入选择不同处理路径：

```mermaid
flowchart TD
    A[用户问题] --> B[Router]
    B -->|技术问题| C[技术 Agent]
    B -->|订单问题| D[订单工具]
    B -->|闲聊| E[普通聊天]
    B -->|敏感问题| F[安全拒答/人工处理]
```

这就是最像“决策树”的模式。

工程实践中，Router 可以是：

```text
规则判断
分类模型
LLM structured output
embedding similarity
业务配置表
```

生产系统里，不要所有 router 都交给 LLM。能用规则稳定判断的，就先用规则。

---

## 12.3 Parallelization：并行处理

适合多个互不依赖的任务同时跑：

```mermaid
flowchart TD
    A[问题] --> B[并行分发]
    B --> C[检索内部知识库]
    B --> D[检索外部网页]
    B --> E[查询业务数据库]
    C --> F[结果汇总]
    D --> F
    E --> F
    F --> G[生成答案]
```

注意：并行之后要处理 state 合并问题，尤其多个节点写同一个字段时，要设计 reducer。

---

## 12.4 Orchestrator-Worker：调度者-工作者

适合复杂任务拆解：

```mermaid
flowchart TD
    A[复杂任务] --> B[Orchestrator<br/>任务拆解]
    B --> C[Worker 1]
    B --> D[Worker 2]
    B --> E[Worker 3]
    C --> F[汇总]
    D --> F
    E --> F
    F --> G[最终结果]
```

适合：

```text
Deep Research
代码审查
多文档分析
数据报告生成
多 Agent 协作
```

这里的 Worker 可以是普通 node，也可以是 agent，也可以是 subgraph。

---

## 12.5 Evaluator-Optimizer：评估-优化循环

这是很多 AI 工程系统的核心模式：

```mermaid
flowchart TD
    A[生成初稿] --> B[评估质量]
    B -->|通过| C[输出]
    B -->|不通过| D[生成改进意见]
    D --> E[重新生成]
    E --> B
```

适合：

```text
代码生成
写作优化
RAG 答案校验
SQL 生成校验
安全检查
工具调用结果核验
```

但要加停止条件：

```text
最大重试次数
token 成本上限
超时时间
人工接管条件
```

否则容易变成无限循环或高成本系统。

---

# 13. 一个完整的 Multi-step RAG Workflow 示例

典型企业 RAG 不应该只是：

```text
query -> retrieve -> answer
```

更合理的是：

```mermaid
flowchart TD
    A[用户问题] --> B[Query Rewrite]
    B --> C[Retrieve]
    C --> D[Re-rank]
    D --> E{证据是否足够}
    E -->|否| F[扩展查询 / 二次检索]
    F --> C
    E -->|是| G[Generate Answer]
    G --> H[Grounding Check]
    H -->|通过| I[Final Answer]
    H -->|不通过| J[Refine Answer]
    J --> G
```

这就是 LangGraph 很适合的地方：

|问题|LangGraph 解法|
|---|---|
|检索不够怎么办|条件边回到 retrieve|
|答案不 grounded 怎么办|evaluator-optimizer loop|
|多源检索怎么合并|parallel nodes + reducer|
|需要人工确认怎么办|interrupt|
|对话要持续怎么办|checkpointer|
|需要长期偏好怎么办|store|
|线上问题怎么排查|tracing / LangSmith|

---

# 14. 最小代码骨架：一个可循环的 RAG Workflow

下面是概念骨架，不是完整可运行项目，但结构接近真实工程。

```python
from typing import TypedDict, Annotated, Literal
from operator import add

from langgraph.graph import StateGraph, START, END


class RagState(TypedDict):
    question: str
    rewritten_query: str
    # 多轮检索结果可以累加，所以用 reducer
    documents: Annotated[list[str], add]
    draft_answer: str
    final_answer: str
    retry_count: int
    is_grounded: bool


def rewrite_query(state: RagState):
    """将用户问题改写为更适合检索的查询。"""
    # 实际工程中这里可以调用 LLM structured output
    return {"rewritten_query": state["question"]}


def retrieve_docs(state: RagState):
    """根据查询检索文档。"""
    docs = search_vector_db(state["rewritten_query"])
    return {
        "documents": docs,
        "retry_count": state.get("retry_count", 0) + 1,
    }


def generate_answer(state: RagState):
    """基于检索文档生成答案。"""
    answer = llm_generate(
        question=state["question"],
        documents=state["documents"],
    )
    return {"draft_answer": answer}


def check_grounding(state: RagState):
    """检查答案是否有足够证据支撑。"""
    passed = grounding_check(
        answer=state["draft_answer"],
        documents=state["documents"],
    )
    return {"is_grounded": passed}


def route_after_check(state: RagState) -> Literal["retrieve_docs", "finalize", "fallback"]:
    """根据评估结果决定下一步。"""
    if state["is_grounded"]:
        return "finalize"

    if state["retry_count"] >= 3:
        return "fallback"

    return "retrieve_docs"


def finalize(state: RagState):
    """输出最终答案。"""
    return {"final_answer": state["draft_answer"]}


def fallback(state: RagState):
    """检索多次仍失败时降级处理。"""
    return {
        "final_answer": "当前资料不足，无法给出可靠答案。建议补充更多上下文或转人工处理。"
    }


builder = StateGraph(RagState)

builder.add_node("rewrite_query", rewrite_query)
builder.add_node("retrieve_docs", retrieve_docs)
builder.add_node("generate_answer", generate_answer)
builder.add_node("check_grounding", check_grounding)
builder.add_node("finalize", finalize)
builder.add_node("fallback", fallback)

builder.add_edge(START, "rewrite_query")
builder.add_edge("rewrite_query", "retrieve_docs")
builder.add_edge("retrieve_docs", "generate_answer")
builder.add_edge("generate_answer", "check_grounding")

builder.add_conditional_edges(
    "check_grounding",
    route_after_check,
    {
        "retrieve_docs": "retrieve_docs",
        "finalize": "finalize",
        "fallback": "fallback",
    },
)

builder.add_edge("finalize", END)
builder.add_edge("fallback", END)

graph = builder.compile()
```

这里已经包含了：

```text
State
Node
Edge
Conditional Edge
Loop
Reducer
Fallback
```

这就是 LangGraph Workflow 的基本形态。

---

# 15. 加上 Checkpointer：从 Demo 走向工程

普通 demo：

```python
graph = builder.compile()
```

有状态、多轮、可恢复：

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

result = graph.invoke(
    {"question": "LangGraph workflow 是什么？"},
    {"configurable": {"thread_id": "user-123-session-001"}},
)
```

生产环境不要用内存 checkpointer。官方明确提示，`MemorySaver` / `InMemorySaver` 进程重启会丢失数据，生产应使用持久化 checkpointer，例如 PostgresSaver 或 SqliteSaver。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/persistence "Persistence - Docs by LangChain"))

生产形态大概是：

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://user:password@localhost:5432/langgraph"

with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)
```

---

# 16. LangGraph 和设计模式的对应关系

你说“组合模式”，非常重要。再系统一点：

|设计模式 / 架构概念|LangGraph 中的体现|说明|
|---|---|---|
|决策树|Conditional Edge|根据条件选择路径|
|状态机|State + Edge|当前状态决定下一步迁移|
|责任链|Node 顺序传递 State|一个节点处理完交给下一个|
|策略模式|Router 选择不同处理策略|搜索、工具、直接回答等|
|组合模式|Subgraph as Node|子图可以作为节点复用|
|模板方法|固定 Workflow 骨架，节点可替换|适合标准业务流程|
|Saga / 工作流引擎|长任务、补偿、恢复|适合外部工具和业务操作|
|观察者 / 日志追踪|Tracing / streaming|观察每一步行为|

可以用这张图串起来：

```mermaid
flowchart TD
    A[LangGraph Workflow] --> B[决策树<br/>Conditional Routing]
    A --> C[状态机<br/>State Transition]
    A --> D[组合模式<br/>Node / Subgraph]
    A --> E[责任链<br/>Step-by-step Processing]
    A --> F[策略模式<br/>Route to Strategy]
    A --> G[工作流引擎<br/>Checkpoint / Resume]
```

---

# 17. 工程实践：什么时候该用 LangGraph？

## 17.1 适合用 LangGraph 的场景

|场景|为什么适合|
|---|---|
|多步骤 RAG|有检索、重排、生成、评估、重试|
|Agent Tool Use|需要控制工具调用边界|
|Multi-agent|多个 agent 协作、路由、聚合|
|长任务|需要中断、恢复、失败重试|
|人工审批|需要 human-in-the-loop|
|高风险操作|需要审核、回滚、审计|
|复杂状态|中间状态多，不能只靠 messages|
|生产 Agent|需要 tracing、eval、debugging|

## 17.2 不适合上 LangGraph 的场景

|场景|更合适做法|
|---|---|
|单轮 Chatbot|直接调用模型 API|
|简单 RAG QA|普通函数 / LCEL / LangChain Runnable|
|简单工具调用|OpenAI / Anthropic tool calling|
|一次性脚本|Python 函数即可|
|前期快速验证|先写最小 agent loop|

判断标准：

```text
如果你只有 2-3 步固定流程，不需要状态恢复，不需要复杂分支，LangGraph 可能过度设计。

如果你开始出现分支、循环、并发、状态、人工审批、恢复、观测，LangGraph 就开始有价值。
```

---

# 18. 生产级 LangGraph Workflow 应该怎么设计？

## 18.1 先画状态，不要先画节点

很多人一上来就问：

```text
我要几个 node？
怎么连 edge？
```

更好的顺序是：

```text
1. 这个任务有哪些稳定状态？
2. 每个状态字段由谁写入？
3. 哪些字段需要 reducer？
4. 哪些节点只读 state？
5. 哪些节点会产生副作用？
6. 哪些路径需要 checkpoint / interrupt？
```

例如一个文档问答 workflow：

```python
class QaState(TypedDict):
    user_id: str
    question: str
    rewritten_query: str
    retrieved_docs: list[dict]
    selected_docs: list[dict]
    answer: str
    citations: list[str]
    risk_level: str
    approved: bool
    error: str | None
```

State 设计好了，Graph 通常就自然出来了。

---

## 18.2 Node 要小，不要写成 God Node

反例：

```python
def agent_node(state):
    # 里面做分类、检索、生成、评估、调用工具、写数据库、发邮件
    ...
```

这会让 LangGraph 失去意义。

更好的拆法：

```text
classify_intent
rewrite_query
retrieve_docs
rerank_docs
generate_answer
check_grounding
risk_check
human_review
finalize
```

原则：

```text
一个 Node 做一类清晰动作；
副作用节点单独拆；
高风险节点前加审批；
可失败节点加 retry / fallback；
LLM 节点尽量 structured output。
```

---

## 18.3 Router 要尽量可测试

Router 是 workflow 的“分岔口”，生产里很关键。

不要写成完全黑盒：

```python
def router(state):
    return llm.invoke("下一步应该去哪？")
```

更好的方式：

```text
优先规则判断
再结构化分类
最后才让 LLM 开放判断
```

示例：

```python
def route_by_risk(state):
    if state["risk_level"] == "high":
        return "human_review"
    if not state["retrieved_docs"]:
        return "fallback"
    return "generate_answer"
```

LLM router 也要 structured output：

```python
class RouteDecision(BaseModel):
    next_step: Literal["retrieve", "answer", "human_review", "fallback"]
    reason: str
```

---

## 18.4 所有循环都要有停止条件

Agent 系统最常见的问题之一是无限循环。

必须设计：

```text
max_retry
max_tool_calls
max_tokens
timeout
成本预算
人工接管条件
fallback 节点
```

比如：

```python
def route_after_eval(state):
    if state["quality_passed"]:
        return "final"

    if state["retry_count"] >= 3:
        return "fallback"

    return "revise"
```

---

## 18.5 工具调用要加边界

Tool node 是风险点。

生产里要关心：

```text
工具是否只读？
是否会写数据库？
是否会发邮件？
是否会删文件？
是否会调用外部支付/订单/退款接口？
是否需要人工确认？
是否需要审计日志？
```

高风险工具不要让 Agent 直接自由调用。

推荐结构：

```mermaid
flowchart TD
    A[Agent proposes tool call] --> B[Policy Check]
    B -->|safe read-only| C[Execute Tool]
    B -->|risky write| D[Human Approval]
    D -->|approve| C
    D -->|reject| E[Return to Agent]
    C --> F[Record Audit Log]
```

---

## 18.6 持久化要分层

不要把所有数据都放 checkpoint。

推荐：

|数据|放哪里|
|---|---|
|当前任务中间状态|State / Checkpointer|
|多轮对话上下文|Short-term memory / Checkpointer|
|用户偏好|Store|
|文档知识|Vector DB|
|业务数据|MySQL / PostgreSQL|
|审计日志|独立日志表 / tracing|
|大文件|Object Storage|

官方也明确区分了 checkpointer 和 store：前者保存单个 thread 的 graph state 快照，后者保存跨 thread 的应用数据。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/persistence "Persistence - Docs by LangChain"))

---

# 19. LangGraph 与普通后端工作流的关系

如果你有 Java 后端背景，可以这样理解：

```text
LangGraph 不是替代 Spring Boot；
它更像 AI 应用内部的流程编排层。
```

类比：

|Java 后端概念|LangGraph 概念|
|---|---|
|DTO / Context|State|
|Service 方法|Node|
|if/else 路由|Conditional Edge|
|Pipeline|Sequential Edges|
|状态机|StateGraph|
|Spring Batch Step|Node|
|Flowable / BPMN|Graph Workflow|
|Saga 状态表|Checkpointer|
|审批流|Interrupt / Human-in-the-loop|
|子流程|Subgraph|
|日志 / APM|LangSmith tracing|

但是区别也很明显：

```text
传统 workflow：节点通常是确定性业务代码
LangGraph workflow：节点里可能包含 LLM 这种非确定性执行单元
```

所以 LangGraph 的核心价值是：

> **用传统工程的流程控制方式，包住 LLM 的不确定性。**

---

# 20. LangGraph Workflow 的最佳心智模型

你可以分四层理解：

```mermaid
flowchart TD
    A[业务任务层<br/>客服/检索/写作/代码/审批] --> B[Workflow 层<br/>流程、分支、循环、并发]
    B --> C[Agent 能力层<br/>LLM 决策、工具调用、记忆]
    C --> D[Runtime 层<br/>State、Checkpoint、Interrupt、Streaming、Tracing]
```

每一层解决不同问题：

|层|问题|
|---|---|
|业务任务层|这个系统到底要完成什么业务目标|
|Workflow 层|任务应该按什么路径执行|
|Agent 能力层|哪些步骤需要 LLM 动态决策|
|Runtime 层|如何保存、恢复、观测、调试、部署|

很多人学 LangGraph 学混，是因为把这四层混在一起了。

---

# 21. 常见误区

## 误区 1：LangGraph = Agent 框架

不准确。

LangGraph 是 **orchestration runtime**。Agent 可以跑在 LangGraph 上，但 LangGraph 不只是 Agent。

官方文档也说，如果刚开始学 agent 或想要高层抽象，可以先用 LangChain agents；LangGraph 更关注 durable execution、streaming、human-in-the-loop 等底层编排能力。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/overview "LangGraph overview - Docs by LangChain"))

---

## 误区 2：Node = Agent

不准确。

Node 是函数，可以是普通代码、LLM 调用、工具调用、Agent、Subgraph。

---

## 误区 3：Workflow 一定比 Agent 低级

不准确。

Workflow 更可控，Agent 更灵活。

生产系统常常是：

```text
外层 Workflow 控制边界
内层 Agent 处理开放任务
关键节点加评估和人工审批
```

---

## 误区 4：用了 LangGraph 就生产级了

不准确。

LangGraph 提供机制，但生产级还需要：

```text
状态 schema 设计
错误处理
重试策略
成本控制
权限控制
审计日志
工具安全
Eval 测试集
Tracing 观测
灰度发布
版本兼容
```

框架不能替你完成工程治理。

---

# 22. 最终总结

你可以这样记：

> **LangGraph Workflow 是一个用 StateGraph 显式描述 LLM 应用执行过程的有状态图编排系统。Node 负责执行具体动作，Edge 负责控制流转，State 负责保存上下文，Reducer 负责合并更新，Checkpointer / Store 负责持久化，Interrupt 负责人类介入，Subgraph 负责组合复用。**

更工程化一点：

> **它的价值不是让 Agent 更“智能”，而是让 Agent 更可控、可恢复、可观测、可组合、可上线。**

更短一点：

```text
简单 LLM 应用：函数调用就够。
普通 RAG：chain 就够。
复杂 RAG / Agent / 多步骤任务：LangGraph 开始有价值。
需要恢复、审批、多 Agent、长任务、生产观测：LangGraph 很适合。
```

所以你前面的理解可以最终收束成这一句话：

> **LangGraph Workflow 可以看作“带状态的决策图”：它用组合模式组织 Node / Agent / Subgraph，用状态机控制流转，用 checkpoint 和 interrupt 把 LLM 的不确定执行过程纳入可恢复、可审计的工程运行框架。**