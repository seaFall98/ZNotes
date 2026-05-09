**充血模型**，指将对象的属性信息与行为逻辑聚合到一个类中，常用的手段如：在对象内提供属于当前对象的`信息校验`、`拼装缓存Key`、`不含服务接口调用的逻辑处理`等。
![[Pasted image 20260509232427.png]]
- 这样的方式可以在使用一个对象时，就顺便拿到这个对象的提供的一系列方法信息，所有使用对象的逻辑方法，都不需要自己再次处理同类逻辑。
- 但不要只是把充血模型，仅限于一个类的设计和一个类内的方法设计。充血还可以是整个包结构，一个包下包括了用于实现此包 Service 服务所需的各类零部件（模型、仓储、工厂），也可以被看做充血模型。
- 同时我们还会再一个同类的类下，提供对应的内部类，如用户实名，包括了，通信类、实名卡、银行卡、四要素等。它们都被写进到一个用户类下的内部子类，这样在代码编写中也会清晰的看到子类的所属信息，更容易理解代码逻辑，也便于维护迭代。



# DDD 充血模型

DDD 里的**充血模型**，本质上是在强调：

> **领域对象不只是数据容器，而应该同时包含数据、行为和业务规则。**

它对应的反面是 Java 后端里非常常见的**贫血模型**。

---

# 一、先说结论

在 DDD 中，推荐的模型不是：

```java
article.setStatus(ArticleStatus.PUBLISHED);
articleMapper.updateById(article);
```

而是：

```java
article.publish();
articleRepository.save(article);
```

区别在于：

|模型|数据在哪里|业务逻辑在哪里|特点|
|---|---|---|---|
|贫血模型|Entity / POJO|Service|对象只是字段壳，业务逻辑集中在 Service|
|充血模型|Entity / Value Object|Entity / Aggregate / Domain Service|对象自己维护业务规则和合法状态|

DDD 更倾向于充血模型。

---

# 二、什么是贫血模型？

贫血模型就是：

> 对象只有字段、getter、setter，几乎没有业务行为。

比如一个文章实体：

```java
public class Article {

    private Long id;
    private String title;
    private String content;
    private Integer status;

    public Long getId() {
        return id;
    }

    public Integer getStatus() {
        return status;
    }

    public void setStatus(Integer status) {
        this.status = status;
    }

    public String getContent() {
        return content;
    }
}
```

然后业务逻辑全部写在 Service：

```java
@Service
public class ArticleService {

    @Transactional
    public void publish(Long articleId) {
        Article article = articleMapper.selectById(articleId);

        if (article == null) {
            throw new IllegalArgumentException("文章不存在");
        }

        if (!Objects.equals(article.getStatus(), 0)) {
            throw new IllegalStateException("只有草稿文章才能发布");
        }

        if (article.getContent() == null || article.getContent().isBlank()) {
            throw new IllegalStateException("空文章不能发布");
        }

        article.setStatus(1);

        articleMapper.updateById(article);
    }
}
```

这就是典型贫血模型。

它能跑，但 DDD 视角下问题很多。

---

# 三、贫血模型的问题

## 1. 业务规则散落在 Service

比如发布文章的规则：

```text
只有草稿文章才能发布
空文章不能发布
已删除文章不能发布
发布后记录发布时间
发布后产生领域事件
```

如果这些逻辑都写在 Service，后面可能出现：

```java
article.setStatus(ArticleStatus.PUBLISHED);
```

某个地方绕过了发布规则，直接改了状态。

---

## 2. Entity 不能保护自己的合法状态

贫血模型通常会暴露大量 setter：

```java
article.setStatus(ArticleStatus.PUBLISHED);
article.setPublishedAt(null);
article.setTitle("");
```

这会导致对象可以进入非法状态。

比如：

```text
status = PUBLISHED
publishedAt = null
content = ""
title = ""
```

从数据库看有数据，但从业务上看是脏对象。

---

## 3. Service 越写越胖

一开始 Service 只是简单 CRUD：

```java
createArticle()
updateArticle()
deleteArticle()
```

后面变成：

```java
publishArticle()
archiveArticle()
restoreArticle()
checkCanPublish()
validateArticleContent()
calculateArticleScore()
syncArticleStatus()
```

最后 Service 变成“上帝类”。

这种代码通常叫：

```text
Transaction Script：事务脚本
```

它不是不能用，但复杂业务下会退化得很快。

---

## 4. 业务语义不清楚

贫血模型里，代码通常是：

```java
article.setStatus(1);
```

充血模型里是：

```java
article.publish();
```

后者语义明显更强。

你不需要去猜 `1` 是什么，也不需要去找哪个 Service 里隐藏了发布规则。

---

# 四、什么是充血模型？

充血模型就是：

> 领域对象自己拥有业务行为，并维护自己的不变量。

例如：

```java
public class Article {

    private ArticleId id;
    private ArticleTitle title;
    private ArticleContent content;
    private ArticleStatus status;
    private LocalDateTime publishedAt;

    public void publish() {
        if (status != ArticleStatus.DRAFT) {
            throw new IllegalStateException("只有草稿文章才能发布");
        }

        if (content.isBlank()) {
            throw new IllegalStateException("空文章不能发布");
        }

        this.status = ArticleStatus.PUBLISHED;
        this.publishedAt = LocalDateTime.now();
    }

    public void archive() {
        if (status != ArticleStatus.PUBLISHED) {
            throw new IllegalStateException("只有已发布文章才能归档");
        }

        this.status = ArticleStatus.ARCHIVED;
    }

    public void rename(ArticleTitle newTitle) {
        if (status == ArticleStatus.DELETED) {
            throw new IllegalStateException("已删除文章不能修改标题");
        }

        this.title = newTitle;
    }
}
```

这里 `Article` 不只是数据结构，而是一个有行为的领域对象。

它知道：

```text
自己什么时候能发布
自己什么时候能归档
自己什么时候能改标题
发布后状态怎么变化
哪些状态是非法的
```

---

# 五、充血模型的核心思想

## 1. 数据和行为放在一起

传统贫血模型：

```text
Article：保存数据
ArticleService：处理行为
```

DDD 充血模型：

```text
Article：保存数据 + 执行业为 + 保护规则
ArticleApplicationService：编排流程
```

例如：

```java
@Transactional
public void publishArticle(Long articleId) {
    Article article = articleRepository.findById(new ArticleId(articleId))
            .orElseThrow(() -> new IllegalArgumentException("文章不存在"));

    article.publish();

    articleRepository.save(article);
}
```

`ApplicationService` 不负责判断文章能不能发布。

它只负责：

```text
加载聚合
调用业务行为
保存聚合
发布事件
处理事务
```

真正的规则在：

```java
article.publish();
```

---

## 2. 用业务方法替代 setter

贫血模型：

```java
article.setStatus(ArticleStatus.PUBLISHED);
article.setPublishedAt(LocalDateTime.now());
```

充血模型：

```java
article.publish();
```

贫血模型：

```java
order.setStatus(OrderStatus.CANCELED);
```

充血模型：

```java
order.cancel(reason);
```

贫血模型：

```java
account.setBalance(account.getBalance().subtract(amount));
```

充血模型：

```java
account.withdraw(amount);
```

业务方法表达业务意图。

---

## 3. 实体负责维护不变量

所谓不变量，就是任何时候都必须成立的规则。

例如订单聚合：

```text
订单项数量必须大于 0
订单总价必须等于订单项小计之和
已支付订单不能修改商品
空订单不能提交
```

充血模型里，这些规则由聚合根保护：

```java
public class Order {

    private final List<OrderItem> items = new ArrayList<>();
    private OrderStatus status;
    private Money totalAmount;

    public void addItem(ProductId productId, ProductName productName, Money price, int quantity) {
        ensureDraft();

        OrderItem item = OrderItem.create(productId, productName, price, quantity);
        this.items.add(item);

        recalculateTotalAmount();
    }

    public void submit() {
        ensureDraft();

        if (items.isEmpty()) {
            throw new IllegalStateException("空订单不能提交");
        }

        this.status = OrderStatus.SUBMITTED;
    }

    private void ensureDraft() {
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("只有草稿订单可以修改");
        }
    }

    private void recalculateTotalAmount() {
        this.totalAmount = items.stream()
                .map(OrderItem::subtotal)
                .reduce(Money.zero(), Money::add);
    }
}
```

外部代码不能直接改 `items`，也不能直接改 `totalAmount`。

---

# 六、充血模型不是“把所有代码都塞进 Entity”

这是一个关键误区。

DDD 充血模型不是说：

> 所有业务代码都必须写到实体里。

实体适合放：

```text
与自身状态强相关的规则
自身生命周期行为
自身不变量维护
聚合内部一致性逻辑
```

不适合放：

```text
数据库查询
HTTP 调用
MQ 发送
Redis 操作
权限系统调用
跨聚合复杂协调
报表查询
页面组装
```

---

# 七、哪些逻辑应该放在哪里？

## 1. 放 Entity / Aggregate Root

适合：

```text
文章发布
订单提交
账户冻结
钱包扣款
商品改价
评论审核
任务完成
```

示例：

```java
article.publish();
order.submit();
account.freeze();
wallet.withdraw(amount);
comment.approve();
```

这些行为和对象自身状态强相关。

---

## 2. 放 Value Object

适合：

```text
金额计算
邮箱校验
标题长度校验
地址完整性校验
时间范围重叠判断
```

示例：

```java
public record Money(BigDecimal amount, Currency currency) {

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("不同币种不能相加");
        }

        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

Money 自己知道怎么加钱。

Email 自己知道邮箱是否合法。

ArticleTitle 自己知道标题是否为空、是否超长。

---

## 3. 放 Domain Service

当一个规则不自然属于某一个实体时，放领域服务。

例如：

```text
判断用户名是否唯一
计算复杂定价
跨多个账户转账
判断用户是否有发布资格
```

示例：

```java
public class TransferService {

    public void transfer(Account from, Account to, Money amount) {
        if (from.sameAs(to)) {
            throw new IllegalArgumentException("不能给自己转账");
        }

        from.withdraw(amount);
        to.deposit(amount);
    }
}
```

这里“转账”涉及两个账户，不完全属于某一个账户，所以可以用领域服务。

---

## 4. 放 Application Service

Application Service 负责用例编排，不负责核心业务规则。

```java
@Service
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public void submitOrder(SubmitOrderCommand command) {
        Order order = orderRepository.findById(new OrderId(command.orderId()))
                .orElseThrow(() -> new IllegalArgumentException("订单不存在"));

        order.submit();

        orderRepository.save(order);

        eventPublisher.publishAll(order.pullDomainEvents());
    }
}
```

它负责：

```text
事务
权限上下文
加载聚合
调用领域行为
保存聚合
发布领域事件
调用外部服务适配器
```

但它不应该负责：

```text
空订单能不能提交
已支付订单能不能取消
金额怎么算
状态怎么流转
```

这些应该进领域模型。

---

## 5. 放 Infrastructure

基础设施层负责技术细节：

```text
MyBatis / JPA
Redis
MQ
HTTP Client
文件存储
向量数据库
搜索引擎
邮件服务
```

领域模型不要直接依赖它们。

错误：

```java
public class Article {

    private SearchClient searchClient;

    public void publish() {
        this.status = ArticleStatus.PUBLISHED;
        searchClient.index(this);
    }
}
```

更好：

```java
public void publish() {
    this.status = ArticleStatus.PUBLISHED;
    this.addDomainEvent(new ArticlePublishedEvent(this.id));
}
```

然后由事件处理器去同步搜索引擎。

---

# 八、一个完整例子：文章发布

## 1. 贫血模型写法

```java
@Service
public class ArticleService {

    @Transactional
    public void publish(Long articleId) {
        ArticlePO article = articleMapper.selectById(articleId);

        if (article == null) {
            throw new IllegalArgumentException("文章不存在");
        }

        if (!Objects.equals(article.getStatus(), 0)) {
            throw new IllegalStateException("只有草稿文章才能发布");
        }

        if (article.getContent() == null || article.getContent().isBlank()) {
            throw new IllegalStateException("空文章不能发布");
        }

        article.setStatus(1);
        article.setPublishedAt(LocalDateTime.now());

        articleMapper.updateById(article);
    }
}
```

问题：

```text
ArticlePO 是数据库对象
业务规则在 Service
status 用 0 / 1 表示
其他代码仍然可以绕过 publish 直接 setStatus
```

---

## 2. DDD 充血模型写法

### 领域实体

```java
public class Article {

    private ArticleId id;
    private ArticleTitle title;
    private ArticleContent content;
    private ArticleStatus status;
    private LocalDateTime publishedAt;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    public Article(
            ArticleId id,
            ArticleTitle title,
            ArticleContent content
    ) {
        this.id = Objects.requireNonNull(id);
        this.title = Objects.requireNonNull(title);
        this.content = Objects.requireNonNull(content);
        this.status = ArticleStatus.DRAFT;
    }

    public void publish() {
        if (status != ArticleStatus.DRAFT) {
            throw new IllegalStateException("只有草稿文章才能发布");
        }

        if (content.isBlank()) {
            throw new IllegalStateException("空文章不能发布");
        }

        this.status = ArticleStatus.PUBLISHED;
        this.publishedAt = LocalDateTime.now();

        this.domainEvents.add(new ArticlePublishedEvent(id, publishedAt));
    }

    public void reviseContent(ArticleContent newContent) {
        ensureNotDeleted();
        this.content = Objects.requireNonNull(newContent);
    }

    public void rename(ArticleTitle newTitle) {
        ensureNotDeleted();
        this.title = Objects.requireNonNull(newTitle);
    }

    private void ensureNotDeleted() {
        if (status == ArticleStatus.DELETED) {
            throw new IllegalStateException("已删除文章不能修改");
        }
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = List.copyOf(domainEvents);
        domainEvents.clear();
        return events;
    }

    public ArticleId getId() {
        return id;
    }
}
```

### Value Object

```java
public record ArticleTitle(String value) {

    public ArticleTitle {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("文章标题不能为空");
        }

        if (value.length() > 100) {
            throw new IllegalArgumentException("文章标题不能超过 100 个字符");
        }
    }
}
```

```java
public record ArticleContent(String value) {

    public ArticleContent {
        if (value == null) {
            throw new IllegalArgumentException("文章内容不能为 null");
        }
    }

    public boolean isBlank() {
        return value.isBlank();
    }
}
```

### Repository 接口

```java
public interface ArticleRepository {

    Optional<Article> findById(ArticleId articleId);

    void save(Article article);
}
```

### Application Service

```java
@Service
public class ArticleApplicationService {

    private final ArticleRepository articleRepository;
    private final DomainEventPublisher domainEventPublisher;

    public ArticleApplicationService(
            ArticleRepository articleRepository,
            DomainEventPublisher domainEventPublisher
    ) {
        this.articleRepository = articleRepository;
        this.domainEventPublisher = domainEventPublisher;
    }

    @Transactional
    public void publishArticle(Long articleId) {
        Article article = articleRepository.findById(new ArticleId(articleId))
                .orElseThrow(() -> new IllegalArgumentException("文章不存在"));

        article.publish();

        articleRepository.save(article);

        domainEventPublisher.publishAll(article.pullDomainEvents());
    }
}
```

核心变化：

```text
发布规则从 ArticleService 移到了 Article.publish()
Article 自己保护自己的合法状态
ApplicationService 只负责编排
Repository 隔离持久化
```

---

# 九、再举一个更典型的例子：账户扣款

## 贫血模型

```java
@Transactional
public void withdraw(Long accountId, BigDecimal amount) {
    AccountPO account = accountMapper.selectById(accountId);

    if (account.getStatus() != 1) {
        throw new IllegalStateException("账户不可用");
    }

    if (account.getBalance().compareTo(amount) < 0) {
        throw new IllegalStateException("余额不足");
    }

    account.setBalance(account.getBalance().subtract(amount));

    accountMapper.updateById(account);
}
```

问题是：扣款规则在 Service。

---

## 充血模型

```java
public class Account {

    private AccountId id;
    private Money balance;
    private AccountStatus status;

    public void withdraw(Money amount) {
        if (status != AccountStatus.ACTIVE) {
            throw new IllegalStateException("账户不可用");
        }

        if (balance.lessThan(amount)) {
            throw new IllegalStateException("余额不足");
        }

        this.balance = this.balance.subtract(amount);
    }

    public void deposit(Money amount) {
        if (status != AccountStatus.ACTIVE) {
            throw new IllegalStateException("账户不可用");
        }

        this.balance = this.balance.add(amount);
    }

    public void freeze() {
        if (status == AccountStatus.CLOSED) {
            throw new IllegalStateException("已销户账户不能冻结");
        }

        this.status = AccountStatus.FROZEN;
    }
}
```

Application Service：

```java
@Transactional
public void withdraw(WithdrawCommand command) {
    Account account = accountRepository.findById(new AccountId(command.accountId()))
            .orElseThrow(() -> new IllegalArgumentException("账户不存在"));

    account.withdraw(new Money(command.amount()));

    accountRepository.save(account);
}
```

这就是充血模型的优势：**账户自己知道如何扣款**。

---

# 十、充血模型和聚合的关系

在 DDD 中，充血模型通常落在：

```text
Entity
Value Object
Aggregate Root
Domain Service
```

尤其是聚合根。

聚合根负责保护整个聚合的一致性。

例如订单聚合：

```java
public class Order {

    private OrderId id;
    private List<OrderItem> items;
    private OrderStatus status;
    private Money totalAmount;

    public void addItem(ProductId productId, Money price, int quantity) {
        ensureDraft();

        OrderItem item = OrderItem.create(productId, price, quantity);
        items.add(item);

        recalculateTotalAmount();
    }

    public void submit() {
        ensureDraft();

        if (items.isEmpty()) {
            throw new IllegalStateException("空订单不能提交");
        }

        this.status = OrderStatus.SUBMITTED;
    }
}
```

`Order` 不是只保存订单数据，它还负责：

```text
添加订单项
校验订单状态
计算总价
提交订单
维护订单内部一致性
```

所以：

> **聚合根通常就是充血模型的主要承载点。**

---

# 十一、充血模型是否必须完全替代 Service？

不是。

DDD 不是不要 Service，而是要区分 Service 类型。

|类型|主要职责|是否放核心业务规则|
|---|---|---|
|Controller|接收 HTTP 请求|否|
|Application Service|用例编排、事务、调用领域对象|少量流程规则|
|Domain Service|放不属于单个实体的领域规则|是|
|Infrastructure Service|调外部系统、技术实现|否|

Application Service 仍然存在，但不应该变成业务规则垃圾桶。

正确分工：

```text
Application Service：
    我要完成“发布文章”这个用例，需要加载文章、调用 publish、保存、发事件。

Article：
    我什么时候能发布，发布后状态怎么变。

Repository：
    我怎么从数据库加载和保存 Article。

EventHandler：
    文章发布后怎么同步搜索索引、发通知。
```

---

# 十二、充血模型的代码特征

一个比较像 DDD 的充血模型，通常有这些特征：

```text
1. 字段 private
2. 没有随意 public setter
3. 构造器或工厂方法保证创建时合法
4. 业务方法使用领域语言命名
5. 方法内部维护状态流转
6. Value Object 封装基础校验和计算
7. 聚合根保护内部集合
8. 领域层不依赖 MyBatis、Redis、HTTP、MQ
```

例如：

```java
order.submit();
order.cancel(reason);
order.addItem(productId, price, quantity);

article.publish();
article.archive();
article.rename(title);

account.withdraw(amount);
account.freeze();

comment.approve();
comment.reject(reason);
```

这些都比：

```java
setStatus()
setType()
setDeleted()
setAmount()
```

更有业务语义。

---

# 十三、充血模型的常见误区

## 误区 1：把数据库 Entity 加几个方法就叫 DDD

例如：

```java
@TableName("article")
public class ArticleEntity {

    private Long id;
    private String title;
    private String content;
    private Integer status;

    public void publish() {
        this.status = 1;
    }
}
```

这不一定是好的 DDD。

问题：

```text
status 仍然是数据库编码
没有封装规则
仍然和 MyBatis 注解耦合
字段可能被外部 setter 绕过
```

更合理的是领域模型和持久化模型适度分离：

```text
domain.model.Article
infrastructure.persistence.ArticlePO
```

当然，小项目可以适当简化，但要知道自己在简化什么。

---

## 误区 2：Entity 里直接注入 Repository

不建议：

```java
public class Article {

    @Autowired
    private ArticleRepository articleRepository;

    public void publish() {
        // ...
        articleRepository.save(this);
    }
}
```

领域对象通常不应该自己保存自己。

保存是 Application Service / Repository 的职责：

```java
article.publish();
articleRepository.save(article);
```

---

## 误区 3：Entity 里调用外部服务

不建议：

```java
public class Article {

    private SearchIndexClient searchIndexClient;

    public void publish() {
        this.status = ArticleStatus.PUBLISHED;
        searchIndexClient.index(this);
    }
}
```

这会污染领域模型。

更好：

```java
article.publish(); // 产生 ArticlePublishedEvent
```

然后事件处理器：

```java
public void on(ArticlePublishedEvent event) {
    searchIndexClient.index(event.articleId());
}
```

---

## 误区 4：所有规则都塞进实体

有些规则天然跨多个对象。

比如：

```text
用户名是否唯一
两个账户之间转账
文章是否达到推荐标准
用户今天是否超过发布次数
```

这些可能不适合放单个实体里。

可以放：

```text
Domain Service
Specification
Policy
Application Service + Repository 查询
```

---

## 误区 5：为了充血而充血

有些系统就是简单 CRUD。

例如后台配置表：

```text
新增配置
修改配置
删除配置
查询配置
```

没有复杂状态流转，也没有强业务规则。

这种场景强行 DDD，收益很低。

可以直接 MVC + CRUD。

DDD 适合：

```text
业务规则复杂
状态流转多
一致性要求强
领域概念多
后续变化频繁
```

---

# 十四、充血模型和 Java 实战的冲突点

Java 后端长期受 MyBatis / JPA / Lombok 影响，常见写法是：

```java
@Data
@TableName("xxx")
public class XxxEntity {
    private Long id;
    private String name;
}
```

这天然偏贫血模型。

如果想往 DDD 靠，可以这样调整。

---

## 1. 少用 Lombok `@Data`

`@Data` 会生成所有 getter / setter。

领域对象里不建议随便用。

可以用：

```java
@Getter
```

然后手写业务方法。

```java
@Getter
public class Article {

    private ArticleId id;
    private ArticleTitle title;
    private ArticleContent content;
    private ArticleStatus status;

    public void publish() {
        // ...
    }
}
```

更严格一点：连 getter 也谨慎暴露。

---

## 2. 避免 public setter

不要这样：

```java
article.setStatus(PUBLISHED);
```

改成：

```java
article.publish();
```

不要这样：

```java
order.setShippingAddress(address);
```

改成：

```java
order.changeShippingAddress(address);
```

---

## 3. 使用 Value Object 替代裸类型

贫血模型常见：

```java
private String email;
private BigDecimal amount;
private String title;
```

充血模型更建议：

```java
private Email email;
private Money amount;
private ArticleTitle title;
```

这样很多校验和计算可以下沉到值对象。

---

## 4. Repository 返回领域对象，不返回 PO

DDD 中理想情况：

```java
Article article = articleRepository.findById(articleId);
```

而不是：

```java
ArticlePO articlePO = articleMapper.selectById(articleId);
```

`ArticleRepositoryImpl` 内部负责转换：

```text
ArticlePO -> Article
Article -> ArticlePO
```

---

# 十五、在你的 DevWiki / Blog 项目里怎么用？

你的项目里，如果有这些概念：

```text
Article
Source
Chunk
WikiDraft
Skill
Artifact
```

可以这样判断。

---

## 1. Article 适合充血模型

文章有明显状态流转：

```text
DRAFT -> PUBLISHED -> ARCHIVED -> DELETED
```

适合放行为：

```java
article.rename(title);
article.reviseContent(content);
article.publish();
article.archive();
article.delete();
```

---

## 2. Source 适合充血模型

知识源可能有状态：

```text
REGISTERED
PARSING
PARSED
FAILED
DISABLED
```

可以放行为：

```java
source.startParsing();
source.markParsed();
source.markFailed(reason);
source.disable();
source.rename(name);
```

---

## 3. Chunk 不一定需要复杂充血

如果 Chunk 只是文本切片和 embedding 状态，可以适度建模：

```java
chunk.markEmbedded(embeddingId);
chunk.markEmbeddingFailed(reason);
chunk.updateMetadata(metadata);
```

但如果只是纯检索数据，也可以偏数据模型，不必强行复杂化。

---

## 4. WikiDraft 适合充血模型

WikiDraft 可能有生成、编辑、确认、发布状态：

```java
wikiDraft.generateFrom(chunks);
wikiDraft.reviseContent(content);
wikiDraft.approve();
wikiDraft.reject(reason);
wikiDraft.publish();
```

如果它有明确业务流，就值得做成充血模型。

---

# 十六、怎么从 MVC 贫血模型迁移到 DDD 充血模型？

不要一口气全改。

可以按这个顺序来。

## 第一步：识别核心业务对象

先别改所有 Entity。

只挑核心对象：

```text
Article
Order
Account
WikiDraft
KnowledgeSource
```

非核心配置表继续 CRUD。

---

## 第二步：把状态修改改成业务方法

原来：

```java
article.setStatus(PUBLISHED);
```

改成：

```java
article.publish();
```

---

## 第三步：把校验逻辑移进实体

原来 Service 里：

```java
if (article.getContent().isBlank()) {
    throw new IllegalStateException("空文章不能发布");
}
```

移到：

```java
article.publish();
```

---

## 第四步：用 Value Object 封装关键字段

原来：

```java
String title;
String email;
BigDecimal amount;
```

改成：

```java
ArticleTitle title;
Email email;
Money amount;
```

---

## 第五步：Repository 隔离持久化

领域层定义：

```java
public interface ArticleRepository {
    Optional<Article> findById(ArticleId id);
    void save(Article article);
}
```

基础设施层实现：

```java
@Repository
public class ArticleRepositoryImpl implements ArticleRepository {
    private final ArticleMapper articleMapper;
    private final ArticleConverter converter;

    public Optional<Article> findById(ArticleId id) {
        ArticlePO po = articleMapper.selectById(id.value());
        return Optional.ofNullable(po).map(converter::toDomain);
    }

    public void save(Article article) {
        articleMapper.updateById(converter.toPO(article));
    }
}
```

---

# 十七、贫血模型、充血模型、事务脚本怎么选择？

不是所有地方都必须 DDD。

|场景|推荐方式|
|---|---|
|简单后台 CRUD|MVC + 贫血模型可以接受|
|字典表、配置表|CRUD 即可|
|业务规则多|充血模型|
|状态流转复杂|充血模型|
|强一致性边界明显|聚合根充血模型|
|报表查询|DTO / Query Model / SQL|
|高并发计数|单独模型或基础设施优化，不一定放进聚合|

比较实际的策略是：

> **核心领域用充血模型，边缘 CRUD 不强行 DDD。**

---

# 十八、一段对比代码总结

## 贫血模型

```java
@Transactional
public void publishArticle(Long articleId) {
    ArticlePO article = articleMapper.selectById(articleId);

    if (article.getStatus() != 0) {
        throw new IllegalStateException("不能发布");
    }

    if (article.getContent() == null || article.getContent().isBlank()) {
        throw new IllegalStateException("内容为空");
    }

    article.setStatus(1);
    article.setPublishedAt(LocalDateTime.now());

    articleMapper.updateById(article);
}
```

特点：

```text
Service 负责全部业务规则
Entity 是数据壳
容易被其他代码绕过规则
```

---

## 充血模型

```java
@Transactional
public void publishArticle(Long articleId) {
    Article article = articleRepository.findById(new ArticleId(articleId))
            .orElseThrow(() -> new IllegalArgumentException("文章不存在"));

    article.publish();

    articleRepository.save(article);
}
```

领域对象：

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
}
```

特点：

```text
Service 编排流程
Entity 负责规则
业务语义清晰
对象能保护自己的合法状态
```

---

# 十九、最关键的一句话

DDD 充血模型不是“Entity 里多写几个方法”，而是：

> **让领域对象成为业务规则的真正承载者，让对象自己维护自己的合法状态，而不是让 Service 在外面随意操纵对象字段。**

你可以这样记：

```text
贫血模型：
    Service 问对象要数据，然后自己判断、自己修改。

充血模型：
    Service 告诉对象要做什么，对象自己判断能不能做，并完成状态变化。
```

例如：

```java
// 贫血模型
if (article.getStatus() == DRAFT) {
    article.setStatus(PUBLISHED);
}

// 充血模型
article.publish();
```

这就是 DDD 充血模型的核心。