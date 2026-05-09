# 总述
在 DDD 的设计方法中，领域层做到了只关心领域服务实现。最能体现这样设计的就是仓库和适配器的设计。通常在 `Service + 数据模型`的设计中，会在 Service 中引入 Redis、RPC、配置中心等各类其他外部服务。但在 DDD 中，通过仓储和适配器以及基础设施层的定义，解耦了这部分内容。
![[DDD仓储和适配器.png]]- 特征：
    
    - 封装持久化操作：Repository负责封装所有与数据源交互的操作，如创建、读取、更新和删除（CRUD）操作。这样，领域层的代码就可以避免直接处理数据库或其他存储机制的复杂性。
    - 领域对象的集合管理：Repository通常被视为领域对象的集合，提供了查询和过滤这些对象的方法，使得领域对象的获取和管理更加方便。
    - 抽象接口：Repository定义了一个与持久化机制无关的接口，这使得领域层的代码可以在不同的持久化机制之间切换，而不需要修改业务逻辑。
- 用途：
    
    - 数据访问抽象：Repository为领域层提供了一个清晰的数据访问接口，使得领域对象可以专注于业务逻辑的实现，而不是数据访问的细节。
    - 领域对象的查询和管理：Repository使得对领域对象的查询和管理变得更加方便和灵活，支持复杂的查询逻辑。
    - 领域逻辑与数据存储分离：通过Repository模式，领域逻辑与数据存储逻辑分离，提高了领域模型的纯粹性和可测试性。
    - 优化数据访问：Repository实现可以包含数据访问的优化策略，如缓存、批处理操作等，以提高应用程序的性能。
- 实现手段：
    
    - 定义Repository接口：在领域层定义一个或多个Repository接口，这些接口声明了所需的数据访问方法。
    - 实现Repository接口：在基础设施层或数据访问层实现这些接口，具体实现可能是使用ORM（对象关系映射）框架，如MyBatis、Hibernate等，或者直接使用数据库访问API，如JDBC等。
    - 依赖注入：在应用程序中使用依赖注入（DI）来将具体的Repository实现注入到需要它们的领域服务或应用服务中。这样做可以进一步解耦领域层和数据访问层，同时也便于单元测试。
    - 使用规范模式（Specification Pattern）：有时候，为了构建复杂的查询，可以结合使用规范模式，这是一种允许将业务规则封装为单独的业务逻辑单元的模式，这些单元可以被Repository用来构建查询。

> Repository模式是DDD（领域驱动设计）中的一个核心概念，它有助于保持领域模型的聚焦和清晰，同时提供了灵活、可测试和可维护的数据访问策略。

仓储解耦的手段使用了依赖倒置的设计，所有领域需要的外部服务，不在直接引入外部的服务，而是通过定义接口的方式，让基础设施层实现领域层接口（仓储/适配器）的方式来处理。

那么也就是基础设置层负责原则对接`数据库`、`缓存`、`配置中心`、`RPC接口`、`HTTP接口`、`MQ推送`等各项资源，并承接领域服务的接口调用各项服务为领域层提供数据能力。

同时这也会体现出，领域层的实现是具有业务语义的，而到了基础设置层则没有业务语义，都是原子的方法。通过原子方法的组合为领域业务语义提供支撑。

# DDD 中的“仓储”和“适配器”

一句话：

> **仓储负责隔离持久化细节，适配器负责隔离外部系统细节。它们共同目标是：让领域层只表达业务规则，不关心技术实现。**

---

# 1. 先看传统 Service 写法的问题

很多 Java 后端项目会这样写：

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private InventoryRpcClient inventoryRpcClient;

    @Autowired
    private ConfigService configService;

    public void submitOrder(Long orderId) {
        OrderDO orderDO = orderMapper.selectById(orderId);

        Boolean locked = inventoryRpcClient.lockStock(orderDO.getSkuId(), orderDO.getCount());

        if (!locked) {
            throw new RuntimeException("库存不足");
        }

        redisTemplate.opsForValue().set("order:" + orderId, orderDO);

        orderDO.setStatus("SUBMITTED");
        orderMapper.updateById(orderDO);
    }
}
```

这类代码的问题是：

```text
业务逻辑
数据库访问
Redis 缓存
RPC 调用
配置中心
事务控制
技术异常处理
```

全部混在一个 Service 里。

结果就是：

```text
Service 变成了大杂烩
领域规则散落在技术代码中
业务模型被数据模型绑架
更换 Redis / RPC / ORM 成本很高
单元测试很难写
```

---

# 2. DDD 的核心处理方式

DDD 会把依赖关系反过来。

不是领域层依赖数据库、Redis、RPC，而是：

```text
领域层定义自己需要什么能力
基础设施层去实现这些能力
```

可以理解为：

```text
领域层说：我需要“保存订单”
基础设施层说：我用 MySQL 帮你保存

领域层说：我需要“锁定库存”
基础设施层说：我用 RPC 调库存系统

领域层说：我需要“读取风控规则”
基础设施层说：我从配置中心读取
```

领域层只面向抽象，不面向具体技术。

---

# 3. 仓储 Repository 是什么？

## 3.1 仓储的本质

DDD 中的 Repository 不是简单的 DAO，也不是 Mapper。

它的本质是：

> **以聚合为单位，提供对象的获取和保存能力。**

注意关键词：**聚合**。

比如订单系统里有一个订单聚合：

```text
Order 聚合
├── OrderId
├── BuyerId
├── OrderItem
├── OrderStatus
└── Money
```

那么仓储应该围绕 `Order` 聚合设计，而不是围绕数据库表设计。

---

# 4. Repository 和 DAO / Mapper 的区别

|对比项|DAO / Mapper|DDD Repository|
|---|---|---|
|面向对象|数据表|聚合根|
|返回对象|DO / PO / Entity 数据对象|领域对象 / 聚合根|
|所属层|基础设施层|接口在领域层，实现通常在基础设施层|
|关注点|SQL、CRUD、字段映射|聚合的保存、加载、重建|
|粒度|表级别|聚合级别|
|典型方法|`selectById`, `insert`, `update`|`findById`, `save`, `nextIdentity`|

传统 Mapper：

```java
public interface OrderMapper {
    OrderDO selectById(Long id);
    void insert(OrderDO orderDO);
    void updateById(OrderDO orderDO);
}
```

DDD Repository：

```java
public interface OrderRepository {

    Order find(OrderId orderId);

    void save(Order order);

    OrderId nextIdentity();
}
```

区别非常关键：

```text
Mapper 关心 OrderDO 怎么存
Repository 关心 Order 聚合怎么被取出和保存
```

---

# 5. Repository 接口放在哪里？

通常放在**领域层**。

```text
domain
├── model
│   └── Order.java
├── repository
│   └── OrderRepository.java
└── service
    └── OrderDomainService.java
```

领域层定义接口：

```java
public interface OrderRepository {

    Order find(OrderId orderId);

    void save(Order order);
}
```

基础设施层实现接口：

```text
infrastructure
├── persistence
│   ├── OrderRepositoryImpl.java
│   ├── OrderMapper.java
│   ├── OrderDO.java
│   └── OrderConverter.java
```

实现类：

```java
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderMapper orderMapper;

    public OrderRepositoryImpl(OrderMapper orderMapper) {
        this.orderMapper = orderMapper;
    }

    @Override
    public Order find(OrderId orderId) {
        OrderDO orderDO = orderMapper.selectById(orderId.value());
        return OrderConverter.toDomain(orderDO);
    }

    @Override
    public void save(Order order) {
        OrderDO orderDO = OrderConverter.toDO(order);
        orderMapper.saveOrUpdate(orderDO);
    }
}
```

关键点：

```text
领域层知道 OrderRepository
领域层不知道 OrderMapper
领域层不知道 OrderDO
领域层不知道 MyBatis
领域层不知道 MySQL
```

---

# 6. Repository 不是“什么都能查”的万能查询器

这是初学 DDD 很容易犯的错。

不推荐这样：

```java
public interface OrderRepository {

    List<Order> queryByUserId(Long userId);

    List<Order> queryByStatus(String status);

    List<Order> queryByDateRange(LocalDateTime start, LocalDateTime end);

    Page<Order> pageQuery(OrderQuery query);

    BigDecimal sumAmountByUserId(Long userId);
}
```

这会让 Repository 变成传统 DAO。

更好的方式是：

```java
public interface OrderRepository {

    Order find(OrderId orderId);

    void save(Order order);
}
```

复杂查询通常可以放到：

```text
application query service
read model
CQRS query repository
report repository
```

例如：

```java
public interface OrderQueryService {

    Page<OrderDTO> queryUserOrders(UserOrderQuery query);
}
```

也就是说：

```text
Repository 主要服务于领域模型
QueryService 主要服务于页面展示 / 报表查询
```

---

# 7. 什么是适配器 Adapter？

适配器的本质是：

> **把外部系统、第三方服务、技术组件包装成领域层可理解的接口。**

外部系统包括：

```text
Redis
MQ
RPC / HTTP Client
配置中心
短信服务
支付网关
风控系统
文件存储
搜索引擎
AI 服务
```

领域层不应该直接看到这些东西。

比如领域层不应该直接写：

```java
redisTemplate.opsForValue().get(...)
```

也不应该直接写：

```java
paymentFeignClient.pay(...)
```

也不应该直接写：

```java
apolloConfig.getProperty(...)
```

而应该定义一个领域需要的抽象接口。

---

# 8. 适配器示例：库存服务

## 8.1 错误写法

应用服务直接依赖 RPC Client：

```java
@Service
public class OrderApplicationService {

    private final InventoryFeignClient inventoryFeignClient;

    public void submitOrder(SubmitOrderCommand command) {
        Boolean success = inventoryFeignClient.lockStock(
            command.getSkuId(),
            command.getQuantity()
        );

        if (!success) {
            throw new RuntimeException("库存不足");
        }
    }
}
```

这个写法的问题是：

```text
订单业务被库存 RPC 协议污染
Feign 细节泄漏到业务逻辑
库存系统返回值变化会影响订单服务
测试时必须 mock RPC client
```

---

## 8.2 DDD 写法

领域层定义接口：

```java
public interface InventoryService {

    boolean lockStock(SkuId skuId, Quantity quantity);
}
```

基础设施层实现：

```java
@Component
public class InventoryRpcAdapter implements InventoryService {

    private final InventoryFeignClient inventoryFeignClient;

    public InventoryRpcAdapter(InventoryFeignClient inventoryFeignClient) {
        this.inventoryFeignClient = inventoryFeignClient;
    }

    @Override
    public boolean lockStock(SkuId skuId, Quantity quantity) {
        LockStockRequest request = new LockStockRequest(
            skuId.value(),
            quantity.value()
        );

        LockStockResponse response = inventoryFeignClient.lockStock(request);

        return response.isSuccess();
    }
}
```

业务代码只依赖：

```java
private final InventoryService inventoryService;
```

而不是：

```java
private final InventoryFeignClient inventoryFeignClient;
```

这样一来，领域层只知道：

```text
我需要锁库存
```

不知道：

```text
这是 HTTP 调用
这是 Dubbo 调用
这是 Feign 调用
这是 gRPC 调用
这是 MQ 异步调用
```

---

# 9. 适配器示例：Redis 缓存

很多人会问：

> Redis 在 DDD 里放哪里？

答案通常是：

```text
Redis 属于基础设施层
领域层不直接依赖 Redis
```

不推荐：

```java
public class UserDomainService {

    private final RedisTemplate<String, Object> redisTemplate;
}
```

推荐：

```java
public interface UserLoginAttemptCache {

    int getFailedCount(UserId userId);

    void increaseFailedCount(UserId userId);

    void clear(UserId userId);
}
```

基础设施层：

```java
@Component
public class RedisUserLoginAttemptCache implements UserLoginAttemptCache {

    private final RedisTemplate<String, Object> redisTemplate;

    @Override
    public int getFailedCount(UserId userId) {
        String key = "login:failed:" + userId.value();
        Object value = redisTemplate.opsForValue().get(key);
        return value == null ? 0 : (int) value;
    }

    @Override
    public void increaseFailedCount(UserId userId) {
        String key = "login:failed:" + userId.value();
        redisTemplate.opsForValue().increment(key);
    }

    @Override
    public void clear(UserId userId) {
        String key = "login:failed:" + userId.value();
        redisTemplate.delete(key);
    }
}
```

领域层看到的是：

```text
登录失败次数缓存
```

而不是：

```text
Redis key
RedisTemplate
序列化方式
TTL
连接池
```

---

# 10. 仓储和适配器的区别

|项目|Repository 仓储|Adapter 适配器|
|---|---|---|
|主要目的|保存和加载聚合|对接外部系统或技术组件|
|面向对象|聚合根|外部能力|
|常见实现|MyBatis、JPA、MongoDB|Redis、RPC、MQ、HTTP、配置中心|
|接口位置|领域层|领域层或应用层|
|实现位置|基础设施层|基础设施层|
|典型命名|`OrderRepository`|`PaymentGateway`, `InventoryService`, `MessageSender`|
|关注点|领域对象持久化|外部能力适配|

简单理解：

```text
Repository：领域对象怎么存、怎么取
Adapter：外部能力怎么接入
```

---

# 11. 更准确的分层关系

一个常见 DDD 分层结构：

```text
interfaces / adapter
├── controller
├── request
└── response

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
│   ├── mapper
│   ├── dataobject
│   └── repositoryimpl
├── rpc
│   ├── client
│   └── adapter
├── cache
│   └── redis
├── mq
│   └── publisher
└── config
```

依赖方向：

```text
Controller
   ↓
Application Service
   ↓
Domain Model / Domain Service
   ↓
Repository 接口 / 外部服务接口
   ↑
Infrastructure 实现
```

重点是：

```text
domain 不依赖 infrastructure
infrastructure 依赖 domain
```

这就是所谓的**依赖倒置**。

---

# 12. 结合一次下单流程看

## 12.1 领域层

```java
public class Order {

    private OrderId id;
    private OrderStatus status;
    private List<OrderItem> items;

    public void submit() {
        if (items.isEmpty()) {
            throw new DomainException("订单明细不能为空");
        }

        if (status != OrderStatus.CREATED) {
            throw new DomainException("只有待提交订单可以提交");
        }

        this.status = OrderStatus.SUBMITTED;
    }
}
```

领域对象只关心自己的业务规则。

---

## 12.2 Repository 接口

```java
public interface OrderRepository {

    Order find(OrderId orderId);

    void save(Order order);
}
```

---

## 12.3 外部能力接口

```java
public interface InventoryGateway {

    boolean lockStock(List<OrderItem> items);
}
```

也可以叫：

```text
InventoryService
InventoryClient
InventoryGateway
InventoryPort
```

我个人更推荐 `Gateway` 或 `Port`，因为它明显表达了“外部能力边界”。

---

## 12.4 应用服务编排

```java
@Service
public class SubmitOrderApplicationService {

    private final OrderRepository orderRepository;
    private final InventoryGateway inventoryGateway;

    public void submit(SubmitOrderCommand command) {
        Order order = orderRepository.find(new OrderId(command.orderId()));

        boolean stockLocked = inventoryGateway.lockStock(order.getItems());

        if (!stockLocked) {
            throw new ApplicationException("库存锁定失败");
        }

        order.submit();

        orderRepository.save(order);
    }
}
```

这里应用服务负责：

```text
加载聚合
调用外部能力
执行业务行为
保存聚合
```

但它不关心：

```text
订单存在 MySQL 还是 MongoDB
库存是 Dubbo 还是 HTTP
缓存是 Redis 还是 Caffeine
消息是 RocketMQ 还是 Kafka
```

---

## 12.5 基础设施实现

```java
@Component
public class InventoryRpcGateway implements InventoryGateway {

    private final InventoryFeignClient inventoryFeignClient;

    @Override
    public boolean lockStock(List<OrderItem> items) {
        LockStockRequest request = convert(items);
        LockStockResponse response = inventoryFeignClient.lockStock(request);
        return response.success();
    }
}
```

这就是适配器。

---

# 13. Port / Adapter 架构视角

DDD 经常和六边形架构一起使用。

六边形架构里有两个关键词：

```text
Port：端口，也就是领域层定义的接口
Adapter：适配器，也就是基础设施层的实现
```

例如：

```text
OrderRepository 是 Port
OrderRepositoryImpl 是 Adapter

PaymentGateway 是 Port
AlipayPaymentAdapter 是 Adapter

InventoryGateway 是 Port
InventoryRpcAdapter 是 Adapter

MessagePublisher 是 Port
RocketMqMessagePublisher 是 Adapter
```

所以你可以这样理解：

```text
Port 是领域层向外声明的能力需求
Adapter 是基础设施层对这个能力需求的具体实现
```

---

# 14. 为什么这套设计重要？

## 14.1 领域层变干净

领域层不再出现：

```java
RedisTemplate
RestTemplate
FeignClient
JdbcTemplate
Mapper
DO
PO
MQProducer
ConfigService
```

领域层只出现：

```java
Order
OrderItem
OrderRepository
InventoryGateway
PaymentGateway
DomainEventPublisher
```

这让代码更接近业务语言。

---

## 14.2 技术替换成本低

例如你原来用 Feign 调库存：

```text
InventoryFeignClient
```

后来改成 MQ 异步库存锁定：

```text
InventoryMqAdapter
```

应用层和领域层可以尽量不变，只替换基础设施实现。

---

## 14.3 单元测试更容易

你可以直接 mock 接口：

```java
class FakeInventoryGateway implements InventoryGateway {

    @Override
    public boolean lockStock(List<OrderItem> items) {
        return true;
    }
}
```

测试领域行为时，不需要启动：

```text
MySQL
Redis
Nacos
Kafka
RPC Server
```

---

## 14.4 业务语义更清楚

对比这两个字段：

```java
private RedisTemplate<String, Object> redisTemplate;
```

和：

```java
private UserLoginAttemptCache userLoginAttemptCache;
```

后者明显更表达业务含义。

这就是 DDD 的价值：

```text
不是为了多写接口
而是为了让代码表达业务边界
```

---

# 15. 容易混淆的点

## 15.1 Repository 不是数据库表 Mapper

错误理解：

```text
一张表一个 Repository
```

更准确：

```text
一个聚合根一个 Repository
```

例如：

```text
OrderRepository 管理 Order 聚合
而不是 OrderTableRepository、OrderItemTableRepository
```

Order 聚合内部可能涉及多张表：

```text
order
order_item
order_address
order_discount
```

但领域层只看到：

```java
OrderRepository.save(order)
```

---

## 15.2 Adapter 不一定只在 interface 层

很多人看到 Adapter 会想到：

```text
Controller Adapter
```

但在 DDD / 六边形架构中，Adapter 泛指所有边界适配。

包括：

```text
Web Adapter
Persistence Adapter
RPC Adapter
MQ Adapter
Cache Adapter
Config Adapter
```

也就是说：

```text
Controller 是入站适配器
RepositoryImpl 是出站适配器
RpcClientWrapper 是出站适配器
```

---

## 15.3 不是所有外部调用都应该放进领域层接口

例如：

```text
发短信验证码
导出 Excel
上传头像
查询列表页
统计报表
```

这些不一定是领域核心能力，很多时候放在应用层接口或基础设施服务即可。

判断标准：

```text
这个外部能力是否参与领域规则判断？
```

如果参与核心业务规则，可以抽象成领域接口。

比如：

```text
支付状态确认
库存锁定
风控校验
额度冻结
合同签署
```

这些往往需要通过 Gateway / Port 隔离。

---

# 16. 实战命名建议

## Repository

```java
OrderRepository
UserRepository
AccountRepository
ProductRepository
ContractRepository
```

方法建议：

```java
Order find(OrderId id);

void save(Order order);

void remove(Order order);

OrderId nextIdentity();
```

少放复杂查询。

---

## Gateway / Adapter

接口命名：

```java
PaymentGateway
InventoryGateway
RiskControlGateway
MessagePublisher
UserLoginAttemptCache
ConfigProvider
FileStorage
```

实现命名：

```java
AlipayPaymentGateway
InventoryRpcGateway
RedisUserLoginAttemptCache
NacosConfigProvider
S3FileStorage
RocketMqMessagePublisher
```

或者：

```java
PaymentGatewayImpl
InventoryGatewayImpl
```

但如果有多种实现，推荐带技术名：

```java
AlipayPaymentGateway
WechatPaymentGateway
RedisUserLoginAttemptCache
CaffeineUserLoginAttemptCache
```

---

# 17. 最后用一张图总结

```text
                 ┌────────────────────────┐
                 │      Controller         │
                 │   接收 HTTP 请求        │
                 └───────────┬────────────┘
                             │
                             ▼
                 ┌────────────────────────┐
                 │ Application Service     │
                 │ 编排用例 / 事务控制      │
                 └───────────┬────────────┘
                             │
                             ▼
                 ┌────────────────────────┐
                 │        Domain           │
                 │ 实体 / 值对象 / 聚合      │
                 │ 领域服务 / 领域规则       │
                 └───────────┬────────────┘
                             │
                只依赖抽象接口 │
                             ▼
       ┌──────────────────────────────────────┐
       │ Repository / Gateway / Port 接口      │
       └───────────────┬──────────────────────┘
                       │ 由基础设施层实现
                       ▼
       ┌──────────────────────────────────────┐
       │ Infrastructure                        │
       │ MyBatis / Redis / RPC / MQ / Nacos    │
       └──────────────────────────────────────┘
```

核心结论：

```text
Repository 解决“领域对象怎么存取”
Adapter 解决“外部能力怎么接入”
Infrastructure 解决“具体技术怎么实现”
Domain 只保留“业务规则和业务语言”
```

所以，DDD 里仓储和适配器最重要的价值不是“多套一层”，而是：

> **把领域模型从数据库、缓存、RPC、MQ、配置中心等技术细节中解放出来。**