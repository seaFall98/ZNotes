先不要记“实体、值对象、聚合根”这些细碎名词。

先建立一个整体判断：

> **DDD 是一套把复杂业务系统拆清楚、说清楚、写清楚的方法。**  
> 它先解决“业务边界怎么划”，再解决“每个边界内部代码怎么组织”。

---

# 1. DDD 的全局地图

![[DDD全局认知图.png]]

```mermaid
flowchart TD
    A[复杂业务系统] --> B[战略设计：先划边界]
    B --> B1[领域 Domain]
    B --> B2[子域 Subdomain]
    B --> B3[限界上下文 Bounded Context]
    B --> B4[通用语言 Ubiquitous Language]

    A --> C[战术设计：边界内部怎么写代码]
    C --> C1[实体 Entity]
    C --> C2[值对象 Value Object]
    C --> C3[聚合 Aggregate]
    C --> C4[聚合根 Aggregate Root]
    C --> C5[领域服务 Domain Service]
    C --> C6[领域事件 Domain Event]
    C --> C7[仓储 Repository]
```

你可以先这样记：

|层次|问题|
|---|---|
|战略设计|业务怎么拆？边界怎么划？|
|战术设计|每个业务模块内部，代码怎么建模？|

---

# 2. DDD 最核心的思路

传统开发容易这样：

```mermaid
flowchart LR
    A[前端请求] --> B[Controller]
    B --> C[Service]
    C --> D[Mapper / DAO]
    D --> E[数据库]

    C -.大量业务逻辑.-> C
```

问题是：**Service 越写越胖，业务逻辑都堆在里面。**

DDD 希望变成这样：

```mermaid
flowchart LR
    A[前端请求] --> B[Controller]
    B --> C[Application Service<br/>应用服务：编排流程]
    C --> D[Domain Model<br/>领域模型：承载业务规则]
    C --> E[Repository<br/>保存/读取聚合]
    E --> F[Database]

    D --> D1[实体]
    D --> D2[值对象]
    D --> D3[聚合]
    D --> D4[领域事件]
```

核心区别：

|传统三层|DDD|
|---|---|
|Service 写大量业务逻辑|Domain Model 写核心业务规则|
|Entity 主要映射数据库表|Entity 表达业务对象|
|以数据库表为中心|以业务模型为中心|
|适合 CRUD|适合复杂业务|

---

# 3. DDD 的整体分层图

DDD 落到代码里，常见是四层：

```mermaid
flowchart TD
    A[Interfaces 接口层<br/>Controller / Request / Response]
    B[Application 应用层<br/>用例编排 / 事务 / Command / Query]
    C[Domain 领域层<br/>实体 / 值对象 / 聚合 / 领域服务 / 领域事件]
    D[Infrastructure 基础设施层<br/>数据库 / Redis / MQ / HTTP / AI API]

    A --> B
    B --> C
    B --> D
    D --> C
```

可以简单理解：

|层|职责|
|---|---|
|Interfaces|接 HTTP、RPC、MQ 请求|
|Application|编排一次业务用例|
|Domain|真正的业务规则|
|Infrastructure|技术实现细节|

---

# 4. 用一句话理解每一层

```mermaid
flowchart LR
    A[Controller] -->|接请求| B[Application Service]
    B -->|编排流程| C[Domain Model]
    C -->|执行业务规则| D[Repository]
    D -->|持久化| E[Database]
```

举个“支付订单”的例子：

```text
Controller：
收到 /orders/{id}/pay 请求

Application Service：
查订单 -> 调支付网关 -> 调 order.pay() -> 保存订单 -> 发事件

Domain Model：
判断订单能不能支付，改变订单状态

Repository：
把订单保存到数据库

Infrastructure：
真正执行 SQL、调用支付 API、发 MQ
```

---

# 5. DDD 的两个大脑：战略设计 + 战术设计

## 5.1 战略设计：先拆业务地图

```mermaid
flowchart TD
    A[电商系统] --> B[商品子域]
    A --> C[订单子域]
    A --> D[库存子域]
    A --> E[支付子域]
    A --> F[营销子域]
    A --> G[售后子域]
```

战略设计关注：

> 这个系统里有哪些业务领域？  
> 哪些是核心？  
> 哪些边界应该分开？

例如“商品”这个词，在不同上下文里含义不同：

```mermaid
flowchart TD
    A[商品这个词] --> B[商品上下文<br/>Product：标题、详情、图片、类目]
    A --> C[库存上下文<br/>StockItem：SKU、库存数量、仓库]
    A --> D[订单上下文<br/>OrderItem：下单时的商品快照]
    A --> E[营销上下文<br/>PromotionProduct：活动价、优惠规则]
```

所以 DDD 不鼓励全系统共用一个巨大的 `Product` 对象。

---

## 5.2 战术设计：边界内部怎么建模

以订单上下文为例：

```mermaid
flowchart TD
    A[Order 聚合根] --> B[OrderItem 子实体]
    A --> C[Money 值对象]
    A --> D[Address 值对象]
    A --> E[OrderStatus 状态]
    A --> F[业务行为]
    F --> F1[pay]
    F --> F2[cancel]
    F --> F3[addItem]
    F --> F4[changeAddress]
```

这就是战术设计：

> 在一个业务边界内部，用实体、值对象、聚合、领域事件等对象，把业务规则组织起来。

---

# 6. DDD 中最重要的几个概念关系图

```mermaid
flowchart TD
    A[限界上下文 Bounded Context] --> B[聚合 Aggregate]
    B --> C[聚合根 Aggregate Root]
    B --> D[实体 Entity]
    B --> E[值对象 Value Object]
    C --> F[业务行为]
    F --> G[领域事件 Domain Event]

    H[Repository] --> B
    I[Application Service] --> C
    I --> H
```

你可以先记住这条主线：

```text
限界上下文里面有聚合
聚合由聚合根负责对外
聚合根里面包含实体和值对象
业务行为写在聚合根或领域对象里
状态变化后可以产生领域事件
Repository 负责保存聚合
Application Service 负责调用它们
```

---

# 7. 实体、值对象、聚合根的区别图

```mermaid
flowchart LR
    A[领域对象] --> B[实体 Entity]
    A --> C[值对象 Value Object]
    A --> D[聚合根 Aggregate Root]

    B --> B1[有唯一ID]
    B --> B2[有生命周期]
    B --> B3[状态会变化]

    C --> C1[没有ID]
    C --> C2[只看值是否相等]
    C --> C3[通常不可变]

    D --> D1[也是实体]
    D --> D2[是聚合入口]
    D --> D3[负责保护内部一致性]
```

用订单举例：

```mermaid
flowchart TD
    A[Order<br/>聚合根] --> B[OrderItem<br/>实体]
    A --> C[Money<br/>值对象]
    A --> D[Address<br/>值对象]
    A --> E[OrderStatus<br/>枚举/状态]
```

区别：

|概念|例子|特点|
|---|---|---|
|实体|Order、User、Article|有 ID，有生命周期|
|值对象|Money、Address、Email|没 ID，值相等就是相等|
|聚合根|Order|外部访问聚合的唯一入口|

---

# 8. “聚合”到底是什么？

聚合不是随便把对象放在一起。

聚合的本质是：

> 一组必须保持强一致性的领域对象。

例如订单：

```mermaid
flowchart TD
    A[Order 聚合] --> B[Order 聚合根]
    B --> C[OrderItem 列表]
    B --> D[订单总金额]
    B --> E[订单状态]

    F[业务规则] --> G[订单至少有一个商品]
    F --> H[订单总金额 = 所有订单项小计之和]
    F --> I[已支付订单不能修改金额]
    F --> J[已取消订单不能支付]
```

所以外部不能随便这样改：

```java
order.getItems().add(item);
order.setStatus(PAID);
```

而应该这样：

```java
order.addItem(productId, quantity, price);
order.pay(paymentId);
order.cancel(reason);
```

区别在于：

```text
setStatus(PAID) 只是改数据
pay(paymentId) 表达一个业务动作，并且可以校验规则
```

---

# 9. 一次 DDD 调用流程

以“支付订单”为例：

```mermaid
sequenceDiagram
    participant Client as 前端
    participant Controller as OrderController
    participant App as PayOrderApplicationService
    participant Repo as OrderRepository
    participant Domain as Order 聚合
    participant Pay as PaymentGateway
    participant DB as Database
    participant Event as EventPublisher

    Client->>Controller: POST /orders/{id}/pay
    Controller->>App: PayOrderCommand
    App->>Repo: findById(orderId)
    Repo->>DB: SELECT orders/order_items
    DB-->>Repo: 数据
    Repo-->>App: Order 聚合
    App->>Pay: 发起支付
    Pay-->>App: PaymentResult
    App->>Domain: order.pay(paymentId)
    Domain-->>Domain: 校验状态并修改为 PAID
    App->>Repo: save(order)
    Repo->>DB: UPDATE orders
    App->>Event: publish OrderPaidEvent
    App-->>Controller: success
    Controller-->>Client: 200 OK
```

这个流程里：

|角色|职责|
|---|---|
|Controller|接请求|
|Application Service|编排流程|
|Repository|取出和保存聚合|
|Order 聚合|执行业务规则|
|PaymentGateway|调外部支付系统|
|EventPublisher|发布领域事件|

---

# 10. DDD 和 MVC 的位置关系

你之前问过 MVC 和 DDD 的 VO，这里可以顺便对齐一下：

```mermaid
flowchart TD
    A[MVC] --> A1[Controller]
    A --> A2[View / Response]
    A --> A3[Model]

    B[DDD] --> B1[Application]
    B --> B2[Domain]
    B --> B3[Infrastructure]

    A1 --> B1
    B1 --> B2
    B1 --> B3
```

简单说：

|MVC|DDD|
|---|---|
|更偏表现层组织方式|更偏业务建模方式|
|关心请求怎么进来、响应怎么出去|关心业务规则怎么表达|
|Controller 是入口|Domain 是核心|

所以二者不是替代关系。

可以是：

```text
MVC 负责接口入口
DDD 负责业务核心
```

---

# 11. DDD 和数据库的关系

很多人上来就问：

> DDD 的实体是不是数据库 Entity？

答案：**不是一回事。**

```mermaid
flowchart LR
    A[Domain Model<br/>领域模型] --> B[Mapper / Converter]
    B --> C[Persistence Model<br/>数据库模型]
    C --> D[Database Table]
```

例如：

```text
领域模型：
Order 聚合
- Order
- OrderItem
- Money
- Address

数据库模型：
orders 表
order_items 表
```

它们可以对应，但不应该强行等同。

DDD 更倾向：

```text
先按业务建模，再考虑怎么落库。
```

传统 CRUD 更倾向：

```text
先设计表，再生成 Entity。
```

---

# 12. DDD 推荐的代码结构

不要按技术横切：

```text
controller
service
mapper
entity
dto
```

更推荐按业务上下文纵向切：

```text
com.example.mall
├── order
│   ├── interfaces
│   ├── application
│   ├── domain
│   └── infrastructure
│
├── product
│   ├── interfaces
│   ├── application
│   ├── domain
│   └── infrastructure
│
├── inventory
│   ├── interfaces
│   ├── application
│   ├── domain
│   └── infrastructure
│
└── payment
    ├── interfaces
    ├── application
    ├── domain
    └── infrastructure
```

图示：

```mermaid
flowchart TD
    A[mall 项目] --> B[order 上下文]
    A --> C[product 上下文]
    A --> D[inventory 上下文]
    A --> E[payment 上下文]

    B --> B1[interfaces]
    B --> B2[application]
    B --> B3[domain]
    B --> B4[infrastructure]
```

这样代码结构直接反映业务结构。

---

# 13. 用 DevWiki Studio 类比

如果用你的知识库项目来理解，DDD 的边界可能是：

```mermaid
flowchart TD
    A[DevWiki Studio] --> B[Source 上下文]
    A --> C[Ingestion 上下文]
    A --> D[Wiki 上下文]
    A --> E[Artifact 上下文]
    A --> F[Blog 上下文]
    A --> G[System 上下文]
```

每个上下文内部再建模：

```mermaid
flowchart TD
    A[Wiki 上下文] --> B[WikiDraft 聚合]
    A --> C[WikiPage 聚合]

    B --> B1[WikiDraft 聚合根]
    B --> B2[WikiSection 值对象/子对象]
    B --> B3[WikiDraftStatus]
    B --> B4[submitForReview]
    B --> B5[approve]
    B --> B6[publish]

    C --> C1[WikiPage 聚合根]
    C --> C2[PageContent 值对象]
    C --> C3[PageVersion]
```

例如：

```java
wikiDraft.approve();
wikiDraft.publish();
```

比下面这种更有业务表达力：

```java
wikiDraft.setStatus(2);
wikiDraftMapper.updateById(wikiDraft);
```

---

# 14. DDD 的一张总结构图

你可以把 DDD 记成这一张图：

```mermaid
flowchart TD
    A[业务问题] --> B[领域 Domain]
    B --> C[子域 Subdomain]
    C --> D[限界上下文 Bounded Context]
    D --> E[通用语言 Ubiquitous Language]

    D --> F[聚合 Aggregate]
    F --> G[聚合根 Aggregate Root]
    F --> H[实体 Entity]
    F --> I[值对象 Value Object]

    G --> J[业务行为]
    J --> K[领域事件 Domain Event]

    L[Application Service] --> G
    L --> M[Repository]
    M --> F

    N[Controller] --> L
    O[Infrastructure] --> M
```

顺序是：

```text
业务问题
  -> 领域
  -> 子域
  -> 限界上下文
  -> 聚合
  -> 聚合根
  -> 实体 / 值对象
  -> 业务行为
  -> 领域事件
```

这条链路比单独背概念重要。

---

# 15. 最小心智模型

先不要背全部术语。你只需要先记住 5 个东西。

## 1. 限界上下文

> 把大系统拆成几个业务语义明确的模块。

例如：

```text
订单上下文
库存上下文
支付上下文
商品上下文
```

---

## 2. 聚合

> 在一个上下文内部，把必须强一致的对象放在一起。

例如：

```text
Order + OrderItem + Money + Address
```

---

## 3. 聚合根

> 聚合的唯一入口。

例如：

```text
外部只能调用 order.pay()，不能直接改 orderItem 或 status。
```

---

## 4. 值对象

> 用小对象表达业务值，而不是到处传 String、BigDecimal、Integer。

例如：

```text
Money
Email
Address
SourcePath
DateRange
```

---

## 5. 应用服务

> 负责流程编排，但不负责核心业务规则。

例如：

```text
查订单 -> 调支付 -> order.pay() -> 保存 -> 发事件
```

---

# 16. 最重要的一句话

DDD 不是让你多写几层代码。

它真正想做的是：

> **把业务复杂度从 Service 的 if-else 里，转移到有边界、有语义、有行为的领域模型里。**

所以你以后看到这种代码：

```java
order.setStatus(PAID);
```

要问一句：

> 这里是不是应该是一个业务行为？

比如：

```java
order.pay(paymentId);
```

这就是 DDD 的核心感觉。