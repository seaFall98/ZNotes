# 总述

在 DDD 中，每一个领域都是界限上下文拆分的独立结果，而实现业务流程的功能则需要串联各个领域模块提供一整条链路的完整服务。所以也常说领域内事务一致性，领域外最终一致性。

同时这些领域模块因为是独立的，所以也可以被复用。在不同的场景功能诉求下，可以选择不同的领域模块进行组装，这个过程就像搭积木一样。

但这里有一个取舍，如果项目相对来说并不大，也没有太多的编排处理。那么可以直接让触发器层对接领域层，减少编排层后，编码会更加便捷。


# DDD 里的“领域编排”是什么？

一句话：

> **领域编排就是把多个领域能力、仓储、外部适配器、事务、事件、流程步骤串起来，完成一个完整业务用例。**

它通常发生在：

```text
Application Service 应用服务层
```

而不是领域对象内部。

---

# 1. 为什么需要领域编排？

单个领域模型通常只负责自己的业务规则。

例如订单领域：

```text
订单可以创建
订单可以提交
订单可以取消
订单可以支付成功
```

库存领域：

```text
库存可以锁定
库存可以扣减
库存可以释放
```

支付领域：

```text
支付单可以创建
支付可以确认
支付可以退款
```

优惠领域：

```text
优惠券可以校验
优惠券可以核销
优惠券可以退回
```

但是一个完整的“下单流程”不是单个领域对象能完成的。

它可能是：

```text
校验用户
校验商品
计算价格
校验优惠券
创建订单
锁定库存
创建支付单
发送领域事件
返回下单结果
```

这就是编排。

---

# 2. 领域能力 vs 领域编排

要区分两个概念：

|概念|负责什么|典型位置|
|---|---|---|
|领域能力|单个领域内部的核心业务规则|Domain 层|
|领域编排|把多个领域能力串成完整用例|Application 层|

举个例子：

```java
order.submit();
```

这是领域能力。

```java
orderRepository.find();
inventoryGateway.lockStock();
couponGateway.freezeCoupon();
order.submit();
orderRepository.save();
eventPublisher.publish();
```

这是领域编排。

---

# 3. 领域编排不等于领域服务

这是很多人容易混淆的点。

## 领域服务 Domain Service

领域服务负责**领域内的业务规则**，通常是某个规则不适合放到单个实体或值对象里。

例如：

```java
public class PricingDomainService {

    public Money calculatePrice(Order order, PromotionPlan promotionPlan) {
        // 复杂价格计算规则
    }
}
```

它还是在表达业务规则。

---

## 应用服务 Application Service

应用服务负责**用例编排**。

例如：

```java
@Service
public class SubmitOrderApplicationService {

    private final OrderRepository orderRepository;
    private final InventoryGateway inventoryGateway;
    private final CouponGateway couponGateway;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public SubmitOrderResult submit(SubmitOrderCommand command) {
        Order order = orderRepository.find(command.orderId());

        couponGateway.freeze(command.couponId());
        inventoryGateway.lock(order.items());

        order.submit();

        orderRepository.save(order);
        eventPublisher.publish(new OrderSubmittedEvent(order.id()));

        return SubmitOrderResult.success(order.id());
    }
}
```

这个类本身不应该写大量业务判断。

它应该主要负责：

```text
查对象
调能力
控事务
发事件
返回结果
```

---

# 4. 一个关键判断标准

当你不知道某段逻辑应该放领域层还是应用层，可以问：

> **这段逻辑是在表达业务规则，还是在安排执行步骤？**

如果是业务规则，放领域层。

```java
if (order.status() != OrderStatus.CREATED) {
    throw new DomainException("只有待创建订单才能提交");
}
```

如果是执行步骤，放应用层。

```java
Order order = orderRepository.find(orderId);
inventoryGateway.lock(order.items());
order.submit();
orderRepository.save(order);
```

---

# 5. 领域编排的典型内容

领域编排一般包含这些东西：

```text
1. 接收 Command
2. 加载聚合
3. 调用领域对象行为
4. 调用领域服务
5. 调用 Repository 保存聚合
6. 调用 Gateway / Adapter 访问外部能力
7. 控制事务边界
8. 发布领域事件
9. 组装返回 DTO
```

例如：

```java
public PayOrderResult pay(PayOrderCommand command) {
    Order order = orderRepository.find(command.orderId());

    Payment payment = paymentFactory.create(order, command.payMethod());

    paymentGateway.pay(payment);

    order.markPaying(payment.id());

    orderRepository.save(order);
    paymentRepository.save(payment);

    return PayOrderResult.of(order.id(), payment.id());
}
```

这里的编排流程是：

```text
加载订单
创建支付单
调用支付网关
修改订单状态
保存订单
保存支付单
返回结果
```

---

# 6. 领域内事务一致性，领域外最终一致性

你提到的这句话很关键：

> **领域内事务一致性，领域外最终一致性。**

可以这样理解。

---

## 6.1 领域内事务一致性

一个聚合内部的状态变化，应该在一个事务内完成。

例如订单聚合内部：

```text
订单主表
订单明细
订单收货地址
订单价格快照
```

它们属于同一个订单聚合。

保存订单时应该整体一致：

```java
@Transactional
public void createOrder(CreateOrderCommand command) {
    Order order = orderFactory.create(command);
    orderRepository.save(order);
}
```

如果保存订单成功，但订单明细失败，这就是严重问题。

所以领域内要求强一致。

---

## 6.2 领域外最终一致性

订单领域和库存领域、优惠券领域、积分领域、物流领域，通常不应该全部塞进一个本地事务。

否则会变成：

```text
订单事务
+ 库存事务
+ 优惠券事务
+ 支付事务
+ 积分事务
+ MQ 事务
```

这会带来：

```text
事务范围过大
锁时间过长
系统耦合严重
失败回滚复杂
性能下降
可用性下降
```

所以更常见的做法是：

```text
订单先完成自己的状态变更
然后通过事件通知其他领域继续处理
失败后通过重试、补偿、对账实现最终一致
```

例如：

```text
订单已创建
    ↓ 发布事件
OrderCreatedEvent
    ↓
库存服务锁库存
优惠券服务冻结券
积分服务预占积分
```

---

# 7. 同步编排 vs 异步编排

领域编排通常有两种方式。

---

## 7.1 同步编排

应用服务直接调用多个领域能力。

```java
@Transactional
public SubmitOrderResult submit(SubmitOrderCommand command) {
    Order order = orderFactory.create(command);

    inventoryGateway.lock(order.items());
    couponGateway.freeze(command.couponId());

    order.submit();

    orderRepository.save(order);

    return SubmitOrderResult.success(order.id());
}
```

适合：

```text
流程简单
调用链短
一致性要求高
用户需要立即知道结果
失败可以直接返回
```

缺点：

```text
强耦合
链路变长
一个外部服务失败会影响整体流程
不适合复杂长流程
```

---

## 7.2 异步编排

应用服务只完成本领域事务，然后发事件。

```java
@Transactional
public CreateOrderResult create(CreateOrderCommand command) {
    Order order = orderFactory.create(command);

    orderRepository.save(order);

    domainEventPublisher.publish(new OrderCreatedEvent(order.id()));

    return CreateOrderResult.success(order.id());
}
```

然后其他处理器订阅事件：

```java
@Component
public class OrderCreatedEventHandler {

    public void handle(OrderCreatedEvent event) {
        inventoryGateway.lock(event.orderId());
        couponGateway.freeze(event.orderId());
    }
}
```

适合：

```text
流程复杂
跨多个领域
允许稍后完成
需要削峰填谷
需要解耦上下游
```

缺点：

```text
状态会变复杂
需要重试
需要幂等
需要补偿
需要对账
用户感知不是立即完成
```

---

# 8. 编排和协同的区别

在分布式系统里，经常会说：

```text
Orchestration 编排
Choreography 协同 / 编舞
```

## 编排 Orchestration

有一个中心流程控制者。

```text
OrderApplicationService
    ↓
InventoryGateway
    ↓
CouponGateway
    ↓
PaymentGateway
```

特点：

```text
流程清晰
容易追踪
中心编排器较重
上下游依赖集中
```

---

## 协同 Choreography

没有中心控制者，各领域通过事件自己响应。

```text
OrderCreatedEvent
    ↓
InventoryHandler
CouponHandler
PointHandler
RiskControlHandler
```

特点：

```text
解耦程度高
扩展方便
链路不直观
排查问题更困难
一致性治理更复杂
```

---

# 9. 领域编排放在哪一层？

常见分层是：

```text
interfaces
  └── controller

application
  ├── service
  ├── command
  ├── dto
  └── assembler

domain
  ├── model
  ├── service
  ├── repository
  └── event

infrastructure
  ├── persistence
  ├── rpc
  ├── mq
  └── cache
```

领域编排通常放在：

```text
application/service
```

例如：

```text
CreateOrderApplicationService
SubmitOrderApplicationService
PayOrderApplicationService
CancelOrderApplicationService
```

不要把复杂编排塞进 Controller。

Controller 应该只负责：

```text
接收请求
参数转换
调用应用服务
返回响应
```

不要把复杂编排塞进 Domain Entity。

Entity 应该只负责：

```text
维护自身状态
保证自身业务不变量
暴露领域行为
```

---

# 10. 触发器层直接对接领域层可以吗？

你提到：

> 如果项目不大，也没有太多编排处理，可以直接让触发器层对接领域层，减少编排层。

可以，但要明确代价。

所谓触发器层一般包括：

```text
Controller
MQ Consumer
定时任务
RPC Provider
CLI Command
```

它们都是业务流程的入口。

小项目中可以这样：

```java
@RestController
public class OrderController {

    private final OrderRepository orderRepository;

    @PostMapping("/orders/{id}/submit")
    public void submit(@PathVariable Long id) {
        Order order = orderRepository.find(new OrderId(id));
        order.submit();
        orderRepository.save(order);
    }
}
```

这确实简单。

适合：

```text
CRUD 为主
业务流程短
没有跨领域编排
没有复杂事务
团队规模小
项目生命周期短
```

但如果流程变复杂，Controller 很快会变成：

```text
参数校验
查库
调 RPC
调 Redis
调 MQ
控制事务
领域规则
异常转换
返回 DTO
```

这时候就应该引入应用层。

---

# 11. 是否需要应用层的判断标准

可以按这个标准判断。

## 可以省略应用层

当满足这些条件时，可以少一层：

```text
一个接口只操作一个聚合
没有跨领域调用
没有外部系统编排
没有复杂事务控制
没有异步事件
没有复杂权限和流程状态
```

例如：

```text
创建分类
修改昵称
更新文章标题
启用/禁用标签
```

---

## 应该引入应用层

只要出现下面情况，就建议引入：

```text
一个用例涉及多个聚合
一个用例涉及多个领域模块
需要调用外部系统
需要控制事务边界
需要发领域事件
需要幂等、重试、补偿
需要组合多个领域能力完成一个业务流程
需要同时支持 HTTP、MQ、定时任务等多个入口
```

例如：

```text
提交订单
支付回调
取消订单
退款申请
合同签署
课程报名
用户注册后初始化资产
文章发布后生成索引和通知
```

---

# 12. 用你熟悉的 Java 后端视角理解

可以把 Application Service 理解为：

> **Use Case Service，用例服务。**

它不是传统意义上的“业务大泥球 Service”。

它应该薄，但不是没有价值。

它负责：

```text
流程编排
事务边界
权限校验
幂等控制
调用领域模型
调用仓储
调用外部网关
发布事件
```

它不应该负责：

```text
领域规则本身
复杂价格计算
状态流转合法性
聚合内部一致性
实体字段随意 set
```

---

# 13. 一个更完整的下单编排示例

## Controller

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final CreateOrderApplicationService createOrderApplicationService;

    @PostMapping
    public CreateOrderResponse create(@RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = new CreateOrderCommand(
                request.userId(),
                request.items(),
                request.couponId()
        );

        CreateOrderResult result = createOrderApplicationService.create(command);

        return new CreateOrderResponse(result.orderId());
    }
}
```

Controller 只做请求转换。

---

## Application Service

```java
@Service
public class CreateOrderApplicationService {

    private final UserGateway userGateway;
    private final ProductGateway productGateway;
    private final CouponGateway couponGateway;
    private final OrderRepository orderRepository;
    private final PricingDomainService pricingDomainService;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public CreateOrderResult create(CreateOrderCommand command) {
        UserSnapshot user = userGateway.getUser(command.userId());
        List<ProductSnapshot> products = productGateway.getProducts(command.skuIds());

        CouponSnapshot coupon = couponGateway.getCoupon(command.couponId());

        Order order = Order.create(
                command.userId(),
                command.items(),
                products
        );

        Money finalAmount = pricingDomainService.calculate(order, coupon);

        order.confirmPrice(finalAmount);
        orderRepository.save(order);

        eventPublisher.publish(new OrderCreatedEvent(order.id()));

        return new CreateOrderResult(order.id());
    }
}
```

这里做的是编排：

```text
查用户
查商品
查优惠券
创建订单
计算价格
确认价格
保存订单
发布事件
```

---

## Domain Model

```java
public class Order {

    private OrderId id;
    private UserId userId;
    private List<OrderItem> items;
    private Money amount;
    private OrderStatus status;

    public static Order create(
            UserId userId,
            List<CreateOrderItemCommand> items,
            List<ProductSnapshot> products
    ) {
        if (items.isEmpty()) {
            throw new DomainException("订单明细不能为空");
        }

        return new Order(
                OrderId.newId(),
                userId,
                OrderItem.create(items, products),
                OrderStatus.CREATED
        );
    }

    public void confirmPrice(Money amount) {
        if (amount.isNegative()) {
            throw new DomainException("订单金额不能为负数");
        }

        this.amount = amount;
    }

    public void submit() {
        if (this.status != OrderStatus.CREATED) {
            throw new DomainException("只有已创建订单可以提交");
        }

        this.status = OrderStatus.SUBMITTED;
    }
}
```

领域对象只表达订单规则。

---

# 14. 编排层容易写坏的地方

## 14.1 把应用服务写成传统大 Service

错误示例：

```java
public void createOrder(CreateOrderCommand command) {
    if (command.getItems().size() == 0) {
        throw new RuntimeException("订单不能为空");
    }

    BigDecimal total = BigDecimal.ZERO;

    for (...) {
        total = total.add(...);
    }

    if (total.compareTo(BigDecimal.ZERO) < 0) {
        throw new RuntimeException("金额异常");
    }

    orderMapper.insert(...);
}
```

问题：

```text
领域规则泄漏到应用层
领域对象变贫血
应用服务膨胀
测试困难
```

更好的方式：

```text
应用层编排流程
领域层承载规则
基础设施层处理技术细节
```

---

## 14.2 过度抽象

另一个极端是小项目也搞很多层：

```text
Controller
Facade
ApplicationService
CommandHandler
DomainService
Repository
Adapter
Assembler
Factory
Specification
```

如果业务只是简单 CRUD，这会增加成本。

DDD 不是为了把项目拆复杂，而是为了在复杂业务里控制复杂度。

---

# 15. 领域编排和“搭积木”的关系

你引用的那段话里说：

> 不同场景功能诉求下，可以选择不同领域模块进行组装，像搭积木一样。

这个很准确。

比如电商里有这些领域模块：

```text
用户领域
商品领域
订单领域
库存领域
优惠领域
支付领域
物流领域
积分领域
通知领域
```

不同业务流程可以复用不同模块。

---

## 普通下单

```text
用户
商品
订单
库存
支付
```

---

## 秒杀下单

```text
用户
活动
商品
库存
订单
支付
风控
```

---

## 取消订单

```text
订单
库存
优惠券
支付
积分
通知
```

---

## 退款

```text
订单
支付
售后
库存
积分
通知
```

这些流程不是每个领域自己完成的，而是由应用层、流程引擎、Saga、事件机制组合起来。

这就是“领域编排”。

---

# 16. 简单项目里的务实建议

对于你现在理解 DDD，建议不要一上来就追求完整架构。

可以这样分阶段：

## 第一阶段：简单用例

```text
Controller → Domain → Repository
```

适合简单操作。

例如：

```java
@PostMapping("/articles/{id}/publish")
public void publish(@PathVariable Long id) {
    Article article = articleRepository.find(new ArticleId(id));
    article.publish();
    articleRepository.save(article);
}
```

---

## 第二阶段：出现流程编排

```text
Controller → Application Service → Domain → Repository / Gateway
```

例如文章发布后：

```text
发布文章
生成搜索索引
生成 DevWiki Chunk
通知订阅用户
刷新缓存
```

这时就应该引入应用服务。

---

## 第三阶段：跨领域最终一致性

```text
Application Service
  ↓
Domain Event
  ↓
Event Handler
  ↓
Other Domain / External System
```

例如：

```text
ArticlePublishedEvent
  ↓
SearchIndexHandler
  ↓
WikiChunkGenerateHandler
  ↓
NotificationHandler
```

这时要考虑：

```text
事件可靠投递
消费幂等
失败重试
补偿处理
状态机
对账任务
```

---

# 17. 最重要的边界

可以记住这四句话：

```text
Controller 负责接请求，不负责业务流程。

Application Service 负责编排用例，不负责领域规则。

Domain 负责业务规则，不负责技术细节。

Infrastructure 负责技术实现，不反向污染领域模型。
```

---

# 18. 最后总结

领域编排的核心不是“多加一层 Service”，而是：

> **把一个完整业务用例拆成多个领域能力，然后由应用层按照业务流程、事务边界和一致性要求进行组装。**

具体来说：

```text
领域层：提供积木
应用层：搭积木
基础设施层：提供工具和材料
触发器层：接收外部请求
```

是否需要应用层，取决于复杂度：

```text
简单项目：Controller 可以直接调用领域对象，减少层次。

复杂流程：必须引入 Application Service 做编排。

跨领域流程：优先考虑事件、Saga、补偿、最终一致性。
```

最精简的一句话：

> **领域编排 = 用应用服务把多个领域能力、仓储和适配器组织成一个完整业务流程，同时控制事务边界和一致性策略。**