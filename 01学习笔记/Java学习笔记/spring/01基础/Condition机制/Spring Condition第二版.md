[[Spring Condition第一版]]
要求简化成了 概述+（概念+用法）+总结 的形式，还是很难读啊
[[ChatGPT生成内容更具可读性]]
[[Spring Condition第三版]]
# Spring Condition 相关的实际用法

## 概述

`Condition` 可以理解成：

> **Spring 启动时的 Bean 加载条件。**

它解决的问题是：

```text
这个 Bean / 配置类 / 自动配置类，要不要注册进 Spring 容器？
```

它不是业务代码里的 `if-else`。

对比一下：

```text
业务 if-else：一次请求来了，代码怎么走？
Condition：应用启动时，哪些组件要加载？
```

Spring 原生提供了底层注解 `@Conditional`，它表示：只有指定条件全部满足时，组件才有资格被注册。Spring 官方文档也说明，`@Conditional` 可以用于组件注册前的条件判断。([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Conditional.html?utm_source=chatgpt.com "Conditional (Spring Framework 7.0.5 API)"))

Spring Boot 又在 `@Conditional` 基础上封装了一批更常用的注解：

```text
@ConditionalOnProperty
@ConditionalOnMissingBean
@ConditionalOnBean
@ConditionalOnClass
@ConditionalOnMissingClass
@ConditionalOnResource
@ConditionalOnWebApplication
```

实际业务开发里，不需要一上来就自己实现 `Condition` 接口。你先掌握下面几种用法就够用了。

---

# 用法 1：`@ConditionalOnProperty` —— 根据配置开关加载 Bean

## 概念

这是最常用的 Condition 注解。

它的作用是：

> **根据 `application.yml` / `application.properties` 里的配置，决定 Bean 是否创建。**

适合：

```text
短信开关
AI 功能开关
定时任务开关
MQ 消费开关
支付渠道开关
监控组件开关
```

Spring Boot 文档也说明，`@ConditionalOnProperty` 可以根据 Spring Environment 里的属性决定配置是否生效。([Home](https://docs.spring.io/spring-boot/docs/3.2.5/reference/htmlsingle/?utm_source=chatgpt.com "Spring Boot Reference Documentation"))

---

## 代码案例：短信功能开关

配置：

```yaml
sms:
  enabled: true
```

短信发送器：

```java
public class SmsSender {

    public void send(String phone, String content) {
        System.out.println("发送短信：" + phone + "，内容：" + content);
    }
}
```

配置类：

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

使用：

```java
@Service
@RequiredArgsConstructor
public class SmsService {

    private final SmsSender smsSender;

    public void sendLoginCode(String phone) {
        smsSender.send(phone, "你的验证码是 123456");
    }
}
```

效果：

```text
sms.enabled=true
    -> SmsSender 被注册进 Spring 容器

sms.enabled=false
    -> SmsSender 不会注册进 Spring 容器
```

---

## 关键参数

```java
@ConditionalOnProperty(
        prefix = "sms",
        name = "enabled",
        havingValue = "true",
        matchIfMissing = false
)
```

|参数|含义|
|---|---|
|`prefix`|配置前缀|
|`name`|配置项名称|
|`havingValue`|期望值|
|`matchIfMissing`|配置缺失时是否也匹配|

一般建议：

```java
matchIfMissing = false
```

尤其是这些功能：

```text
发短信
发邮件
扣款
真实支付
MQ 消费
定时任务
外部接口调用
```

不要默认开启，避免生产环境误触发。

---

# 用法 2：`@ConditionalOnMissingBean` —— 用户没配，我给默认实现

## 概念

它的作用是：

> **当 Spring 容器里没有某个 Bean 时，才创建当前 Bean。**

这是 Spring Boot Starter 里非常核心的设计思想。

可以理解成：

```text
你没配置，我帮你默认配置一个。
你自己配置了，我尊重你的配置。
```

Spring Boot 官方文档也提到，`@ConditionalOnMissingBean` 常用于自动配置，允许用户自定义 Bean 覆盖默认配置。([Home](https://docs.spring.io/spring-boot/docs/3.2.5/reference/htmlsingle/?utm_source=chatgpt.com "Spring Boot Reference Documentation"))

---

## 代码案例：默认缓存实现

定义接口：

```java
public interface CacheService {

    void put(String key, String value);

    String get(String key);
}
```

默认本地缓存实现：

```java
public class LocalCacheService implements CacheService {

    private final Map<String, String> cache = new ConcurrentHashMap<>();

    @Override
    public void put(String key, String value) {
        cache.put(key, value);
    }

    @Override
    public String get(String key) {
        return cache.get(key);
    }
}
```

自动配置类：

```java
@Configuration
public class CacheAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(CacheService.class)
    public CacheService cacheService() {
        return new LocalCacheService();
    }
}
```

业务方如果没有自己定义 `CacheService`，Spring 就使用 `LocalCacheService`。

如果业务方自己定义了：

```java
@Configuration
public class MyCacheConfiguration {

    @Bean
    public CacheService cacheService(RedisTemplate<String, String> redisTemplate) {
        return new RedisCacheService(redisTemplate);
    }
}
```

那么：

```text
LocalCacheService 不会创建
RedisCacheService 会生效
```

---

## 实际意义

这类写法特别适合你以后写自己的组件，比如：

```text
dendro-ai-spring-boot-starter
dendro-vector-spring-boot-starter
dendro-storage-spring-boot-starter
```

例如：

```java
@Bean
@ConditionalOnMissingBean
public LlmClient llmClient() {
    return new DefaultLlmClient();
}
```

意思是：

```text
业务项目没配置 LlmClient，我给默认实现。
业务项目自己配置了 LlmClient，我就不插手。
```

这就是 Starter 友好的地方。

---

# 用法 3：`@ConditionalOnBean` —— 有 A，才创建 B

## 概念

它的作用是：

> **只有容器中已经存在某个 Bean 时，才创建当前 Bean。**

适合表达依赖关系：

```text
有 RedisClient，才创建 RedisCacheService
有 LlmClient，才创建 ChatService
有 DataSource，才创建某些 Repository
有 MQ Producer，才创建 MessagePublisher
```

Spring Boot 文档说明，`@ConditionalOnBean` 和 `@ConditionalOnMissingBean` 可以根据 Bean 是否存在来决定是否包含某个 Bean。([Home](https://docs.spring.io/spring-boot/docs/1.4.x/reference/html/boot-features-developing-auto-configuration.html?utm_source=chatgpt.com "43. Creating your own auto-configuration - Spring"))

---

## 代码案例：有 LlmClient 才创建 ChatService

接口：

```java
public interface LlmClient {

    String chat(String prompt);
}
```

实现：

```java
public class DeepSeekLlmClient implements LlmClient {

    @Override
    public String chat(String prompt) {
        return "DeepSeek response: " + prompt;
    }
}
```

配置：

```java
@Configuration
public class AiClientConfiguration {

    @Bean
    @ConditionalOnProperty(prefix = "ai", name = "enabled", havingValue = "true")
    public LlmClient llmClient() {
        return new DeepSeekLlmClient();
    }
}
```

上层服务：

```java
public class ChatService {

    private final LlmClient llmClient;

    public ChatService(LlmClient llmClient) {
        this.llmClient = llmClient;
    }

    public String chat(String message) {
        return llmClient.chat(message);
    }
}
```

配置类：

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

效果：

```text
ai.enabled=true
    -> 创建 LlmClient
    -> 创建 ChatService

ai.enabled=false
    -> 不创建 LlmClient
    -> ChatService 也不会创建
```

---

## 实际意义

这能避免很多启动错误。

如果你直接写：

```java
@Bean
public ChatService chatService(LlmClient llmClient) {
    return new ChatService(llmClient);
}
```

但是 `LlmClient` 没有创建，Spring 启动时就会报错。

加上：

```java
@ConditionalOnBean(LlmClient.class)
```

就表示：

```text
底层依赖存在，我才创建上层服务。
```

---

# 用法 4：`@ConditionalOnClass` —— 引了某个 jar，才启用配置

## 概念

它的作用是：

> **classpath 中存在某个类时，才加载某个配置。**

换成人话：

```text
项目引入了某个依赖，Spring Boot 才帮你自动配置对应组件。
```

这是 Spring Boot 自动配置最典型的机制之一。

比如：

```text
引入 spring-boot-starter-web
    -> 自动配置 WebMVC

引入 spring-boot-starter-data-redis
    -> 自动配置 RedisTemplate

引入 mybatis-spring-boot-starter
    -> 自动配置 MyBatis 相关组件
```

Spring Boot 文档也说明，`@ConditionalOnClass` / `@ConditionalOnMissingClass` 可以根据指定类是否存在来决定配置是否生效。([Home](https://docs.spring.io/spring-boot/docs/1.4.x/reference/html/boot-features-developing-auto-configuration.html?utm_source=chatgpt.com "43. Creating your own auto-configuration - Spring"))

---

## 代码案例：引入 Redisson 才配置分布式锁

定义锁接口：

```java
public interface LockClient {

    void lock(String key);

    void unlock(String key);
}
```

Redisson 实现：

```java
public class RedissonLockClient implements LockClient {

    private final RedissonClient redissonClient;

    public RedissonLockClient(RedissonClient redissonClient) {
        this.redissonClient = redissonClient;
    }

    @Override
    public void lock(String key) {
        redissonClient.getLock(key).lock();
    }

    @Override
    public void unlock(String key) {
        redissonClient.getLock(key).unlock();
    }
}
```

自动配置：

```java
@Configuration
@ConditionalOnClass(RedissonClient.class)
public class LockAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public LockClient lockClient(RedissonClient redissonClient) {
        return new RedissonLockClient(redissonClient);
    }
}
```

效果：

```text
项目引入 Redisson
    -> RedissonClient 类存在
    -> LockAutoConfiguration 生效

项目没引入 Redisson
    -> RedissonClient 类不存在
    -> LockAutoConfiguration 不生效
```

---

## 实际意义

`@ConditionalOnClass` 特别适合写 Starter。

比如你写一个 AI Starter：

```java
@Configuration
@ConditionalOnClass(name = "dev.langchain4j.model.chat.ChatLanguageModel")
public class LangChain4jAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public AiChatService aiChatService() {
        return new AiChatService();
    }
}
```

意思是：

```text
用户项目里引入了 LangChain4j，我才自动配置相关能力。
没引入，我就不配置，避免 ClassNotFoundException。
```

---

# 用法 5：`@ConditionalOnMissingClass` —— 没有某个类时，启用降级方案

## 概念

它和 `@ConditionalOnClass` 相反。

作用是：

> **classpath 中不存在某个类时，才启用当前配置。**

这个用得少一些，但在兼容不同依赖版本、提供 fallback 实现时有用。

---

## 代码案例：没有 Redis 时使用本地缓存

本地缓存：

```java
public class LocalCacheService implements CacheService {

    private final Map<String, String> cache = new ConcurrentHashMap<>();

    @Override
    public void put(String key, String value) {
        cache.put(key, value);
    }

    @Override
    public String get(String key) {
        return cache.get(key);
    }
}
```

配置：

```java
@Configuration
@ConditionalOnMissingClass("org.springframework.data.redis.core.RedisTemplate")
public class LocalCacheConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public CacheService cacheService() {
        return new LocalCacheService();
    }
}
```

效果：

```text
项目没有引入 RedisTemplate
    -> 使用 LocalCacheService

项目引入了 RedisTemplate
    -> 这个本地缓存配置不生效
```

---

# 用法 6：`@ConditionalOnResource` —— 某个资源文件存在才启用

## 概念

作用是：

> **classpath 或指定位置存在某个资源文件时，才注册 Bean。**

适合：

```text
证书文件存在才启用 HTTPS Client
prompt 文件存在才启用 AI Prompt 模板
模板文件存在才启用邮件模板服务
规则文件存在才启用规则引擎
```

---

## 代码案例：存在 prompt 文件才创建 PromptTemplate

资源文件：

```text
src/main/resources/prompts/system-prompt.md
```

模板类：

```java
public class PromptTemplate {

    private final String resourcePath;

    public PromptTemplate(String resourcePath) {
        this.resourcePath = resourcePath;
    }

    public String getResourcePath() {
        return resourcePath;
    }
}
```

配置类：

```java
@Configuration
public class PromptConfiguration {

    @Bean
    @ConditionalOnResource(resources = "classpath:prompts/system-prompt.md")
    public PromptTemplate systemPromptTemplate() {
        return new PromptTemplate("classpath:prompts/system-prompt.md");
    }
}
```

效果：

```text
system-prompt.md 存在
    -> 创建 PromptTemplate

system-prompt.md 不存在
    -> 不创建 PromptTemplate
```

---

# 用法 7：`@Profile` —— 按环境加载 Bean

## 概念

`@Profile` 不是 Spring Boot 的 `@ConditionalOnXxx`，但它本质上也是一种条件装配。

作用是：

> **根据当前激活的环境，决定 Bean 是否注册。**

适合：

```text
dev 环境用 Mock
test 环境用测试实现
prod 环境用真实实现
```

---

## 代码案例：开发环境 Mock 支付，生产环境真实支付

接口：

```java
public interface PaymentClient {

    void pay(String orderId, BigDecimal amount);
}
```

开发环境实现：

```java
@Service
@Profile("dev")
public class MockPaymentClient implements PaymentClient {

    @Override
    public void pay(String orderId, BigDecimal amount) {
        System.out.println("Mock 支付：" + orderId);
    }
}
```

生产环境实现：

```java
@Service
@Profile("prod")
public class RealPaymentClient implements PaymentClient {

    @Override
    public void pay(String orderId, BigDecimal amount) {
        System.out.println("真实支付：" + orderId);
    }
}
```

配置：

```yaml
spring:
  profiles:
    active: dev
```

效果：

```text
dev 环境
    -> MockPaymentClient 注册

prod 环境
    -> RealPaymentClient 注册
```

---

## `@Profile` 和 `@ConditionalOnProperty` 的区别

|对比|`@Profile`|`@ConditionalOnProperty`|
|---|---|---|
|判断依据|当前环境|具体配置项|
|粒度|较粗|更细|
|常见用途|dev/test/prod 切换|某个功能开关|
|例子|`@Profile("prod")`|`sms.enabled=true`|

简单理解：

```text
环境隔离：用 @Profile
功能开关：用 @ConditionalOnProperty
```

---

# 用法 8：自定义 `Condition` —— 复杂条件自己写

## 概念

前面那些注解都是 Spring Boot 封装好的。

如果条件太复杂，现成注解不够用，就可以自己实现 `Condition` 接口。

底层接口长这样：

```java
public interface Condition {

    boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
}
```

核心就是：

```text
matches 返回 true  -> Bean 注册
matches 返回 false -> Bean 不注册
```

Spring Framework 文档中说明，`Condition` 是在 BeanDefinition 注册前判断的条件，并且 Condition 不应该和 Bean 实例交互。([Home](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/annotation/Condition.html?utm_source=chatgpt.com "Condition (Spring Framework 7.0.6 API)"))

---

## 代码案例：只有 Windows 环境才启用某个 Bean

Condition：

```java
public class WindowsCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String osName = context.getEnvironment().getProperty("os.name");
        return osName != null && osName.toLowerCase().contains("windows");
    }
}
```

配置类：

```java
@Configuration
public class SystemConfiguration {

    @Bean
    @Conditional(WindowsCondition.class)
    public FilePathService windowsFilePathService() {
        return new WindowsFilePathService();
    }
}
```

效果：

```text
Windows 系统
    -> windowsFilePathService 注册

非 Windows 系统
    -> windowsFilePathService 不注册
```

---

## 再来一个更贴近业务的例子

需求：

```text
只有同时满足：
1. ai.enabled=true
2. ai.provider=deepseek
3. 当前环境不是 test

才创建 DeepSeekClient。
```

Condition：

```java
public class DeepSeekEnabledCondition implements Condition {

    @Override
    public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        Environment env = context.getEnvironment();

        Boolean enabled = env.getProperty("ai.enabled", Boolean.class, false);
        String provider = env.getProperty("ai.provider", "");
        List<String> activeProfiles = Arrays.asList(env.getActiveProfiles());

        return enabled
                && "deepseek".equalsIgnoreCase(provider)
                && !activeProfiles.contains("test");
    }
}
```

配置：

```java
@Configuration
public class AiConfiguration {

    @Bean
    @Conditional(DeepSeekEnabledCondition.class)
    public LlmClient deepSeekClient() {
        return new DeepSeekClient();
    }
}
```

这个就比单纯的 `@ConditionalOnProperty` 更灵活。

---

# 用法 9：自定义条件注解 —— 把复杂 Condition 包装得更好用

## 概念

直接写：

```java
@Conditional(DeepSeekEnabledCondition.class)
```

能用，但语义不够好。

可以封装成自己的注解：

```java
@ConditionalOnDeepSeekEnabled
```

这样更像 Spring Boot 的风格。

---

## 代码案例

自定义注解：

```java
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Conditional(DeepSeekEnabledCondition.class)
public @interface ConditionalOnDeepSeekEnabled {
}
```

使用：

```java
@Configuration
public class AiConfiguration {

    @Bean
    @ConditionalOnDeepSeekEnabled
    public LlmClient deepSeekClient() {
        return new DeepSeekClient();
    }
}
```

这样业务代码可读性更好：

```text
@ConditionalOnDeepSeekEnabled
= DeepSeek 开启时才注册
```

---

# 用法 10：组合多个 Condition 注解

## 概念

多个条件注解叠加时，通常可以理解成：

```text
AND 关系
```

也就是全部满足才生效。

---

## 代码案例：Redis 分布式锁自动配置

```java
@Configuration
@ConditionalOnClass(RedissonClient.class)
@ConditionalOnProperty(prefix = "lock.redis", name = "enabled", havingValue = "true")
public class RedisLockAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(LockClient.class)
    public LockClient lockClient(RedissonClient redissonClient) {
        return new RedissonLockClient(redissonClient);
    }
}
```

这段配置生效需要同时满足：

```text
1. 项目引入了 Redisson
2. lock.redis.enabled=true
3. 容器里还没有 LockClient
```

只要有一个不满足，`LockClient` 就不会创建。

---

# 实战判断：到底该用哪个？

## 1. 配置开关

用：

```java
@ConditionalOnProperty
```

例如：

```text
ai.enabled=true
sms.enabled=true
job.enabled=true
```

---

## 2. 默认实现

用：

```java
@ConditionalOnMissingBean
```

例如：

```text
用户没定义 CacheService，我给 LocalCacheService
用户没定义 LlmClient，我给 DefaultLlmClient
```

---

## 3. 依赖某个 Bean

用：

```java
@ConditionalOnBean
```

例如：

```text
有 LlmClient，才创建 ChatService
有 RedisClient，才创建 CacheService
```

---

## 4. 依赖某个 jar

用：

```java
@ConditionalOnClass
```

例如：

```text
引入 Redisson，才启用分布式锁配置
引入 Milvus SDK，才启用向量库配置
```

---

## 5. 环境隔离

用：

```java
@Profile
```

例如：

```text
dev 用 Mock
prod 用真实实现
```

---

## 6. 复杂判断

用：

```java
自定义 Condition
```

例如：

```text
配置 + profile + 操作系统 + 资源文件 综合判断
```

---

# Condition 和策略模式的区别

这个一定要分清。

## Condition：启动时决定有没有

```java
@Bean
@ConditionalOnProperty(prefix = "ai", name = "provider", havingValue = "deepseek")
public LlmClient deepSeekClient() {
    return new DeepSeekClient();
}
```

意思是：

```text
应用启动时决定容器里有没有 DeepSeekClient。
```

适合：

```text
这个部署环境固定使用 deepseek
这个服务固定不开启 sms
这个实例固定使用 redis
```

---

## 策略模式：运行时决定用哪个

```java
@Service
@RequiredArgsConstructor
public class LlmRouter {

    private final Map<String, LlmClient> llmClientMap;

    public String chat(String provider, String prompt) {
        return llmClientMap.get(provider).chat(prompt);
    }
}
```

意思是：

```text
容器里同时存在 openai / deepseek / qwen。
每次请求来了，再根据 provider 选择一个。
```

适合：

```text
这个用户用 openai
那个用户用 deepseek
这个请求用 qwen
那个请求用 claude
```

---

## 判断标准

|场景|用什么|
|---|---|
|启动时决定组件是否存在|Condition|
|运行时根据参数选择实现|策略模式 / Map 注入|
|功能开关|Condition|
|多渠道动态路由|策略模式|
|Starter 自动配置|Condition|
|业务分发|策略模式|

---

# 使用 Condition 的几个注意点

## 1. 不要在 Condition 里写业务逻辑

不要判断：

```text
用户是不是 VIP
订单金额是否大于 100
库存是否充足
优惠券是否可用
```

这些是业务逻辑，不是组件装配逻辑。

Condition 判断的是：

```text
这个组件要不要进入 Spring 容器
```

---

## 2. 不要在 Condition 里拿 Bean 实例

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
Condition 执行时，很多 Bean 还没创建。
你强行 getBean，容易引发提前初始化、循环依赖、启动顺序混乱。
```

Condition 里适合判断：

```text
配置
环境变量
classpath
资源文件
profile
注解元数据
```

不适合判断：

```text
数据库状态
远程接口结果
Bean 实例内部状态
一次业务请求的数据
```

---

## 3. `matchIfMissing` 要谨慎

```java
@ConditionalOnProperty(
        prefix = "sms",
        name = "enabled",
        havingValue = "true",
        matchIfMissing = true
)
```

表示：

```text
sms.enabled 没配置时，也启用短信功能。
```

这可能很危险。

对外部副作用强的功能，建议：

```java
matchIfMissing = false
```

比如：

```text
短信
邮件
支付
扣费
MQ 消费
定时任务
外部接口调用
```

---

# 总结

`Condition` 不需要学成一个很大的专题。

																								你先抓住这个主线：

```text
Condition = Spring 启动时决定 Bean 要不要加载。
```

实际开发最常用的是：

```text
@ConditionalOnProperty
    根据配置开关加载 Bean

@ConditionalOnMissingBean
    用户没配，我给默认实现

@ConditionalOnBean
    有某个依赖 Bean，才创建当前 Bean

@ConditionalOnClass
    引入某个 jar，才启用自动配置

@Profile
    根据 dev/test/prod 环境加载 Bean
```

自定义 `Condition` 只在现成注解不够用时再写。

最终判断口诀：

```text
启动时决定有没有组件 -> Condition
运行时决定走哪个分支 -> if-else / 策略模式 / 责任链
用户没配就给默认 -> @ConditionalOnMissingBean
配置开关控制启停 -> @ConditionalOnProperty
依赖存在才创建 -> @ConditionalOnBean
jar 存在才自动配置 -> @ConditionalOnClass
```

对业务开发来说，掌握这些已经够用了。等你以后写自己的 Spring Boot Starter，再深入研究自定义 `Condition`、自动配置顺序、`AutoConfiguration.imports` 这些更底层的内容。