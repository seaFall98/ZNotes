**LangChain 的 `InMemoryStore` 是通用 key-value store；LangGraph 的 `InMemoryStore` 是给 LangGraph persistence / long-term memory 用的 namespaced JSON store，并且可选支持语义检索。**

它们名字一样，但不是同一个抽象。

## 1. LangChain 的 `InMemoryStore`

常见导入：

```python
from langchain_core.stores import InMemoryStore
```

它是一个很普通的内存 KV 存储：

```python
store = InMemoryStore()
store.mset([("user:1", {"name": "Bob"})])
value = store.mget(["user:1"])
```

特点：

|项|LangChain `InMemoryStore`|
|---|---|
|包|`langchain_core.stores`|
|定位|通用 key-value store|
|key|通常是字符串|
|value|任意 Python 对象|
|API 风格|`mget` / `mset` / `mdelete` / `yield_keys`|
|是否专门服务 LangGraph memory|不是|
|是否有 namespace|没有 LangGraph 那种 tuple namespace|
|是否支持 semantic search|通常不支持|
|典型用途|缓存、简单 KV、retriever/docstore 辅助存储、测试|

官方 reference 对它的描述就是 “in-memory store for any type of data”。([reference.langchain.com](https://reference.langchain.com/python/langchain-core/stores/InMemoryStore?utm_source=chatgpt.com "InMemoryStore | langchain_core"))

## 2. LangGraph 的 `InMemoryStore`

常见导入：

```python
from langgraph.store.memory import InMemoryStore
```

它是 LangGraph 的 **Store** 实现，主要用于 long-term memory：

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

namespace = ("user-123", "memories")

store.put(
    namespace,
    "profile",
    {"name": "Bob", "likes": ["short answers"]}
)

item = store.get(namespace, "profile")
items = store.search(namespace)
```

特点：

|项|LangGraph `InMemoryStore`|
|---|---|
|包|`langgraph.store.memory`|
|定位|LangGraph long-term memory / persistence store|
|key|namespace + key|
|value|通常是 JSON-like dict|
|API 风格|`put` / `get` / `search` / `delete`|
|namespace|有，通常是 tuple，比如 `("user_id", "memories")`|
|semantic search|可选支持，需要配置 embeddings / index|
|典型用途|跨 thread 的用户偏好、事实、长期记忆、共享知识|
|生产建议|官方建议生产用 DB-backed store，不要用内存版|

LangGraph 文档明确说：long-term memory 会作为 JSON documents 存在 store 中，每条 memory 由自定义 `namespace` 和 `key` 组织；`InMemoryStore` 只是保存到内存字典，生产环境应使用数据库支持的 store。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/memory "Memory overview - Docs by LangChain"))

## 3. 它和 Checkpointer 也不是一回事

LangGraph 里容易混淆的还有：

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.store.memory import InMemoryStore
```

区别是：

|名称|用途|
|---|---|
|`InMemorySaver`|保存 graph state checkpoint，短期记忆，thread-scoped|
|`InMemoryStore`|保存应用定义数据，长期记忆，cross-thread|

LangGraph 官方 persistence 文档说得很清楚：checkpointer 保存 thread 的 graph state；store 保存 graph state 之外的应用数据。checkpointer 用于短期、thread-scoped memory；store 用于长期、cross-thread memory，例如用户偏好、事实、共享知识。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/persistence "Persistence - Docs by LangChain"))

## 4. 最关键的 API 差异

### LangChain core store

```python
from langchain_core.stores import InMemoryStore

store = InMemoryStore()

store.mset([
    ("a", {"x": 1}),
    ("b", {"x": 2}),
])

print(store.mget(["a", "b"]))
```

它是平铺的 KV。

### LangGraph store

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()

namespace = ("user-1", "memories")

store.put(namespace, "a", {"x": 1})
store.put(namespace, "b", {"x": 2})

print(store.get(namespace, "a"))
print(store.search(namespace))
```

它是：

```text
namespace -> key -> JSON document
```

也就是说，LangGraph 的 store 天然适合这种结构：

```text
("user-1", "memories") / "preference-001"
("user-1", "profile")  / "main"
("org-9", "rules")     / "coding-style"
```

## 5. 什么时候用哪个？

### 用 LangChain `InMemoryStore`

当你只是想要一个简单内存 KV：

```text
缓存
临时保存对象
测试 retriever/docstore
非 LangGraph 场景
不需要 namespace/search/memory 语义
```

### 用 LangGraph `InMemoryStore`

当你在写 LangGraph agent / graph，并且需要：

```text
长期记忆
跨 thread 共享数据
用户偏好
profile
facts
semantic memory search
在 node 里通过 store 读写 memory
```

例如：

```python
graph = builder.compile(
    checkpointer=checkpointer,
    store=store,
)
```

LangGraph 文档里的标准用法也是把 `InMemoryStore` 传给 `builder.compile(..., store=store)`。([LangChain 文档](https://docs.langchain.com/oss/python/langgraph/persistence "Persistence - Docs by LangChain"))

## 一句话记忆

**`langchain_core.stores.InMemoryStore` 是普通内存 KV。**

**`langgraph.store.memory.InMemoryStore` 是 LangGraph 的长期记忆 store：有 namespace、有 search，可接 semantic index，专门服务 agent memory / persistence。**

所以，如果你在写 LangGraph agent，通常用：

```python
from langgraph.store.memory import InMemoryStore
```

如果你只是要一个普通 KV 存储，才用：

```python
from langchain_core.stores import InMemoryStore
```