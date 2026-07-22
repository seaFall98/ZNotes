
MCP Server 可以同时暴露三类能力：Tools、Resources 和 Prompts。Tools 是“让模型执行动作”，类似后端的函数或 POST 接口；Resources 是“给应用读取上下文”，类似 GET 资源或文档；Prompts 是“给用户选择的可复用提示模板”。三者可以存在于同一个 MCP Server 中，但控制方不同：Tool 通常由模型决定调用，Resource 通常由客户端或应用加载，Prompt 通常由用户选择。MCP 本身不负责调用大模型，Host 才负责把这些能力接入模型对话。


## 为什么要单独的resource呢，做成一个search tool不行吗？


**很多简单场景只做一个 `search_orders` Tool 就够了**。Resource 不是必须的，也不是 Tool 的替代品。

区别在于：Tool 负责“让模型决定做什么”，Resource 负责“让应用决定把什么内容放进上下文”。

只用 Tool：

```
@mcp.tool()
def search_orders(status: str) -> list[dict]:
    """搜索订单。"""
    return [
        {
            "order_id": "A1001",
            "status": "shipping",
            "logistics": "顺丰 SF123456",
        }
    ]
```

流程是：

```
模型调用 search_orders
    ↓
Tool 直接返回完整订单数据
    ↓
模型根据结果回答
```

完全可以工作。

Resource 的价值通常出现在“搜索结果很多、内容很大、或者应用想控制上下文加载”的场景：

```
@mcp.tool()
def search_orders(status: str) -> list[dict]:
    """搜索订单，只返回摘要和资源地址。"""
    return [
        {
            "order_id": "A1001",
            "uri": "order://A1001",
            "status": "shipping",
        }
    ]


@mcp.resource("order://{order_id}")
def get_order_detail(order_id: str) -> str:
    """读取指定订单的完整详情。"""
    return f"""
订单号：{order_id}
状态：运输中
物流公司：顺丰
物流单号：SF123456
客服备注：用户要求工作日派送
"""
```

流程变成：

```
模型调用 search_orders
    ↓
得到 A1001 和 order://A1001
    ↓
Host 或应用决定是否读取 order://A1001
    ↓
把完整详情放进模型上下文
```

这样拆分的原因是：

- 搜索结果可以只返回摘要，不把大量内容一次性塞进模型上下文。
- 用户在界面中点击某个资源时，应用可以确定性地加载它。
- Resource 有稳定 URI，方便缓存、复用、展示和审计。
- 同一份订单详情可以被多个 Agent、编辑器或 UI 读取，不必都通过 Tool 调用。
- Tool 更适合搜索、修改、发送、取消等“动作”；Resource 更适合文档、文件、详情等“材料”。

但要注意：**Resource 不是天然更安全，也不是性能一定更好**，它只是另一种控制方式，权限仍然必须由服务端负责。

可以这样做工程判断：

```
数据少、逻辑简单、模型直接拿结果即可
    → 只用 Tool

数据大、需要用户选择、应用控制上下文加载
    → Tool + Resource

需要执行动作或产生副作用
    → Tool
```

所以如果只是“搜索订单并返回订单内容”，`search_orders` 一个 Tool 就够了。Resource 主要解决的是“搜索”和“加载上下文”需要分开控制的问题。[MCP Resources 官方规范](https://modelcontextprotocol.io/specification/latest/server/resources)