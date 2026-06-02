
> 从 Prompt Engineering、Context Engineering 到 Harness Engineering，看 OpenAI 与 Anthropic 正在收敛的 AI 工程路线

---

## 导语

过去一段时间，我对 AI 编程的理解发生过几次变化。

最开始，我以为关键是 **Prompt Engineering**：只要把需求说清楚，把角色、步骤、边界、输出格式写明白，AI 就能更稳定地完成任务。

后来发现，这还不够。AI 不是孤立地执行一句提示词，它需要理解项目的上下文、文档、架构、历史决策、代码规范、测试方式。于是问题转向了 **Context Engineering**：如何让 AI 在正确阶段看到正确材料，而不是把一堆文档粗暴塞进上下文窗口。

再往后，我发现即使 Prompt 写得很清楚，Context 也组织得不错，AI 仍然可能出现一些很工程化的问题：

- 它可能声称完成，但实际只是写了 mock 或 stub；
    
- 它可能改了很多文件，但没有真正跑通测试；
    
- 它可能生成漂亮的总结，但代码和文档状态并不一致；
    
- 它可能在长任务里逐渐漂移，忘记最初的目标；
    
- 它可能在多轮修复中越改越散，最后没人知道当前真实进度。
    

这时我意识到，AI Coding 真正要解决的，不只是“怎么让模型写代码”，而是：

> **如何把 AI Agent 放进一个可控、可验证、可恢复、可迭代的软件工程系统里。**

这就是我现在理解的 **Harness Engineering**。

它不是一个新的玄学词，也不只是多写几条 Prompt。它更像是：

> **给 AI Agent 制定的软件工程治理体系。**

过去我们用软件工程约束人类团队：需求、设计、开发、测试、评审、CI/CD、上线、回归、复盘。  
现在，AI Agent 进入开发流程后，这套体系没有消失，反而要被重新设计，用来约束一个新的参与者：**会写代码、会调用工具、会自主执行，但也会幻觉、漂移、误判和假完成的 Agent。**

---

## 图 1：从 Prompt 到 Context，再到 Harness

下面这张图是全文的基本框架。

![[AI Coding 工程化三层.svg]]

---

# 1. 一个反直觉现象：AI 会写代码后，工程复杂度并没有消失

在一些中型代码项目的 AI Coding 实践里，我反复观察到一个现象：

> AI 越能写代码，工程师越不能只盯着“代码有没有生成”。

因为真正的问题往往不是 AI 不会写，而是：

```text
它写了，但不一定写对；
它写对了一部分，但不一定接进系统；
它接进去了，但不一定经过验证；
它验证了一次，但不一定能持续回归；
它总结得很漂亮，但不一定和真实代码状态一致。
```

这和传统软件开发里的问题很像，但又多了一层 AI 特有的不确定性。

人类开发者也会写 bug，也会理解错需求，也会忘记补测试。  
但人类通常知道自己有没有真正跑过项目，知道哪些地方是临时方案，知道哪些东西只是占位。

AI Agent 的问题在于，它很容易把“看起来合理的中间状态”表达成“已经完成”。这不是简单的道德问题，而是 Agent 工作方式本身带来的工程风险。

比如：

- 需求说“接入真实接口”，它可能先写一个 mock service，然后总结成“接口已完成”；
    
- 需求说“修复后端问题”，它可能只改了前端展示；
    
- 需求说“运行测试”，它可能没有真实执行命令，只是根据代码推断测试应该通过；
    
- 长任务执行到后半段，它可能忘记最初定义的边界，开始扩大修改范围；
    
- 多 Agent 协作时，一个 Agent 的总结被另一个 Agent 当成事实，但总结本身可能不准确。
    

这说明 AI Coding 的核心问题已经从“代码生成能力”扩展到了“工程治理能力”。

---

## 图 2：AI Coding 常见失控链路

这张图描述了只靠 Prompt 时，AI Coding 容易出现的典型失控路径。

![[AI Coding 常见失控链路.svg]]

---

# 2. Prompt Engineering：把模糊需求变成任务协议

Prompt Engineering 不是“写几句神奇咒语”。

在 AI Coding 里，它更像是**任务表达工程**：

> 把人类模糊、跳跃、隐含约束很多的需求，转化为 AI Agent 能理解、能执行、能验收的任务协议。

它对应很多实际动作：

|实践|工程意义|
|---|---|
|Brainstorm|把模糊目标拆成多个可能方向|
|spec.md|把需求固化成任务规格|
|plan mode|让 AI 先制定执行计划，而不是直接改代码|
|acceptance criteria|定义什么叫完成|
|角色约束|明确 Agent 是架构师、实现者、评审者还是测试者|
|输出格式约束|降低沟通成本，方便后续审查|

Prompt Engineering 解决的是：

> **AI 知不知道要做什么。**

这一步很重要。没有好的任务表达，AI 只会在模糊需求里猜。

但是，Prompt Engineering 也有明显边界。

它不能保证：

- AI 看到的是最新项目上下文；
    
- AI 不会引用过期文档；
    
- AI 不会忘记历史约束；
    
- AI 一定会真实运行测试；
    
- AI 的总结一定对应真实代码状态；
    
- AI 做错后能被稳定发现和纠正。
    

所以，Prompt 是入口，但不是终点。

---

# 3. Context Engineering：把项目知识变成 Agent 可用的信息供应链

当任务稍微复杂一点，Prompt 就会遇到天花板。

因为 AI Coding 不是在空白文本上写作文，而是在一个不断演进的工程系统里修改代码。

它需要知道：

- 项目架构是什么；
    
- 代码分层是什么；
    
- 依赖怎么管理；
    
- 接口规范是什么；
    
- 数据模型怎么设计；
    
- 哪些模块不能乱改；
    
- 哪些文档是最新事实；
    
- 哪些历史决策不能推翻；
    
- 测试、构建、部署命令是什么；
    
- 当前任务进行到哪一步。
    

这些不是单靠一条 Prompt 能解决的。

这就是 Context Engineering 的位置。

我现在更愿意把 Context Engineering 理解成：

> **为 AI Agent 设计一条信息供应链。**

不是“把所有材料塞给 AI”，而是控制：

```text
它该看什么？
什么时候看？
看多少？
从哪里看？
看到的信息是否可信？
信息过期后如何淘汰？
长任务状态如何外置？
```

Anthropic 在 2025 年 9 月发布过一篇关于 AI agents 的 context engineering 文章，强调 context engineering 的关键在于为 agent 的下一步动作提供正确的信息，而不是盲目增加上下文量；Anthropic 工程页也把 effective context engineering、long-running agents、managed agents、containment 等主题放在同一条工程脉络里。([Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents "Effective context engineering for AI agents \ Anthropic"))

在实际 AI Coding 中，Context Engineering 可以落到这些做法：

- 项目文档体系；
    
- 状态外置，而不是只存在聊天记录里；
    
- 一个任务一个会话，避免上下文污染；
    
- 长任务用 progress file / task list / git history 交接；
    
- Skill 元信息索引，先判断需要加载什么能力；
    
- 按需加载，而不是全量塞入；
    
- 渐进式披露，先给摘要，再按需展开；
    
- 把仓库作为事实源，而不是把 AI 总结当事实源。
    

OpenAI 的 Codex App 资料也提到，技能可以被 check into repository，让团队共享；Codex App 还支持 Automations，用于周期性任务，如 issue triage、CI failures 汇总、release briefs 等。([OpenAI](https://openai.com/index/introducing-the-codex-app/?utm_source=chatgpt.com "Introducing the Codex app"))

这说明，Context Engineering 已经不只是“聊天上下文管理”，而是在向“项目知识组织”和“Agent 工作材料管理”演进。

但是，Context Engineering 仍然不够。

因为即使 AI 看到了正确材料，它也可能：

- 没有按材料执行；
    
- 执行后没有验证；
    
- 验证后没有回归；
    
- 回归失败后没有修复；
    
- 修复后没有留下可交接状态；
    
- 状态总结不可信。
    

这时问题进入第三层：Harness Engineering。

---

# 4. Harness Engineering：给 AI Agent 制定的软件工程

我现在对 Harness Engineering 的定义是：

> **Harness Engineering 是面向 AI Agent 的软件工程控制层。它把传统软件工程里的流程、测试、评审、CI/CD、权限、日志、回归、复盘等机制，改造成适合 AI Agent 的工作环境和约束系统。**

这句话里有两个重点。

第一，它不是凭空发明的新东西。

很多 Harness 机制，本来就是传统软件工程里的东西：

|传统软件工程|AI Coding 中的 Harness 适配|
|---|---|
|需求分析|spec.md / plan mode|
|任务拆分|batch / issue / subtask|
|编码规范|AGENTS.md / conventions|
|Code Review|AI reviewer + human reviewer|
|单元测试|防止假实现|
|CI/CD|阻断不可构建代码|
|验收标准|acceptance criteria|
|日志与监控|给 Agent 提供运行反馈|
|事故复盘|把踩坑变成回归规则|
|权限控制|sandbox / approval / egress control|

第二，它必须针对 AI 做额外适配。

因为 AI Agent 有一些人类开发者不完全一样的问题：

- 幻觉；
    
- 假完成；
    
- 长任务漂移；
    
- 上下文遗忘；
    
- 工具误用；
    
- 过度修改；
    
- 对权限风险判断不稳定；
    
- 把外部恶意内容当成正常指令；
    
- 把自己或其他 Agent 的总结误认为事实。
    

所以 Harness Engineering 的关键不是“多写规范”，而是形成闭环：

```text
规则 → 执行 → 检查 → 反馈 → 修复 → 复盘 → 回归
```

如果只是写一条规则：

> 不要假实现。

这不是 Harness。

真正的 Harness 应该是：

- spec 里明确禁止 mock 冒充真实实现；
    
- code review checklist 检查 stub / TODO / fake implementation；
    
- 测试用例覆盖真实接口；
    
- CI 自动运行；
    
- AI 完成后必须给出测试命令和结果；
    
- 如果踩坑，写入下一轮 checklist；
    
- 之后类似问题自动被拦截。
    

这才是工程系统。

---

## 图 3：Harness Engineering 闭环图

![[Harness Engineering 闭环图.svg]]

---

# 5. OpenAI 的路线：从 Codex Agent Loop 到 Symphony 和 Self-improving Agents

OpenAI 这条线，如果只写“百万行代码奇迹”，已经不够了。

更准确地看，OpenAI 的 Harness 实践正在形成一条很清楚的演进路线：

```text
Codex Agent Loop
  ↓
Codex App Server
  ↓
Agent-first Codebase / Harness Engineering
  ↓
Long-horizon Codex
  ↓
Symphony Orchestration
  ↓
Self-improving Agents
  ↓
Sandbox / Safety Boundary
```

这条路线说明：OpenAI 关注的不是“让 Codex 写更多代码”这么简单，而是把 Codex 从一个交互式 coding agent，逐步工程化成一套可嵌入、可编排、可长期运行、可从生产反馈中改进、并受 sandbox 约束的开发平台。

---

## 5.1 Codex Agent Loop：Harness 的最小内核

OpenAI 在 2026 年 1 月的 Codex agent loop 文章中，把 Codex harness 描述为所有 Codex 体验底层的核心 agent loop 和执行逻辑。这个 loop 负责组织用户、模型和工具之间的交互：模型提出工具调用，harness 执行工具，再把结果反馈给模型，直到任务完成。([OpenAI](https://openai.com/index/unrolling-the-codex-agent-loop/?utm_source=chatgpt.com "Unrolling the Codex agent loop"))

这说明 OpenAI 对 Harness 的理解不是“写一个流程文档”，而是一个真正的运行时系统。

它关心的是：

```text
用户输入如何进入模型？
模型如何选择工具？
工具调用如何执行？
执行结果如何回到上下文？
上下文如何被维护？
权限和沙箱如何参与执行？
```

也就是说，Harness 是模型之外的工程层。

模型负责推理和生成，Harness 负责组织执行环境。

---

## 5.2 App Server：把 Harness 平台化

到了 Codex App Server，OpenAI 开始把 Codex harness 从单一 CLI 体验中抽象出来。

OpenAI 的 App Server 文章提到，Codex harness 包括 thread lifecycle and persistence、config and auth、tool execution and extensions 等部分；Codex core 既是 agent code 所在的 library，也是能运行 agent loop、管理 thread persistence 的 runtime。App Server 则通过 JSON-RPC 协议把这套能力暴露给不同客户端。([OpenAI](https://openai.com/index/unlocking-the-codex-harness/?utm_source=chatgpt.com "Unlocking the Codex harness: how we built the App Server"))

这件事的工程意义很大。

它意味着 Codex 不再只是一个命令行工具，而是在变成一个可以被 IDE、桌面端、Web、其他客户端复用的 Agent 运行平台。

可以类比传统软件架构：

```text
早期：一个功能直接写在应用里
后来：抽成服务、协议、运行时、事件流、客户端适配层
```

Codex harness 也在经历类似过程。

---

## 5.3 Agent-first Codebase：仓库成为 Agent 的事实源

OpenAI 的 Harness Engineering 文章中，一个最有冲击力的案例是：他们从空仓库开始，用 Codex 生成了约百万行代码，并通过约 1500 个 PR 推进项目。更重要的不是代码量，而是他们重新设计了工程流程：工程师通过 prompt 驱动 Codex，Codex 使用标准开发工具、仓库内 skills、PR review loop 和本地/云端 agent reviewers 来迭代，直到评审满意。([OpenAI](https://openai.com/index/harness-engineering/?utm_source=chatgpt.com "Harness engineering: leveraging Codex in an agent-first ..."))

这个案例最容易被误读成：

> AI 已经能替代工程师写代码。

但我认为它真正说明的是另一件事：

> **当 AI 大规模参与开发时，仓库、文档、测试、CI、技能、日志、评审流程，都必须变成 Agent 可读、可执行、可验证的工程环境。**

这就是 Agent-first codebase。

它不是把人类工程师排除出去，而是把人类工程师的位置上移：

```text
过去：人类直接写代码
现在：人类定义目标、设计结构、设置不变量、建立反馈回路
```

---

## 5.4 Long-horizon Codex：从短任务走向长任务

AI Coding 的下一步，不是让 Agent 一次回答更长，而是让它能在更长时间跨度里持续执行、观察、修复。

OpenAI Developers 关于 long-horizon tasks 的文章提到，Codex 可以围绕任务进行计划、编辑代码、运行工具、观察结果、修复失败、更新文档和状态，并重复这个循环；这类长任务需要真实反馈、外部化状态和可持续转向能力。([OpenAI](https://openai.com/index/introducing-codex/?utm_source=chatgpt.com "Introducing Codex"))

这和日常工程经验高度一致。

长任务不是靠“超级长 Prompt”硬撑，而是靠：

- 计划；
    
- 分阶段执行；
    
- 工具反馈；
    
- 状态外置；
    
- 工作区隔离；
    
- 持续验证；
    
- 失败修复；
    
- 可交接记录。
    

这已经不是普通代码生成，而是软件工程过程的自动化。

---

## 5.5 Symphony：从人盯多个 Agent，到任务系统编排 Agent

OpenAI 在 2026 年 4 月发布了 Symphony，把 issue tracker 变成 coding agents 的控制平面。OpenAI 表示，在 agent-first repository 成功后，他们遇到的新瓶颈是 context switching：人类可以同时运行多个 Codex session，但 session 数量增加后，很快会忘记每个 session 在做什么，需要频繁切换和手动推动。Symphony 的目标是让每个 open task 都获得一个 agent，agents 持续运行，人类主要 review 结果。([OpenAI](https://openai.com/index/open-source-codex-orchestration-symphony/?utm_source=chatgpt.com "An open-source spec for Codex orchestration: Symphony."))

这一步非常关键。

它说明 AI Coding 的瓶颈正在从“AI 能不能写”转向：

> **人类如何管理多个 Agent 的并行工作。**

传统的交互式 coding agent 仍然需要人盯着：

```text
这个 Agent 卡住了吗？
那个 Agent 测试跑完了吗？
哪个任务该继续？
哪个 PR 要 review？
哪个 session 需要重启？
```

Symphony 的方向是：不要让人管理一个个聊天窗口，而是让 issue tracker / project board 成为 Agent orchestration 的控制平面。

这和传统工程里的“任务系统 + CI/CD + 自动化流水线”非常像。

---

## 5.6 Self-improving Agents：从生产反馈到自动修复闭环

OpenAI 2026 年 5 月的 self-improving tax agents 案例，把 Harness 往前推进了一步：不只是让 Codex 写代码，而是把生产 traces、专家纠错、targeted evals 和 Codex-driven improvement loop 连接起来。文章中提到，系统可以把生产中发现的问题包装成 targeted eval set，Codex 再结合 trace、eval、repo、skills 去调查根因、提出修改、运行评测和回归。([OpenAI](https://openai.com/index/building-self-improving-tax-agents-with-codex/?utm_source=chatgpt.com "Building self-improving tax agents with Codex"))

这非常重要。

因为它说明未来的 AI 工程闭环可能是这样的：

```text
生产环境出错
  ↓
专家纠错 / 用户反馈
  ↓
生成 targeted eval
  ↓
Codex 调查 trace / repo / skills
  ↓
生成修复 PR
  ↓
运行 targeted eval + regression suite
  ↓
人类 review
  ↓
进入下一轮生产
```

这不是“AI 一次性写代码”，而是“生产反馈驱动系统持续改进”。

---

## 5.7 Sandbox / Safety：Harness 必须有权限边界

OpenAI 2026 年 5 月的 Codex Windows sandbox 文章提到，Codex 运行在开发者机器上，需要操作系统强制执行的 isolation features；默认要控制文件访问和网络访问，而不是让用户在“每个命令都审批”和“完全放开权限”之间二选一。([OpenAI](https://openai.com/index/building-codex-windows-sandbox/?utm_source=chatgpt.com "Building a safe, effective sandbox to enable Codex on ..."))

OpenAI 关于安全运行 Codex 的文章也强调，sandbox 和 approval policy 是配合关系：sandbox 定义技术执行边界，包括 Codex 能写哪里、能否访问网络、哪些路径受保护；approval policy 决定什么时候需要请求用户批准。([OpenAI](https://openai.com/index/running-codex-safely/?utm_source=chatgpt.com "Running Codex safely at OpenAI"))

这说明 Harness 不只是流程和测试，也包括：

- 文件系统边界；
    
- 网络访问控制；
    
- approval policy；
    
- sandbox；
    
- protected paths；
    
- workspace 写入限制；
    
- 外部内容带来的 prompt injection 风险。
    

Agent 越强，越不能只靠“模型听话”。  
必须有操作系统和运行时级别的护栏。

---

## 图 4：OpenAI Harness 演进时间线

![[OpenAI Harness 演进线.svg]]

---

## 图 5：Codex Harness 平台化结构图

![[Codex Harness 平台化结构.svg]]

---

# 6. Anthropic 的路线：从 Long-running Agents 到 Managed Agents 和 Containment

如果说 OpenAI 的路线更像“Codex 工程平台化”，Anthropic 的路线则更像“Agent 运行时与安全治理”。

它的演进线可以概括为：

```text
Claude Code best practices / effective agents
  ↓
Effective context engineering
  ↓
Effective harnesses for long-running agents
  ↓
Harness design for long-running application development
  ↓
Managed Agents
  ↓
Claude Code auto mode
  ↓
Containment across products
```

Anthropic 工程页面在 2025–2026 年连续发布了 context engineering、long-running agents、harness design、auto mode、Managed Agents、containment 等文章，形成了一条相当清楚的工程化脉络。([Anthropic](https://www.anthropic.com/engineering "Engineering \ Anthropic"))

---

## 6.1 Long-running agents：长任务不是硬撑上下文，而是状态外置

Anthropic 在 long-running agents 文章中讨论了如何让 Agent 执行长任务。它的核心不是让一个 Agent 永远保持同一个上下文，而是通过 initializer agent、coding agent、progress file、git history 等机制，让任务状态可以外置和交接。([Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents "Effective harnesses for long-running agents \ Anthropic"))

这点很重要。

很多人把长任务问题理解成：

> 只要上下文窗口足够大，Agent 就能持续干活。

但真实工程里不是这样。

长任务需要的是：

- 环境初始化；
    
- 当前状态记录；
    
- 增量提交；
    
- 上下文 reset；
    
- 新 Agent 接手；
    
- 失败恢复；
    
- 明确下一步任务。
    

这和人类团队交接项目很像。  
一个新人接手项目，不能只靠上一位同事口头说“我差不多做完了”，而需要文档、commit history、issue 状态、测试结果和剩余任务。

Agent 也是一样。

---

## 6.2 Planner / Generator / Evaluator：三 Agent 架构的真正意义

Anthropic 在 2026 年 3 月的 long-running application development 文章中讨论了 planner、generator、evaluator 这样的三 Agent 架构，用于长期应用开发任务。这个结构不是简单“多几个 Agent 更聪明”，而是把不同职责拆开：规划、生成、评价。([Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps "Harness design for long-running application development \ Anthropic"))

这更像传统软件团队里的角色分工：

|Agent 角色|人类团队类比|
|---|---|
|Planner|架构师 / 技术负责人|
|Generator|开发者|
|Evaluator|测试 / 审查 / QA|

它的价值在于：

> **让一个 Agent 不必同时负责计划、实现和评价自己。**

因为让同一个 Agent 自己写、自己评、自己宣布完成，很容易形成闭环幻觉。

三 Agent 架构本质上是把软件工程里的“职责分离”引入 Agent 系统。

---

## 6.3 Managed Agents：把 brain 和 hands 解耦

Anthropic 2026 年 4 月的 Managed Agents 文章提出了一个很关键的思路：decoupling the brain from the hands。也就是把 Claude / harness / sandbox 这些部分拆开，形成更稳定的 agent runtime 抽象。([Anthropic](https://www.anthropic.com/engineering/managed-agents "Scaling Managed Agents: Decoupling the brain from the hands \ Anthropic"))

这和 OpenAI 的 App Server 有某种相似性。

两者都在做一件事：

> **把 Agent 从一个不可中断的聊天窗口，升级成可恢复、可观察、可替换、可持久化的工程运行时。**

聊天窗口适合短交互。  
但长任务、生产任务、多 Agent 任务，需要的是运行时系统：

- session 可以持久化；
    
- harness 可以重启；
    
- sandbox 可以隔离；
    
- 工具调用可以审计；
    
- 事件流可以回放；
    
- 任务可以恢复。
    

这才是工程化。

---

## 6.4 Auto mode：权限弹窗不是可靠的安全系统

Anthropic 在 Claude Code auto mode 文章中指出，依赖用户不断点击权限弹窗会产生 approval fatigue。Anthropic 的 containment 文章进一步提到，遥测显示用户批准了约 93% 的权限提示；随着提示变多，用户对每个提示的注意力会下降。([Anthropic](https://www.anthropic.com/engineering/claude-code-auto-mode "How we built Claude Code auto mode: a safer way to skip permissions \ Anthropic"))

这个结论很现实。

人类并不是一个可靠的 per-action classifier。  
尤其当 Agent 开始频繁读文件、改代码、运行命令、访问网络时，用户不可能认真判断每一个动作。

所以安全机制必须前移到：

- sandbox；
    
- filesystem boundary；
    
- egress control；
    
- tool output inspection；
    
- classifier；
    
- least privilege；
    
- trusted workspace；
    
- project trust boundary。
    

这和传统安全工程一致：不要把系统安全建立在“用户每次都认真阅读弹窗”的假设上。

---

## 6.5 Containment：Agent 越强，越要限制爆炸半径

Anthropic 2026 年 5 月的 containment 文章非常关键。文章指出，随着 agents 能力增强，它们的 blast radius 也会增加；工程问题变成了如何 cap blast radius。Anthropic 把防御分成 model、environment、external content 等层次，并强调环境层的 sandbox、VM、filesystem boundaries、egress controls 等机制。([Anthropic](https://www.anthropic.com/engineering/how-we-contain-claude "How we contain Claude across products \ Anthropic"))

这说明 Harness Engineering 的安全部分，不能只靠模型。

模型层防御可以降低风险，但模型是概率性的，不可能 100% 保证。  
环境层防御才是硬边界。

一句话：

> **不要只监督 Agent 想做什么，还要限制 Agent 能做什么。**

这也是 AI Agent 和传统软件工程结合最深的地方。

---

## 图 6：Anthropic Harness 演进时间线

![[Anthropic Harness 演进线.svg]]

---

# 7. 两条路线的共同结论：AI Coding 正在从聊天窗口走向工程系统

OpenAI 和 Anthropic 的路线看起来不同。

OpenAI 更像在说：

> Codex 如何从 coding agent 演进为可嵌入、可编排、可长期运行、可从生产反馈中改进的工程平台。

Anthropic 更像在说：

> Claude 如何在长任务、上下文交接、managed runtime、auto mode 和 containment 上变得更可靠、更安全。

但是两条路线的底层方向正在收敛。

它们都意识到：

> **AI Coding 的核心战场，正在从“模型会不会写代码”，转向“Agent 能不能被工程系统可靠地运行、编排、验证和约束”。**

对比一下：

|维度|OpenAI 路线|Anthropic 路线|
|---|---|---|
|起点|Codex coding agent|Claude Code / Claude agents|
|核心抓手|Codex harness / App Server / Symphony|long-running harness / Managed Agents / containment|
|工程重点|平台化、编排、生产反馈闭环|运行时解耦、长任务交接、安全边界|
|任务组织|issue tracker as control plane|progress file / git history / context reset|
|安全思路|sandbox + approval policy + workspace boundary|sandbox / VM / egress control / external content defense|
|共同目标|让 Agent 持续交付|让 Agent 可恢复、可控、可审计|

两者共同指向一个结论：

```text
AI Coding 不会停留在 Chat UI。
它会进入 repo、issue tracker、CI/CD、eval、sandbox、observability 和 production feedback loop。
```

这就是为什么我说，AI 越会写代码，软件工程反而越重要。

因为 AI 让“写代码”变快了，但它没有自动解决：

- 需求是否清楚；
    
- 架构是否合理；
    
- 上下文是否正确；
    
- 测试是否覆盖；
    
- 权限是否安全；
    
- 改动是否可回滚；
    
- 结果是否可验证；
    
- 失败是否可复盘；
    
- 经验是否能沉淀为规则。
    

这些仍然是软件工程的问题。

而且，因为 AI 执行速度更快、并发更多、权限更大，这些问题反而被放大了。

---

## 图 7：OpenAI vs Anthropic Harness 路线对比

![[OpenAI 与 Anthropic 的 Harness 收敛.svg]]

---

# 8. 普通开发者如何搭建轻量级 AI Coding Harness

看到 OpenAI 和 Anthropic 的实践，普通开发者很容易产生一种距离感：

> 这些都是大厂基础设施，我现在能做什么？

其实不需要一上来就做完整平台。

对个人开发者、小团队、后端工程项目来说，可以先搭一个轻量级 Harness。

我建议从七件事开始。

---

## 8.1 每个任务先写 spec，而不是直接让 AI 改代码

不要一上来就说：

```text
帮我实现这个功能。
```

而是先让 AI 帮你整理：

```text
目标是什么？
不做什么？
涉及哪些模块？
输入输出是什么？
验收标准是什么？
风险在哪里？
需要哪些测试？
```

spec.md 的价值不是形式，而是把模糊需求变成任务契约。

---

## 8.2 把项目事实放进仓库，而不是放在聊天记录里

聊天记录不是可靠事实源。

真正稳定的上下文应该放在仓库里，例如：

```text
/docs/architecture.md
/docs/conventions.md
/docs/decisions.md
/specs/feature-xxx.spec.md
/agent/review-checklist.md
/agent/progress.md
```

AI 可以读，开发者可以改，Git 可以追踪。

---

## 8.3 一个任务一个会话，避免上下文污染

很多 AI Coding 失控，不是因为模型差，而是因为会话太乱。

一个会话里讨论了需求、架构、bug、部署、闲聊、复盘，最后上下文变成垃圾场。

更好的方式是：

```text
一个任务
一个 spec
一个会话
一组明确交付物
```

任务完成后，把状态沉淀到文件，而不是继续拖着会话跑。

---

## 8.4 长任务必须状态外置

长任务不能完全依赖模型记忆。

可以让 AI 维护：

```text
当前目标
已完成事项
未完成事项
当前阻塞
关键决策
测试结果
下一步动作
```

这些内容要写入文件，而不是只存在聊天上下文里。

---

## 8.5 AI 做完必须提供可验证证据

不要接受这种总结：

```text
功能已经完成，测试通过。
```

要让 AI 给出：

```text
改了哪些文件？
为什么改？
运行了哪些命令？
测试结果是什么？
有没有失败项？
有没有未完成项？
如何手动验收？
```

没有证据的完成，不算完成。

---

## 8.6 用 review checklist 阻断假完成

建议每个项目都有一份 AI Coding review checklist。

至少包含：

```text
是否存在 TODO / stub / mock 冒充真实实现？
是否只改了前端，没有接后端？
是否接口定义和实现不一致？
是否文档声称完成，但代码没有完成？
是否没有运行测试却声称测试通过？
是否改动范围超出任务边界？
是否引入了未说明的新依赖？
是否破坏了已有架构约定？
```

这份 checklist 比“请认真检查”有效得多。

---

## 8.7 每次翻车都沉淀成下一轮规则

AI Coding 里最浪费的是：

> 同一个坑反复踩。

如果某次发现 AI 用 mock 冒充真实实现，就把它写入 checklist。  
如果某次发现 AI 没跑测试却说通过，就把“必须提供命令和输出”写入验收标准。  
如果某次发现长任务上下文漂移，就把任务状态外置写入流程。

这就是轻量级 Harness 的核心：

```text
事故 → 复盘 → 规则 → 检查 → 回归
```

---

## 图 8：普通开发者轻量级 AI Coding Harness 模板

![[轻量级 AI Coding Harness.svg]]

---

# 9. 结语：工程师不会消失，但工作层级正在上移

AI 编程最容易被误解成一个简单问题：

> AI 会不会替代程序员？

我现在觉得这个问题太粗了。

更准确的问题是：

> 当 AI Agent 能够大量生成代码、运行工具、修改项目、发起 PR、处理 issue、甚至根据生产反馈修复系统时，工程师应该站在哪一层？

答案不是“消失”，而是“上移”。

过去，工程师的大量时间花在直接实现上：

```text
写代码
调接口
改 bug
补测试
查日志
写文档
```

以后，这些工作不会完全消失，但越来越多会被 Agent 承担一部分。

工程师更重要的工作会变成：

```text
定义目标
拆解任务
组织上下文
设计架构边界
设置验收标准
建立测试和回归体系
约束权限和安全边界
审查 AI 的输出
把事故沉淀成规则
维护整个 AI Coding Harness
```

这不是软件工程的退场，而是软件工程的升级。

AI 越强，越需要工程系统来放大它的能力，也约束它的风险。

所以我现在对 AI 工程的理解可以压缩成三句话：

```text
Prompt Engineering：让 AI 明白任务。
Context Engineering：让 AI 拿到正确材料。
Harness Engineering：让 AI 在可控系统里持续交付。
```

最后一句更重要：

> **AI Coding 的终点不是“模型会写代码”，而是“人类能不能设计出让 Agent 可靠工作的工程系统”。**

---

# 关键词

AI Coding、Prompt Engineering、Context Engineering、Harness Engineering、Codex、Claude Code、Agent、软件工程、CI/CD、Eval、Sandbox、Agent Orchestration、Long-running Agents、Managed Agents、Self-improving Agents、Agent-first Codebase、Symphony、Containment、AI 工程化。