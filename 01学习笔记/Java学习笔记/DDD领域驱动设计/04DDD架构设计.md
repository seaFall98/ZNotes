[xfg基础版](https://github.com/fuzhengwei/CodeGuide/blob/master/docs/md/road-map/ddd.md)

我们把这章《DDD 架构设计》重新讲成一堂 **Java 后端开发者能真正落地的 DDD 工程课**。

这章不要按“DDD 理论课”理解，而要按下面这句话理解：

> **DDD 架构设计，不是把 MVC 的 Controller / Service / DAO 换个包名，而是重新规定：业务规则、用例编排、技术实现、外部入口分别应该住在哪里。**

小傅哥这章的核心结构是：先指出 MVC 在复杂长期项目中的问题，再提出“触发 -> 函数 -> 连接”的开发抽象，最后落到 `api / app / domain / infrastructure / trigger / types / case` 这些工程模块上。文章中明确把 HTTP、RPC、MQ、Task 都归为“触发”，并强调 `infrastructure` 依赖 `domain` 中定义的仓储接口，这是一种依赖倒置设计。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

---

# 1. 这章到底在讲什么？

一句话：

```text
这章讲的是：DDD 项目代码应该怎么摆。
```

不是讲：

```text
什么是实体？
什么是值对象？
什么是聚合？
什么是仓储？
```

这些前面你已经学过了。

这章真正要解决的是：

```text
Controller 放哪？
RPC 接口放哪？
MQ Consumer 放哪？
定时任务放哪？
领域对象放哪？
Repository 接口放哪？
Repository 实现放哪？
Mapper / DAO / PO 放哪？
AOP、配置、启动类放哪？
Response、ErrorCode、Constants 放哪？
```

所以它的学习目标不是“理解 DDD 名词”，而是建立一个完整的工程空间感。

你要从这章学会：

```text
外部请求进入系统后，代码应该怎么流动；
业务规则应该停留在哪一层；
数据库细节应该被隔离在哪一层；
每个模块的边界是什么；
怎么避免写出披着 DDD 外衣的 MVC。
```

---

# 2. 为什么 MVC 后期会痛？

先看传统 MVC。

很多 Java 项目最后会变成这样：

```text
Controller
  -> Service
      -> DAO / Mapper
      -> Redis
      -> MQ
      -> 外部 HTTP
      -> 各种 if else
      -> 各种状态变更
      -> 各种 DTO / VO / PO 混用
```

一开始很爽：

```text
Controller 接请求
Service 写业务
Mapper 查数据库
```

但是业务一复杂，Service 层会膨胀成“万能业务垃圾场”。

典型症状是：

```text
1. Service 方法越来越长
2. 一个 Service 调十几个 DAO
3. DTO、VO、PO、Entity 混着传
4. 业务规则散落在 if else 里
5. 数据库字段变化牵动业务代码
6. 一个需求改动影响一大片
7. 新人读代码只能从数据库表反推业务
```

小傅哥文章里也指出，长期维护的大型 MVC 项目中，DAO、PO、VO 等对象会在 Service 层相互调用，状态和行为分离，最终让代码意图模糊、膨胀、臃肿、不稳定。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

这句话非常关键：

```text
MVC 最大的问题，不是分层少。
MVC 最大的问题，是业务表达能力弱。
```

Service 里写的是过程：

```java
if (order.getStatus() != WAIT_PAY) {
    throw new BizException("订单状态不允许支付");
}
order.setStatus(PAID);
orderMapper.update(order);
```

但 DDD 想让代码表达业务：

```java
order.pay(paymentId);
orderRepository.save(order);
```

看出来区别了吗？

MVC Service 经常表达的是：

```text
怎么改数据
```

DDD 领域模型表达的是：

```text
业务发生了什么
```

---

# 3. 小傅哥的“触发 -> 函数 -> 连接”怎么理解？

文章里把 DDD 代码开发抽象成：

```text
触发 -> 函数 -> 连接
```

这个说法非常适合 Java 后端理解。文章中提到，DDD 常用于微服务场景，一个系统的调用方式不只是 HTTP，还可能包括 RPC、MQ、TASK，这些都可以理解为“触发”。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

展开就是：

```text
触发：外部世界怎么进入系统
函数：系统内部提供什么业务能力
连接：系统如何访问数据库、缓存、MQ、外部服务
```

对应代码：

```text
触发
  Controller
  RPC Provider
  MQ Consumer
  Scheduled Task

函数
  Application Service / UseCase
  Domain Service
  Aggregate 行为

连接
  Repository
  DAO / Mapper
  Redis Client
  MQ Producer
  HTTP Client
```

举个支付订单例子。

## MVC 写法

```text
PayOrderController
  -> OrderService.pay()
      -> orderMapper.selectById()
      -> paymentClient.check()
      -> orderMapper.updateStatus()
      -> mqTemplate.send()
```

所有东西挤在 `OrderService.pay()` 里。

## DDD 写法

```text
PayOrderController / PayOrderRpc / PayOrderConsumer
  -> PayOrderUseCase
      -> OrderRepository.findById()
      -> PaymentGateway.check()
      -> Order.pay()
      -> OrderRepository.save()
      -> DomainEventPublisher.publish(OrderPaidEvent)
```

区别在于：

```text
Controller / RPC / MQ 只是不同触发方式；
PayOrderUseCase 是用例编排；
Order.pay() 承载核心业务规则；
Repository / Gateway 隔离技术细节。
```

也就是说：

```text
同一个支付订单能力，可以被 HTTP 触发，也可以被 RPC 触发，也可以被 MQ 触发。
但业务能力本身不应该写死在 Controller 里。
```

这就是“触发”和“函数”分离。

---

# 4. DDD 工程分层总览

小傅哥这章给出的模块大致是：

```text
xfg-frame-api
xfg-frame-app
xfg-frame-domain
xfg-frame-infrastructure
xfg-frame-trigger
xfg-frame-types
xfg-frame-case    可选
```

文章中对这些模块有明确解释：`api` 放 RPC 接口定义，`app` 放启动和配置，`domain` 放领域模型服务，`infrastructure` 放仓储实现，`trigger` 放接口实现、消息接收、任务执行，`types` 放通用类型，`case/application` 用于复杂项目中的领域编排。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

我们重新整理成一张更好理解的表。

|层 / 模块|中文理解|放什么|不放什么|
|---|---|---|---|
|`api`|对外契约层|RPC 接口、请求响应模型|业务逻辑|
|`trigger`|触发器 / 适配器层|Controller、RPC 实现、MQ Consumer、Task|核心业务规则|
|`case` / `application`|用例编排层|一个完整业务用例的流程编排|具体技术实现|
|`domain`|领域层|聚合、实体、值对象、领域服务、仓储接口|MyBatis、Redis、HTTP Client|
|`infrastructure`|基础设施层|Repository 实现、DAO、Mapper、PO、外部服务适配|业务决策规则|
|`app`|应用启动装配层|SpringBoot 启动、配置、AOP、线程池|领域规则|
|`types`|通用类型层|Response、ErrorCode、Constants|大量业务公共模型|

---

# 5. 最重要的是依赖方向

DDD 工程设计最重要的不是目录名，而是依赖方向。

健康的方向应该是：

```text
trigger
  -> application / case
      -> domain
          -> repository interface

infrastructure
  -> domain
```

注意这个反直觉的地方：

```text
不是 domain 依赖 infrastructure。
而是 infrastructure 依赖 domain。
```

文章里也强调，`infrastructure` 基础层依赖 `domain` 领域层，因为 `domain` 层定义了仓储接口，需要在基础层实现，这是依赖倒置。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

也就是说：

```java
// domain 层
public interface IOrderRepository {
    OrderAggregate findById(OrderId orderId);
    void save(OrderAggregate order);
}
```

然后：

```java
// infrastructure 层
@Repository
public class OrderRepository implements IOrderRepository {

    private final OrderMapper orderMapper;

    @Override
    public OrderAggregate findById(OrderId orderId) {
        OrderPO po = orderMapper.selectById(orderId.value());
        return converter.toAggregate(po);
    }

    @Override
    public void save(OrderAggregate order) {
        OrderPO po = converter.toPO(order);
        orderMapper.update(po);
    }
}
```

这样设计的目的是什么？

```text
domain 层只知道“我要保存订单”；
domain 层不知道“订单存在 MySQL 还是 MongoDB”；
domain 层不知道“用 MyBatis 还是 JPA”；
domain 层不知道“表字段叫什么”。
```

这就是 DDD 里非常核心的隔离。

---

# 6. 一次请求在 DDD 里怎么流动？

这是你最应该掌握的主线。

以“支付订单”为例。

完整链路：

```text
HTTP Request
  -> PayOrderController
  -> PayOrderCommand
  -> PayOrderUseCase
  -> OrderRepository.findById()
  -> OrderAggregate
  -> order.pay(paymentId)
  -> OrderRepository.save(order)
  -> OrderPaidEvent
  -> Response
```

对应层次：

```text
trigger
  -> application
      -> domain
      -> domain repository interface
          <- infrastructure repository impl
              <- mapper / dao / po / database
```

用代码感受一下。

## 1. trigger 层：接请求

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final PayOrderUseCase payOrderUseCase;

    @PostMapping("/{orderId}/pay")
    public Response<Void> pay(@PathVariable String orderId,
                              @RequestBody PayOrderRequest request) {
        PayOrderCommand command = new PayOrderCommand(
                orderId,
                request.getPaymentId()
        );

        payOrderUseCase.execute(command);

        return Response.success();
    }
}
```

这一层只做：

```text
1. 接 HTTP 参数
2. 做基础参数转换
3. 调用用例
4. 返回响应
```

不应该做：

```text
1. 判断订单是否能支付
2. 修改订单状态
3. 直接调 Mapper
4. 直接发 MQ
```

---

## 2. application / case 层：编排用例

```java
@Service
public class PayOrderUseCase {

    private final OrderRepository orderRepository;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public void execute(PayOrderCommand command) {
        OrderAggregate order = orderRepository.findById(
                new OrderId(command.orderId())
        );

        order.pay(new PaymentId(command.paymentId()));

        orderRepository.save(order);

        eventPublisher.publish(new OrderPaidEvent(order.getOrderId()));
    }
}
```

这一层负责：

```text
1. 一个业务用例的流程
2. 事务边界
3. 调用领域对象
4. 调用仓储
5. 发布事件
6. 协调多个领域能力
```

不负责：

```text
1. 订单状态是否合法的核心判断
2. MyBatis 查询细节
3. HTTP 请求参数解析
4. PO 和表结构操作
```

这里要特别说一下：小傅哥把 `case/application` 设为可选层。文章说，对于较大且复杂的项目，为了更好防腐和提供通用服务，一般会添加 `case/application` 层，用于对 domain 逻辑进行封装组合处理。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

我的建议更明确：

```text
真实项目里，建议保留 application / usecase 层。
```

否则 Controller 很容易直接调用 Domain Service，然后 Controller 又开始承担业务编排，最后退回 MVC。

---

## 3. domain 层：表达业务规则

```java
public class OrderAggregate {

    private OrderId orderId;
    private OrderStatus status;
    private Money amount;
    private List<OrderItem> items;

    public void pay(PaymentId paymentId) {
        if (this.status != OrderStatus.WAIT_PAY) {
            throw new DomainException("当前订单状态不允许支付");
        }

        if (this.amount.isZeroOrNegative()) {
            throw new DomainException("订单金额异常");
        }

        this.status = OrderStatus.PAID;
        this.paidAt = LocalDateTime.now();
        this.paymentId = paymentId;
    }
}
```

这一层的代码应该像业务语言：

```text
order.pay()
order.cancel()
order.confirm()
order.ship()
order.refund()
```

而不是：

```text
order.setStatus(PAID)
order.setPaidAt(now)
order.setPaymentId(paymentId)
```

DDD 的关键变化是：

```text
不是外部 Service 操纵贫血对象；
而是领域对象自己保护自己的状态变化。
```

这就是充血模型的意义。

---

## 4. domain repository：定义业务需要的数据能力

```java
public interface OrderRepository {

    OrderAggregate findById(OrderId orderId);

    void save(OrderAggregate order);
}
```

注意，这里不是：

```java
OrderPO selectById(Long id);

void updateStatus(Long id, Integer status);
```

为什么？

因为 domain 关心的是：

```text
加载订单聚合
保存订单聚合
```

而不是：

```text
查 order 表
改 status 字段
```

这就是抽象层次的差异。

---

## 5. infrastructure：实现技术细节

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderMapper orderMapper;
    private final OrderItemMapper orderItemMapper;
    private final OrderConverter orderConverter;

    @Override
    public OrderAggregate findById(OrderId orderId) {
        OrderPO orderPO = orderMapper.selectById(orderId.value());
        List<OrderItemPO> itemPOList = orderItemMapper.selectByOrderId(orderId.value());

        return orderConverter.toAggregate(orderPO, itemPOList);
    }

    @Override
    public void save(OrderAggregate order) {
        OrderPO orderPO = orderConverter.toOrderPO(order);
        List<OrderItemPO> itemPOList = orderConverter.toItemPOList(order);

        orderMapper.update(orderPO);
        orderItemMapper.batchUpdate(itemPOList);
    }
}
```

这一层可以出现：

```text
PO
Mapper
DAO
RedisTemplate
RabbitTemplate
RestTemplate
FeignClient
MyBatis
JPA
SQL
```

但这些东西不应该泄露到 domain。

一句话：

```text
infrastructure 是技术脏活层。
domain 是业务纯净层。
```

---

# 7. 每一层到底放什么？

下面是实战判断标准。

## trigger 层

小傅哥的 `trigger` 层非常值得保留。

文章里说，触发器层一般也叫 adapter 适配器层，用于提供接口实现、消息接收、任务执行等。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

它包括：

```text
http
  Controller

rpc
  Dubbo Provider / gRPC Adapter

mq
  Consumer / Listener

task
  Scheduled Job / XXL-Job Handler
```

它的本质是：

```text
外部世界进入系统的入口。
```

所以：

```text
HTTP 调用是触发；
RPC 调用是触发；
MQ 消息是触发；
定时任务也是触发。
```

它们只是入口不同，背后应该调用同一个用例能力。

例如：

```text
用户主动点击支付
  -> HTTP Controller
      -> PayOrderUseCase

支付回调通知
  -> HTTP CallbackController
      -> ConfirmPaymentUseCase

支付超时扫描
  -> Scheduled Task
      -> CloseTimeoutOrderUseCase

支付成功消息
  -> MQ Consumer
      -> HandlePaymentSucceededUseCase
```

这比 MVC 里到处写 Service 清晰得多。

---

## application / case 层

它负责“业务流程”，不负责“业务规则细节”。

比如支付订单：

```text
1. 查订单
2. 调用领域行为 order.pay()
3. 保存订单
4. 发布领域事件
```

这些是流程编排。

但下面这个判断：

```text
订单只有 WAIT_PAY 状态才能支付
```

应该在领域模型里。

所以分工是：

```text
Application 负责：先做什么，后做什么。
Domain 负责：什么是合法的业务行为。
```

---

## domain 层

domain 是 DDD 的核心。

文章中也明确说，无论怎么做 DDD 分层架构，`domain` 都肯定存在；每个服务包中有模型、仓库、服务三部分。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

典型结构：

```text
domain
  order
    model
      aggregate
      entity
      valobj
    repository
    service
```

这里要特别注意：

```text
domain 不是 service 包。
domain 是业务模型的家。
```

domain 里可以有：

```text
Aggregate
Entity
Value Object
Domain Service
Repository Interface
Domain Event
Specification
Policy
Factory
```

domain 里不应该有：

```text
Controller
DTO
PO
Mapper
DAO
RedisTemplate
RabbitTemplate
FeignClient
SQL
```

---

## infrastructure 层

这一层是“技术实现层”。

它负责把 domain 想要的抽象能力落到真实技术上。

例如：

```text
domain 说：我要保存订单。
infrastructure 说：我用 MyBatis 保存到 MySQL。

domain 说：我要查询规则树。
infrastructure 说：我查 RuleTree、RuleTreeNode、RuleTreeNodeLine 三张表，然后组装成聚合。

domain 说：我要调用支付网关。
infrastructure 说：我用 Feign 调支付服务，再把返回值转成领域对象。
```

所以 infrastructure 的职责是：

```text
1. 实现 Repository 接口
2. 调用 DAO / Mapper
3. 处理 PO 转 Domain
4. 处理外部接口适配
5. 处理缓存、消息、存储等技术细节
```

---

## app 层

文章中 `app` 是启动和配置层，放 SpringBoot 启动类、AOP、config、镜像打包等。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

也就是：

```text
Application.java
配置类
线程池配置
限流 AOP
日志配置
环境配置
Dockerfile
启动脚本
```

这里可以放技术性横切能力。

例如文章中放了限流 AOP，切点作用于 `trigger` 包下方法。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

这个设计可以理解为：

```text
app 层负责把整个应用装起来。
```

不要在 app 里写业务。

---

## api 层

主要服务于微服务 RPC 场景。

放：

```text
接口定义
Request
Response
DTO
```

例如：

```java
public interface IRuleService {
    DecisionMatterResponse decide(DecisionMatterRequest request);
}
```

调用方只需要依赖 `api` jar，不需要依赖你的整个应用。

注意：

```text
api 是对外契约，不是领域模型。
```

不要把 `OrderAggregate` 暴露给外部系统。

---

## types 层

文章里说 `types` 放 Response、Constants、枚举等通用类型。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

这个层可以有，但要克制。

可以放：

```text
Response
ErrorCode
BizException
Constants
通用枚举
```

不要放：

```text
OrderDTO
UserVO
ProductBO
各种 Utils
各种 Manager
```

否则它会变成新的垃圾桶。

我的建议：

```text
types 层越薄越好。
```

---

# 8. 这章最容易误解的地方

## 误解一：有这些目录就是 DDD

不是。

你可以建出这样的目录：

```text
domain
infrastructure
trigger
app
```

然后在 `domain service` 里继续写：

```java
orderMapper.selectById();
redisTemplate.opsForValue().get();
rabbitTemplate.convertAndSend();
```

这还是 MVC。

真正的 DDD 判断标准是：

```text
领域层是否表达业务？
领域层是否隔离技术？
业务规则是否内聚？
对象是否有行为？
依赖方向是否正确？
```

---

## 误解二：DDD 必须微服务

不是。

DDD 可以用于单体应用。

微服务只是让 DDD 的边界更有价值，因为不同限界上下文可以演化成不同服务。

但你完全可以先做：

```text
模块化单体 + DDD 分层
```

比如：

```text
order-context
payment-context
inventory-context
promotion-context
```

它们先在一个单体工程里，边界清晰即可。

等业务复杂度真的上来，再拆微服务。

---

## 误解三：聚合必须和数据库表一一对应

不是。

文章中说很多情况下 Entity 和 PO 是 1:1，但也可能出现 1:n。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

更准确地说：

```text
领域模型按业务一致性建模；
数据库表按存储效率和查询需求建模。
```

它们可以相似，但不是同一个东西。

例如订单聚合：

```text
OrderAggregate
  OrderEntity
  List<OrderItemEntity>
  Money
  Address
```

数据库可能是：

```text
order
order_item
order_payment
order_delivery
```

一个聚合可能对应多张表。

也可能多个简单值对象被存在一张表里。

所以：

```text
Domain 不等于 Table。
Entity 不等于 PO。
Aggregate 不等于 Join 查询结果。
```

---

## 误解四：所有业务逻辑都塞进聚合

也不对。

文章里也提醒，不要以为定义了聚合对象，就把超过一个对象以外的逻辑都封装到聚合中，否则后期会越来越难维护；聚合更应该关注和本对象相关的封装场景，重核心业务可以放到 service 实现。([GitHub](https://raw.githubusercontent.com/fuzhengwei/CodeGuide/master/docs/md/road-map/ddd.md "raw.githubusercontent.com"))

判断标准：

### 放聚合里

```text
这个规则只依赖聚合自身状态。
```

例如：

```java
order.pay();
order.cancel();
order.addItem();
order.changeAddress();
```

### 放领域服务里

```text
这个规则涉及多个聚合，或者不自然属于某一个实体。
```

例如：

```java
pricingService.calculatePrice(order, coupon, userLevel);
riskService.check(order, user);
inventoryDomainService.reserve(orderItems);
```

### 放应用服务里

```text
这是流程编排，不是业务规则本身。
```

例如：

```text
查订单 -> 校验支付 -> 扣库存 -> 保存订单 -> 发事件
```

---

# 9. 用一个完整案例把这章串起来

我们用“创建订单”来走一遍。

## 业务需求

```text
用户提交商品和收货地址，系统创建待支付订单。
规则：
1. 商品不能为空
2. 收货地址必须合法
3. 订单初始状态是 WAIT_PAY
4. 订单金额由订单项计算得出
5. 创建成功后保存订单
```

---

## MVC 典型写法

```java
@Service
public class OrderService {

    public void createOrder(CreateOrderRequest request) {
        if (request.getItems() == null || request.getItems().isEmpty()) {
            throw new BizException("商品不能为空");
        }

        BigDecimal totalAmount = BigDecimal.ZERO;
        for (ItemRequest item : request.getItems()) {
            ProductPO product = productMapper.selectById(item.getProductId());
            totalAmount = totalAmount.add(product.getPrice().multiply(new BigDecimal(item.getQuantity())));
        }

        OrderPO order = new OrderPO();
        order.setUserId(request.getUserId());
        order.setStatus("WAIT_PAY");
        order.setAmount(totalAmount);
        orderMapper.insert(order);

        for (ItemRequest item : request.getItems()) {
            OrderItemPO itemPO = new OrderItemPO();
            itemPO.setOrderId(order.getId());
            itemPO.setProductId(item.getProductId());
            itemPO.setQuantity(item.getQuantity());
            orderItemMapper.insert(itemPO);
        }
    }
}
```

问题：

```text
1. Service 直接处理业务规则
2. Service 直接处理 PO
3. Service 直接处理 Mapper
4. 订单没有行为，只是数据容器
5. 业务语义弱
```

---

## DDD 写法

### trigger

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final CreateOrderUseCase createOrderUseCase;

    @PostMapping
    public Response<CreateOrderResponse> create(@RequestBody CreateOrderRequest request) {
        CreateOrderCommand command = CreateOrderAssembler.toCommand(request);
        OrderId orderId = createOrderUseCase.execute(command);
        return Response.success(new CreateOrderResponse(orderId.value()));
    }
}
```

trigger 只接请求。

---

### application

```java
@Service
public class CreateOrderUseCase {

    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;

    @Transactional
    public OrderId execute(CreateOrderCommand command) {
        List<Product> products = productRepository.findByIds(command.productIds());

        OrderAggregate order = OrderAggregate.create(
                new UserId(command.userId()),
                command.toOrderItems(products),
                new Address(command.address())
        );

        orderRepository.save(order);

        return order.getOrderId();
    }
}
```

application 编排流程。

---

### domain aggregate

```java
public class OrderAggregate {

    private OrderId orderId;
    private UserId userId;
    private List<OrderItem> items;
    private Address address;
    private Money totalAmount;
    private OrderStatus status;

    public static OrderAggregate create(UserId userId,
                                        List<OrderItem> items,
                                        Address address) {
        if (items == null || items.isEmpty()) {
            throw new DomainException("订单商品不能为空");
        }

        OrderAggregate order = new OrderAggregate();
        order.orderId = OrderId.generate();
        order.userId = userId;
        order.items = items;
        order.address = address;
        order.totalAmount = calculateTotalAmount(items);
        order.status = OrderStatus.WAIT_PAY;

        return order;
    }

    private static Money calculateTotalAmount(List<OrderItem> items) {
        return items.stream()
                .map(OrderItem::subtotal)
                .reduce(Money.ZERO, Money::add);
    }
}
```

业务规则进入领域对象。

---

### domain repository

```java
public interface OrderRepository {

    void save(OrderAggregate order);

    OrderAggregate findById(OrderId orderId);
}
```

domain 定义它需要什么。

---

### infrastructure repository

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderMapper orderMapper;
    private final OrderItemMapper orderItemMapper;
    private final OrderConverter orderConverter;

    @Override
    public void save(OrderAggregate order) {
        OrderPO orderPO = orderConverter.toOrderPO(order);
        List<OrderItemPO> itemPOList = orderConverter.toOrderItemPOList(order);

        orderMapper.insert(orderPO);
        orderItemMapper.batchInsert(itemPOList);
    }
}
```

infrastructure 处理数据库细节。

---

# 10. 这章的真正精髓：对象不要乱穿层

DDD 工程里，最容易乱的是对象。

你要记住这条对象边界：

```text
Request DTO
  -> Command
  -> Domain Object
  -> PO
```

对应：

```text
trigger
  -> application
  -> domain
  -> infrastructure
```

对象流转应该像这样：

```text
HTTP Request DTO
  -> Assembler
  -> Command
  -> UseCase
  -> Aggregate / Entity / Value Object
  -> Repository
  -> Converter
  -> PO
  -> Mapper
  -> Database
```

关键禁忌：

```text
DTO 不进入 Domain
PO 不离开 Infrastructure
Domain Object 不直接返回给前端
Mapper 不进入 Application / Domain
```

错误写法：

```java
public void pay(PayOrderRequest request) {
    OrderPO orderPO = orderMapper.selectById(request.getOrderId());
    orderPO.setStatus("PAID");
    orderMapper.update(orderPO);
}
```

较好写法：

```java
public void pay(PayOrderCommand command) {
    OrderAggregate order = orderRepository.findById(new OrderId(command.orderId()));
    order.pay(new PaymentId(command.paymentId()));
    orderRepository.save(order);
}
```

这就是 DDD 工程分层的核心价值。

---

# 11. 小傅哥这章的质量怎么评价？

我给它一个比较客观的评价。

## 它好的地方

```text
1. 非常适合 Java 后端入门 DDD 工程结构
2. 没有停留在空泛理论，而是给了真实模块拆分
3. trigger 这个说法很好理解
4. 强调 infrastructure 依赖 domain，是正确的依赖倒置方向
5. 给了完整工程 tree，能让初学者看到 DDD 项目长什么样
6. 把 HTTP / RPC / MQ / TASK 统一成触发入口，这个抽象很实用
```

## 它不够好的地方

```text
1. 和前面概念章节重复较多
2. Application / UseCase 层的重要性讲得不够重
3. 聚合、实体、值对象的边界仍然偏简略
4. 容易让初学者误以为“按目录建包就是 DDD”
5. 对领域事件、ACL、上下文边界、事务一致性没有充分展开
6. 一些表达偏经验化，不是严格的 DDD 标准教材式表述
```

所以我的最终评价是：

```text
作为 DDD 工程骨架入门：8 分
作为 DDD 理论教材：6.5 分
作为 Java 后端落地参考：8.5 分
```

你现在不需要把它当标准答案，而应该把它当作：

```text
一个可以参考的 Java DDD 工程模板。
```

---

# 12. 你读这章时应该重点看什么？

你不用逐字读。

按这个顺序看：

```text
1. 看 MVC 问题：为什么 Service 会膨胀
2. 看“触发 -> 函数 -> 连接”：理解外部入口和业务能力分离
3. 看模块分层：api / app / domain / infrastructure / trigger / types / case
4. 看依赖方向：infrastructure 依赖 domain，而不是反过来
5. 看 domain 内部结构：model / repository / service
6. 看 infrastructure：DAO / PO / RepositoryImpl
7. 看 trigger：HTTP / RPC / MQ / TASK
8. 看 app：启动、配置、AOP
```

略看的部分：

```text
环境配置
Docker
AOP 限流实现细节
测试启动步骤
```

这些是工程配置，不是 DDD 主线。

---

# 13. 一句话把每层背下来

你可以直接记这个版本：

```text
trigger：谁来触发我。
application：这个用例怎么编排。
domain：业务规则是什么。
infrastructure：技术细节怎么实现。
api：我对外暴露什么契约。
app：应用怎么启动和装配。
types：通用类型放哪里。
```

再压缩成更短：

```text
trigger 接请求；
application 编流程；
domain 管业务；
infrastructure 接技术；
api 定契约；
app 做装配；
types 放通用。
```

---

# 14. 你真正应该形成的判断力

学完这章，你要能判断下面这些问题。

## 问题 1：Controller 能不能直接调 Mapper？

不能。

```text
Controller 属于 trigger。
Mapper 属于 infrastructure。
它们之间应该隔着 application 和 domain。
```

---

## 问题 2：Domain 能不能依赖 MyBatis？

不能。

```text
Domain 应该是业务核心。
MyBatis 是技术实现。
业务核心不应该依赖技术细节。
```

---

## 问题 3：Repository 接口放哪？

放 domain。

```text
因为 Repository 表达的是领域需要的数据访问能力。
```

---

## 问题 4：Repository 实现放哪？

放 infrastructure。

```text
因为 RepositoryImpl 需要使用 Mapper / DAO / PO / SQL。
```

---

## 问题 5：PO 能不能传到 domain？

不要。

```text
PO 是持久化对象。
Domain Object 是业务对象。
两者职责不同。
```

---

## 问题 6：DTO 能不能传到 domain？

不要。

```text
DTO 是接口传输对象。
Domain 不应该感知接口协议。
```

---

## 问题 7：复杂流程写在哪里？

写 application / case。

```text
Application 编排多个领域对象、仓储、外部服务。
```

---

## 问题 8：核心业务规则写在哪里？

优先写 domain。

```text
和聚合自身状态强相关的规则，放聚合。
跨对象但属于业务规则的，放领域服务。
```

---

# 15. 最终满分版总结

这章你要这样理解：

```text
DDD 架构设计的本质，不是多建几个 module。
而是通过工程分层，把复杂系统里的不同变化原因隔离开。
```

再说得工程一点：

```text
trigger 隔离外部入口变化；
application 隔离用例流程变化；
domain 承载核心业务规则；
infrastructure 隔离数据库、缓存、MQ、外部接口变化；
api 隔离服务契约；
app 负责启动装配；
types 提供少量通用类型。
```

最核心的一条调用链是：

```text
HTTP / RPC / MQ / TASK
  -> Trigger
  -> Application / Case
  -> Domain Service / Aggregate
  -> Repository Interface
  -> Infrastructure Repository Impl
  -> DAO / Mapper
  -> PO
  -> Database
```

最核心的一条依赖规则是：

```text
业务核心不依赖技术细节。
技术细节反过来实现业务核心定义的接口。
```

最核心的一条对象规则是：

```text
DTO 不进 Domain；
PO 不出 Infrastructure；
Domain Object 不等于数据库表；
Controller 不写核心业务；
Mapper 不进入领域层。
```

所以，这章真正想教你的不是“DDD 有哪些概念”，而是：

> **一个 Java 后端项目，如何把业务复杂度和技术复杂度分开管理。**

这才是 DDD 架构设计的价值。