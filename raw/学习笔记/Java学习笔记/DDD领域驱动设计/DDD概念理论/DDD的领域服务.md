## Domain Service 的作用是什么？

在 DDD 里，**Domain Service，领域服务**，用于承载那些：

> **属于领域规则，但不适合放进某一个 Entity / Value Object / Aggregate 里的业务行为。**

一句话：

> **Entity 负责“我自己的行为”，Domain Service 负责“多个领域对象之间的业务协作或独立领域规则”。**

---

# 1. 先看它解决什么问题

假设订单系统里有一个业务：

> 创建订单时，需要根据用户、商品、优惠券、库存、地址等信息计算最终订单价格。

这个逻辑明显是业务逻辑，不是简单 CRUD。

但它应该放在哪里？

```java
User user;
List<Product> products;
Coupon coupon;
Address address;
```

如果放进 `Order`：

```java
order.calculatePrice(user, products, coupon, address);
```

就有点奇怪。

因为此时订单可能还没创建出来，而且价格计算依赖了很多外部领域对象。

如果放进 `Product`：

```java
product.calculateOrderPrice(...);
```

也不合适，因为这是订单维度的规则，不是商品自己的行为。

如果放进 Application Service：

```java
OrderAppService.createOrder(...)
```

也不理想，因为价格计算是核心业务规则，不应该变成应用层的过程式代码。

这时就可以引入：

```java
OrderPricingService
```

它就是一个 **Domain Service**。

---

# 2. Domain Service 的典型特征

Domain Service 通常有几个特征：

|特征|说明|
|---|---|
|无状态|通常不保存业务状态|
|表达领域动作|名字通常是业务动作，而不是技术动作|
|承载领域规则|里面是业务判断、业务计算、业务协作|
|不属于单个实体|逻辑无法自然放进某一个 Entity|
|可以操作多个聚合/实体/值对象|但要注意不要破坏聚合边界|

---

# 3. 举个订单价格计算例子

## 不推荐：把核心业务逻辑写在 Application Service

```java
@Service
public class OrderApplicationService {

    public Order createOrder(CreateOrderCommand command) {
        User user = userRepository.findById(command.userId());
        List<Product> products = productRepository.findByIds(command.productIds());
        Coupon coupon = couponRepository.findById(command.couponId());

        BigDecimal total = BigDecimal.ZERO;

        for (Product product : products) {
            total = total.add(product.getPrice());
        }

        if (coupon != null && coupon.isAvailable()) {
            total = total.subtract(coupon.getDiscountAmount());
        }

        if (user.isVip()) {
            total = total.multiply(new BigDecimal("0.9"));
        }

        Order order = new Order(user.getId(), total);

        orderRepository.save(order);

        return order;
    }
}
```

这段代码的问题是：

> Application Service 变成了“业务规则大杂烩”。

它不只是编排流程，还在决定：

- 商品如何计价；
    
- 优惠券如何抵扣；
    
- VIP 如何打折；
    
- 订单最终金额如何生成。
    

这些都是领域规则。

---

## 更合适：抽出 Domain Service

```java
public class OrderPricingService {

    public Money calculate(
            User user,
            List<Product> products,
            Coupon coupon
    ) {
        Money total = Money.zero();

        for (Product product : products) {
            total = total.add(product.price());
        }

        if (coupon != null && coupon.canUseBy(user)) {
            total = total.subtract(coupon.discountAmount());
        }

        if (user.isVip()) {
            total = total.multiply("0.9");
        }

        return total;
    }
}
```

Application Service 只负责流程编排：

```java
@Service
public class OrderApplicationService {

    private final UserRepository userRepository;
    private final ProductRepository productRepository;
    private final CouponRepository couponRepository;
    private final OrderRepository orderRepository;
    private final OrderPricingService orderPricingService;

    public Order createOrder(CreateOrderCommand command) {
        User user = userRepository.findById(command.userId());
        List<Product> products = productRepository.findByIds(command.productIds());
        Coupon coupon = couponRepository.findById(command.couponId());

        Money totalAmount = orderPricingService.calculate(user, products, coupon);

        Order order = Order.create(
                user.getId(),
                products,
                totalAmount
        );

        orderRepository.save(order);

        return order;
    }
}
```

这样职责就清楚了：

|对象|职责|
|---|---|
|`OrderApplicationService`|编排用例流程|
|`OrderPricingService`|承载订单定价领域规则|
|`Order`|表达订单自身状态和行为|
|`Money`|表达金额计算规则|
|Repository|获取和保存领域对象|

---

# 4. Domain Service 和 Application Service 的区别

这是最容易混的地方。

## Application Service：应用服务

它负责：

> **完成一个用例流程。**

比如：

```text
用户下单
支付订单
取消订单
确认收货
```

它通常做这些事：

- 接收 Command / DTO；
    
- 调用 Repository 查询聚合；
    
- 调用 Domain Service；
    
- 调用聚合方法；
    
- 保存聚合；
    
- 发布事件；
    
- 控制事务。
    

它偏“流程编排”。

---

## Domain Service：领域服务

它负责：

> **表达领域规则。**

比如：

```text
订单定价
库存分配
风控授信
优惠券匹配
配送费计算
账号转账
```

它偏“业务判断和业务计算”。

---

## 对比表

|对比项|Application Service|Domain Service|
|---|---|---|
|所属层|应用层|领域层|
|主要职责|用例编排|领域规则|
|是否包含业务规则|尽量少|是|
|是否关心事务|通常关心|通常不关心|
|是否调用 Repository|经常调用|尽量少调用，视情况而定|
|是否调用外部接口|可以通过端口调用|通常不直接调用技术设施|
|命名|`OrderApplicationService` / `CreateOrderUseCase`|`OrderPricingService` / `TransferService`|

---

# 5. Domain Service 和 Entity 的区别

判断一个业务方法该放 Entity 还是 Domain Service，可以用这个标准：

> **如果这个行为天然属于某个对象，就放进 Entity。  
> 如果这个行为跨多个对象，且不属于某个单一对象，就放进 Domain Service。**

---

## 应该放 Entity 的例子

订单支付：

```java
public class Order {

    private OrderStatus status;
    private PaymentId paymentId;

    public void pay(PaymentId paymentId) {
        if (this.status != OrderStatus.CREATED) {
            throw new BusinessException("只有待支付订单可以支付");
        }

        this.paymentId = paymentId;
        this.status = OrderStatus.PAID;
    }
}
```

这个逻辑应该放进 `Order`。

因为：

> 支付会改变订单自身状态，这是订单自己的行为。

---

## 应该放 Domain Service 的例子

跨账户转账：

```java
public class TransferService {

    public void transfer(Account from, Account to, Money amount) {
        if (from.sameAs(to)) {
            throw new BusinessException("不能给自己转账");
        }

        from.withdraw(amount);
        to.deposit(amount);
    }
}
```

这里为什么不是 `from.transferTo(to, amount)`？

其实也可以。

但如果转账规则越来越复杂：

- 检查转出账户状态；
    
- 检查转入账户状态；
    
- 检查转账限额；
    
- 检查手续费；
    
- 检查风控；
    
- 生成转账流水；
    
- 处理跨币种汇率；
    

这时抽成 `TransferService` 更自然。

---

# 6. Domain Service 不是什么？

## 不是“Service 层”

很多 Java 项目里喜欢写：

```java
UserService
OrderService
ProductService
```

然后里面全是：

```java
create()
update()
delete()
query()
```

这种通常不是 DDD 的 Domain Service。

它更像传统 MVC 里的 Service。

DDD 的 Domain Service 不是为了“每张表配一个 Service”，而是为了表达**领域行为**。

---

## 不推荐的 Domain Service

```java
public class OrderDomainService {

    public void createOrder() {}

    public void updateOrder() {}

    public void deleteOrder() {}

    public void queryOrder() {}
}
```

这不太像领域服务。

它只是 CRUD Service。

---

## 更好的 Domain Service 命名

```java
OrderPricingService
InventoryAllocationService
CouponMatchingService
RiskControlService
ShippingFeeCalculator
CreditEvaluationService
TransferService
```

这些名字更像领域服务，因为它们表达的是具体业务能力。

---

# 7. Domain Service 是否可以调用 Repository？

这个问题有争议。

我的建议是：

> **可以，但要克制。**

更严格的 DDD 风格里，Domain Service 尽量只接收已经加载好的领域对象：

```java
public Money calculate(User user, List<Product> products, Coupon coupon) {
    // 领域计算
}
```

而不是自己去查：

```java
public Money calculate(UserId userId, List<ProductId> productIds, CouponId couponId) {
    User user = userRepository.findById(userId);
    List<Product> products = productRepository.findByIds(productIds);
    Coupon coupon = couponRepository.findById(couponId);

    // 领域计算
}
```

更推荐前者。

因为加载数据是应用层编排职责，领域服务应该专注业务规则。

但在复杂领域中，某些领域服务确实可能需要 Repository，例如：

```java
public class AccountUniquenessChecker {

    private final AccountRepository accountRepository;

    public boolean isUnique(AccountName accountName) {
        return !accountRepository.existsByName(accountName);
    }
}
```

这种也可以接受，因为“账号名唯一性”本身就是领域规则，而 Repository 是领域层定义的接口。

---

# 8. 判断是否需要 Domain Service 的标准

可以用这几个问题判断：

## 适合用 Domain Service

当你发现某段逻辑：

1. 是核心业务规则；
    
2. 不只是技术流程；
    
3. 不自然属于某一个实体；
    
4. 需要多个领域对象协作；
    
5. 如果放 Application Service，会导致应用层变胖；
    
6. 如果强塞进 Entity，会导致实体职责怪异。
    

这时就可以考虑 Domain Service。

---

## 不适合用 Domain Service

下面这些通常不该放 Domain Service：

|逻辑|更适合放在哪里|
|---|---|
|参数校验|Controller / Application Service / Command Validator|
|查数据库|Repository|
|发 HTTP 请求|Infrastructure Adapter|
|发 MQ|Application Service / Event Publisher|
|简单 CRUD|Application Service|
|实体自身状态变化|Entity / Aggregate Root|
|金额、时间、地址等小规则|Value Object|

---

# 9. 在分层架构里的位置

DDD 常见分层：

```text
interfaces
  └── controller / trigger

application
  └── application service / use case

domain
  ├── aggregate
  ├── entity
  ├── value object
  ├── domain service
  ├── repository interface
  └── domain event

infrastructure
  ├── repository impl
  ├── mq
  ├── rpc
  └── db
```

Domain Service 位于：

```text
domain 层
```

它可以依赖：

```text
Entity
Value Object
Aggregate
Repository interface
Domain Event
Domain Policy / Specification
```

但不应该直接依赖：

```text
Controller
DTO
MyBatis Mapper
JPA Entity
RedisTemplate
RestTemplate
MQ Client
```

---

# 10. 最关键的一句话

你可以这样记：

> **聚合根负责维护自己的业务一致性；  
> Domain Service 负责表达不属于单个聚合根的领域规则；  
> Application Service 负责把一次业务用例跑完整。**

对应到订单系统：

```text
Order.pay()
```

放在 `Order`。

```text
OrderPricingService.calculate()
```

放在 Domain Service。

```text
CreateOrderApplicationService.createOrder()
```

放在 Application Service。

---

# 11. 最小判断口诀

```text
能放实体，优先放实体；
能放值对象，优先放值对象；
跨多个对象，又是领域规则，放 Domain Service；
只是流程编排，放 Application Service；
只是技术实现，放 Infrastructure。
```

Domain Service 的价值不是“多加一层”，而是防止核心业务规则散落在应用层、Controller、Mapper、工具类里。它让领域模型保持表达力，同时避免把不属于单个对象的行为硬塞进实体。