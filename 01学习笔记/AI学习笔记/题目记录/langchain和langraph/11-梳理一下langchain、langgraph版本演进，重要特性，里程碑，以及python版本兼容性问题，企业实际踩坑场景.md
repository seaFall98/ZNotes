

---

## 一、LangChain 版本演进与里程碑

- **2022年10月**：LangChain 诞生（约 800 行代码的单文件）。
- **2022年12月**：引入基于 ReAct 论文的通用 Agent。
- **2023年1月**：适配 OpenAI Chat Completion API；JS 版本发布。
- **2023年2月**：LangChain Inc. 正式成立。
- **2023年3月**：适配 OpenAI Function Calling。
- **2023年6月**：LangSmith 发布。
- **2024年1月：v0.1.0 发布**，走向稳定。
- **2024年9月：v0.3.0 发布**，核心变更为 Pydantic v1 升级至 v2，停止支持 Python 3.8。
- **2025年10月22日：v1.0.0 发布**（与 LangGraph 1.0 同步），从零开始的重写，放弃旧 Chain 设计，统一为基于 LangGraph 的 `create_agent` 抽象，引入 Middleware 机制。
- **后续迭代**：v1.1.0（摘要中间件）、v1.3.1（最新，要求 Python >= 3.10）。

---

## 二、LangGraph 版本演进与里程碑

| 版本 | **确切发布时间** | 核心特性 |
|------|----------------|----------|
| v0.1.x ~ v0.5.x | 2024年—2025年初 | 早期图引擎迭代，奠定状态图、Checkpointer、中断恢复基础 |
| **v0.6.0** | **2025年7月28日**（官方 PyPI 发布时间） | **Context API（运行时上下文）重磅登场**。引入 `context_schema` 和 `Runtime` 对象，实现类型安全的上下文传递，取代旧有的多层嵌套字典配置方式 |
| **v1.0.0** | **2025年10月22日** | 与 LangChain 1.0 同步发布，API 稳定性锁定，`create_react_agent` 被弃用（统一为 LangChain 的 `create_agent`），引入类型化 Interrupts（中断）机制 |
| v1.1.0 | 2026年3月 | 类型安全流式处理（version="v2"）、Pydantic/dataclass 自动强制转换 |
| v1.2.7 | 2026年6月 | 最新版本，requires-python >= 3.10 |

### 关于 v0.6 的 **Context API（Runtime）** 纠正说明

我之前只说“动态模型选择”是错的，v0.6 的 Context API 核心解决的是**配置传递的“字典地狱”问题**：

- **旧方式（v0.5 及以前）**：通过 `RunnableConfig` 中的 `configurable` 字典层层嵌套传递业务参数（如 `user_id`），毫无类型提示，极易拼写错误。
- **新方式（v0.6+）**：定义 `Context` 数据类，在节点函数中通过 `Runtime[Context]` 对象直接访问，IDE 自动补全，类型安全。

```python
from dataclasses import dataclass
from langgraph.runtime import Runtime

@dataclass
class Context:
    user_id: str
    tenant_id: str

def my_node(state: State, runtime: Runtime[Context]):
    # 不再是 config["configurable"]["user_id"]，而是直接类型安全访问
    print(runtime.context.user_id) 
```

---

## 三、Python 版本兼容性

| 框架 | 版本 | Python 要求 |
|------|------|-------------|
| LangChain | v0.1.x | 3.8+ |
| LangChain | **v0.3** | **3.9+**（放弃 3.8，因 3.8 于 2024-10 EOL） |
| LangChain | **v1.0+** | **3.10+**（3.9 于 2025-10 EOL） |
| LangGraph | **v1.0+** | **3.10+** |
| LangGraph Platform | 部署环境 | 仅支持 3.11 和 3.12 |

---

## 四、企业实际踩坑场景

1. **Chain 迁移之痛（0.x → 1.0）**：大量基于旧 `LLMChain` 的代码在升级 1.0 时需完全重写为 `create_agent` 或自定义 Graph。
2. **Checkpointer 存储膨胀（LangGraph 通用）**：长时间运行 Agent 的状态快照导致数据库写放大（实测 1.5MB 写入膨胀至 15倍存储），需设计定期清理或摘要压缩策略。
3. **多 Agent 互相干扰**：指令竞态、状态幻觉、无限辩论循环。
4. **上下文溢出**：长文档（如代码 PR）直接超限，需引入摘要中间件（LangChain v1.1 加强了此能力）。
5. **PII 数据泄露**：通过 Middleware 注入脱敏逻辑。
6. **静态图过度设计**：23个节点、47条边，业务变更牵一发动全身，应评估是否真的需要复杂 Graph。
7. **依赖地狱（高频踩坑）**：
   - `langgraph < 0.7.0` 的约束会阻止 LangChain 1.0 安装。
   - `langchain-community` 不遵循 SemVer，盲目升级必炸。
   - 必须使用 `poetry` 或 `uv` 严格锁定全平台版本。
8. **性能权衡**：有实测 LangGraph 比纯 LCEL 慢约 68%，控制力换性能，需按场景取舍。

---

## 五、最终建议（重新校准）

- **新项目**：**必须** Python 3.10+，**必须** LangChain 1.0+ & LangGraph 1.0+。
- **遗留 0.3 项目**：仅接受安全补丁，不建议新增业务逻辑，尽早规划迁移。
- **LangGraph 使用者**：务必拥抱 v0.6+ 的 `Runtime` Context API，告别配置字典。

以上内容已基于官方 PyPI 发布日志和 GitHub Release Notes 严格校准。如有其他疑问，欢迎继续指出。