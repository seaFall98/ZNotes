下面按三层讲：

1. **Streamable HTTP 是什么**
    
2. **它和旧 HTTP+SSE 的协议差异**
    
3. **Python + LangChain 实战：一个远程 MCP 工具服务 + LangChain Agent 调用**
    

---

# 1. Streamable HTTP 是什么

**Streamable HTTP** 是 MCP 现在推荐的远程传输方式，用来替代早期的 **HTTP+SSE**。MCP 官方规范把它列为标准传输之一，并明确说明它取代了 2024-11-05 版本里的 HTTP+SSE transport。它使用 **HTTP POST / GET**，并且服务端可以按需使用 **SSE** 来流式返回多条 MCP 消息。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

它的核心设计是：

> 一个 MCP server 暴露一个统一 endpoint，例如 `http://localhost:8000/mcp`。  
> client 通过 `POST` 发送 JSON-RPC 消息；server 可以直接返回 JSON，也可以返回 `text/event-stream` 做流式响应。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

---

# 2. 协议机制详解

## 2.1 单一 MCP endpoint

旧 HTTP+SSE 通常有两个 endpoint：

```text
GET  /sse        # server -> client
POST /messages   # client -> server
```

Streamable HTTP 收敛为一个 endpoint：

```text
POST /mcp        # client -> server，发送 JSON-RPC request / notification / response
GET  /mcp        # client 可选打开 SSE 流，接收 server 主动消息
DELETE /mcp      # 可选，用于结束 session
```

官方要求 server 必须提供一个支持 `POST` 和 `GET` 的 MCP endpoint。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

---

## 2.2 POST：客户端发送 MCP 消息

client 每发一个 JSON-RPC 消息，就发一个新的 HTTP POST：

```http
POST /mcp
Accept: application/json, text/event-stream
Content-Type: application/json
MCP-Protocol-Version: 2025-11-25
```

body 是一个 JSON-RPC request、notification 或 response。

server 的响应分两类：

|输入类型|server 响应|
|---|---|
|JSON-RPC notification / response|通常返回 `202 Accepted`，无 body|
|JSON-RPC request|返回 `application/json` 单个响应，或 `text/event-stream` 流式响应|

官方规范要求 client 在 POST 的 `Accept` header 中同时声明支持 `application/json` 和 `text/event-stream`。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

---

## 2.3 返回 JSON vs 返回 SSE

Streamable HTTP 的名字容易误解。它不是“永远 streaming”，而是：

```text
简单请求 -> application/json
长任务 / 多消息 / server-to-client 交互 -> text/event-stream
```

例如：

### 简单工具调用

```text
client POST /mcp
server 200 application/json
```

适合：

- 加法计算；
    
- 查询配置；
    
- 短 SQL 查询；
    
- 简单 API wrapper。
    

### 长任务工具调用

```text
client POST /mcp
server 200 text/event-stream
event: message
data: {...progress...}

event: message
data: {...final response...}
```

适合：

- 爬取任务；
    
- 长 SQL 分析；
    
- 文件处理；
    
- RAG 批量检索；
    
- 需要 progress notification 的工具。
    

---

## 2.4 GET：服务端主动消息

client 可以对同一个 `/mcp` endpoint 发 GET，用来打开一个 SSE 流：

```http
GET /mcp
Accept: text/event-stream
```

server 可以通过这个流向 client 发送：

- notifications；
    
- server-to-client requests；
    
- logging；
    
- progress；
    
- elicitation 请求。
    

如果 server 不支持这种独立 SSE 流，可以返回 `405 Method Not Allowed`。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

---

## 2.5 session 管理

Streamable HTTP 支持有状态 session。

初始化完成时，server 可以在响应 header 中返回：

```http
MCP-Session-Id: <secure-session-id>
```

之后 client 必须在后续请求里带上：

```http
MCP-Session-Id: <secure-session-id>
```

官方规范要求 session ID 应全局唯一、密码学安全，并且 client 要安全处理该 ID。server 如果要求 session ID，可以对缺失 session ID 的非初始化请求返回 `400 Bad Request`。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

---

## 2.6 断线恢复

如果 server 使用 SSE 流式响应，它可以给 SSE event 附带 `id`：

```text
id: stream-123:42
event: message
data: {...}
```

client 断线后，可以用：

```http
GET /mcp
Last-Event-ID: stream-123:42
```

请求 server 从上次收到的位置恢复。官方规范规定，恢复永远通过 `GET` + `Last-Event-ID` 完成，而且 server 不能重放其他 stream 的消息。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

---

# 3. Streamable HTTP 相比 HTTP+SSE 的优势

|维度|HTTP+SSE|Streamable HTTP|
|---|---|---|
|endpoint 数量|SSE endpoint + POST endpoint|单一 `/mcp` endpoint|
|简单请求|也要依赖 SSE 通道|可直接 JSON 响应|
|长任务|SSE|可选 SSE|
|client 实现|双通道协调|单入口，按需 streaming|
|server 部署|代理、鉴权、session 绑定更麻烦|更接近普通 HTTP 服务|
|断线恢复|语义不统一|`Last-Event-ID` 明确|
|规范状态|Deprecated|当前推荐远程传输|

---

# 4. Python + LangChain 实战

下面做一个实际例子：

```text
LangChain Agent
    |
    | Streamable HTTP
    v
MCP Server: http://localhost:8000/mcp
    |
    +-- get_fx_rate()
    +-- convert_currency()
```

这里为了避免外部 API 依赖，汇率用内存假数据。真实生产环境可以替换成数据库、内部 API、Redis、风控系统、CRM、工单系统等。

LangChain 官方文档说明，`langchain-mcp-adapters` 可以把 MCP server 暴露的 tools 转成 LangChain tools；HTTP transport，也就是 Streamable HTTP，在 LangChain 配置里使用 `"transport": "http"`。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/mcp "Model Context Protocol (MCP) - Docs by LangChain"))

---

## 4.1 安装依赖

```bash
pip install fastmcp langchain langchain-mcp-adapters langchain-openai
```

也可以用其他模型 provider。下面的代码把模型名放到环境变量里，不绑定具体供应商。

```bash
export MODEL="openai:gpt-5.5"
export OPENAI_API_KEY="你的 API key"
```

---

# 5. MCP Server：`server.py`

```python
# server.py
from decimal import Decimal, ROUND_HALF_UP
from fastmcp import FastMCP

mcp = FastMCP("currency-mcp")


RATES: dict[str, Decimal] = {
    "USD": Decimal("1.0000"),
    "SGD": Decimal("1.3500"),
    "CNY": Decimal("7.2500"),
    "EUR": Decimal("0.9200"),
    "JPY": Decimal("157.0000"),
}


@mcp.tool()
def get_supported_currencies() -> list[str]:
    """Return the list of supported currency codes."""
    return sorted(RATES.keys())


@mcp.tool()
def get_fx_rate(base: str, quote: str) -> str:
    """
    Return the FX rate from base currency to quote currency.

    Example:
        base="USD", quote="SGD" returns "1.3500"
    """
    base = base.upper()
    quote = quote.upper()

    if base not in RATES:
        raise ValueError(f"Unsupported base currency: {base}")
    if quote not in RATES:
        raise ValueError(f"Unsupported quote currency: {quote}")

    rate = RATES[quote] / RATES[base]
    return str(rate.quantize(Decimal("0.0001"), rounding=ROUND_HALF_UP))


@mcp.tool()
def convert_currency(amount: float, base: str, quote: str) -> dict:
    """
    Convert an amount from base currency to quote currency.

    Returns amount, base, quote, rate, and converted_amount.
    """
    base = base.upper()
    quote = quote.upper()

    if base not in RATES:
        raise ValueError(f"Unsupported base currency: {base}")
    if quote not in RATES:
        raise ValueError(f"Unsupported quote currency: {quote}")

    amount_decimal = Decimal(str(amount))
    rate = RATES[quote] / RATES[base]
    converted = amount_decimal * rate

    return {
        "amount": str(amount_decimal),
        "base": base,
        "quote": quote,
        "rate": str(rate.quantize(Decimal("0.0001"), rounding=ROUND_HALF_UP)),
        "converted_amount": str(
            converted.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)
        ),
    }


if __name__ == "__main__":
    # FastMCP 的 HTTP transport 对应 MCP Streamable HTTP。
    # 默认 endpoint 通常是 /mcp。
    mcp.run(
        transport="http",
        host="127.0.0.1",
        port=8000,
    )
```

启动：

```bash
python server.py
```

服务地址：

```text
http://127.0.0.1:8000/mcp
```

FastMCP 文档说明，HTTP transport 会把 MCP server 暴露为网络服务，默认可通过 `/mcp` endpoint 访问，并且该 transport 使用 Streamable HTTP 协议。([FastMCP](https://gofastmcp.com/deployment/running-server?utm_source=chatgpt.com "Running Your Server"))

---

# 6. LangChain Client：直接加载 MCP tools

先不接 LLM，只验证 LangChain 能从 MCP server 加载工具。

创建 `inspect_tools.py`：

```python
# inspect_tools.py
import asyncio

from langchain_mcp_adapters.client import MultiServerMCPClient


async def main() -> None:
    client = MultiServerMCPClient(
        {
            "currency": {
                "transport": "http",
                "url": "http://127.0.0.1:8000/mcp",
            }
        }
    )

    tools = await client.get_tools()

    print("Loaded tools:")
    for tool in tools:
        print(f"- {tool.name}: {tool.description}")


if __name__ == "__main__":
    asyncio.run(main())
```

运行：

```bash
python inspect_tools.py
```

预期输出类似：

```text
Loaded tools:
- get_supported_currencies: Return the list of supported currency codes.
- get_fx_rate: Return the FX rate from base currency to quote currency.
- convert_currency: Convert an amount from base currency to quote currency.
```

LangChain 文档中也使用 `MultiServerMCPClient(...).get_tools()` 从 MCP server 加载 tools，并把它们传给 agent。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/mcp "Model Context Protocol (MCP) - Docs by LangChain"))

---

# 7. LangChain Agent 调用 MCP tool

创建 `agent_client.py`：

```python
# agent_client.py
import asyncio
import os

from langchain.agents import create_agent
from langchain_mcp_adapters.client import MultiServerMCPClient


async def main() -> None:
    model = os.environ.get("MODEL")
    if not model:
        raise RuntimeError(
            "Missing MODEL env var. Example: export MODEL='openai:gpt-5.5'"
        )

    client = MultiServerMCPClient(
        {
            "currency": {
                "transport": "http",
                "url": "http://127.0.0.1:8000/mcp",
                # 生产环境可以加鉴权 header：
                # "headers": {
                #     "Authorization": "Bearer <token>",
                # }
            }
        }
    )

    tools = await client.get_tools()

    agent = create_agent(
        model=model,
        tools=tools,
        system_prompt=(
            "You are a finance operations assistant. "
            "Use the available currency tools when calculation or FX lookup is needed. "
            "Do not invent exchange rates."
        ),
    )

    result = await agent.ainvoke(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "把 250 新加坡元换算成人民币，并告诉我使用的汇率。",
                }
            ]
        }
    )

    print(result["messages"][-1].content)


if __name__ == "__main__":
    asyncio.run(main())
```

运行：

```bash
python agent_client.py
```

可能输出：

```text
250 SGD 约等于 1342.59 CNY。使用的汇率是 1 SGD = 5.3704 CNY。
```

这里 agent 不需要知道 MCP 协议细节。它看到的是普通 LangChain tool；底层由 `langchain-mcp-adapters` 负责连接 MCP server、初始化 session、列出 tools、执行 tool call。LangChain 文档说明 MCP tools 会被转换为 LangChain tools，可直接用于 LangChain agent 或 workflow。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/mcp "Model Context Protocol (MCP) - Docs by LangChain"))

---

# 8. 带鉴权 header 的写法

生产环境不要裸露 MCP endpoint。LangChain MCP adapter 支持给 HTTP / SSE transport 传自定义 headers，例如 `Authorization`。([LangChain 文档](https://docs.langchain.com/oss/python/langchain/mcp "Model Context Protocol (MCP) - Docs by LangChain"))

```python
client = MultiServerMCPClient(
    {
        "currency": {
            "transport": "http",
            "url": "https://mcp.example.com/mcp",
            "headers": {
                "Authorization": f"Bearer {os.environ['MCP_TOKEN']}",
                "X-Request-Source": "langchain-agent",
            },
        }
    }
)
```

server 端则应校验：

- `Authorization`;
    
- `Origin`;
    
- session；
    
- rate limit；
    
- tool-level permission。
    

MCP 官方规范特别要求 Streamable HTTP server 校验 `Origin`，本地运行时应优先绑定 `127.0.0.1`，并应实现适当认证，以防 DNS rebinding 等攻击。([Model Context Protocol](https://modelcontextprotocol.io/specification/2025-11-25/basic/transports "Transports - Model Context Protocol"))

---

# 9. 更贴近生产的目录结构

```text
mcp-streamable-http-demo/
├── server.py
├── agent_client.py
├── inspect_tools.py
├── requirements.txt
└── README.md
```

`requirements.txt`：

```txt
fastmcp
langchain
langchain-mcp-adapters
langchain-openai
```

启动顺序：

```bash
# terminal 1
python server.py

# terminal 2
python inspect_tools.py

# terminal 3
export MODEL="openai:gpt-5.5"
export OPENAI_API_KEY="你的 API key"
python agent_client.py
```

---

# 10. 实际落地时的设计建议

## 10.1 MCP server 不要直接暴露敏感操作

不要这样：

```python
@mcp.tool()
def execute_sql(sql: str) -> str:
    ...
```

更安全的做法：

```python
@mcp.tool()
def get_customer_balance(customer_id: str) -> dict:
    ...
```

即：

- 工具语义明确；
    
- 参数结构化；
    
- 权限可控；
    
- 不让模型自由拼 SQL；
    
- server 端做 allowlist 和审计。
    

---

## 10.2 每个 MCP server 聚焦一个 bounded context

推荐：

```text
finance-mcp
crm-mcp
jira-mcp
warehouse-mcp
search-mcp
```

不推荐：

```text
mega-company-mcp
```

原因是权限、审计、schema 演化、tool routing 都会更清晰。

---

## 10.3 HTTP 层建议放在网关后面

生产环境典型架构：

```text
LangChain Agent
    |
API Gateway / Reverse Proxy
    |
Auth / Rate Limit / Audit
    |
MCP Server /mcp
    |
Internal APIs / DB / Queue
```

关键点：

- TLS；
    
- OAuth2 / JWT / mTLS；
    
- Origin 校验；
    
- request ID；
    
- structured logs；
    
- tool call audit；
    
- per-user authorization；
    
- timeout；
    
- retry；
    
- circuit breaker。
    

---

# 11. 什么时候用 Streamable HTTP，什么时候用 stdio

|场景|推荐 transport|
|---|---|
|本地 Claude Desktop 插件|stdio|
|本机开发调试|stdio 或 Streamable HTTP|
|多个 agent 共用一个工具服务|Streamable HTTP|
|企业内部统一 tool gateway|Streamable HTTP|
|需要鉴权、审计、限流|Streamable HTTP|
|serverless / container 部署|Streamable HTTP|
|单用户、本地文件系统操作|stdio 更简单|

---

# 12. 结论

**Streamable HTTP 是 MCP 远程化、服务化、生产化的主力 transport。**

它的关键价值是：

- 单一 `/mcp` endpoint；
    
- 普通 HTTP 请求优先；
    
- 需要时才使用 SSE streaming；
    
- 支持 session；
    
- 支持断线恢复；
    
- 更适合鉴权、网关、负载均衡、容器部署；
    
- LangChain 可以通过 `langchain-mcp-adapters` 直接消费 MCP tools。