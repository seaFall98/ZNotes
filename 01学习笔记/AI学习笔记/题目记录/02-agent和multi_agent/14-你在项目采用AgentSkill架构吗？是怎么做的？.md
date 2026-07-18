
2025 年 10 月中旬，Anthropic 正式发布 Claude Skills。两个月后，Agent Skills 作为开放标准被进一步发布，我觉得：是诞生了一种新的 AI Agent 项目开发方案。单 Agent 可通过加载不同的 Skills 包，来具备不同的专业知识、工具使用能力，稳定完成特定任务。我的理解：可以把 Skills 理解为**“通用 Agent 的扩展包”**，所以整个 Agent 项目，可以采用模块化、技能化的封装，这种 Agent 方案，可以这极大减少了上下文长度，降低了模型的认知负荷，使工具选择更加精准和快速。

但是，目前 Skills 主要用于各种平台工具，像 Claude code 和 Cursor 平台，在实际的企业项目中，应该需要根据业务来自己定义 Skills，并和 langgraph 的 Agent 整合，来完可定制化的项目代码落地。我的实现步骤是：

1. 定义 Skill 技能包的配置文件，里面包括当前 Skill 的提示词和可调用的工具名列表（文本）
2. 结合 langchain 框架，开发一个 load_skills 工具，该工具必须传入一个参数 (Skill 的名字)，返回 Agent 从 Skills 列表中意图识别 选择的 某一个 Skills 的配置信息，并把选中的 Skill 记录到 Agent 的状态中，Agent 根据上下文和用户指令来做出意图决策。
3. 自定义一个 SkillMiddleware，它在每次模型调用前拦截请求，并执行核心逻辑：检查当前状态中已加载的技能，然后根据技能 配置工具映射关系，动态构建一个仅包含所需工具的列表，加载该列表，并替代原始的全量工具列表。
4. 创建 Agent，初始时仅暴露 Load_skills 或者其他少数基础工具。将自定义的 SkillMiddleware 整合到 Agent 中。并设置清晰的系统提示词，引导智能体遵循“先从 Skills 列表中加载技能，后使用工具”的工作流程。

我这几个步骤关键在于遵循了“状态管理 → 动态路由 → 按需加载”的工程范式，Agent Skills 架构能有效解决复杂智能体系统中的工具管理难题，未来可能是一种模块化开发 Agent 的模式。

---

**update : 而实际上，现在deepagent已经支持skill了，这些步骤已经封装好，开箱即用了**

---

## 一、起源：Anthropic的Agent Skills

### 1.1 设计初衷

2025年10月，Anthropic在Claude Code中首次引入Agent Skills概念。同年12月18日，Anthropic宣布将Agent Skills作为**开放标准**发布。

这一设计的核心洞察是：**通用型Agent虽然推理能力强，但缺乏特定领域的经验积累**。Skills正是为了解决这个问题——将领域知识、工作流、最佳实践和脚本打包成Agent可以按需发现和应用的标准化能力单元。

### 1.2 核心设计：渐进式披露（Progressive Disclosure）

Agent Skills最核心的设计是**三层渐进式披露**机制：

```
┌─────────────────────────────────────────────────────────────────┐
│              渐进式披露（Progressive Disclosure）               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  第一层：元数据 (Metadata)                              │    │
│  │  • 启动时预加载所有技能的 name + description            │    │
│  │  • 仅占数十个token                        │    │
│  │  • 格式：SKILL.md 的 YAML frontmatter    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           ↓                                     │
│            Agent判断技能是否与当前任务相关                       │
│                           ↓                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  第二层：技能主体 (SKILL.md 完整内容)                  │    │
│  │  • 触发时加载完整的 Markdown 指令        │    │
│  │  • 含详细步骤、示例、注意事项                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           ↓                                     │
│               SKILL.md 中引用需要额外资源                       │
│                           ↓                                     │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  第三层：附加文件和脚本                                │    │
│  │  • Python脚本、参考文档、模板等          │    │
│  │  • 仅在需要时加载或执行                               │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

这种设计的精妙之处在于：Agent可以**配备数百个技能**而不会撑爆上下文窗口。无关任务时，Agent仅加载技能元数据，几乎不占用上下文；一旦识别出相关技能，再逐步加载详细内容。

### 1.3 SKILL.md格式

每个Skill是一个文件夹，核心是`SKILL.md`文件：

```markdown
---
name: pdf-processing
description: Extract and process text from PDF documents, handle forms, and generate PDF reports
---

# PDF Processing Skill

## When to use
Use this skill when the user asks to extract text from PDFs, fill PDF forms, or generate PDF documents.

## Instructions
1. Use `pypdf` to extract text content
2. For forms, use `pdfrw` to fill form fields
3. For generation, use `reportlab` to create new PDFs

## References
- See [forms.md](./forms.md) for form-filling detailed guide
- See [examples.md](./examples.md) for common use cases
```

### 1.4 演进脉络

Agent Skills的演进有几个关键节点：

| 时间 | 里程碑 |
|------|--------|
| 2025年10月 | Anthropic在Claude Code中引入Agent Skills概念 |
| 2025年12月18日 | 宣布为开放标准，建立独立GitHub仓库 |
| 2025年12月 | OpenAI在Codex中支持Skills |
| 2026年 | Microsoft(VS Code/GitHub)、Cursor、Goose等20+平台采纳 |
| 2026年3月 | LangChain推出LangChain Skills |
| 2026年5月 | LangChain推出Interpreter Skills |

值得注意的是，**Commands在2025年10月已被合并进Skills**——`.claude/commands/x.md`和`.claude/skills/x/SKILL.md`行为一致，Skills是推荐的超集。


## 二、LangChain生态中的Skill体系

### 2.1 LangChain Skills

2026年3月，LangChain正式发布了LangChain Skills。其核心理念与Anthropic一致：Skills是**精选的指令、脚本和资源**，通过渐进式披露动态加载。

LangChain Skills仓库维护了**11个技能**，分为三大类：

- **LangChain类**：`create_agent()`、middleware、tool patterns
- **LangGraph类**：StateGraph、Human-in-the-Loop、持久化执行
- **Deep Agents类**：预置middleware、FileSystem集成

实际效果数据令人震撼：在LangChain生态任务评估中，Claude Code不加Skills时通过率仅**25%**，加入Skills后飙升至**95%**。

### 2.2 LangGraph中的Skill集成

LangGraph作为**状态机编排框架**，与Skill体系天然契合。Skills在LangGraph中的核心价值在于：**有状态的模式（如Handoffs、Skills）在重复请求上可节省40-50%的调用次数**。

Skills在LangGraph中的集成方式主要有两种：

1. **作为状态节点**：Skill可以是一个LangGraph节点，通过状态管理实现渐进式加载
2. **通过Middleware注入**：LangGraph的middleware机制允许在系统提示中动态注入Skill元数据

### 2.3 Deep Agents：官方Skill实现

**Deep Agents是LangChain官方推出的Agent Harness**，内置了对Skills的一流支持。其核心设计是：Skills将领域专业知识（如工作流、最佳实践、脚本和参考文档）打包成可复用的目录，Agent仅在相关时发现和读取。

#### SkillsMiddleware源码分析

Deep Agents通过`SkillsMiddleware`实现Anthropic的渐进式披露模式：

```python
from deepagents.middleware.skills import SkillsMiddleware
from deepagents.backends.filesystem import FilesystemBackend

middleware = SkillsMiddleware(
    backend=FilesystemBackend(root_dir="/path/to/skills"),
    sources=[
        "/path/to/skills/user/",      # 用户级技能
        "/path/to/skills/project/",   # 项目级技能
        # 后面的覆盖前面的
    ],
)
```

SkillsMiddleware的核心机制：
- 从backend sources加载技能
- 使用渐进式披露注入系统提示（先metadata，按需加载完整内容）
- 按source顺序加载，后面的覆盖前面的

#### 子Agent独立Skill集

Subagent可以拥有**独立的Skill集合**，不与主Agent共享：

```python
subagent_config = {
    "skills_dirs": ["./skills/subagent/"],  # 子Agent专用Skills
}
```

#### 完整代码示例

以下是一个使用Deep Agents创建带Skills的Agent的完整示例：

```python
# 安装: pip install deepagents langchain langchain-openai

import os
from langchain_openai import ChatOpenAI
from deepagents import create_deep_agent
from deepagents.middleware.skills import SkillsMiddleware
from deepagents.backends import FilesystemBackend

# 1. 定义Skill目录结构
# ./skills/
#   └── data-analysis/
#       ├── SKILL.md
#       └── analysis.py

# 2. 创建Skills中间件
skills_middleware = SkillsMiddleware(
    backend=FilesystemBackend(root_dir="./skills/")
)

# 3. 创建Deep Agent
model = ChatOpenAI(model="gpt-4")
agent = create_deep_agent(
    model=model,
    system_prompt="You are a helpful data analysis assistant.",
    middleware=[skills_middleware],  # 注入Skills
)

# 4. 调用Agent - 会自动发现并加载相关Skill
result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": "Analyze the sales data from last quarter and generate a report"
    }]
})

print(result["messages"][-1].content)
```

**关键机制**：启动时，Deep Agents读取每个SKILL.md的name和description；当用户请求与某Skill相关时，Agent自动读取完整SKILL.md并执行。

### 2.4 Interpreter Skills（2026年5月最新演进）

2026年5月，LangChain推出了**Interpreter Skills**——Skill体系的重大演进。

[langchain skills and interpreter](https://www.langchain.com/blog/interpreter-skills)官方文章的" Can't I just include a script file?"特意强调了跟/scripts下的脚本代码的区别

[[LangChain 让 Agent 的技能不再只靠提示词：Interpreter Skills 把确定性写进代码 - 智能体工程 - 创艺提示符]]
#### 传统Skill vs Interpreter Skill

传统Skill的本质是**提示词驱动的 specialization**——告诉Agent"怎么做"，希望Agent遵循指令。Interpreter Skill则**直接附带可执行的TypeScript模块**，Agent可以在解释器中导入并运行。

````markdown
---
name: github-triage
description: Triage GitHub issues, PRs, and discussions
metadata:
  module: ./index.ts
---

Use this skill when a user asks for repository triage.
Import the module using the interpreter and call `triage(repo, options)`:

```ts
const { triage } = await import("@/skills/github-triage");
const result = await triage("langchain-ai/deepagents", { 
  issues: true, 
  prs: true 
});
result.toMarkdown();
```
````

#### 核心区别

| 维度 | 传统Script Skill | Interpreter Skill |
|------|------------------|-------------------|
| **执行方式** | 模型读取指令→理解→执行 | 模型判断触发→**直接import执行代码模块** |
| **确定性** | 依赖模型"正确理解"，每次可能不同 | **确定性逻辑在代码中**，可测试、可迭代 |
| **与Agent交互** | 脚本独立运行 | **代码可以直接操控Agent**——spawn子Agent、管理任务图、处理部分失败 |
| **可评估性** | "Agent是否遵循了指令？" | "是否调用了预期函数？" |

Interpreter Skill解决的核心矛盾是：**提示词能描述意图，但保不住执行路径**。模型只负责判断"现在该用这个Skill了"，一旦判断完成，**整个执行路径由代码100%控制**。

#### 为什么选择TypeScript？

Interpreter运行在**QuickJS**——一个轻量级的嵌入式JavaScript/TypeScript运行时。它被设计为**默认全关、逐个放行**的安全模型：
- 默认无文件系统访问
- 默认无网络访问
- 默认无shell执行
- 工具调用需通过`atools`命名空间显式暴露


## 三、Skill体系的设计好处

### 3.1 上下文效率（Token Efficiency）

传统方式需要将所有工具和指令一次性塞入上下文，很快撑爆。Skill通过渐进式披露解决了这个问题——**只有YAML frontmatter默认加载，Agent仅在需要时读取完整SKILL.md**。每个Skill在摘要状态下仅占数十个token。

### 3.2 可组合性与可复用性

Skills是**可移植、可共享**的——由Markdown文件和脚本组成。不同团队可以独立开发和维护Skills。LangSmith Fleet甚至可以在创建新Agent时**自动生成相关Skills**。

### 3.3 性能提升

实证数据最有说服力：
- LangChain Skills将Claude Code在生态任务上的通过率从**25%提升到95%**
- 有状态的Skills模式在重复请求上节省**40-50%的调用次数**
- 行业调研显示，采用技能封装方案的企业平均将AI应用开发周期缩短**58%**，系统维护成本降低**35%**

### 3.4 轻量级组合

Skills比完整的Sub-agent更轻量。Skills与Subagents解决不同问题：
- **Skill**：通过渐进式披露加载到当前上下文的**知识/流程**
- **Subagent**：独立的上下文窗口，返回一个**摘要结果**


## 四、实际使用方法

### 4.1 创建Skill

最小可行结构：

```
my-skill/
├── SKILL.md          # 必需：YAML frontmatter + 指令
├── reference.md      # 可选：参考文档
└── script.py         # 可选：可执行脚本
```

SKILL.md模板：

```markdown
---
name: sql-query-builder
description: Build optimized SQL queries from natural language questions
---

# SQL Query Builder

## When to use
Use when user asks to query a database in natural language.

## Instructions
1. Understand the user's data question
2. Identify relevant tables and columns from schema
3. Build the SQL query with proper JOINs and WHERE clauses
4. Optimize for performance (index usage, avoid SELECT *)
5. Return the query and explanation
```

### 4.2 安装LangChain Skills

使用`npx skills`命令安装：

```bash
# 安装所有LangChain Skills到当前项目
npx skills add langchain-ai/langchain-skills --skill '*' --yes

# 全局安装
npx skills add langchain-ai/langchain-skills --skill '*' --yes --global

# 链接到Claude Code
npx skills add langchain-ai/langchain-skills --agent claude-code --skill '*' --yes --global
```

### 4.3 在Deep Agents中使用

```python
from deepagents import create_deep_agent

agent = create_deep_agent(
    model="your-model",
    skills_dirs=["./skills/"],  # 指定Skills目录
    # 或通过middleware更精细控制
)
```

Subagent使用独立Skills：

```python
agent = create_deep_agent(
    model="your-model",
    skills_dirs=["./skills/main/"],
    subagents=[{
        "name": "data-analyzer",
        "skills_dirs": ["./skills/analyzer/"],  # 子Agent专用
        # ...
    }]
)
```

### 4.4 与其他技术组合

实际工程中，Skills常与MCP等技术组合使用：

```
┌─────────────────────────────────────────────────────┐
│                    应用层                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │   Agent     │  │   Agent     │  │   Agent     │ │
│  │  (编排器)   │  │  (专家)     │  │  (专家)     │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘ │
│         │                │                │         │
│  ┌──────▼────────────────▼────────────────▼──────┐ │
│  │              Skills Layer                     │ │
│  │  (渐进式披露的领域知识 + 可执行脚本)           │ │
│  └──────┬────────────────────────────────────────┘ │
│         │                                          │
│  ┌──────▼────────────────────────────────────────┐ │
│  │              MCP Layer                        │ │
│  │  (标准化工具连接协议)                          │ │
│  └──────┬────────────────────────────────────────┘ │
│         │                                          │
│  ┌──────▼────────────────────────────────────────┐ │
│  │           外部系统 / 数据源                    │ │
│  └───────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```


## 五、总结

Agent Skill体系代表了一次**从"单体Agent"到"可组合能力"的范式转变**。

**核心设计**：渐进式披露的三层结构（元数据→完整指令→附加资源），让Agent可以携带数百个Skill而不撑爆上下文。

**技术演进**：从Anthropic的原始设计，到LangChain的生态化实现（11个技能，3大类），再到Deep Agents的生产级Harness和Interpreter Skills的代码化演进——**提示词定义意图，代码保障执行**。

**实际效果**：性能从25%提升到95%，开发周期缩短58%——这不是概念验证，而是已被验证的生产级范式。Skill正在成为2026年AI工程化最核心的抓手。