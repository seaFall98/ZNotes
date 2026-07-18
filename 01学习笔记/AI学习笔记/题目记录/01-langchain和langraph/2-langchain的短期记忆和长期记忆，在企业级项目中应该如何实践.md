
```
当我们使用Langgraph来定义一个graph图的时候，首先需要先定一个状态state 。这个state包含了graph的所有的schema 以及reducer函数，定一个state状态我们通常继承一个父类叫MessageState ，因为父类中已经帮我们定义好了一个messages的key 。这样的话在状态中就包含了一个默认的消息列表。 
 
在langgraph中，不管是长期记忆还是短期记忆，都是围绕着这个状态state的。这些记忆主要是存储state中的历史的交互数据。让大模型从反馈中学习，并且适应用户的偏好。提高大模型生成答案和意图识别的准确率。  

其中短期记忆是存储单个会话中的交互数据，我们通过设置一个checkpointer，如果短期记忆只是存储在内存中，可以采用InMemorySaver作为checkpointer。是在我之前做的项目中，我们还要保证短期记忆的数据持久化，所以我们采用的是PostgresSaver。但是在每次调用智能体或者执行graph的时候，都必须要传一个thread_id。在一次会话中，随着交付的轮次越来越多。短期记忆中的交互数据很有可能会超出上下文长度。所以在我们之前的项目中，我还做了短期记忆的摘要处理。据我所知，在LongGraph1.0之后，还可以通过中间件，专门负责短期记忆的摘要处理。  

LongGraph的长期记忆是保存所有会话的交互数据。并且一般都会直接持久化到关系型数据库中或者向量数据库中。一般需要设置一个store。我在之前的项目中用过PostgresStore。但是往往需要自己手动调用put函数把数据持久化到数据库中。可以根据需要，把长期记忆数据通过嵌入模型向量化，然后再保存到知识库中。这样就可以实现长期记忆存储的语义检索。
```



在企业级项目中实践LangChain的记忆机制，核心在于理解并分离**短期记忆（Short-term Memory）**与**长期记忆（Long-term Memory）** 的职责，并选择合适的技术方案来落地。

### 1. 记忆的职责分离

根据LangChain的官方定义，两种记忆的职责和范围有着清晰的划分。

*   **短期记忆 (Short-term Memory)**：**会话内**的上下文。它负责跟踪当前对话的连贯性，存储多轮对话历史、工具调用的中间结果等。这种记忆是**线程（Thread）作用域**的，即每个独立的对话线程拥有自己独立的短期记忆。
*   **长期记忆 (Long-term Memory)**：**跨会话**的用户或应用级知识。它负责记住用户的长期偏好、个人档案、历史事实等，并能被不同的对话线程共享和访问。这种记忆是**命名空间（Namespace）作用域**的，可以按用户、团队或业务域进行灵活隔离。

### 2. 核心实现技术栈

LangChain（特别是LangGraph）为实现这两种记忆提供了标准化的组件。

| 记忆类型     | 核心LangGraph组件 | 主要作用                                                     | 生产环境推荐方案                                             |
| :----------- | :---------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| **短期记忆** | **Checkpointer**  | 保存Graph状态的快照，用于恢复对话、实现“时光回溯”（time travel）和故障恢复。 | PostgreSQL (`PostgresSaver`)、MongoDB (`MongoDBSaver`)等持久化数据库。 |
| **长期记忆** | **Store**         | 存储和检索应用自定义的键值对数据，用于保存用户偏好、事实等跨线程信息。 | 可根据需求选择Redis、向量数据库（如Milvus）或关系型数据库。  |

> **注意**：`InMemorySaver` 和 `MemorySaver` 仅用于开发测试，**切忌在生产环境使用**，因为服务重启会导致所有记忆丢失。

### 3. 企业级实践架构与步骤

一个典型的企业级记忆架构如下图所示：

```mermaid
graph TD
    subgraph User [用户]
        A[用户A] --> B[对话线程 1]
        A --> C[对话线程 2]
    end

    subgraph LangGraph Application [LangGraph 应用]
        B --> D["短期记忆 (Checkpointer)"]
        C --> D
        D --> E[LLM 模型]
        
        F["长期记忆 (Store)"] --> E
    end

    subgraph Storage [存储层]
        D --> G[(PostgreSQL)]
        F --> H[(Redis/向量库)]
    end
```



**实践步骤：**

1. **短期记忆 (Checkpointer) 实现**

   * 在编译Graph时，注入一个持久化的`checkpointer`。

   * 每次调用时，通过`config`指定唯一的`thread_id`，即可实现对话历史的自动保存与恢复。

   * **示例代码（PostgreSQL）**:

     ```python
     from langgraph.checkpoint.postgres import PostgresSaver
     
     DB_URI = "postgresql://user:pass@localhost:5432/db"
     with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
         # 首次使用需执行 checkpointer.setup()
         graph = builder.compile(checkpointer=checkpointer)
         # 通过 thread_id 区分不同会话
         config = {"configurable": {"thread_id": "user_123_session_456"}}
         graph.invoke({"messages": [...]}, config)
     ```

2. **长期记忆 (Store) 实现**

   * 在Graph节点或应用代码中，通过`Store`接口读写数据。

   * 使用**命名空间（Namespace）** 来组织数据，例如按`user_id`隔离。

   * **示例概念**:

     ```python
     # 在节点函数中
     def save_user_preference(state, store: BaseStore):
         user_id = state["user_id"]
         # 在 "user_prefs" 命名空间下，以 user_id 为键存储
         store.put(("user_prefs", user_id), "preferred_language", "Chinese")
     
     def load_user_preference(state, store: BaseStore):
         user_id = state["user_id"]
         pref = store.get(("user_prefs", user_id), "preferred_language")
         # 将偏好注入到提示词或状态中
     ```

3. **记忆的读取与写入循环**

   *   **写入（记忆形成）**：在一次会话结束后，可以通过分析对话内容，提取关键信息（如用户偏好、新学到的知识），然后写入长期记忆。
   *   **读取（记忆召回）**：在新会话开始时，根据`user_id`等标识从长期记忆中检索相关信息，并将其作为系统提示词或上下文的一部分，注入到短期记忆中，从而影响模型行为。

### 4. 关键挑战与应对策略

*   **短期记忆的上下文窗口限制**：长对话会导致Token超限、成本增加和性能下降。
    *   **解决方案**：实施**记忆修剪（Trimming）** 或**总结（Summarization）**，只保留最近N轮对话或生成历史摘要。
*   **长期记忆的检索效率与准确性**：随着数据量增长，如何快速准确地找到相关记忆是挑战。
    *   **解决方案**：采用**混合检索策略**（如BM25+向量检索）；利用**命名空间**进行数据分片，缩小检索范围。
*   **多租户与数据隔离**：企业级应用必须严格隔离不同用户或团队的数据。
    *   **解决方案**：利用长期记忆的**命名空间（Namespace）** 机制实现数据隔离。例如，使用`(<TenantID>, <UserID>)`作为命名空间。
*   **记忆的准确性与“遗忘”**：如何确保存储的记忆是准确的，以及如何处理过时或错误的记忆。
    *   **解决方案**：建立**置信度**机制，对记忆进行评分；设计**记忆失效或更新**策略，例如基于时间的衰减或用户明确的修正指令。

### 5. 总结

在LangChain企业级实践中，应将**短期记忆**视为**工作内存**，通过`Checkpointer`实现会话状态的持久化；将**长期记忆**视为**知识库**，通过`Store`实现用户画像和事实的跨会话共享。两者协同工作，是构建连贯、个性化、可扩展的企业级AI应用的基础。



## 2 追问 多用户的情况呢？

在多用户（尤其是企业级多租户）场景下，核心挑战从“如何存”转变为**“如何安全隔离与高效路由”**。以下是针对短期记忆和长期记忆的具体实践策略：

```mermaid
flowchart TD
    subgraph Client["客户端层"]
        U1["用户A"] --> T1["请求线程 thread_001"]
        U2["用户B"] --> T2["请求线程 thread_002"]
        U2 --> T3["请求线程 thread_003"]
    end

    subgraph Gateway["安全拦截层 (中间件核心)"]
        T1 & T2 & T3 --> Auth["身份认证<br>提取 user_id 与 tenant_id"]
        Auth --> Mapping{"权限校验<br>thread_id 是否归属于 user_id ?"}
        Mapping -->|"校验通过"| Graph["LangGraph 执行引擎"]
        Mapping -->|"校验失败"| Reject["拒绝访问 / 抛出 403"]
    end

    subgraph LangGraph_App["LangGraph 运行时"]
        direction LR
        Graph --> ShortMem["短期记忆<br>(Checkpointer)"]
        Graph --> LongMem["长期记忆<br>(Store 接口)"]
    end

    subgraph Storage["持久化与物理隔离层"]
        ShortMem --> PG[("PostgreSQL<br>按 tenant_id 列表分区<br>字段含 user_id, thread_id")]
        LongMem --> KV[("Redis / 向量库<br>Key/命名空间结构<br>{tenant}:{user}:* ")]
    end

    style Reject fill:#ffcccc,stroke:#cc0000,stroke-width:2px
    style Mapping fill:#fff3cd,stroke:#ffc107,stroke-width:2px
```



### 1. 短期记忆（Checkpointer）：绝对的服务端绑定

短期记忆基于 `thread_id`，但**绝不能信任客户端传入的 `thread_id`**，否则用户A可通过伪造ID窃取用户B的对话历史。

**实践原则：服务端映射（Server-side Mapping）**

- **方案一（推荐）：ID编码**。在服务端生成 `thread_id` 时，强制包含 `user_id` 或 `tenant_id`，例如格式为 `{tenant_id}:{user_id}:{uuid}`。后端在处理请求时，解析并校验该前缀是否与当前认证用户匹配。
- **方案二（高安全）：独立映射表**。在业务数据库中建立 `thread_id` 到 `user_id` 的绑定关系表。每次访问 Checkpointer 前，中间件先查询该表进行**权限校验（Authorization）**，确认该线程归属当前用户后再放行。

**代码落地示例（中间件校验逻辑）：**

```python
# 伪代码：在调用 graph.invoke 之前执行
def validate_and_get_config(request_user_id: str, requested_thread_id: str):
    # 1. 如果 thread_id 未编码，查数据库映射
    owner = db.query("SELECT user_id FROM sessions WHERE thread_id = ?", requested_thread_id)
    if owner != request_user_id:
        raise PermissionError("无权访问该会话")
    
    # 2. 构造配置，确保 checkpoint 存入正确线程
    return {"configurable": {"thread_id": requested_thread_id}}
```

### 2. 长期记忆（Store）：利用命名空间实现天然隔离

LangChain 的 `Store` 接口支持**多级命名空间（Namespaces）**，这是实现多用户隔离的最佳内置特性。

**实践策略：层级化命名空间**

将命名空间设计为 `(TenantID, UserID, Category)` 的层级结构。这样既能物理隔离数据，又能灵活控制检索范围（例如：只查租户级公共记忆，或只查个人级记忆）。

```python
# 写入用户偏好
store.put(("tenant_123", "user_456", "preferences"), "language", "Chinese")
store.put(("tenant_123", "user_456", "facts"), "company_name", "TechCorp")

# 检索时，精确限定到该用户命名空间
user_prefs = store.search(("tenant_123", "user_456", "preferences")) 
```

### 3. 敏感操作的“双重写入”机制

对于涉及隐私或合规的场景（如金融、医疗），建议采用**双重写入**策略：

- **写入短期记忆**：仅保存当次会话的上下文。
- **异步抽取写入长期记忆**：在后台异步任务中，利用 LLM 对对话进行摘要或实体抽取，**经过脱敏处理（去除姓名、身份证号等PII）** 后，再存入长期记忆的向量库或关系库中。

### 4. 持久化层的性能优化（多租户数据膨胀应对）

当用户量达到万级以上时，数据库压力骤增，需提前规划：

| 组件                            | 优化策略           | 具体方案                                                     |
| :------------------------------ | :----------------- | :----------------------------------------------------------- |
| **Checkpointer (如PostgreSQL)** | **分区表**         | 按 `tenant_id` 或时间进行列表/范围分区，避免单表过大导致 checkpoint 恢复缓慢。 |
| **Store (如Redis/向量库)**      | **Key前缀与集合**  | 在 Redis 中使用 `{tenant_id}:{user_id}:*` 的 Key 结构；在向量库（如 Milvus）中使用 **Partition Key（分区键）** 按 `tenant_id` 物理隔离。 |
| **连接池**                      | **多租户连接复用** | 根据租户级别配置不同的数据库连接池参数，防止“吵闹的邻居”问题拖垮整体性能。 |

### 5. 多租户下的“冷热”记忆策略

- **短期记忆的热更新**：对于高频用户，可将最近 N 轮的 Checkpointer 状态缓存在 Redis 中，加速召回；低频用户则直接从 PostgreSQL 读取。
- **长期记忆的预热**：在用户发起首次提问时，异步预加载其长期记忆（用户画像、历史工单）并注入到 System Prompt 中，减少首轮响应的延迟。

### 总结：多用户实践清单

1. **禁止**客户端自定 `thread_id`，必须在服务端生成并绑定 `user_id`。
2. **利用** `Store` 的多级命名空间 `(租户, 用户)` 实现长期记忆隔离。
3. **加入** 权限校验中间件，拦截任何越权访问线程或记忆的行为。
4. **引入** 分区表/分区键，应对 SaaS 场景下的海量数据增长。

进一步可讨论 **基于Spring Boot或FastAPI的完整拦截器/中间件代码模板**，或者 **PostgreSQL按租户分表的DDL设计**

