
LangGraph 的并行节点执行，是基于 **Pregel/BSP（批量同步并行）** 模型设计的。其核心思想是将执行过程分解为多个“**超步（Superstep）**”。

在单个超步内，所有被激活的节点都会**并行执行**。当前超步的所有节点执行完毕后，它们产生的状态更新会被统一应用，然后图才会进入下一个超步。

### 📊 核心概念：Fan-out 与 Fan-in

*   **Fan-out (扇出)**：将一个任务**分解**为多个可并行处理的子任务。
*   **Fan-in (扇入)**：将多个并行子任务的执行结果**汇聚**起来。

### 🏗️ 两种实现方式

LangGraph 提供了两种实现方式：

1.  **静态 Fan-out/Fan-in（常规边）**
    这种模式在**编译图时**结构就已经确定。

    ```mermaid
    flowchart LR
        START --> A[Node A]
        A --> B[Node B]
        A --> C[Node C]
        B --> D[Node D]
        C --> D[Node D]
        D --> END
    ```

    代码实现如下:
    ```python
    import operator
    from typing import Annotated, Any
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END

    # 1. 定义状态，使用 operator.add 作为 reducer 来聚合列表
    class State(TypedDict):
        aggregate: Annotated[list, operator.add]

    # 2. 定义一个可复用的节点类，向 state 中添加值
    class ReturnNodeValue:
        def __init__(self, node_secret: str):
            self._value = node_secret

        def __call__(self, state: State) -> Any:
            print(f"Adding {self._value} to {state['aggregate']}")
            return {"aggregate": [self._value]}

    # 3. 构建图
    builder = StateGraph(State)
    builder.add_node("a", ReturnNodeValue("I'm A"))
    builder.add_node("b", ReturnNodeValue("I'm B"))
    builder.add_node("c", ReturnNodeValue("I'm C"))
    builder.add_node("d", ReturnNodeValue("I'm D"))

    builder.add_edge(START, "a")  # 入口
    builder.add_edge("a", "b")    # 从 A 扇出到 B
    builder.add_edge("a", "c")    # 从 A 扇出到 C
    builder.add_edge("b", "d")    # 从 B 和 C 扇入到 D
    builder.add_edge("c", "d")
    builder.add_edge("d", END)    # 出口

    graph = builder.compile()
    ```

    执行时，`A` 先执行，然后 `B` 和 `C` 会在同一个超步内**并行执行**，最后 `D` 执行。

2.  **动态 Fan-out（条件边 + `Send` API）**
    当并行任务的数量**在运行时才能确定**时，就需要动态 Fan-out。这通常通过条件边（`add_conditional_edges`）返回一个 `Send` 对象列表来实现。

    ```mermaid
    flowchart LR
        START --> A[Node A]
        A -->|条件边: 返回 Send 列表| B1[Task 1]
        A -->|条件边: 返回 Send 列表| B2[Task 2]
        A -->|条件边: 返回 Send 列表| B3[Task ...]
        B1 --> C[Join Node]
        B2 --> C[Join Node]
        B3 --> C[Join Node]
        C --> END
    ```

    代码实现如下:
    ```python
    from langgraph.graph import StateGraph, START, END
    from langgraph.types import Send

    class State(TypedDict):
        subjects: list
        # ... 其他字段

    # 分析问题，生成需要研究的主题列表
    def analyze_problem(state: State):
        subjects = ["quantum computing", "neural networks", "climate change"]
        return {"subjects": subjects}

    # 条件边路由函数：为每个主题创建一个并行任务
    def continue_to_research(state: State):
        # 为 subjects 列表中的每个元素，创建一个 Send 对象
        # 每个 Send 对象都会启动一个名为 "research_topic" 的节点
        return [Send("research_topic", {"topic": t}) for t in state["subjects"]]

    # 单个研究任务节点
    def research_topic(state: State):
        topic = state['topic']
        # ... 执行具体的研究逻辑
        return {"research_results": [f"Results for {topic}"]}

    # 汇聚所有并行任务结果的节点
    def aggregate_results(state: State):
        # ... 整合所有研究结果
        return {"final_report": "..."}

    builder = StateGraph(State)
    builder.add_node("analyze", analyze_problem)
    builder.add_node("research_topic", research_topic)
    builder.add_node("aggregate", aggregate_results)

    builder.add_edge(START, "analyze")
    # 从 analyze 节点动态扇出到多个 research_topic 节点
    builder.add_conditional_edges("analyze", continue_to_research)
    # 所有 research_topic 节点完成后，扇入到 aggregate 节点
    builder.add_edge("research_topic", "aggregate")
    builder.add_edge("aggregate", END)
    ```

### ⚙️ 超步（Superstep）与事务性

并行节点的执行具有**事务性**：
*   同一个超步内的所有并行节点，要么**全部成功**，要么**全部失败**。
*   如果任何一个并行节点抛出异常，整个超步的更新都会被**回滚**，状态不会发生部分改变。

### 🚀 实际案例：性能提升

通过一个实际案例可以直观地看到并行执行的性能优势。

*   **串行执行**：总耗时约 **4.97 秒**。
    ```python
    workflow.add_edge(START, "create_model")
    workflow.add_edge("create_model", "get_dsa_topics")
    workflow.add_edge("get_dsa_topics", "get_system_design_topics")
    workflow.add_edge("get_system_design_topics", "get_project_ideas")
    ```
*   **并行执行**：总耗时约 **2.29 秒**，性能提升超过 **50%**。
    ```python
    workflow.add_edge(START, "create_model")
    workflow.add_edge("create_model", "get_dsa_topics")
    workflow.add_edge("create_model", "get_system_design_topics")  # 并行
    workflow.add_edge("create_model", "get_project_ideas")         # 并行
    ```

### 💎 工程实践要点

1.  **正确使用 Reducer**：当多个并行节点可能修改状态中的**同一个键**时，必须为该键指定一个 **Reducer 函数**（如 `operator.add`）。
2.  **区分静态与动态**：任务数量和结构在编译时已知，用**静态 Fan-out**；任务数量取决于运行时的数据，用 **`Send` API**。
3.  **利用 `Command` 进行复杂路由**：`Command` 对象允许节点在返回状态更新的同时，指定下一步的路由，是实现多Agent切换等复杂逻辑的推荐方式。
4.  **优化 I/O 密集型任务**：Fan-out/Fan-in 模式非常适合优化 I/O 密集型操作，如**并发调用多个LLM API、查询不同数据库、调用多个外部服务**等。

总而言之，Fan-out/Fan-in 模式是 LangGraph 实现高性能并行计算的核心。理解超步（Superstep）和事务性的概念，并根据场景选择静态或动态的实现方式，是构建高效、可靠 Agent 应用的关键。