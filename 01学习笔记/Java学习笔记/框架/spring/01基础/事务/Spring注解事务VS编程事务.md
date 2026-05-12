# Spring 事务的两种实际用法

Spring 事务主要分两类：

```text
1. 声明式事务：@Transactional
2. 编程式事务：TransactionTemplate / PlatformTransactionManager
```

大多数业务代码优先用 **注解事务**。  
遇到更细粒度控制时，再用 **编程事务**。

---

# 一、注解事务：`@Transactional`

## 1. 最常见用法：一个业务方法一个事务

比如用户下单：

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private AccountMapper accountMapper;

    @Transactional(rollbackFor = Exception.class)
    public void createOrder(CreateOrderCommand command) {
        // 1. 创建订单
        OrderPO order = new OrderPO();
        order.setUserId(command.getUserId());
        order.setProductId(command.getProductId());
        order.setAmount(command.getAmount());
        orderMapper.insert(order);

        // 2. 扣减余额
        accountMapper.decreaseBalance(command.getUserId(), command.getAmount());

        // 3. 模拟异常
        if (command.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new RuntimeException("订单金额异常");
        }
    }
}
```

这个方法的事务含义是：

```text
createOrder 方法开始前：开启事务
方法正常结束：提交事务
方法抛出异常：回滚事务
```

也就是说：

```text
订单插入成功 + 扣余额成功 => 一起提交
订单插入成功 + 扣余额失败 => 一起回滚
```

---

## 2. 为什么常写 `rollbackFor = Exception.class`

默认情况下，Spring 只会对：

```text
RuntimeException
Error
```

自动回滚。

也就是说，下面这种受检异常默认不一定回滚：

```java
@Transactional
public void importUser() throws IOException {
    userMapper.insert(...);

    throw new IOException("文件读取失败");
}
```

如果希望所有 `Exception` 都回滚，应该写：

```java
@Transactional(rollbackFor = Exception.class)
public void importUser() throws IOException {
    userMapper.insert(...);

    throw new IOException("文件读取失败");
}
```

企业项目里常见写法：

```java
@Transactional(rollbackFor = Exception.class)
```

原因很简单：

```text
业务失败了，就别提交一半数据。
```

---

# 二、注解事务的几个高频参数

## 1. `propagation`：事务传播行为

传播行为解决的是：

> 一个事务方法调用另一个事务方法时，事务应该怎么处理？

最常用的是：

```java
@Transactional(propagation = Propagation.REQUIRED)
```

含义：

```text
外面有事务：加入外面的事务
外面没有事务：新建一个事务
```

示例：

```java
@Service
public class OrderService {

    @Autowired
    private CouponService couponService;

    @Transactional(rollbackFor = Exception.class)
    public void createOrder() {
        orderMapper.insert(...);

        couponService.freezeCoupon(...);

        paymentMapper.insert(...);
    }
}
```

```java
@Service
public class CouponService {

    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void freezeCoupon() {
        couponMapper.freeze(...);
    }
}
```

调用链：

```text
OrderService.createOrder()
    -> CouponService.freezeCoupon()
```

如果两个方法都是 `REQUIRED`，那么它们在同一个事务里。

结果是：

```text
订单创建失败，优惠券冻结也回滚
优惠券冻结失败，订单创建也回滚
```

---

## 2. `REQUIRES_NEW`：强制新开事务

有些场景需要“主流程回滚，但日志必须保留”。

例如：

```java
@Service
public class OrderService {

    @Autowired
    private OperationLogService operationLogService;

    @Transactional(rollbackFor = Exception.class)
    public void createOrder() {
        try {
            orderMapper.insert(...);

            throw new RuntimeException("下单失败");
        } catch (Exception e) {
            operationLogService.saveLog("下单失败：" + e.getMessage());
            throw e;
        }
    }
}
```

日志服务：

```java
@Service
public class OperationLogService {

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,
        rollbackFor = Exception.class
    )
    public void saveLog(String content) {
        operationLogMapper.insert(content);
    }
}
```

这里的效果是：

```text
createOrder 使用一个事务
saveLog 强制开启一个新事务
```

所以：

```text
订单事务回滚
日志事务提交
```

这就是 `REQUIRES_NEW` 的典型用途。

适合场景：

```text
操作日志
审计日志
失败记录
消息发送记录
补偿任务记录
```

---

## 3. `NESTED`：嵌套事务

`NESTED` 表示嵌套事务，依赖数据库 savepoint。

```java
@Transactional(propagation = Propagation.NESTED)
public void deductCoupon() {
    couponMapper.use(...);
}
```

它的效果更像：

```text
外层事务中设置一个保存点
内层失败，可以回滚到保存点
外层事务仍可继续
```

示意：

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() {
    orderMapper.insert(...);

    try {
        couponService.deductCoupon();
    } catch (Exception e) {
        // 优惠券扣减失败，但订单流程继续
    }

    paymentMapper.insert(...);
}
```

不过实际项目里，`NESTED` 用得比 `REQUIRED` 和 `REQUIRES_NEW` 少。

原因是：

```text
1. 依赖数据库和事务管理器支持
2. 可读性不如 REQUIRES_NEW 清晰
3. 很多团队不希望业务流程依赖复杂嵌套事务
```

---

## 4. `isolation`：隔离级别

隔离级别解决的是并发事务之间的数据可见性问题。

常见写法：

```java
@Transactional(isolation = Isolation.DEFAULT)
```

表示使用数据库默认隔离级别。

MySQL InnoDB 默认一般是：

```text
REPEATABLE READ
```

更常见的业务代码通常不主动改隔离级别。

只有在特殊场景下才会设置：

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
public void updateInventory() {
    inventoryMapper.decrease(...);
}
```

面试里要知道四种隔离级别：

|隔离级别|脏读|不可重复读|幻读|
|---|--:|--:|--:|
|READ_UNCOMMITTED|可能|可能|可能|
|READ_COMMITTED|避免|可能|可能|
|REPEATABLE_READ|避免|避免|MySQL InnoDB 通常可避免|
|SERIALIZABLE|避免|避免|避免|

实际开发中更重要的是：

```text
隔离级别不是越高越好。
隔离级别越高，并发性能通常越差。
```

---

## 5. `timeout`：事务超时时间

```java
@Transactional(timeout = 10, rollbackFor = Exception.class)
public void createOrder() {
    orderMapper.insert(...);
    inventoryMapper.decrease(...);
}
```

含义：

```text
事务最多执行 10 秒。
超过时间后，事务会被标记为回滚。
```

适合用于：

```text
防止事务长时间占用连接
防止长时间持有数据库锁
防止慢 SQL 拖垮业务线程
```

但是要注意：

```text
timeout 不是万能熔断器。
它主要约束事务执行，不等于完整的业务超时控制。
```

---

## 6. `readOnly`：只读事务

查询方法可以写：

```java
@Transactional(readOnly = true)
public OrderDTO getOrder(Long orderId) {
    OrderPO order = orderMapper.selectById(orderId);
    return convert(order);
}
```

作用：

```text
告诉 Spring 和数据库：这是只读事务，可以做一些优化。
```

但是不要过度迷信它。

实际效果取决于：

```text
数据库
驱动
事务管理器
ORM 框架
```

一般原则是：

```text
查询类方法可以加 readOnly = true
写操作方法不要加
```

---

# 三、注解事务最容易踩的坑

## 坑 1：同类方法内部调用，事务不生效

错误示例：

```java
@Service
public class OrderService {

    public void createOrder() {
        this.doCreateOrder();
    }

    @Transactional(rollbackFor = Exception.class)
    public void doCreateOrder() {
        orderMapper.insert(...);
    }
}
```

这里 `doCreateOrder()` 的事务可能不会生效。

原因是：

```text
Spring 事务基于 AOP 代理。
只有通过 Spring 代理对象调用方法，事务拦截器才会生效。
this.doCreateOrder() 是对象内部调用，绕过了代理。
```

正确做法之一：

```java
@Service
public class OrderService {

    @Autowired
    private OrderCreateService orderCreateService;

    public void createOrder() {
        orderCreateService.doCreateOrder();
    }
}
```

```java
@Service
public class OrderCreateService {

    @Transactional(rollbackFor = Exception.class)
    public void doCreateOrder() {
        orderMapper.insert(...);
    }
}
```

---

## 坑 2：异常被 catch 掉，事务不会回滚

错误示例：

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() {
    try {
        orderMapper.insert(...);
        inventoryMapper.decrease(...);
    } catch (Exception e) {
        log.error("创建订单失败", e);
    }
}
```

问题是：

```text
异常被 catch 掉了，没有继续抛出。
Spring 认为方法正常结束，于是提交事务。
```

正确做法：

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() {
    try {
        orderMapper.insert(...);
        inventoryMapper.decrease(...);
    } catch (Exception e) {
        log.error("创建订单失败", e);
        throw e;
    }
}
```

或者手动标记回滚：

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() {
    try {
        orderMapper.insert(...);
        inventoryMapper.decrease(...);
    } catch (Exception e) {
        log.error("创建订单失败", e);
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
}
```

但一般更推荐：

```text
记录日志后继续抛异常。
```

---

## 坑 3：`private` 方法事务不生效

错误示例：

```java
@Transactional(rollbackFor = Exception.class)
private void doCreateOrder() {
    orderMapper.insert(...);
}
```

原因：

```text
Spring AOP 默认基于代理。
private 方法不能被代理增强。
```

通常事务方法应该是：

```java
public
```

---

## 坑 4：事务方法所在类没有交给 Spring 管理

错误示例：

```java
public class OrderService {

    @Transactional
    public void createOrder() {
        orderMapper.insert(...);
    }
}
```

如果这个类不是 Spring Bean，比如没有：

```java
@Service
@Component
```

那 `@Transactional` 不会生效。

正确写法：

```java
@Service
public class OrderService {

    @Transactional(rollbackFor = Exception.class)
    public void createOrder() {
        orderMapper.insert(...);
    }
}
```

---

## 坑 5：事务里做了太多慢操作

不推荐：

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() {
    orderMapper.insert(...);

    // 远程调用
    paymentClient.prepay(...);

    // 发 MQ
    mqClient.send(...);

    // 调第三方接口
    smsClient.send(...);
}
```

问题：

```text
事务时间太长
数据库连接长期占用
行锁长期持有
外部接口慢会拖住数据库事务
```

更好的做法：

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() {
    orderMapper.insert(...);
    orderEventMapper.insert(new OrderCreatedEvent(...));
}

// 事务提交后，异步发送 MQ / 调用外部系统
```

核心原则：

```text
事务里只做必要的数据库操作。
不要把远程调用、耗时计算、大文件处理放进事务里。
```

---

# 四、编程式事务：`TransactionTemplate`

注解事务适合“大块业务方法”。  
编程式事务适合“局部事务控制”。

## 1. 基本用法

```java
@Service
public class OrderService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private InventoryMapper inventoryMapper;

    public void createOrder(CreateOrderCommand command) {

        transactionTemplate.executeWithoutResult(status -> {
            orderMapper.insert(...);
            inventoryMapper.decrease(...);
        });

    }
}
```

这段代码的含义是：

```text
transactionTemplate 内部的代码在事务中执行。
代码正常结束：提交事务。
代码抛出异常：回滚事务。
```

---

## 2. 编程事务可以返回结果

```java
public Long createOrder(CreateOrderCommand command) {
    Long orderId = transactionTemplate.execute(status -> {
        OrderPO order = new OrderPO();
        order.setUserId(command.getUserId());
        orderMapper.insert(order);

        inventoryMapper.decrease(command.getProductId());

        return order.getId();
    });

    return orderId;
}
```

适合这种场景：

```text
事务内创建数据
事务外继续使用结果
```

---

## 3. 手动回滚

```java
public void createOrder(CreateOrderCommand command) {
    transactionTemplate.executeWithoutResult(status -> {
        try {
            orderMapper.insert(...);
            inventoryMapper.decrease(...);
        } catch (Exception e) {
            status.setRollbackOnly();
            log.error("创建订单失败", e);
        }
    });
}
```

这里没有继续抛异常，但通过：

```java
status.setRollbackOnly();
```

手动标记事务回滚。

不过实际项目里，更常见的是直接抛异常：

```java
transactionTemplate.executeWithoutResult(status -> {
    orderMapper.insert(...);
    inventoryMapper.decrease(...);

    if (库存不足) {
        throw new BizException("库存不足");
    }
});
```

---

# 五、编程事务的典型使用场景

## 场景 1：事务之后再执行非事务逻辑

比如：

```text
1. 保存订单，需要事务
2. 事务提交成功后，再发送 MQ
```

如果用注解事务，容易把 MQ 发送也包进事务。

编程式事务可以写得更清楚：

```java
public void createOrder(CreateOrderCommand command) {

    Long orderId = transactionTemplate.execute(status -> {
        OrderPO order = new OrderPO();
        orderMapper.insert(order);

        inventoryMapper.decrease(command.getProductId());

        return order.getId();
    });

    // 事务提交后再执行
    mqProducer.sendOrderCreatedMessage(orderId);
}
```

这个结构非常清晰：

```text
transactionTemplate 里面：数据库事务
transactionTemplate 外面：非事务操作
```

这是编程事务的一个核心价值。

---

## 场景 2：循环处理时，每条数据一个独立事务

比如批量导入用户：

```java
public void importUsers(List<UserImportDTO> users) {
    for (UserImportDTO user : users) {
        try {
            transactionTemplate.executeWithoutResult(status -> {
                userMapper.insert(convert(user));
                userProfileMapper.insert(convertProfile(user));
            });
        } catch (Exception e) {
            log.error("导入用户失败，user={}", user, e);
        }
    }
}
```

效果：

```text
第 1 条成功，提交
第 2 条失败，回滚
第 3 条成功，提交
```

如果直接在整个方法上加：

```java
@Transactional
```

那就是：

```text
一条失败，全部回滚
```

所以批处理场景经常会用编程事务。

---

## 场景 3：一段方法中只有中间部分需要事务

```java
public void processOrder(CreateOrderCommand command) {
    // 1. 参数校验，不需要事务
    checkParam(command);

    // 2. 查询外部系统，不建议放进事务
    ProductInfo productInfo = productClient.getProduct(command.getProductId());

    // 3. 只有数据库写入需要事务
    Long orderId = transactionTemplate.execute(status -> {
        orderMapper.insert(...);
        inventoryMapper.decrease(...);
        return order.getId();
    });

    // 4. 后置通知，不需要事务
    notifyService.notifyOrderCreated(orderId);
}
```

这种写法比直接给整个方法加 `@Transactional` 更干净。

因为事务范围被精确收缩到了：

```text
真正需要原子性的数据库操作。
```

---

# 六、更底层的编程事务：`PlatformTransactionManager`

`TransactionTemplate` 是对 `PlatformTransactionManager` 的封装。  
如果要更底层控制，可以直接使用：

```java
@Autowired
private PlatformTransactionManager transactionManager;
```

示例：

```java
public void createOrder(CreateOrderCommand command) {
    DefaultTransactionDefinition definition = new DefaultTransactionDefinition();
    definition.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    definition.setIsolationLevel(TransactionDefinition.ISOLATION_DEFAULT);
    definition.setTimeout(30);

    TransactionStatus status = transactionManager.getTransaction(definition);

    try {
        orderMapper.insert(...);
        inventoryMapper.decrease(...);

        transactionManager.commit(status);
    } catch (Exception e) {
        transactionManager.rollback(status);
        throw e;
    }
}
```

这种写法更啰嗦，一般业务代码不常用。

优先级通常是：

```text
@Transactional > TransactionTemplate > PlatformTransactionManager
```

实际项目中：

```text
@Transactional 用得最多
TransactionTemplate 用于精细控制
PlatformTransactionManager 较少直接使用
```

---

# 七、注解事务 vs 编程事务

|对比项|注解事务|编程事务|
|---|---|---|
|写法|`@Transactional`|`TransactionTemplate`|
|侵入性|低|较高|
|可读性|简洁|事务边界更明确|
|控制粒度|方法级别为主|代码块级别|
|常见程度|极高|中等|
|适合场景|普通业务方法|局部事务、批处理、事务后置逻辑|
|主要问题|AOP 代理失效、异常被吞|代码略复杂|

一句话：

```text
大部分业务用注解事务。
需要精确控制事务边界时，用编程事务。
```

---

# 八、实际项目推荐写法

## 1. 普通写业务：用注解事务

```java
@Transactional(rollbackFor = Exception.class)
public void payOrder(Long orderId) {
    orderMapper.updateStatus(orderId, OrderStatus.PAID);
    accountMapper.addIncome(orderId);
}
```

---

## 2. 事务外要发消息：用编程事务

```java
public void payOrder(Long orderId) {
    transactionTemplate.executeWithoutResult(status -> {
        orderMapper.updateStatus(orderId, OrderStatus.PAID);
        accountMapper.addIncome(orderId);
    });

    mqProducer.sendPaySuccessMessage(orderId);
}
```

---

## 3. 批量导入：每条一个事务

```java
public ImportResult importData(List<DataDTO> dataList) {
    ImportResult result = new ImportResult();

    for (DataDTO data : dataList) {
        try {
            transactionTemplate.executeWithoutResult(status -> {
                dataMapper.insert(convert(data));
            });
            result.success();
        } catch (Exception e) {
            result.fail(data, e.getMessage());
        }
    }

    return result;
}
```

---

## 4. 日志必须落库：用 `REQUIRES_NEW`

```java
@Transactional(rollbackFor = Exception.class)
public void createOrder() {
    try {
        orderMapper.insert(...);
        inventoryMapper.decrease(...);
    } catch (Exception e) {
        operationLogService.saveFailLog(e.getMessage());
        throw e;
    }
}
```

```java
@Transactional(
    propagation = Propagation.REQUIRES_NEW,
    rollbackFor = Exception.class
)
public void saveFailLog(String message) {
    operationLogMapper.insert(message);
}
```

---

# 九、面试加分项

## 1. 能说清楚 Spring 事务本质

可以这样说：

```text
Spring 声明式事务本质上是基于 AOP 代理实现的。
调用事务方法时，代理对象会在目标方法执行前开启事务，
方法正常返回则提交事务，
方法抛出符合回滚规则的异常则回滚事务。
```

关键词：

```text
AOP
代理对象
TransactionInterceptor
PlatformTransactionManager
commit
rollback
```

---

## 2. 能说清楚事务为什么会失效

高频事务失效场景：

|场景|原因|
|---|---|
|同类方法内部调用|绕过 Spring 代理|
|private 方法|无法被代理增强|
|final 方法|可能无法被代理|
|类没有交给 Spring 管理|不是 Bean，没有代理|
|异常被 catch 掉|Spring 感知不到异常|
|抛出受检异常但没配置 rollbackFor|默认不回滚|
|多线程中使用事务|事务上下文基于线程绑定，子线程拿不到|
|数据库表不支持事务|比如 MySQL MyISAM|

这块是面试高频。

---

## 3. 能区分 `REQUIRED` 和 `REQUIRES_NEW`

这个非常加分。

```text
REQUIRED：
外面有事务就加入，没有就新建。

REQUIRES_NEW：
无论外面有没有事务，都新建一个事务。
如果外面有事务，先挂起外层事务。
```

案例：

```text
下单失败要回滚订单，但失败日志必须保存。
```

答案：

```text
订单方法用 REQUIRED。
日志方法用 REQUIRES_NEW。
```

---

## 4. 能说清楚为什么事务里不要放远程调用

可以这样说：

```text
事务期间会占用数据库连接，并可能持有行锁。
如果在事务中调用远程接口、发送 MQ、处理文件，就会拉长事务时间。
一旦外部服务变慢，会导致数据库连接和锁资源被长时间占用，进而影响系统吞吐。
```

更好的做法：

```text
先在事务中落库。
事务提交后，再发消息或执行外部调用。
必要时使用本地消息表、事务消息、Outbox Pattern。
```

这个是高级项目经验点。

---

## 5. 能说出事务和 MQ 的一致性问题

比如：

```java
@Transactional
public void createOrder() {
    orderMapper.insert(...);
    mqProducer.send(...);
}
```

这里有问题：

```text
数据库还没提交，消息已经发出去了。
如果后面事务回滚，消费者可能拿到一条不存在的订单消息。
```

更好的方案：

```text
1. 本地消息表
2. RocketMQ 事务消息
3. Outbox Pattern
4. 事务提交后回调发送消息
```

这属于分布式系统加分项。

---

## 6. 能说清楚事务隔离级别和锁不是一回事

很多人会混。

可以这样回答：

```text
事务隔离级别定义的是并发事务之间的数据可见性规则。
锁是数据库实现隔离性的一种手段。
隔离级别越高，通常并发冲突越多，但具体行为还要看数据库实现。
```

例如 MySQL InnoDB：

```text
MVCC
快照读
当前读
行锁
间隙锁
Next-Key Lock
```

这些和事务隔离级别有关，但不是同一个概念。

---

## 7. 能说出 `@Transactional` 应该放在哪一层

推荐放在：

```text
Service 层 / Application 层
```

不推荐放在：

```text
Controller 层
Mapper 层
Entity 层
```

原因：

```text
事务应该包住一个完整业务用例。
Controller 只是协议入口。
Mapper 只是单表/SQL 操作。
Service/Application 才是业务编排边界。
```

比如：

```java
@RestController
public class OrderController {

    @PostMapping("/orders")
    public void createOrder(@RequestBody CreateOrderRequest request) {
        orderApplicationService.createOrder(request);
    }
}
```

```java
@Service
public class OrderApplicationService {

    @Transactional(rollbackFor = Exception.class)
    public void createOrder(CreateOrderRequest request) {
        orderService.createOrder(...);
        inventoryService.decrease(...);
        couponService.freeze(...);
    }
}
```

事务放在这里更合理：

```text
一个用例，一个事务边界。
```

---

# 十、最终记忆版

## 注解事务

```java
@Transactional(rollbackFor = Exception.class)
public void businessMethod() {
    // 多个数据库操作
}
```

适合：

```text
一个业务方法整体要么成功，要么失败。
```

---

## 编程事务

```java
transactionTemplate.executeWithoutResult(status -> {
    // 只有这段代码在事务里
});
```

适合：

```text
只想让部分代码进入事务。
事务提交后还要执行外部调用。
批量处理时每条数据一个事务。
```

---

## 面试一句话总结

可以这样答：

> Spring 事务分声明式事务和编程式事务。声明式事务通过 `@Transactional` 基于 AOP 代理实现，适合大多数 Service 层业务方法；编程式事务通过 `TransactionTemplate` 或 `PlatformTransactionManager` 手动控制事务边界，适合局部事务、批处理独立提交、事务提交后再执行外部动作等场景。实际开发中优先使用注解事务，遇到事务边界需要精细控制时再使用编程事务。同时要注意事务失效问题，比如同类方法内部调用、异常被 catch、受检异常未配置 `rollbackFor`、方法不是 public、类没交给 Spring 管理等。