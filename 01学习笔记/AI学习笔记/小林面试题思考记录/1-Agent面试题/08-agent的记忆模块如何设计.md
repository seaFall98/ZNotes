
这个问题，后面学完项目 要专门做一个文档 全面的讲一遍设计 下面的知识都很碎


而且要讲真实的自己做了的记忆设计，肯定做不到完善，然后结合deepagent项目里的记忆设计尤其是用户偏好和几个中间件的使用、mem0这个开源的agent记忆框架、trae的experience机制、codebuddy的memory.md机制（可以聊记忆的拆分和存储粒度，太粗太细都不好，以一次关键时间、错误复盘来存，就很有价值），来进行补充，表面一个agent的记忆系统有大量细节可扩展。


复习deepagent项目的记忆系统 文档

要结合实际 这种记忆对应的实际场景，实现方案，上下文工程里的处理，而不是只记名词


---
## 小林版的  浏览要点 注意信息时效性

**思路 ! ! ! ：任务开始前读记忆-执行中用记忆-完成后更新记忆**

划分方式很多 下面这种四层我觉得还行

Agent需要记忆才能在多步任务中保持状态、跨任务积累知识。

  [原文](https://xiaolinnote.com/ai/agent/8_memory.html#%E5%9B%9B%E7%A7%8D%E8%AE%B0%E5%BF%86%E7%B1%BB%E5%9E%8B-%E4%BB%8E%E6%9C%80%E7%9F%AD%E6%9A%82%E5%88%B0%E6%9C%80%E6%8C%81%E4%B9%85) 

推荐看原文 细节太多了

记忆机制分四层（分短期记忆和长期记忆就行了，我觉得这是最简单明了的，不纠结名词，谈具体的场景和作用）：

感知记忆（当前输入的原始内容）（感觉也可以不提这个）
短期记忆（contextwindow里的对话历史 messages列表 摘要+最近N轮，就是只在这次agent生命周期里关心的）
长期记忆（存在外部数据库、语义检索召回）（包括用户偏好，持久化的短期记忆、经验和错误复盘等，就是一切可以复用的）
实体记忆（结构化提取的关键事实 ） （类似于deepagent的/memory，用户偏好，感觉可以归到长期记忆）

实际设计时要解决三个核心问题：存什么、怎么存、什么时候取出来用，根据信息类型选合适的存储方式，再搭配主动检索和按需检索两种策略使用。

1-记忆怎么存？
需要语义检索的内容，比如文档知识、对话摘要这类非结构化的文本，适合存进向量数据库，用
embedding编码后通过相似度检索。结构化的用户偏好和状态字段，比如语言偏好、项目配置这些可以精确查询的内容，更适合用关系数据库或Key-Value存储，查询速度快，不需要语义理解。整段文档或知识库则适合存进向量数据库，配合RAG流程做召回。

混合存储是主流做法：结构化的偏好字段用关系数据库精确查，非结构化的知识和历史用向量数据库语义检索，两者配合使用。在deepagent里，用户偏好可以直接用md存和查，md存到mongodb里

用户偏好的存储时机，after_agent 触发中间件，进行异步更新和存储

2-记忆怎么用？任务开始前，把长期记忆里的历史聊天、用户偏好、系统提示词、用户问题、rag召回结果，进行拼接，输入给llm。任务过程中，提供检索工具，当llm判定需要某类特定知识时，通过工具调用主动获取。


记忆模块可以引入知识图谱负责实体关系比如用户关联的公司和业务，与向量数据库配合使用处理语义匹配。

3-记忆怎么更新？碎片？整合 ？这个deepagent项目里通过memory update middleware是解决了的


---

CodeBuddy有专门的项目级.memory
Trae的云端experience list
(各个agent都有类似的)

deepagent 对 中间推理过程 给了 代码解释器 ; 工具原始数据 以及上下文摘要 , 可以用内置+自定义摘要中间件(针对预期会产生大量中间过程、数据和输出的工具调用，可以after_tool每次都出发，而不是等达到85%窗口自动触发，可以offload+summarize，提前做好上下文窗口的控制，也避免后续ReAct发生偏航，发生错误传播的现象)

---
agent记忆框架： mem0 https://github.com/mem0ai/mem0
Memo是目前社区最活跃的Agent记忆框架之一（GitHub上超过5万星），它的核心思路是把记忆管理做成一个独立的服务层。你只需要调用memory.add(）存记忆、memory.search()查记忆，底层的embedding、去重、冲突消解它全帮你做了。Memo特别适合「个性化记忆」场景，比如记住每个用户的偏好和习惯，它可以按userid做记忆隔离，不同用户的记忆互不干扰。而且它同时支持向量存储和图存储（知识图谱），在需要关系推理的场景也能用。

---

# 详细 扫一眼即可

粗略看看

说法完全不统一 感知记忆 短期记忆 工作记忆 长期记忆(实体记忆)

## 一、Agent记忆系统的通用设计

### 1.1 为什么记忆是Agent的核心瓶颈

大模型本身是“过目就忘”的——每次API调用都是一张白纸。当Agent从单轮问答走向需要自主规划和执行长流程任务的场景时，记忆已经从“效率优化工具”升级为决定系统可靠性的关键能力。若Agent无法准确记录自身状态、任务进度和环境反馈，就容易出现重复执行、步骤跳转错误或连续任务中断。

从工程视角看，单次session内的交互已包含大量上下文信息（用户输入、工具调用结果、静态/动态知识、模型推理过程等），扩展到跨session、跨用户、多Agent协同乃至多应用协同时，系统需要处理的上下文关系会急剧增加。因此，Agent需要一个专门的**记忆增强层**，用于屏蔽复杂的状态管理、知识组织与上下文调度问题。

### 1.2 记忆系统的核心能力链路

一个完整的记忆系统需要具备五类核心能力：

| 能力     | 说明                                                                       |
| ------ | ------------------------------------------------------------------------ |
| **抽取** | 识别“什么信息应该被记住”——从对话、工具调用和企业文档中提取关键事实、偏好、约束和知识片段 --不需要的就offload summarize  |
| **组织** | 对时间、版本与关系进行建模，因为记忆会随事件和时间不断演进                                            |
| **检索** | 在合适的时机调用相关记忆，辅助推理和生成                                                     |
| **更新** | 对过时、冲突或不再适用的记忆进行修正、替换、遗忘与隔离                                              |
| **共享** | 在多Agent、多应用和企业协同场景中平衡知识复用与数据安全                                           |

### 1.3 记忆的分层架构

当前主流的设计将记忆分为三个层次：

**短期记忆（Short-Term Memory / STM）** ：在单次对话或线程内保持上下文连贯性。包括对话历史、任务计划、工具调用结果和任务进度等。其挑战在于：长对话可能超出LLM的上下文窗口，即使模型支持长上下文，也会因“分心”于过时内容而导致性能下降和成本上升。

**中期记忆**：介于短期和长期之间，如会话级别的结构化任务状态。

**长期记忆（Long-Term Memory / LTM）** ：跨会话记住用户偏好、历史交互和领域知识。需要分类型存储（事实性记忆、程序性记忆、情景性记忆）与精准检索。

### 1.4 两类实现路径

当前记忆增强主要有两条技术路径：

- **模型内生驱动**：通过改进基座模型架构、训练目标或参数编辑方法，让模型自身具备更强的记忆能力。例如对记忆进行分层建模。
- **应用外置驱动**：在系统层构建独立的记忆增强层，统一管理模型参数外的明文记忆、模型参数内的参数记忆，以及运行时缓存中的激活记忆。通过多层次协同和多触点调度，在读取效率、写入成本、更新速度与治理能力之间取得平衡。

### 1.5 2026年的前沿方向

2026年的研究正在推动记忆系统向更自主、更智能的方向演进：

- **Agentic Memory（AgeMem）** ：将长期记忆和短期记忆管理统一集成到Agent的策略中，将记忆操作暴露为基于工具（Tool-based）的动作，让LLM Agent自主决定何时存储、检索、更新、总结或丢弃信息。
- **分层与图结构记忆**：如GAM（Hierarchical Graph-based Agentic Memory）和MAGMA（Multi-Graph based Agentic Memory），通过图结构组织记忆以支持长时推理。
- **记忆作为服务（Memory as a Service）** ：如Katra、Mem0等开源记忆基础设施，提供情景回忆、语义搜索、知识图谱和时间分析能力。
- **团队记忆**：从个人记忆走向团队资产，让项目上下文、团队经验和任务过程可继承、可复用、可治理。


## 二、基于LangChain + LangGraph + DeepAgent的Python实现

### 2.1 整体架构：LangChain记忆体系的三层结构

LangChain生态的记忆系统建立在三个层次上：

```
┌─────────────────────────────────────────────────────────┐
│                    Deep Agents                          │
│        (文件系统级记忆 + 自动持久化)                      │
├─────────────────────────────────────────────────────────┤
│                    LangGraph                            │
│     Checkpointer（短期/线程级） + Store（长期/跨线程）    │
├─────────────────────────────────────────────────────────┤
│                    LangChain                            │
│          create_agent + Memory Middleware               │
└─────────────────────────────────────────────────────────┘
```

### 2.2 LangGraph：Checkpointer + Store 双轨制

LangGraph提供两种互补的持久化系统：

| 维度 | Checkpointer（短期记忆） | Store（长期记忆） |
|------|------------------------|------------------|
| 持久化内容 | 图状态快照 | 应用定义的键值数据 |
| 作用域 | 单个线程（thread） | 跨线程 |
| 记忆类型 | 短期、线程级记忆 | 长期、跨线程记忆 |
| 用途 | 对话连续性、人机协同、时间旅行、容错 | 用户偏好、事实、共享知识 |
| 访问方式 | 通过thread_id在graph config中传递 | 从节点或应用代码读写 |

#### 短期记忆实现

短期记忆通过**Checkpointer**实现，本质是将Graph的状态持久化：

```python
from langchain.agents import create_agent
from langgraph.checkpoint.memory import InMemorySaver

# 开发环境：内存存储
agent = create_agent(
    model="openai:gpt-5.5",
    tools=[get_user_info],
    checkpointer=InMemorySaver(),  # 短期记忆
)

thread_config = {"configurable": {"thread_id": "1"}}
response = agent.invoke(
    {"messages": [{"role": "user", "content": "Hi! My name is Bob."}]},
    thread_config,
)
# 第二次调用，Agent记得用户名字
response = agent.invoke(
    {"messages": [{"role": "user", "content": "What's my name?"}]},
    thread_config,
)
# 输出: "You are Bob!"
```

**生产环境**必须使用持久化的Checkpointer：

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:postgres@localhost:5432/postgres"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    # 首次使用需调用 checkpointer.setup()
    graph = builder.compile(checkpointer=checkpointer)
```

支持的Checkpointer包括：PostgresSaver、SqliteSaver、MongoDBSaver等。

#### 长期记忆实现

长期记忆通过**Store**实现，数据以JSON文档形式组织在namespace和key下：

```python
from langchain.agents import create_agent
from langgraph.store.memory import InMemoryStore
from langgraph.store.base import IndexConfig

# 支持向量检索的Store
def embed(texts):
    return [[1.0, 2.0] for _ in texts]

store = InMemoryStore(index=IndexConfig(embed=embed, dims=2))

# 写入长期记忆
user_id = "my-user"
namespace = (user_id, "chitchat")
store.put(
    namespace,
    "user_preferences",
    {
        "rules": [
            "User likes short, direct language",
            "User only speaks English & Python",
        ],
    },
)

# 读取记忆
item = store.get(namespace, "user_preferences")

# 向量检索记忆
items = store.search(
    namespace,
    filter={"my-key": "my-value"},
    query="language preferences"
)

# 创建带长期记忆的Agent
agent = create_agent(
    model="claude-sonnet-4-6",
    tools=[],
    store=store,  # 长期记忆
)
```

工具可以通过`runtime.store`参数读写Store。

### 2.3 Deep Agents：文件系统级的一等公民记忆

Deep Agents是LangChain官方推出的“开箱即用”Agent框架，在LangGraph基础上封装了文件系统、子Agent、上下文管理和长期记忆等能力。

#### 记忆即文件

Deep Agents将记忆视为文件系统中的Markdown文件：

- **短期记忆**：Agent在任务执行期间创建的文件（计划、工具调用输出、任务进度），存在于对话/线程期间
- **长期记忆**：Agent保存到`/memories/`路径的文件，跨对话持久化

```python
from deepagents import create_deep_agent

# 创建带长期记忆的Deep Agent
agent = create_deep_agent(
    model="google_genai:gemini-3.5-flash",
    memory=["/memories/AGENTS.md"],  # 启动时加载到system prompt
    skills=["/skills/"],             # 按需加载的技能
)
```

记忆文件在启动时被加载到系统提示词中。Agent可以：
- 在对话开始时读取记忆文件到system prompt
- 在对话中按需读取
- 使用内置的`edit_file`工具更新记忆文件

#### Agent作用域记忆 vs 用户作用域记忆

Deep Agents支持两种记忆作用域：

```python
# Agent作用域：所有用户共享同一个Agent人格
# 记忆文件在 (assistant_id,) namespace下
# Agent通过与所有用户的交互积累知识、精炼方法

# 用户作用域：每个用户有独立的记忆
# 记忆文件在 (user_id,) namespace下
```

#### 可插拔的Backend系统

Deep Agents使用可插拔的Backend控制文件存储位置和访问权限：

- **StateBackend**：临时存储（对话期间）
- **StoreBackend**：持久化存储（跨对话）
- **CompositeBackend**：路由到不同的后端

```python
from deepagents import create_deep_agent, CompositeBackend, StateBackend, StoreBackend

agent = create_deep_agent(
    memory=["/memories/AGENTS.md"],
    backend=CompositeBackend(
        StateBackend(),
        {"/memories/": StoreBackend(namespace=lambda rt: (assistant_id,))}
    )
)
```

#### MemoryMiddleware

Deep Agents通过中间件（Middleware）机制加载记忆：

```python
from deepagents.middleware import createMemoryMiddleware

middleware = createMemoryMiddleware({
    # 从配置的源加载记忆内容并注入system prompt
    # 支持多个源合并
})
```

`createDeepAgent`默认预装了文件系统、摘要、子Agent和提示缓存等能力。

#### 自动上下文压缩

Deep Agents内置了上下文压缩机制，当工作记忆接近上下文窗口限制时，自动将旧消息替换为摘要或压缩表示：

- 保留所有对话历史在虚拟文件系统中，支持摘要后恢复
- 自主上下文压缩：模型在适当时机自行压缩上下文窗口

### 2.4 Managed Deep Agents：零配置的持久化记忆

LangChain的Managed Deep Agents为每个部署提供**开箱即用的持久化长期记忆**：

- Agent自动记住每个用户的偏好和跨线程/会话的上下文
- 无需手动构建持久化层
- 记忆由Context Hub支持
- 可通过`disableMemory`/`disable_memory`禁用（用于无状态Agent或外部管理持久化）

### 2.5 生产环境最佳实践

#### 存储选型

| 场景 | 推荐方案 |
|------|---------|
| 开发/测试 | `InMemorySaver` + `InMemoryStore` |
| 生产环境短期记忆 | `PostgresSaver` / `SqliteSaver` / `MongoDBSaver` |
| 生产环境长期记忆 | PostgreSQL-backed Store |
| 云原生 | AgentRun的OTSCheckpointSaver（表格存储Tablestore） |

#### 记忆治理要点

1. **防止记忆污染**：错误记忆一旦写入，会在后续检索、更新和共享环节持续传播。需要引入校验、版本管理和治理机制。
2. **控制记忆规模**：保持记忆文件精简，避免上下文过载；使用Skills（按需加载）而非全部预加载。
3. **定期清理**：长期对话中Checkpoint会不断累积，需设置保留策略定期清理。
4. **多用户隔离**：通过namespace（如`(user_id, application_context)`）实现记忆隔离。

#### 与外部记忆层集成

LangChain生态可与专业记忆层服务集成，如Mem0：

```python
# 典型集成模式：每个Agent轮次执行三个操作
# 1. 检索相关记忆（推理前）
# 2. 执行LLM调用
# 3. 存储新交互（推理后）
```

### 2.6 技术栈版本要求

- Python 3.10+（推荐3.11+）
- langgraph-checkpoint-postgres（PostgreSQL Checkpointer）
- langgraph-checkpoint-mongodb（MongoDB Checkpointer）
- deepagents>=1.9.0（支持完整的记忆和Backend功能）

---

# 图解

## 补充：Agent记忆系统Mermaid架构图

以下五张图完整覆盖从通用分层设计到LangChain生态具体实现的完整链路，所有设计均基于2026年7月的技术现状。

---

### 图1：Agent记忆系统通用分层架构（概念层）

该图展示记忆在Agent系统中的**三层存储结构**及**核心治理能力**的流转关系。

```mermaid
flowchart TB
    subgraph Application["🧠 Agent应用层"]
        User["用户交互"]
        Plan["任务规划"]
        Tool["工具调用"]
    end

    subgraph MemoryCore["📚 记忆核心层"]
        direction TB
        STM["短期记忆 STM<br/>(对话上下文/工具结果)"]
        MTM["中期记忆<br/>(任务状态/会话级)"]
        LTM["长期记忆 LTM<br/>(偏好/事实/情景/程序)"]
    end

    subgraph Capabilities["⚙️ 记忆治理能力"]
        Extract["抽取<br/>(关键事实/偏好)"]
        Organize["组织<br/>(时序/版本/图谱)"]
        Retrieve["检索<br/>(语义/时间/关联)"]
        Update["更新<br/>(修正/遗忘/隔离)"]
        Share["共享<br/>(多Agent/多应用)"]
    end

    subgraph Storage["💾 持久化存储层"]
        VectorDB["向量数据库<br/>(语义检索)"]
        RelDB["关系数据库<br/>(结构化记忆)"]
        FS["文件系统<br/>(Markdown/JSON)"]
    end

    User --> STM
    Plan --> MTM
    Tool --> LTM
    
    STM --> Extract
    MTM --> Organize
    LTM --> Retrieve
    
    Extract --> Update
    Organize --> Update
    Retrieve --> Update
    Update --> Share
    
    LTM --> VectorDB
    MTM --> RelDB
    STM & LTM --> FS
    
    style STM fill:#e1f5fe
    style MTM fill:#fff3e0
    style LTM fill:#f3e5f5
    style Extract fill:#e8f5e9
    style Organize fill:#e8f5e9
    style Retrieve fill:#e8f5e9
    style Update fill:#e8f5e9
    style Share fill:#e8f5e9
```

---

### 图2：LangGraph双轨记忆机制（Checkpointer vs Store）

LangGraph通过**两条独立管道**分别管理短期和长期记忆，这是整个LangChain生态的基石。

```mermaid
flowchart LR
    subgraph Input["用户请求"]
        ThreadID["thread_id (会话ID)"]
        UserID["user_id (用户ID)"]
    end

    subgraph LangGraph["LangGraph 记忆引擎"]
        direction TB
        
        subgraph Checkpointer["Checkpointer (短期记忆)"]
            CP1["存储: 图状态快照 (State Snapshot)"]
            CP2["作用域: 单线程 (thread)"]
            CP3["能力: 时间旅行/重放/容错"]
            CP4["后端: Postgres/Mongo/Sqlite"]
        end
        
        subgraph Store["Store (长期记忆)"]
            S1["存储: 键值JSON文档"]
            S2["作用域: 跨线程 (namespace)"]
            S3["能力: 语义向量检索 + 过滤"]
            S4["后端: PostgreSQL / InMemory"]
        end
    end

    subgraph Agent["Deep Agent 调用"]
        AgentRun["agent.invoke(config={thread_id, ...})"]
    end

    ThreadID -->|"绑定"| Checkpointer
    UserID -->|"命名空间 (user_id, ...)"| Store
    
    AgentRun -->|"自动加载状态"| Checkpointer
    AgentRun -->|"通过store参数访问"| Store
    
    Checkpointer -->|"保持连续性"| Response1["当前对话记忆"]
    Store -->|"注入System Prompt"| Response2["跨会话用户画像"]
```

---

### 图3：Deep Agent 内部记忆架构（核心）

这张图展示 `create_deep_agent` 内部的**Middleware机制**、**CompositeBackend路由**和**自动压缩**如何协同工作。

```mermaid
flowchart TB
    subgraph DeepAgent["🤖 Deep Agent 内部结构"]
        AgentAPI["create_deep_agent(memory, skills, backend)"]
        
        subgraph Middleware["中间件层 (Middleware)"]
            MemMid["MemoryMiddleware<br/>(加载 /memories/ 到system prompt)"]
            SumMid["SummaryMiddleware<br/>(自动上下文压缩)"]
            SkillMid["SkillMiddleware<br/>(按需加载技能)"]
        end
        
        subgraph Backend["可插拔后端 (CompositeBackend)"]
            StateB["StateBackend<br/>(临时文件: /, /plans/)"]
            StoreB["StoreBackend<br/>(持久化: /memories/)"]
            Router["路由规则: 前缀匹配"]
        end
        
        subgraph MemoryFS["虚拟文件系统"]
            MemFiles["/memories/AGENTS.md<br/>(启动时预加载)"]
            WorkFiles["/plan.md<br/>(任务进行中)"]
            LogFiles["/logs/output.log<br/>(工具调用记录)"]
        end
    end

    subgraph StorageLayer["外部存储"]
        PG["PostgreSQL<br/>(Checkpointer + Store)"]
        LocalFS["本地文件系统"]
    end

    AgentAPI --> MemMid
    AgentAPI --> SumMid
    AgentAPI --> SkillMid
    
    MemMid -->|"读取"| MemFiles
    MemFiles -->|"存储到"| StoreB
    WorkFiles -->|"存储到"| StateB
    LogFiles -->|"存储到"| StateB
    
    Router -->|"/memories/*"| StoreB
    Router -->|"/*"| StateB
    
    StateB -->|"持久化"| PG
    StoreB -->|"持久化"| PG
    StateB -->|"缓存"| LocalFS
    
    SumMid -.->|"触发条件: 上下文超限"| Compress["生成摘要替换旧消息"]
    MemMid -.->|"多源合并"| Merge["合并多个记忆文件内容"]
```

---

### 图4：一次完整Agent推理周期的记忆访问时序

展示从用户输入到最终回复的完整**记忆读写时序**，涵盖所有参与组件。

```mermaid
sequenceDiagram
    participant User as 用户
    participant Agent as Deep Agent
    participant Mid as MemoryMiddleware
    participant Check as Checkpointer<br/>(短期)
    participant Store as Store<br/>(长期)
    participant LLM as 大模型
    participant Tool as 工具/文件系统

    User->>Agent: 1. 发送请求 (含 thread_id, user_id)
    
    activate Agent
    Agent->>Check: 2. 加载当前线程状态快照<br/>(对话历史/任务进度)
    Check-->>Agent: 返回状态 State
    
    Agent->>Mid: 3. 触发记忆中间件
    Mid->>Store: 4. 根据 user_id 查询长期记忆
    Store-->>Mid: 返回 用户偏好/事实文档
    Mid-->>Agent: 5. 注入记忆到 System Prompt
    
    Agent->>LLM: 6. 构建完整上下文 (System+记忆+状态+用户输入)
    activate LLM
    LLM-->>Agent: 7. 返回推理结果 (可能含工具调用)
    deactivate LLM
    
    alt 需要调用工具
        Agent->>Tool: 8. 执行工具 (读写文件/API)
        Tool-->>Agent: 返回工具结果
        Agent->>Check: 9. 更新短期记忆 (写入工具结果)
        Agent->>Store: 10. 若模型判定重要，写入长期记忆
    end
    
    Agent->>Check: 11. 保存最终状态快照 (checkpoint)
    Agent-->>User: 12. 返回最终回复
    deactivate Agent
    
    Note over Check,Store: 检查点自动持久化到 PostgreSQL<br/>长期记忆通过向量检索优化
```

---

### 图5：2026年前沿趋势——Agentic Memory自主管理循环

基于2026年7月最新的 **Agentic Memory (AgeMem)** 范式，记忆操作被封装为**Tool-based动作**，由Agent自主调度。

```mermaid
stateDiagram-v2
    [*] --> 接收任务
    
    接收任务 --> 检索阶段: 开始执行
    
    检索阶段 --> 决策阶段: 获取相关记忆
    
    决策阶段 --> 工具执行: 需要外部操作
    决策阶段 --> 生成回答: 信息充足
    
    工具执行 --> 记忆评估: 工具返回结果
    
    记忆评估 --> 写入记忆: 发现新事实/偏好
    记忆评估 --> 更新记忆: 旧记忆冲突/过时
    记忆评估 --> 删除记忆: 信息无效/隐私要求
    记忆评估 --> 压缩记忆: 上下文窗口压力
    
    写入记忆 --> 决策阶段
    更新记忆 --> 决策阶段
    删除记忆 --> 决策阶段
    压缩记忆 --> 决策阶段
    
    生成回答 --> [*]
    
    note right of 检索阶段
        支持多种检索策略:
        - 语义向量检索 (Store.search)
        - 时间衰减加权
        - 图谱关联遍历 (GAM/MAGMA)
    end note
    
    note right of 记忆评估
        Agent自主决定记忆操作
        无需人工干预或硬编码规则
    end note
    
    note left of 压缩记忆
        触发条件: 
        - 消息数 > 阈值
        - Token接近窗口限制
        方式: 摘要替换 + 文件归档
    end note
```

---

### 图6：生产环境部署架构（结合存储选型）

2026年生产级Agent记忆系统的**完整技术栈部署图**。

```mermaid
flowchart TB
    subgraph Client["客户端"]
        App["应用 / 前端"]
    end

    subgraph AgentService["Agent 服务 (Python 3.11+)"]
        DeepAg["deepagents>=1.9.0"]
        LangGraph["LangGraph Core"]
        MemoryMid["MemoryMiddleware"]
        ContextComp["自动上下文压缩"]
    end

    subgraph StorageCluster["存储集群"]
        subgraph Rel["关系型存储"]
            PG_Master["PostgreSQL 主库<br/>(Checkpointer + Store)"]
            PG_Replica["PostgreSQL 只读副本<br/>(检索加速)"]
        end
        
        subgraph Vec["向量索引"]
            PgVector["pgvector 扩展<br/>(Store 语义检索)"]
        end
        
        subgraph Cache["缓存层"]
            Redis["Redis<br/>(会话级记忆缓存)"]
        end
        
        subgraph FileStorage["对象存储"]
            OSS["S3/OSS<br/>(记忆文件备份/跨域共享)"]
        end
    end

    subgraph Governance["治理与监控"]
        Audit["审计日志<br/>(记忆变更追踪)"]
        Version["版本管理<br/>(记忆回滚)"]
        Policy["访问策略<br/>(namespace隔离)"]
    end

    App -->|"HTTP/gRPC"| DeepAg
    DeepAg --> LangGraph
    LangGraph --> MemoryMid
    LangGraph --> ContextComp
    
    MemoryMid -->|"读写"| PG_Master
    MemoryMid -->|"缓存"| Redis
    PG_Master -->|"同步"| PG_Replica
    PG_Master -->|"向量索引"| PgVector
    
    LangGraph -->|"Checkpoint备份"| OSS
    Store -->|"跨Agent共享"| OSS
    
    PG_Master --> Audit
    PG_Master --> Version
    Store --> Policy
    
    style PG_Master fill:#c62828,color:#fff
    style Redis fill:#ff6f00,color:#fff
    style PgVector fill:#1565c0,color:#fff
    style MemoryMid fill:#2e7d32,color:#fff
```

---

### 关键实施建议（匹配上图）

| 图表 | 核心落地要点 |
|------|-------------|
| **图2** | 务必区分 `thread_id`（Checkpointer）和 `user_id`（Store），两者不可混用 |
| **图3** | 生产环境务必配置 `CompositeBackend`，将 `/memories/` 路由到持久化Store，避免内存溢出 |
| **图4** | 时序中的“记忆评估”环节不要硬编码，交由Deep Agent的模型自主决策（2026年最佳实践） |
| **图5** | 为实现AgeMem范式，需在工具集中暴露 `save_memory`、`update_memory`、`delete_memory` 工具 |
| **图6** | PostgreSQL必须安装 `pgvector` 扩展以支持Store的语义检索；Redis缓存层可将QPS提升3-5倍 |
