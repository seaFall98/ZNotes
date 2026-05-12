[xfg原文](https://github.com/fuzhengwei/CodeGuide/edit/master/docs/md/road-map/mvc2ddd.md)

下面把 **“MVC 到 DDD 的重构”** 这个主题讲到一个真正能落地的版本。

你可以把它当成一套判断框架：

> **不是把包名从 controller/service/dao 改成 application/domain/infrastructure 就叫 DDD。**  
> 真正的 MVC → DDD 重构，是把“散落在 Service 里的业务规则”，迁移到“有边界、有语义、有行为的领域模型”里。

小傅哥这章的核心观点是：**MVC 老工程腐化严重，DDD 新工程全量重构又步子太大，所以更现实的方案是低成本、渐进式地把 MVC 工程改造成具备 DDD 思想的结构。** 这一点是很务实的。原文也强调，MVC 和 DDD 在这里主要是在“工程落地结构”语境下比较，不是完整的软件设计思想大对决。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/mvc2ddd.md "raw.githubusercontent.com"))

---

# 1. 先说结论：MVC 到 DDD 到底在重构什么？

一句话：

> **从“过程式 Service 编排数据库操作”，重构为“应用层编排用例，领域层表达业务，基础设施层负责技术细节”。**

MVC 老项目常见结构是：

```text
controller
  └── 接收请求、参数校验、调用 service

service
  └── 写业务流程、if else、查库、改库、发 MQ、调 RPC、写缓存

dao / mapper
  └── 操作数据库

domain / entity / po
  └── 贫血对象，只放字段 getter setter
```

问题在于：

```text
Service = 业务流程 + 业务规则 + 数据访问 + 事务 + 缓存 + MQ + RPC + 参数转换
```

时间一长，Service 就会变成：

```text
巨大函数
巨大类
到处 if else
到处复制粘贴
对象被乱传
PO 被传到 Controller
DTO 被传进 DAO
业务规则没人知道归谁管
```

DDD 重构要做的是：

```text
Controller / Trigger：负责接收外部请求
Application：负责用例编排
Domain：负责业务规则、领域行为、状态变更
Infrastructure：负责数据库、缓存、MQ、RPC、外部系统
```

也就是把原来的：

```text
Service 一层包打天下
```

拆成：

```text
外部输入
  ↓
应用用例
  ↓
领域模型
  ↓
仓储接口
  ↓
基础设施实现
```

---

# 2. MVC 为什么会腐化？

![[Pasted image 20260510215939.png]]

MVC 本身不是垃圾架构。

MVC 的优点是：

```text
简单
直接
上手快
适合 CRUD
适合早期交付
```

但 MVC 的问题是：**它没有足够强的业务边界约束。**

比如一个订单系统，早期可能这样写：

```java
public void payOrder(Long orderId, String paymentNo) {
    OrderPO order = orderMapper.selectById(orderId);

    if (order == null) {
        throw new RuntimeException("订单不存在");
    }

    if (!"WAIT_PAY".equals(order.getStatus())) {
        throw new RuntimeException("订单状态不允许支付");
    }

    order.setStatus("PAID");
    order.setPaymentNo(paymentNo);
    order.setPayTime(new Date());

    orderMapper.update(order);

    inventoryService.deduct(orderId);
    couponService.markUsed(order.getCouponId());
    mqProducer.send("ORDER_PAID", orderId);
}
```

这段代码看起来没问题。

但过几个月需求来了：

```text
虚拟商品不扣库存
积分订单不能用优惠券
风控拒绝的订单不能支付
部分订单支持组合支付
支付成功后要发积分
支付成功后要通知履约系统
支付超时订单不能支付
企业订单走另外一套审批流
```

于是 Service 变成：

```java
public void payOrder(...) {
    // 查订单
    // 查用户
    // 查商品
    // 查优惠券
    // 查风控
    // 查库存

    if (...) {
    }

    if (...) {
    }

    if (...) {
    }

    // 改状态
    // 扣库存
    // 改优惠券
    // 发积分
    // 发 MQ
    // 调履约
    // 写日志
}
```

这就是典型 MVC 腐化。

**不是因为 MVC 不能用，而是因为业务复杂后，Service 成了所有逻辑的垃圾桶。**

小傅哥原文也指出，MVC 工程腐化的根本在于对象、服务、组件被交叉混乱使用，Service 互相调用，贫血对象被到处复用，长期迭代后会越来越难维护。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/mvc2ddd.md "raw.githubusercontent.com"))

---

# 3. MVC 和 DDD 的核心差异

## 3.1 MVC 关注“技术分层”

MVC 常见思维是：

```text
Controller 是接口
Service 是业务
DAO 是数据库
Entity/PO 是数据对象
```

它的分层依据主要是技术职责：

```text
Web 层
业务层
数据访问层
```

所以包结构通常是：

```text
com.xxx.order
  controller
  service
  mapper
  entity
  dto
```

这对简单 CRUD 很有效。

---

## 3.2 DDD 关注“业务边界 + 领域语义”

![[fbaf45d6-d749-41ba-9188-efc38b3adb27 1.png]]
DDD 更关心：

```text
这个业务属于哪个上下文？
这个对象是不是聚合根？
这个行为应该由谁负责？
这个状态变化是否符合业务规则？
这个对象是否应该暴露到外部？
这个数据访问是否应该通过仓储隔离？
```

DDD 的结构不是简单地把 service 改名成 domainService，而是要重新划分职责：

```text
interfaces / trigger
application
domain
infrastructure
```

更重要的是，DDD 里的对象不是单纯的数据容器，而是带业务行为的模型。

比如 MVC 里可能这样：

```java
order.setStatus(OrderStatus.PAID);
```

DDD 里更倾向这样：

```java
order.pay(paymentNo);
```

前者是直接改字段。

后者是表达业务动作。

`pay()` 方法里面可以封装：

```text
只有 WAIT_PAY 状态可以支付
支付后状态变为 PAID
记录支付单号
记录支付时间
产生 OrderPaidEvent
```

这就是从贫血模型到充血模型的变化。

DDD分层调用链路

![[f95eee371a4331741143233ab4aaea72.png]]

---

# 4. MVC 到 DDD 不是“换目录”，而是“迁移责任”

这是重点。
![[fc727cdfc6abf900017342c055349d08.png]]

很多人重构 DDD 会变成这样：

```text
controller 改名 interfaces
service 改名 application
entity 改名 domain
mapper 改名 infrastructure
```

然后代码逻辑原封不动。

这没有意义。

小傅哥原文也用了一个很形象的说法：从 MVC 到 DDD 只是换了一个更大、更清晰的房子，但如果你只是把老房子里的破旧家具搬进去，代码不会自动变干净。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/mvc2ddd.md "raw.githubusercontent.com"))
![[Pasted image 20260510230152.png]]

真正要迁移的是：

|原 MVC 中的问题|DDD 中的归宿|
|---|---|
|Controller 里做业务判断|移到 Application 或 Domain|
|Service 里写一堆 if else 业务规则|移到 Domain Entity / Aggregate / Domain Service / Specification|
|Service 里直接操作 Mapper|改成调用 Repository 接口|
|Service 里直接发 MQ|改成领域事件，由 Infrastructure 发送|
|Service 里直接调外部 RPC|改成 Domain Gateway / Adapter 接口|
|PO 到处传|限制在 Infrastructure|
|DTO 到处传|限制在 Interface / Application 入参|
|Entity 只有字段|增加业务行为，变成领域对象|

---

# 5. 用一个订单支付案例讲透

假设我们有一个接口：

```text
POST /orders/{orderId}/pay
```

用户支付订单。

---

## 5.1 MVC 写法

典型 MVC 代码：

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @PostMapping("/{orderId}/pay")
    public Response<Void> pay(@PathVariable Long orderId,
                              @RequestBody PayOrderRequest request) {
        orderService.pay(orderId, request.getPaymentNo());
        return Response.success();
    }
}
```

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private CouponService couponService;

    @Autowired
    private MqProducer mqProducer;

    @Transactional
    public void pay(Long orderId, String paymentNo) {
        OrderPO order = orderMapper.selectById(orderId);

        if (order == null) {
            throw new BizException("订单不存在");
        }

        if (!"WAIT_PAY".equals(order.getStatus())) {
            throw new BizException("订单状态不允许支付");
        }

        order.setStatus("PAID");
        order.setPaymentNo(paymentNo);
        order.setPayTime(LocalDateTime.now());

        orderMapper.updateById(order);

        inventoryService.deduct(orderId);

        if (order.getCouponId() != null) {
            couponService.markUsed(order.getCouponId());
        }

        mqProducer.send("ORDER_PAID", orderId);
    }
}
```

短期看：简单。

长期看：危险。

因为这里混了很多东西：

```text
订单状态规则
订单支付行为
数据库读写
库存处理
优惠券处理
MQ 发送
事务控制
外部系统协作
```

---

## 5.2 DDD 重构后的结构

可以设计成：

```text
order-service
├── interfaces
│   └── controller
│       └── OrderController
│
├── application
│   ├── command
│   │   └── PayOrderCommand
│   └── service
│       └── PayOrderUseCase
│
├── domain
│   └── order
│       ├── model
│       │   ├── Order
│       │   ├── OrderItem
│       │   ├── OrderStatus
│       │   └── PaymentNo
│       ├── repository
│       │   └── OrderRepository
│       ├── event
│       │   └── OrderPaidEvent
│       └── service
│           └── OrderPaymentDomainService
│
├── infrastructure
│   ├── persistence
│   │   ├── OrderPO
│   │   ├── OrderMapper
│   │   └── OrderRepositoryImpl
│   ├── mq
│   │   └── OrderEventPublisherImpl
│   └── gateway
│       └── InventoryGatewayImpl
```


---

# 6. DDD 重构后的调用链路

支付订单的调用链路应该是：

```text
HTTP Request
  ↓
Controller
  ↓
PayOrderCommand
  ↓
PayOrderUseCase
  ↓
OrderRepository.findById()
  ↓
Order 聚合
  ↓
order.pay(paymentNo)
  ↓
OrderRepository.save(order)
  ↓
publish OrderPaidEvent
  ↓
MQ / Event Handler / 后续流程
```

重点是：

```text
Controller 不写业务规则
Application 不承载核心业务规则
Domain 不依赖数据库技术
Infrastructure 不反向污染 Domain
```

---

# 7. 分层逐个讲

## 7.1 Interfaces / Trigger：触发器层

小傅哥原文里把接口、消息监听、定时任务这些统一理解为“触发动作”，也就是 trigger / adapter 层。它可以是 HTTP 接口，也可以是 MQ 消费，也可以是 Job 定时任务。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/mvc2ddd.md "raw.githubusercontent.com"))

这一层负责：

```text
接收请求
参数基础校验
DTO 转 Command
调用 Application UseCase
返回 Response
```

示例：

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final PayOrderUseCase payOrderUseCase;

    public OrderController(PayOrderUseCase payOrderUseCase) {
        this.payOrderUseCase = payOrderUseCase;
    }

    @PostMapping("/{orderId}/pay")
    public Response<Void> pay(@PathVariable Long orderId,
                              @RequestBody PayOrderRequest request) {
        PayOrderCommand command = new PayOrderCommand(
                orderId,
                request.getPaymentNo()
        );

        payOrderUseCase.pay(command);

        return Response.success();
    }
}
```

注意：

```text
PayOrderRequest 是接口 DTO
PayOrderCommand 是应用层入参
二者不要混用
```

为什么？

因为 DTO 是外部接口协议，可能受前端、开放 API、RPC 协议影响。

Command 是内部用例入参，应该服务于业务用例表达。

---

## 7.2 Application：应用层 / 用例层

Application 层负责“编排”，但不负责“核心业务规则”。

它做这些事情：

```text
开启事务
加载聚合
调用领域行为
保存聚合
发布领域事件
协调多个领域服务
```

示例：

```java
@Service
public class PayOrderUseCase {

    private final OrderRepository orderRepository;
    private final DomainEventPublisher eventPublisher;

    public PayOrderUseCase(OrderRepository orderRepository,
                           DomainEventPublisher eventPublisher) {
        this.orderRepository = orderRepository;
        this.eventPublisher = eventPublisher;
    }

    @Transactional
    public void pay(PayOrderCommand command) {
        Order order = orderRepository.findById(command.orderId())
                .orElseThrow(() -> new BizException("订单不存在"));

        order.pay(command.paymentNo());

        orderRepository.save(order);

        eventPublisher.publishAll(order.domainEvents());
        order.clearDomainEvents();
    }
}
```

这一层可以有事务，因为事务通常是应用用例边界。

但是这里不要写：

```java
if (order.getStatus() != WAIT_PAY) {
    throw ...
}
```

这个规则应该归 Order 自己。

---

## 7.3 Domain：领域层

领域层是 DDD 的核心。

它负责：

```text
业务规则
业务行为
业务状态变化
聚合一致性
领域事件
领域接口定义
```

示例：

```java
public class Order {

    private final Long id;
    private OrderStatus status;
    private String paymentNo;
    private LocalDateTime paidAt;

    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public Order(Long id, OrderStatus status, String paymentNo, LocalDateTime paidAt) {
        this.id = id;
        this.status = status;
        this.paymentNo = paymentNo;
        this.paidAt = paidAt;
    }

    public void pay(String paymentNo) {
        if (this.status != OrderStatus.WAIT_PAY) {
            throw new BizException("只有待支付订单可以支付");
        }

        this.status = OrderStatus.PAID;
        this.paymentNo = paymentNo;
        this.paidAt = LocalDateTime.now();

        this.domainEvents.add(new OrderPaidEvent(this.id, paymentNo, this.paidAt));
    }

    public List<DomainEvent> domainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearDomainEvents() {
        this.domainEvents.clear();
    }
}
```

这才是领域模型。

它不是：

```java
@Data
public class Order {
    private Long id;
    private String status;
}
```

而是：

```java
public class Order {
    public void pay(...) {
        // 业务规则
        // 状态变更
        // 领域事件
    }
}
```

---

## 7.4 Repository：仓储接口

领域层定义仓储接口：

```java
public interface OrderRepository {

    Optional<Order> findById(Long orderId);

    void save(Order order);
}
```

注意：这是领域层的接口，不是 MyBatis Mapper。

它表达的是：

```text
我要根据订单 ID 获取一个 Order 聚合
我要保存一个 Order 聚合
```

而不是：

```text
select * from order where id = ?
update order set status = ?
```

Repository 是领域视角。

Mapper 是数据库视角。

这两个不能混。

---

## 7.5 Infrastructure：基础设施层

基础设施层负责技术实现：

```text
数据库
MyBatis
Redis
MQ
RPC
HTTP Client
配置中心
文件系统
第三方服务
```

示例：

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderMapper orderMapper;

    public OrderRepositoryImpl(OrderMapper orderMapper) {
        this.orderMapper = orderMapper;
    }

    @Override
    public Optional<Order> findById(Long orderId) {
        OrderPO po = orderMapper.selectById(orderId);
        if (po == null) {
            return Optional.empty();
        }
        return Optional.of(OrderConverter.toDomain(po));
    }

    @Override
    public void save(Order order) {
        OrderPO po = OrderConverter.toPO(order);
        orderMapper.updateById(po);
    }
}
```

这里有一个非常关键的转换：

```text
PO → Domain
Domain → PO
```

PO 不能泄漏到领域层。

Domain 也不应该直接绑定数据库表结构。

小傅哥原文也强调，基础设施层通过依赖倒置实现领域层定义的接口，可以隔离数据库 PO 等基础设施对象不向外暴露。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/mvc2ddd.md "raw.githubusercontent.com"))

---

# 8. 对象转换关系：DTO、Command、Domain、PO

MVC 腐化的一个典型表现是对象乱用：

```text
Controller 直接收 PO
Service 直接传 DTO
DAO 返回 VO
Domain 对象直接暴露给前端
```

DDD 要把对象边界立起来。

正确关系：

```text
HTTP Request DTO
  ↓
Command
  ↓
Domain Aggregate
  ↓
PO
  ↓
Database Table
```

分别解释：

|对象|所属层|作用|
|---|---|---|
|Request DTO|Interfaces|接口入参，适配 HTTP/RPC 协议|
|Command|Application|用例入参，表达一次业务意图|
|Domain Aggregate|Domain|承载业务规则和状态变化|
|PO|Infrastructure|数据库持久化对象|
|Table|Database|数据库表结构|

边界规则：

```text
DTO 不进入 Domain
PO 不离开 Infrastructure
Domain 不等于数据库表
Application 不直接操作 Mapper
Controller 不直接操作 Repository
```

这个规则非常重要。

---

# 9. MVC 到 DDD 的映射关系

小傅哥原文提出一种低成本迁移方式：不一定要马上改变原有工程模块依赖，而是在原 MVC 结构中逐步把 Service 里的领域逻辑拆到 Domain，把 DAO 扩展为基础设施实现，把 Service 收缩为 Application/Case 编排层。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/mvc2ddd.md "raw.githubusercontent.com"))

可以这样映射：

|MVC 旧结构|DDD 新职责|
|---|---|
|controller/web|trigger / interfaces|
|service|application / case|
|domain/entity|domain model|
|dao/mapper|infrastructure persistence|
|rpc/client|infrastructure gateway / adapter|
|mq/producer|infrastructure event publisher|
|common/constant|types / shared kernel|

但是注意：

**映射不是改名。**

真正要做的是：

```text
Service 变薄
Domain 变厚
Infrastructure 变专
对象边界变清晰
```

---

# 10. 重构前后对比

## 10.1 重构前

```text
OrderService.pay()
  ├── 查订单
  ├── 判断状态
  ├── 改状态
  ├── 扣库存
  ├── 改优惠券
  ├── 发 MQ
  └── 调履约
```

问题：

```text
所有逻辑集中在一个 Service
业务规则和技术细节混在一起
不好测试
不好复用
不好扩展
```

---

## 10.2 重构后

```text
PayOrderUseCase.pay()
  ├── orderRepository.findById()
  ├── order.pay()
  ├── orderRepository.save()
  └── eventPublisher.publish(OrderPaidEvent)
```

领域规则进入：

```text
Order.pay()
```

技术细节进入：

```text
OrderRepositoryImpl
OrderEventPublisherImpl
InventoryGatewayImpl
```

最终效果：

```text
Application 看流程
Domain 看规则
Infrastructure 看技术实现
Interfaces 看协议适配
```

这就是 DDD 的可读性价值。

---

# 11. 什么逻辑应该放 Domain？什么逻辑不应该放？

这是 MVC → DDD 重构最容易混乱的地方。

## 11.1 应该放 Domain 的逻辑

只要它是业务规则，就优先考虑放 Domain。

例如：

```text
只有待支付订单可以支付
已取消订单不能发货
订单金额不能小于 0
优惠券必须在有效期内使用
账户余额不足不能扣款
库存不能扣成负数
一个订单最多只能申请一次退款
```

这些规则不应该散落在 Service 里。

应该进入：

```text
Entity
Aggregate Root
Value Object
Domain Service
Specification
Policy
```

---

## 11.2 不应该放 Domain 的逻辑

这些不要放 Domain：

```text
HTTP 参数解析
JSON 序列化
数据库 SQL
Redis key 拼接
MQ topic 选择
RPC client 调用细节
Spring 注解强依赖
事务传播机制
分页对象
前端展示字段
```

Domain 要尽量保持业务纯净。

---

# 12. Entity、Aggregate、Domain Service 怎么选？

这是重构时的关键判断。

## 12.1 能放实体对象，就不要放 Domain Service

比如：

```text
订单支付
订单取消
订单发货
```

这些明显属于订单自身行为。

应该放：

```java
order.pay()
order.cancel()
order.ship()
```

而不是：

```java
orderDomainService.pay(order)
```

---

## 12.2 跨多个对象的业务规则，可以放 Domain Service

比如：

```text
订单支付时，需要同时校验订单、账户、支付渠道、风控结果
```

如果这个逻辑不自然属于某一个实体，就可以放领域服务：

```java
public class OrderPaymentDomainService {

    public void pay(Order order, Payment payment, RiskResult riskResult) {
        if (!riskResult.passed()) {
            throw new BizException("风控拒绝支付");
        }

        order.pay(payment.paymentNo());
    }
}
```

领域服务不是原来 MVC 的 Service。

它只处理领域规则，不处理数据库、MQ、Controller。

---

## 12.3 用 Application 编排跨系统流程

比如：

```text
查订单
查支付结果
调用领域服务
保存订单
发布事件
```

这属于用例流程，放 Application：

```java
@Transactional
public void pay(PayOrderCommand command) {
    Order order = orderRepository.findById(command.orderId())
            .orElseThrow(...);

    PaymentResult result = paymentGateway.query(command.paymentNo());

    orderPaymentDomainService.pay(order, result);

    orderRepository.save(order);

    eventPublisher.publish(new OrderPaidEvent(order.id()));
}
```

---

# 13. Repository 和 DAO 的区别

这个非常重要。

## 13.1 DAO / Mapper 是数据库视角

```java
OrderPO selectById(Long id);

int updateById(OrderPO po);
```

它关心：

```text
表
字段
SQL
分页
索引
数据库事务
```

---

## 13.2 Repository 是领域视角

```java
Optional<Order> findById(OrderId orderId);

void save(Order order);
```

它关心：

```text
聚合
领域对象
业务状态
对象完整性
```

Repository 可以底层调用 Mapper，但 Repository 本身不是 Mapper。

```text
Application / Domain
  ↓
OrderRepository 接口
  ↓
OrderRepositoryImpl
  ↓
OrderMapper
  ↓
Database
```

---

# 14. MQ 应该放哪里？

常见错误：

```java
order.pay();
mqProducer.send("ORDER_PAID", orderId);
```

直接在 Domain 里调用 MQ 是不好的。

因为 Domain 不应该知道 Kafka、RocketMQ、RabbitMQ 这些技术细节。

更好的做法：

```java
order.pay(paymentNo);
```

Order 内部产生事件：

```java
new OrderPaidEvent(orderId, paymentNo, paidAt)
```

Application 发布事件：

```java
eventPublisher.publishAll(order.domainEvents());
```

Infrastructure 实现发送 MQ：

```java
@Component
public class RocketMqDomainEventPublisher implements DomainEventPublisher {

    @Override
    public void publish(DomainEvent event) {
        // 转 MQ message
        // 发送 RocketMQ
    }
}
```

这样：

```text
Domain 只知道“发生了订单已支付事件”
Infrastructure 才知道“这个事件要发到 RocketMQ 的哪个 topic”
```

---

# 15. Redis 应该放哪里？

Redis 是基础设施。

不要在 Domain 里写：

```java
redisTemplate.opsForValue().set(...)
```

正确方式是抽象接口：

```java
public interface OrderLockService {
    boolean lock(OrderId orderId);
    void unlock(OrderId orderId);
}
```

Infrastructure 实现：

```java
@Component
public class RedisOrderLockService implements OrderLockService {
    // Redis 实现
}
```

但是要注意：不是所有 Redis 操作都应该抽象成领域接口。

如果只是查询缓存优化，例如：

```text
先查 Redis，没有再查 DB
```

通常可以封装在 RepositoryImpl 里面。

如果 Redis 表达的是业务概念，例如：

```text
订单支付防重锁
用户活动资格占用
库存预占
```

可以抽象成领域语义接口。

---

# 16. 外部 RPC / HTTP 应该放哪里？

比如订单支付要调用支付中心。

不要在 Domain 里直接写：

```java
paymentClient.queryPayment(...)
```

可以定义 Gateway：

```java
public interface PaymentGateway {
    PaymentResult queryPayment(String paymentNo);
}
```

Infrastructure 实现：

```java
@Component
public class PaymentGatewayImpl implements PaymentGateway {

    private final PaymentRpcClient paymentRpcClient;

    @Override
    public PaymentResult queryPayment(String paymentNo) {
        PaymentDTO dto = paymentRpcClient.query(paymentNo);
        return PaymentResultConverter.toDomain(dto);
    }
}
```

这样 Domain/Application 依赖的是：

```text
PaymentGateway
```

不是具体 RPC Client。

这就是依赖倒置。

---

# 17. 事务边界怎么处理？

一般建议：

```text
事务放 Application 层
```

因为 Application 层代表一次用例。

例如：

```java
@Transactional
public void pay(PayOrderCommand command) {
    Order order = orderRepository.findById(command.orderId()).orElseThrow(...);
    order.pay(command.paymentNo());
    orderRepository.save(order);
}
```

为什么不放 Domain？

因为 Domain 不应该依赖 Spring 的事务技术。

为什么不放 Infrastructure？

因为事务边界通常不只是单个 Mapper 操作，而是一次完整业务用例。

---

# 18. 跨聚合、跨领域怎么处理？

DDD 里有一个常见原则：

```text
聚合内强一致
聚合间最终一致
```

例如订单支付后：

```text
订单状态变为 PAID
优惠券标记已使用
库存扣减
积分增加
履约开始
```

不要试图在一个大事务里把所有东西都改完。

更好的做法：

```text
Order 聚合完成支付
发布 OrderPaidEvent
库存上下文消费事件扣库存
积分上下文消费事件加积分
履约上下文消费事件启动履约
```

也就是：

```text
订单内：事务一致
订单外：事件驱动最终一致
```

这就是 DDD 和微服务很契合的地方。

但注意：

如果你还是单体应用，也可以先在 Application 层同步调用其他领域服务。

不用一开始就上 MQ。

---

# 19. MVC 到 DDD 的渐进式重构路线

不要一上来全量推倒重来。

推荐路线：

## 第一步：识别腐化最严重的 Service

优先找这些类：

```text
超过 1000 行
大量 if else
调用很多 Mapper
调用很多外部服务
改动频率高
Bug 频率高
业务规则复杂
```

不要从简单 CRUD 开始。

简单 CRUD 用 MVC 就够了。

---

## 第二步：按用例切开 Service

比如原来：

```java
OrderService
  createOrder()
  payOrder()
  cancelOrder()
  refundOrder()
  shipOrder()
```

可以先拆成应用用例：

```text
CreateOrderUseCase
PayOrderUseCase
CancelOrderUseCase
RefundOrderUseCase
ShipOrderUseCase
```

这一步还不是完整 DDD，但已经可以显著降低 Service 巨类问题。

---

## 第三步：抽出领域对象行为

把这种代码：

```java
if (!order.getStatus().equals("WAIT_PAY")) {
    throw ...
}
order.setStatus("PAID");
```

迁移成：

```java
order.pay(paymentNo);
```

这是最关键的一步。

DDD 的味道从这里开始出现。

---

## 第四步：隔离 PO 和 Domain

原来可能全系统都用：

```java
OrderPO
```

现在逐渐改成：

```text
OrderPO：只在 infrastructure.persistence
Order：只在 domain
```

通过 Converter 转换：

```java
OrderConverter.toDomain(po);
OrderConverter.toPO(order);
```

这一步工作量不小，但收益很大。

因为它能阻止数据库结构污染业务模型。

---

## 第五步：抽 Repository 接口

在 domain 定义：

```java
public interface OrderRepository {
    Optional<Order> findById(Long orderId);
    void save(Order order);
}
```

在 infrastructure 实现：

```java
public class OrderRepositoryImpl implements OrderRepository {
    private final OrderMapper orderMapper;
}
```

这样 Application 就不再直接依赖 Mapper。

---

## 第六步：抽 Gateway / Adapter

把外部系统调用封装起来：

```text
PaymentGateway
InventoryGateway
CouponGateway
UserGateway
RiskGateway
```

这样业务层不再直接依赖各种 FeignClient / DubboClient。

---

## 第七步：引入领域事件

先从重要状态变化开始：

```text
OrderCreatedEvent
OrderPaidEvent
OrderCancelledEvent
RefundApprovedEvent
```

不要为了事件而事件。

只有当这个业务动作会影响其他模块时，领域事件才有价值。

---

# 20. 一个实用的重构判断公式

看到一段 Service 代码时，可以这样判断它该去哪里：

## 20.1 是接口协议相关？

比如：

```text
HTTP header
RequestBody
Response
分页参数
前端字段
```

放：

```text
interfaces / trigger
```

---

## 20.2 是用例流程编排？

比如：

```text
加载订单
调用支付
保存订单
发布事件
```

放：

```text
application
```

---

## 20.3 是业务规则？

比如：

```text
只有待支付订单可以支付
逾期账户不能提额
库存不能为负
优惠券过期不能使用
```

放：

```text
domain
```

---

## 20.4 是数据库、缓存、MQ、RPC？

比如：

```text
SQL
Redis
Kafka
RocketMQ
Feign
Dubbo
OSS
Elasticsearch
```

放：

```text
infrastructure
```

---

# 21. DDD 重构中的设计模式

小傅哥原文后半部分重点讲了：DDD 只是工程骨架，真正让代码变好的还需要设计原则和设计模式，比如工厂、策略、模板、责任链等。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/mvc2ddd.md "raw.githubusercontent.com"))

这点非常关键。

因为 DDD 不是魔法。

如果你在 Domain 里继续写 2000 行 if else，那只是把垃圾从 Service 搬到了 Domain。

---

## 21.1 策略模式：处理可变规则

比如优惠计算：

```text
满减优惠
折扣优惠
新人优惠
会员优惠
积分抵扣
```

不要写：

```java
if (couponType == FULL_REDUCTION) {
} else if (couponType == DISCOUNT) {
} else if (couponType == NEW_USER) {
}
```

可以写：

```java
public interface DiscountStrategy {
    Money calculate(Order order, Coupon coupon);
}
```

不同策略：

```text
FullReductionDiscountStrategy
PercentageDiscountStrategy
NewUserDiscountStrategy
MemberDiscountStrategy
```

---

## 21.2 工厂模式：选择策略

```java
@Component
public class DiscountStrategyFactory {

    private final Map<CouponType, DiscountStrategy> strategyMap;

    public DiscountStrategy get(CouponType couponType) {
        DiscountStrategy strategy = strategyMap.get(couponType);
        if (strategy == null) {
            throw new BizException("不支持的优惠类型");
        }
        return strategy;
    }
}
```

---

## 21.3 模板方法：固定主流程

比如信用额度调整流程：

```text
参数校验
查询申请单
加锁
查询账户
规则校验
执行调额
解锁
```

主流程稳定，部分步骤变化，就可以用模板方法。

这也是小傅哥原文里提到的典型场景：用抽象类定义流程结构，再用策略和工厂处理频繁变化的规则。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/mvc2ddd.md "raw.githubusercontent.com"))

---

## 21.4 责任链：处理一串校验

比如订单支付校验：

```text
订单存在校验
订单状态校验
风控校验
支付金额校验
用户状态校验
```

可以用责任链：

```java
public interface PayOrderCheckHandler {
    void check(PayOrderContext context);
}
```

然后组装：

```text
OrderExistsCheckHandler
OrderStatusCheckHandler
RiskCheckHandler
PaymentAmountCheckHandler
UserStatusCheckHandler
```

适合规则多、顺序明确、经常插拔的场景。

---

# 22. 一个完整的 MVC → DDD 重构示例

## 22.1 重构前：典型贫血 Service

```java
@Service
public class OrderService {

    @Transactional
    public void cancelOrder(Long orderId, String reason) {
        OrderPO order = orderMapper.selectById(orderId);

        if (order == null) {
            throw new BizException("订单不存在");
        }

        if ("PAID".equals(order.getStatus())) {
            throw new BizException("已支付订单不能直接取消");
        }

        if ("SHIPPED".equals(order.getStatus())) {
            throw new BizException("已发货订单不能取消");
        }

        order.setStatus("CANCELLED");
        order.setCancelReason(reason);
        order.setCancelTime(LocalDateTime.now());

        orderMapper.updateById(order);

        couponMapper.release(order.getCouponId());

        mqProducer.send("ORDER_CANCELLED", orderId);
    }
}
```

---

## 22.2 重构后：领域对象承载规则

```java
public class Order {

    private Long id;
    private OrderStatus status;
    private String cancelReason;
    private LocalDateTime cancelledAt;
    private List<DomainEvent> domainEvents = new ArrayList<>();

    public void cancel(String reason) {
        if (this.status == OrderStatus.PAID) {
            throw new BizException("已支付订单不能直接取消");
        }

        if (this.status == OrderStatus.SHIPPED) {
            throw new BizException("已发货订单不能取消");
        }

        if (this.status == OrderStatus.CANCELLED) {
            return;
        }

        this.status = OrderStatus.CANCELLED;
        this.cancelReason = reason;
        this.cancelledAt = LocalDateTime.now();

        this.domainEvents.add(new OrderCancelledEvent(this.id));
    }
}
```

---

## 22.3 应用层只编排

```java
@Service
public class CancelOrderUseCase {

    private final OrderRepository orderRepository;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public void cancel(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId())
                .orElseThrow(() -> new BizException("订单不存在"));

        order.cancel(command.reason());

        orderRepository.save(order);

        eventPublisher.publishAll(order.domainEvents());
        order.clearDomainEvents();
    }
}
```

---

## 22.4 基础设施层负责存储

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderMapper orderMapper;

    @Override
    public Optional<Order> findById(Long orderId) {
        OrderPO po = orderMapper.selectById(orderId);
        return Optional.ofNullable(po).map(OrderConverter::toDomain);
    }

    @Override
    public void save(Order order) {
        OrderPO po = OrderConverter.toPO(order);
        orderMapper.updateById(po);
    }
}
```

---

# 23. 重构后的收益是什么？

## 23.1 业务语义更清晰

以前看：

```java
order.setStatus("PAID");
```

你不知道这是不是合法。

现在看：

```java
order.pay(paymentNo);
```

你知道这是一个业务动作。

---

## 23.2 规则更集中

以前：

```text
支付规则散落在 OrderService、PaymentService、Job、MQ Consumer
```

现在：

```text
支付规则集中在 Order.pay() 或 OrderPaymentDomainService
```

---

## 23.3 测试更容易

领域对象可以直接单测：

```java
@Test
void should_pay_order_when_status_is_wait_pay() {
    Order order = new Order(1L, OrderStatus.WAIT_PAY);

    order.pay("PAY123");

    assertEquals(OrderStatus.PAID, order.status());
}
```

不需要启动 Spring。

不需要连数据库。

不需要 mock 一堆 Mapper。

---

## 23.4 技术细节更容易替换

比如从 MyBatis 换 JPA，理论上只影响：

```text
infrastructure.persistence
```

从 RocketMQ 换 Kafka，只影响：

```text
infrastructure.mq
```

Domain 不应该受影响。

---

# 24. 常见误区

## 误区 1：DDD 就是包多

错误。

DDD 的价值不是包多，而是：

```text
边界清楚
职责清楚
业务规则集中
领域模型有行为
技术细节被隔离
```

如果业务很简单，包多只会增加负担。

---

## 误区 2：所有项目都要 DDD

错误。

适合 DDD 的场景：

```text
业务规则复杂
生命周期长
多人协作
需求频繁变化
有多个业务上下文
核心业务价值高
```

不适合重度 DDD 的场景：

```text
简单 CRUD
后台管理表单
临时活动页面
数据搬运脚本
低复杂度工具系统
```

简单 CRUD 可以用 MVC。

复杂核心域再上 DDD。

---

## 误区 3：Domain 里不能有 Service

错误。

Domain Service 是合理存在的。

但它不是 MVC 里的 Service。

Domain Service 应该处理：

```text
不自然属于某个实体的领域规则
多个领域对象协作
复杂业务策略
```

不应该处理：

```text
查库
发 MQ
调 Redis
拼 SQL
组装 Response
```

---

## 误区 4：聚合根越大越好

错误。

聚合不是把相关对象全塞进去。

聚合的核心是：

```text
一致性边界
```

如果一个订单和订单项需要一起保证状态一致，可以在一个聚合内。

但订单、库存、支付、优惠券通常不应该塞进一个巨型聚合。

---

## 误区 5：DDD 一定要微服务

错误。

DDD 可以用于单体。

甚至我建议初学者先在单体里练 DDD。

因为你先把：

```text
业务边界
领域模型
应用用例
基础设施隔离
```

做好，再考虑是否拆微服务。

不要反过来。

---

# 25. 判断一段重构是否成功

你可以用下面这几个问题检查。

## 25.1 Service 是否变薄？

如果 Application 里还是 1000 行 if else，失败。

---

## 25.2 Domain 是否有行为？

如果 Domain 还是只有 getter/setter，失败。

---

## 25.3 PO 是否被隔离？

如果 Controller 返回的是 PO，失败。

如果 Domain 里依赖 OrderPO，失败。

---

## 25.4 业务规则是否集中？

如果“订单是否能支付”的规则散落在多个类，失败。

---

## 25.5 基础设施是否被抽象？

如果 Application 直接依赖 Mapper、RedisTemplate、RocketMqTemplate，说明重构还不彻底。

---

# 26. 推荐的 Java 包结构

一个比较实用的结构：

```text
com.example.order
├── OrderApplication.java
│
├── interfaces
│   ├── controller
│   │   └── OrderController.java
│   ├── request
│   │   └── PayOrderRequest.java
│   └── response
│       └── OrderResponse.java
│
├── application
│   ├── command
│   │   └── PayOrderCommand.java
│   ├── service
│   │   └── PayOrderUseCase.java
│   └── assembler
│       └── OrderAssembler.java
│
├── domain
│   └── order
│       ├── model
│       │   ├── Order.java
│       │   ├── OrderItem.java
│       │   ├── OrderStatus.java
│       │   └── Money.java
│       ├── repository
│       │   └── OrderRepository.java
│       ├── service
│       │   └── OrderPaymentDomainService.java
│       └── event
│           └── OrderPaidEvent.java
│
├── infrastructure
│   ├── persistence
│   │   ├── po
│   │   │   └── OrderPO.java
│   │   ├── mapper
│   │   │   └── OrderMapper.java
│   │   ├── converter
│   │   │   └── OrderConverter.java
│   │   └── repository
│   │       └── OrderRepositoryImpl.java
│   ├── mq
│   │   └── RocketMqDomainEventPublisher.java
│   └── gateway
│       └── PaymentGatewayImpl.java
│
└── common
    ├── exception
    └── response
```

---

# 27. 最核心的一张脑图

你可以把 MVC → DDD 重构记成这个：

```text
Controller
  只负责协议适配
  DTO → Command

Application
  只负责用例编排
  加载聚合 → 调用领域行为 → 保存聚合 → 发布事件

Domain
  负责业务本身
  Entity / ValueObject / Aggregate / DomainService / Event / Repository接口

Infrastructure
  负责技术实现
  DB / Redis / MQ / RPC / HTTP / Mapper / PO
```

更简化：

```text
Controller：外部怎么调用我
Application：这一单业务怎么完成
Domain：业务规则到底是什么
Infrastructure：技术上怎么实现
```

---

# 28. 学这章应该真正掌握什么？

这章学完，不是为了会背：

```text
DDD 四层架构
聚合根
领域服务
仓储
防腐层
领域事件
```

而是你看到一段 MVC 烂代码时，能知道怎么动刀。

你应该形成这样的能力：

```text
1. 看出 Service 里哪些是用例编排，哪些是领域规则，哪些是技术细节。
2. 把业务规则迁移到领域对象或领域服务。
3. 把数据库访问隔离到 Repository。
4. 把 PO、DTO、Domain 的边界切干净。
5. 把 MQ、Redis、RPC 放到 Infrastructure。
6. 用策略、模板、工厂、责任链治理复杂业务流程。
7. 渐进式重构，而不是一上来推倒重写。
```

---

# 29. 最终总结

**MVC 到 DDD 的重构，本质不是架构升级，而是业务责任归位。**

MVC 老项目的问题通常是：

```text
Service 太重
对象太贫血
边界太模糊
技术细节和业务规则混杂
长期迭代后腐化严重
```

DDD 重构要做的是：

```text
让 Controller 回到协议适配
让 Application 回到用例编排
让 Domain 承载业务规则
让 Infrastructure 承载技术细节
让对象转换隔离层间污染
让领域事件表达关键业务变化
```

最重要的一句话：

> **DDD 不是让代码看起来高级，而是让复杂业务在长期演进中不失控。**

对 Java 后端开发者来说，MVC → DDD 的真正分水岭不是你会不会建 `domain` 包，而是你能不能把这句代码：

```java
order.setStatus(OrderStatus.PAID);
```

重构成：

```java
order.pay(paymentNo);
```

并且让 `pay()` 背后承载清晰、完整、可测试、可演进的业务规则。