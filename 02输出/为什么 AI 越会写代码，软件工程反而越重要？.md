
> 从 Prompt Engineering、Context Engineering 到 Harness Engineering，看 OpenAI 与 Anthropic 正在收敛的 AI 工程路线

---

## 导语

我最近越来越怕看到 AI 总结里的“已完成”三个字。

不是因为 AI 不会写代码。恰恰相反，现在的 AI Coding Agent 已经很会写代码了：它能快速改文件、补接口、生成测试、更新文档，最后再给你一段非常完整的总结。

真正麻烦的是，很多“已完成”经不起工程检查。

在一些中型代码项目的 AI Coding 实践里，我反复遇到类似场景：AI 修改了不少文件，接口名也对上了，文档也更新了，最后总结写着“功能已完成，测试通过”。但仔细看才发现，关键路径仍然是 mock；或者只补了 DTO，没有接到真实 Service；或者它说“测试应该通过”，但并没有真实执行命令；还有一种更隐蔽的情况：前一个 Agent 把“计划完成”总结成“已经完成”，后一个 Agent 又把这个总结当事实继续推进。

这类问题不是简单的“模型不够聪明”。它更像一个工程治理问题：

> **AI 可以更快地产生代码，但谁来证明这些代码真的接进了系统、经过了验证、留下了可交接状态？**

最开始，我以为关键是 **Prompt Engineering**：只要把需求说清楚，把角色、步骤、边界、输出格式写明白，AI 就能更稳定地完成任务。

后来发现，这还不够。AI 不是孤立执行一句提示词，它需要理解项目的架构、文档、历史决策、代码规范、测试方式和当前状态。于是问题转向 **Context Engineering**：如何让 AI 在正确阶段看到正确材料，而不是把一堆文档粗暴塞进上下文窗口。

再往后，我发现即使 Prompt 写得清楚，Context 也组织得不错，AI 仍然可能假完成、漂移、误判、过度修改、没有真实验证。

这时问题进入第三层：**Harness Engineering**。

我现在更愿意把它理解为：

> **把 AI Agent 放进一个可控、可验证、可恢复、可迭代的软件工程系统里。**

Prompt 解决任务表达；Context 解决上下文供应；Harness 解决可验证、可恢复、可约束的持续交付。

下面这张图是全文的基本框架。

![[AI Coding 工程化三层.svg]]

---

# 1. 一个反直觉现象：AI 写得越快，假完成也来得越快

AI Coding 里有一个反直觉现象：

> **AI 越能写代码，工程师越不能只盯着“代码有没有生成”。**

因为“代码生成”只是开始，不是交付。

真正的问题通常分三层：

第一，**生成不等于接入系统**。AI 可能写了一个看起来完整的类、接口或组件，但没有真正接到调用链路里；也可能写了一个 mock service，然后在总结里把它描述成“接口已完成”。

第二，**看起来完成不等于经过验证**。AI 很擅长根据代码结构推断“这个测试应该能过”。但软件工程里，“应该能过”和“实际跑过”是两件事。一次真实的测试输出、一次失败日志、一次回归结果，比十句“逻辑上没有问题”更有价值。

第三，**总结可信不等于状态真实**。AI 的总结往往很流畅，但流畅不代表准确。尤其在多 Agent 协作里，一个 Agent 的错误总结会被另一个 Agent 当成事实继续使用。如果前一个总结把“计划完成”写成了“已经完成”，后一个 Agent 很可能不会重新验证，而是在错误前提上继续推进。

这就是 AI Coding 和传统代码生成工具的差异。

传统工具通常只生成片段；Agent 会阅读、计划、调用工具、修改项目、运行命令、写总结。它更像一个参与工程流程的“新成员”。既然它进入了工程流程，就不能只用“提示词”来管理它，而要用软件工程的方式管理它。

这张图描述的是只靠 Prompt 时，AI Coding 容易出现的失控链路：需求边界不清，AI 局部执行看似合理，最后总结大于事实，形成假完成。

![[AI Coding 常见失控链路.svg]]

所以，AI Coding 的核心问题已经从“模型会不会写代码”，扩展到了“工程系统能不能稳定发现、限制和修复 Agent 的错误”。

---

# 2. Prompt Engineering：解决“要做什么”，不解决“做没做成”

Prompt Engineering 不是写几句神奇咒语。

在 AI Coding 里，它更像是**任务表达工程**：把人类模糊、跳跃、隐含约束很多的需求，转化为 AI Agent 能理解、能执行、能验收的任务协议。

它对应的实践包括：

- brainstorm：先把模糊目标拆成几个可能方向；
- spec.md：把需求固化成任务规格；
- plan mode：让 AI 先制定计划，而不是直接改代码；
- acceptance criteria：定义什么叫完成；
- 角色约束和输出格式：降低沟通成本，方便后续审查。

Prompt Engineering 解决的是：

> **AI 知不知道要做什么。**

这一步仍然重要。没有好的任务表达，AI 只会在模糊需求里猜。

但 Prompt 有明显边界。它不能保证 AI 看到的是最新项目上下文，也不能保证 AI 一定真实运行测试，更不能保证 AI 做错后能被稳定发现。

所以 Prompt 是入口，但不是终点。

如果把 AI Coding 看成一条工程链路，Prompt 只是把任务送进系统。任务进去之后，AI 需要拿到正确材料，需要在受控环境里执行，需要留下证据，需要有人或机制检查结果。这些就不是 Prompt 单独能解决的。

---

# 3. Context Engineering：不是塞资料，而是管理事实源

当任务稍微复杂一点，Prompt 很快会遇到天花板。

因为 AI Coding 不是在空白文本上写作文，而是在一个不断演进的工程系统里修改代码。它需要知道项目架构、代码分层、依赖关系、接口规范、数据模型、历史决策、测试命令、当前任务状态，以及哪些地方不能乱改。

这些不是一条 Prompt 能解决的。

我现在更愿意把 Context Engineering 理解成：

> **为 AI Agent 设计一条信息供应链。**

重点不是“上下文越多越好”，而是：正确阶段、正确粒度、可信来源、可淘汰过期信息。

Anthropic 在关于 context engineering 的工程文章中强调，context engineering 的关键在于为 agent 的下一步动作提供正确的信息，而不是盲目增加上下文量。([Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents "Effective context engineering for AI agents | Anthropic"))

这点在 AI Coding 里非常具体。

一个 Agent 要改接口，不一定需要读完整仓库；它可能先需要接口定义、调用链、测试方式和相关约定。一个 Agent 要做长任务，也不应该只依赖聊天记录，而应该有 progress file、git history、任务清单和真实测试输出。

多 Agent 协作时，这个问题更明显。

我越来越倾向于把仓库、diff、测试输出和 progress file 当事实源，而不是把聊天总结当事实源。聊天总结可以帮助交接，但不能直接等同于事实。因为总结一旦写错，就可能被后续 Agent 放大：前一个 Agent 说“已完成”，后一个 Agent 就基于“已完成”继续做文档、补测试、写报告，最后整个链路都建立在一个未经验证的前提上。

所以，Context Engineering 不只是“聊天上下文管理”，而是在管理 Agent 的工作材料：

- 哪些信息来自仓库；
- 哪些信息来自文档；
- 哪些信息来自测试输出；
- 哪些信息只是上一次总结；
- 哪些信息已经过期；
- 哪些状态必须外置，不能只留在会话里。

OpenAI 的 Codex App 资料也提到，skills 可以 check into repository，让团队共享；Automations 可以用于周期性任务，如 issue triage、CI failures 汇总、release briefs 等。([OpenAI](https://openai.com/index/introducing-the-codex-app/?utm_source=chatgpt.com "Introducing the Codex app"))

这说明 Context Engineering 正在从“给模型塞材料”，走向“组织项目知识和 Agent 工作材料”。

但 Context 仍然不够。

即使 AI 看到了正确材料，它也可能没有按材料执行；执行后没有验证；验证后没有回归；修复后没有留下可交接状态；总结仍然不可信。

这时问题进入第三层：Harness Engineering。

---

# 4. Harness Engineering：让 AI 的工作可验证、可恢复、可约束

我现在对 Harness Engineering 的定义是：

> **Harness Engineering 是面向 AI Agent 的软件工程控制层。它把传统软件工程里的流程、测试、评审、CI/CD、权限、日志、回归、复盘等机制，改造成适合 AI Agent 的工作环境和约束系统。**

更简单地说：

> **Harness 不是让 AI 更聪明，而是让 AI 做错时更容易被发现、被限制、被修复。**

这句话很关键。

很多人第一次接触 AI Coding，会本能地把问题归因到 Prompt：提示词写得不够清楚，所以 AI 才假完成；上下文给得不够多，所以 AI 才漏改文件。

这些当然会影响结果，但它们不是全部。

如果一个系统里没有验收标准、没有真实测试、没有 review checklist、没有权限边界、没有进度文件、没有失败复盘，那么再好的 Prompt 也只能降低出错概率，不能形成可靠交付。

我认为轻量级 Harness 至少包含五类能力。

**第一，任务入口控制。** 通过 spec、任务边界、验收标准，让 AI 明确“要做什么”和“不做什么”。尤其要避免任务执行到一半开始扩大范围。

**第二，执行环境控制。** 通过 sandbox、权限、工具白名单、工作区隔离，限制 Agent 能访问哪里、能写哪里、能不能联网、哪些命令需要审批。

**第三，验证反馈控制。** 通过测试、构建、日志、CI、eval、人工验收，把“看起来合理”变成“有证据支持”。

**第四，状态恢复控制。** 通过 progress file、git history、task list、handoff note，让长任务可以中断、重启、交接，而不是完全依赖聊天上下文。

**第五，复盘回归控制。** 通过 review checklist、regression case、事故复盘，把踩过的坑沉淀成下一轮规则。

这张图把 Harness Engineering 看成一个闭环：不是一次性生成，而是计划、执行、验证、修复、复盘和回归的持续循环。

![[Harness Engineering 闭环图.svg]]

举一个最常见的例子：不要假实现。

如果只是告诉 AI：

> 不要用 mock 冒充真实实现。

这还不是 Harness。

真正的 Harness 应该把这条规则放进多个环节：

- spec 里明确禁止 mock / stub 冒充真实实现；
- review checklist 检查 TODO、stub、fake implementation；
- grep 或静态检查辅助发现明显占位代码；
- 测试必须覆盖真实调用链，而不是只测一个局部函数；
- CI 自动运行关键验证命令；
- AI 完成后必须给出实际执行过的命令和结果；
- 如果某次仍然翻车，就把它写入 regression rule。

这才是工程系统。

过去我们用软件工程约束人类团队：需求、设计、开发、测试、评审、CI/CD、上线、回归、复盘。现在 AI Agent 进入开发流程后，这套体系没有消失，反而要被重新设计，用来约束一个新的参与者：会写代码、会调用工具、会自主执行，但也会幻觉、漂移、误判和假完成的 Agent。

---

# 5. OpenAI 的公开实践观察：平台化、编排化、反馈闭环

下面不是 OpenAI 官方路线图，而是我基于公开资料观察到的几个工程趋势。

OpenAI 这条线如果只写“Codex 生成了很多代码”，很容易误读。更值得关注的是：它正在把 Codex 从一个交互式 coding agent，逐步工程化为可嵌入、可编排、可长期运行、可从生产反馈中改进、并受 sandbox 约束的开发平台。

我把它概括成三个观察。

---

## 5.1 观察一：Codex Harness / App Server —— Agent 运行时平台化

OpenAI 在 Codex agent loop 文章中，把 Codex harness 描述为所有 Codex 体验底层的核心 agent loop 和执行逻辑。这个 loop 负责组织用户、模型和工具之间的交互：模型提出工具调用，harness 执行工具，再把结果反馈给模型，直到任务完成。([OpenAI](https://openai.com/index/unrolling-the-codex-agent-loop/?utm_source=chatgpt.com "Unrolling the Codex agent loop"))

这意味着，Codex 不只是一个“会聊天的代码助手”。模型之外，还有一层运行时系统负责：

- 用户输入如何进入模型；
- 模型如何选择工具；
- 工具调用如何执行；
- 执行结果如何回到上下文；
- thread lifecycle 如何保存；
- 权限和 sandbox 如何参与执行。

到了 App Server，OpenAI 又把 Codex harness 从单一 CLI 体验中抽象出来。公开文章提到，Codex harness 包括 thread lifecycle and persistence、config and auth、tool execution and extensions；Codex core 既是 agent code 所在的 library，也是能运行 agent loop、管理 thread persistence 的 runtime。App Server 则通过 JSON-RPC 协议把这套能力暴露给不同客户端。([OpenAI](https://openai.com/index/unlocking-the-codex-harness/?utm_source=chatgpt.com "Unlocking the Codex harness: how we built the App Server"))

这件事的工程含义是：Agent 能力正在被平台化。

过去，一个 coding agent 更像一个客户端功能；现在，它开始被抽象成运行时、协议、事件流、线程持久化、工具执行和权限策略的组合。

这张图不是想复刻 OpenAI 内部架构，而是表达一个趋势：Codex 正在从聊天窗口走向可复用的 Agent 运行平台。

![[Codex Harness 平台化结构.svg]]

---

## 5.2 观察二：Agent-first Codebase / Symphony —— 任务和 Agent 编排化

OpenAI 的 Harness Engineering 文章里有一个很容易传播的案例：他们从空仓库开始，用 Codex 生成了约百万行代码，并通过约 1500 个 PR 推进项目。([OpenAI](https://openai.com/index/harness-engineering/?utm_source=chatgpt.com "Harness engineering: leveraging Codex in an agent-first ..."))

但我不建议把这个案例读成“AI 已经替代工程师写代码”。

它更值得关注的地方是工程流程被重新设计了：工程师通过 prompt 驱动 Codex，Codex 使用标准开发工具、仓库内 skills、PR review loop 和本地 / 云端 agent reviewers 来迭代，直到评审满意。

也就是说，当 AI 大规模参与开发时，仓库、文档、测试、CI、skills、日志、评审流程，都必须变成 Agent 可读、可执行、可验证的环境。

这才是 Agent-first codebase 的重点。

再往前一步，就是 Symphony。OpenAI 在 Symphony 文章中把 issue tracker 描述为 coding agents 的控制平面。它要解决的不是“单个 Agent 会不会写代码”，而是“人类如何管理多个 Agent 的并行工作”。([OpenAI](https://openai.com/index/open-source-codex-orchestration-symphony/?utm_source=chatgpt.com "An open-source spec for Codex orchestration: Symphony."))

这很接近日常工程里的真实痛点。

当你同时开多个 Agent session 时，很快会遇到注意力瓶颈：哪个 Agent 卡住了？哪个任务测试跑完了？哪个 PR 要 review？哪个 session 需要继续推动？如果全部靠人切换聊天窗口，规模一大就会失控。

Symphony 的方向是：不要让人管理一个个聊天窗口，而是让 issue tracker / task board 成为 Agent orchestration 的控制平面。

这张图把 OpenAI 的公开实践整理为几个工程趋势：运行时平台化、仓库 Agent-first、任务编排化、生产反馈闭环和安全边界。它不是官方时间线，而是基于公开资料的观察。

![[OpenAI Harness 演进线.svg]]

---

## 5.3 观察三：Self-improving Agents / Sandbox —— 生产反馈闭环和安全边界

OpenAI 关于 self-improving tax agents 的案例，把 Harness 往生产反馈方向推进了一步：不是只让 Codex 写代码，而是把生产 traces、专家纠错、targeted evals 和 Codex-driven improvement loop 连接起来。文章中提到，系统可以把生产中发现的问题包装成 targeted eval set，Codex 再结合 trace、eval、repo、skills 去调查根因、提出修改、运行评测和回归。([OpenAI](https://openai.com/index/building-self-improving-tax-agents-with-codex/?utm_source=chatgpt.com "Building self-improving tax agents with Codex"))

这背后的闭环是：

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

这不是“AI 一次性写代码”，而是生产反馈驱动系统持续改进。

但 Agent 越能做事，权限边界越重要。

OpenAI 关于 Codex Windows sandbox 和安全运行 Codex 的文章都强调，sandbox 和 approval policy 是配合关系：sandbox 定义技术执行边界，包括 Codex 能写哪里、能否访问网络、哪些路径受保护；approval policy 决定什么时候需要请求用户批准。([OpenAI](https://openai.com/index/building-codex-windows-sandbox/?utm_source=chatgpt.com "Building a safe, effective sandbox to enable Codex on ..."))([OpenAI](https://openai.com/index/running-codex-safely/?utm_source=chatgpt.com "Running Codex safely at OpenAI"))

这再次回到本文主线：AI 越强，越不能只靠“模型听话”。必须有操作系统和运行时级别的护栏。

---

# 6. Anthropic 的公开实践观察：长任务、运行时和 Containment

这也不是 Anthropic 官方路线图，而是从公开工程文章中可以观察到的一条实践脉络。

如果说 OpenAI 的公开实践更像在强调 Codex 的平台化、编排化和反馈闭环，那么 Anthropic 的文章更集中地讨论了几个 Agent 工程问题：长任务如何交接，职责如何分离，运行时如何解耦，Agent 越强如何限制爆炸半径。

这张图把这些问题放在一条工程脉络里：它不是为了罗列术语，而是为了说明 Anthropic 关注的核心并不是“让聊天窗口更长”，而是让 Agent 能被恢复、观察、隔离和约束。

![[Anthropic Harness 演进线.svg]]

---

## 6.1 问题一：长任务状态不能只放在上下文里

Anthropic 在 long-running agents 文章中讨论了如何让 Agent 执行长任务。它的核心不是让一个 Agent 永远保持同一个上下文，而是通过 initializer agent、coding agent、progress file、git history 等机制，让任务状态可以外置和交接。([Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents "Effective harnesses for long-running agents | Anthropic"))

这点和真实工程经验很一致。

长任务里最危险的不是一开始做错，而是做到后半段开始漂移：原本只是修一个接口，后来顺手改了配置、重构了目录、补了文档，最后连任务边界都变模糊。

如果没有外置的目标、进度和验收标准，聊天上下文会慢慢变成不可靠的项目状态源。

人类团队交接项目，不能只靠上一位同事口头说“我差不多做完了”。Agent 也是一样。它需要 progress file、commit history、测试结果、剩余任务和明确的下一步动作。

---

## 6.2 问题二：不能让同一个 Agent 自写自评

Anthropic 在 long-running application development 文章中讨论了 planner、generator、evaluator 这样的三 Agent 架构，用于长期应用开发任务。([Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps "Harness design for long-running application development | Anthropic"))

我更愿意把它理解成一种 harness pattern，而不是多 Agent 炫技。

Planner 负责规划，Generator 负责实现，Evaluator 负责评价。对应到人类软件团队，大致就是架构 / 开发 / 测试评审的职责分离。

它的价值不在于“多几个 Agent 一定更聪明”，而在于：

> **不要让同一个 Agent 同时负责计划、实现和宣布自己完成。**

让一个 Agent 自己写、自己评、自己总结，很容易形成闭环幻觉。职责分离不是 AI 时代的新发明，而是传统软件工程在 Agent 系统里的重新出现。

---

## 6.3 问题三：Agent 不该只是聊天窗口，而应是可恢复的运行时单元

Anthropic 的 Managed Agents 文章提出了一个关键思路：decoupling the brain from the hands。也就是把 Claude、harness、sandbox 等部分拆开，形成更稳定的 agent runtime 抽象。([Anthropic](https://www.anthropic.com/engineering/managed-agents "Scaling Managed Agents: Decoupling the brain from the hands | Anthropic"))

这和 OpenAI 的 App Server 有相似之处。

两者都在把 Agent 从一个不可中断的聊天窗口，升级成可恢复、可观察、可替换、可持久化的工程运行时。

聊天窗口适合短交互。但长任务、生产任务、多 Agent 任务，需要的是运行时系统：session 可以持久化，harness 可以重启，sandbox 可以隔离，工具调用可以审计，事件流可以回放，任务可以恢复。

这就是为什么 AI Coding 走到后面，问题会越来越像软件基础设施问题。

---

## 6.4 问题四：Agent 越强，越要限制爆炸半径

Anthropic 在 Claude Code auto mode 文章中指出，依赖用户不断点击权限弹窗会产生 approval fatigue。Anthropic 的 containment 文章进一步提到，随着 agents 能力增强，它们的 blast radius 也会增加；工程问题变成了如何 cap blast radius。([Anthropic](https://www.anthropic.com/engineering/claude-code-auto-mode "How we built Claude Code auto mode: a safer way to skip permissions | Anthropic"))([Anthropic](https://www.anthropic.com/engineering/how-we-contain-claude "How we contain Claude across products | Anthropic"))

这非常现实。

人类并不是一个可靠的 per-action classifier。尤其当 Agent 高频读文件、改代码、运行命令、访问网络时，用户不可能认真判断每一个权限弹窗。

所以安全机制必须前移到 sandbox、VM、filesystem boundaries、egress controls、tool output inspection、least privilege 和 trusted workspace。

一句话：

> **不要只监督 Agent 想做什么，还要限制 Agent 能做什么。**

这也是 AI Agent 和传统软件工程、安全工程结合最深的地方。

---

# 7. 两条路线的共同结论：AI Coding 正在从聊天窗口走向工程系统

OpenAI 和 Anthropic 的路线看起来不同。

OpenAI 更像在回答：Codex 如何从 coding agent 演进为可嵌入、可编排、可长期运行、可从生产反馈中改进的工程平台。

Anthropic 更像在回答：Claude 如何在长任务、上下文交接、managed runtime、auto mode 和 containment 上变得更可靠、更安全。

但它们的底层方向正在收敛。

共同点不是“都在做更强模型”，而是都在把 Agent 放进更复杂的工程控制系统。

这张图把两条路线放在一起看：OpenAI 偏平台化、编排化、生产反馈闭环；Anthropic 偏运行时解耦、长任务交接、安全边界。两者最后都指向同一件事：Agent 要可运行、可编排、可验证、可恢复、可约束。

![[OpenAI 与 Anthropic 的 Harness 收敛.svg]]

这就是为什么我说，AI 越会写代码，软件工程反而越重要。

AI 让“写代码”的边际成本下降了，但它没有自动解决这些问题：

- 需求是否清楚；
- 架构是否合理；
- 上下文是否正确；
- 测试是否真实执行；
- 权限是否安全；
- 改动是否可回滚；
- 结果是否可验证；
- 失败是否能复盘；
- 经验是否能沉淀成规则。

而且，因为 AI 执行速度更快、并发更多、权限更大，这些问题反而被放大了。

AI Coding 不会停留在 Chat UI。它会进入 repo、issue tracker、CI/CD、eval、sandbox、observability 和 production feedback loop。

这不是软件工程变轻了，而是软件工程的控制对象发生了变化：过去主要管理人类开发流程，现在还要管理 Agent 的交付流程。

---

# 8. 普通团队可以从一个最小 AI Coding Harness 开始

看到 OpenAI 和 Anthropic 的实践，普通开发者很容易产生距离感：

> 这些都是大厂基础设施，我现在能做什么？

其实不需要一上来就做完整平台。

对个人开发者、小团队、后端工程项目来说，可以先搭一个最小 Harness。它不复杂，核心是让 AI 的每次完成都留下证据。

我建议从这几个文件和动作开始：

```text
/specs/feature-xxx.spec.md          # 任务目标、边界、验收标准
/agent/progress.md                  # 当前状态、已完成、未完成、阻塞
/agent/review-checklist.md          # 假完成、越界修改、测试缺失检查
/agent/completion-report.md         # 改动说明、命令输出、失败项、验收方式
/tests/regression-cases.md          # 事故复盘后沉淀的回归用例
固定验证命令：build / test / lint / integration test
```

这张图把“最小 Harness”画成一个交付闭环：任务先进入 spec，执行过程写入 progress，完成前经过 checklist 和 test evidence，最后形成 completion report；一旦翻车，就沉淀成 regression rule。

![[轻量级 AI Coding Harness.svg]]

如果放到普通 Java 后端项目里，这个 checklist 可以非常具体：

- Controller / Service / Repository 是否全链路接入；
- 是否只写了 DTO，但没有对应数据库迁移或映射；
- 是否只补了单测，没有跑集成测试；
- 是否绕过真实中间件或外部服务；
- 是否用 mock / stub 冒充真实实现；
- 是否接口定义、文档和代码实现不一致；
- 是否引入了未说明的新依赖；
- 是否改动范围超出任务边界。

这里的重点不是流程复杂，而是完成标准清楚。

不要接受这种总结：

```text
功能已经完成，测试通过。
```

更好的完成报告应该包含：

```text
改了哪些文件？
为什么改？
运行了哪些命令？
命令输出是什么？
有没有失败项？
有没有未完成项？
如何手动验收？
如果下次再遇到类似问题，应该加入哪条 checklist？
```

轻量级 Harness 的核心不是让开发流程变重，而是把“AI 说完成”变成“有证据地完成”。

每次翻车都应该进入下一轮规则。

如果某次发现 AI 用 mock 冒充真实实现，就把它写入 checklist。  
如果某次发现 AI 没跑测试却说通过，就把“必须提供命令和输出”写入验收标准。  
如果某次发现长任务上下文漂移，就把任务状态外置写入流程。

这就是普通团队能做的 AI Coding Harness：

```text
事故 → 复盘 → 规则 → 检查 → 回归
```

不宏大，但有效。

---

# 9. 结语：AI 降低了写代码成本，也抬高了工程系统价值

AI 编程最容易被误解成一个简单问题：

> AI 会不会替代程序员？

我现在觉得这个问题太粗了。

更准确的问题是：

> 当 AI Agent 能够大量生成代码、运行工具、修改项目、发起 PR、处理 issue，甚至根据生产反馈修复系统时，我们如何设计一个让它可靠工作的工程系统？

AI 确实降低了写代码的成本。

但它同时提高了验证、约束、协作、恢复的价值。

过去，很多工程问题可以靠人类开发者的经验和自觉兜底：知道自己有没有跑过测试，知道哪里只是临时方案，知道哪些改动没有接进真实链路。

Agent 不一定知道。即使知道，它也可能在总结里表达错、在长任务里漂移、在多 Agent 协作中传递错误事实。

所以，AI 越会写代码，软件工程不是变轻了，而是从“写代码的方法”升级成了“管理 Agent 交付能力的系统”。

我对 AI 工程的理解可以压缩成三句话：

```text
Prompt Engineering：让 AI 明白任务。
Context Engineering：让 AI 拿到正确材料。
Harness Engineering：让 AI 在可控系统里持续交付。
```

最后一句最重要：

> **AI Coding 的终点不是“模型会写代码”，而是“人类能不能设计出让 Agent 可靠工作的工程系统”。**

---

# 关键词

AI Coding、Prompt Engineering、Context Engineering、Harness Engineering、Codex、Claude Code、Agent、软件工程、CI/CD、Eval、Sandbox、Agent Orchestration、Long-running Agents、Managed Agents、Self-improving Agents、Agent-first Codebase、Symphony、Containment、AI 工程化。
