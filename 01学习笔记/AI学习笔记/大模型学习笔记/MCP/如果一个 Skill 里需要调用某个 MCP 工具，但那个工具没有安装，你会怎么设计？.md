### 如果一个 Skill 里需要调用某个 MCP 工具，但那个工具没有安装，你会怎么设计？

**回答：**

我不会把 Skill 和具体的 MCP 工具强绑定，而是让 Skill 依赖**能力（Capability）**，而不是依赖某一个具体实现。

比如一个 Code Review Skill，它需要的是"获取 GitHub PR"这个能力，而不是必须调用 GitHub MCP。

Agent 在执行 Skill 时，先检查当前环境是否具备所需能力：

- 如果 GitHub MCP 已安装，就调用 GitHub MCP。
- 如果没有 GitHub MCP，但有 GitHub REST API Tool，也可以调用 API。
- 如果都没有，可以尝试使用 `git` 命令读取本地仓库。
- 如果这些能力都不存在，再告诉用户缺少相应工具，并引导安装。

也就是说，我会把"检查能力是否存在""选择具体实现""异常降级"放到 Agent 或 Tool Registry 层，而不是写死在 Skill 里面。

整个执行流程大概是：

> Skill 提出需要的能力 → Agent 查询当前可用 Tool → 匹配最合适的实现 → 成功则执行，失败则降级或提示安装。

这样做有几个好处：

第一，**Skill 的可移植性更强**。同一个 Skill 可以运行在不同环境，不需要因为工具不同而修改 Prompt。

第二，**扩展性更好**。以后新增一个 MCP 或新的 Tool，只需要注册能力映射，Skill 不需要改。

第三，**符合依赖倒置原则**。高层的 Skill 依赖抽象能力，而不是依赖某个具体 MCP，实现了解耦。

所以在我的设计里，Skill 不应该写"调用 GitHub MCP"，而应该写"获取 PR 信息"。至于最终由 GitHub MCP、REST API 还是本地 Git 来完成，是底层能力调度层负责解决的问题。