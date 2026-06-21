这三个东西是 Python AI 应用开发里非常核心的基础。你可以按这个关系理解：

```text
类型注解：告诉 Python / IDE / 框架，一个变量应该是什么类型
dataclass：快速定义“数据对象”
Pydantic：带类型校验、数据转换、JSON Schema 的高级数据模型
```

它们在大模型应用里经常用于：

```text
请求参数
配置对象
Agent Tool 入参
RAG 检索结果
LLM 结构化输出
API 返回值
```

---

# 1. 类型注解：Python 里的“类型说明”

Python 本身是动态类型语言。

```python
name = "Tom"
age = 18
```

这段代码可以运行，但你看不出函数参数和返回值的结构。

加上类型注解后：

```python
def add(a: int, b: int) -> int:
    return a + b
```

意思是：

```text
a 应该是 int
b 应该是 int
返回值应该是 int
```

但注意：**Python 的类型注解默认不强制运行时校验**。

比如：

```python
def add(a: int, b: int) -> int:
    return a + b


print(add("1", "2"))  # 可以运行，输出 "12"
```

所以类型注解主要作用是：

|作用|说明|
|---|---|
|提升可读性|看函数签名就知道参数类型|
|IDE 提示|PyCharm / VS Code 能做补全和检查|
|静态检查|mypy / pyright 可以提前发现类型错误|
|框架解析|FastAPI、Pydantic、LangChain 会读取类型信息|
|生成 Schema|大模型结构化输出、API 文档会用到|

---

## 常见写法

### 基础类型

```python
name: str = "Tom"
age: int = 18
score: float = 99.5
is_active: bool = True
```

### list / dict

Python 3.9+ 推荐：

```python
names: list[str] = ["Tom", "Jerry"]
scores: dict[str, int] = {"Tom": 90, "Jerry": 95}
```

### Optional：可能为空

```python
from typing import Optional

nickname: Optional[str] = None
```

Python 3.10+ 可以写成：

```python
nickname: str | None = None
```

### 函数返回值

```python
def get_user_name(user_id: int) -> str:
    return "Tom"
```

### 大模型应用里的例子

```python
def ask_llm(prompt: str, temperature: float = 0.7) -> str:
    """
    向大模型发送 prompt，并返回文本结果。
    """
    return "mock answer"
```

这比下面这种更清楚：

```python
def ask_llm(prompt, temperature=0.7):
    return "mock answer"
```

---

# 2. dataclass：快速定义数据对象

Java 里你经常会写 DTO / VO / POJO：

```java
public class UserDTO {
    private String name;
    private Integer age;

    public UserDTO(String name, Integer age) {
        this.name = name;
        this.age = age;
    }

    // getter setter toString equals hashCode ...
}
```

Python 里可以用 `dataclass` 快速定义：

```python
from dataclasses import dataclass


@dataclass
class UserDTO:
    name: str
    age: int
```

使用：

```python
user = UserDTO(name="Tom", age=18)

print(user.name)
print(user)
```

输出类似：

```text
Tom
UserDTO(name='Tom', age=18)
```

`@dataclass` 会自动帮你生成：

```text
__init__
__repr__
__eq__
```

也就是构造方法、打印方法、比较方法。

---

## dataclass 适合什么？

适合定义简单的数据结构，比如：

```python
from dataclasses import dataclass


@dataclass
class DocumentChunk:
    id: str
    content: str
    score: float
    source: str
```

在 RAG 里可以这样用：

```python
chunks = [
    DocumentChunk(
        id="chunk_001",
        content="Python supports type annotations.",
        score=0.92,
        source="python_notes.md"
    )
]
```

这个类的作用就类似 Java 里的：

```java
public record DocumentChunk(
    String id,
    String content,
    double score,
    String source
) {}
```

如果用 Java 17+ 的 `record` 类比，会更接近。

---

## dataclass 不会默认校验类型

这一点很重要。

```python
from dataclasses import dataclass


@dataclass
class UserDTO:
    name: str
    age: int


user = UserDTO(name="Tom", age="十八")
print(user)
```

这段代码默认也能运行。

也就是说：

```text
dataclass 主要是减少样板代码
不是运行时参数校验框架
```

---

# 3. Pydantic：带校验能力的数据模型

Pydantic 是 Python AI / Web API 生态里非常常见的库。

它的核心能力是：

```text
基于类型注解定义模型
运行时自动校验数据
自动做类型转换
自动生成 JSON Schema
```

简单例子：

```python
from pydantic import BaseModel


class UserDTO(BaseModel):
    name: str
    age: int
```

使用：

```python
user = UserDTO(name="Tom", age="18")

print(user.age)
print(type(user.age))
```

输出：

```text
18
<class 'int'>
```

这里 `"18"` 是字符串，但 Pydantic 自动转换成了 `int`。

如果无法转换：

```python
user = UserDTO(name="Tom", age="十八")
```

会报校验错误。

---

# 4. dataclass 和 Pydantic 的核心区别

|对比项|dataclass|Pydantic|
|---|---|---|
|标准库|是|否，需要安装|
|主要作用|快速定义数据对象|数据校验、转换、Schema|
|是否默认校验类型|否|是|
|JSON Schema|不擅长|很擅长|
|Web API 入参|一般不用它做强校验|FastAPI 常用|
|LLM 结构化输出|可用但不主流|很常用|
|性能|更轻|较重，但功能强|
|类比 Java|record / DTO|DTO + Validation + Jackson + Swagger Schema|

---

# 5. 为什么大模型应用里 Pydantic 很重要？

因为大模型返回的内容经常是不稳定的。

你希望模型返回：

```json
{
  "summary": "这是一段总结",
  "confidence": 0.85,
  "tags": ["Python", "AI"]
}
```

但模型可能返回：

```json
{
  "summary": "这是一段总结",
  "confidence": "high",
  "tags": "Python, AI"
}
```

这时候你需要一个结构定义和校验层。

用 Pydantic 可以这样写：

```python
from pydantic import BaseModel


class LLMAnswer(BaseModel):
    summary: str
    confidence: float
    tags: list[str]
```

然后：

```python
data = {
    "summary": "这是一段总结",
    "confidence": 0.85,
    "tags": ["Python", "AI"]
}

answer = LLMAnswer(**data)

print(answer.summary)
print(answer.confidence)
print(answer.tags)
```

如果字段类型不对，Pydantic 会直接报错。

这对 AI 应用很关键，因为你可以把：

```text
大模型的不稳定自然语言输出
```

转成：

```text
后端系统可以信任的结构化对象
```

---

# 6. 在 FastAPI 里的典型用法

如果你以后用 Python 写 AI 服务，FastAPI + Pydantic 很常见。

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()


class ChatRequest(BaseModel):
    question: str
    top_k: int = 5


class ChatResponse(BaseModel):
    answer: str
    sources: list[str]


@app.post("/chat", response_model=ChatResponse)
def chat(req: ChatRequest):
    return ChatResponse(
        answer=f"你问的是：{req.question}",
        sources=["doc_001.md"]
    )
```

这里 `ChatRequest` 会自动完成：

```text
JSON 请求体解析
字段类型校验
默认值处理
接口文档生成
错误响应生成
```

这很像 Java 里的：

```java
@PostMapping("/chat")
public ChatResponse chat(@RequestBody @Valid ChatRequest request) {
    ...
}
```

其中：

```text
Pydantic BaseModel ≈ Java DTO + Bean Validation + Jackson 反序列化 + Swagger Schema
```

---

# 7. 在 Agent Tool 里的典型用法

假设你定义一个搜索工具，入参是：

```python
from pydantic import BaseModel, Field


class SearchInput(BaseModel):
    query: str = Field(description="用户要搜索的问题")
    top_k: int = Field(default=5, ge=1, le=20, description="返回结果数量")
```

这个模型可以表达：

```text
query 是字符串
top_k 默认是 5
top_k 最小是 1，最大是 20
字段有自然语言说明
```

然后工具可以这样写：

```python
def search_tool(input_data: SearchInput) -> list[str]:
    return [
        f"result for {input_data.query}"
    ]
```

很多 Agent 框架会把 `SearchInput` 转成 JSON Schema，告诉大模型：

```text
这个工具需要 query 和 top_k 两个参数
query 是 string
top_k 是 integer
```

这样模型才能更稳定地调用工具。

---

# 8. 三者关系图

```text
类型注解
  ↓
告诉 Python / IDE / 框架：字段应该是什么类型

dataclass
  ↓
基于类型注解，快速生成普通数据类

Pydantic
  ↓
基于类型注解，生成可校验、可转换、可导出 Schema 的模型
```

换成 Java 后端视角：

```text
类型注解
≈ Java 字段类型 / 方法签名类型

dataclass
≈ Java record / Lombok @Data DTO

Pydantic
≈ DTO + @Valid + Jackson + Swagger/OpenAPI Schema
```

---

# 9. 你现在怎么学？

建议顺序：

## 第一步：先熟悉类型注解

重点掌握：

```python
str
int
float
bool
list[str]
dict[str, int]
str | None
```

能看懂函数签名：

```python
def retrieve(query: str, top_k: int = 5) -> list[str]:
    ...
```

---

## 第二步：掌握 dataclass

重点会写这种：

```python
from dataclasses import dataclass


@dataclass
class Chunk:
    content: str
    score: float
    source: str
```

用于内部数据传递。

---

## 第三步：重点掌握 Pydantic

因为它在 AI 应用开发里更实用。

重点学：

```python
BaseModel
Field
默认值
可选字段
list / dict 嵌套
model_dump()
model_validate()
JSON Schema
```

例如：

```python
from pydantic import BaseModel, Field


class RagRequest(BaseModel):
    question: str = Field(description="用户问题")
    top_k: int = Field(default=5, ge=1, le=20)


class RagResponse(BaseModel):
    answer: str
    sources: list[str]
    confidence: float
```

---

# 10. 最小记忆版

你可以先这样记：

```text
类型注解：写清楚变量、参数、返回值是什么类型。
dataclass：快速创建轻量 DTO，不负责严格校验。
Pydantic：可校验、可转换、可生成 Schema 的强 DTO，AI 应用和 FastAPI 很常用。
```

对于你做 **Java 后端 + AI 应用**：

```text
类型注解：必须掌握
dataclass：应该掌握
Pydantic：重点掌握
```

尤其是 Pydantic，因为它会频繁出现在：

```text
FastAPI
RAG 接口
Agent Tool 参数
LLM 结构化输出
配置管理
数据校验
```