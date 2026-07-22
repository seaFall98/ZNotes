
这个问题要深入解读的话，很泛的

狭义的范式是三种 

**ReAct** 推理行动，Thought -> Action -> Observation循环，但是走一步看一步容易在全局规划上迷失方向  这是langchain的基本agent loop

**Plan-and-Execute** 能解决ReAct的短板，它把规划和执行彻底分开，先让一个LLM专门做规划，输出一个完整的步骤列表，然后由另一个LLM（或同一个模型以不同角色）逐步执行。规划可以根据每一步的执行反馈动态修改。 deepagent的write_todo开箱即用

**Reflection（反思）**  可以理解为reviewer， langgraph通过拦截模型调用前后的生命周期（如 `before_model`），让 Agent 在生成结果后自动触发一个“反思器（Critic）”进行评估，如果评估不通过，则通过返回 `Command` 指令强制流程跳转回生成节点或终止流程。


实际是混用的：用 Plan-and-Execute 做整体规划，每个步骤内部用 ReAct 来执行，关键步骤再加上 Reflection 做质量把关


还可以扩展 

MultiAgent

Agentic Workflow

---
避雷

第一个雷是设计范式不熟，ReAct是最常见的，但Plan-and-Execute（把规划和执行解耦）和Reflection（执行后加自我评估环节）也是必须说出来的，三个范式各有适用场景。

第二个雷是把Reflection当调试手段，它是正式的运行时机制，内嵌在Agent的执行流程里，代价是增加token消耗和延迟，这个取舍在面试里经常被追问。

第三个雷也是最重要的一个：以为Agent是生产环境的首选。实际上纯Agent模式在生产里用得很少，因为行为不确定、难以调试、成本容易失控。

真正的工程答案是AgenticWorkflow：整体用Workflow框住主流程保证可控(其实就是langraph呢)，在需要灵活判断的节点嵌入Agent能力。能主动说出「为什么纯Agent在生产里有局限」，是这道题拿高分的关键。