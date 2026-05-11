
```
这些协议具体规定了什么？长什么样子？在哪儿能看到？

代码中、工作中、实践中，这些协议体现在哪儿？怎么用？
```


# 1. 先把“协议”这个词讲实：它不是一个概念名词，而是一套通信合同

在 Dubbo 里说“协议体系”，不要只理解成：

> Dubbo 支持 dubbo、tri、rest、grpc、hessian、thrift……

更准确地说，**协议规定的是一次远程调用在网络上传输时必须遵守的格式和规则**。

它至少规定这些东西：

|维度|协议要规定什么|在 Dubbo 里的体现|
|---|---|---|
|传输层|走 TCP、HTTP/1.1、HTTP/2、HTTP/3？|`dubbo` 走 TCP，`tri/triple` 基于 HTTP/1/HTTP/2|
|报文边界|一次请求从哪里开始，到哪里结束？|Dubbo2 有固定 16 字节 Header；Triple 使用 HTTP Header + DATA Frame|
|路由语义|调哪个服务？哪个方法？|service/interface、method、version、group、path|
|编码方式|参数和返回值怎么变成字节？|Hessian2、Protobuf、JSON、Fastjson2、Kryo 等|
|调用模型|普通请求响应？流式调用？单向调用？|Unary、Streaming、2-way、1-way、heartbeat|
|元数据|超时、traceId、token、attachment 怎么传？|Dubbo attachment / HTTP header / gRPC metadata|
|错误语义|成功、超时、服务不存在、业务异常怎么表达？|Dubbo status code / HTTP status / grpc-status|
|扩展机制|怎么加过滤器、安全、监控、序列化？|Dubbo SPI：`Protocol`、`Serialization`、`Filter`|

所以你真正要建立的认识是：

> **Dubbo 协议 = “接口调用”变成“网络字节流”的规则。**

---

# 2. Dubbo 3.x 主线协议：`tri/triple` 和 `dubbo`

Dubbo 3.x 里重点看两个：

|协议|定位|适合场景|
|---|---|---|
|`tri` / Triple|Dubbo 3 主力协议，基于 HTTP/1 / HTTP/2，兼容 gRPC|新项目、跨语言、云原生、网关、浏览器/cURL 调试|
|`dubbo` / Dubbo2 TCP|老牌私有 TCP 协议，高性能、紧凑|Java 内部系统、存量 Dubbo2 升级、强 Dubbo SDK 绑定场景|

官方文档明确说，Dubbo 当前支持 `triple`、`dubbo` 以及其他扩展协议；其中 Triple 基于 HTTP/1 和 HTTP/2，兼容 gRPC，并支持 Unary、Streaming 等通信模式；而 `dubbo` 是基于 TCP 的私有高性能协议，更适合 Dubbo SDK 之间调用。([Apache Dubbo](https://dubbo.apache.org/en/overview/mannual/java-sdk/tasks/protocols/protocol/ "RPC Communication Protocols Supported by Dubbo | Apache Dubbo"))

---

# 3. Dubbo2 TCP 协议：它长得像一个“自定义二进制包”

## 3.1 Dubbo2 协议真正规定了什么？

Dubbo2 协议是一个典型的：

```text
固定长度 Header + 变长 Body
```

它的 Header 固定 16 字节。

大致结构是：

```text
+----------------+----------------+----------------+----------------+
| magic 0xdabb   | flags/status   | request id                      |
+----------------+----------------+----------------+----------------+
| request id continued                                              |
+----------------+----------------+----------------+----------------+
| body length                                                     |
+----------------+----------------+----------------+----------------+
| serialized body ...                                              |
+-------------------------------------------------------------------+
```

官方 Dubbo2 协议规范里明确写了这些字段：

|字段|长度|作用|
|---|--:|---|
|Magic|16 bit|魔数，Dubbo 协议是 `0xdabb`|
|Req/Res|1 bit|是请求还是响应|
|2 Way|1 bit|是否需要返回值|
|Event|1 bit|是否事件消息，例如心跳|
|Serialization ID|5 bit|使用哪种序列化方式|
|Status|8 bit|响应状态，例如 OK、超时、服务不存在等|
|Request ID|64 bit|请求唯一 ID，用于请求响应匹配|
|Data Length|32 bit|Body 字节长度|
|Variable Part|不定长|具体调用内容，由序列化协议编码|

官方规范里还说明，Dubbo 请求 Body 里按顺序包含 Dubbo 版本、服务名、服务版本、方法名、参数类型、参数值、attachments；响应 Body 里包含返回值类型和返回值内容。([Apache Dubbo](https://dubbo.apache.org/en/overview/reference/protocols/tcp/ "Dubbo2 Protocol Specification | Apache Dubbo"))

这就非常具体了。

---

## 3.2 一次 Dubbo2 调用在网络上是什么样？

假设你调用：

```java
UserDTO user = userService.getUserById(1001L);
```

在 Java 代码里你看到的是普通接口调用。

但 Dubbo2 协议里，网络上传输的逻辑内容更接近：

```text
Header:
  magic: 0xdabb
  request: true
  twoWay: true
  event: false
  serializationId: 2    # 例如 hessian2，具体以实现映射为准
  requestId: 123456789
  bodyLength: 约 N 字节

Body:
  dubboVersion: "3.x.x"
  serviceName: "com.demo.UserService"
  serviceVersion: "1.0.0"
  methodName: "getUserById"
  parameterTypes: "J"
  arguments:
    1001
  attachments:
    path: "com.demo.UserService"
    interface: "com.demo.UserService"
    timeout: "3000"
    group: ...
    version: ...
    traceId: ...
```

返回时：

```text
Header:
  magic: 0xdabb
  request: false
  status: 20       # OK
  requestId: 123456789
  bodyLength: 约 M 字节

Body:
  responseType: RESPONSE_VALUE
  returnValue:
    UserDTO(id=1001, name="z")
```

所以 Dubbo2 协议不是玄学，它就是：

```text
固定 16 字节头部 + Hessian2/其他序列化后的调用参数
```

---

## 3.3 Dubbo2 协议在代码中体现在哪里？

你在业务代码中通常看不到协议，因为 Dubbo 把它藏在代理、Invoker、Exchange、Codec、Netty 下面了。

业务代码：

```java
@DubboReference
private UserService userService;

UserDTO user = userService.getUserById(1001L);
```

底层链路大概是：

```text
代理对象
  -> Invoker.invoke()
    -> Cluster / LoadBalance / Filter
      -> DubboProtocol
        -> ExchangeClient
          -> Netty Channel
            -> DubboCodec 编码
              -> TCP 字节流
```

在 Dubbo 源码里，协议扩展是通过 SPI 注册的。官方扩展文档里可以看到常用协议映射：

```text
dubbo = org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
tri   = org.apache.dubbo.rpc.protocol.tri.TripleProtocol
rest  = org.apache.dubbo.rpc.protocol.rest.RestProtocol
grpc  = org.apache.dubbo.rpc.protocol.grpc.GrpcProtocol
injvm = org.apache.dubbo.rpc.protocol.injvm.InjvmProtocol
```

这说明你在配置里写的 `dubbo.protocol.name=dubbo` 或 `tri`，最后会通过 SPI 找到对应的 `Protocol` 实现类。([Dubbo](https://dubbo.apache.ac.cn/en/overview/tasks/extensibility/protocol/ "协议 | Apache Dubbo 中文"))

---

# 4. Triple 协议：它长得更像“HTTP/gRPC 风格的 RPC”

## 4.1 Triple 解决了 Dubbo2 的什么问题？

Dubbo2 私有 TCP 协议性能好，但有明显缺点：

```text
只有 Dubbo SDK 真正懂它。
```

这意味着：

|问题|Dubbo2 TCP 的痛点|
|---|---|
|浏览器访问|不方便|
|cURL 调试|不方便|
|API 网关接入|不自然|
|跨语言|成本更高|
|gRPC 生态互通|不天然|
|HTTP/2 Streaming|不天然|

Triple 的设计目标就是把 Dubbo RPC 迁移到更开放的 HTTP/gRPC 生态里。官方 Triple 规范说明，它参考了 gRPC、gRPC-Web 和通用 HTTP，目标是开发友好、兼容基于 HTTP/2 的 gRPC，并支持 Streaming；Triple 可同时运行在 HTTP/1 和 HTTP/2 上。([Apache Dubbo](https://dubbo.apache.org/en/overview/reference/protocols/triple-spec/ "Triple Protocol Design Philosophy and Specification | Apache Dubbo"))

---

## 4.2 Triple Unary 请求长什么样？

官方文档给了一个非常直观的 HTTP/1 示例：

```http
POST /org.apache.dubbo.demo.GreetService/Greet HTTP/1.1
Host: 127.0.0.1:30551
Content-Type: application/json

["Dubbo"]
```

响应：

```http
HTTP/1.1 200 OK
Content-Type: application/json

{"greeting": "Hello, Dubbo!"}
```

官方规范还展示了通过 Header 传超时信息的例子，例如 `Rest-service-timeout: 5000`。([Apache Dubbo](https://dubbo.apache.org/en/overview/reference/protocols/triple-spec/ "Triple Protocol Design Philosophy and Specification | Apache Dubbo"))

你会发现 Triple 的“长相”明显更接近 HTTP API：

```text
POST /接口全限定名/方法名
Content-Type: application/json 或 application/proto
Body: 参数数组或 Protobuf 消息
```

所以 Triple 的一个重大变化是：

> **Dubbo 调用可以被 HTTP 工具直接观察、理解、调试。**

---

## 4.3 Triple Streaming 请求长什么样？

Triple 在 HTTP/2 下兼容 gRPC 风格。

官方规范里的 HTTP/2 Streaming 形态大概是：

```text
HEADERS:
  :method = POST
  :path = /google.pubsub.v2.PublisherService/CreateTopic
  content-type = application/grpc+proto
  grpc-timeout = 1S

DATA:
  <Length-Prefixed Message>

Response HEADERS:
  :status = 200
  content-type = application/grpc+proto

DATA:
  <Length-Prefixed Message>

Trailers:
  grpc-status = 0
```

官方说明，Triple 只有在 HTTP/2 上支持 Streaming RPC；为了兼容 gRPC，HTTP/2 下的 Triple 实现，包括 Streaming RPC，与标准 gRPC 协议保持一致。([Apache Dubbo](https://dubbo.apache.org/en/overview/reference/protocols/triple-spec/ "Triple Protocol Design Philosophy and Specification | Apache Dubbo"))

这就解释了为什么 Dubbo 3 的 Triple 可以做这些事情：

```text
Unary RPC
Client Streaming
Server Streaming
Bidirectional Streaming
gRPC 互通
HTTP 网关接入
cURL / Postman 调试
```

---

# 5. “协议”和“序列化”不要混在一起

这是学习 Dubbo 协议时最容易混的地方。

## 5.1 协议负责“外壳”，序列化负责“内容”

可以这样理解：

```text
协议 = 快递箱规格
序列化 = 箱子里面东西的打包方式
```

比如 Dubbo2：

```text
Dubbo2 TCP Header
  + Hessian2 Body
```

比如 Triple：

```text
HTTP/2 Header
  + Protobuf Body
```

或者：

```text
HTTP/1.1 Header
  + JSON Body
```

官方序列化文档也把 RPC 通信协议和序列化协议分开说明：Triple IDL 模式默认使用 protobuf / protobuf-json；Triple Java Interface 模式可以使用 hessian；Dubbo 协议 Java Interface 模式默认使用 hessian，并且还支持 protostuff、gson、avro、msgpack、kryo、fastjson2 等。([Apache Dubbo](https://dubbo.apache.org/en/overview/mannual/java-sdk/reference-manual/serialization/serialization/ "Introduction to Dubbo Serialization Mechanism | Apache Dubbo"))

---

## 5.2 你在配置里看到的就是这两层

例如：

```yaml
dubbo:
  protocol:
    name: tri
    port: 50051
  provider:
    serialization: protobuf
```

这里：

```text
name: tri
```

决定的是：

```text
通信协议用 Triple
```

而：

```text
serialization: protobuf
```

决定的是：

```text
参数和返回值怎么编码成字节
```

再比如：

```yaml
dubbo:
  protocol:
    name: dubbo
    port: 20880
  provider:
    serialization: hessian2
```

表示：

```text
通信协议：Dubbo2 TCP
序列化：Hessian2
```

---

# 6. 这些协议在实际项目里体现在哪儿？

## 6.1 体现一：配置文件

最直接的体现就是 provider 暴露服务时配置协议。

### 使用 Triple

```yaml
dubbo:
  application:
    name: user-service
  registry:
    address: nacos://127.0.0.1:8848
  protocol:
    name: tri
    port: 50051
```

### 使用 Dubbo2 TCP

```yaml
dubbo:
  application:
    name: user-service
  registry:
    address: nacos://127.0.0.1:8848
  protocol:
    name: dubbo
    port: 20880
```

### 同时暴露多个协议

```yaml
dubbo:
  protocols:
    tri:
      name: tri
      port: 50051
    dubbo:
      name: dubbo
      port: 20880
```

业务含义是：

```text
同一个服务可以同时提供 Triple 入口和 Dubbo2 TCP 入口。
```

这在迁移期很常见：

```text
老 Java 消费者继续走 dubbo://
新消费者或跨语言消费者走 tri://
```

---

## 6.2 体现二：注册中心里的 URL

Dubbo 服务注册到 Nacos/Zookeeper 后，本质上会注册服务 URL。

你可以把它理解成类似：

```text
tri://192.168.1.10:50051/com.demo.UserService
  ?application=user-service
  &version=1.0.0
  &serialization=protobuf
  &methods=getUserById,createUser
```

或者：

```text
dubbo://192.168.1.10:20880/com.demo.UserService
  ?application=user-service
  &version=1.0.0
  &serialization=hessian2
  &methods=getUserById,createUser
```

消费者订阅服务后，拿到这些 URL，就知道：

```text
调用哪个 IP
调用哪个端口
使用哪个协议
使用哪个序列化方式
有哪些方法
有哪些治理参数
```

所以在工作中排查 Dubbo 问题时，一个核心动作就是看注册中心里的 provider URL。

---

## 6.3 体现三：日志和报错

你在项目里经常会看到类似错误：

```text
No such extension org.apache.dubbo.rpc.Protocol by name tri
```

这说明：

```text
配置里用了 tri 协议，但当前 classpath 里没有对应协议扩展实现。
```

或者：

```text
Unsupported serialization: xxx
```

说明：

```text
消费者或提供者协商到的序列化方式，当前进程不支持。
```

或者：

```text
Failed to bind NettyServer on /0.0.0.0:20880
```

说明：

```text
Dubbo2 TCP 协议端口绑定失败。
```

或者 Triple 下看到 HTTP/gRPC 风格错误：

```text
grpc-status = 14
UNAVAILABLE
```

这说明：

```text
已经进入 Triple/gRPC 语义，而不是传统 Dubbo2 TCP status 语义。
```

---

## 6.4 体现四：抓包和调试方式完全不同

### Dubbo2 TCP

Dubbo2 是私有二进制协议，你用浏览器打不开：

```bash
curl http://127.0.0.1:20880
```

通常没有意义。

你需要：

```bash
tcpdump
Wireshark
Dubbo telnet/qos
Arthas
Dubbo Admin
日志
源码 Debug
```

抓包时你能看到关键魔数：

```text
0xda 0xbb
```

也就是 Dubbo2 协议头里的 `0xdabb`。

### Triple

Triple 是 HTTP/gRPC 友好协议，可以更自然地用：

```bash
curl
grpcurl
Postman
HTTP 网关
Envoy
APISIX
Nginx 部分场景
```

例如 JSON 方式调用可能接近：

```bash
curl \
  -X POST \
  -H "Content-Type: application/json" \
  http://127.0.0.1:50051/com.demo.UserService/getUserById \
  -d '[1001]'
```

实际 path、content-type、参数格式要以你的 Dubbo 版本、接口定义方式、IDL/非 IDL 模式为准，但思路是这个。

---

# 7. Dubbo 协议体系在源码里的主线

你不用一上来啃全源码，先抓这几个点。

## 7.1 `Protocol` 是协议扩展入口

核心抽象大概是：

```java
public interface Protocol {

    <T> Exporter<T> export(Invoker<T> invoker) throws RpcException;

    <T> Invoker<T> refer(Class<T> type, URL url) throws RpcException;

    void destroy();
}
```

它表达的意思是：

|方法|作用|
|---|---|
|`export`|服务提供者暴露服务|
|`refer`|服务消费者引用远程服务|
|`destroy`|销毁协议资源|

所以你配置：

```yaml
dubbo.protocol.name=tri
```

本质就是选择：

```text
TripleProtocol.export()
TripleProtocol.refer()
```

你配置：

```yaml
dubbo.protocol.name=dubbo
```

本质就是选择：

```text
DubboProtocol.export()
DubboProtocol.refer()
```

官方协议扩展文档也展示了 `dubbo`、`tri`、`rest`、`grpc` 等协议名和具体实现类的 SPI 映射关系。([Dubbo](https://dubbo.apache.ac.cn/en/overview/tasks/extensibility/protocol/ "协议 | Apache Dubbo 中文"))

---

## 7.2 Provider 侧协议链路

服务提供者启动时：

```text
扫描 @DubboService
  -> 生成 ServiceConfig
    -> 创建 Invoker
      -> Protocol.export(invoker)
        -> 启动协议服务器
          -> 注册 provider URL 到注册中心
```

如果是 `dubbo` 协议：

```text
DubboProtocol
  -> NettyServer
  -> DubboCodec
  -> TCP 20880
```

如果是 `tri` 协议：

```text
TripleProtocol
  -> HTTP/1 或 HTTP/2 Server
  -> Triple/gRPC 编解码
  -> 50051 或你配置的端口
```

---

## 7.3 Consumer 侧协议链路

消费者启动时：

```text
扫描 @DubboReference
  -> 从注册中心订阅 provider URL
    -> 根据 URL 协议选择 Protocol.refer()
      -> 创建远程 Invoker
        -> 生成接口代理对象
```

调用时：

```text
userService.getUserById(1001)
  -> 代理对象拦截
    -> Invoker.invoke()
      -> Filter 链
        -> Cluster 容错
          -> LoadBalance 选择 Provider
            -> Protocol Client 编码
              -> 网络发送
```

这就是为什么你业务代码像本地调用，但底层其实是协议调用。

---

# 8. 工作中怎么选择协议？

## 8.1 新项目默认优先 Triple

建议：

```yaml
dubbo:
  protocol:
    name: tri
```

原因：

|理由|说明|
|---|---|
|云原生友好|HTTP/2、gRPC 生态更成熟|
|跨语言友好|Protobuf + gRPC 兼容|
|调试友好|可以用 HTTP/gRPC 工具|
|网关友好|更容易接 API 网关、Service Mesh|
|支持 Streaming|适合 AI、推送、实时数据等场景|

Dubbo 3 官方也强调 Triple 是基于 HTTP/2 的新协议，兼容 gRPC，支持 Protobuf 和多种 Streaming 模式。([Apache Dubbo](https://dubbo.apache.org/en/docs/v3.0/whats-new-in-dubbo3/?utm_source=chatgpt.com "What's new in Apache Dubbo 3.x?"))

---

## 8.2 存量 Java 内部系统可以继续 Dubbo2 TCP

适合：

```text
纯 Java 内部 RPC
历史系统已经稳定
没有跨语言需求
没有 HTTP 网关需求
追求极致紧凑二进制协议
```

配置：

```yaml
dubbo:
  protocol:
    name: dubbo
    port: 20880
```

Dubbo2 TCP 的优点是协议紧凑，Header 设计非常小；官方协议分析也提到它的设计很紧凑，例如可以用 1 bit 表示的布尔标识就不会用一个字节。([Apache Dubbo](https://dubbo.apache.org/en/overview/reference/protocols/tcp/ "Dubbo2 Protocol Specification | Apache Dubbo"))

---

## 8.3 迁移期可以双协议暴露

例如：

```yaml
dubbo:
  protocols:
    old:
      name: dubbo
      port: 20880
    new:
      name: tri
      port: 50051
```

策略：

```text
老消费者继续走 dubbo
新消费者逐步切 tri
跨语言消费者直接走 tri/gRPC
稳定后下掉 dubbo
```

这比“一刀切升级协议”安全。

---

# 9. 代码里怎么“用协议”？

## 9.1 普通 Java Interface 模式

接口：

```java
public interface UserService {
    UserDTO getUserById(Long userId);
}
```

Provider：

```java
@DubboService(version = "1.0.0")
public class UserServiceImpl implements UserService {

    @Override
    public UserDTO getUserById(Long userId) {
        return new UserDTO(userId, "z");
    }
}
```

Consumer：

```java
@Component
public class UserClient {

    @DubboReference(version = "1.0.0")
    private UserService userService;

    public UserDTO query(Long userId) {
        return userService.getUserById(userId);
    }
}
```

协议不写在业务代码里，而是在配置里：

```yaml
dubbo:
  protocol:
    name: tri
    port: 50051
```

或者：

```yaml
dubbo:
  protocol:
    name: dubbo
    port: 20880
```

这就是工作中最常见的体现：

> **业务代码不关心协议，部署配置决定协议。**

---

## 9.2 IDL / Protobuf 模式

如果你希望跨语言、更标准化，可以定义 `.proto`：

```proto
syntax = "proto3";

package demo;

service UserService {
  rpc GetUserById (GetUserRequest) returns (GetUserResponse);
}

message GetUserRequest {
  int64 userId = 1;
}

message GetUserResponse {
  int64 userId = 1;
  string name = 2;
}
```

这个模式下，Triple + Protobuf 的优势更明显：

```text
Java Provider
Go Consumer
Python Consumer
gRPC 工具调试
统一 IDL 契约
```

官方序列化文档也说明，Triple IDL 模式默认使用 protobuf / protobuf-json；Java Interface 模式也可以使用其他序列化方式。([Apache Dubbo](https://dubbo.apache.org/en/overview/mannual/java-sdk/reference-manual/serialization/serialization/ "Introduction to Dubbo Serialization Mechanism | Apache Dubbo"))

---

# 10. 怎么真正“看到”协议？

## 10.1 看官方协议规范

你主要看这几类：

|内容|看什么|
|---|---|
|Dubbo2 TCP 报文结构|Dubbo2 Protocol Specification|
|Triple 设计和报文示例|Triple Protocol Design Philosophy and Specification|
|协议选择和配置|RPC Communication Protocols Supported by Dubbo|
|序列化支持|Serialization Mechanism|
|SPI 扩展|Protocol Extension|

上面这些官方页面已经给出了 Header 字段、HTTP 示例、Streaming 示例、序列化组合和 SPI 映射。([Apache Dubbo](https://dubbo.apache.org/en/overview/reference/protocols/tcp/ "Dubbo2 Protocol Specification | Apache Dubbo"))

---

## 10.2 看源码路径

重点看这些包名即可：

```text
org.apache.dubbo.rpc.Protocol
org.apache.dubbo.rpc.protocol.dubbo.DubboProtocol
org.apache.dubbo.rpc.protocol.tri.TripleProtocol
org.apache.dubbo.remoting.exchange
org.apache.dubbo.remoting.transport.netty4
org.apache.dubbo.remoting.exchange.codec
org.apache.dubbo.common.serialize
```

你要带着问题看源码，不要泛读：

|问题|看哪里|
|---|---|
|配置 `tri` 怎么找到 TripleProtocol？|Dubbo SPI / `META-INF/dubbo`|
|Provider 怎么暴露端口？|`Protocol.export()`|
|Consumer 怎么建立连接？|`Protocol.refer()`|
|请求怎么编码成字节？|Codec / Serialization|
|返回怎么匹配请求？|requestId / Future|
|attachment 怎么传？|RpcContext / Invocation attachments|
|心跳怎么实现？|event flag / heartbeat|

---

## 10.3 用抓包看 Dubbo2

启动一个 Dubbo2 TCP 服务：

```yaml
dubbo:
  protocol:
    name: dubbo
    port: 20880
```

抓包：

```bash
tcpdump -i lo -nn -X port 20880
```

你应该能在字节流里看到类似：

```text
da bb
```

这就是 Dubbo2 协议魔数 `0xdabb`。

看到它，你就知道：

```text
这是 Dubbo2 TCP 报文开始。
```

---

## 10.4 用 curl / grpcurl 看 Triple

Triple 如果以 HTTP/JSON 方式暴露，可以尝试：

```bash
curl \
  -X POST \
  -H "Content-Type: application/json" \
  http://127.0.0.1:50051/com.demo.UserService/getUserById \
  -d '[1001]'
```

如果是 Protobuf/gRPC 风格，可以尝试：

```bash
grpcurl \
  -plaintext \
  -d '{"userId": 1001}' \
  127.0.0.1:50051 \
  demo.UserService/GetUserById
```

这就是 Triple 相比 Dubbo2 TCP 最直观的体验差异：

```text
Dubbo2 更像内部私有 RPC。
Triple 更像开放 HTTP/gRPC RPC。
```

---

# 11. 一个完整调用链：从 Java 方法到协议报文

假设代码是：

```java
@DubboReference
private OrderService orderService;

orderService.createOrder(command);
```

底层可以拆成：

```mermaid
flowchart LR
    A[Java接口调用] --> B[动态代理]
    B --> C[Invocation]
    C --> D[Filter链]
    D --> E[Cluster容错]
    E --> F[LoadBalance选择Provider]
    F --> G[Protocol客户端]
    G --> H[Serialization序列化]
    H --> I[网络报文]
    I --> J[Provider协议服务器]
    J --> K[反序列化]
    K --> L[调用真实Service]
    L --> M[返回结果]
```

如果是 `dubbo` 协议：

```text
I = Dubbo2 TCP Header + Hessian2 Body
```

如果是 `tri` 协议：

```text
I = HTTP Header + Protobuf/JSON Body
```

所以“协议体系”最终就落在这一步：

```text
Invocation 怎么被编码成网络报文。
```

---

# 12. 实战排查时，按这张表看

|现象|优先怀疑|
|---|---|
|服务注册了但调不通|注册中心 URL 里的协议、端口、IP|
|`No such extension Protocol by name tri`|缺少 Triple 协议实现依赖或版本不匹配|
|消费者找不到 Provider|group/version/interface/protocol 不匹配|
|参数反序列化失败|DTO 类结构、包名、序列化方式、白名单|
|Triple 用 curl 不通|path、content-type、HTTP/1/HTTP/2 模式不匹配|
|gRPC 客户端调不通 Triple|proto service/method 名、package、reflection、plaintext/TLS|
|Dubbo2 抓包看不懂|正常，私有二进制协议，需要按 Header 解析|
|超时|timeout attachment / grpc-timeout / 网络延迟 / 线程池耗尽|
|老服务升级失败|协议和序列化同时变化，应该分阶段迁移|

---

# 13. 最核心的理解

你可以这样记：

```text
Dubbo 的业务模型：
  调接口、调方法、传参数、拿返回值。

Dubbo 的协议模型：
  把接口名、方法名、参数、attachments、requestId、状态码编码成网络报文。

Dubbo2 协议：
  自定义 TCP 二进制协议，16 字节 Header + 序列化 Body。

Triple 协议：
  HTTP/gRPC 风格协议，HTTP Header + Protobuf/JSON Body，支持 Unary 和 Streaming。

序列化协议：
  负责 Body 里面的对象怎么变成字节，不等于 RPC 通信协议。
```

工作中的落点就是四个地方：

```text
1. application.yml 里的 protocol.name
2. 注册中心里的 provider URL
3. 网络抓包里的 Header / HTTP path / content-type
4. 源码里的 Protocol.export / Protocol.refer / Codec / Serialization
```

这四个地方看明白，Dubbo 协议体系就不再是“概念”。