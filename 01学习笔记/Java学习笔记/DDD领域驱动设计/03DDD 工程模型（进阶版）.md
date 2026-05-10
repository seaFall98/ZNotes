[xfg基础版](https://github.com/fuzhengwei/CodeGuide/blob/master/docs/md/road-map/ddd-guide-03.md)

下面按**Java 后端开发者真正落地 DDD 工程模型**来讲。

你可以把这篇当成：

> **DDD 从“建模方法”进入“工程落地”的分水岭。**

前面学的是：

```text
业务怎么拆？
领域怎么识别？
聚合怎么设计？
实体和值对象怎么区分？
```

现在要学的是：

```text
这些东西在 Java 工程里到底放哪？
Controller 怎么调？
Application Service 干什么？
Domain Service 干什么？
Repository 接口和实现放哪？
DTO / PO / Entity / VO 怎么隔离？
为什么 MVC 项目越大越乱？
DDD 工程结构到底解决什么问题？
```

---

# 1. 先评价小傅哥这篇文章的核心价值

小傅哥这篇《DDD 工程模型》的核心观点是对的：

> **MVC 在简单系统里够用，但在复杂业务、微服务、外部依赖、消息、任务、缓存、RPC、领域规则变多之后，Service 层会迅速膨胀，变成“万能垃圾桶”。**

工程开发核心科目
![[Pasted image 20260510193038.png]]
他的文章里明确提到，工程结构的作用是定义：哪个层访问数据库、哪个层使用缓存、哪个层调用外部接口、哪个层做功能实现；而传统 `Service + 贫血模型` 三层结构没有细分面向对象工程结构，所以很多内容会堆到 Service 实现类里。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd-guide-03.md "raw.githubusercontent.com"))

MVC分层架构
![[Pasted image 20260510193104.png]]

这就是 DDD 工程模型要解决的问题。

不过，小傅哥的文章偏“工程脚手架展示”，对初学者有两个难点：

1. **概念很多，但缺少一条完整业务请求链路**
    
2. **模块很多，但不容易判断每个类到底该放哪**
    

所以我这次用一个更清晰的结构来讲：

```text
为什么 MVC 会乱
DDD 工程模型的核心思想
标准分层结构
每层职责
对象流转
订单案例
包结构设计
调用链路
常见误区
怎么从 MVC 迁移到 DDD
```

---

# 2. DDD 工程模型到底是什么？

一句话：

> **DDD 工程模型，就是把“领域模型”安放到一套稳定的代码结构里，让业务逻辑有明确归属，让技术细节被隔离。**

工程模型
![[ChatGPT Image May 10, 2026, 07_54_47 PM 1.png]]
领域模型
![[Pasted image 20260510200542.png]]


它不是单纯换包名。

不是把：

```text
controller
service
mapper
entity
```

改成：

```text
trigger
case
domain
infrastructure
```

就叫 DDD。

真正的 DDD 工程模型要解决四件事：

|问题|MVC 常见做法|DDD 工程模型做法|
|---|---|---|
|业务规则放哪|Service 里堆逻辑|放到领域对象 / 领域服务|
|数据库细节放哪|Service 直接调 Mapper|Infrastructure 实现 Repository|
|外部系统调用放哪|Service 直接调 Feign / SDK|Gateway / Adapter 隔离|
|接口、MQ、任务怎么组织|Controller、Listener、Job 各写一套|统一作为 Trigger 触发应用用例|

所以 DDD 工程模型的本质是：

```text
外部触发
  ↓
应用编排
  ↓
领域模型执行业务规则
  ↓
基础设施完成技术落地
```

---

# 3. 为什么 MVC 项目会越来越乱？

MVC上帝类 VS DDD分层职责
![[ChatGPT Image May 10, 2026, 08_12_50 PM.png]]



先看一个典型 MVC 电商下单 Service：

```java
@Service
public class OrderService {

    @Autowired
    private ProductMapper productMapper;
    @Autowired
    private InventoryMapper inventoryMapper;
    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private CouponService couponService;
    @Autowired
    private PaymentClient paymentClient;
    @Autowired
    private RedisTemplate redisTemplate;
    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    @Transactional
    public Long createOrder(CreateOrderRequest request) {
        // 1. 查商品
        // 2. 校验商品状态
        // 3. 查库存
        // 4. 校验库存
        // 5. 查优惠券
        // 6. 计算价格
        // 7. 创建订单
        // 8. 扣库存
        // 9. 写数据库
        // 10. 删缓存
        // 11. 发消息
        // 12. 调支付
        return orderId;
    }
}
```

这种 Service 看起来很正常，但规模一大就会出问题：

```text
Service = 业务规则 + 流程编排 + 数据库访问 + 缓存访问 + 外部接口 + 消息发送 + 事务控制
```

这就是所谓的“Service 上帝类”。

问题不是 MVC 不能用，而是它只有粗粒度三层：

```text
Controller
Service
DAO / Mapper
```

对于复杂业务，它没有告诉你：

```text
价格计算规则放哪？
订单状态流转放哪？
优惠券校验放哪？
库存扣减调用放哪？
领域事件放哪？
DTO 和 Entity 怎么转换？
外部支付接口怎么隔离？
```

于是所有东西都自然掉进 Service。

小傅哥文章也指出，随着微服务演进，越来越多内容填充到工程中，MVC 结构会变得混乱，一个 Service 为了实现功能会引入一堆东西，原子功能与 Service 自身耦合在一起，维护成本越来越大。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd-guide-03.md "raw.githubusercontent.com"))

---

# 4. DDD 工程模型的核心原则

DDD 工程模型可以记住 5 条铁律。

## 4.1 Domain 是核心，不依赖外部技术

领域层不应该关心：

```text
HTTP
RPC
MyBatis
Redis
Kafka
RocketMQ
Spring MVC
数据库表结构
第三方 SDK
```

领域层只关心：

```text
订单能不能创建
订单能不能支付
订单金额怎么算
订单状态怎么流转
库存是否足够
优惠是否有效
```

也就是说：

```text
Domain 不认识数据库
Domain 不认识 HTTP
Domain 不认识 MQ
Domain 只认识业务
```

---

## 4.2 外部依赖向内适配，而不是领域向外妥协

DDD 工程模型常和六边形架构、洋葱架构、整洁架构一起出现。

洋葱架构 
![[Pasted image 20260510193929.png]]

整洁架构
![[Pasted image 20260510194846.png]]

DDD六边形
![[Pasted image 20260510195052.png]]

小傅哥文章里也提到，六边形、洋葱、菱形架构目标都是以领域服务为核心，隔离内部实现与外部资源耦合；基础设施层用于承接外部资源调用，触发器层向外部提供服务。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd-guide-03.md "raw.githubusercontent.com"))

核心方向是：

```text
外部技术依赖领域
不是领域依赖外部技术
```

例如：

```java
// domain 层定义接口
public interface OrderRepository {
    Order findById(OrderId orderId);
    void save(Order order);
}
```

然后：

```java
// infrastructure 层实现接口
@Repository
public class OrderRepositoryImpl implements OrderRepository {
    private final OrderMapper orderMapper;

    @Override
    public Order findById(OrderId orderId) {
        OrderPO po = orderMapper.selectById(orderId.value());
        return OrderConverter.toDomain(po);
    }
}
```

这样领域层不知道 MyBatis 的存在。

---

## 4.3 应用层负责编排，不负责核心业务判断

Application Service / Case / UseCase 层应该做：

```text
开启事务
参数转换
调用仓储加载聚合
调用领域模型执行业务行为
保存聚合
发布领域事件
调用外部服务
返回结果
```

但它不应该写大量业务判断。

错误写法：

```java
if (order.getStatus() != OrderStatus.CREATED) {
    throw new BizException("订单不能支付");
}
order.setStatus(OrderStatus.PAID);
```

更好的写法：

```java
order.pay(paymentId);
```

因为“订单什么状态可以支付”是订单自己的业务规则，不应该散落在应用服务里。

---

## 4.4 Infrastructure 负责技术实现，不负责业务决策

Infrastructure 可以做：

```text
MyBatis 查询
Redis 缓存
Feign 调用
MQ 发送
ES 查询
文件上传
第三方 SDK 调用
PO / DTO 转换
```

但不要在 Infrastructure 里写：

```text
订单是否能支付
优惠券是否可用
用户是否满足授信条件
风控规则是否通过
```

Infrastructure 是技术层，不是业务层。

---

## 4.5 Trigger 只是入口，不是业务处理器

Trigger 可以是：

```text
HTTP Controller
RPC Provider
MQ Consumer
定时任务 Job
命令行任务
事件监听器
```

小傅哥文章中把接口调用、消息监听、任务调度等统一放到 trigger，基础设施层则负责数据库、缓存、配置、外部接口调用等。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd-guide-03.md "raw.githubusercontent.com"))

Trigger 只负责：

```text
接收请求
做基础参数校验
调用应用层
返回结果
```

不要在 Controller 里写复杂业务逻辑。

---

# 5. 一套推荐的 DDD Java 工程结构

我更推荐你先学习这一套结构：

```text
xxx-project
├── xxx-api
├── xxx-app
├── xxx-trigger
├── xxx-application
├── xxx-domain
├── xxx-infrastructure
└── xxx-types
```

对应职责如下。

|模块|职责|类似小傅哥文章中的角色|
|---|---|---|
|`api`|对外暴露接口契约、DTO、Response|api 层|
|`app`|Spring Boot 启动、配置装配|app 层|
|`trigger`|HTTP、MQ、RPC、Job 入口|trigger 层|
|`application` / `case`|用例编排、事务控制|case 编排层|
|`domain`|领域模型、聚合、实体、值对象、领域服务、仓储接口|domain 层|
|`infrastructure`|数据库、缓存、外部接口、仓储实现|infrastructure 层|
|`types`|通用异常、枚举、工具、响应码|types 层|

小傅哥文章里的示例工程也大致包含 `api`、`app`、`domain`、`infrastructure`、`trigger`、`types` 等模块，并且由 `app` 模块的 Spring Boot 应用启动。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd-guide-03.md "raw.githubusercontent.com"))

---

# 6. 最核心的是这 4 层

先不要被模块数量吓到。你真正要理解的是这 4 层：

```text
Trigger        触发层
Application    应用层 / 用例层 / 编排层
Domain         领域层
Infrastructure 基础设施层
```

可以画成这样：

```text
┌──────────────────────────────┐
│ Trigger                      │
│ Controller / MQ / Job / RPC  │
└───────────────┬──────────────┘
                │
                ▼
┌──────────────────────────────┐
│ Application                  │
│ 用例编排 / 事务 / 调用领域模型 │
└───────────────┬──────────────┘
                │
                ▼
┌──────────────────────────────┐
│ Domain                       │
│ 聚合 / 实体 / 值对象 / 领域服务 │
└───────────────▲──────────────┘
                │ 接口定义
                │
┌───────────────┴──────────────┐
│ Infrastructure               │
│ DB / MQ / Redis / RPC / SDK  │
└──────────────────────────────┘
```

注意依赖方向：

```text
Trigger → Application → Domain
Infrastructure → Domain
```

不是：

```text
Domain → Infrastructure
```

这点非常关键。

---

# 7. 用电商订单系统理解工程模型

继续用你之前熟悉的电商订单系统。

电商订单的DDD工程模型
![[ChatGPT Image May 10, 2026, 08_14_47 PM.png]]

业务流程：

```text
用户下单
  ↓
创建订单
  ↓
锁定库存
  ↓
计算优惠
  ↓
生成待支付订单
  ↓
用户支付
  ↓
订单状态变为 PAID
  ↓
发布 OrderPaidEvent
```

在 DDD 工程模型中，拆成这样。

---

## 7.1 Trigger 层：接收 HTTP 请求

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase;
    private final PayOrderUseCase payOrderUseCase;

    @PostMapping
    public Response<CreateOrderResponseDTO> createOrder(
            @RequestBody CreateOrderRequestDTO requestDTO) {
        return Response.success(createOrderUseCase.execute(requestDTO));
    }

    @PostMapping("/{orderId}/pay")
    public Response<Void> payOrder(
            @PathVariable Long orderId,
            @RequestBody PayOrderRequestDTO requestDTO) {
        payOrderUseCase.execute(orderId, requestDTO);
        return Response.success();
    }
}
```

Controller 不做业务判断。

不要写：

```java
if (requestDTO.getItems().isEmpty()) {
    ...
}
if (coupon expired) {
    ...
}
if (stock not enough) {
    ...
}
```

它只做：

```text
接收请求
调用用例
返回响应
```

---

## 7.2 Application 层：编排用例

```java
@Service
public class PayOrderUseCase {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public void execute(Long orderId, PayOrderRequestDTO requestDTO) {
        Order order = orderRepository.findById(new OrderId(orderId));

        PaymentResult paymentResult = paymentGateway.queryPaymentResult(
                requestDTO.getPaymentId()
        );

        order.pay(new PaymentId(requestDTO.getPaymentId()), paymentResult);

        orderRepository.save(order);

        eventPublisher.publish(new OrderPaidEvent(order.getId()));
    }
}
```

Application 层关心的是：

```text
这个用例要串哪些步骤？
事务边界在哪里？
需要加载哪个聚合？
需要调用哪个外部能力？
最后保存什么？
发布什么事件？
```

但它不应该关心：

```text
订单支付状态怎么流转
什么订单可以支付
支付失败怎么改变订单内部状态
订单金额是否合法
```

这些属于 Domain。

---

## 7.3 Domain 层：表达业务规则

```java
public class Order {

    private OrderId id;
    private UserId userId;
    private List<OrderItem> items;
    private Money totalAmount;
    private OrderStatus status;
    private Address deliveryAddress;

    public void pay(PaymentId paymentId, PaymentResult paymentResult) {
        if (this.status != OrderStatus.CREATED) {
            throw new DomainException("当前订单状态不能支付");
        }

        if (!paymentResult.isSuccess()) {
            throw new DomainException("支付未成功，订单不能变更为已支付");
        }

        this.status = OrderStatus.PAID;
    }

    public void cancel() {
        if (this.status == OrderStatus.PAID) {
            throw new DomainException("已支付订单不能直接取消");
        }

        this.status = OrderStatus.CANCELED;
    }
}
```

这才是 DDD 的核心。

不是写很多包。

不是画很多图。

而是让代码里出现真正的业务语言：

```java
order.pay(paymentId, paymentResult);
order.cancel();
order.confirmReceipt();
order.applyCoupon(coupon);
order.changeAddress(address);
```

而不是：

```java
order.setStatus(OrderStatus.PAID);
orderMapper.update(order);
```

---

## 7.4 Infrastructure 层：实现数据库、外部接口、MQ

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderMapper orderMapper;
    private final OrderItemMapper orderItemMapper;

    @Override
    public Order findById(OrderId orderId) {
        OrderPO orderPO = orderMapper.selectById(orderId.value());
        List<OrderItemPO> itemPOS = orderItemMapper.selectByOrderId(orderId.value());

        return OrderConverter.toDomain(orderPO, itemPOS);
    }

    @Override
    public void save(Order order) {
        OrderPO orderPO = OrderConverter.toPO(order);
        orderMapper.updateById(orderPO);
    }
}
```

Infrastructure 可以认识：

```text
OrderPO
OrderMapper
MyBatis
Redis
RocketMQ
FeignClient
第三方支付 SDK
```

Domain 不认识这些。

---

# 8. 各种对象到底放哪？

这是 DDD 工程模型里最容易绕晕的部分。

DDD工程模型中的对象转换关系
![[ChatGPT Image May 10, 2026, 08_17_04 PM.png]]

## 8.1 DTO

DTO 是数据传输对象。

放在：

```text
api
trigger
application
infrastructure
```

都可能出现，但含义不同。

常见场景：

```text
接口入参：CreateOrderRequestDTO
接口出参：CreateOrderResponseDTO
外部支付接口入参：PaymentRequestDTO
外部支付接口出参：PaymentResponseDTO
```

DTO 不应该有复杂业务逻辑。

---

## 8.2 PO

PO 是持久化对象。

放在：

```text
infrastructure
```

例如：

```java
@TableName("t_order")
public class OrderPO {
    private Long id;
    private Long userId;
    private BigDecimal totalAmount;
    private String status;
}
```

PO 是数据库表结构的映射。

不要把 PO 传进 Domain。

---

## 8.3 Entity

Entity 是领域实体。

放在：

```text
domain
```

例如：

```java
public class OrderItem {
    private OrderItemId id;
    private ProductId productId;
    private Money price;
    private Integer quantity;
}
```

Entity 有身份标识，有生命周期，可能会变化。

---

## 8.4 Value Object

Value Object 是值对象。

放在：

```text
domain
```

例如：

```java
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new DomainException("金额不能小于 0");
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new DomainException("币种不同，不能相加");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

值对象没有独立身份，通过属性值判断相等。

小傅哥文章也把领域模型中的对象分成 `aggregate`、`entity`、`valobj`，并强调值对象通过属性值识别，实体对象具有业务身份，聚合对象用于组织实体和值对象。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd-guide-03.md "raw.githubusercontent.com"))

---

## 8.5 Aggregate

Aggregate 是聚合。

放在：

```text
domain
```

例如：

```java
public class Order {
    private OrderId id;
    private List<OrderItem> items;
    private Money totalAmount;
    private OrderStatus status;
}
```

聚合负责保证内部一致性。

例如订单聚合可以保证：

```text
订单明细不能为空
订单总价必须等于明细价格之和
订单只能从 CREATED 变成 PAID
已取消订单不能支付
```

---

## 8.6 Repository

Repository 接口放在：

```text
domain
```

Repository 实现放在：

```text
infrastructure
```

例如：

```text
domain
└── order
    └── repository
        └── OrderRepository.java

infrastructure
└── repository
    └── OrderRepositoryImpl.java
```

为什么接口放 domain？

因为领域层需要表达：

```text
我需要保存订单
我需要查找订单
```

但不关心：

```text
用 MyBatis 查
用 JPA 查
用 Redis 查
用 RPC 查
```

---

# 9. 推荐包结构：按业务领域分包，而不是按技术类型分包

小傅哥文章里提到两种分包方式：

1. 按 DDD 科目类型分包
    
2. 按业务领域分包
    

并举例说，如果一个工程下有用户、积分、抽奖、支付，可以按业务包拆，也可以把所有 model、repository、service 放在统一目录下。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd-guide-03.md "raw.githubusercontent.com"))
![[Pasted image 20260510195801.png]]

我更推荐你优先使用：

> **按业务领域分包。**

也就是：

```text
domain
├── order
│   ├── model
│   │   ├── aggregate
│   │   ├── entity
│   │   └── valueobject
│   ├── repository
│   └── service
├── product
│   ├── model
│   ├── repository
│   └── service
└── payment
    ├── model
    ├── repository
    └── service
```

而不是：

```text
domain
├── aggregate
│   ├── Order.java
│   ├── Product.java
│   └── Payment.java
├── entity
│   ├── OrderItem.java
│   └── ProductSku.java
├── repository
│   ├── OrderRepository.java
│   └── ProductRepository.java
└── service
    ├── OrderService.java
    └── PaymentService.java
```

原因很简单：

```text
DDD 首先按业务边界组织代码，而不是按技术类型组织代码。
```

当你维护“订单领域”时，你希望订单相关的模型、仓储、服务都在一起。

---

# 10. 一条完整调用链路

以“支付订单”为例。

```text
HTTP 请求
  ↓
OrderController.pay()
  ↓
PayOrderUseCase.execute()
  ↓
OrderRepository.findById()
  ↓
OrderRepositoryImpl 查询数据库，PO → Domain
  ↓
PaymentGateway 查询支付状态
  ↓
Order.pay()
  ↓
OrderRepository.save()
  ↓
DomainEventPublisher.publish(OrderPaidEvent)
  ↓
返回 Response
```

对应代码归属：

|步骤|类|所属层|
|---|---|---|
|接收 HTTP|`OrderController`|trigger|
|编排支付用例|`PayOrderUseCase`|application|
|加载订单|`OrderRepository`|domain 接口|
|查询数据库|`OrderRepositoryImpl`|infrastructure|
|支付业务行为|`Order.pay()`|domain|
|发布事件|`DomainEventPublisher`|domain 接口 / infrastructure 实现|
|返回响应|`Response`|api / types|

---

# 11. DDD 工程模型和六边形架构的关系

六边形架构的核心是：

```text
核心业务在内部
外部系统通过端口和适配器连接进来
```

对应到 DDD：

```text
Domain = 核心业务
Repository / Gateway = Port
Infrastructure Impl = Adapter
Controller / MQ / Job = Driving Adapter
DB / RPC / MQ / Redis = Driven Adapter
```

可以这样理解：

```text
用户、MQ、定时任务
        ↓
   入口适配器
        ↓
   应用用例
        ↓
   领域模型
        ↑
   出口端口
        ↑
数据库、缓存、外部系统适配器
```

小傅哥文章里说六边形架构会把对外提供的能力放到 trigger，比如接口调用、消息监听、任务调度；把调用外部同类能力放到 infrastructure，比如数据库、缓存、配置、调用其他接口。这个划分就是典型的端口适配器思想。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd-guide-03.md "raw.githubusercontent.com"))

---

# 12. DDD 工程模型里的“依赖倒置”

这是最关键的技术点。

普通 MVC：

```text
Service → Mapper → DB
```

DDD：

```text
Application → Domain Repository 接口
Infrastructure RepositoryImpl → Mapper → DB
```

也就是：

```text
领域层定义需要什么
基础设施层负责满足它
```

示例：

```java
// domain
public interface InventoryGateway {
    boolean hasEnoughStock(ProductId productId, int quantity);
    void lockStock(ProductId productId, int quantity);
}
```

```java
// infrastructure
@Component
public class InventoryGatewayImpl implements InventoryGateway {

    private final InventoryFeignClient inventoryFeignClient;

    @Override
    public boolean hasEnoughStock(ProductId productId, int quantity) {
        return inventoryFeignClient.checkStock(productId.value(), quantity).isEnough();
    }
}
```

这样 Domain 不依赖 Feign。

它只依赖一个业务语义接口：

```java
inventoryGateway.hasEnoughStock(productId, quantity)
```

---

# 13. Application Service 和 Domain Service 的区别

这个非常重要。

## 13.1 Application Service

负责用例编排。

例如：

```java
public class CreateOrderUseCase {

    public CreateOrderResponse execute(CreateOrderCommand command) {
        // 查用户
        // 查商品
        // 查优惠
        // 调用领域服务创建订单
        // 保存订单
        // 返回结果
    }
}
```

特点：

```text
偏流程
偏编排
可以有事务
可以调多个领域对象
可以调外部网关
不承载核心业务规则
```

---

## 13.2 Domain Service

负责领域逻辑。

什么时候需要 Domain Service？

当某个业务规则：

```text
不自然属于某一个实体或值对象
但确实是领域规则
```

例如：

```java
public class OrderPricingService {

    public Money calculateTotal(
            List<OrderItem> items,
            Coupon coupon,
            MemberLevel memberLevel) {
        // 商品金额
        // 优惠券折扣
        // 会员折扣
        // 满减规则
        // 运费规则
        return total;
    }
}
```

价格计算可能涉及：

```text
订单明细
优惠券
会员等级
营销规则
配送规则
```

它不完全属于 `Order`，也不完全属于 `Coupon`，所以可以放到领域服务。

---

# 14. DDD 工程模型里的事务边界

常见规则：

> **事务通常放在 Application 层。**

例如：

```java
@Transactional
public void payOrder(PayOrderCommand command) {
    Order order = orderRepository.findById(command.orderId());
    order.pay(command.paymentId());
    orderRepository.save(order);
}
```

为什么不放 Domain？

因为 Domain 不应该依赖 Spring 的 `@Transactional`。

Domain 应该是纯业务对象。

---

# 15. DDD 工程模型里的事件

领域事件一般在 Domain 产生，在 Application 发布。

例如：

```java
public class Order {

    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public void pay(PaymentId paymentId) {
        if (status != OrderStatus.CREATED) {
            throw new DomainException("订单不能支付");
        }

        this.status = OrderStatus.PAID;
        this.domainEvents.add(new OrderPaidEvent(this.id));
    }

    public List<DomainEvent> pullEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }
}
```

Application：

```java
@Transactional
public void execute(PayOrderCommand command) {
    Order order = orderRepository.findById(command.orderId());

    order.pay(command.paymentId());

    orderRepository.save(order);

    order.pullEvents().forEach(eventPublisher::publish);
}
```

注意：

```text
领域事件是业务事实
MQ 消息是技术实现
```

`OrderPaidEvent` 是领域事件。

发 RocketMQ / Kafka 是基础设施实现。

不要把二者混为一谈。

---

# 16. 一个完整推荐目录

对于 Java 后端项目，可以这样设计：

```text
mall-order
├── mall-order-api
│   └── com.example.order.api
│       ├── OrderApi.java
│       ├── dto
│       │   ├── CreateOrderRequestDTO.java
│       │   ├── CreateOrderResponseDTO.java
│       │   └── PayOrderRequestDTO.java
│       └── response
│           └── Response.java
│
├── mall-order-app
│   └── com.example.order
│       ├── OrderApplication.java
│       └── config
│
├── mall-order-trigger
│   └── com.example.order.trigger
│       ├── http
│       │   └── OrderController.java
│       ├── mq
│       │   └── PaymentResultConsumer.java
│       └── job
│           └── OrderTimeoutCancelJob.java
│
├── mall-order-application
│   └── com.example.order.application
│       ├── command
│       │   ├── CreateOrderCommand.java
│       │   └── PayOrderCommand.java
│       ├── usecase
│       │   ├── CreateOrderUseCase.java
│       │   └── PayOrderUseCase.java
│       └── assembler
│           └── OrderAssembler.java
│
├── mall-order-domain
│   └── com.example.order.domain
│       ├── order
│       │   ├── model
│       │   │   ├── aggregate
│       │   │   │   └── Order.java
│       │   │   ├── entity
│       │   │   │   └── OrderItem.java
│       │   │   └── valueobject
│       │   │       ├── OrderId.java
│       │   │       ├── Money.java
│       │   │       ├── Address.java
│       │   │       └── OrderStatus.java
│       │   ├── repository
│       │   │   └── OrderRepository.java
│       │   ├── service
│       │   │   └── OrderPricingService.java
│       │   └── event
│       │       └── OrderPaidEvent.java
│       └── shared
│           └── DomainException.java
│
├── mall-order-infrastructure
│   └── com.example.order.infrastructure
│       ├── persistence
│       │   ├── mapper
│       │   │   ├── OrderMapper.java
│       │   │   └── OrderItemMapper.java
│       │   ├── po
│       │   │   ├── OrderPO.java
│       │   │   └── OrderItemPO.java
│       │   └── repository
│       │       └── OrderRepositoryImpl.java
│       ├── gateway
│       │   ├── InventoryGatewayImpl.java
│       │   └── PaymentGatewayImpl.java
│       ├── mq
│       │   └── RocketMqDomainEventPublisher.java
│       └── converter
│           └── OrderConverter.java
│
└── mall-order-types
    └── com.example.order.types
        ├── Constants.java
        ├── ErrorCode.java
        └── BizException.java
```

初学时可以不用这么复杂，但这套结构是比较清晰的目标形态。

---

# 17. 简化版：适合个人项目 / 中小项目

你现在学习和做项目，不一定一开始就拆这么多 Maven Module。

可以先用单模块多包结构：

```text
src/main/java/com/example/order
├── OrderApplication.java
├── trigger
│   ├── http
│   ├── mq
│   └── job
├── application
│   ├── command
│   ├── usecase
│   └── assembler
├── domain
│   ├── order
│   │   ├── model
│   │   ├── repository
│   │   ├── service
│   │   └── event
│   └── shared
├── infrastructure
│   ├── persistence
│   ├── gateway
│   ├── mq
│   └── converter
└── types
```

这对你的阶段更适合。

因为你现在最需要练的是：

```text
职责分离
领域模型
调用链路
对象转换
业务规则归属
```

不一定急着搞多模块 Maven。

---

# 18. 判断一个类该放哪的规则

你以后写代码时，可以用这张表判断。

|代码内容|应该放哪|
|---|---|
|Controller|trigger/http|
|MQ Consumer|trigger/mq|
|定时任务|trigger/job|
|一个完整业务用例|application/usecase|
|事务控制|application|
|参数 DTO 转 Command|application/assembler|
|订单支付规则|domain/order/model/aggregate|
|金额计算|domain/order/service 或 valueobject|
|订单状态枚举|domain/order/model/valueobject|
|Repository 接口|domain/order/repository|
|Repository 实现|infrastructure/persistence/repository|
|Mapper|infrastructure/persistence/mapper|
|PO|infrastructure/persistence/po|
|Feign Client|infrastructure/gateway/client|
|第三方 SDK 调用|infrastructure/gateway|
|MQ 发送实现|infrastructure/mq|
|统一异常码|types|
|Response 包装|api 或 types|

---

# 19. DDD 工程模型的核心收益

## 19.1 业务逻辑更集中

以前：

```text
Controller 有一点
Service 有一点
Mapper XML 有一点
外部接口回调有一点
定时任务又有一点
```

现在：

```text
订单规则主要在 Order 聚合
价格规则主要在 PricingService
库存规则通过 InventoryGateway 隔离
```

---

## 19.2 技术细节可替换

比如你原来用 MyBatis，后来想换 JPA。

只要领域层依赖的是：

```java
OrderRepository
```

而不是：

```java
OrderMapper
```

那么替换成本会小很多。

---

## 19.3 测试更容易

领域对象可以脱离 Spring 测试。

```java
@Test
void paidOrderCannotBePaidAgain() {
    Order order = OrderTestFactory.createdOrder();

    order.pay(new PaymentId("p1"), PaymentResult.success());

    assertThrows(DomainException.class, () -> {
        order.pay(new PaymentId("p2"), PaymentResult.success());
    });
}
```

这种测试不需要数据库，不需要 Spring Boot，不需要 MockMVC。

这是 DDD 很重要的收益。

---

## 19.4 大项目更容易维护

因为你知道：

```text
接口入口去哪找
业务用例去哪找
核心规则去哪找
数据库实现去哪找
外部接口去哪找
```

而不是所有东西都在：

```text
XxxServiceImpl
```

---

# 20. 常见误区

## 20.1 误区一：DDD = 多建几个包

错误理解：

```text
我有 domain、infrastructure、trigger，所以我用了 DDD。
```

不对。

如果你的核心逻辑还是这样：

```java
order.setStatus(OrderStatus.PAID);
orderMapper.update(order);
```

而不是：

```java
order.pay(paymentId);
orderRepository.save(order);
```

那只是 MVC 换皮。

---

## 20.2 误区二：Entity 和 PO 混用

错误：

```java
@TableName("t_order")
public class Order {
    private Long id;
    private BigDecimal amount;
    private String status;
}
```

然后这个 `Order` 既当数据库对象，又当领域对象。

这会导致：

```text
数据库字段污染领域模型
领域行为不敢写
对象变成 getter/setter 容器
```

推荐分开：

```text
Order        领域聚合
OrderPO      数据库持久化对象
OrderDTO     接口传输对象
```

---

## 20.3 误区三：Application 层写成新版 ServiceImpl

有些项目改成 DDD 后：

```java
CreateOrderUseCase
PayOrderUseCase
CancelOrderUseCase
```

里面还是堆满业务判断。

这只是把 `OrderServiceImpl` 改名了。

Application 应该薄一点，Domain 应该厚一点。

---

## 20.4 误区四：聚合设计过大

比如：

```text
Order 聚合
包含 User
包含 Product
包含 Inventory
包含 Coupon
包含 Payment
包含 Logistics
```

这就太大了。

订单聚合应该只维护订单内部一致性。

其他上下文通过：

```text
ID
领域事件
Gateway
应用层编排
```

协作。

---

## 20.5 误区五：所有逻辑都塞进聚合

聚合不是万能容器。

如果一个规则涉及多个对象，而且不自然属于某一个对象，可以放到领域服务。

比如：

```text
价格计算
授信评估
风控决策
抽奖规则树
优惠策略选择
```

这些可以是 Domain Service 或 Strategy。

小傅哥文章里也提醒，不要以为定义了聚合对象，就把超过一个对象的逻辑都封装到聚合中；聚合更适合与本对象相关的单一封装场景，复杂核心业务可以放到 service 里。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd-guide-03.md "raw.githubusercontent.com"))

---

# 21. 从 MVC 迁移到 DDD 的 practical 路线

你真正做项目时，不要一口吃成胖子。

推荐分 5 步。

## 第一步：先识别业务用例

把原来的 Service 方法整理成用例：

```text
createOrder
payOrder
cancelOrder
confirmReceipt
applyRefund
```

每个用例对应一个 Application Service。

---

## 第二步：把贫血 Entity 改成领域对象

从：

```java
order.setStatus(OrderStatus.PAID);
```

改成：

```java
order.pay(paymentId);
```

这是最重要的一步。

---

## 第三步：抽 Repository 接口

把：

```java
orderMapper.selectById()
orderMapper.update()
```

从业务代码里移出去。

改成：

```java
Order order = orderRepository.findById(orderId);
orderRepository.save(order);
```

---

## 第四步：隔离外部系统

把：

```java
paymentFeignClient.query(...)
inventoryFeignClient.lock(...)
```

改成：

```java
paymentGateway.query(...)
inventoryGateway.lock(...)
```

Application 或 Domain 只看到业务语义。

---

## 第五步：补领域事件

把跨聚合、跨上下文的后续动作改成事件驱动：

```text
OrderPaidEvent
  ↓
库存确认
积分增加
优惠券核销
通知用户
发货流程启动
```

---

# 22. 对你当前学习阶段的建议

你是 Java 后端 + AI 开发方向，我建议你不要只背 DDD 概念，而是直接用代码练。

可以按这个顺序练：

```text
1. 用 MVC 写一个简单订单系统
2. 故意把逻辑写到 OrderServiceImpl
3. 重构出 Application 层
4. 重构出 Order 聚合
5. 把 Mapper 隔离到 RepositoryImpl
6. 加 PaymentGateway / InventoryGateway
7. 加 OrderPaidEvent
8. 写领域对象单元测试
```

你会很快理解：

```text
DDD 不是为了显得架构高级
而是为了让复杂业务有地方安放
```

---

# 23. 最终给你一个“10分心智模型”

你可以这样记 DDD 工程模型：

```text
Trigger：谁来触发业务？
Application：这个业务用例怎么编排？
Domain：业务规则本身是什么？
Infrastructure：技术细节怎么落地？
```

再浓缩一点：

```text
Controller 不做业务
Application 不写核心规则
Domain 不碰技术细节
Infrastructure 不做业务决策
Repository 接口在 Domain
Repository 实现在 Infrastructure
DTO 不进 Domain
PO 不出 Infrastructure
聚合保护一致性
事件表达业务事实
```

这是 DDD 工程模型的核心。  
掌握这套之后，你再看六边形架构、洋葱架构、整洁架构、COLA，其实都不会陌生。

它们本质上都在回答同一个问题：

> **如何让业务核心稳定，让外部技术变化可控。**