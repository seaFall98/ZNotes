**RubricMiddleware**是LangChain在2026年5月28日发布的`deepagents`库0.6.5版本中引入的一个新组件。它是一个为Deep Agents设计的**自评估与迭代中间件**，核心作用是让AI代理能够根据你预设的标准清单（即"评分准则"），自己检查工作成果，并反复修改，直到达标为止。
https://www.langchain.com/blog/introducing-rubrics-for-deepagents
https://blog.csdn.net/qq_50655286/article/details/162522385

---

在RAG Agent场景，可以用Rubric做在线答案校验和自修正，而Ragas用于离线评估和系统优化，两者属于互补关系

---

### 🎯 核心价值：解决"差最后一步"的问题

在复杂任务中，AI代理经常会产生方向正确但不够完善的结果。`RubricMiddleware`相当于在任务链路中嵌入了一位**自动质检员**，让代理明白什么才算真正的"完成"，而不仅仅是生成一个看起来差不多的答案。它尤其适用于那些拥有**清晰、可验证成功标准**的任务，例如：
*   代码重构：需要**所有测试用例通过**。
*   报告生成：必须**覆盖所有要求的章节**。
*   创意写作（如俳句）：需符合**特定的音节模式**。

### ⚙️ 工作原理：一个"生成-评估-修改"的闭环

它的工作流程可以概括为以下几个步骤：

1.  **设定标准**：你在调用代理时，通过`rubric`参数传入一个文本清单，定义"完成"的标准。
2.  **初次生成**：Deep Agent像往常一样，根据你的指令生成初始结果。
3.  **独立评审**：在代理即将结束任务时，`RubricMiddleware`不会直接放行，而是会调用一个**独立的"评审员"子代理（Grader Sub-agent）**。这个评审员会审查整个对话记录和结果，对照你设定的`rubric`逐项检查。
    *   **评审员专用模型**：你可以为评审员配置一个独立的模型（如成本更低的`claude-haiku-4-5`），用于和主代理的模型（如能力更强的`claude-sonnet-4-6`）解耦，优化成本与性能。
    *   **评审工具**：评审员还可以配备工具（`tools`），例如让它**实际运行一次测试套件**来验证代码，而不是凭空推理。
4.  **反馈与迭代**：如果评审员发现不达标的标准，它会生成具体的、逐条反馈（例如："一个测试未通过：`test_unhashable`。函数在处理列表等不可哈希类型时崩溃"），并将这些反馈作为消息注入对话，让主代理再次运行并修改。
5.  **循环终止**：这个"修改-评估"的循环会一直持续，直到所有标准都满足（`satisfied`），或者达到你预设的最大迭代次数上限（`max_iterations_reached`）。

### 🚀 如何配置与使用

1.  **定义中间件**：创建`RubricMiddleware`实例，配置评审员使用的模型、可选工具和最大迭代次数。

    ```python
    from deepagents import RubricMiddleware

    rubric_middleware = RubricMiddleware(
        model="anthropic:claude-haiku-4-5",  # 评审员模型
        tools=[run_test_suite],              # 评审员可用的工具
        max_iterations=3,                    # 最多迭代3次
    )
    ```

2.  **挂载到Deep Agent**：在创建Deep Agent时，将中间件添加到`middleware`列表中。

    ```python
    from deepagents import create_deep_agent

    agent = create_deep_agent(
        model="anthropic:claude-sonnet-4-6", # 主模型
        middleware=[rubric_middleware],
        # ... 其他配置
    )
    ```

3.  **调用时传入Rubric**：在调用`agent.invoke()`时，通过状态传入你的评分准则。

    ```python
    result = agent.invoke(
        {
            "messages": [HumanMessage(content="写一个查找列表中重复元素的Python函数")],
            "rubric": (
                "- 所有测试在 run_test_suite 中通过\n"
                "- 函数命名为 `find_duplicates` 并接受一个列表参数\n"
            ),
        }
    )
    ```

### 📊 状态监控与关键细节
*   **观察进度**：可以通过`on_evaluation`回调函数，或在`agent.get_state()`中检查私有状态（如`_rubric_status`、`_rubric_evaluations`）来获取每次评估的详细结果。
*   **标记合成消息**：由评审员生成的反馈消息会被标记来源为`rubric_grader`，方便你在UI或日志中识别，避免与真实用户消息混淆。
*   **Beta状态**：该功能目前仍处于Beta阶段，其API在未来可能发生变化。

