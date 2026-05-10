
[xfg依赖注入](https://bugstack.cn/md/road-map/spring-dependency-injection.html)

# Spring Dependency Injection：依赖注入使用技巧

这篇文章的核心不是讲“什么是 DI”，而是讲：**在实际业务项目中，如何把 Spring 的注入能力用成一种工程设计手段**。

小傅哥这篇文章的案例工程分成 `config / domain / infrastructure / trigger / ApiTest` 几块，分别演示配置类、多个实现类、仓储对象、触发器层注入、对象初始化销毁监听等场景。文章重点案例包括：构造器注入、`List/Map` 注入、空注入、`@Primary`、`@ConditionalOnMissingBean`、`@ConditionalOnProperty`、自定义 `Condition`、`@Profile`、XML 配置引入、原型对象等。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

下面我按**实际运用优先级**来讲。

---

# 1. 先建立正确认知：DI 不是“少写 new”

很多人理解依赖注入：

```java
@Autowired
private UserService userService;
```

然后觉得 DI 就是：

> 不用自己 `new`，Spring 帮我创建对象。

这个理解太浅了。

更准确地说，Spring DI 的价值是：

```text
对象创建权交给容器
对象依赖关系交给容器
对象生命周期交给容器
对象选择策略交给容器
对象扩展点交给容器
```

所以它不是简单的“帮你 new 对象”，而是一个**对象装配系统**。

在业务开发里，DI 的真正用途是：

```text
解耦实现类
消除 if-else
支持策略模式
支持插件化扩展
支持环境隔离
支持组件自动装配
支持测试替换
支持配置化开关
```

这才是依赖注入的实战价值。

---

# 2. 最推荐：构造器注入

## 2.1 为什么不要优先用字段注入？

很多老项目里常见：

```java
@Service
public class OrderService {

    @Autowired
    private PayService payService;

    @Autowired
    private InventoryService inventoryService;
}
```

能跑，但不推荐。

问题是：

```text
依赖关系不显式
字段不能声明 final
单元测试不方便
对象可能处于半初始化状态
循环依赖容易被掩盖
类职责膨胀不明显
```

Spring 官方文档也明确建议：强制依赖优先使用构造器注入，因为它可以让组件不可变，并确保必需依赖不为 `null`；如果一个构造器参数太多，也是一种职责过重的坏味道。([Home](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html "Dependency Injection :: Spring Framework"))

---

## 2.2 推荐写法

```java
@Service
public class OrderAppService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;

    public OrderAppService(OrderRepository orderRepository,
                           PaymentService paymentService,
                           InventoryService inventoryService) {
        this.orderRepository = orderRepository;
        this.paymentService = paymentService;
        this.inventoryService = inventoryService;
    }
}
```

如果用 Lombok：

```java
@Service
@RequiredArgsConstructor
public class OrderAppService {

    private final OrderRepository orderRepository;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
}
```

这是现在 Java 后端项目中最推荐的风格。

---

## 2.3 实战判断标准

|场景|推荐方式|
|---|---|
|业务服务依赖 DAO / Repository|构造器注入|
|Controller 依赖 Application Service|构造器注入|
|Domain Service 依赖其他领域服务|构造器注入|
|必须存在的依赖|构造器注入|
|可选依赖|`ObjectProvider` / `Optional` / `@Autowired(required = false)`|
|第三方 SDK Client|`@Bean` + 构造器注入|
|多实现策略|`Map<String, Interface>` 注入|

---

# 3. 最实用技巧：`Map<String, Interface>` 注入

这是文章里最值得掌握的点。

小傅哥的案例是：

```java
private final List<IAwardService> awardServices;
private final Map<String, IAwardService> awardServiceMap;

public AwardController(List<IAwardService> awardServices,
                       Map<String, IAwardService> awardServiceMap) {
    this.awardServices = awardServices;
    this.awardServiceMap = awardServiceMap;
}
```

如果 Spring 容器里有多个 `IAwardService` 实现，Spring 可以自动注入成一个 `List` 或 `Map`；其中 `Map` 的 key 是 Bean 名称，value 是对应 Bean 实例。文章里用这个能力根据 `awardKey` 直接定位奖品发放服务，从而减少大量 `if-else`。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

---

## 3.1 典型错误写法：大量 if-else

假设你有不同奖品：

```text
优惠券
积分
红包
会员卡
AI 模型额度
```

新手很容易写成：

```java
public void distribute(String userId, String awardType) {
    if ("coupon".equals(awardType)) {
        couponAwardService.distribute(userId);
    } else if ("point".equals(awardType)) {
        pointAwardService.distribute(userId);
    } else if ("red_packet".equals(awardType)) {
        redPacketAwardService.distribute(userId);
    } else if ("vip".equals(awardType)) {
        vipAwardService.distribute(userId);
    }
}
```

问题：

```text
每新增一种奖品，都要改主流程
违反开闭原则
业务分支越来越臃肿
测试困难
多人改同一个类，冲突概率高
```

---

## 3.2 更好的写法：策略接口 + Map 注入

定义接口：

```java
public interface AwardService {

    void distribute(String userId);

    String awardType();
}
```

优惠券实现：

```java
@Service("coupon")
public class CouponAwardService implements AwardService {

    @Override
    public void distribute(String userId) {
        // 发优惠券
    }

    @Override
    public String awardType() {
        return "coupon";
    }
}
```

积分实现：

```java
@Service("point")
public class PointAwardService implements AwardService {

    @Override
    public void distribute(String userId) {
        // 发积分
    }

    @Override
    public String awardType() {
        return "point";
    }
}
```

红包实现：

```java
@Service("red_packet")
public class RedPacketAwardService implements AwardService {

    @Override
    public void distribute(String userId) {
        // 发红包
    }

    @Override
    public String awardType() {
        return "red_packet";
    }
}
```

统一发奖服务：

```java
@Service
@RequiredArgsConstructor
public class AwardDispatchService {

    private final Map<String, AwardService> awardServiceMap;

    public void distribute(String userId, String awardType) {
        AwardService awardService = awardServiceMap.get(awardType);

        if (awardService == null) {
            throw new IllegalArgumentException("Unsupported award type: " + awardType);
        }

        awardService.distribute(userId);
    }
}
```

这就是典型的：

```text
Spring DI + 策略模式 + 注册表模式
```

---

## 3.3 这个技巧适合哪些业务？

非常多。

|业务场景|key|value|
|---|---|---|
|支付渠道|`alipay` / `wechat` / `stripe`|`PaymentHandler`|
|登录方式|`password` / `sms` / `github`|`LoginHandler`|
|文件存储|`local` / `oss` / `s3`|`StorageService`|
|消息发送|`email` / `sms` / `wechat`|`MessageSender`|
|订单状态处理|`CREATED` / `PAID` / `CANCELLED`|`OrderStateHandler`|
|AI 模型调用|`openai` / `deepseek` / `claude`|`ModelClient`|
|数据导入|`csv` / `excel` / `json`|`ImportHandler`|
|规则引擎节点|`risk` / `coupon` / `stock`|`RuleNode`|

你现在做 Java 后端 + AI 项目，最适合用在：

```text
多模型调用
多 embedding provider
多向量库适配
多文档解析器
多 chunk 策略
多 agent tool handler
```

例如：

```java
public interface ChatModelClient {
    ChatResponse chat(ChatRequest request);
}
```

```java
@Service("openai")
public class OpenAIChatModelClient implements ChatModelClient {
}
```

```java
@Service("deepseek")
public class DeepSeekChatModelClient implements ChatModelClient {
}
```

```java
@Service("qwen")
public class QwenChatModelClient implements ChatModelClient {
}
```

```java
@Service
@RequiredArgsConstructor
public class ChatModelRouter {

    private final Map<String, ChatModelClient> modelClientMap;

    public ChatResponse chat(String provider, ChatRequest request) {
        ChatModelClient client = modelClientMap.get(provider);

        if (client == null) {
            throw new IllegalArgumentException("Unsupported model provider: " + provider);
        }

        return client.chat(request);
    }
}
```

这比到处写：

```java
if ("openai".equals(provider)) ...
else if ("deepseek".equals(provider)) ...
else if ("qwen".equals(provider)) ...
```

强太多。

---

# 4. `List<T>` 注入：适合责任链、过滤器链、规则链

`Map<String, T>` 适合**按 key 精确路由**。

`List<T>` 适合**一组对象依次执行**。

例如风控规则：

```java
public interface RiskRule {

    void check(Order order);
}
```

```java
@Component
@Order(1)
public class UserStatusRiskRule implements RiskRule {

    @Override
    public void check(Order order) {
        // 检查用户状态
    }
}
```

```java
@Component
@Order(2)
public class AmountRiskRule implements RiskRule {

    @Override
    public void check(Order order) {
        // 检查金额
    }
}
```

```java
@Component
@Order(3)
public class FrequencyRiskRule implements RiskRule {

    @Override
    public void check(Order order) {
        // 检查下单频率
    }
}
```

统一执行：

```java
@Service
@RequiredArgsConstructor
public class RiskCheckService {

    private final List<RiskRule> riskRules;

    public void check(Order order) {
        for (RiskRule rule : riskRules) {
            rule.check(order);
        }
    }
}
```

这种写法适合：

```text
责任链
过滤器链
校验链
规则链
导入处理链
审批流程节点
订单状态推进流程
```

---

# 5. `@Primary`：多个实现里指定默认实现

文章里的例子是：一个 `IAwardService` 有多个实现类时，直接注入接口可能出现 `NoUniqueBeanDefinitionException`，可以用 `@Primary` 指定首选 Bean。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

例如：

```java
public interface ModelClient {
    String chat(String prompt);
}
```

```java
@Service
@Primary
public class DeepSeekModelClient implements ModelClient {

    @Override
    public String chat(String prompt) {
        return "deepseek response";
    }
}
```

```java
@Service
public class OpenAIModelClient implements ModelClient {

    @Override
    public String chat(String prompt) {
        return "openai response";
    }
}
```

这样直接注入：

```java
@Service
@RequiredArgsConstructor
public class AiService {

    private final ModelClient modelClient;
}
```

默认拿到的是 `DeepSeekModelClient`。

---

## 5.1 什么时候用 `@Primary`？

适合：

```text
有多个实现，但系统需要一个默认实现
大多数场景都用默认实现
少数场景才指定具体实现
```

例如：

```text
默认支付渠道
默认 AI 模型
默认文件存储
默认风控策略
默认缓存实现
默认消息推送方式
```

---

## 5.2 `@Primary` 和 `@Qualifier` 的区别

|注解|作用|
|---|---|
|`@Primary`|在多个候选 Bean 中指定默认 Bean|
|`@Qualifier`|明确指定注入哪个 Bean|
|`Map<String, T>`|把所有同类型 Bean 都注入进来，自行按 key 选择|
|`List<T>`|把所有同类型 Bean 按顺序注入进来，统一遍历执行|

例如：

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final PaymentClient paymentClient;
}
```

如果有多个 `PaymentClient`，用 `@Primary` 表示默认支付渠道。

但是如果这个类必须用微信支付：

```java
@Service
public class WechatRefundService {

    private final PaymentClient paymentClient;

    public WechatRefundService(@Qualifier("wechatPaymentClient") PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }
}
```

---

# 6. `@Qualifier`：精确指定注入对象

文章中在 `@Bean` 方法参数里使用了 `@Qualifier("redisson01")`，用于指定具体注入哪个 Bean。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

示例：

```java
@Configuration
public class RedisConfig {

    @Bean("defaultRedissonClient")
    public RedissonClient defaultRedissonClient() {
        return createClient("default");
    }

    @Bean("orderRedissonClient")
    public RedissonClient orderRedissonClient() {
        return createClient("order");
    }
}
```

使用：

```java
@Service
public class OrderLockService {

    private final RedissonClient redissonClient;

    public OrderLockService(@Qualifier("orderRedissonClient") RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }
}
```

适合场景：

```text
多个 Redis Client
多个 DataSource
多个 RestTemplate
多个 WebClient
多个 MQ Producer
多个 AI Client
```

---

# 7. 空注入：`@Autowired(required = false)` 要慎用

文章提到，如果某个 Bean 没有注册，可以使用：

```java
@Autowired(required = false)
private NullAwardService nullAwardService;
```

这样 Bean 不存在时不会报错。文章给的用途是：外部接口对接测试阶段，有些服务可能需要关闭，不实例化对象。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

但实际项目里，这个要慎用。

因为它会让代码里出现：

```java
if (someService != null) {
    someService.doSomething();
}
```

这类分支会逐渐污染业务代码。

---

## 7.1 更推荐：`ObjectProvider<T>`

```java
@Service
@RequiredArgsConstructor
public class AiCallbackService {

    private final ObjectProvider<CallbackHandler> callbackHandlerProvider;

    public void callback(String data) {
        CallbackHandler handler = callbackHandlerProvider.getIfAvailable();

        if (handler == null) {
            return;
        }

        handler.handle(data);
    }
}
```

好处：

```text
语义更明确：这个依赖是可选的
不会强行制造 null 字段
可以延迟获取
更适合 Starter / SDK / 插件类组件
```

---

## 7.2 也可以使用 `Optional<T>`

```java
@Service
@RequiredArgsConstructor
public class NotifyService {

    private final Optional<SmsSender> smsSender;

    public void send(String phone, String content) {
        smsSender.ifPresent(sender -> sender.send(phone, content));
    }
}
```

适合小范围可选依赖。

---

# 8. `@ConditionalOnMissingBean`：做组件和 Starter 时非常重要

文章里的例子是：

```java
@Bean("redisson01")
@ConditionalOnMissingBean
public String redisson01() {
    return "模拟的 Redis 客户端 01";
}
```

它的作用是：当 Spring 应用上下文里不存在某个特定类型的 Bean 时，才创建当前 Bean；文章也指出，这通常用于组件开发，避免业务工程引入同类组件后重复创建对象。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

---

## 8.1 实战场景：提供默认实现，但允许业务覆盖

假设你开发一个 AI SDK Starter：

```java
@Configuration
public class AiAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public ModelClient modelClient() {
        return new DefaultModelClient();
    }
}
```

业务方如果不配置，就用默认实现。

业务方如果自己配置：

```java
@Bean
public ModelClient modelClient() {
    return new DeepSeekModelClient();
}
```

那么 Starter 里的默认 Bean 就不会创建。

这就是 Spring Boot 自动装配最常见的扩展模式：

```text
框架提供默认 Bean
业务系统可以覆盖 Bean
框架不强绑业务实现
```

---

# 9. `@ConditionalOnProperty`：配置开关控制 Bean 是否创建

文章中使用：

```java
@Bean
@ConditionalOnProperty(
    value = "sdk.config.enabled",
    havingValue = "true",
    matchIfMissing = false
)
public String createTopic(...) {
    return redisson;
}
```

它可以根据配置决定是否创建 Bean，例如 `sdk.config.enabled=false` 时不创建。文章指出，这种方式适合组件或业务功能的启停，不需要通过删除依赖或注释代码来关闭功能。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

---

## 9.1 实战：AI 功能开关

```yaml
ai:
  chat:
    enabled: true
    provider: deepseek
```

配置类：

```java
@Configuration
@EnableConfigurationProperties(AiProperties.class)
public class AiAutoConfiguration {

    @Bean
    @ConditionalOnProperty(
        prefix = "ai.chat",
        name = "enabled",
        havingValue = "true"
    )
    public ChatService chatService(ModelClient modelClient) {
        return new ChatService(modelClient);
    }
}
```

这样你可以做到：

```text
开发环境关闭 AI 调用
测试环境使用 mock provider
生产环境开启真实 provider
私有化部署环境按需开启
```

---

# 10. 自定义 `Condition`：复杂条件下控制 Bean 创建

文章中展示了实现 `Condition` 接口：

```java
public class BeanCreateCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String active = System.getProperty("isOpenWhitelistedUsers");
        return null != active && active.equals("true");
    }
}
```

然后：

```java
@Bean
@Conditional(BeanCreateCondition.class)
public List<String> whitelistedUsers() {
    return List.of("user001", "user002", "user003");
}
```

也就是自己写判断逻辑，决定 Bean 是否创建。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

---

## 10.1 什么时候需要自定义 Condition？

一般不用。优先用：

```text
@ConditionalOnProperty
@ConditionalOnMissingBean
@ConditionalOnClass
@Profile
```

只有这些不够时，再自定义 `Condition`。

适合：

```text
根据系统属性判断
根据 classpath 判断
根据操作系统判断
根据是否存在某个文件判断
根据许可证 license 判断
根据多个配置组合判断
```

例如：

```java
public class VectorDbCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();

        String enabled = env.getProperty("vector.enabled");
        String provider = env.getProperty("vector.provider");

        return "true".equals(enabled) && "milvus".equals(provider);
    }
}
```

---

# 11. `@Profile`：按环境创建 Bean

文章里的例子：

```java
@Service
@Profile({"prod", "test"})
@Lazy
public class AliPayAwardService implements IAwardService {
}
```

只有在 `prod` 或 `test` profile 下才实例化。文章给的场景是：某些支付类服务必须指定上线或测试环境才实例化。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

---

## 11.1 实战例子

开发环境用 mock 支付：

```java
@Service
@Profile("dev")
public class MockPaymentClient implements PaymentClient {
}
```

生产环境用真实支付：

```java
@Service
@Profile("prod")
public class AliPayClient implements PaymentClient {
}
```

配置：

```yaml
spring:
  profiles:
    active: dev
```

这比在代码里写：

```java
if ("dev".equals(env)) {
    mockPay();
} else {
    realPay();
}
```

更干净。

---

# 12. `@Bean`：把第三方对象交给 Spring 管理

有些类不是你写的，不能加 `@Component`。

例如：

```text
RedissonClient
ObjectMapper
RestTemplate
WebClient
OkHttpClient
ThreadPoolExecutor
OpenAI Client
Milvus Client
MinIO Client
```

这时用 `@Bean`。

```java
@Configuration
public class HttpClientConfig {

    @Bean
    public OkHttpClient okHttpClient() {
        return new OkHttpClient.Builder()
                .connectTimeout(Duration.ofSeconds(3))
                .readTimeout(Duration.ofSeconds(30))
                .build();
    }
}
```

然后注入：

```java
@Service
@RequiredArgsConstructor
public class RemoteApiService {

    private final OkHttpClient okHttpClient;
}
```

一句话：

```text
自己写的业务类，用 @Component / @Service
第三方类、需要复杂构造的类，用 @Bean
```

---

# 13. `@ConfigurationProperties`：不要到处写 `@Value`

文章中使用了：

```java
@ConfigurationProperties(prefix = "sdk.config", ignoreInvalidFields = true)
public class AutoConfigProperties {

    private boolean enable;
    private String apiHost;
    private String apiSecretKey;
}
```

这类配置对象非常适合管理成组配置。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

---

## 13.1 不推荐

```java
@Value("${ai.openai.api-key}")
private String apiKey;

@Value("${ai.openai.base-url}")
private String baseUrl;

@Value("${ai.openai.timeout}")
private Integer timeout;
```

字段一多，非常散。

---

## 13.2 推荐

```yaml
ai:
  openai:
    api-key: sk-xxx
    base-url: https://api.openai.com
    timeout-seconds: 30
```

```java
@Data
@ConfigurationProperties(prefix = "ai.openai")
public class OpenAiProperties {

    private String apiKey;
    private String baseUrl;
    private Integer timeoutSeconds;
}
```

```java
@Configuration
@EnableConfigurationProperties(OpenAiProperties.class)
public class OpenAiConfig {

    @Bean
    public OpenAiClient openAiClient(OpenAiProperties properties) {
        return new OpenAiClient(
            properties.getApiKey(),
            properties.getBaseUrl(),
            properties.getTimeoutSeconds()
        );
    }
}
```

这才是工程化写法。

---

# 14. `@Scope("prototype")`：原型对象，不要乱用

文章中：

```java
@Component
@Scope("prototype")
public class LogicChain {
}
```

然后通过：

```java
applicationContext.getBean(LogicChain.class)
```

每次获取都是新对象。文章给出的用途是：对于动态、不同责任链创建，可以确保每个对象都是自己的实例。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

---

## 14.1 什么时候用 prototype？

适合：

```text
每次请求都需要新状态对象
对象内部有状态，不能共享
动态构造责任链
动态构造流程节点
临时任务上下文
```

例如：

```java
@Component
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class ImportTaskContext {

    private String taskId;
    private int successCount;
    private int failCount;
}
```

每个导入任务要一个独立上下文。

---

## 14.2 注意：prototype 注入 singleton 有坑

如果你这样写：

```java
@Service
@RequiredArgsConstructor
public class ImportService {

    private final ImportTaskContext context;
}
```

`ImportService` 是单例，`context` 只会在 `ImportService` 创建时注入一次。

你以为每次使用都是新的，其实不是。

正确方式：

```java
@Service
@RequiredArgsConstructor
public class ImportService {

    private final ObjectProvider<ImportTaskContext> contextProvider;

    public void importFile(String fileId) {
        ImportTaskContext context = contextProvider.getObject();
        context.setTaskId(fileId);
    }
}
```

---

# 15. `@Lazy`：延迟初始化，不是性能银弹

文章中 `@Profile` 示例也带了 `@Lazy`。`@Lazy` 的含义是：Bean 不在容器启动时立即创建，而是在第一次使用时创建。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

适合：

```text
初始化很重的对象
低频使用的对象
启动时不希望立即连接外部服务
某些可选功能模块
```

例如：

```java
@Service
@Lazy
public class HeavyReportService {

    public HeavyReportService() {
        // 初始化很重
    }
}
```

但不要滥用。

因为它会导致：

```text
启动时发现不了问题
首次调用时才报错
线上流量第一次打进来时抖动
依赖问题暴露延迟
```

Spring 官方文档也提到，默认预实例化单例 Bean 的好处是应用启动时就能暴露配置和依赖问题，而不是等到第一次请求对象时才出错。([Home](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html "Dependency Injection :: Spring Framework"))

---

# 16. `@DependsOn`：控制 Bean 初始化顺序

文章提到：

```java
@DependsOn({"openai_model", "openai_use_count", "user_credit_random"})
```

用于声明某个 Bean 初始化之前，先初始化哪些 Bean。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

适合：

```text
初始化顺序敏感的底层组件
必须先完成配置加载
必须先完成连接池初始化
必须先完成注册中心初始化
```

但业务代码里不应大量使用。

如果你大量依赖 `@DependsOn`，可能说明：

```text
对象设计耦合过重
初始化逻辑不清晰
启动流程被业务代码污染
```

---

# 17. `Environment`：读取运行环境配置

文章提到可以注入：

```java
@Autowired
private Environment env;
```

然后读取配置：

```java
env.getProperty("app.name");
env.getProperty("app.version");
```

适合一些框架型、底层型逻辑。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

但业务项目里，优先级应该是：

```text
@ConfigurationProperties > @Value > Environment
```

也就是说：

```java
@ConfigurationProperties
```

最适合管理业务配置。

`Environment` 更适合写框架、Starter、条件判断、动态读取。

---

# 18. `@ImportResource`：兼容老 Spring XML

文章中演示了：

```java
@ImportResource("classpath:spring/spring.xml")
@PropertySource("classpath:properties/application.properties")
```

这种方式可以在 Spring Boot 项目中引入 XML 配置，用于兼容老项目或还没有升级成 Spring Boot Starter 的组件。([BugStack](https://bugstack.cn/md/road-map/spring-dependency-injection.html "Spring Dependency Injection | 小傅哥 bugstack 虫洞栈"))

现在新项目一般不主动写 XML。

但是在以下情况你会遇到：

```text
接手老系统
接入老中间件
公司内部历史组件
Dubbo 老项目
传统 Spring MVC 项目迁移到 Spring Boot
```

这时要看得懂。

---

# 19. 循环依赖：不要靠 Spring “帮你兜底”

Spring 官方文档提到，如果主要使用构造器注入，可能出现无法解决的循环依赖。例如 A 构造器依赖 B，B 构造器又依赖 A，Spring 会在运行时检测到循环引用并抛出 `BeanCurrentlyInCreationException`。([Home](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html "Dependency Injection :: Spring Framework"))

例如：

```java
@Service
@RequiredArgsConstructor
public class AService {
    private final BService bService;
}
```

```java
@Service
@RequiredArgsConstructor
public class BService {
    private final AService aService;
}
```

这说明两个类互相知道太多。

不要第一反应就用：

```java
@Lazy
```

或者改成字段注入绕过去。

更好的处理是重构。

---

## 19.1 常见解决方式

### 方式一：抽出第三方协调者

错误：

```text
A 调 B
B 调 A
```

改成：

```text
C 编排 A 和 B
A 不知道 B
B 不知道 A
```

```java
@Service
@RequiredArgsConstructor
public class OrderPayCoordinator {

    private final OrderService orderService;
    private final PaymentService paymentService;

    public void pay(Long orderId) {
        Order order = orderService.get(orderId);
        paymentService.pay(order);
        orderService.markPaid(orderId);
    }
}
```

### 方式二：用事件解耦

```java
public record OrderPaidEvent(Long orderId) {
}
```

```java
applicationEventPublisher.publishEvent(new OrderPaidEvent(orderId));
```

监听：

```java
@Component
@RequiredArgsConstructor
public class CouponOnOrderPaidListener {

    private final CouponService couponService;

    @EventListener
    public void on(OrderPaidEvent event) {
        couponService.issue(event.orderId());
    }
}
```

### 方式三：重新划分职责

如果两个 Service 互相调用，通常说明：

```text
边界没分清
职责混乱
应用服务和领域服务混在一起
```

---

# 20. 结合 DDD 怎么用 DI？

这部分很关键。

你最近一直在学 DDD，Spring DI 在 DDD 工程里应该这样用：

```text
Controller / Trigger
  注入 Application Service

Application Service
  注入 Repository 接口、Domain Service、外部 Port、Publisher

Domain Model
  尽量不依赖 Spring

Infrastructure
  实现 Repository、Client、Mapper

Configuration
  装配第三方 SDK、策略实现、外部组件
```

---

## 20.1 不要在 Entity / Aggregate 里注入 Spring Bean

不要这样：

```java
public class Order {

    @Autowired
    private CouponService couponService;
}
```

领域对象不应该变成 Spring Bean。

聚合应该这样：

```java
public class Order {

    private OrderStatus status;

    public void pay() {
        if (this.status != OrderStatus.CREATED) {
            throw new IllegalStateException("订单状态不允许支付");
        }

        this.status = OrderStatus.PAID;
    }
}
```

外部依赖放到 Application Service：

```java
@Service
@RequiredArgsConstructor
public class PayOrderUseCase {

    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public void pay(Long orderId) {
        Order order = orderRepository.findById(orderId);

        paymentClient.pay(order.getPayAmount());

        order.pay();

        orderRepository.save(order);

        eventPublisher.publish(new OrderPaidEvent(orderId));
    }
}
```

这里 DI 的作用是装配应用层依赖，而不是污染领域模型。

---

# 21. 实战模板：支付渠道路由

这是最典型的 DI 使用技巧。

## 21.1 接口

```java
public interface PaymentHandler {

    PayResult pay(PayCommand command);
}
```

## 21.2 支付宝

```java
@Service("alipay")
public class AliPayHandler implements PaymentHandler {

    @Override
    public PayResult pay(PayCommand command) {
        // 调用支付宝
        return PayResult.success();
    }
}
```

## 21.3 微信

```java
@Service("wechat")
public class WechatPayHandler implements PaymentHandler {

    @Override
    public PayResult pay(PayCommand command) {
        // 调用微信支付
        return PayResult.success();
    }
}
```

## 21.4 路由器

```java
@Service
@RequiredArgsConstructor
public class PaymentRouter {

    private final Map<String, PaymentHandler> paymentHandlerMap;

    public PayResult pay(PayCommand command) {
        PaymentHandler handler = paymentHandlerMap.get(command.getChannel());

        if (handler == null) {
            throw new IllegalArgumentException("Unsupported payment channel: " + command.getChannel());
        }

        return handler.pay(command);
    }
}
```

## 21.5 业务调用

```java
@Service
@RequiredArgsConstructor
public class PayOrderUseCase {

    private final PaymentRouter paymentRouter;

    public void pay(PayOrderCommand command) {
        paymentRouter.pay(new PayCommand(
            command.getOrderId(),
            command.getChannel(),
            command.getAmount()
        ));
    }
}
```

这个结构比 `if-else` 清晰很多。

---

# 22. 实战模板：AI Provider 路由

你做 AI 应用时非常有用。

```java
public interface LlmClient {

    ChatResponse chat(ChatRequest request);
}
```

```java
@Service("openai")
public class OpenAiClient implements LlmClient {

    @Override
    public ChatResponse chat(ChatRequest request) {
        return null;
    }
}
```

```java
@Service("deepseek")
public class DeepSeekClient implements LlmClient {

    @Override
    public ChatResponse chat(ChatRequest request) {
        return null;
    }
}
```

```java
@Service("qwen")
public class QwenClient implements LlmClient {

    @Override
    public ChatResponse chat(ChatRequest request) {
        return null;
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class LlmRouter {

    private final Map<String, LlmClient> llmClientMap;

    public ChatResponse chat(String provider, ChatRequest request) {
        LlmClient client = llmClientMap.get(provider);

        if (client == null) {
            throw new IllegalArgumentException("Unsupported LLM provider: " + provider);
        }

        return client.chat(request);
    }
}
```

配置：

```yaml
ai:
  default-provider: deepseek
```

```java
@Data
@ConfigurationProperties(prefix = "ai")
public class AiProperties {

    private String defaultProvider;
}
```

```java
@Service
@RequiredArgsConstructor
public class AiChatService {

    private final LlmRouter llmRouter;
    private final AiProperties aiProperties;

    public ChatResponse chat(ChatRequest request) {
        return llmRouter.chat(aiProperties.getDefaultProvider(), request);
    }
}
```

这就是一个可扩展的多模型架构。

---

# 23. 实战模板：文档解析器路由

适合 RAG / 知识库系统。

```java
public interface DocumentParser {

    List<DocumentChunk> parse(Resource resource);

    boolean support(String fileType);
}
```

```java
@Service("pdf")
public class PdfDocumentParser implements DocumentParser {
}
```

```java
@Service("docx")
public class DocxDocumentParser implements DocumentParser {
}
```

```java
@Service("md")
public class MarkdownDocumentParser implements DocumentParser {
}
```

```java
@Service
@RequiredArgsConstructor
public class DocumentParseService {

    private final Map<String, DocumentParser> parserMap;

    public List<DocumentChunk> parse(String fileType, Resource resource) {
        DocumentParser parser = parserMap.get(fileType);

        if (parser == null) {
            throw new IllegalArgumentException("Unsupported file type: " + fileType);
        }

        return parser.parse(resource);
    }
}
```

以后新增 Excel 解析器：

```java
@Service("xlsx")
public class ExcelDocumentParser implements DocumentParser {
}
```

主流程不用动。

---

# 24. 依赖注入的工程准则

## 24.1 优先级建议

```text
构造器注入 > @Bean 参数注入 > Setter 注入 > 字段注入
```

字段注入不是不能用，但在严肃项目里不推荐作为默认风格。

---

## 24.2 Bean 命名要稳定

如果你用：

```java
@Service("deepseek")
```

那么配置里也应该用：

```yaml
ai:
  provider: deepseek
```

不要今天叫：

```text
deepSeekModelClient
```

明天叫：

```text
deepseek
```

后天又叫：

```text
deepseek-v3
```

建议形成规范：

```text
支付渠道：alipay / wechat / stripe
模型厂商：openai / deepseek / qwen / claude
存储类型：local / oss / s3 / minio
消息类型：sms / email / webhook
```

---

## 24.3 不要把 DI 当 Service Locator 滥用

少写这种：

```java
ApplicationContext.getBean(beanName);
```

这会把代码重新写回“主动查找依赖”的风格。

可以在少数动态场景使用，比如：

```text
prototype Bean 获取
插件加载
复杂运行时路由
框架底层能力
```

但业务服务里优先使用：

```java
Map<String, SomeInterface>
ObjectProvider<T>
构造器注入
```

---

## 24.4 一个类构造器参数太多，要警惕

如果你看到：

```java
public OrderService(A a, B b, C c, D d, E e, F f, G g, H h, I i) {
}
```

不要急着怪构造器注入麻烦。

它是在暴露设计问题：

```text
这个类可能职责太多
应用服务可能变成上帝类
流程编排可能需要拆分
部分逻辑应该下沉到领域服务
部分逻辑应该抽成策略
部分逻辑应该交给事件监听
```

Spring 官方文档也提到，构造器参数过多通常是类职责过重的坏味道。([Home](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-collaborators.html "Dependency Injection :: Spring Framework"))

---

# 25. 依赖注入常见反模式

## 反模式一：Controller 直接注入一堆 Repository

不推荐：

```java
@RestController
@RequiredArgsConstructor
public class OrderController {

    private final OrderMapper orderMapper;
    private final UserMapper userMapper;
    private final CouponMapper couponMapper;
}
```

推荐：

```java
@RestController
@RequiredArgsConstructor
public class OrderController {

    private final OrderAppService orderAppService;
}
```

Controller 只负责 HTTP 入参和出参，不负责业务编排。

---

## 反模式二：Service 里到处 `ApplicationContext.getBean`

不推荐：

```java
Object bean = applicationContext.getBean(type);
```

推荐：

```java
private final Map<String, Handler> handlerMap;
```

---

## 反模式三：所有类都加 `@Service`

不是所有类都应该交给 Spring。

这些通常不需要：

```text
Entity
Value Object
DTO
Command
Query
PO
普通工具对象
纯算法对象
```

这些适合交给 Spring：

```text
Controller
Application Service
Domain Service
Repository 实现
Client
Mapper
Listener
Scheduler
Configuration
Factory
Strategy Handler
```

---

## 反模式四：为了修循环依赖乱加 `@Lazy`

不推荐：

```java
@Lazy
@Autowired
private SomeService someService;
```

先问：

```text
为什么它们互相依赖？
是不是边界错了？
是不是需要事件？
是不是需要协调者？
是不是职责没拆？
```

---

# 26. 最实用的选择表

|需求|推荐方案|
|---|---|
|必需依赖|构造器注入|
|可选依赖|`ObjectProvider<T>`|
|多个实现，选默认|`@Primary`|
|多个实现，指定某个|`@Qualifier`|
|多个实现，按 key 路由|`Map<String, T>`|
|多个实现，依次执行|`List<T>` + `@Order`|
|第三方对象注册|`@Bean`|
|配置开关控制创建|`@ConditionalOnProperty`|
|默认组件可被业务覆盖|`@ConditionalOnMissingBean`|
|按环境创建|`@Profile`|
|每次获取新对象|`@Scope("prototype")` + `ObjectProvider`|
|老 XML 兼容|`@ImportResource`|
|复杂条件创建 Bean|自定义 `Condition`|

---

# 27. 这篇文章真正应该学到什么？

不要只记注解。

这篇文章的真正重点是：

```text
Spring DI 可以帮你做对象装配
对象装配可以变成业务扩展机制
业务扩展机制可以减少 if-else
减少 if-else 可以让系统更符合开闭原则
开闭原则可以让项目更容易扩展和维护
```

最值得你优先掌握的不是所有注解，而是这 5 个实战能力：

```text
1. 构造器注入，建立清晰依赖关系
2. Map 注入，用于策略路由
3. List 注入，用于责任链 / 规则链
4. Conditional 注解，用于组件开关和自动装配
5. Profile / Properties，用于环境隔离和配置化
```

你可以把 Spring DI 理解为一句话：

> **它不是简单地把对象塞进来，而是让你的业务系统具备可组合、可替换、可扩展的对象装配能力。**