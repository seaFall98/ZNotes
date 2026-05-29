---
title: “Harness Engineering 到底是什么？你的项目用了吗？”
source: https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247553908&idx=1&sn=3838c01de39b3d948a35d7ae8733a5de&chksm=cea15cbff9d6d5a92eae1c9f2c47e1ae0a458a860200138bb1caa839b5cd557f892185bbbdbc&scene=178&cur_album_id=4412413577266053125&search_click_id=#rd
author:
  - "[[Guide]]"
published:
created: 2026-05-30
description:
tags:
  - clippings
---
Guide *2026年4月9日 14:09*

上个周末，我发了一条吐槽 [AI 时代各种新概念太多](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247553844&idx=1&sn=d1050a02b731f177bcb0df90393aef05&scene=21#wechat_redirect) 的文字消息。没想到评论区有读者留言，实习面试被问到对 **Harness Engineering** 的理解。

![图片](https://mmbiz.qpic.cn/mmbiz_png/icJuV9QZRqg75icuC2DPiaKT23n9BBeZVKTleOh874O1V3MAEbJU3rIr5EWEamgsayZ7mqbPEkRWtYPSkCJL7Iqqnyb5092BMNU7CPKrm4CL6s/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=0)

这个概念火起来也就最近半个月不到的事，面试官就开始问了，不得不说追新的速度确实有点离谱。Guide 对面试官的行为不做评价——可能人家就是想考察求职者对新概念的了解程度。只是如果换做我当面试官，大概率会避开这么新的概念，毕竟技术迭代太快。

吐槽归吐槽，既然面试已经在问了，那就得搞清楚。而且退一步讲，就算不为面试，只要做 AI Agent 相关的工作，Harness Engineering 也是绕不开的。Can.ac 做过一个实验：同一个模型，只换了文件编辑接口的调用方式，编码基准分数从 6.7% 直接跳到 68.3%。 **模型没变，变的是外围的那套系统。** 这就是 Harness Engineering 在做的事。

Mitchell Hashimoto 在博客里用了这个说法（他原话是”我不知道业界有没有公认的术语，我自己管这叫 harness engineering”），OpenAI 几天后发了一篇百万行代码的实验报告，Birgitta Böckeler 在 Martin Fowler 网站上写了深度分析，Anthropic 在三月份又放出了全新的多智能体架构设计。几周之内，Harness 成了讨论 AI Agent 开发绕不开的概念。

今天 Guide 就来系统梳理 Harness Engineering 的核心概念和工程方法，帮你搞清楚： **决定 Agent 表现的天花板，到底在哪里。** 本文接近 1.3w 字，建议收藏，通过本文你将搞懂：

1. **Harness 到底是什么** ：为什么说“你不是模型，那你就是 Harness”？Agent = Model + Harness 这个公式怎么理解？和 Prompt Engineering、Context Engineering 是什么关系？六层架构长什么样？
2. ⭐ **为什么瓶颈不在模型而在 Harness** ：同一个模型只换了接口格式，分数从 6.7% 跳到 68.3%？上下文用到 40% Agent 就开始变蠢？
3. ⭐ **从零搭建 Harness 的行动清单** ：P0/P1/P2 三个优先级，按需取用。
4. ⭐ **一线团队实战案例** （附录）：OpenAI 三人五月百万行零手写、Anthropic 的 GAN 式三智能体架构和 context resets 交接棒策略、Stripe 每周 1300+ 无人值守 PR、Mitchell Hashimoto 的六步进阶。

> **📌 系列阅读** ：本文是 AI Agent 系列的一部分，相关文章：
> 
> - AI Agent 核心概念：Agent Loop、Context Engineering、Tools 注册 <sup>[1]</sup>
> - Agent Skills 详解：是什么？怎么用？和 Prompt、MCP 有什么区别？ <sup>[2]</sup>
> - 万字拆解 MCP，附带工程实践 <sup>[3]</sup>

## ⭐️ Harness 核心概念

### Harness 到底是什么？

一句话： **Agent = Model + Harness。你不是模型，那你就是 Harness。**

这句话是不是感觉听起来有点绝对，我第一次看到也是这种感觉。不过，其实这样简单的一句话反而抓住了关键。

**Harness 就是模型之外的一切** ——系统提示词、工具调用、文件系统、沙箱环境、编排逻辑、钩子中间件、反馈回路、约束机制。模型本身只是能力的来源，只有通过 Harness 把状态、工具、反馈和约束串起来，它才真正变成一个 Agent。

LangChain 的 Vivek Trivedi 在《The Anatomy of an Agent Harness》里把这个定义讲得很清楚： **先搞清楚模型负责什么，剩下的系统要补什么，用这条线把整个系统切开。**

**通俗理解：** 模型是 CPU，Harness 是操作系统。CPU 再强，OS 拉胯也白搭。你买了最新款 M5 芯片，装了个崩溃不断的系统，体验还不如老芯片配稳定的 OS。

![Agent = Model + Harness](https://mmbiz.qpic.cn/sz_mmbiz_png/icJuV9QZRqg6mAzgpe4CRaRo2C9F3SHmicEUibqbnyIicedH6bnzeZStdI5ibeiaQXtIeYTjVLkSYIrgERt9UI5hL06Tq2avqDWV58HqWsJAOOdicc/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=2)

Agent = Model + Harness

### Harness 和 Prompt/Context Engineering 是什么关系？

三者不是并列关系，而是嵌套关系。更重要的是， **每一层解决的是完全不同的问题** ：

![Harness 和 Prompt/Context Engineering 的关系](https://mmbiz.qpic.cn/sz_mmbiz_png/icJuV9QZRqg7LtmG9nqsPY9zIGndicAz87kuFExMib7mqkTkv2hIuNGEChRdLR6uwia1HXV21p1YiaQdZvgYRrmkHYmXbyDl9DMEVKHBkCZSaicvA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=3)

Harness 和 Prompt/Context Engineering 的关系

| 层级 | 解决的核心问题 | 关注点 | 典型工作 |
| --- | --- | --- | --- |
| **Prompt Engineering** | 表达——怎么写好指令 | 塑造局部概率空间，让模型听懂意图 | 系统提示词设计、Few-shot 示例、思维链引导 |
| **Context Engineering** | 信息——给 Agent 看什么 | 确保模型在合适的时机拿到正确且必要的事实信息 | 上下文管理、RAG、记忆注入、Token 优化 |
| **Harness Engineering** | 执行——整个系统怎么防崩、怎么量化、怎么持续运转 | 长链路任务中的持续正确、偏差纠正、故障恢复 | 文件系统、沙箱、约束执行、熵管理、反馈回路 |

Guide 的理解是：简单任务里，提示词最重要——你把话说清楚就行；依赖外部知识的任务里，上下文很关键——你得把正确的信息喂进去；但在长链路、可执行、低容错的真实商业场景里，Harness 才是决定成败的东西。这也是为什么一线团队的重心都放在了 Harness 上。

### Harness 包含哪些组件？

理解 Harness 的最好方式，不是直接看它包含什么，而是看模型做不到什么。不管大模型看起来多能干，本质就是一个文本（或图像、音频）进、文本出的函数。

**模型做不到的，就是 Harness 要补的：**

| 模型做不到 | Harness 怎么补 | 核心组件 |
| --- | --- | --- |
| 记住多轮对话历史 | 维护对话历史，每次请求时拼进上下文 | **记忆系统** |
| 执行代码、跑命令 | 提供 Bash + 代码执行环境 | **通用执行环境** |
| 获取实时信息（新库版本、API 变化） | Web Search、MCP 工具 | **外部知识获取** |
| 操作文件和环境 | 文件系统抽象 + Git 版本控制 | **文件系统** |
| 知道自己做对了没有 | 沙箱环境 + 测试工具 + 浏览器自动化 | **验证闭环** |
| 在长任务中保持连贯 | 上下文压缩、记忆文件、进度追踪 | **上下文管理** |

**通俗理解：** 把这些“模型做不了但你希望 Agent 能做到”的事情一个个补上，就得到了 Harness 的核心组件。LangChain 有一位大佬把这件事拆解为五个子系统：文件系统（持久化）、Bash 执行（通用工具）、沙箱环境（安全隔离）、记忆机制（跨会话积累）、上下文压缩（对抗衰减）。

## Harness 进阶

### ⭐️ 一个成熟的 Harness 长什么样？

上面对组件的理解是“缺什么补什么”的思路。但如果从系统设计的角度看，一个成熟的 Harness 其实有清晰的层次结构。

我在油管看到一位技术大佬分享了一个六层体系，Guide 觉得这个框架把 Harness 的全貌描绘得比较完整：

![Harness Engineering 六层架构](https://mmbiz.qpic.cn/mmbiz_svg/Q3auHgzwzM7o1jwVs8BvSWrwgYTWyvcUuJpIISMY40HjBHpiaia69hfrmRicWzNDT7GVCDCcAaFjBehe9fMleibibJCjKRNGCMWI9tHegTwZ4hPOBzE4QeRBjhg/640?wx_fmt=svg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=4)

Harness Engineering 六层架构

| 层级 | 名称 | 解决什么问题 | 关键设计 |
| --- | --- | --- | --- |
| **L1** | **信息边界层** | Agent 该知道什么、不该知道什么 | 定义角色与目标，裁剪无关信息，结构化组织任务状态 |
| **L2** | **工具系统层** | Agent 怎么跟外部世界交互 | 工具的选拔、调用时机、结果的提炼与反馈 |
| **L3** | **执行编排层** | 多步骤任务怎么串起来 | 让模型像人一样走完“理解目标 → 判断信息 → 分析 → 生成 → 检查”的完整轨道 |
| **L4** | **记忆与状态层** | 长任务中间结果怎么管 | 独立管理当前任务状态、中间产物和长期记忆，防止系统混乱 |
| **L5** | **评估与观测层** | Agent 怎么知道自己做对了没有 | 建立独立于生成过程的验证机制，让 Agent 具备“自知之明” |
| **L6** | **约束、校验与恢复层** | 出错了怎么办 | 预设规则拦截错误，失败时（API 超时、格式混乱）提供重试或回滚机制 |

**通俗理解：** 你可以把它类比成给一个新手员工搭建的完整工作环境。L1 是岗位说明书（告诉 ta 该关注什么），L2 是办公工具（给 ta 用什么干活），L3 是标准操作流程（按什么步骤做事），L4 是项目管理系统和笔记本（怎么记住做过的事），L5 是质检流程（怎么检验做对了没有），L6 是红线规则和应急预案（什么事绝对不能做、出了事怎么补救）。

这个六层架构最大的价值在于——它不是简单的功能堆叠，而是一个从“定义边界”到“兜底恢复”的完整闭环。附录中一线团队的实践也印证了这一点：他们的做法都可以映射到这六层里。

⚠️ **注意** ：不要试图一开始就搭齐六层。从 L1（信息边界）和 L6（约束与恢复）入手，这两层投入产出比最高。L1 决定了 Agent 知道该干什么，L6 决定了它搞砸了能不能拉回来。中间的层次随着项目复杂度增长逐步补齐。

### 为什么瓶颈不在模型而在 Harness？

说实话，Guide 第一次看到这个结论的时候也觉得有点反直觉——不是应该等更强的模型出来就好了吗？但数据确实不支持这个想法。OpenAI、Anthropic、Stripe、LangChain、Can.ac 的实验数据指向同一个结论： **基础设施才是瓶颈，而非智能水平。**

🐛 **常见误区** ：很多团队一遇到 Agent 表现不好，第一反应是“换更强的模型”或“调整提示词”。但 Can.ac 的实验证明，同一模型只换了工具调用格式，效果就能差十倍。 **瓶颈大概率不在模型智能水平，而在 Harness 的基础设施质量。**

LangChain 那边也印证了这个结论：他们优化了 Agent 运行环境（文档组织方式、验证回路、追踪系统），在 Terminal Bench 2.0 上从全球第 30 名升到第 5 名，得分从 52.8% 提升到 66.5%。模型没换，Harness 换了。

> **📌 一个值得注意的发现** ：
> 
> LangChain 还指出了一个 model-harness 耦合问题——当前的 Agent 产品（如 Claude Code、Codex）是模型和 Harness 一起训练的，这导致一种过拟合： **换了工具逻辑后模型表现会变差** 。
> 
> 他们在 Terminal Bench 2.0 排行榜上观察到，Opus 在 Claude Code 中的 Harness 下的得分，远低于它在其他 Harness 中的得分。结论是："the best harness for your task is not necessarily the one a model was post-trained with"——为你的任务选择 Harness 时，不要被模型的默认 Harness 束缚。

### ⭐️ 为什么上下文喂越多，Agent 反而越蠢？

Dex Horthy 观察到一个现象：168K token 的上下文窗口，用到大约 40% 的时候，Agent 的输出质量就开始明显下降。

![上下文利用率的 40% 阈值现象](https://mmbiz.qpic.cn/mmbiz_svg/Q3auHgzwzM4w3uriamGibqQvWS5ibxNiaGcuz9vU9MU2Eicp7dIkSuQoicnrgBNCkbJagMmicfC0RfWm3olmXdvyrG3iaibiaFww2oq3duLbVnuicmlmmyTCZ1Y5gctYg/640?wx_fmt=svg&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=5)

上下文利用率的 40% 阈值现象

| 区间 | 占比 | 表现 |
| --- | --- | --- |
| **Smart Zone** | 0 - ~40% | 推理聚焦、工具调用准确、代码质量高 |
| **Dumb Zone** | 超过 ~40% | 幻觉增多、兜圈子、格式混乱、低质量代码 |

Anthropic 在自己的实践中也碰到了类似的问题，他们叫“上下文焦虑”：Sonnet 4.5 在上下文快填满时会变得犹豫，倾向于提前收工——哪怕任务还没做完。光靠压缩不够，他们最终的做法是直接清空上下文窗口，但通过结构化的交接文档把关键状态留下来（详见附录中 Anthropic 的 context resets 策略）。

你的目标不是给 Agent 塞更多信息，而是让它在任何时候都运行在干净、相关的上下文里。一线团队的实践都围绕着“渐进式披露”和“分层管理”在做，背后的原因就是这个 40% 阈值。

> ⚠️ **工程视角** ：在生产环境中监控上下文利用率是第一优先级。建议设置 40% 阈值告警——当 Agent 的上下文占用超过这个比例时，就应该触发上下文压缩或任务交接。等到 Agent 已经变蠢了再处理就晚了。

### ⭐️ 如果你要开始搭 Harness，应该从哪里入手？

综合一线团队的实践经验（详见附录），Guide 梳理了一个按优先级的行动路线。说实话你不需要一开始就把所有东西都搞齐，先把 P0 做了效果就会很明显。

#### P0：不用犹豫，立即可以做

| 行动 | 为什么 | 参考实践 |
| --- | --- | --- |
| 创建 `AGENTS.md` 并持续维护 | Agent 每次启动自动加载，犯错就更新，形成反馈循环 | Hashimoto 每一行对应一个历史失败案例 |
| 构建自定义 Linter + 修复指令 | 错误消息里直接告诉 Agent 怎么改，纠错的同时在“教” | OpenAI 的 Linter 报错自带修复方法 |
| 把团队知识放进仓库 | 写在 Slack/Wiki/Docs 里的知识对 Agent 等于不存在 | OpenAI 以仓库为唯一事实源 |

> 🐛 **常见误区** ：很多团队把 `AGENTS.md` 当成“超级 System Prompt”来写，恨不得把所有规则塞进一个文件。结果上下文窗口被撑爆，Agent 反而更蠢了。正确做法是像 OpenAI 一样—— `AGENTS.md` 只当目录用（约 100 行），详细规则放在子文档中按需加载。

#### P1：P0 做完之后，可以考虑这些

| 行动 | 为什么 | 参考实践 |
| --- | --- | --- |
| 分层管理上下文 | 不要把所有东西塞进一个文件，渐进式披露 | OpenAI AGENTS.md 当目录用（约 100 行） |
| 建立进度文件和功能列表 | JSON 格式追踪功能状态，Agent 不太会乱改结构化数据 | Anthropic 初始化 Agent + 编码 Agent 两阶段 |
| 给 Agent 端到端验证能力 | 浏览器自动化让 Agent 能像用户一样验证功能 | Anthropic 用 Playwright/Puppeteer MCP |
| 控制上下文利用率 | 尽量不超过 40%，增量执行 | Dex Horthy 的 Smart Zone / Dumb Zone |

#### P2：有余力再考虑

| 行动 | 为什么 | 参考实践 |
| --- | --- | --- |
| Agent 专业化分工 | 每个 Agent 携带更少无关信息，留在 Smart Zone | Carlini 的去重/优化/文档 Agent |
| 定期垃圾回收 | 确保清理速度跟得上生成速度 | OpenAI 的后台清理 Agent |
| 可观测性集成 | 把“性能优化”从玄学变成可度量的工作 | OpenAI 接入 Chrome DevTools |

### 你的 Harness 到哪个阶段了？

| 阶段 | 特征 | 工程师角色 |
| --- | --- | --- |
| Level 0：无 Harness | 直接给 Agent prompt，无结构化约束 | 手动写代码 + 偶尔使用 AI |
| Level 1：基础约束 | `AGENTS.md`  \+ 基础 Linter + 手动测试 | 主要写代码，AI 辅助 |
| Level 2：反馈回路 | CI/CD 集成 + 自动化测试 + 进度追踪 | 规划 + 审查为主 |
| Level 3：专业化 Agent | 多 Agent 分工 + 分层上下文 + 持久化记忆 | 环境设计 + 管理为主 |
| Level 4：自治循环 | 无人值守并行化 + 自动化熵管理 + 自修复 | 架构师 + 质量把关者 |

## 面试准备要点

Guide 把 Harness Engineering 相关的高频面试问题整理在下面，方便你快速回顾：

**基础概念**

| 问题 | 核心回答 |
| --- | --- |
| **Harness 是什么？** | 模型之外的一切——系统提示词、工具调用、文件系统、沙箱、编排逻辑、约束机制。Agent = Model + Harness。 |
| **Harness 和 Prompt Engineering、Context Engineering 的关系？** | 嵌套关系：Prompt ⊂ Context ⊂ Harness。三者分别解决表达、信息、执行三个层面的问题。 |
| **为什么瓶颈不在模型而在 Harness？** | Can.ac 实验证明同一模型只换工具调用格式，分数从 6.7% 跳到 68.3%。基础设施质量决定了模型能力的实际发挥。 |

**架构设计**

| 问题 | 核心回答 |
| --- | --- |
| **Harness 六层架构是什么？** | L1 信息边界 → L2 工具系统 → L3 执行编排 → L4 记忆与状态 → L5 评估与观测 → L6 约束校验与恢复。从“定义边界”到“兜底恢复”的完整闭环。 |
| **上下文管理有什么经验法则？** | 利用率控制在 40% 以内。超过后 Agent 质量明显下降（幻觉增多、兜圈子）。策略是压缩或交接，不是继续塞信息。 |
| **单 Agent 还是多 Agent？** | 规模决定。小项目单 Agent 够用（Hashimoto 模式），大项目几乎必然需要专业化分工（Carlini 用 16 个并行 Agent）。 |

**实战方案**

| 问题 | 核心回答 |
| --- | --- |
| **OpenAI 的 Harness 实践核心是什么？** | 五大方法论：地图式文档（渐进式披露）、机械化约束（自定义 Linter）、可观测性接入、熵管理（定期垃圾回收）、仓库即事实源。 |
| **Anthropic 如何解决上下文焦虑？** | Context resets 策略：不压缩，而是启动一个全新“干净”的 Agent，通过结构化交接文档恢复状态。类似重启进程解决内存泄漏。 |
| **从零搭 Harness 先做什么？** | P0：创建 AGENTS.md + 自定义 Linter + 团队知识仓库化。投入产出比最高。 |

## 还没有答案的问题

Harness Engineering 是一个快速发展的领域，仍有许多未解的问题。Guide 觉得了解这些“不知道”同样重要——面试时能展现你的思考深度。

| 问题 | 现状 | 谁在关注 |
| --- | --- | --- |
| **棕地项目怎么改造？** | 所有公开案例全是绿地项目，零方法论 | Böckeler：比作“在从没用过静态分析的代码库上跑静态分析”。她还提出“Ambient Affordances”概念：环境本身的结构特性（类型系统、模块边界、框架抽象）决定了 Harness 能做多好 |
| **怎么验证 Agent 做对了事？** | 大家擅长“约束不做错事”，但“验证做对了事”远未解决 | Böckeler 批评：用 AI 生成的测试来验证 AI 生成的代码，本质上是“用同一双眼睛检查自己的作业”——"that's not good enough yet" |
| **AI 生成代码的长期可维护性？** | LLM 代码经常重新实现已有功能，长期效果未知 | Greg Brockman 提出至今无人回答 |
| **Harness 该做厚还是做薄？** | Manus 五次重写越做越简单 vs OpenAI 五个月越做越复杂 | 场景决定：通用产品追求最小化，特定产品可以高度定制。而且随着模型变强，已有 Harness 应该定期简化（Anthropic 实测验证） |
| **单 Agent 还是多 Agent？** | Hashimoto 坚持单 Agent vs Carlini 用 16 个并行 Agent | 规模决定：小项目单 Agent 够用，大项目几乎必然需要专业化 |

绿地项目和棕地项目是软件工程里的经典比喻：

- 绿地项目（Greenfield）：从零开始的新项目，没有历史包袱。就像在一片空地上盖房 子，想怎么设计都行。
- 棕地项目（Brownfield）：在已有代码库上改造，有历史架构、技术债、遗留逻辑的约 束。就像在老旧城区搞翻新，到处是管线不能随便动。

OpenAI、Anthropic、Stripe、Hashimoto 这些成功案例，全部是在全新项目上从零搭 Harness。但现实中绝大多数团队面对的是已经跑了多年的代码库——怎么把 Harness 入一个十年历史、没有架构约束、到处是技术债的项目？目前没有任何公开方法论。

## 总结

写到这里，Guide 觉得可以用一句话概括 Harness Engineering 做的事情： **承认模型有边界，然后把边界之外的需求一个个工程化地补上。**

有一句话我特别认同： **模型决定了系统的上限，Harness 决定了系统的底线。**

在简单任务中提示词最重要，在依赖外部知识的任务中上下文很关键，但在长链路、可执行、低容错的真实商业场景中，Harness 才是 AI 稳定落地的前提条件。

**如果只记一句话：模型决定上限，Harness 决定底线。与其纠结选哪个模型，不如先把 Harness 搭好。**

## 附录：一线团队实战案例

在由于篇幅问题，这里就不放了，感兴趣的可以到 JavaGuide(javaguide.cn)上看，已经同步上去了：

参考资料

\[1\]

AI Agent 核心概念：Agent Loop、Context Engineering、Tools 注册: *[https://javaguide.cn/ai/agent/agent-basis.html](https://javaguide.cn/ai/agent/agent-basis.html)*

\[2\]

Agent Skills 详解：是什么？怎么用？和 Prompt、MCP 有什么区别？: *[https://javaguide.cn/ai/agent/skills.html](https://javaguide.cn/ai/agent/skills.html)*

\[3\]

万字拆解 MCP，附带工程实践: *[https://javaguide.cn/ai/agent/mcp.html](https://javaguide.cn/ai/agent/mcp.html)*

**⭐️推荐阅读**:

- [《SpringAI 智能面试平台+RAG 知识库》](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247552320&idx=1&sn=a7e4e5a8d957446e6bb032d78b2fa5fb&scene=21#wechat_redirect)
- [万字拆解 LLM 运行机制：Token、上下文与采样参数](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247553608&idx=1&sn=c87156405d51922bf7214ceaff6848a8&scene=21#wechat_redirect)
- [一文搞懂 AI Agent 核心概念：Agent Loop、Context Engineering、Tools 注册](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247552979&idx=1&sn=d13c59f631a26d3e4ba35fe5d2fe06ff&scene=21#wechat_redirect)
- [万字详解 Agent 核心方式： ReAct、Reflection、A2A、Agentic Workflows](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247553026&idx=1&sn=158a1c17cc7ac0dabc39327764ac312f&scene=21#wechat_redirect)
- [万字详解 RAG 向量索引算法和向量数据库](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247553552&idx=1&sn=389d583388474c52cec7769f4b920459&scene=21#wechat_redirect)
- [JDK 26 正式发布，人已麻。。。](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247553463&idx=1&sn=6b988a85df41c7c84c0b405bd54305b0&scene=21#wechat_redirect)

AI 核心技术与面试实战 · 目录

阅读原文

继续滑动看下一个

JavaGuide

向上滑动看下一个

文章目录