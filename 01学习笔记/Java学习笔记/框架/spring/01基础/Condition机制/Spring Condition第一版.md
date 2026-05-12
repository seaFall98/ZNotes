一个Condition分了30个章节，真学晕了吧
要求简化之后[[Spring Condition第二版]]

# Spring Condition：系统讲一遍

先把它放到一句话里：

> **Condition 是 Spring 在“容器启动阶段”做组件装配决策的机制：满足条件就注册 Bean，不满足条件就跳过。**

它解决的不是：

```java
if (orderAmount > 100) {
    discount();
}
```

而是：

```text
当前环境 / 配置 / classpath / Bean 状态 是否满足？
满足 -> 这个 Bean 加入 Spring 容器
不满足 -> 这个 Bean 根本不进入 Spring 容器
```

所以 Condition 本质上是：

```text
Spring 容器级别的组件开关机制
```

不是普通业务代码里的 `if-else`。

---

# 1. Condition 解决的核心问题

在真实项目里，你经常会遇到这些问题：

```text
开发环境用 Mock 支付，生产环境用真实支付
配置打开时才启用短信发送
引入 Redis 依赖时才创建 RedisCacheService
用户自己定义了一个 Bean，就不要创建框架默认 Bean
Web 项目才注册拦截器，非 Web 项目不注册
某个模块关闭时，整个模块相关 Bean 都不加载
```

这些都不是运行时业务分支，而是：

> **启动时决定哪些组件应该存在。**

这就是 Condition 的职责。

---

# 2. 它在 Spring 生命周期里的位置

Spring 启动大致可以理解成：

```text
读取配置类 / 扫描组件
        ↓
解析 @Configuration / @Component / @Bean
        ↓
判断 @Conditional 相关条件
        ↓
注册 BeanDefinition
        ↓
实例化 Bean
        ↓
依赖注入
        ↓
初始化 Bean
```

Condition 发生在：

```text
注册 BeanDefinition 之前
```

也就是说：

```text
条件不满足时，不是 Bean 创建了但不用
而是 BeanDefinition 都不会注册
```

这是它和普通 `if-else` 最大的区别。

---

# 3. 最底层：`Condition` 接口

核心接口是：

```java
public interface Condition {

    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);

}
```

你只需要实现：

```java
boolean matches(...)
```

返回：

```text
true  -> 条件成立，允许注册
false -> 条件不成立，跳过注册
```

示例：

```java
public class AiEnabledCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String enabled = context.getEnvironment().getProperty("ai.enabled");
        return "true".equalsIgnoreCase(enabled);
    }
}
```

使用：

```java
@Configuration
public class AiConfiguration {

    @Bean
    @Conditional(AiEnabledCondition.class)
    public AiService aiService() {
        return new AiService();
    }
}
```

配置：

```yaml
ai:
  enabled: true
```

结果：

```text
ai.enabled=true  -> aiService 注册
ai.enabled=false -> aiService 不注册
```

---

# 4. `@Conditional` 是最原始的条件注解

`@Conditional` 是 Spring Framework 提供的基础注解。

它可以放在：

```text
@Configuration 配置类上
@Bean 方法上
@Component 类上
```

---

## 4.1 放在 `@Bean` 方法上

```java
@Configuration
public class PaymentConfiguration {

    @Bean
    @Conditional(AlipayEnabledCondition.class)
    public PaymentClient alipayClient() {
        return new AlipayClient();
    }

    @Bean
    public OrderService orderService() {
        return new OrderService();
    }
}
```

含义：

```text
只控制 alipayClient 这个 Bean
不影响 orderService
```

---

## 4.2 放在配置类上

```java
@Configuration
@Conditional(AlipayEnabledCondition.class)
public class AlipayConfiguration {

    @Bean
    public PaymentClient alipayClient() {
        return new AlipayClient();
    }

    @Bean
    public RefundClient refundClient() {
        return new RefundClient();
    }
}
```

含义：

```text
整个 AlipayConfiguration 是否生效都由条件决定
条件不满足，里面所有 Bean 都不会注册
```

---

## 4.3 放在组件类上

```java
@Service
@Conditional(SmsEnabledCondition.class)
public class SmsSender {
}
```

含义：

```text
条件满足，SmsSender 被扫描并注册
条件不满足，SmsSender 被跳过
```

实际项目里，更推荐把条件逻辑放在配置类上，而不是大量散落在业务类上。

---

# 5. `ConditionContext` 能拿什么？

`matches` 方法里第一个参数：

```java
ConditionContext context
```

它代表当前 Spring 容器条件判断上下文。

常用方法：

```java
context.getEnvironment();
context.getRegistry();
context.getBeanFactory();
context.getResourceLoader();
context.getClassLoader();
```

实际最常用的是：

```java
context.getEnvironment()
```

用来读取：

```text
application.yml
application.properties
环境变量
JVM system property
profile 信息
```

例如：

```java
public class DeepSeekCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();

        Boolean enabled = env.getProperty("ai.enabled", Boolean.class, false);
        String provider = env.getProperty("ai.provider");

        return enabled && "deepseek".equalsIgnoreCase(provider);
    }
}
```

配置：

```yaml
ai:
  enabled: true
  provider: deepseek
```

---

# 6. `AnnotatedTypeMetadata` 是什么？

第二个参数：

```java
AnnotatedTypeMetadata metadata
```

它代表当前被判断对象上的**注解元信息**。

简单说：

```text
如果 @Conditional 标在类上，它能读这个类上的注解
如果 @Conditional 标在 @Bean 方法上，它能读这个方法上的注解
```

它常用于自定义条件注解。

例如你想写一个自己的注解：

```java
@ConditionalOnFeature("chat")
```

然后根据配置：

```yaml
features:
  chat: true
  payment: false
```

决定某个功能是否启用。

---

# 7. 自定义业务条件注解

## 7.1 定义注解

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(OnFeatureCondition.class)
public @interface ConditionalOnFeature {

    String value();
}
```

## 7.2 实现 Condition

```java
public class OnFeatureCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Map<String, Object> attributes =
                metadata.getAnnotationAttributes(ConditionalOnFeature.class.getName());

        if (attributes == null) {
            return false;
        }

        String featureName = (String) attributes.get("value");
        String propertyKey = "features." + featureName;

        return context.getEnvironment()
                .getProperty(propertyKey, Boolean.class, false);
    }
}
```

## 7.3 使用

```java
@Configuration
@ConditionalOnFeature("chat")
public class ChatConfiguration {

    @Bean
    public ChatService chatService() {
        return new ChatService();
    }
}
```

配置：

```yaml
features:
  chat: true
```

结果：

```text
features.chat=true  -> ChatConfiguration 生效
features.chat=false -> ChatConfiguration 不生效
```

这就是自定义 Condition 的典型用法。

---

# 8. 但实际开发中，你很少直接写 `Condition`

因为 Spring Boot 已经封装好了大量常用条件注解。

你平时用得最多的是这些：

```text
@ConditionalOnProperty
@ConditionalOnBean
@ConditionalOnMissingBean
@ConditionalOnClass
@ConditionalOnMissingClass
@ConditionalOnResource
@ConditionalOnWebApplication
@ConditionalOnNotWebApplication
@ConditionalOnExpression
@Profile
```

它们底层本质都属于：

```text
@Conditional + Condition 实现类
```

只不过 Spring Boot 帮你封装好了。

---

# 9. 最常用：`@ConditionalOnProperty`

根据配置项决定 Bean 是否创建。

```java
@Configuration
public class SmsConfiguration {

    @Bean
    @ConditionalOnProperty(
            prefix = "sms",
            name = "enabled",
            havingValue = "true",
            matchIfMissing = false
    )
    public SmsSender smsSender() {
        return new SmsSender();
    }
}
```

配置：

```yaml
sms:
  enabled: true
```

含义：

```text
sms.enabled=true  -> 创建 SmsSender
sms.enabled=false -> 不创建 SmsSender
配置缺失          -> 不创建 SmsSender
```

---

## 9.1 参数解释

```java
@ConditionalOnProperty(
    prefix = "sms",
    name = "enabled",
    havingValue = "true",
    matchIfMissing = false
)
```

|参数|说明|
|---|---|
|`prefix`|配置前缀|
|`name`|配置属性名|
|`havingValue`|期望值|
|`matchIfMissing`|配置缺失时是否匹配|

---

## 9.2 典型场景

```text
sms.enabled=true         -> 启用短信发送
mq.consumer.enabled=true -> 启用 MQ 消费者
xxl-job.enabled=true     -> 启用定时任务
ai.chat.enabled=true     -> 启用 AI 对话能力
audit.enabled=true       -> 启用审计日志
```

例如：

```java
@Bean
@ConditionalOnProperty(prefix = "ai.chat", name = "enabled", havingValue = "true")
public ChatService chatService() {
    return new ChatService();
}
```

配置：

```yaml
ai:
  chat:
    enabled: true
```

---

# 10. `@ConditionalOnMissingBean`

这个在 Starter 和组件开发里极其重要。

含义：

> **容器里没有这个类型的 Bean 时，才创建当前 Bean。**

示例：

```java
@Configuration
public class CacheAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public CacheService cacheService() {
        return new LocalCacheService();
    }
}
```

如果业务项目没有自己定义 `CacheService`，就使用默认的 `LocalCacheService`。

如果业务项目自己定义了：

```java
@Bean
public CacheService cacheService() {
    return new RedisCacheService();
}
```

那么默认的 `LocalCacheService` 就不会创建。

---

## 10.1 它的本质用途

```text
框架提供默认实现
业务系统可以覆盖默认实现
```

这是 Spring Boot 自动配置的核心风格。

例如：

```java
@Bean
@ConditionalOnMissingBean
public ObjectMapper objectMapper() {
    return new ObjectMapper();
}
```

意思是：

```text
你没配，我给你默认配一个
你自己配了，我尊重你的配置
```

这就是组件设计里的“可覆盖默认值”。

---

# 11. `@ConditionalOnBean`

含义：

> **容器里存在某个 Bean 时，才创建当前 Bean。**

示例：

```java
@Configuration
public class ChatConfiguration {

    @Bean
    @ConditionalOnBean(LlmClient.class)
    public ChatService chatService(LlmClient llmClient) {
        return new ChatService(llmClient);
    }
}
```

含义：

```text
只有 LlmClient 存在时，才创建 ChatService
```

适合这种依赖链：

```text
底层 Client 存在 -> 才创建上层 Service
DataSource 存在 -> 才创建 Repository
RedisClient 存在 -> 才创建 RedisCacheService
LlmClient 存在 -> 才创建 ChatService
```

---

# 12. `@ConditionalOnClass`

含义：

> **classpath 中存在某个类时，才启用配置。**

这在 Starter 里最常见。

例如：

```java
@Configuration
@ConditionalOnClass(RedissonClient.class)
public class RedissonLockAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public LockClient lockClient(RedissonClient redissonClient) {
        return new RedissonLockClient(redissonClient);
    }
}
```

含义：

```text
项目引入了 Redisson 依赖 -> 启用 RedissonLockAutoConfiguration
项目没引入 Redisson 依赖 -> 跳过这个配置
```

它常用于：

```text
引入 Redis 依赖才装配 Redis 相关 Bean
引入 Kafka 依赖才装配 Kafka 相关 Bean
引入 MyBatis 依赖才装配 Mapper 相关 Bean
引入 WebMvc 依赖才装配 Web 相关 Bean
引入 Milvus SDK 才装配向量库客户端
```

---

# 13. `@ConditionalOnMissingClass`

和 `@ConditionalOnClass` 相反。

```java
@Configuration
@ConditionalOnMissingClass("com.example.RealClient")
public class FallbackConfiguration {

    @Bean
    public FallbackClient fallbackClient() {
        return new FallbackClient();
    }
}
```

含义：

```text
某个类不存在时，启用 fallback 配置
```

这个用得比 `@ConditionalOnClass` 少，但在兼容不同版本依赖时有用。

---

# 14. `@ConditionalOnResource`

含义：

> **某个资源文件存在时，才创建 Bean。**

示例：

```java
@Bean
@ConditionalOnResource(resources = "classpath:prompt/system-prompt.md")
public PromptTemplate promptTemplate() {
    return new PromptTemplate("prompt/system-prompt.md");
}
```

适合：

```text
证书文件存在才启用 HTTPS 客户端
模板文件存在才启用模板渲染
本地模型文件存在才启用本地推理
prompt 文件存在才启用某个 AI 能力
```

---

# 15. `@ConditionalOnWebApplication`

含义：

> **当前应用是 Web 应用时才生效。**

例如：

```java
@Configuration
@ConditionalOnWebApplication
public class WebConfiguration {

    @Bean
    public HandlerInterceptor authInterceptor() {
        return new AuthInterceptor();
    }
}
```

适合：

```text
Controller
Filter
Interceptor
WebMvcConfigurer
WebFlux 配置
异常处理器
```

非 Web 项目，比如命令行批处理、MQ Worker，就不应该加载这些 Bean。

---

# 16. `@ConditionalOnNotWebApplication`

含义相反：

```java
@Configuration
@ConditionalOnNotWebApplication
public class CliConfiguration {

    @Bean
    public CommandLineTask commandLineTask() {
        return new CommandLineTask();
    }
}
```

适合：

```text
命令行应用
批处理任务
Worker 程序
非 Web 消费者程序
```

---

# 17. `@ConditionalOnExpression`

根据 SpEL 表达式判断。

```java
@Bean
@ConditionalOnExpression("'${ai.provider}' == 'deepseek' && '${ai.enabled}' == 'true'")
public LlmClient deepSeekClient() {
    return new DeepSeekClient();
}
```

这个不太推荐大量使用。

原因：

```text
字符串表达式不利于重构
复杂后可读性差
IDE 支持弱
容易写出难维护的条件逻辑
```

简单判断可以用。

复杂判断更推荐：

```text
@ConditionalOnProperty
自定义 Condition
```

---

# 18. `@Profile` 算不算 Condition？

从使用效果看，它就是一种条件装配机制。

```java
@Service
@Profile("dev")
public class MockPaymentClient implements PaymentClient {
}
```

```java
@Service
@Profile("prod")
public class RealPaymentClient implements PaymentClient {
}
```

配置：

```yaml
spring:
  profiles:
    active: dev
```

结果：

```text
dev  -> MockPaymentClient 注册
prod -> RealPaymentClient 注册
```

---

## `@Profile` 和 `@ConditionalOnProperty` 的区别

|对比|`@Profile`|`@ConditionalOnProperty`|
|---|---|---|
|判断依据|当前环境|具体配置项|
|粒度|环境级别|功能级别|
|常见值|`dev` / `test` / `prod`|`xxx.enabled=true`|
|适合|环境隔离|功能开关|
|灵活性|较粗|更细|

例如：

```java
@Profile("prod")
```

表示：

```text
生产环境才启用
```

而：

```java
@ConditionalOnProperty(prefix = "sms", name = "enabled", havingValue = "true")
```

表示：

```text
短信功能开关打开才启用
```

两者可以组合：

```java
@Bean
@Profile("prod")
@ConditionalOnProperty(prefix = "sms", name = "enabled", havingValue = "true")
public SmsSender smsSender() {
    return new SmsSender();
}
```

含义：

```text
生产环境 + sms.enabled=true
才创建 SmsSender
```

---

# 19. 多个 Condition 叠加是什么关系？

比如：

```java
@Bean
@ConditionalOnClass(RedissonClient.class)
@ConditionalOnProperty(prefix = "lock.redis", name = "enabled", havingValue = "true")
@ConditionalOnMissingBean(LockClient.class)
public LockClient lockClient(RedissonClient redissonClient) {
    return new RedissonLockClient(redissonClient);
}
```

这几个条件是：

```text
AND 关系
```

也就是全部满足才注册：

```text
1. classpath 中存在 RedissonClient
2. lock.redis.enabled=true
3. 容器里还没有 LockClient
```

只要一个不满足，这个 Bean 就不会注册。

---

# 20. Condition 和 `@Primary` / `@Qualifier` 的区别

这几个经常被混淆。

## 20.1 Condition 解决“有没有”

```java
@Bean
@ConditionalOnProperty(prefix = "ai", name = "provider", havingValue = "deepseek")
public LlmClient deepSeekClient() {
    return new DeepSeekClient();
}
```

它决定：

```text
deepSeekClient 是否注册
```

---

## 20.2 `@Primary` 解决“默认选谁”

```java
@Bean
@Primary
public LlmClient deepSeekClient() {
    return new DeepSeekClient();
}
```

它的前提是：

```text
多个 LlmClient 都已经存在
```

然后决定：

```text
默认注入哪个
```

---

## 20.3 `@Qualifier` 解决“明确选谁”

```java
public ChatService(@Qualifier("openAiClient") LlmClient llmClient) {
    this.llmClient = llmClient;
}
```

它也是在多个 Bean 都存在时，精确指定一个。

---

## 20.4 总结

|机制|解决问题|
|---|---|
|`@Conditional`|这个 Bean 要不要注册|
|`@Primary`|多个 Bean 都存在时，默认选谁|
|`@Qualifier`|多个 Bean 都存在时，明确选谁|
|`Map<String, T>`|多个 Bean 都存在时，运行时按 key 选谁|

---

# 21. Condition 和策略模式的区别

这个也很重要。

## 21.1 Condition：启动时选择

```yaml
ai:
  provider: deepseek
```

```java
@Bean
@ConditionalOnProperty(prefix = "ai", name = "provider", havingValue = "deepseek")
public LlmClient deepSeekClient() {
    return new DeepSeekClient();
}
```

效果：

```text
应用启动后，容器里只有 DeepSeekClient
```

适合：

```text
这个部署实例固定只用 deepseek
这个环境固定只用 redis
这个服务固定只启用某个模块
```

---

## 21.2 Map 策略：运行时选择

```java
@Service
@RequiredArgsConstructor
public class LlmRouter {

    private final Map<String, LlmClient> llmClientMap;

    public ChatResponse chat(String provider, ChatRequest request) {
        LlmClient client = llmClientMap.get(provider);
        return client.chat(request);
    }
}
```

效果：

```text
容器里同时存在 openai / deepseek / qwen
每次请求根据 provider 参数选择一个
```

适合：

```text
这个用户用 openai
那个用户用 deepseek
这次请求用 qwen
下一次请求用 claude
```

---

## 21.3 判断标准

|场景|用什么|
|---|---|
|启动时固定选择一个实现|Condition|
|运行时根据请求选择实现|Map 策略|
|某个能力关闭后 Bean 不应存在|Condition|
|多个能力都存在，只是按 key 分发|Map 策略|
|做 Starter 自动配置|Condition|
|做业务策略路由|Map 注入|

---

# 22. 实战案例一：短信模块开关

## 配置

```yaml
sms:
  enabled: true
  provider: aliyun
```

## Properties

```java
@ConfigurationProperties(prefix = "sms")
public class SmsProperties {

    private boolean enabled;
    private String provider;

    // getter setter
}
```

## 配置类

```java
@Configuration
@EnableConfigurationProperties(SmsProperties.class)
@ConditionalOnProperty(prefix = "sms", name = "enabled", havingValue = "true")
public class SmsAutoConfiguration {

    @Bean
    @ConditionalOnProperty(prefix = "sms", name = "provider", havingValue = "aliyun")
    public SmsSender aliyunSmsSender(SmsProperties properties) {
        return new AliyunSmsSender(properties);
    }

    @Bean
    @ConditionalOnProperty(prefix = "sms", name = "provider", havingValue = "tencent")
    public SmsSender tencentSmsSender(SmsProperties properties) {
        return new TencentSmsSender(properties);
    }

    @Bean
    @ConditionalOnBean(SmsSender.class)
    public SmsService smsService(SmsSender smsSender) {
        return new SmsService(smsSender);
    }
}
```

效果：

```text
sms.enabled=false
    -> 整个短信模块不启用

sms.enabled=true + sms.provider=aliyun
    -> 创建 AliyunSmsSender + SmsService

sms.enabled=true + sms.provider=tencent
    -> 创建 TencentSmsSender + SmsService
```

---

# 23. 实战案例二：AI Client 自动装配

## 配置

```yaml
ai:
  enabled: true
  provider: deepseek
  deepseek:
    api-key: sk-xxx
  openai:
    api-key: sk-xxx
```

## 接口

```java
public interface LlmClient {

    ChatResponse chat(ChatRequest request);
}
```

## DeepSeek 配置

```java
@Configuration
@ConditionalOnProperty(prefix = "ai", name = "provider", havingValue = "deepseek")
public class DeepSeekConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public LlmClient deepSeekClient(AiProperties properties) {
        return new DeepSeekClient(properties.getDeepseek().getApiKey());
    }
}
```

## OpenAI 配置

```java
@Configuration
@ConditionalOnProperty(prefix = "ai", name = "provider", havingValue = "openai")
public class OpenAiConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public LlmClient openAiClient(AiProperties properties) {
        return new OpenAiClient(properties.getOpenai().getApiKey());
    }
}
```

## 上层服务

```java
@Configuration
@ConditionalOnProperty(prefix = "ai", name = "enabled", havingValue = "true")
public class AiServiceConfiguration {

    @Bean
    @ConditionalOnBean(LlmClient.class)
    public ChatService chatService(LlmClient llmClient) {
        return new ChatService(llmClient);
    }
}
```

这个结构的好处是：

```text
AI 总开关控制整个模块
provider 控制具体实现
上层 ChatService 只依赖 LlmClient 抽象
业务方可以覆盖默认 LlmClient
```

---

# 24. 实战案例三：Starter 自动配置标准写法

假设你要做一个：

```text
dendro-vector-spring-boot-starter
```

功能：

```text
项目引入 Milvus SDK 时，自动配置 MilvusVectorStore
业务方自己定义 VectorStore 时，不创建默认实现
配置关闭时，整个向量能力不启用
```

标准写法：

```java
@AutoConfiguration
@ConditionalOnClass(name = "io.milvus.client.MilvusServiceClient")
@ConditionalOnProperty(
        prefix = "dendro.vector",
        name = "enabled",
        havingValue = "true",
        matchIfMissing = true
)
@EnableConfigurationProperties(VectorProperties.class)
public class VectorAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public VectorStore vectorStore(VectorProperties properties) {
        return new MilvusVectorStore(properties);
    }

    @Bean
    @ConditionalOnBean(VectorStore.class)
    @ConditionalOnMissingBean
    public VectorSearchService vectorSearchService(VectorStore vectorStore) {
        return new VectorSearchService(vectorStore);
    }
}
```

这个写法背后的判断链：

```text
1. classpath 中有 Milvus SDK
2. dendro.vector.enabled=true 或者没配置但默认启用
3. 容器里没有业务方自定义 VectorStore
        ↓
创建默认 MilvusVectorStore

4. 容器里存在 VectorStore
5. 容器里没有业务方自定义 VectorSearchService
        ↓
创建默认 VectorSearchService
```

这就是 Spring Boot Starter 的典型 Condition 用法。

---

# 25. 自定义 Condition 的真实适用场景

大多数业务项目里，你不需要自己实现 `Condition`。

优先级应该是：

```text
@ConditionalOnProperty
@ConditionalOnBean
@ConditionalOnMissingBean
@ConditionalOnClass
@Profile
```

只有下面这些情况，才值得自定义 Condition：

```text
多个配置组合判断
复杂 OR 条件
根据注解参数动态判断
根据资源文件和配置共同判断
根据 classpath 版本判断
根据操作系统判断
根据某个模块注册状态判断
```

例如：

```text
只有在：
features.chat=true
且 ai.provider 不为空
且 prompt/system.md 存在
时，才启用 ChatAgent
```

这时可以自定义。

---

# 26. 自定义 Condition 的注意事项

## 26.1 不要获取 Bean 实例

不要这样：

```java
public class BadCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        MyService myService = context.getBeanFactory().getBean(MyService.class);
        return myService.isEnabled();
    }
}
```

原因：

```text
Condition 执行时 Bean 可能还没创建
强行 getBean 可能触发提前初始化
容易制造循环依赖和启动顺序问题
```

Condition 应该判断：

```text
配置
环境
classpath
资源
BeanDefinition
注解元数据
```

不应该依赖：

```text
运行时 Bean 实例
数据库查询
远程接口调用
复杂业务状态
```

---

## 26.2 不要做远程调用

不要这样：

```java
public boolean matches(...) {
    return licenseServer.check();
}
```

问题：

```text
拖慢启动
网络抖动导致应用无法启动
条件判断有副作用
排查困难
```

Condition 应该是：

```text
轻量
确定
无副作用
本地可判断
```

---

## 26.3 不要写业务逻辑

不要把这些塞进 Condition：

```text
订单金额是否大于 100
用户是不是 VIP
库存是否充足
这次请求要不要走风控
```

这些是业务运行时逻辑。

Condition 判断的是：

```text
组件是否应该存在
```

不是：

```text
某次请求怎么处理
```

---

# 27. `@ConditionalOnProperty` 的常见坑

## 27.1 配置名写错

代码：

```java
@ConditionalOnProperty(prefix = "ai.chat", name = "enabled", havingValue = "true")
```

配置却写成：

```yaml
ai:
  chat:
    enable: true
```

结果：

```text
Bean 不创建
```

因为代码要的是：

```text
ai.chat.enabled
```

你写的是：

```text
ai.chat.enable
```

---

## 27.2 `matchIfMissing` 用错

```java
@ConditionalOnProperty(
        prefix = "sms",
        name = "enabled",
        havingValue = "true",
        matchIfMissing = true
)
```

含义：

```text
配置缺失时，也认为条件成立
```

这对一些默认启用的基础组件可以。

但是对这些功能要谨慎：

```text
发短信
发邮件
扣款
定时任务
MQ 消费者
真实外部接口调用
```

这些通常建议默认关闭：

```java
matchIfMissing = false
```

---

## 27.3 布尔值判断要统一

建议配置统一写：

```yaml
feature:
  enabled: true
```

不要混用：

```yaml
feature:
  enable: yes
  open: 1
  switch: on
```

否则项目越大越混乱。

---

# 28. `@ConditionalOnMissingBean` 的常见坑

## 28.1 默认 Bean 覆盖不了

错误写法：

```java
@Bean
public CacheService cacheService() {
    return new LocalCacheService();
}
```

如果这是一个框架默认配置，业务方想覆盖就比较难。

更好的写法：

```java
@Bean
@ConditionalOnMissingBean
public CacheService cacheService() {
    return new LocalCacheService();
}
```

---

## 28.2 Bean 类型判断不清楚

比如：

```java
@Bean
@ConditionalOnMissingBean
public LlmClient deepSeekClient() {
    return new DeepSeekClient();
}
```

如果容器里已经有任何一个 `LlmClient`，这个 Bean 就不会注册。

有时候你想判断的是具体类型：

```java
@Bean
@ConditionalOnMissingBean(DeepSeekClient.class)
public DeepSeekClient deepSeekClient() {
    return new DeepSeekClient();
}
```

两者含义不同：

|写法|含义|
|---|---|
|`@ConditionalOnMissingBean` 放在返回 `LlmClient` 的方法上|没有任何 `LlmClient` 才创建|
|`@ConditionalOnMissingBean(DeepSeekClient.class)`|没有 `DeepSeekClient` 才创建|

---

# 29. 推荐工程规范

## 29.1 普通业务项目

优先使用：

```text
@ConditionalOnProperty
@Profile
@ConditionalOnBean
```

典型写法：

```java
@Configuration
@ConditionalOnProperty(prefix = "payment", name = "enabled", havingValue = "true")
public class PaymentConfiguration {

    @Bean
    public PaymentService paymentService(PaymentClient paymentClient) {
        return new PaymentService(paymentClient);
    }
}
```

---

## 29.2 组件 / Starter 项目

优先使用：

```text
@ConditionalOnClass
@ConditionalOnProperty
@ConditionalOnMissingBean
@ConditionalOnBean
@EnableConfigurationProperties
```

典型写法：

```java
@AutoConfiguration
@ConditionalOnClass(SomeSdkClient.class)
@EnableConfigurationProperties(SomeProperties.class)
public class SomeAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public SomeClient someClient(SomeProperties properties) {
        return new SomeClient(properties);
    }

    @Bean
    @ConditionalOnBean(SomeClient.class)
    @ConditionalOnMissingBean
    public SomeService someService(SomeClient someClient) {
        return new SomeService(someClient);
    }
}
```

---

## 29.3 不建议

不建议到处在业务类上写：

```java
@Service
@Conditional(...)
public class XxxService {
}
```

更推荐：

```java
@Configuration
public class XxxConfiguration {

    @Bean
    @ConditionalOnProperty(...)
    public XxxService xxxService() {
        return new XxxService();
    }
}
```

原因：

```text
条件装配集中管理
组件边界更清楚
配置类更适合表达装配逻辑
业务类更纯净
```

---

# 30. 最终心智模型

你可以把 Condition 体系分成三层：

```text
第一层：底层接口
Condition.matches()

第二层：基础注解
@Conditional

第三层：Spring Boot 封装注解
@ConditionalOnProperty
@ConditionalOnBean
@ConditionalOnMissingBean
@ConditionalOnClass
@ConditionalOnResource
@ConditionalOnWebApplication
@Profile
```

实际开发时，按这个顺序思考：

```text
1. 这是启动时组件是否存在的问题吗？
   是 -> 可以考虑 Condition
   否 -> 不要用 Condition

2. 能不能用 Spring Boot 现成注解？
   能 -> 不要自定义 Condition

3. 是配置开关吗？
   用 @ConditionalOnProperty

4. 是默认实现吗？
   用 @ConditionalOnMissingBean

5. 是依赖某个 Bean 吗？
   用 @ConditionalOnBean

6. 是依赖某个 jar 包吗？
   用 @ConditionalOnClass

7. 是环境隔离吗？
   用 @Profile

8. 是复杂组合条件吗？
   再自定义 Condition
```

一句话总结：

> **Condition 是 Spring 的“组件装配条件系统”。它决定 Bean 是否进入容器；不是处理业务流程，而是处理应用启动时的组件结构。**