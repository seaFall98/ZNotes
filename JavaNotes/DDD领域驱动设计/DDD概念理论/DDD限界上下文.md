## 1. “限界上下文”是什么？

**限界上下文**，英文是 **Bounded Context**。

一句话：

> **限界上下文就是一个业务模型有效的边界。**

再直白一点：

> 同一个词，在不同业务场景下含义可能不同。  
> 限界上下文就是用来明确：**这个词、这个模型、这套规则，到底在哪个范围内成立。**

DDD 里非常重视这个概念，因为复杂系统最大的问题不是代码复杂，而是：

> **业务语言混乱，模型边界混乱，职责混乱。**

限界上下文就是用来解决这个问题的。

---

# 2. 为什么需要限界上下文？

假设你做一个电商系统，里面都有一个词叫：

```text
商品
```

但是不同部门眼里的“商品”并不是一个东西。

|上下文|“商品”关心什么|
|---|---|
|商品上下文|标题、描述、类目、品牌、图片、规格|
|库存上下文|SKU、仓库、可售库存、锁定库存|
|订单上下文|下单时的商品快照、价格、数量|
|营销上下文|是否参与活动、优惠规则、标签|
|搜索上下文|关键词、分词、权重、排序因子|

你会发现：

```text
商品 ≠ 商品 ≠ 商品
```

它们名字一样，但业务含义不同。

如果你强行搞一个统一的 `Product` 类，让所有模块都共用它，就很容易变成：

```java
public class Product {
    private Long id;
    private String title;
    private String description;
    private String brand;
    private String category;
    private BigDecimal price;
    private Integer stock;
    private Integer lockedStock;
    private Boolean onSale;
    private Boolean joinedPromotion;
    private BigDecimal promotionPrice;
    private String searchKeyword;
    private Integer searchWeight;
    private String warehouseCode;
    private List<String> imageUrls;
    private List<Sku> skus;
    // ...
}
```

这就是典型的：

> **大泥球模型 / 上帝类 / 贫血大对象 / 混乱公共模型**

因为你试图用一个模型承载所有业务语义。

DDD 的做法是：

> 不要追求全系统统一模型，而是在不同边界内建立各自清晰的模型。

这就是限界上下文。

---

# 3. 限界上下文不是“模块”那么简单

很多人会把限界上下文理解成：

```text
一个限界上下文 = 一个模块
```

这个理解不完全错，但太浅。

更准确地说：

> **限界上下文是业务语义边界，不是单纯的代码目录边界。**

代码模块只是它的实现方式之一。

比如：

```text
Order Context
```

里面可以包含：

```text
订单聚合
订单明细
订单状态
订单仓储
订单领域服务
订单应用服务
订单事件
订单防腐层
```

它不是简单的：

```text
controller/service/mapper/entity
```

而是一套完整的业务模型边界。

---

# 4. 一个典型例子：订单系统

假设我们有如下几个限界上下文：

```text
Product Context      商品上下文
Order Context        订单上下文
Inventory Context    库存上下文
Payment Context      支付上下文
Promotion Context    营销上下文
```

重点看 **Order Context**。

在订单上下文里，核心模型可能是：

```text
Order
OrderItem
OrderStatus
Money
Address
```

在这个上下文里，“商品”不是完整商品模型，而是订单里的商品快照：

```java
public class OrderItem {
    private ProductId productId;
    private String productName;
    private Money unitPrice;
    private int quantity;
}
```

注意，这里不需要：

```java
商品详情
商品图片
商品类目树
库存数量
搜索权重
营销标签
```

因为订单上下文只关心：

```text
用户买了什么
买的时候多少钱
买了几个
订单总价是多少
订单状态如何变化
```

所以在订单上下文里，商品只是 `OrderItem` 的一部分，而不是商品中心的完整 `Product`。

---

# 5. 限界上下文的核心作用

## 作用一：防止模型膨胀

没有限界上下文时，大家都往同一个模型里塞字段。

商品模块要加字段：

```java
private String categoryName;
```

库存模块要加字段：

```java
private Integer availableStock;
```

营销模块要加字段：

```java
private Boolean joinedCouponCampaign;
```

搜索模块要加字段：

```java
private Integer searchScore;
```

最后 `Product` 变成几百个字段。

限界上下文的思想是：

> 每个上下文只建自己需要的模型。

商品上下文有自己的 `Product`。

订单上下文有自己的 `OrderItem`。

库存上下文有自己的 `StockItem`。

营销上下文有自己的 `PromotionProduct`。

它们不需要强行共用一个类。

---

## 作用二：保证语言一致

DDD 里有一个概念叫：

```text
统一语言 Ubiquitous Language
```

意思是：

> 业务人员、产品经理、开发人员，在同一个上下文内，对同一个词要有一致理解。

比如在订单上下文里：

```text
已支付
已取消
待支付
已完成
```

这些状态要有清晰定义。

```java
public enum OrderStatus {
    CREATED,
    PAID,
    CANCELLED,
    COMPLETED
}
```

但是到了支付上下文里，状态可能是：

```java
public enum PaymentStatus {
    INIT,
    PROCESSING,
    SUCCESS,
    FAILED,
    REFUNDED
}
```

不要强行把 `OrderStatus` 和 `PaymentStatus` 合成一个状态枚举。

因为：

```text
订单已支付 ≠ 支付单成功
```

它们有关联，但不是同一个概念。

---

## 作用三：控制一致性边界

在 DDD 中，一般有一句话：

> **上下文内强一致，上下文间最终一致。**

比如订单上下文内部：

```text
创建订单时，订单金额、订单明细、订单状态必须一致。
```

这通常可以放在一个事务里完成。

但是订单上下文和库存上下文之间：

```text
创建订单 -> 锁定库存
```

这不一定要放在一个大事务里。

更常见的方式是：

```text
订单创建成功 -> 发布 OrderCreatedEvent -> 库存上下文消费事件 -> 锁定库存
```

也就是说：

```text
限界上下文之间通过 API / 消息 / 领域事件协作
```

而不是互相直接操作对方的数据库和领域对象。

---

# 6. 限界上下文和微服务是什么关系？

这是一个非常容易混淆的问题。

## 结论

> **限界上下文是业务边界，微服务是部署边界。**

它们不是一回事。

|概念|本质|
|---|---|
|限界上下文|业务模型边界|
|微服务|运行时部署单元|
|Java Module|代码组织边界|
|数据库 Schema|数据存储边界|

一个限界上下文可以实现为一个微服务：

```text
Order Context -> order-service
```

也可以多个限界上下文先放在一个单体应用里：

```text
shop-app
  ├── product-context
  ├── order-context
  ├── inventory-context
  ├── payment-context
```

DDD 不要求你一上来就拆微服务。

更好的实践通常是：

```text
先拆业务边界，再决定是否拆部署边界。
```

也就是：

> 先做模块化单体，再根据团队规模、性能压力、发布频率、数据隔离要求，决定是否拆成微服务。

---

# 7. 限界上下文和聚合是什么关系？

这两个也经常被混在一起。

## 简单区分

|概念|粒度|解决什么问题|
|---|---|---|
|限界上下文|大|业务模型边界|
|聚合|中|一致性边界|
|实体/值对象|小|领域对象建模|

比如：

```text
Order Context
```

是一个限界上下文。

在它里面可能有多个聚合：

```text
Order 聚合
AfterSale 聚合
Invoice 聚合
```

而 `Order` 聚合里面可能有：

```text
Order       聚合根
OrderItem   实体
Money       值对象
Address     值对象
```

层级大概是：

```text
限界上下文 Bounded Context
  └── 聚合 Aggregate
        └── 实体 Entity
        └── 值对象 Value Object
```

所以不要说：

```text
Order 是一个限界上下文
```

更准确的是：

```text
Order Context 是限界上下文
Order 是 Order Context 里的聚合根
```

---

# 8. 限界上下文之间如何协作？

常见方式有四种。

## 方式一：直接 API 调用

订单上下文调用支付上下文：

```text
Order Context -> Payment Context
```

比如：

```java
paymentClient.createPayment(orderId, amount);
```

适合：

```text
同步查询
需要立即结果
失败需要明确反馈
```

缺点是：

```text
上下文之间会产生运行时依赖
```

---

## 方式二：领域事件 / 消息

订单支付成功后发布事件：

```java
OrderPaidEvent
```

库存上下文、营销上下文、积分上下文都可以订阅：

```text
Order Context -> OrderPaidEvent -> Inventory Context
Order Context -> OrderPaidEvent -> Promotion Context
Order Context -> OrderPaidEvent -> Points Context
```

适合：

```text
异步处理
最终一致
解耦多个上下文
```

例如：

```text
订单已支付 -> 扣减库存
订单已支付 -> 增加积分
订单已支付 -> 发送通知
```

---

## 方式三：防腐层 ACL

如果订单上下文需要调用外部支付系统，但外部系统的模型很难看：

```json
{
  "trade_no": "xxx",
  "pay_amt": 10000,
  "stat": "S",
  "chnl": "ALI"
}
```

不要让这种外部模型污染你的领域模型。

可以做一个 ACL：

```java
public class PaymentAcl {
    public PaymentResult pay(OrderPaymentCommand command) {
        ExternalPaymentResponse response = externalPaymentClient.pay(...);
        return convertToDomainResult(response);
    }
}
```

ACL 的作用是：

```text
把外部模型翻译成当前上下文能理解的模型。
```

---

## 方式四：共享内核 Shared Kernel

少数非常稳定、非常基础的模型可以共享，比如：

```text
Money
UserId
OrderId
```

但要谨慎。

共享内核的问题是：

> 一旦共享，就会产生耦合。

所以不要轻易搞一个：

```text
common-domain
```

然后所有东西都往里面塞。

这很容易变成新的垃圾桶模块。

---

# 9. 如何识别限界上下文？

这是实战中的难点。

可以从几个角度判断。

## 角度一：看业务语言是否变化

如果同一个词在不同场景下含义不同，通常意味着边界存在。

比如：

```text
商品
```

在商品、订单、库存、营销里含义都不同。

这就应该拆上下文。

---

## 角度二：看核心规则是否不同

比如：

```text
订单上下文：关心订单状态流转
库存上下文：关心库存锁定和扣减
支付上下文：关心支付单、退款、对账
```

它们的核心规则完全不同，所以应该分开。

---

## 角度三：看数据变化原因是否不同

同一个对象如果被不同原因频繁修改，说明它可能承担了多个职责。

比如商品数据：

```text
运营修改标题
仓库修改库存
营销修改活动价
搜索调整排序权重
```

这些变化来源不同，最好不要塞进一个大模型。

---

## 角度四：看团队组织

如果公司里天然有不同团队：

```text
商品团队
交易团队
支付团队
履约团队
营销团队
```

这通常也暗示了不同限界上下文。

但是注意：

> 团队边界可以参考，但不能机械照搬。

因为组织结构有时候也是混乱的。

---

# 10. 一个反例：没有限界上下文会怎样？

假设你做一个传统 MVC 项目：

```text
controller
service
mapper
entity
```

你可能会有一个巨大的：

```java
ProductEntity
```

所有业务都用它。

```java
商品详情页用 ProductEntity
购物车用 ProductEntity
订单下单用 ProductEntity
库存扣减用 ProductEntity
营销活动用 ProductEntity
搜索索引用 ProductEntity
```

短期看很爽：

```text
复用性很高
开发很快
字段都在一个对象里
```

长期会出问题：

```text
字段越来越多
逻辑越来越散
一个修改影响多个业务
没人敢删字段
没人敢改状态
service 里充满 if else
模型没有表达力
```

最后变成：

```text
数据库表驱动开发
Service 脚本式编程
领域对象只是 getter/setter
```

DDD 的限界上下文就是在说：

> 不要让一个模型跨越太多业务语义边界。

---

# 11. 代码结构可以怎么落地？

如果是 Java 后端项目，模块化单体可以先这样组织：

```text
com.example.shop
  ├── product
  │   ├── domain
  │   ├── application
  │   ├── infrastructure
  │   └── interfaces
  │
  ├── order
  │   ├── domain
  │   ├── application
  │   ├── infrastructure
  │   └── interfaces
  │
  ├── inventory
  │   ├── domain
  │   ├── application
  │   ├── infrastructure
  │   └── interfaces
  │
  ├── payment
  │   ├── domain
  │   ├── application
  │   ├── infrastructure
  │   └── interfaces
```

在 `order` 里面：

```text
order
  ├── domain
  │   ├── model
  │   │   ├── Order.java
  │   │   ├── OrderItem.java
  │   │   ├── OrderStatus.java
  │   │   ├── Money.java
  │   │   └── Address.java
  │   ├── repository
  │   │   └── OrderRepository.java
  │   └── event
  │       └── OrderPaidEvent.java
  │
  ├── application
  │   └── OrderAppService.java
  │
  ├── infrastructure
  │   ├── persistence
  │   ├── acl
  │   └── messaging
  │
  └── interfaces
      └── OrderController.java
```

重点是：

```text
order 包内部有自己的领域模型
product 包内部也有自己的领域模型
不要到处复用对方的 Entity
```

---

# 12. 最容易犯的错误

## 错误一：把数据库表当成限界上下文

比如看到有表：

```text
product
order
payment
inventory
```

就以为：

```text
每张表一个上下文
```

这是错的。

限界上下文不是数据库表边界，而是业务语义边界。

---

## 错误二：把微服务当成限界上下文

很多项目是这样：

```text
user-service
order-service
product-service
```

但里面还是互相查库、共用 DTO、共用 Entity。

这只是物理拆分，不是 DDD 拆分。

真正的限界上下文要求：

```text
模型独立
语言独立
规则独立
数据所有权清晰
交互方式清晰
```

---

## 错误三：过度拆分

小项目一上来拆成：

```text
商品上下文
库存上下文
订单上下文
支付上下文
优惠券上下文
积分上下文
会员上下文
风控上下文
结算上下文
履约上下文
```

然后每个上下文都搞一套：

```text
domain
application
infrastructure
interfaces
```

最后复杂度爆炸。

DDD 不是为了拆而拆。

更务实的做法是：

> 先识别边界，但实现上可以先合并部署、合并工程。  
> 等复杂度真的上来，再逐步拆开。

---

## 错误四：搞一个巨大的 common 模块

很多项目会出现：

```text
common
  ├── ProductDTO
  ├── OrderDTO
  ├── UserDTO
  ├── PaymentDTO
  ├── CouponDTO
```

然后所有上下文都依赖它。

这会导致：

```text
表面复用，实际耦合。
```

DDD 里更推荐：

```text
各上下文有自己的模型。
必要时通过转换器进行模型转换。
```

---

# 13. 限界上下文的本质

可以把它理解成三句话。

## 第一，模型不是全局通用的

```text
一个模型只在一个明确边界内成立。
```

## 第二，同名概念可以有不同含义

```text
订单上下文里的商品，不等于商品上下文里的商品。
```

## 第三，上下文之间通过明确契约协作

```text
API、消息、事件、ACL，而不是互相侵入模型。
```

---

# 14. 用一句工程化的话总结

对 Java 后端开发来说，限界上下文可以这样理解：

> **限界上下文就是把一个复杂业务系统按业务语义切成若干个“模型自治区”。每个自治区内部有自己的统一语言、领域模型、业务规则和数据所有权；自治区之间通过接口、事件或防腐层协作，而不是共用一个大模型。**

---

# 15. 最后给你一个判断口诀

看到一个业务概念时，可以问：

```text
1. 这个词在不同部门/场景下含义一样吗？
2. 这个模型的核心规则是否一致？
3. 这个数据是谁负责维护的？
4. 修改这个模型会不会影响一堆不相关业务？
5. 它和其他模型是强一致，还是最终一致？
```

如果答案开始变复杂，大概率这里就有：

```text
限界上下文边界
```

尤其记住这一句：

> **限界上下文不是为了把代码切碎，而是为了让模型在正确的边界内保持清晰。**