## 1. 什么是循环依赖

Spring 里的循环依赖，本质是 **Bean 之间互相依赖，导致 Spring 创建 Bean 时形成闭环**。

最简单的例子：

```java
@Service
public class AService {
    private final BService bService;

    public AService(BService bService) {
        this.bService = bService;
    }
}

@Service
public class BService {
    private final AService aService;

    public BService(AService aService) {
        this.aService = aService;
    }
}
```

依赖关系是：

```text
AService -> BService -> AService
```

Spring 创建 `AService` 时发现需要 `BService`；  
创建 `BService` 时又发现需要 `AService`；  
于是陷入闭环。

Spring 官方文档也明确提到：如果主要使用 **构造器注入**，`A` 构造器依赖 `B`，`B` 构造器又依赖 `A`，这种场景是不可解析的，Spring 会抛出 `BeanCurrentlyInCreationException`。([Home](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html?utm_source=chatgpt.com "Dependency Injection :: Spring Framework"))

---

## 2. Spring 已经禁用了循环依赖

准确说：

> **Spring Boot 从 2.6 开始，默认禁止 Bean 循环引用。**

对应配置是：

```yaml
spring:
  main:
    allow-circular-references: false
```

Spring Boot 官方配置文档里，`spring.main.allow-circular-references` 的默认值就是 `false`，含义是是否允许 Bean 之间的循环引用并尝试自动解决。([Home](https://docs.spring.io/spring-boot/appendix/application-properties/index.html?utm_source=chatgpt.com "Common Application Properties :: Spring Boot"))

Spring Boot 2.6 Release Notes 也明确说明：Bean 之间的 circular references 从该版本开始默认被禁止，如果应用因为 `BeanCurrentlyInCreationException` 启动失败，官方建议是修改设计，打破依赖环。([GitHub](https://github.com/spring-projects/spring-boot/wiki/Spring-Boot-2.6-Release-Notes?utm_source=chatgpt.com "Spring Boot 2.6 Release Notes"))

所以你看到的报错通常类似：

```text
The dependencies of some of the beans in the application context form a cycle:

┌─────┐
|  aService
↑     ↓
|  bService
└─────┘

Relying upon circular references is discouraged and they are prohibited by default.
```

---

## 3. 以前为什么有时候能工作？

Spring Framework 以前对某些循环依赖是可以“兜底解决”的，尤其是：

```text
单例 Bean + setter 注入 / field 注入
```

例如：

```java
@Service
public class AService {
    @Autowired
    private BService bService;
}

@Service
public class BService {
    @Autowired
    private AService aService;
}
```

这种情况下，Spring 可以先把 `AService` 的“早期对象引用”暴露出去，再继续完成 `BService` 的注入，最后回填属性。

这依赖 Spring 的 **三级缓存机制**。

简化理解：

```text
一级缓存 singletonObjects
    已经完整创建好的 Bean

二级缓存 earlySingletonObjects
    提前暴露的半成品 Bean

三级缓存 singletonFactories
    用于创建早期 Bean 引用的工厂，尤其和 AOP 代理有关
```

一个简化流程：

```text
1. 创建 AService
2. AService 还没完全初始化，但 Spring 先暴露一个早期引用
3. AService 依赖 BService，于是创建 BService
4. BService 依赖 AService
5. Spring 从早期引用里拿到 AService
6. BService 创建完成
7. AService 注入 BService
8. AService 创建完成
```

所以以前很多项目即使有循环依赖，也能启动。

---

## 4. 那为什么现在默认禁用？

因为循环依赖虽然“能解决”，但通常说明代码设计有问题。

### 主要问题有 4 个

|问题|说明|
|---|---|
|职责边界混乱|A 调 B，B 又调 A，说明服务边界不清晰|
|初始化顺序复杂|Bean 的创建过程变得隐式、脆弱|
|AOP 代理容易出问题|早期暴露的可能是原始对象，也可能是代理对象，容易产生行为差异|
|构造器注入无法解决|构造器要求对象创建时依赖必须完整，而循环依赖天然无法满足|

尤其是现在 Spring 更推荐 **构造器注入**：

```java
private final XxxService xxxService;

public SomeService(XxxService xxxService) {
    this.xxxService = xxxService;
}
```

构造器注入的优点是依赖清晰、不可变、方便测试。  
但它会直接暴露循环依赖问题，不会像 field 注入那样“偷偷绕过去”。

---

## 5. 哪些循环依赖 Spring 能解决，哪些不能？

### 5.1 构造器注入：不能解决

```java
@Service
public class AService {
    public AService(BService bService) {}
}

@Service
public class BService {
    public BService(AService aService) {}
}
```

这是死结。

因为：

```text
创建 A 必须先有 B
创建 B 必须先有 A
```

没有任何一个对象能先被创建出来。

---

### 5.2 field 注入 / setter 注入：以前可能能解决

```java
@Service
public class AService {
    @Autowired
    private BService bService;
}

@Service
public class BService {
    @Autowired
    private AService aService;
}
```

这种以前可以通过提前暴露引用解决。

但 Spring Boot 2.6+ 默认也不鼓励了。

---

### 5.3 prototype Bean 循环依赖：不能解决

如果是多例 Bean：

```java
@Scope("prototype")
```

Spring 不能像单例那样缓存早期引用，所以通常解决不了。

---

### 5.4 AOP 场景：可能更复杂

例如：

```java
@Transactional
@Service
public class AService {
    @Autowired
    private BService bService;
}
```

如果循环依赖里涉及事务代理、异步代理、缓存代理，早期暴露对象可能涉及代理创建时机问题。

这也是 Spring Boot 默认禁用循环依赖的重要原因之一。

---

## 6. 临时解决：打开循环依赖开关

可以这样配置：

```yaml
spring:
  main:
    allow-circular-references: true
```

或者：

```properties
spring.main.allow-circular-references=true
```

但这只是 **临时兼容旧项目**，不建议作为长期方案。

它只能处理部分 setter / field 注入型循环依赖，不能处理构造器注入死循环。

---

## 7. 推荐修复方式

### 方式一：重新拆分职责，打破服务互调

坏设计：

```text
OrderService -> PayService
PayService   -> OrderService
```

更合理的方式：

```text
OrderService -> PayService
PayService   -> OrderStatusUpdater
```

例如：

```java
@Service
public class OrderStatusUpdater {

    public void markPaid(Long orderId) {
        // 修改订单状态
    }
}
```

然后：

```java
@Service
public class PayService {

    private final OrderStatusUpdater orderStatusUpdater;

    public PayService(OrderStatusUpdater orderStatusUpdater) {
        this.orderStatusUpdater = orderStatusUpdater;
    }

    public void pay(Long orderId) {
        // 支付逻辑
        orderStatusUpdater.markPaid(orderId);
    }
}
```

本质是把双方都需要的能力抽成第三方组件。

---

### 方式二：引入领域事件

适合业务流程解耦。

原来：

```text
UserService -> CouponService
CouponService -> UserService
```

可以改成：

```text
UserService 发布 UserRegisteredEvent
CouponService 监听事件发优惠券
```

示例：

```java
public record UserRegisteredEvent(Long userId) {}
```

```java
@Service
public class UserService {

    private final ApplicationEventPublisher publisher;

    public UserService(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }

    public void register(String username) {
        // 1. 创建用户
        Long userId = 100L;

        // 2. 发布事件
        publisher.publishEvent(new UserRegisteredEvent(userId));
    }
}
```

```java
@Component
public class CouponListener {

    @EventListener
    public void onUserRegistered(UserRegisteredEvent event) {
        // 给新用户发券
    }
}
```

这样 `UserService` 不需要直接依赖 `CouponService`。

---

### 方式三：用 `@Lazy` 延迟注入

适合短期修复，不是首选架构方案。

```java
@Service
public class AService {

    private final BService bService;

    public AService(@Lazy BService bService) {
        this.bService = bService;
    }
}
```

含义是：  
`AService` 创建时不立刻创建真实 `BService`，而是注入一个懒加载代理。

等真正调用 `bService` 方法时，再去容器里拿真实 Bean。

缺点：

```text
依赖关系仍然是循环的，只是把问题延后了。
```

---

### 方式四：用 `ObjectProvider`

比 `@Lazy` 更显式。

```java
@Service
public class AService {

    private final ObjectProvider<BService> bServiceProvider;

    public AService(ObjectProvider<BService> bServiceProvider) {
        this.bServiceProvider = bServiceProvider;
    }

    public void doSomething() {
        BService bService = bServiceProvider.getObject();
        bService.doSomething();
    }
}
```

它的语义是：

```text
我现在不需要 BService，真正用的时候再获取。
```

适合依赖确实是“运行时可选/延迟”的场景。

---

## 8. 企业项目中最常见的循环依赖来源

### 8.1 Service 之间互相调用

```text
UserService -> OrderService
OrderService -> UserService
```

这是最常见的。

通常说明两个 Service 的职责划分不清。

---

### 8.2 Manager / Service 分层混乱

```text
OrderService -> OrderManager
OrderManager -> OrderService
```

这种通常是“分层名义上存在，实际上边界没切开”。

---

### 8.3 Security 配置循环依赖

例如：

```text
SecurityConfig -> UserDetailsService
UserDetailsService -> PasswordEncoder
PasswordEncoder -> SecurityConfig
```

解决方式通常是把 `PasswordEncoder` 独立成配置 Bean，不要让它依赖完整的 `SecurityConfig`。

---

### 8.4 Mapper / Converter 互相依赖

```text
UserConverter -> RoleConverter
RoleConverter -> UserConverter
```

这种要小心。  
DTO 转换器最好保持简单，不要做复杂业务查询。

---

## 9. 判断是否应该修复的标准

遇到循环依赖，不要第一反应就加：

```yaml
spring.main.allow-circular-references: true
```

应该先问：

```text
1. A 调 B，B 又调 A，是不是职责边界错了？
2. 是否可以抽出第三个组件？
3. 是否可以改成事件驱动？
4. 是否可以把一个方向的调用改成 Repository / Domain Service？
5. 是否只是某个工具类、转换器、配置类被错误注入？
```

如果是老项目临时升级 Spring Boot，短期可以开配置；  
如果是新项目或正在重构，建议彻底打破循环依赖。

---

## 10. 总结

一句话：

> **Spring Boot 默认禁用循环依赖，是为了强制暴露不健康的 Bean 设计，而不是因为 Spring 完全没有能力处理。**

核心结论：

```text
1. 构造器循环依赖：本来就无法解决。
2. field/setter 循环依赖：以前 Spring 可以通过三级缓存部分解决。
3. Spring Boot 2.6+ 默认禁止循环引用。
4. 可以用 allow-circular-references=true 临时放开。
5. 最佳实践是重构代码，打破 Bean 依赖闭环。
```

面试加分说法：

> Spring 的三级缓存可以解决部分单例 Bean 的 setter/field 循环依赖，但不能解决构造器循环依赖。Spring Boot 2.6 之后默认关闭循环引用，是为了避免不清晰的 Bean 生命周期、AOP 代理提前暴露、职责边界混乱等问题。生产项目中应优先通过职责拆分、事件机制、延迟依赖等方式打破循环，而不是依赖配置放开。