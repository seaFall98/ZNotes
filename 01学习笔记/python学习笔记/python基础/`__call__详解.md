# `__call__`：让“对象像函数一样被调用”

你可以先记一句话：

> **Python 里如果一个类实现了 `__call__` 方法，那么这个类的实例就可以像函数一样使用。**

例如：

```python
class Greeter:
    def __call__(self, name: str) -> str:
        return f"Hello, {name}"


greeter = Greeter()

print(greeter("Tom"))  # Hello, Tom
```

这里的关键是：

```python
greeter("Tom")
```

本质上等价于：

```python
greeter.__call__("Tom")
```

---

# 1. Java 视角怎么理解？

你是 Java 后端背景，可以这样类比。

Java 里如果你想让一个对象“像函数一样执行”，通常会写：

```java
@FunctionalInterface
public interface Handler {
    String handle(String input);
}
```

然后：

```java
public class GreetingHandler implements Handler {
    @Override
    public String handle(String input) {
        return "Hello, " + input;
    }
}
```

调用时是：

```java
handler.handle("Tom");
```

Python 的 `__call__` 相当于让你可以写成：

```python
handler("Tom")
```

也就是说：

|Java|Python|
|---|---|
|`handler.handle(input)`|`handler(input)`|
|`Runnable.run()`|`obj()`|
|`Function.apply(x)`|`obj(x)`|
|策略对象执行方法|可调用对象|

所以 `__call__` 的核心价值是：

```text
把一个“有状态对象”伪装成“函数”
```

---

# 2. 为什么不用普通方法，非要 `__call__`？

你当然可以写普通方法：

```python
class Greeter:
    def greet(self, name: str) -> str:
        return f"Hello, {name}"


greeter = Greeter()
print(greeter.greet("Tom"))
```

但如果用 `__call__`：

```python
class Greeter:
    def __call__(self, name: str) -> str:
        return f"Hello, {name}"


greeter = Greeter()
print(greeter("Tom"))
```

区别在于：

```text
普通方法：对象.方法名(...)
__call__：对象(...)
```

这在框架里很有用，因为框架通常只关心“这个东西能不能调用”，不关心它到底是函数、类实例、闭包还是装饰器对象。

---

# 3. `callable()`：判断一个对象能不能被调用

Python 里可以用 `callable()` 判断对象是否可调用：

```python
def hello():
    return "hello"


class A:
    pass


class B:
    def __call__(self):
        return "called"


print(callable(hello))  # True
print(callable(A))      # True，类本身可以被调用，用来创建实例
print(callable(A()))    # False，A 的实例不可调用
print(callable(B()))    # True，B 的实例实现了 __call__
```

注意这里：

```python
callable(A)
```

是 `True`，因为类本身可以调用：

```python
a = A()
```

这其实是“调用类来创建对象”。

---

# 4. `__call__` 的典型用途一：有状态函数

普通函数通常没有对象状态。

例如：

```python
def add_prefix(text: str) -> str:
    return "[INFO] " + text
```

如果前缀要可配置，可以用闭包：

```python
def make_prefixer(prefix: str):
    def wrapper(text: str) -> str:
        return prefix + text

    return wrapper


info_prefixer = make_prefixer("[INFO] ")
print(info_prefixer("server started"))
```

也可以用 `__call__`：

```python
class Prefixer:
    def __init__(self, prefix: str):
        self.prefix = prefix

    def __call__(self, text: str) -> str:
        return self.prefix + text


info_prefixer = Prefixer("[INFO] ")
print(info_prefixer("server started"))
```

这里：

```python
info_prefixer("server started")
```

看起来像函数，但 `info_prefixer` 内部保存了状态：

```python
self.prefix
```

这就是 `__call__` 的核心用途之一：

```text
函数式调用形式 + 面向对象状态管理
```

---

# 5. 典型用途二：策略模式 / Handler / Pipeline Step

比如你做 AI 应用，有一个处理链：

```text
用户问题
  → query rewrite
  → retrieve
  → rerank
  → generate answer
```

每一步都可以设计成可调用对象。

```python
class QueryRewriter:
    def __call__(self, question: str) -> str:
        # 实际项目里这里可能调用 LLM 做问题改写
        return question.strip()


class Retriever:
    def __call__(self, query: str) -> list[str]:
        # 实际项目里这里可能查向量库
        return [f"doc for {query}"]


class AnswerGenerator:
    def __call__(self, docs: list[str]) -> str:
        # 实际项目里这里可能调用 LLM 生成答案
        return f"answer based on {docs}"
```

使用：

```python
rewrite = QueryRewriter()
retrieve = Retriever()
generate = AnswerGenerator()

query = rewrite("  Python 的 __call__ 是什么？ ")
docs = retrieve(query)
answer = generate(docs)

print(answer)
```

这种写法的好处是：

```text
每个步骤都是对象，可以持有配置、客户端、缓存、日志组件
每个步骤又都能像函数一样调用
Pipeline 写起来很干净
```

---

# 6. 典型用途三：AI Tool / Agent Tool

在 Agent 系统里，一个 Tool 可以设计成这样：

```python
from dataclasses import dataclass


@dataclass
class SearchTool:
    endpoint: str
    top_k: int = 5

    def __call__(self, query: str) -> list[str]:
        # 实际项目中这里会请求搜索服务或向量数据库
        return [
            f"Search result 1 for {query}",
            f"Search result 2 for {query}",
        ][: self.top_k]


search_tool = SearchTool(endpoint="https://search.example.com", top_k=2)

results = search_tool("RAG production best practices")
print(results)
```

这里的 `search_tool` 既是对象，又像函数：

```python
search_tool("RAG production best practices")
```

好处是它可以保存工具配置：

```python
endpoint
top_k
api_key
timeout
retry_policy
```

但调用方式仍然非常简洁。

Java 类比：

```java
public class SearchTool implements Function<String, List<String>> {
    private final String endpoint;
    private final int topK;

    @Override
    public List<String> apply(String query) {
        ...
    }
}
```

Python 的 `__call__` 就像把 `apply()` 变成了对象本身的调用语法。

---

# 7. 典型用途四：装饰器类

你之前学 Python，肯定会碰到装饰器。

普通函数装饰器：

```python
def log_time(func):
    def wrapper(*args, **kwargs):
        print("before")
        result = func(*args, **kwargs)
        print("after")
        return result

    return wrapper
```

类也可以写装饰器，靠的就是 `__call__`：

```python
class LogCall:
    def __init__(self, func):
        self.func = func

    def __call__(self, *args, **kwargs):
        print("before")
        result = self.func(*args, **kwargs)
        print("after")
        return result


@LogCall
def say_hello(name: str) -> str:
    return f"Hello, {name}"


print(say_hello("Tom"))
```

这里：

```python
@LogCall
def say_hello(...):
    ...
```

大概等价于：

```python
say_hello = LogCall(say_hello)
```

所以现在的 `say_hello` 已经不是普通函数，而是 `LogCall` 的实例。

当你调用：

```python
say_hello("Tom")
```

其实执行的是：

```python
say_hello.__call__("Tom")
```

---

# 8. 典型用途五：机器学习模型推理

很多深度学习框架里，模型对象可以直接调用：

```python
output = model(input)
```

这背后通常就是 `__call__`。

简化理解：

```python
class Model:
    def __call__(self, x):
        return self.forward(x)

    def forward(self, x):
        return x * 2


model = Model()

print(model(10))        # 20
print(model.forward(10))  # 20
```

为什么不直接调用 `forward()`？

因为框架可以在 `__call__` 里统一做额外逻辑，例如：

```text
参数预处理
hook 调用
梯度上下文
日志
性能统计
异常包装
调用 forward
```

所以用户写：

```python
model(input)
```

框架内部走：

```text
__call__
  → 前置处理
  → forward
  → 后置处理
```

这也是你学 AI / 大模型框架时会经常看到的模式。

---

# 9. `__call__` 和 `__init__` 的区别

这两个很容易混。

```python
class Demo:
    def __init__(self, name: str):
        print("__init__ called")
        self.name = name

    def __call__(self, message: str):
        print("__call__ called")
        return f"{self.name}: {message}"


demo = Demo("agent")
```

这一步调用：

```python
Demo("agent")
```

触发的是创建对象过程，其中会调用：

```python
__init__
```

然后：

```python
demo("hello")
```

触发的是实例调用过程：

```python
__call__
```

完整例子：

```python
class Demo:
    def __init__(self, name: str):
        print("__init__ called")
        self.name = name

    def __call__(self, message: str):
        print("__call__ called")
        return f"{self.name}: {message}"


demo = Demo("agent")
print(demo("hello"))
```

输出：

```text
__init__ called
__call__ called
agent: hello
```

记忆方式：

```text
__init__：对象创建时执行
__call__：对象被当成函数调用时执行
```

---

# 10. `__call__` 和 `__new__` 的关系

更底层一点：

```python
obj = MyClass()
```

看起来是在“调用类”。

类对象本身也可以被调用，因为类的元类 `type` 实现了 `__call__`。

大致流程是：

```text
MyClass()
  → type.__call__(MyClass, ...)
  → MyClass.__new__(...)
  → MyClass.__init__(...)
  → 返回实例
```

所以你之前学元类时，可以把它串起来：

```text
普通对象调用：
    obj(...)
    → obj.__call__(...)

类对象调用：
    MyClass(...)
    → type.__call__(...)
    → __new__
    → __init__
```

这也是为什么：

```python
callable(MyClass)  # True
```

因为类本身也是可调用对象。

---

# 11. 和大模型应用最相关的理解

你做 Java + AI 应用时，最常遇到 `__call__` 的地方大概是这几类：

## 1. Embedding 模型

```python
embedding = EmbeddingModel(...)
vector = embedding("hello world")
```

对象里保存：

```text
model_name
api_key
batch_size
timeout
```

调用时像函数一样生成向量。

---

## 2. LLM Client

```python
llm = ChatModel(model="gpt-4.1", temperature=0.7)
answer = llm("解释 RAG")
```

对象保存配置，调用时发请求。

---

## 3. Runnable / Chain

```python
chain = rewrite | retrieve | generate
result = chain(question)
```

很多框架会把每个节点设计成可调用对象，方便组合成 pipeline。

---

## 4. Tool

```python
search = SearchTool(api_key="xxx")
results = search("Python __call__")
```

工具对象既能保存配置，又能被统一调度器调用。

---

# 12. 什么时候适合自己写 `__call__`？

适合：

|场景|是否适合|
|---|--:|
|对象本质上就是一个“处理器”|适合|
|对象需要保存配置和状态|适合|
|希望传给框架当 callback / handler|适合|
|写 pipeline step|适合|
|写类装饰器|适合|
|只是普通业务对象|不建议|
|方法很多，语义复杂|不建议滥用|

比较推荐的场景：

```python
class Reranker:
    def __init__(self, model_name: str):
        self.model_name = model_name

    def __call__(self, query: str, docs: list[str]) -> list[str]:
        # 根据 query 对 docs 重新排序
        return docs
```

不太推荐的场景：

```python
class UserService:
    def __call__(self):
        ...
```

因为 `UserService()`、`user_service()` 语义不清楚。

更好的写法是：

```python
user_service.create_user(...)
user_service.get_user(...)
user_service.delete_user(...)
```

---

# 13. 最小实战例子：RAG Pipeline Step

```python
from dataclasses import dataclass


@dataclass
class QueryRewriteStep:
    """负责对用户问题做简单清洗或改写。"""

    lowercase: bool = True

    def __call__(self, question: str) -> str:
        # strip 去掉首尾空白
        query = question.strip()

        # 根据配置决定是否转小写
        if self.lowercase:
            query = query.lower()

        return query


@dataclass
class RetrieveStep:
    """负责根据 query 检索文档。"""

    top_k: int = 3

    def __call__(self, query: str) -> list[str]:
        # 这里用 mock 数据模拟向量检索结果
        return [
            f"doc_1 related to {query}",
            f"doc_2 related to {query}",
            f"doc_3 related to {query}",
        ][: self.top_k]


rewrite = QueryRewriteStep(lowercase=True)
retrieve = RetrieveStep(top_k=2)

query = rewrite("  What is RAG?  ")
docs = retrieve(query)

print(query)
print(docs)
```

输出：

```text
what is rag?
['doc_1 related to what is rag?', 'doc_2 related to what is rag?']
```

这个例子里：

```python
rewrite(...)
retrieve(...)
```

都是对象调用，本质都是：

```python
rewrite.__call__(...)
retrieve.__call__(...)
```

---

# 14. 一句话总结

```text
__call__ 的作用：
让一个对象可以像函数一样被调用。
```

更工程化地说：

```text
__call__ = 函数式调用形式 + 对象状态封装
```

对于你现在学 Python + 大模型应用，重点记住：

```text
1. obj(...) 等价于 obj.__call__(...)
2. 它常用于模型、工具、回调、装饰器、Pipeline Step
3. 它适合“对象本质上就是一个处理器”的场景
4. 不要给普通 Service 类滥用 __call__
```

Java 类比：

```text
Python __call__
≈ Java Function.apply()
≈ Runnable.run()
≈ Handler.handle()
≈ 策略对象的统一入口
```

但 Python 语法上可以直接写成：

```python
result = handler(input)
```