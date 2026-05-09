[小傅哥基础版](https://github.com/fuzhengwei/CodeGuide/blob/master/docs/md/road-map/ddd-guide-02.md)
# DDD 建模方法：从业务现场到 Java 代码

先给结论：

> **DDD 建模不是画类图，也不是换包名，而是把业务复杂度切成可理解、可演进、可实现的模型边界。**

它要解决的问题不是：

```text
Controller、Service、Mapper 怎么分？
```

而是：

```text
这个业务到底由哪些子领域组成？
每个领域里的核心概念是什么？
哪些规则必须强一致？
哪些流程可以最终一致？
哪些对象应该放在一起？
哪些对象绝对不能揉在一起？
```

---

# 1. DDD 建模的最终产物是什么？

DDD 建模最后不是一张漂亮图，而是这些东西：

|建模产物|说明|最终代码表现|
|---|---|---|
|子领域|业务能力拆分|`order`、`payment`、`inventory`|
|界限上下文|模型语义边界|独立模块 / 包 / 服务|
|通用语言|业务和技术共用词汇|类名、方法名、事件名|
|聚合|一致性边界|`Order`、`Cart`、`Account`|
|实体|有身份、有生命周期的对象|`OrderItem`、`User`|
|值对象|无身份，只看值|`Money`、`Address`|
|命令|业务意图|`PayOrderCommand`|
|领域事件|已发生的业务事实|`OrderPaidEvent`|
|领域服务|不适合放入单个实体的领域规则|`PricingService`|
|仓储|聚合的保存和重建|`OrderRepository`|
|防腐层|外部系统隔离|`PaymentGateway`|

一句话：

> **建模的终点，是能指导代码结构、事务边界、对象职责和系统协作。**

---

# 2. DDD 建模的核心心法

## 2.1 不要从数据库开始

传统 CRUD 思维是：

```text
表
  ↓
DO / Entity
  ↓
Mapper
  ↓
Service
  ↓
Controller
```

DDD 思维是：

```text
业务场景
  ↓
业务动作
  ↓
业务事实
  ↓
业务规则
  ↓
领域对象
  ↓
一致性边界
  ↓
代码结构
```

比如订单系统，不要一开始问：

```text
order 表怎么设计？
order_item 表几个字段？
status 用 varchar 还是 int？
```

而是先问：

```text
订单是怎么产生的？
订单什么时候可以支付？
订单什么时候不能取消？
支付成功后哪些事情必须发生？
库存扣减和订单支付是否必须在同一个事务里？
```

---

## 2.2 建模不是为了复刻现实世界

这是初学者常犯的错误。

比如现实中有：

```text
用户
商品
购物车
订单
支付
库存
优惠券
物流
商家
```

你不能直接把这些名词全部建成对象，然后塞进一个大模型里。

DDD 建模关注的是：

```text
这些概念在当前业务场景中承担什么职责？
它们属于同一个模型边界吗？
它们之间需要强一致吗？
它们是否只是其他上下文的引用？
```

例如在订单上下文里，商品不一定是完整的 `Product`。

更合理的是：

```java
public class OrderItem {
    private ProductId productId;
    private String productNameSnapshot;
    private Money salePrice;
    private int quantity;
}
```

为什么？

因为订单关心的是：

```text
下单当时买了什么
当时价格是多少
数量是多少
```

它不应该关心商品上下文里的：

```text
商品上下架
商品详情页图片
商品类目
商品搜索标签
商品 SEO 信息
```

这就是模型边界。

---

# 3. DDD 建模的 7 步法

我建议你按这 7 步学。

```text
业务场景
  ↓
用例分析
  ↓
事件风暴
  ↓
子领域划分
  ↓
界限上下文划分
  ↓
聚合设计
  ↓
代码落地
```

下面逐步拆。

---

# 4. 第一步：选择业务场景

DDD 建模不能脱离业务场景空谈。

不要一上来建：

```text
电商系统
```

这个太大。

应该选择一个具体业务闭环：

```text
用户下单支付
```

这个场景足够小，但又包含订单、库存、支付、营销等多个领域，非常适合练习。

业务流程可以先写成自然语言：

```text
用户选择商品后提交订单。
系统校验商品、价格、库存和优惠。
订单创建成功后，系统锁定库存。
用户发起支付。
支付成功后，订单状态变为已支付。
系统发布订单已支付事件。
库存系统扣减库存。
营销系统核销优惠。
物流系统准备发货。
```

这一步的目标不是设计代码，而是把业务说清楚。

---

# 5. 第二步：识别用例

用例是参与者想完成的业务目标。

参与者：

```text
买家
商家
支付系统
库存系统
营销系统
物流系统
```

核心用例：

```text
买家提交订单
买家支付订单
买家取消订单
系统锁定库存
系统扣减库存
系统核销优惠
商家发货
买家确认收货
```

这里要注意：  
**用例不是接口。**

错误理解：

```text
POST /orders
POST /orders/{id}/pay
GET /orders/{id}
```

正确理解：

```text
提交订单
支付订单
取消订单
确认收货
```

接口只是用例的一种技术入口。

---

# 6. 第三步：事件风暴

事件风暴是 DDD 建模里非常关键的一步。

它的目标是：

> **先找业务事实，再反推命令、规则、对象和边界。**

## 6.1 先找领域事件

领域事件是已经发生的业务事实，通常用过去时命名：

```text
OrderCreatedEvent       订单已创建
InventoryLockedEvent    库存已锁定
PaymentSucceededEvent   支付已成功
OrderPaidEvent          订单已支付
CouponUsedEvent         优惠券已使用
StockDeductedEvent      库存已扣减
OrderCancelledEvent     订单已取消
OrderShippedEvent       订单已发货
OrderReceivedEvent      订单已收货
```

这里有一个关键点：

> **领域事件不是 MQ 消息的同义词。**

领域事件首先是业务事实。  
至于后面要不要发 MQ，那是技术实现问题。

---

## 6.2 再找命令

命令表示系统要执行的业务意图。

```text
CreateOrderCommand      创建订单
LockInventoryCommand    锁定库存
PayOrderCommand         支付订单
UseCouponCommand        使用优惠券
DeductStockCommand      扣减库存
CancelOrderCommand      取消订单
ShipOrderCommand        发货
ReceiveOrderCommand     确认收货
```

命令和事件的关系：

```text
CreateOrderCommand  -> OrderCreatedEvent
PayOrderCommand     -> OrderPaidEvent
CancelOrderCommand  -> OrderCancelledEvent
```

区别很重要：

|类型|含义|时态|是否一定成功|
|---|---|---|---|
|Command|希望系统做什么|将来/现在|不一定|
|Event|已经发生了什么|过去|已发生|

比如：

```text
支付订单     是命令
订单已支付   是事件
```

---

## 6.3 找策略和规则

命令变成事件，中间一定有规则。

例如 `PayOrderCommand -> OrderPaidEvent` 中间有规则：

```text
订单必须存在
订单状态必须是 CREATED
支付金额必须等于订单应付金额
订单不能已取消
支付结果必须成功
支付单不能重复绑定
```

这些规则就是领域模型的核心。

如果你没有识别规则，DDD 就会退化成 CRUD。

---

## 6.4 找业务对象

围绕规则，开始识别对象。

订单上下文里有：

```text
Order
OrderItem
Money
Address
OrderStatus
PaymentId
ProductId
UserId
```

库存上下文里有：

```text
Inventory
Stock
StockReservation
SkuId
```

支付上下文里有：

```text
PaymentOrder
PaymentChannel
PaymentStatus
TransactionId
```

营销上下文里有：

```text
Coupon
Promotion
DiscountRule
UserCoupon
```

这时候你会发现：  
同一个词在不同上下文里含义不一样。

例如 `商品`：

|上下文|商品含义|
|---|---|
|商品上下文|可编辑、可上下架、可配置详情|
|订单上下文|下单时的商品快照|
|库存上下文|SKU 库存单位|
|营销上下文|可参与活动的促销对象|

这就是 DDD 里说的：

> **同名概念，不一定是同一个模型。**

---

# 7. 第四步：划分子领域

子领域是从业务能力角度拆分系统。

电商下单支付可以拆成：

```text
商品子领域
订单子领域
库存子领域
支付子领域
营销子领域
物流子领域
会员子领域
```

DDD 里常见的子领域类型有三种：

|类型|含义|例子|
|---|---|---|
|核心域|公司竞争力所在|订单履约、营销策略、风控决策|
|支撑域|支撑核心业务|库存、会员、结算|
|通用域|可外购或通用能力|短信、支付通道、文件存储|

这一步的目的：

> **判断哪里值得精细建模，哪里可以简单处理。**

不是所有模块都值得 DDD。

例如短信发送：

```text
SmsService.send()
```

够了，不需要复杂建模。

但营销规则、订单状态流转、库存占用释放，就值得认真建模。

---

# 8. 第五步：划分界限上下文

这是 DDD 最重要的部分之一。

## 8.1 什么是界限上下文？

界限上下文就是：

> **一套模型只在一个明确边界内成立。**

例如 `Order` 这个词在订单上下文里成立。

但是支付上下文不应该直接依赖订单上下文里的 `Order` 对象。

支付上下文应该有自己的模型：

```java
public class PaymentOrder {
    private PaymentOrderId id;
    private String bizOrderNo;
    private Money amount;
    private PaymentStatus status;
}
```

它只需要知道：

```text
业务订单号
支付金额
支付状态
支付渠道
交易流水
```

它不需要知道：

```text
订单有几个商品
订单收货地址是什么
订单优惠券是什么
```

---

## 8.2 上下文之间怎么协作？

常见方式有四种：

```text
ID 引用
领域事件
应用服务编排
防腐层 ACL
```

例如订单支付：

```text
Order Context
  ↓ 发布 OrderPaidEvent
Inventory Context
  ↓ 消费事件并扣减库存
Promotion Context
  ↓ 消费事件并核销优惠
Logistics Context
  ↓ 消费事件并准备发货
```

不要这样：

```java
order.getInventory().deduct();
order.getCoupon().use();
order.getLogistics().createDelivery();
```

这是把多个上下文揉成一团。

---

# 9. 第六步：设计聚合

聚合是 DDD 建模最容易误解的概念。

## 9.2 判断聚合边界的几个问题

设计聚合时，问这几个问题：

```text
这个对象是否必须跟聚合根一起修改？
这个对象是否能脱离聚合根独立存在？
这个规则是否必须在同一个事务内保证？
外部是否需要直接修改这个对象？
这个对象是否属于另一个上下文？
```

以 `OrderItem` 为例：

```text
OrderItem 不能脱离 Order 独立存在
OrderItem 的数量影响订单总金额
OrderItem 的变更必须通过 Order
```

所以它适合放在 `Order` 聚合内。

以 `Payment` 为例：

```text
Payment 有自己的生命周期
Payment 有自己的状态流转
Payment 可能涉及第三方通道
Payment 不应该被 Order 内部直接修改
```

所以它不适合放进 `Order` 聚合。

---

## 9.1 聚合不是对象集合

很多人以为：

```text
Order + OrderItem + Payment + Coupon + Inventory = Order 聚合
```

这是错的。

聚合的核心是：

> **一致性边界。**

也就是说：

```text
哪些对象必须在一次事务里保持业务一致？
```

订单聚合通常包括：

```text
Order
OrderItem
Money
Address
OrderStatus
```

不应该包括：

```text
Payment
Inventory
Coupon
Logistics
```

因为这些通常属于其他上下文。

---

# 10. 第七步：落到 Java 代码💡

下面是一个较标准的 DDD 结构。

```text
com.example.mall
  ├── order
  │   ├── trigger
  │   │   ├── http
  │   │   │   └── OrderController.java
  │   │   └── mq
  │   │       └── PaymentSucceededListener.java
  │   │
  │   ├── application
  │   │   ├── OrderApplicationService.java
  │   │   ├── command
  │   │   │   ├── CreateOrderCommand.java
  │   │   │   └── PayOrderCommand.java
  │   │   └── query
  │   │       └── OrderQueryService.java
  │   │
  │   ├── domain
  │   │   ├── model
  │   │   │   ├── Order.java
  │   │   │   ├── OrderItem.java
  │   │   │   ├── Money.java
  │   │   │   ├── Address.java
  │   │   │   └── OrderStatus.java
  │   │   ├── event
  │   │   │   ├── OrderCreatedEvent.java
  │   │   │   └── OrderPaidEvent.java
  │   │   ├── repository
  │   │   │   └── OrderRepository.java
  │   │   └── service
  │   │       └── OrderPricingService.java
  │   │
  │   └── infrastructure
  │       ├── persistence
  │       │   ├── OrderJpaEntity.java
  │       │   ├── OrderJpaRepository.java
  │       │   └── OrderRepositoryImpl.java
  │       ├── acl
  │       │   ├── PaymentGateway.java
  │       │   └── InventoryGateway.java
  │       └── event
  │           └── DomainEventPublisherImpl.java
```

---

# 11. 核心代码示例

## 11.1 Command

```java
public record PayOrderCommand(
        String orderId,
        String userId,
        String paymentChannel
) {
}
```

Command 表示业务意图，不应该塞太多领域逻辑。

---

## 11.2 领域事件

```java
public record OrderPaidEvent(
        String orderId,
        String userId,
        String paymentId,
        BigDecimal paidAmount,
        LocalDateTime occurredAt
) {
}
```

事件表示已经发生的事实，所以命名用过去式。

---

## 11.3 值对象 Money

```java
public final class Money {

    private final BigDecimal amount;
    private final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        if (amount == null || currency == null) {
            throw new IllegalArgumentException("amount and currency must not be null");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("amount must not be negative");
        }
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        checkCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public boolean equalsTo(Money other) {
        return this.currency.equals(other.currency)
                && this.amount.compareTo(other.amount) == 0;
    }

    private void checkCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("currency mismatch");
        }
    }

    public BigDecimal amount() {
        return amount;
    }

    public Currency currency() {
        return currency;
    }
}
```

值对象的特点：

```text
没有 ID
不可变
通过值判断相等
封装基础规则
```

---

## 11.4 Entity：OrderItem

```java
public class OrderItem {

    private final String orderItemId;
    private final String productId;
    private final String productName;
    private final Money unitPrice;
    private final int quantity;

    public OrderItem(
            String orderItemId,
            String productId,
            String productName,
            Money unitPrice,
            int quantity
    ) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("quantity must be positive");
        }

        this.orderItemId = orderItemId;
        this.productId = productId;
        this.productName = productName;
        this.unitPrice = unitPrice;
        this.quantity = quantity;
    }

    public Money subtotal() {
        return new Money(
                unitPrice.amount().multiply(BigDecimal.valueOf(quantity)),
                unitPrice.currency()
        );
    }

    public String orderItemId() {
        return orderItemId;
    }
}
```

Entity 的特点：

```text
有身份
有生命周期
可以变化
通过 ID 判断身份
```

---

## 11.5 聚合根：Order

```java
public class Order {

    private final String orderId;
    private final String userId;
    private final List<OrderItem> items;
    private final Money totalAmount;
    private OrderStatus status;
    private String paymentId;

    private final List<Object> domainEvents = new ArrayList<>();

    public Order(
            String orderId,
            String userId,
            List<OrderItem> items,
            Money totalAmount
    ) {
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("order items must not be empty");
        }

        this.orderId = orderId;
        this.userId = userId;
        this.items = List.copyOf(items);
        this.totalAmount = totalAmount;
        this.status = OrderStatus.CREATED;

        registerEvent(new OrderCreatedEvent(orderId, userId, LocalDateTime.now()));
    }

    public void pay(String paymentId, Money paidAmount) {
        if (this.status != OrderStatus.CREATED) {
            throw new IllegalStateException("only CREATED order can be paid");
        }

        if (!this.totalAmount.equalsTo(paidAmount)) {
            throw new IllegalArgumentException("paid amount does not match order amount");
        }

        this.paymentId = paymentId;
        this.status = OrderStatus.PAID;

        registerEvent(new OrderPaidEvent(
                this.orderId,
                this.userId,
                paymentId,
                paidAmount.amount(),
                LocalDateTime.now()
        ));
    }

    public void cancel() {
        if (this.status == OrderStatus.PAID) {
            throw new IllegalStateException("paid order cannot be cancelled directly");
        }

        if (this.status == OrderStatus.CANCELLED) {
            return;
        }

        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(orderId, userId, LocalDateTime.now()));
    }

    private void registerEvent(Object event) {
        this.domainEvents.add(event);
    }

    public List<Object> pullDomainEvents() {
        List<Object> events = List.copyOf(domainEvents);
        domainEvents.clear();
        return events;
    }

    public String orderId() {
        return orderId;
    }

    public OrderStatus status() {
        return status;
    }
}
```

重点看这几个方法：

```java
order.pay(paymentId, paidAmount);
order.cancel();
```

它们不是 setter。  
它们是业务行为。

DDD 的核心就是：

> **让领域模型表达业务，而不是让 Service 脚本操纵数据。**

---

## 11.6 Repository

```java
public interface OrderRepository {

    Order findById(String orderId);

    void save(Order order);
}
```

仓储接口属于领域层。

实现放基础设施层：

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderJpaRepository jpaRepository;
    private final OrderConverter converter;

    public OrderRepositoryImpl(
            OrderJpaRepository jpaRepository,
            OrderConverter converter
    ) {
        this.jpaRepository = jpaRepository;
        this.converter = converter;
    }

    @Override
    public Order findById(String orderId) {
        OrderJpaEntity entity = jpaRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("order not found"));

        return converter.toDomain(entity);
    }

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = converter.toEntity(order);
        jpaRepository.save(entity);
    }
}
```

注意：

> **领域层依赖仓储接口，基础设施层实现仓储接口。**

这就是依赖倒置。

---

## 11.7 Application Service

```java
@Service
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final DomainEventPublisher eventPublisher;

    public OrderApplicationService(
            OrderRepository orderRepository,
            PaymentGateway paymentGateway,
            DomainEventPublisher eventPublisher
    ) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public void pay(PayOrderCommand command) {
        Order order = orderRepository.findById(command.orderId());

        PaymentResult paymentResult = paymentGateway.pay(
                command.orderId(),
                command.userId(),
                order.payableAmount(),
                command.paymentChannel()
        );

        order.pay(paymentResult.paymentId(), paymentResult.paidAmount());

        orderRepository.save(order);

        eventPublisher.publish(order.pullDomainEvents());
    }
}
```

Application Service 的职责是：

```text
加载聚合
调用外部系统
调用领域对象执行业务行为
保存聚合
发布事件
控制事务
```

它不应该写大量业务规则。

错误写法：

```java
if (order.getStatus() != OrderStatus.CREATED) {
    throw new RuntimeException("...");
}
order.setStatus(OrderStatus.PAID);
```

正确写法：

```java
order.pay(paymentId, paidAmount);
```

---

# 12. DDD 建模的关键判断：规则放哪里？

这是 Java 后端最容易混乱的地方。

## 12.1 放 Entity / Aggregate Root

如果规则属于某个对象自身，并且依赖对象内部状态，放聚合根或实体。

例如：

```text
订单只有 CREATED 状态才能支付
订单已支付后不能直接取消
订单明细不能为空
```

放在：

```java
Order.pay()
Order.cancel()
```

---

## 12.2 放 Value Object

如果规则是值本身的约束，放值对象。

例如：

```text
金额不能为负
币种必须一致才能相加
地址不能为空
手机号格式校验
```

放在：

```java
Money
Address
PhoneNumber
```

---

## 12.3 放 Domain Service

如果规则不自然属于某一个实体，但仍然是领域规则，放领域服务。

例如：

```text
订单定价
跨多个促销规则计算最优价格
风险评分
复杂库存分配
```

示例：

```java
public class OrderPricingService {

    public Money calculate(List<OrderItem> items, DiscountPolicy discountPolicy) {
        Money originAmount = items.stream()
                .map(OrderItem::subtotal)
                .reduce(Money.zero(Currency.getInstance("CNY")), Money::add);

        return discountPolicy.apply(originAmount);
    }
}
```

---

## 12.4 放 Application Service

如果是流程编排，不是领域规则，放应用服务。

例如：

```text
先查订单
再调支付系统
再保存订单
再发布事件
```

这是应用层职责。

---

## 12.5 放 Infrastructure

如果是技术细节，放基础设施层。

例如：

```text
JPA
MyBatis
Redis
MQ
HTTP Client
第三方支付 SDK
序列化
缓存
```

不要污染领域模型。

---

# 13. 一张判断表

|代码逻辑|应该放哪里|
|---|---|
|订单能不能支付|`Order.pay()`|
|金额能不能相加|`Money.add()`|
|如何计算复杂促销价格|`DomainService`|
|先调支付再改订单状态|`ApplicationService`|
|调用微信支付 API|`Infrastructure / Gateway`|
|保存订单到 MySQL|`RepositoryImpl`|
|HTTP 参数校验|`Controller / DTO`|
|查询订单详情页|`QueryService`|

---

# 14. DDD 与 MVC 的本质区别

MVC 项目通常长这样：

```java
@Service
public class OrderService {

    public void pay(Long orderId) {
        OrderDO order = orderMapper.selectById(orderId);

        if (!"CREATED".equals(order.getStatus())) {
            throw new RuntimeException("订单状态错误");
        }

        paymentClient.pay(order.getAmount());

        order.setStatus("PAID");
        orderMapper.update(order);

        mq.send("ORDER_PAID", orderId);
    }
}
```

问题是：

```text
Service 变成上帝类
DO 只是数据容器
业务规则散落在各处
状态变化靠 setter
模型没有表达力
后续需求一改就到处 if else
```

DDD 写法：

```java
@Transactional
public void pay(PayOrderCommand command) {
    Order order = orderRepository.findById(command.orderId());

    PaymentResult result = paymentGateway.pay(...);

    order.pay(result.paymentId(), result.paidAmount());

    orderRepository.save(order);

    eventPublisher.publish(order.pullDomainEvents());
}
```

区别不在包名，而在：

```text
业务规则由领域对象承载
应用服务只做编排
基础设施细节被隔离
聚合保证一致性
上下文之间通过事件/ID/防腐层协作
```

---

# 15. 建模时最重要的一条线：一致性边界

DDD 里很多概念都服务于一个问题：

> **什么必须强一致，什么可以最终一致？**

电商支付场景：

|业务动作|一致性要求|
|---|---|
|订单状态从 CREATED 到 PAID|强一致|
|订单支付金额校验|强一致|
|订单明细和订单总价|强一致|
|库存最终扣减|可以最终一致，视业务而定|
|发优惠券|通常最终一致|
|发短信通知|最终一致|
|创建物流单|最终一致|

所以订单聚合只保证：

```text
订单内部一致性
```

不要妄图一个聚合保证整个电商系统的一致性。

---

# 16. 事件驱动不是 DDD 的必选项，但很常见

DDD 经常和领域事件一起出现，但要分清两层。

## 16.1 领域事件

```java
new OrderPaidEvent(...)
```

这是领域模型内部产生的业务事实。

## 16.2 集成事件

```text
MQ message: order.paid
```

这是系统之间通信的技术消息。

它们可以相同，也可以不同。

更严谨的做法：

```text
Domain Event
  ↓
Outbox
  ↓
Integration Event
  ↓
MQ
  ↓
Other Context
```

例如：

```text
OrderPaidEvent
  ↓ 转换
OrderPaidIntegrationEvent
  ↓ Kafka / RocketMQ
Inventory Context
```

这样做可以避免领域模型被 MQ 格式绑死。

---

# 17. CQRS：查询和命令可以分开

DDD 里还有一个实用原则：

> **写模型关注业务规则，读模型关注查询效率。**

例如订单详情页需要展示：

```text
订单信息
商品名称
商品图片
支付信息
优惠信息
物流信息
```

不要强行通过 `Order` 聚合拼出来。

可以单独做查询模型：

```java
public class OrderDetailQueryService {

    public OrderDetailDTO getOrderDetail(String orderId) {
        // 可以 JOIN，可以查 ES，可以查 Redis，可以查宽表
    }
}
```

也就是说：

```text
Command Side：领域模型，保证规则
Query Side：读模型，服务页面
```

这能避免很多 DDD 项目过度设计。

---

# 18. DDD 建模常见误区

## 误区 1：把数据库 Entity 当领域 Entity

错误：

```java
@TableName("t_order")
public class Order {
    private Long id;
    private Integer status;
    private BigDecimal amount;
}
```

这只是持久化对象，不是领域模型。

领域模型应该有行为：

```java
order.pay();
order.cancel();
order.changeAddress();
```

---

## 误区 2：所有模块都套 DDD

不是所有地方都需要 DDD。

适合 DDD：

```text
订单
支付
营销
库存
风控
信贷
结算
权限策略
工作流
```

不太需要复杂 DDD：

```text
字典表管理
Banner 配置
简单 CRUD 后台
文件上传
短信发送
系统参数配置
```

---

## 误区 3：聚合设计过大

错误：

```text
Order 聚合包含 Payment、Inventory、Coupon、Logistics
```

结果：

```text
事务巨大
对象巨大
并发差
边界混乱
难以维护
```

正确：

```text
Order 聚合只管订单内部规则
Payment、Inventory、Promotion 各自独立
通过事件或应用服务协作
```

---

## 误区 4：Application Service 写成事务脚本

错误：

```java
public void pay() {
    if (...) {}
    if (...) {}
    if (...) {}
    order.setStatus(...);
}
```

正确：

```java
order.pay(...);
```

Service 不应该成为业务规则垃圾桶。

---

## 误区 5：领域事件滥用

不是所有事情都要事件。

简单同步调用够用时，就同步调用。

事件适合：

```text
跨上下文解耦
异步处理
最终一致
一对多后续动作
审计和通知
```

不适合：

```text
必须立即返回结果的核心校验
强一致事务内规则
简单方法调用
```

---

# 19. 用一句话区分核心概念

|概念|一句话|
|---|---|
|子领域|业务能力拆分|
|界限上下文|模型语义边界|
|实体|有身份、有生命周期|
|值对象|无身份，只看值|
|聚合|一致性边界|
|聚合根|聚合对外唯一入口|
|领域服务|不属于单个对象的领域规则|
|应用服务|业务流程编排|
|仓储|聚合的保存和重建|
|领域事件|已经发生的业务事实|
|防腐层|隔离外部系统模型污染|

---

# 20. 最后给你一套实战建模模板

以后你看到任何业务需求，都可以按这个模板走。

## 20.1 业务场景

```text
这个业务闭环是什么？
谁发起？
目标是什么？
成功结果是什么？
失败情况是什么？
```

## 20.2 命令

```text
用户/系统会发起哪些动作？
这些动作的入参是什么？
动作是否可能失败？
```

## 20.3 事件

```text
动作成功后，发生了什么业务事实？
哪些事件会被其他领域关心？
```

## 20.4 领域对象

```text
规则围绕哪些业务对象发生？
哪些对象有身份？
哪些对象只是值？
哪些对象有生命周期？
```

## 20.5 聚合

```text
哪些对象必须强一致？
谁是聚合根？
外部是否只能通过聚合根修改内部对象？
聚合是否太大？
```

## 20.6 上下文

```text
哪些概念在不同业务域含义不同？
哪些模型不应该互相引用？
上下文之间通过什么协作？
```

## 20.7 代码落地

```text
Controller / Listener / Scheduler 作为触发器
Application Service 编排流程
Domain Model 承载规则
Repository 保存聚合
Infrastructure 对接数据库、MQ、外部系统
QueryService 处理复杂查询
```

---

# 21. 你真正应该记住的版本

DDD 建模可以压缩成这条主线：

```text
先找业务事件
再反推业务命令
再识别业务规则
再沉淀领域对象
再设计聚合边界
再划分上下文
最后落到代码结构
```

或者更短：

```text
事件定事实
命令定意图
对象定规则
聚合定一致性
上下文定边界
应用层定流程
基础设施定技术细节
```

这才是 DDD 建模的骨架。

小傅哥那篇文章的价值，是把你带进“事件风暴/四色建模”的门。  
但要讲到 10 分，必须补上三件事：

```text
1. 为什么这样建模
2. 建模结果如何判断对错
3. 如何落到 Java 代码和工程边界
```

这三件事补齐之后，DDD 才不再是玄学。