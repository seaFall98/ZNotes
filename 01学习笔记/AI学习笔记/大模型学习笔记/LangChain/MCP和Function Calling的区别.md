下面用一个**完全相同的工具：`get_weather(city)` 查天气**来对比。

先看结论：

```text
Function calling:
    你在 Python 代码里定义 tool → 绑定给 LLM → LLM 产生 tool_call → 你的代码执行 tool

MCP:
    你把 tool 放到 MCP server 里 → LangChain 作为 MCP client 发现工具 → 转成 LangChain tools → Agent 调用
```

LangChain 官方文档里，tools 本质上是“有明确定义输入输出的 callable functions”，模型根据上下文决定是否调用、传什么参数；而 `langchain-mcp-adapters` 的作用是让 Agent 使用一个或多个 MCP server 暴露出来的 tools。([LangChain Docs](https://docs.langchain.com/oss/python/langchain/tools?utm_source=chatgpt.com "Tools - Docs by LangChain"))

---

# 1. Function calling：工具写在你的应用代码里

## 安装

```bash
pip install -U langchain langchain-openai
```

## `function_calling_demo.py`

```python
import os
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool
from langchain_core.messages import HumanMessage, ToolMessage


# 1. 这是一个普通 Python 函数
#    @tool 会把它包装成 LangChain Tool，并根据函数签名生成 schema
@tool
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    fake_db = {
        "Singapore": "Singapore: 31°C, humid, partly cloudy.",
        "Beijing": "Beijing: 26°C, clear.",
        "Shanghai": "Shanghai: 29°C, light rain.",
    }
    return fake_db.get(city, f"No weather data for {city}.")


# 2. 创建模型
llm = ChatOpenAI(
    model="gpt-4.1-mini",
    temperature=0,
    api_key=os.environ["OPENAI_API_KEY"],
)


# 3. 把工具绑定给模型
#    这一步就是典型的 function/tool calling
llm_with_tools = llm.bind_tools([get_weather])


# 4. 用户提问
messages = [
    HumanMessage(content="What's the weather in Singapore?")
]


# 5. 第一次调用模型
#    模型不一定直接回答，它可能返回 tool_calls
ai_msg = llm_with_tools.invoke(messages)

print("=== LLM raw response ===")
print(ai_msg)
print("\n=== Tool calls ===")
print(ai_msg.tool_calls)


# 6. 如果模型请求调用工具，则由你的 Python 代码执行
messages.append(ai_msg)

for tool_call in ai_msg.tool_calls:
    if tool_call["name"] == "get_weather":
        result = get_weather.invoke(tool_call["args"])

        messages.append(
            ToolMessage(
                content=result,
                tool_call_id=tool_call["id"],
            )
        )


# 7. 把工具结果交回模型，让模型生成最终自然语言回答
final_msg = llm_with_tools.invoke(messages)

print("\n=== Final answer ===")
print(final_msg.content)
```

## 运行

```bash
export OPENAI_API_KEY="你的 key"
python function_calling_demo.py
```

## 你会看到类似结果

```text
=== Tool calls ===
[
  {
    "name": "get_weather",
    "args": {
      "city": "Singapore"
    },
    "id": "call_xxx"
  }
]

=== Final answer ===
The weather in Singapore is 31°C, humid, and partly cloudy.
```

## 这里发生了什么

关键点是这行：

```python
llm_with_tools = llm.bind_tools([get_weather])
```

它告诉模型：

```text
你现在可以调用 get_weather(city: str)
```

但真正执行的是这段 Python：

```python
result = get_weather.invoke(tool_call["args"])
```

也就是说，**Function calling 里，工具是你的应用代码提供的，执行也由你的应用控制**。

---

# 2. MCP：工具放在独立 MCP server 里

现在换一种架构。

不把 `get_weather` 直接写在主应用里，而是把它暴露成一个 MCP server。

LangChain MCP 文档说明，`langchain-mcp-adapters` 可以让 Agent 使用 MCP servers 中定义的 tools；`MultiServerMCPClient` 默认是无状态的，每次工具调用会创建 MCP session、执行工具、再清理。([LangChain Docs](https://docs.langchain.com/oss/python/langchain/mcp?utm_source=chatgpt.com "Model Context Protocol (MCP) - Docs by LangChain"))

---

## 安装

```bash
pip install -U langchain langchain-openai langgraph langchain-mcp-adapters mcp
```

---

## 文件 1：`weather_mcp_server.py`

```python
from mcp.server.fastmcp import FastMCP


# 1. 创建一个 MCP server
mcp = FastMCP("weather-server")


# 2. 把普通 Python 函数注册成 MCP tool
@mcp.tool()
def get_weather(city: str) -> str:
    """Get current weather for a city."""
    fake_db = {
        "Singapore": "Singapore: 31°C, humid, partly cloudy.",
        "Beijing": "Beijing: 26°C, clear.",
        "Shanghai": "Shanghai: 29°C, light rain.",
    }
    return fake_db.get(city, f"No weather data for {city}.")


# 3. 启动 MCP server
#    stdio 模式：主程序通过子进程标准输入/输出和它通信
if __name__ == "__main__":
    mcp.run(transport="stdio")
```

这个文件的职责只有一个：

```text
暴露 get_weather 这个工具
```

它不关心 LangChain，也不关心具体模型。

---

## 文件 2：`mcp_langchain_client.py`

```python
import os
import asyncio

from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
from langchain_mcp_adapters.client import MultiServerMCPClient


async def main():
    # 1. LangChain 作为 MCP client，连接 MCP server
    client = MultiServerMCPClient(
        {
            "weather": {
                "command": "python",
                "args": ["weather_mcp_server.py"],
                "transport": "stdio",
            }
        }
    )

    # 2. 从 MCP server 动态获取 tools
    #    这里拿到的是 LangChain-compatible tools
    tools = await client.get_tools()

    print("=== Tools discovered from MCP server ===")
    for t in tools:
        print(t.name, "-", t.description)

    # 3. 创建模型
    llm = ChatOpenAI(
        model="gpt-4.1-mini",
        temperature=0,
        api_key=os.environ["OPENAI_API_KEY"],
    )

    # 4. 创建一个 LangGraph/LangChain agent
    #    注意：这里的 tools 不是你本地手写绑定的，而是 MCP server 暴露出来的
    agent = create_react_agent(llm, tools)

    # 5. 调用 agent
    result = await agent.ainvoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "What's the weather in Singapore?",
                }
            ]
        }
    )

    # 6. 打印最终回答
    print("\n=== Final answer ===")
    print(result["messages"][-1].content)


if __name__ == "__main__":
    asyncio.run(main())
```

## 运行

```bash
export OPENAI_API_KEY="你的 key"
python mcp_langchain_client.py
```

你会看到类似：

```text
=== Tools discovered from MCP server ===
get_weather - Get current weather for a city.

=== Final answer ===
The weather in Singapore is 31°C, humid, and partly cloudy.
```

---

# 3. 代码层面的核心区别

## Function calling 版本

工具在主程序里：

```python
@tool
def get_weather(city: str) -> str:
    ...
```

然后直接绑定：

```python
llm_with_tools = llm.bind_tools([get_weather])
```

调用链是：

```text
Python app
  ├─ defines get_weather
  ├─ sends schema to LLM
  ├─ receives tool_call
  ├─ executes get_weather locally
  └─ sends result back to LLM
```

---

## MCP 版本

工具在另一个 server 里：

```python
@mcp.tool()
def get_weather(city: str) -> str:
    ...
```

主程序不直接定义这个工具，而是发现它：

```python
client = MultiServerMCPClient(
    {
        "weather": {
            "command": "python",
            "args": ["weather_mcp_server.py"],
            "transport": "stdio",
        }
    }
)

tools = await client.get_tools()
```

调用链是：

```text
LangChain app
  ├─ starts/connects to MCP server
  ├─ discovers available tools
  ├─ gives tools to agent
  ├─ LLM decides to call get_weather
  ├─ LangChain calls MCP server
  └─ MCP server executes get_weather
```

---

# 4. 最小差异表

|问题|Function calling|MCP|
|---|---|---|
|`get_weather` 写在哪里？|主应用 Python 代码里|MCP server 里|
|LangChain 怎么拿到工具？|`llm.bind_tools([tool])`|`await client.get_tools()`|
|工具是否可被其他客户端复用？|默认不行|可以，比如 Claude Desktop、Cursor、其他 Agent|
|是否需要 MCP server？|不需要|需要|
|是否适合小项目？|更适合|偏重|
|是否适合工具平台化？|不太适合|更适合|

---

# 5. 更实际的例子：数据库查询

假设你有一个内部数据库查询工具。

## Function calling 写法

```python
from langchain_core.tools import tool

@tool
def query_user_order(user_id: str) -> str:
    """Query latest order status for a user."""
    # 这里直接连数据库
    # conn = psycopg.connect(...)
    # result = conn.execute(...)
    return f"User {user_id}'s latest order is shipped."
```

绑定：

```python
llm_with_tools = llm.bind_tools([query_user_order])
```

问题是：

```text
这个数据库工具只属于当前这个 Python app。
其他 Agent、IDE、桌面 AI 应用想复用，需要重新接一遍。
```

---

## MCP 写法

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("order-db-server")

@mcp.tool()
def query_user_order(user_id: str) -> str:
    """Query latest order status for a user."""
    return f"User {user_id}'s latest order is shipped."

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

然后任何支持 MCP 的客户端都可以接：

```python
client = MultiServerMCPClient(
    {
        "orders": {
            "command": "python",
            "args": ["order_mcp_server.py"],
            "transport": "stdio",
        }
    }
)

tools = await client.get_tools()
```

区别是：

```text
数据库能力被封装成了一个 MCP server。
LangChain 只是其中一个 client。
以后 Cursor、Claude Desktop、其他内部 Agent 也可以复用。
```

---

# 6. 你真正该怎么选

## 用 Function calling，如果你只是这样：

```text
我有一个 Python app
里面有 3~10 个工具
只给这个 app 用
工具逻辑比较简单
```

代码结构：

```python
@tool
def my_tool(...):
    ...

llm.bind_tools([my_tool])
```

这就够了。

---

## 用 MCP，如果你是这样：

```text
我有一组内部工具
希望多个 Agent / IDE / AI 客户端都能用
工具要独立部署、独立授权、独立维护
以后可能接 GitHub、DB、Slack、文件系统、浏览器
```

代码结构：

```python
# server side
@mcp.tool()
def my_tool(...):
    ...

# client side
tools = await mcp_client.get_tools()
agent = create_react_agent(llm, tools)
```

---

# 7. 最直接的工程判断

可以这么记：

```text
Function calling 是“把 Python 函数交给模型调用”。

MCP 是“把一整个工具服务接入 LangChain / Agent”。
```

在 LangChain 里最终都会变成类似 `tools`：

```python
agent = create_react_agent(llm, tools)
```

但来源不同：

```python
# Function calling / LangChain native tools
tools = [get_weather]

# MCP tools
tools = await client.get_tools()
```

所以最本质的代码差异就是：

```python
# Function calling
llm.bind_tools([local_python_function])

# MCP
tools = await mcp_client.get_tools()
agent = create_react_agent(llm, tools)
```