# Python 单例模式怎么实现？

先给结论：**Python 里实现单例有很多写法，但最常用、最推荐的是“模块级单例”或“`__new__` 控制实例创建”。**

从 Java 后端视角看，Python 单例比 Java 松一些，因为 Python 本身更动态，也更依赖模块导入缓存机制。

---

# 1. 最简单：模块级单例

Python 里一个 `.py` 文件就是一个模块。模块只会在第一次 `import` 时执行一次，后续 `import` 直接复用缓存。

所以很多时候，Python 不需要专门写单例类。

## config.py

```python
class AppConfig:
    def __init__(self):
        self.app_name = "ai-app"
        self.timeout = 30


# 模块级对象，只会在模块首次 import 时创建一次
config = AppConfig()
```

## main.py

```python
from config import config

print(config.app_name)
```

## service.py

```python
from config import config

print(config.timeout)
```

`main.py` 和 `service.py` 拿到的是同一个 `config` 对象。

这类似 Java 里的：

```java
public class ConfigHolder {
    public static final AppConfig CONFIG = new AppConfig();
}
```

## 优点

| 优点              | 说明                               |
| --------------- | -------------------------------- |
| 简单              | 不需要复杂魔术方法                        |
| Pythonic        | 符合 Python 常见写法                   |
| 适合配置对象          | 比如 LLM client、数据库配置、Embedding 配置 |
| import 缓存天然保证单例 | 同一个进程内只初始化一次                     |

## 缺点

|缺点|说明|
|---|---|
|不够“类级约束”|别人仍然可以手动 `AppConfig()` 创建新对象|
|测试时需要注意状态污染|修改全局对象会影响其他测试|

---

# 2. 用 `__new__` 实现单例

`__new__` 是真正负责**创建对象**的方法。

普通创建对象流程是：

```text
MyClass()
  → __new__ 创建实例
  → __init__ 初始化实例
```

所以可以在 `__new__` 里控制：如果实例已经存在，就直接返回旧实例。

```python
class Singleton:
    _instance = None

    def __new__(cls, *args, **kwargs):
        # 如果还没有实例，就创建一个
        if cls._instance is None:
            cls._instance = super().__new__(cls)

        # 如果已经有实例，直接返回之前那个
        return cls._instance
```

测试：

```python
a = Singleton()
b = Singleton()

print(a is b)  # True
```

`is` 判断的是两个变量是否引用同一个对象。

---

# 3. 注意：`__init__` 会被重复执行

这是很多人第一次写 Python 单例时容易踩的坑。

```python
class AppConfig:
    _instance = None

    def __new__(cls, name: str):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, name: str):
        print("__init__ called")
        self.name = name


a = AppConfig("first")
b = AppConfig("second")

print(a is b)   # True
print(a.name)   # second
```

输出：

```text
__init__ called
__init__ called
True
second
```

虽然对象只创建了一次，但 `__init__` 每次调用类时都会执行。

所以更严谨的写法要加初始化标记。

```python
class AppConfig:
    _instance = None
    _initialized = False

    def __new__(cls, name: str):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self, name: str):
        # 避免重复初始化
        if self.__class__._initialized:
            return

        self.name = name
        self.__class__._initialized = True
```

测试：

```python
a = AppConfig("first")
b = AppConfig("second")

print(a is b)   # True
print(a.name)   # first
print(b.name)   # first
```

---

# 4. 线程安全版本

如果在多线程环境下，两个线程同时第一次创建对象，可能会同时进入：

```python
if cls._instance is None:
```

所以可以加锁。

```python
import threading


class ThreadSafeSingleton:
    _instance = None
    _initialized = False
    _lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        # 第一层判断：避免每次都加锁
        if cls._instance is None:
            with cls._lock:
                # 第二层判断：防止多个线程重复创建
                if cls._instance is None:
                    cls._instance = super().__new__(cls)

        return cls._instance

    def __init__(self, name: str):
        # 初始化也需要防止重复执行
        if self.__class__._initialized:
            return

        with self.__class__._lock:
            if self.__class__._initialized:
                return

            self.name = name
            self.__class__._initialized = True
```

测试：

```python
a = ThreadSafeSingleton("first")
b = ThreadSafeSingleton("second")

print(a is b)   # True
print(a.name)   # first
```

这个写法类似 Java 的双重检查锁：

```java
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

---

# 5. 用装饰器实现单例

装饰器版本适合把单例逻辑复用到多个类上。

```python
def singleton(cls):
    instances = {}

    def get_instance(*args, **kwargs):
        # 每个被装饰的类只创建一个实例
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance
```

使用：

```python
@singleton
class LLMClient:
    def __init__(self, model: str):
        self.model = model


a = LLMClient("gpt-4.1")
b = LLMClient("gpt-5")

print(a is b)       # True
print(a.model)      # gpt-4.1
```

## 注意

装饰器写法有一个问题：`LLMClient` 被替换成了函数 `get_instance`，不再是原始类本身。

```python
print(type(LLMClient))
# <class 'function'>
```

所以如果你依赖继承、类型判断、类属性，装饰器单例可能不太合适。

---

# 6. 用元类实现单例

你前面问过元类。单例也可以用元类实现。

```python
class SingletonMeta(type):
    _instances = {}

    def __call__(cls, *args, **kwargs):
        # cls 表示当前要实例化的类
        if cls not in cls._instances:
            # super().__call__ 会触发 __new__ 和 __init__
            cls._instances[cls] = super().__call__(*args, **kwargs)

        return cls._instances[cls]
```

使用：

```python
class LLMClient(metaclass=SingletonMeta):
    def __init__(self, model: str):
        self.model = model


a = LLMClient("gpt-4.1")
b = LLMClient("gpt-5")

print(a is b)       # True
print(a.model)      # gpt-4.1
```

这里的关键是：

```text
LLMClient(...)
  → SingletonMeta.__call__(...)
  → 如果没有实例，创建
  → 如果已有实例，直接返回
```

这比 `__new__` 更“框架化”，因为元类拦截的是**类被调用创建对象的过程**。

---

# 7. 元类线程安全版本

```python
import threading


class SingletonMeta(type):
    _instances = {}
    _lock = threading.Lock()

    def __call__(cls, *args, **kwargs):
        # 双重检查，减少锁竞争
        if cls not in cls._instances:
            with cls._lock:
                if cls not in cls._instances:
                    cls._instances[cls] = super().__call__(*args, **kwargs)

        return cls._instances[cls]
```

使用：

```python
class EmbeddingClient(metaclass=SingletonMeta):
    def __init__(self, model: str):
        self.model = model


client1 = EmbeddingClient("text-embedding-3-small")
client2 = EmbeddingClient("another-model")

print(client1 is client2)   # True
print(client1.model)        # text-embedding-3-small
```

---

# 8. AI 应用里常见的单例对象

在 Python + 大模型应用里，单例经常用于这些对象：

|对象|是否适合单例|原因|
|---|--:|---|
|配置对象|适合|全局统一配置|
|LLM Client|适合|复用 HTTP 连接、模型配置、限流器|
|Embedding Client|适合|复用模型客户端|
|Vector Store Client|适合|复用数据库连接|
|Tokenizer|适合|初始化成本高|
|Prompt Template Registry|适合|全局模板注册表|
|Logger|适合|全局日志入口|
|User Session|不适合|每个用户状态不同|
|Request Context|不适合|每次请求不同|
|Conversation Memory|通常不适合全局单例|容易串用户数据|

尤其要注意：

```text
不要把用户会话、对话历史、权限上下文做成全局单例。
```

否则在 Web 服务里可能出现严重问题：

```text
A 用户的问题污染 B 用户的上下文
请求之间状态串扰
并发安全问题
数据泄露
```

---

# 9. 推荐你实际项目这么写

## 场景一：全局配置

用模块级单例。

```python
# settings.py
from dataclasses import dataclass


@dataclass(frozen=True)
class Settings:
    app_name: str = "rag-service"
    llm_model: str = "gpt-4.1"
    top_k: int = 5


settings = Settings()
```

使用：

```python
from settings import settings

print(settings.llm_model)
```

`frozen=True` 表示这个 dataclass 创建后尽量不可变，类似 Java 里的不可变配置对象。

---

## 场景二：LLM Client

可以用模块级工厂加缓存。

```python
# llm_client.py
class LLMClient:
    def __init__(self, model: str, timeout: int):
        self.model = model
        self.timeout = timeout

    def chat(self, prompt: str) -> str:
        # 实际项目中这里会调用大模型 API
        return f"[{self.model}] {prompt}"


_client: LLMClient | None = None


def get_llm_client() -> LLMClient:
    global _client

    if _client is None:
        _client = LLMClient(
            model="gpt-4.1",
            timeout=30,
        )

    return _client
```

使用：

```python
from llm_client import get_llm_client

client = get_llm_client()
answer = client.chat("解释 RAG")
```

这个写法在工程里很常见，优点是：

```text
初始化时机可控
测试时容易替换
比硬编码全局对象更灵活
```

---

# 10. FastAPI 里更推荐依赖注入

如果你后面用 FastAPI 写 AI 服务，不一定要自己手写单例，可以用依赖注入和缓存。

```python
from functools import lru_cache
from fastapi import Depends, FastAPI

app = FastAPI()


class LLMClient:
    def __init__(self, model: str):
        self.model = model

    def chat(self, prompt: str) -> str:
        return f"[{self.model}] {prompt}"


@lru_cache
def get_llm_client() -> LLMClient:
    # lru_cache 会缓存函数返回值
    return LLMClient(model="gpt-4.1")


@app.get("/chat")
def chat(
    question: str,
    client: LLMClient = Depends(get_llm_client),
):
    return {
        "answer": client.chat(question)
    }
```

这里：

```python
@lru_cache
def get_llm_client() -> LLMClient:
    ...
```

会让 `get_llm_client()` 第一次调用时创建对象，后续复用。

这个方式非常适合：

```text
配置对象
数据库客户端
LLM 客户端
Embedding 客户端
向量库客户端
```

---

# 11. 各种实现方式怎么选？

|实现方式|推荐度|适合场景|
|---|--:|---|
|模块级单例|高|配置、全局客户端、注册表|
|`get_xxx()` + 缓存|高|需要懒加载、可测试、依赖注入|
|`@lru_cache`|高|FastAPI 依赖、配置加载|
|`__new__`|中|学习对象创建机制，或类本身要强约束|
|元类|中低|框架级封装，多个类共享单例规则|
|装饰器|中低|简单复用，但会改变类对象|
|全局可变对象|谨慎|容易导致状态污染|

---

# 12. 你现在应该重点掌握哪几个？

以你现在的方向：**Java 后端 + AI 应用 + Python 基础**，建议重点掌握三种：

## 第一种：模块级单例

```python
# config.py
config = AppConfig()
```

这是最 Pythonic 的方式。

## 第二种：工厂函数缓存

```python
_client = None

def get_client():
    global _client
    if _client is None:
        _client = Client()
    return _client
```

适合工程项目。

## 第三种：`@lru_cache`

```python
from functools import lru_cache

@lru_cache
def get_settings():
    return Settings()
```

适合 FastAPI、AI 服务依赖注入。

---

# 13. 最小可用模板：AI Client 单例

你可以直接记这个模板。

```python
from functools import lru_cache
from dataclasses import dataclass


@dataclass(frozen=True)
class LLMSettings:
    model: str = "gpt-4.1"
    timeout: int = 30


class LLMClient:
    def __init__(self, settings: LLMSettings):
        self.settings = settings

    def chat(self, prompt: str) -> str:
        # 实际项目中这里替换为真实的大模型 API 调用
        return f"model={self.settings.model}, prompt={prompt}"


@lru_cache
def get_llm_client() -> LLMClient:
    # 第一次调用创建，后续调用直接复用
    settings = LLMSettings()
    return LLMClient(settings)


if __name__ == "__main__":
    client1 = get_llm_client()
    client2 = get_llm_client()

    print(client1 is client2)  # True
    print(client1.chat("什么是 RAG？"))
```

这比你一上来写复杂元类更适合实际 AI 应用。

---

# 14. 最小记忆版

```text
Python 单例优先级：

1. 配置对象：模块级单例
2. 客户端对象：get_client() + 缓存
3. FastAPI：Depends + @lru_cache
4. 学习底层机制：__new__
5. 框架级通用封装：元类
```

一句话：

> **Python 里不必像 Java 那样执着于私有构造器 + static getInstance；更多时候用模块导入缓存、工厂函数缓存、依赖注入就够了。**