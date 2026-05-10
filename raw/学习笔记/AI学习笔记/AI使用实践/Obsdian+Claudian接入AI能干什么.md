可以把它理解成：

> **Obsidian 负责存放你的知识库，Claudian 让 AI Agent 直接进入这个知识库工作。**

它不是简单问答插件，而是让 Claude Code / Codex / Opencode 这类 agent 把你的 vault 当成工作目录，能读文件、写文件、搜索、改笔记、执行多步骤任务。官方说明里也强调了：vault 会变成 agent 的 working directory，支持文件读写、搜索、bash、多步骤 workflow、inline edit、skills、MCP、plan mode 等能力。([GitHub](https://github.com/YishenTu/claudian?utm_source=chatgpt.com "YishenTu/claudian: An Obsidian plugin that embeds ..."))

---

# 一、最适合你的用法：项目文档中枢

你现在做 DevWiki Studio / Dendro，已经有一堆文档：

```text
0_PROJECT_CONTEXT.md
PRD.md
ARCHITECTURE.md
DATA_MODEL.md
ROADMAP.md
CODEX_PROMPT.md
AGENTS.md
开发记录.md
问题清单.md
```

以前你的工作流可能是：

```text
Obsidian 写文档
↓
复制给 ChatGPT / Codex
↓
Codex 改项目
↓
你再把结果整理回 Obsidian
```

用了 Claudian 之后，可以变成：

```text
Obsidian vault
↓
Claudian 调 AI Agent
↓
AI 直接读取多篇项目文档
↓
AI 修改文档、整理计划、生成 prompt、沉淀决策
```

这对你很有用，因为你现在最大的问题不是“单点代码怎么写”，而是：

```text
产品想法很多
文档很多
Codex 会话很多
决策容易散
上下文容易丢
```

Claudian 正好适合做 **项目上下文管理器**。

---

# 二、具体能干哪些事？

## 1. 整理项目文档

比如你可以让它：

```text
阅读当前 vault 中 DevWiki Studio 相关文档，
找出 PRD、ARCHITECTURE、ROADMAP 之间不一致的地方，
列出冲突点，并给出修改建议。
```

它可以帮你发现：

```text
PRD 里说 MVP 支持 Anthropic + OpenAI
ARCHITECTURE 里只写了 OpenAI-compatible
ROADMAP 里又写了 Ollama
```

这种“文档之间的漂移”，人工看很累，AI 很适合做。

---

## 2. 生成 Codex / Claude Code 的开发 prompt

你可以在 Obsidian 里维护一个目录：

```text
Prompts/
  01-项目初始化.md
  02-前端重构.md
  03-后端接口.md
  04-前后端联调.md
  05-测试与CI.md
```

然后让 Claudian：

```text
根据 PRD.md、ARCHITECTURE.md、DATA_MODEL.md 和当前问题清单，
生成一份可以直接喂给 Codex 的 prompt。
要求：
1. 目标明确
2. 输入文件列表明确
3. 禁止大规模重写
4. 输出要求包含测试、提交说明、风险说明
```

这和你现在的工作流高度匹配。

---

## 3. 做知识库重构

你的技术学习笔记后面会越来越多：

```text
Java21/
Redis/
Spring/
DDD/
Gateway/
AI工程化/
CUDA/
LLM Wiki/
```

Claudian 可以帮你做：

```text
扫描 Java21 目录，
把虚拟线程、结构化并发、ScopedValue、Record Pattern 这些笔记整理成一个学习路径。
生成：
1. 目录索引
2. 推荐阅读顺序
3. 每篇笔记的摘要
4. 缺失知识点
```

这就很像你自己正在做的 **DevWiki / Dendro Wiki** 的原型体验。

---

## 4. 直接改笔记

Claudian 支持 inline edit：选中一段内容，然后让 AI 改。README 里提到它可以在 Obsidian 里对选中文本做 inline edit，并展示 word-level diff。([GitHub](https://github.com/YishenTu/claudian?utm_source=chatgpt.com "YishenTu/claudian: An Obsidian plugin that embeds ..."))

适合这些场景：

```text
把这段话改得更像技术文档
把这段 prompt 改得更适合 Codex
把这段架构说明补充 Mermaid 图
把这段中文改成 README 风格
把这段学习笔记改成面试表达
```

这个比普通 ChatGPT 方便，因为不用复制粘贴，直接在笔记里改。

---

## 5. 做技术学习助教

比如你有一篇：

```text
Redis/Redis Cluster Gossip协议.md
```

你可以让它：

```text
基于这篇笔记，生成：
1. 10 个面试问题
2. 每个问题的参考答案
3. 易错点
4. 企业实践中的注意事项
```

或者：

```text
把这篇 Java 虚拟线程笔记改造成：
基础概念 → 原理 → 代码示例 → 适用场景 → 面试加分项 的结构。
```

也可以让它统一你所有学习笔记的模板。

---

## 6. 做产品经理助手

针对 DevWiki Studio，你可以让它长期维护这些文档：

```text
产品定位.md
用户画像.md
核心功能.md
MVP范围.md
竞品分析.md
版本路线图.md
```

常见指令：

```text
根据最近的想法，更新 MVP 范围。
要求：
- 不要扩大 v0.1 范围
- 把不确定内容放到 Backlog
- 输出变更摘要
```

或者：

```text
阅读 ROADMAP.md 和 问题清单.md，
帮我整理下一轮 Codex 开发任务，
按 P0 / P1 / P2 分级。
```

这个比让 Codex 直接看代码更稳，因为 Claudian 在 Obsidian 里负责“想清楚”，Codex 负责“落代码”。

---

## 7. 做会议记录 / 决策沉淀

如果你以后把每次和 ChatGPT、Codex、Claude Code 的讨论都沉淀到 Obsidian，可以让 Claudian 做：

```text
把今天的开发讨论整理成 ADR。
格式：
- 背景
- 决策
- 备选方案
- 取舍理由
- 后续行动
```

ADR 就是 Architecture Decision Record，适合记录技术决策。比如：

```text
ADR-001：MVP 阶段不支持 Ollama
ADR-002：后端 ORM 选择 MyBatis-Plus
ADR-003：前端编辑器选择 TipTap
ADR-004：Provider 抽象先支持 OpenAI-compatible + Anthropic
```

你的项目会越来越需要这种“决策可追溯性”。

---

# 三、推荐你这样组织 Obsidian Vault

可以建一个专门的项目空间：

```text
DevWiki Studio/
  00_Index.md
  01_Product/
    PRD.md
    Product_Positioning.md
    User_Stories.md
    Roadmap.md
  02_Architecture/
    ARCHITECTURE.md
    DATA_MODEL.md
    API_DESIGN.md
    ADR/
  03_Prompts/
    Codex_Backend.md
    Codex_Frontend.md
    Codex_Integration.md
    ClaudeCode_Review.md
  04_Logs/
    2026-05-09.md
    2026-05-10.md
  05_Backlog/
    Issues.md
    Ideas.md
    Risks.md
  06_Learning/
    DDD.md
    Java21.md
    Redis.md
    AI_Gateway.md
```

然后你可以把 Claudian 当成这个 vault 的“项目秘书 + 架构助理 + prompt 工程师”。

---

# 四、你可以直接试的高价值指令

## 1. 项目文档体检

```text
请阅读当前 vault 中 DevWiki Studio 相关文档。

目标：
找出 PRD、ARCHITECTURE、DATA_MODEL、ROADMAP、CODEX_PROMPT 之间的冲突、不一致、重复和缺失。

输出：
1. 文档一致性问题列表
2. 每个问题的影响
3. 推荐修改方案
4. 哪些内容应该进入 v0.1，哪些应该放入 Backlog

不要直接修改文件，先给计划。
```

---

## 2. 生成 Codex 开发 prompt

```text
请基于当前 DevWiki Studio 项目文档，生成一份可以直接交给 Codex 的开发 prompt。

目标：
完成 v0.1 的前后端核心闭环。

要求：
1. 明确让 Codex 先阅读哪些文件
2. 明确本轮开发范围
3. 明确禁止事项，比如不要大规模重写、不引入重依赖
4. 明确测试要求
5. 明确输出格式：修改摘要、测试结果、风险、后续建议

请把结果保存到 03_Prompts/Codex_v0.1_Core_Loop.md。
```

---

## 3. 把学习笔记整理成体系

```text
请扫描 Java21 目录下的所有笔记。

目标：
整理成一套 Java 21 学习路线。

输出到 Java21/00_Index.md，内容包括：
1. 推荐学习顺序
2. 每篇笔记摘要
3. 关键概念
4. 企业级应用场景
5. 面试加分项
6. 后续需要补充的主题
```

---

## 4. 做 ADR

```text
请根据当前关于 DevWiki Studio 的产品和技术文档，生成 ADR 目录。

先创建：
02_Architecture/ADR/0001-provider-abstraction.md

主题：
MVP 阶段 AI Provider 抽象策略。

要求包含：
- 背景
- 决策
- 不选 Ollama 的原因
- 为什么支持 OpenAI-compatible 和 Anthropic
- 风险
- 后续扩展点
```

---

# 五、使用时注意什么？

## 1. 不要一上来让它“全库重构”

不要这样：

```text
帮我整理整个 vault，优化所有笔记。
```

太宽泛，容易乱改。

更好的方式：

```text
只处理 DevWiki Studio/01_Product 目录。
先分析，不要修改。
输出修改计划。
```

---

## 2. 先 Plan Mode，再执行

Claudian 支持 Plan Mode，官方说明里说可以让 AI 先探索和设计，再给计划。([GitHub](https://github.com/YishenTu/claudian?utm_source=chatgpt.com "YishenTu/claudian: An Obsidian plugin that embeds ..."))

你应该经常这样用：

```text
先进入 Plan Mode。
先不要改文件。
先列出你准备读取哪些文件、发现了什么问题、准备怎么改。
等我确认后再执行。
```

这和 Claude Code / Codex 的使用原则一样：**先收敛范围，再让它动手。**

---

## 3. 建议加 Git

如果你的 Obsidian vault 里放项目文档，强烈建议用 Git 管理。

因为 Claudian 能写文件，写错了可以回滚：

```bash
git init
git add .
git commit -m "init vault docs"
```

之后每次让 AI 大改前，先 commit 一次。

---

## 4. 注意自动生成目录污染

我看到 GitHub issue 里有人提到，Claudian / Claude Code skills 集成可能会自动生成类似 `01 Claude Skills` 的目录。这个不一定是你当前版本一定会遇到的问题，但如果 vault 根目录出现这类文件夹，可以考虑在 Obsidian 里隐藏，或者加到 `.gitignore`。([GitHub](https://github.com/YishenTu/claudian/issues/345?utm_source=chatgpt.com "Bug: \"01 Claude Skills\" folder auto-recreates repeatedly ..."))

---

# 六、对你的最佳定位

对你来说，Claudian 最适合承担这个角色：

```text
ChatGPT：讲解、分析、方案设计
Obsidian：长期知识库和项目文档
Claudian：让 AI 直接维护知识库和项目上下文
Codex / Claude Code：真正改代码、提 PR
ccswitch：模型路由和成本控制
```

推荐工作流：

```text
1. 在 Obsidian 写想法和问题
2. 用 Claudian 整理成 PRD / 架构 / Prompt
3. 把 Prompt 交给 Codex / Claude Code 改代码
4. 把结果回填到 Obsidian
5. 用 Claudian 维护 Roadmap、ADR、开发日志
```

一句话：

> **Claudian 不是用来替代 Codex 写代码的，它更适合做你的“Obsidian 项目大脑”，负责管理长期上下文、文档、prompt、路线图和技术决策。**