# 总述
当你对数据库的操作需要使用到多个实体时，可以创建聚合对象。一个聚合对象，代表着一个数据库事务，具有事务一致性。聚合中的实体可以由聚合提供创建操作，实体也被称为聚合根对象。一个订单的聚合，会涵盖：下单用户实体对象、订单实体、订单明细实体和订单收货四级地址值对象。而那个作为入参的购物车实体对象，已经被转换为实体对象了。—— 聚合内事务一致性，聚合外最终一致性。

- 概念：聚合是领域模型中的一个关键概念，它是一组具有内聚性的相关对象的集合，这些对象一起工作以执行某些业务规则或操作。聚合定义了一组对象的边界，这些对象可以被视为一个单一的单元进行处理。
- 特征：
    - **一致性边界**：聚合确保其内部对象的状态变化是一致的。当对聚合内的对象进行操作时，这些操作必须保持聚合内所有对象的一致性。
    - **根实体**：每个聚合都有一个根实体（Aggregate Root），它是聚合的入口点。根实体拥有一个全局唯一的标识符，其他对象通过根实体与聚合交互。
    - **事务边界**：聚合也定义了事务的边界。在聚合内部，所有的变更操作应该是原子的，即它们要么全部成功，要么全部失败，以此来保证数据的一致性。
- 用途：
    1. **封装业务逻辑**：聚合通过将相关的对象和操作封装在一起，提供了一个清晰的业务逻辑模型，有助于业务规则的实施和维护。
    2. **保证一致性**：聚合确保内部状态的一致性，通过定义清晰的边界和规则，聚合可以在内部强制执行业务规则，从而保证数据的一致性。
    3. **简化复杂性**：聚合通过组织相关的对象，简化了领域模型的复杂性。这有助于开发者更好地理解和扩展系统。
- 实现手段：
    - **定义聚合根**：选择合适的聚合根是实现聚合的第一步。聚合根应该是能够代表整个聚合的实体，并且拥有唯一标识。
    - **限制访问路径**：只能通过聚合根来修改聚合内的对象，不允许直接修改聚合内部对象的状态，以此来维护边界和一致性。
    - **设计事务策略**：在聚合内部实现事务一致性，确保操作要么全部完成，要么全部回滚。对于聚合之间的交互，可以采用领域事件或其他机制来实现最终一致性。
    - **封装业务规则**：在聚合内部实现业务规则和逻辑，确保所有的业务操作都遵循这些规则。
    - **持久化**：聚合根通常与数据持久化层交互，以保存聚合的状态。这通常涉及到对象-关系映射（ORM）或其他数据映射技术。

# DDD 里的聚合：Aggregate

DDD 里的 **聚合 Aggregate**，是比实体和值对象更高一层的建模概念。

它解决的问题不是“一个对象怎么设计”，而是：

> **一组强相关的领域对象，应该如何组织、修改、持久化，并保证业务一致性。**

一句话：

> **聚合是一组领域对象的边界，聚合根是这组对象对外暴露的唯一入口。**

---

# 一、为什么需要聚合？

你前面已经学了：

```text
Entity：有身份、有生命周期、有业务行为
Value Object：没有身份，用属性值表达概念
```

但是实际业务里，一个领域模型通常不是孤立对象。

比如订单：

```text
Order
 ├── OrderItem
 ├── ShippingAddress
 ├── Coupon
 ├── PaymentInfo
 └── InvoiceInfo
```

问题来了：

```text
外部代码能不能直接改 OrderItem？
能不能绕过 Order 修改订单状态？
Order 和 OrderItem 是不是必须一起保存？
一个事务里应该修改哪些对象？
哪些对象之间必须强一致？
哪些对象可以最终一致？
```

聚合就是用来回答这些问题的。

---

# 二、聚合是什么？

聚合可以理解为：

> **一个业务一致性边界。**

在这个边界内部，对象之间需要保持强一致。

例如订单聚合：

```text
Order 聚合
 ├── Order：聚合根
 ├── OrderItem：实体
 ├── Money：值对象
 └── Address：值对象
```

聚合内部要保证：

```text
订单不能为空
订单总价必须等于订单项价格之和
已提交订单不能再随便加商品
已支付订单不能修改收货地址
订单项数量不能小于 1
```

这些规则应该由聚合根 `Order` 统一保护。

---

# 三、聚合根是什么？

每个聚合必须有一个 **聚合根 Aggregate Root**。

聚合根也是实体，但它有特殊地位：

> **外部对象只能持有聚合根的引用，不能直接操作聚合内部对象。**

比如：

```text
Order 是聚合根
OrderItem 是聚合内实体
```

外部代码不应该这样：

```java
order.getItems().add(newItem);
```

而应该这样：

```java
order.addItem(productId, productName, price, quantity);
```

原因是：

```text
Order 需要检查订单状态
Order 需要检查商品数量
Order 需要重新计算总价
Order 需要保证订单内部一致性
```

如果外部可以直接改 `items`，聚合根就失去了保护边界的意义。

---

# 四、聚合、聚合根、实体、值对象的关系

可以这么理解：

```text
Aggregate 聚合
 └── Aggregate Root 聚合根，特殊的 Entity
      ├── Entity
      ├── Entity
      ├── Value Object
      └── Value Object
```

订单例子：

```text
OrderAggregate
 ├── Order              聚合根，Entity
 ├── OrderItem          聚合内 Entity
 ├── Money              Value Object
 └── ShippingAddress    Value Object
```

文章例子：

```text
ArticleAggregate
 ├── Article            聚合根，Entity
 ├── ArticleTitle       Value Object
 ├── ArticleContent     Value Object
 ├── AuthorId           Value Object
 └── ArticleStatus      Enum
```

评论例子可能是：

```text
CommentAggregate
 ├── Comment            聚合根，Entity
 ├── CommentContent     Value Object
 ├── ArticleId          Value Object
 └── CommentStatus      Enum
```

注意：

> 聚合不一定很大。很多时候，一个聚合只有一个实体，加几个值对象。

---

# 五、聚合的核心规则

## 1. 外部只能通过聚合根访问聚合内部

错误写法：

```java
OrderItem item = order.getItems().get(0);
item.changeQuantity(999);
```

这会绕过 `Order` 的规则。

更好的写法：

```java
order.changeItemQuantity(itemId, 999);
```

由 `Order` 判断：

```text
订单是否还允许修改
订单项是否存在
数量是否合法
总价是否需要重新计算
```

---

## 2. Repository 只操作聚合根

在 DDD 里，通常只给聚合根配 Repository。

推荐：

```java
public interface OrderRepository {
    Optional<Order> findById(OrderId orderId);

    void save(Order order);
}
```

不推荐：

```java
public interface OrderItemRepository {
    Optional<OrderItem> findById(OrderItemId itemId);

    void save(OrderItem item);
}
```

因为 `OrderItem` 不应该被外部单独加载、单独保存。

它的生命周期属于 `Order`。

---

## 3. 一个事务只修改一个聚合

这是 DDD 里非常重要的工程规则。

理想情况下：

```text
一个应用服务方法
一个事务
加载一个聚合根
调用聚合根业务方法
保存这个聚合根
```

示例：

```java
@Transactional
public void submitOrder(Long orderId) {
    Order order = orderRepository.findById(new OrderId(orderId))
            .orElseThrow(() -> new IllegalArgumentException("订单不存在"));

    order.submit();

    orderRepository.save(order);
}
```

这类用例很清晰：

```text
事务边界 = 聚合边界
一致性边界 = 聚合边界
保存边界 = 聚合边界
```

---

## 4. 聚合之间通过 ID 引用，不直接对象引用

错误写法：

```java
public class Order {
    private User user;
    private List<Product> products;
}
```

这会让聚合之间耦合过重。

更好的写法：

```java
public class Order {
    private UserId userId;
    private List<OrderItem> items;
}
```

`OrderItem` 里面可以存：

```java
public class OrderItem {
    private ProductId productId;
    private ProductName productName;
    private Money priceSnapshot;
    private int quantity;
}
```

订单创建时，可以把商品当时的名称和价格做快照。

这样订单不会强依赖 `Product` 聚合的当前状态。

原因很现实：

```text
商品改名了，不应该影响历史订单
商品涨价了，不应该影响已下单价格
用户改昵称了，不应该影响订单归属
```

---

# 六、聚合是“一致性边界”

这是理解聚合的关键。

聚合内部的数据应该强一致。

聚合之间的数据通常用最终一致。

## 订单内部强一致

例如：

```text
Order.totalAmount 必须等于所有 OrderItem 小计之和
OrderItem.quantity 不能小于 1
Order.status = PAID 后不能修改 items
```

这些必须在同一个聚合里保证。

---

## 订单和库存可以最终一致

例如下单时涉及：

```text
Order 聚合
Inventory 库存聚合
Coupon 优惠券聚合
Account 用户账户聚合
```

如果你把这些都塞进一个巨大聚合，会变得很重。

更常见做法是：

```text
Order 创建成功
发布 OrderCreatedEvent
库存服务扣减库存
优惠券服务锁定优惠券
支付服务创建支付单
```

也就是说：

```text
Order 内部强一致
Order 与 Inventory / Coupon / Payment 最终一致
```

这和微服务里的事件驱动、消息队列、Saga 思想是相通的。

---

# 七、聚合不要设计得太大

初学 DDD 最容易犯的错：

> 把所有有关联的对象都放进同一个聚合。

比如：

```text
User
 ├── Orders
 │   ├── OrderItems
 │   └── Payments
 ├── Addresses
 ├── Coupons
 ├── Comments
 └── Favorites
```

这通常是灾难。

原因：

```text
1. 加载 User 时可能加载一大堆无关数据
2. 修改地址时可能锁住整个用户聚合
3. 并发冲突变多
4. Repository 复杂
5. 事务边界过大
6. 模型变得臃肿
```

DDD 聚合设计有一个常见原则：

> 聚合越小越好，但必须能保护真正的业务一致性。

---

# 八、如何判断哪些对象属于同一个聚合？

可以问几个问题。

## 1. 它们是否必须同时保持强一致？

如果必须强一致，倾向放同一个聚合。

例如：

```text
Order 和 OrderItem
```

订单提交时，订单项不能为空，订单总价要正确，这些规则需要强一致。

---

## 2. 子对象是否有独立生命周期？

如果子对象不能脱离父对象存在，倾向放同一个聚合。

例如：

```text
OrderItem 通常不能脱离 Order 独立存在
```

所以 `OrderItem` 适合作为 `Order` 聚合内部实体。

但：

```text
Product 可以脱离 Order 存在
User 可以脱离 Order 存在
Payment 也可能有自己的生命周期
```

它们更适合独立聚合。

---

## 3. 外部是否需要单独查询、单独修改它？

如果需要被外部独立操作，可能应该是单独聚合根。

例如评论：

```text
Article
 ├── Comment
```

到底 `Comment` 是否属于 `Article` 聚合？

不一定。

如果评论很简单，只能跟随文章一起管理，也可以放在 Article 聚合内。

但实际博客/社区系统里，评论通常有自己的：

```text
评论 ID
审核状态
点赞数
删除状态
举报流程
楼中楼回复
独立分页查询
```

这种情况下，`Comment` 更适合做独立聚合根。

它通过 `ArticleId` 关联文章，而不是作为 `Article` 聚合内部对象。

---

## 4. 并发修改会不会互相影响？

如果两个对象经常被不同用户、不同流程并发修改，就不要强行放在一个大聚合里。

例如：

```text
Article 内容编辑
Comment 评论新增
Like 点赞
ViewCount 浏览量
```

这些如果都放在 Article 聚合里，会导致高并发冲突。

更合理的设计可能是：

```text
Article 聚合：管理文章内容和发布状态
Comment 聚合：管理评论
ArticleStats 聚合：管理浏览量、点赞数
Favorite 聚合：管理用户收藏关系
```

---

# 九、订单聚合示例

## 1. 聚合根 Order

```java
public class Order {

    private final OrderId id;
    private final UserId userId;
    private final List<OrderItem> items = new ArrayList<>();
    private OrderStatus status;
    private Money totalAmount;
    private ShippingAddress shippingAddress;

    public Order(OrderId id, UserId userId, ShippingAddress shippingAddress) {
        if (id == null) {
            throw new IllegalArgumentException("订单 ID 不能为空");
        }
        if (userId == null) {
            throw new IllegalArgumentException("用户 ID 不能为空");
        }
        if (shippingAddress == null) {
            throw new IllegalArgumentException("收货地址不能为空");
        }

        this.id = id;
        this.userId = userId;
        this.shippingAddress = shippingAddress;
        this.status = OrderStatus.DRAFT;
        this.totalAmount = Money.zero();
    }

    public void addItem(
            ProductId productId,
            ProductName productName,
            Money unitPrice,
            int quantity
    ) {
        ensureDraft();

        OrderItem item = OrderItem.create(
                OrderItemId.newId(),
                productId,
                productName,
                unitPrice,
                quantity
        );

        this.items.add(item);
        recalculateTotalAmount();
    }

    public void changeItemQuantity(OrderItemId itemId, int quantity) {
        ensureDraft();

        OrderItem item = findItem(itemId);
        item.changeQuantity(quantity);

        recalculateTotalAmount();
    }

    public void removeItem(OrderItemId itemId) {
        ensureDraft();

        boolean removed = this.items.removeIf(item -> item.getId().equals(itemId));

        if (!removed) {
            throw new IllegalArgumentException("订单项不存在");
        }

        recalculateTotalAmount();
    }

    public void submit() {
        ensureDraft();

        if (items.isEmpty()) {
            throw new IllegalStateException("空订单不能提交");
        }

        this.status = OrderStatus.SUBMITTED;
    }

    public void pay() {
        if (status != OrderStatus.SUBMITTED) {
            throw new IllegalStateException("只有已提交订单才能支付");
        }

        this.status = OrderStatus.PAID;
    }

    public void changeShippingAddress(ShippingAddress newAddress) {
        ensureDraft();

        if (newAddress == null) {
            throw new IllegalArgumentException("收货地址不能为空");
        }

        this.shippingAddress = newAddress;
    }

    private void ensureDraft() {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("只有草稿订单可以修改");
        }
    }

    private OrderItem findItem(OrderItemId itemId) {
        return items.stream()
                .filter(item -> item.getId().equals(itemId))
                .findFirst()
                .orElseThrow(() -> new IllegalArgumentException("订单项不存在"));
    }

    private void recalculateTotalAmount() {
        this.totalAmount = items.stream()
                .map(OrderItem::subtotal)
                .reduce(Money.zero(), Money::add);
    }

    public OrderId getId() {
        return id;
    }

    public Money getTotalAmount() {
        return totalAmount;
    }

    public OrderStatus getStatus() {
        return status;
    }

    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
}
```

这里有几个关键点：

```text
1. items 不允许外部直接修改
2. 添加、删除、改数量都必须经过 Order
3. totalAmount 由 Order 内部维护
4. status 的变更通过 submit()、pay() 表达
5. Order 保护整个聚合的一致性
```

---

## 2. 聚合内实体 OrderItem

```java
public class OrderItem {

    private final OrderItemId id;
    private final ProductId productId;
    private final ProductName productName;
    private final Money unitPrice;
    private int quantity;

    private OrderItem(
            OrderItemId id,
            ProductId productId,
            ProductName productName,
            Money unitPrice,
            int quantity
    ) {
        this.id = id;
        this.productId = productId;
        this.productName = productName;
        this.unitPrice = unitPrice;
        changeQuantity(quantity);
    }

    public static OrderItem create(
            OrderItemId id,
            ProductId productId,
            ProductName productName,
            Money unitPrice,
            int quantity
    ) {
        return new OrderItem(id, productId, productName, unitPrice, quantity);
    }

    public void changeQuantity(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("商品数量必须大于 0");
        }

        this.quantity = quantity;
    }

    public Money subtotal() {
        return unitPrice.multiply(quantity);
    }

    public OrderItemId getId() {
        return id;
    }
}
```

`OrderItem` 是实体，因为它有自己的 `OrderItemId`，但它不是聚合根。

外部不应该有 `OrderItemRepository`。

---

## 3. 值对象 Money

```java
public record Money(BigDecimal amount) {

    public Money {
        if (amount == null) {
            throw new IllegalArgumentException("金额不能为空");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("金额不能为负数");
        }
    }

    public static Money zero() {
        return new Money(BigDecimal.ZERO);
    }

    public Money add(Money other) {
        return new Money(this.amount.add(other.amount));
    }

    public Money multiply(int multiplier) {
        if (multiplier < 0) {
            throw new IllegalArgumentException("乘数不能为负数");
        }
        return new Money(this.amount.multiply(BigDecimal.valueOf(multiplier)));
    }
}
```

---

## 4. Repository 只保存 Order

```java
public interface OrderRepository {

    Optional<Order> findById(OrderId orderId);

    void save(Order order);
}
```

应用服务：

```java
@Service
public class OrderApplicationService {

    private final OrderRepository orderRepository;

    @Transactional
    public void addItem(AddOrderItemCommand command) {
        Order order = orderRepository.findById(new OrderId(command.orderId()))
                .orElseThrow(() -> new IllegalArgumentException("订单不存在"));

        order.addItem(
                new ProductId(command.productId()),
                new ProductName(command.productName()),
                new Money(command.unitPrice()),
                command.quantity()
        );

        orderRepository.save(order);
    }
}
```

这就是典型的聚合使用方式。

---

# 十、结合文章 / DevWiki 场景理解聚合

你的 DevWiki / Blog 系统里，可以考虑这些聚合。

## 1. Article 聚合

```text
Article 聚合
 ├── Article：聚合根
 ├── ArticleTitle：值对象
 ├── ArticleContent：值对象
 ├── ArticleStatus：枚举
 ├── AuthorId：值对象
 └── TagId 列表：值对象 ID
```

职责：

```text
创建文章
修改标题
修改正文
发布文章
归档文章
删除文章
校验文章内容是否允许发布
```

不建议把评论、浏览量、点赞都塞进 Article 聚合。

因为这些东西并发高、生命周期独立、修改频繁。

---

## 2. Comment 聚合

```text
Comment 聚合
 ├── Comment：聚合根
 ├── ArticleId：值对象
 ├── UserId：值对象
 ├── CommentContent：值对象
 └── CommentStatus：枚举
```

职责：

```text
发表评论
审核评论
删除评论
隐藏评论
恢复评论
```

它通过 `ArticleId` 指向文章。

---

## 3. KnowledgeSource 聚合

如果你做 DevWiki，可能有“知识源 Source”：

```text
KnowledgeSource 聚合
 ├── KnowledgeSource：聚合根
 ├── SourceName：值对象
 ├── SourceType：枚举
 ├── SourceUri：值对象
 └── SourceStatus：枚举
```

职责：

```text
注册知识源
变更知识源名称
启用/停用知识源
标记解析状态
```

---

## 4. Chunk 聚合是否独立？

`Chunk` 是不是聚合根，要看你的业务。

如果 Chunk 只是 Source 解析后的内部数据，不会独立操作，可以作为 Source 聚合的一部分。

但如果 Chunk 会被：

```text
独立检索
独立更新 embedding
独立打标签
独立关联 WikiDraft
独立记录溯源证据
```

那它可能更适合作为独立聚合根：

```text
Chunk 聚合
 ├── Chunk：聚合根
 ├── SourceId：值对象
 ├── ChunkText：值对象
 ├── EmbeddingStatus：枚举
 └── Metadata：值对象
```

这就是聚合设计的本质：不是看“数据库外键”，而是看“业务一致性和生命周期”。

---

# 十一、聚合之间怎么协作？

聚合之间不要直接互相改内部状态。

常见方式有三种。

---

## 1. Application Service 编排

适合简单场景。

```java
@Transactional
public void publishArticle(Long articleId) {
    Article article = articleRepository.findById(new ArticleId(articleId))
            .orElseThrow();

    article.publish();

    articleRepository.save(article);
}
```

如果只涉及一个聚合，直接调用就好。

---

## 2. 领域服务 Domain Service

当一个业务规则不自然属于某一个实体或聚合时，用领域服务。

例如：

```text
判断用户名是否唯一
计算跨多个聚合的定价规则
校验某用户是否有发布文章权限
```

示例：

```java
public class ArticlePublishPolicy {

    private final UserRoleChecker userRoleChecker;

    public void checkCanPublish(AuthorId authorId) {
        if (!userRoleChecker.canPublish(authorId)) {
            throw new IllegalStateException("当前用户没有发布权限");
        }
    }
}
```

注意：领域服务仍然表达领域规则，不应该变成事务脚本垃圾桶。

---

## 3. 领域事件 Domain Event

适合跨聚合、跨上下文、跨服务的后续动作。

例如文章发布后：

```text
ArticlePublishedEvent
 ├── 更新搜索索引
 ├── 生成站点地图
 ├── 通知订阅者
 ├── 刷新推荐流
 └── 记录审计日志
```

`Article` 聚合只负责发布文章，不应该直接调用搜索引擎、邮件、MQ。

示例：

```java
public void publish() {
    if (status != ArticleStatus.DRAFT) {
        throw new IllegalStateException("只有草稿文章才能发布");
    }

    if (content.isBlank()) {
        throw new IllegalStateException("空文章不能发布");
    }

    this.status = ArticleStatus.PUBLISHED;
    this.publishedAt = LocalDateTime.now();

    this.addDomainEvent(new ArticlePublishedEvent(this.id, this.publishedAt));
}
```

应用服务保存后再发布事件：

```java
@Transactional
public void publishArticle(Long articleId) {
    Article article = articleRepository.findById(new ArticleId(articleId))
            .orElseThrow();

    article.publish();

    articleRepository.save(article);

    domainEventPublisher.publishAll(article.pullDomainEvents());
}
```

---

# 十二、聚合和数据库设计的关系

聚合不是数据库表。

一个聚合可以对应：

```text
一张表
多张表
JSON 字段
文档数据库中的一个文档
多张表 + 关联表
```

例如订单聚合：

```text
order 表
order_item 表
```

在领域层看是一个 `Order` 聚合。

在数据库层可能是两张表。

---

## 领域模型

```text
Order
 └── List<OrderItem>
```

## 数据库模型

```text
orders
order_items
```

## Repository 负责转换

```text
OrderPO + OrderItemPO -> Order 聚合
Order 聚合 -> OrderPO + OrderItemPO
```

Repository 的职责就是屏蔽这些持久化细节。

领域层不应该关心：

```text
表名
字段名
MyBatis 注解
JPA 注解
SQL Join
```

---

# 十三、聚合设计的常见错误

## 错误 1：聚合过大

比如：

```text
User 聚合
 ├── UserProfile
 ├── Orders
 ├── Comments
 ├── Favorites
 ├── Coupons
 └── LoginLogs
```

这不是聚合，是“对象数据库”。

后果：

```text
加载重
并发冲突多
边界模糊
事务过大
维护困难
```

---

## 错误 2：聚合过小，保护不了一致性

比如把 `Order` 和 `OrderItem` 分成两个完全独立聚合：

```text
Order 聚合
OrderItem 聚合
```

然后外部随便修改订单项。

后果：

```text
订单总价不准
订单状态规则被绕过
已支付订单还可以修改商品
```

如果 `OrderItem` 的生命周期完全依赖 `Order`，通常应该放进 `Order` 聚合。

---

## 错误 3：聚合之间直接对象引用

错误：

```java
public class Order {
    private User user;
    private Product product;
}
```

更好：

```java
public class Order {
    private UserId userId;
    private List<OrderItem> items;
}
```

聚合之间用 ID，减少耦合。

---

## 错误 4：给聚合内部实体建 Repository

错误：

```java
OrderItemRepository
```

如果 `OrderItem` 不是聚合根，就不应该被外部单独持久化。

---

## 错误 5：为了查询方便破坏聚合边界

比如列表页需要查：

```text
订单号
用户昵称
订单金额
订单状态
商品数量
```

于是你想把 User、Order、Product 都塞进一个聚合。

这不对。

查询不是聚合设计的核心驱动力。

在 DDD 里，常见做法是：

```text
写模型：使用聚合保护业务一致性
读模型：使用 Query / DTO / SQL Join / CQRS 直接查展示数据
```

也就是说：

```text
命令侧按领域模型设计
查询侧按页面展示设计
```

---

# 十四、聚合和 CQRS 的关系

聚合更适合处理写操作：

```text
创建订单
添加商品
提交订单
支付订单
取消订单
```

这些是命令 Command，需要业务规则和一致性保护。

查询操作可以不用走完整聚合。

例如订单列表页：

```java
public record OrderListResponse(
        Long orderId,
        String username,
        BigDecimal totalAmount,
        String status,
        LocalDateTime createdAt
) {
}
```

可以直接 SQL Join：

```sql
SELECT
    o.id AS order_id,
    u.nickname AS username,
    o.total_amount,
    o.status,
    o.created_at
FROM orders o
JOIN users u ON o.user_id = u.id
ORDER BY o.created_at DESC
```

这不违反 DDD。

因为：

```text
查询不修改业务状态
不需要加载完整聚合
不需要触发领域行为
```

---

# 十五、聚合设计的实用步骤

## 第一步：列出业务行为

不要先画表。

先列用例：

```text
创建文章
修改标题
修改正文
发布文章
归档文章
删除文章
发表评论
审核评论
点赞文章
收藏文章
生成 Wiki 草稿
解析知识源
更新 Chunk embedding
```

---

## 第二步：找一致性规则

例如：

```text
空文章不能发布
已发布文章不能回到草稿
已删除文章不能修改
评论必须绑定文章
未审核评论不展示
同一用户不能重复点赞同一文章
Chunk 必须属于一个 Source
生成 WikiDraft 必须引用至少一个 Chunk
```

这些规则会提示你聚合边界。

---

## 第三步：找生命周期归属

问：

```text
这个对象能否脱离父对象独立存在？
它是否需要独立查询？
它是否需要独立修改？
它是否有独立状态流转？
```

---

## 第四步：确定聚合根

一般选：

```text
最能代表这个业务边界的实体
外部最常访问的实体
能保护内部一致性的实体
```

比如：

```text
Order 聚合根：Order
Article 聚合根：Article
Comment 聚合根：Comment
KnowledgeSource 聚合根：KnowledgeSource
WikiDraft 聚合根：WikiDraft
```

---

## 第五步：规定访问规则

明确：

```text
哪些对象只能通过聚合根修改？
哪些对象可以独立 Repository？
哪些对象只能通过 ID 引用？
哪些对象跨聚合用事件同步？
```

这一步非常重要，否则代码会慢慢退化成普通 CRUD。

---

# 十六、在 Java 项目里的包结构

可以这样组织：

```text
article
 ├── domain
 │   ├── model
 │   │   ├── Article.java
 │   │   ├── ArticleId.java
 │   │   ├── ArticleTitle.java
 │   │   ├── ArticleContent.java
 │   │   └── ArticleStatus.java
 │   ├── repository
 │   │   └── ArticleRepository.java
 │   └── event
 │       └── ArticlePublishedEvent.java
 │
 ├── application
 │   ├── command
 │   │   ├── CreateArticleCommand.java
 │   │   └── PublishArticleCommand.java
 │   └── ArticleApplicationService.java
 │
 ├── interfaces
 │   ├── controller
 │   ├── request
 │   └── response
 │
 └── infrastructure
     ├── persistence
     │   ├── ArticlePO.java
     │   ├── ArticleMapper.java
     │   └── ArticleRepositoryImpl.java
     └── converter
         └── ArticlePersistenceConverter.java
```

如果 `Article` 这个聚合只有一个实体和几个值对象，直接放在 `domain.model` 下即可。

不一定非要有 `aggregate` 包。

---

# 十七、聚合根代码应该长什么样？

聚合根通常有这些特点：

```text
1. 字段私有
2. 不暴露随意 setter
3. 暴露业务方法
4. 保护内部集合
5. 维护不变量
6. 记录领域事件
7. 不直接依赖数据库、HTTP、MQ
```

反例：

```java
public class Order {
    public List<OrderItem> items;

    public void setStatus(OrderStatus status) {
        this.status = status;
    }
}
```

正例：

```java
public class Order {
    private final List<OrderItem> items = new ArrayList<>();
    private OrderStatus status;

    public void submit() {
        if (items.isEmpty()) {
            throw new IllegalStateException("空订单不能提交");
        }

        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("只有草稿订单可以提交");
        }

        this.status = OrderStatus.SUBMITTED;
    }

    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
}
```

---

# 十八、聚合和事务的关系

在单体 Java 应用里，聚合通常对应事务修改边界。

```java
@Transactional
public void changeOrderAddress(ChangeAddressCommand command) {
    Order order = orderRepository.findById(new OrderId(command.orderId()))
            .orElseThrow();

    order.changeShippingAddress(new ShippingAddress(
            command.province(),
            command.city(),
            command.detail()
    ));

    orderRepository.save(order);
}
```

事务内修改一个聚合，最简单、最稳定。

如果一个用例必须修改多个聚合，例如：

```text
创建订单
扣减库存
锁定优惠券
创建支付单
```

可以考虑：

```text
1. 先完成核心聚合状态变化
2. 发送领域事件
3. 其他聚合异步响应
4. 必要时用 Saga / 补偿事务
```

不要一开始就用一个大事务包住全世界。

---

# 十九、聚合边界的判断例子

## 例子 1：文章和评论是不是一个聚合？

多数情况下，不建议。

```text
Article 聚合：
  管理文章标题、正文、状态、发布流程

Comment 聚合：
  管理评论内容、审核状态、删除状态
```

原因：

```text
评论数量可能很多
评论需要分页
评论频繁新增
评论有独立审核生命周期
评论修改不应该锁住文章
```

所以：

```java
public class Comment {
    private CommentId id;
    private ArticleId articleId;
    private UserId userId;
    private CommentContent content;
    private CommentStatus status;
}
```

通过 `ArticleId` 关联文章。

---

## 例子 2：订单和订单项是不是一个聚合？

多数情况下，是。

```text
Order 聚合：
  Order + OrderItem
```

原因：

```text
订单项依赖订单存在
订单总价依赖订单项
订单状态决定订单项能否修改
提交订单时必须检查订单项
```

---

## 例子 3：用户和订单是不是一个聚合？

通常不是。

```text
User 聚合
Order 聚合
```

原因：

```text
用户和订单都有独立生命周期
订单数量可能很多
用户资料修改不应该影响订单
订单修改不应该锁用户
```

`Order` 持有 `UserId` 即可。

---

## 例子 4：知识源 Source 和 Chunk 是不是一个聚合？

看业务强度。

如果 Chunk 只是 Source 解析结果，数量少、只能随 Source 一起改，可以放在 Source 聚合内。

但实际 AI 知识库里，Chunk 往往会：

```text
数量很多
独立检索
独立 embedding
独立关联证据
独立召回
独立更新状态
```

所以更可能是独立聚合：

```text
KnowledgeSource 聚合
Chunk 聚合
EmbeddingJob 聚合
WikiDraft 聚合
```

它们之间用 ID 和事件关联。

---

# 二十、聚合设计的最终判断公式

你可以用这个公式记：

```text
聚合 = 一致性边界
聚合根 = 修改入口
Repository = 聚合根的持久化入口
事务 = 通常围绕一个聚合
聚合之间 = 用 ID / 事件 / 应用服务协作
```

再压缩成一句：

> **凡是必须一起保持强一致的对象，放进一个聚合；凡是可以最终一致、生命周期独立、并发修改频繁的对象，拆成不同聚合。**

---

# 二十一、和 MVC CRUD 的直观区别

传统 MVC CRUD 容易这样写：

```java
orderItemService.updateQuantity(orderItemId, quantity);
orderService.recalculateTotal(orderId);
```

问题是：

```text
调用顺序靠人记
漏调用就产生脏数据
业务规则散落在多个 Service
订单状态可能被绕过
```

DDD 聚合写法：

```java
Order order = orderRepository.findById(orderId).orElseThrow();

order.changeItemQuantity(orderItemId, quantity);

orderRepository.save(order);
```

好处：

```text
修改入口统一
规则集中在聚合根
内部一致性由模型保证
Service 只负责编排
```

这就是聚合真正的价值。