---
title: "Anthropic 工程师总结的 9 条血泪经验"
source: "https://mp.weixin.qq.com/s/rTn_xwnrfhmRPmbKfeh6Lw"
author:
  - "[[石臻]]"
published:
created: 2026-05-31
description:
tags:
  - "clippings"
---
*2026年5月28日 09:19*

⭐ 设为星标 · 第一时间收到推送

**导读：** Anthropic 内部在用数百个 Claude Code Skills，Thariq 这篇文章把他们踩过的坑、总结出的套路全写出来了。不是官方文档那种平铺直叙，是真正用过之后的经验。

最近想快速提升项目实战能力（包含多个AI项目），或者最近找工作，或者需要找人修改简历，或者想学习AI的小伙伴，可以看看下面👇🏻的这个链接（或许真的能够帮到你）:

[推荐一个牛逼的RAG + MCP系统](https://mp.weixin.qq.com/s?__biz=MzkwNjMwMTgzMQ==&mid=2247538241&idx=1&sn=73d443e9a5ce6ee3317e97e5ce213d07&scene=21#wechat_redirect)

Claude Code 的 Skills 功能火了很久，但大多数人只停在"会用"这一层。会写一个 SKILL.md、会把它放到.claude/skills 目录——然后就没了。

Anthropic 内部用了数百个 Skills，工程师 Thariq 最近把这段经验写成了长文。读下来，跟自己摸索的那套有挺多不一样的地方。

这篇文章提炼了其中我觉得最有价值的部分，同时把原文里所有案例都保留下来了。

![Thariq 关于 Claude Code Skills 的深度分享](https://mmbiz.qpic.cn/mmbiz_png/TkWsojtosvQicNonHeibG32icFdJAiatNwDYcCK7osneLEsDRrWpqOw77eSY9jJUsGgWh5iaarIXiaIhNFKPrCp1bIuChTV2edib7eJ81IILx2XDac/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=0)

Thariq 关于 Claude Code Skills 的深度分享

## Skills 不是 Markdown 文件，是文件夹

这是理解 Skills 的第一个关键转变。

很多人觉得 Skill 就是写一个 SKILL.md 文件，告诉 Claude 要做什么。Thariq 说，这是最常见的误解。Skills 是文件夹，不是文件。文件夹里可以放脚本、资产文件、引用文档、配置……整个文件系统都是上下文工程的一部分。

把 Skills 当文件夹来设计，而不是当 Markdown 来写——这个思路一变，能做到的事完全不同。

举个例子：如果你的 Skill 最终要输出一个 Markdown 报告，可以在 `assets/` 目录里放一个模板文件，让 Claude 直接复制和填充，而不是让它凭空生成格式。如果有函数签名和使用示例，可以单独放到 `references/api.md` ，需要的时候 Claude 自己去读。

这叫"渐进式披露"（progressive disclosure）——告诉 Claude 有哪些文件，它会在合适的时机去读，而不是一次全部塞进上下文里。

## 9 类 Skills，逐一拆解

Thariq 团队把内部所有 Skills 梳理归类，发现基本上能分成 9 类。最有意思的地方是，不少工程师写了一堆 Skills，但只覆盖了其中 2-3 类——有些场景根本没想到可以用 Skill 来解决。

![Claude Code Skills 的 9 大分类图](https://mmbiz.qpic.cn/sz_mmbiz_jpg/TkWsojtosvTXbYUxnjxWibv3uia5j9GHh32sXA6t7WJopHseJtNOa53Qu3DRjYBj9ccZESEMblhf8HgKlZo6Lr2enFImD8yWFTll3UtT6kUk0/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=1)

Claude Code Skills 的 9 大分类图

**① 知识 / 参考类（Knowledge & Reference）**

告诉 Claude 如何正确使用内部库、CLI 或 SDK。适合内部库，也适合 Claude 经常用错的外部库。这类 Skill 通常包含一个参考代码片段目录，以及一份 Claude 写这类代码时要主动避开的坑列表。

原文案例：

- `billing-lib`
	— 内部计费库的边界情况、各种 footgun
- `internal-platform-cli`
	— 内部 CLI 的每个子命令，附带"什么时候用哪个"的示例
- `frontend-design`
	— 让 Claude 更好地理解你的设计系统

**② 验证类（Verification）**

描述如何测试或验证代码是否正确。通常搭配 Playwright、tmux 等外部工具做实际验证。Thariq 专门强调： **让工程师花一周时间把验证 Skills 做好，是值得的投资** 。可以考虑让 Claude 录制操作视频，或者在每一步用断言验证程序状态——这些都可以通过 Skill 里的脚本来实现。

原文案例：

- `signup-flow-driver`
	— 自动跑注册 → 邮箱验证 → onboarding 全流程，每步断言状态
- `checkout-verifier`
	— 用 Stripe 测试卡驱动结账 UI，验证发票状态
- `tmux-cli-driver`
	— 专门用于需要 TTY 的交互式 CLI 测试

**③ 数据访问类（Data Access）**

连接数据和监控系统。通常包含带凭证的数据获取库、仪表盘 ID 等，以及常见查询工作流说明。

原文案例：

- `funnel-query`
	— "我该 join 哪些事件才能看到 注册→激活→付费 的路径"，附带真实的 user\_id 表
- `cohort-compare`
	— 比较两个用户群的留存率或转化率，标记统计显著差异，链接到分群定义
- `grafana`
	— 数据源 UID、集群名称、"症状 → 对应仪表盘"查找表

**④ 自动化工作流类（Automation）**

把重复操作压缩成一条命令。指令通常比较简单，但可能依赖其他 Skills 或 MCP。Thariq 提示：把每次执行的结果存进日志文件，能帮模型在多次执行之间保持一致性，也方便它反思"上次干了什么"。

原文案例：

- `standup-post`
	— 汇总工单系统、GitHub 活动、前一天 Slack 内容，生成格式化日报，只写增量变化
- `create--ticket`
	— 执行字段 schema 校验（有效枚举值、必填字段），附带创建后工作流（通知审查人、同步 Slack）
- `weekly-recap`
	— 已合并 PR + 已关闭工单 + 部署记录 → 格式化周报

**⑤ 脚手架类（Scaffolding）**

为代码库的特定模块生成框架样板。这类 Skill 特别适合当你的脚手架里有自然语言要求、无法完全用代码表达的时候，可以配合可组合脚本一起用。

原文案例：

- `new--workflow`
	— 带注解的新服务/工作流/处理器脚手架
- `new-migration`
	— 数据库迁移文件模板，附带常见踩坑提示
- `create-app`
	— 预置好认证、日志、部署配置的内部应用模板

**⑥ 代码审查类（Code Review）**

执行代码质量检查。可以包含确定性脚本或工具以保证可靠性，也可以放到 Git Hook 或 GitHub Action 里自动触发。

原文案例：

- `adversarial-review`
	— 起一个独立子 agent 专门批评，实施修复，反复迭代直到问题只剩小细节
- `code-style`
	— 强制代码风格，尤其是 Claude 默认不会做好的那些
- `testing-practices`
	— 描述如何写测试、测什么

**⑦ 部署类（Deploy）**

拉取、推送、部署代码。这类 Skill 可能会引用其他 Skills 来收集数据。

原文案例：

- `babysit-pr`
	— 监控 PR → 重试 flaky CI → 解决合并冲突 → 开启自动合并
- `deploy-`
	— 构建 → 冒烟测试 → 灰度流量切换（持续对比错误率）→ 回归时自动回滚
- `cherry-pick-prod`
	— 独立 worktree → cherry-pick → 冲突处理 → 按模板提 PR

**⑧ 调试类（Debugging）**

接收症状（比如 Slack 消息、告警、错误签名），走一遍多工具调查流程，输出结构化排查报告。

原文案例：

- `-debugging`
	— 针对你们流量最高的服务，把"症状 → 工具 → 查询模式"全部映射出来
- `oncall-runner`
	— 拉取告警 → 排查常见嫌疑 → 整理发现
- `log-correlator`
	— 给一个 request ID，从所有可能经手的系统里拉取对应日志

**⑨ 运维类（Operations）**

执行例行维护和操作流程，尤其是涉及破坏性操作的场景——加上防护步骤，让工程师在关键操作上更容易遵守最佳实践。

原文案例：

- `-orphans`
	— 找出孤立的 pod/volume → 发到 Slack → 冷却期 → 用户确认 → 级联清理
- `dependency-management`
	— 你们组织的依赖审批流程
- `cost-investigation`
	— "为什么存储/流量费用突然暴涨"，附带具体的 bucket 和查询模式
![Skills 分类全览](https://mmbiz.qpic.cn/mmbiz_jpg/TkWsojtosvSib1ERa5abicLzRBeUVRECpARDXUf7Fs2yQiarosicSsNhU9gNXfXfTz0pHOKics7qgLowAYicqUYTMdMtYJIicLTpcRBpiaJMA6uk4G4/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=2)

Skills 分类全览

对照这 9 类，检查一下自己有没有盲点——很多人只写了知识类和脚手架类，验证类和运维类往往是最值钱但最容易被忽略的。

## Gotchas 是 Skill 里含金量最高的部分

写 Skill 时怎么做才最有价值？Thariq 给了几个具体建议，其中最直接的一条： **Gotchas 段落是整个 Skill 里信号最强的内容** 。

![Gotchas 示例截图](https://mmbiz.qpic.cn/sz_mmbiz_jpg/TkWsojtosvR9SqYoY3I4Wkms5P7sbZ218icYv9hMTic8iax7zWlskatggVHsy3cb2myRYVWsq2n8yTFiaKz89sfx4BsImibkebQQnUFSnibibic792Q/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=3)

Gotchas 示例截图

Gotchas 应该从 Claude 在使用这个 Skill 时真正踩过的坑里积累。每次遇到新的失败模式，就更新进去。

这背后的逻辑是：Claude Code 对代码库和通用编程已经知道很多了，如果你的 Skill 主要是重复它已经知道的东西，价值有限。 **真正有用的是把 Claude 的默认行为推到你的方向** ——不管是设计品味、组织规范，还是特定的边界情况。

Thariq 举了一个 Anthropic 内部的"设计品味 Skill"的例子：一个工程师通过反复和客户迭代，让 Claude 避免常见的设计陋习（比如 Inter 字体加紫色渐变），形成了一套独到的设计偏好库。

![设计品味 Skill 示例](https://mmbiz.qpic.cn/mmbiz_jpg/TkWsojtosvSbg43JMgvwPJysIzbiaib6tZMnNg4uQSWAjBH4o8959NeNh6Go42yULyLdbX0garibj6njhA0CE5rUNvEr4arn7nsnUPnPf2K5Ys/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=4)

设计品味 Skill 示例

## 指令别写死，给 Claude 留空间

这个点有点反直觉。我们写 Skill 的本能是"写得越详细越好"，但 Thariq 说恰恰相反—— **Skills 复用率很高，太具体的指令反而会让它在边缘情况下变死板** 。

给 Claude 它需要的信息，但留足灵活度去适应具体情境。下面两张图直接对比了过于具体 vs 恰当灵活的写法：

![指令过于具体的反例](https://mmbiz.qpic.cn/sz_mmbiz_jpg/TkWsojtosvT8q5odFez30r44Tgib5OLqb4Eg2B8GMQFDBdDhNVoFIORqOqniboUibb2GWLjcnNGtyWzvQMyJhs4VcK5lCXzbH83lLyq2GssKZs/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=5)

比较典型的场景是需要用户输入的 Skill。比如一个发日报到 Slack 的 Skill，你可能需要先问用户发到哪个频道。做法是把这些配置存到 `config.json` 文件里——如果配置不存在，Claude 就问用户；如果已有配置，直接用。如果要呈现结构化多选题，可以指示 Claude 使用 AskUserQuestion 工具。

![config.json 配置模式示例](https://mmbiz.qpic.cn/mmbiz_jpg/TkWsojtosvQws4rNtruIX2LAPIDcE8m7JiaHBh3fqrZiaTiaSoYS2ghVGwdEB3UEn1LxKqLRVNC3zSNG2hJ3nnMicCfmFKbibBJqwyicX9Sf2iahYk/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=7)

config.json 配置模式示例

## description 字段决定了 Skill 能不能被触发

这里有个细节经常被忽略。

Claude Code 启动时会把所有可用 Skill 的 description 读进来，形成一个索引。每次用户发请求，Claude 扫描这个索引来判断"有没有适合这个任务的 Skill"。

所以 **description 不是说明，是触发条件** 。它应该描述"什么情况下应该用这个 Skill"，而不是"这个 Skill 能做什么"。

![description 字段触发词示例](https://mmbiz.qpic.cn/mmbiz_jpg/TkWsojtosvTW6GM3zLiaaJFXakV5YQ0xOH7IQ2JI1GODXExX5JOWaGATEpYodYUeM1k2pTf4o1Q14S7GRZVUTVQTJdFFJ8pDe3x9fVYKUKK4/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=8)

description 字段触发词示例

## 给 Claude 代码，让它只操心"做什么"

这是 Thariq 认为最强的 Skill 设计模式之一： **把脚本和工具库放进 Skill 里，让 Claude 专注于组合和决策，而不是重建样板代码** 。

举个例子：数据分析 Skill 里放一组从事件源取数据的 helper functions。Claude 动态生成脚本来调用这些函数，完成复杂分析任务，比如"上周二发生了什么"。

![Skill 中的 Helper Functions 示例](https://mmbiz.qpic.cn/sz_mmbiz_jpg/TkWsojtosvSYXZLLiaEQg4H5tAov5IzlgCuLRb5icZaZoSjCdUVwmdwrA2QxjRJbn6uYfwpSl99hMp6uta5DubmaPrFUKnlEQKHbicyJicexW20/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=10)

Skill 中的 Helper Functions 示例

Claude 的每一步都在思考"接下来做什么"，而不是在写取数函数。工作效率的提升非常直接。

## Skill 可以有自己的"记忆"

这个特性很多人不知道。

Skills 可以把数据存在自己目录里——简单的可以是一个 append-only 文本日志，复杂的可以是 SQLite 数据库。这让 Skill 有了跨会话的记忆。

比如 `standup-post` Skill 可以维护一个 `standups.log` ，记录每次生成的日报内容。下次运行时，Claude 读取这份历史，就能判断"自上次以来发生了什么变化"，而不是每次从零开始。

![Skill 内存储数据的示意图](https://mmbiz.qpic.cn/sz_mmbiz_png/TkWsojtosvTgEvm8rjOAwYpamAVe8qLPKIVOH6TZTtZAv9bxOrcrpazUdsabsY3NhladYiarrbmiceDzqxFMef4zPwamkUZ0AXpSMhjDG97VQ/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=12)

Skill 内存储数据的示意图

需要注意的一点：存在 Skill 目录下的数据，升级 Skill 时可能被删掉。Thariq 建议把这类数据存到 `${CLAUDE_PLUGIN_DATA}` 提供的稳定目录下（每个插件独立一个目录）。

## Hooks 只在被调用时激活

Skills 可以内嵌 Hooks，但这些 Hooks 只在 Skill 被调用时生效，会话结束就消失。这个设计很适合"场景化防护"：

- `/careful`
	— 禁止 `rm -rf` 、 `DROP TABLE` 、强制推送、 `kubectl delete` ，触碰生产环境时用
- `/freeze`
	— 限制只能在特定目录下修改文件，调试时防止 Claude 误改不相关代码

如果这类 Hook 全局常驻，会烦死人。作为 Skill 按需激活，刚好。

## 在团队里分发 Skills

两种方式：把 Skills 提交到 repo 的 `.claude/skills` 目录，或者建内部插件市场。

前者适合小团队，简单直接。但每个 Skill 都会给模型上下文加一点负担，规模大了就需要插件市场——让团队自己选安装哪些。

Anthropic 内部的做法：没有中心化审核团队，而是靠自然传播。有好 Skill 就发到 GitHub 沙盒目录，在 Slack 里分享，自然获得使用和反馈。等它有了足够的关注度，技能作者自己提 PR 把它并入市场。

**注意：坏的或者重复的 Skills 很容易冒出来，上线前做好筛选机制很重要。**

Skills 之间可以有依赖，比如"CSV 生成 Skill"依赖"文件上传 Skill"。目前没有原生的依赖管理，但可以在 Skill 里直接引用其他 Skill 的名字，模型会在对方安装的情况下调用它。

![从个人到团队再到内部市场的 Skill 分发路径](https://mmbiz.qpic.cn/mmbiz_png/TkWsojtosvQKvUic2mu4e4DW2f2Mu6GpvR4bFI6cTiczYNzKcRJqctC3tU7TOkP9EekqhCkCrMSvib9nZjlR6J4uIrMB4kuQ8UuvWFduuKPEv4/640?from=appmsg&tp=webp&wxfrom=5&wx_lazy=1#imgIndex=13)

从个人到团队再到内部市场的 Skill 分发路径

想了解 Skill 的实际使用情况？Thariq 提到他们用 PreToolUse hook 记录公司内部每个 Skill 的调用日志，这样就能找出最受欢迎的 Skill，或者发现某个 Skill 触发频率远低于预期——这通常说明 description 没写好。

- **9 类 Skills**
	：知识、验证、数据访问、自动化、脚手架、代码审查、部署、调试、运维——逐一检查有没有盲点
- **文件夹思维**
	：脚本、模板、引用文档都可以放进去，渐进式披露给 Claude
- **Gotchas 优先**
	：记录 Claude 真正踩过的坑，比通用说明更值钱，随时更新
- **description 是触发器**
	：写"什么时候用"而不是"能做什么"
- **给 Claude 代码**
	：helper functions 比详细文字指令更能解放 Claude 的决策带宽
- **加上记忆**
	：用日志文件存历史，让 Skill 跨会话保持连续性
- **验证 Skills 值得专项投入**
	：让 Claude 能自己校验输出，质量差别很大

最近想快速提升项目实战能力（包含多个AI项目），或者最近找工作，或者需要找人修改简历，或者想学习AI的小伙伴，可以看看下面👇🏻的这个链接（或许真的能够帮到你）:

[推荐一个牛逼的RAG + MCP系统](https://mp.weixin.qq.com/s?__biz=MzkwNjMwMTgzMQ==&mid=2247538241&idx=1&sn=73d443e9a5ce6ee3317e97e5ce213d07&scene=21#wechat_redirect)

最近建了几个AI技术交流群，扫描加我微信，备注：AI，即可进群交流和学习，获取最新的AI讯息。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_jpg/ibJZVicC7nz5jN7lz3xCSbNeNj4yNXMvNX12Fgwkvnuibx4wj2uF89ehibUGMOT4DPHibO5dg44kXwBickPvORaQ8JGA/640?wx_fmt=jpeg&from=appmsg&wxfrom=5&wx_lazy=1&tp=webp#imgIndex=9)

继续滑动看下一个

苏三说技术

向上滑动看下一个

文章目录