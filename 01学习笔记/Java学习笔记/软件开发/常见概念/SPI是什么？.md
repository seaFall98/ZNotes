## SPI 是什么？

**SPI = Service Provider Interface，服务提供者接口。**

它是一种**扩展机制**：  
框架或平台先定义一套接口规范，第三方开发者按照规范写实现类，然后通过约定方式注册进去。运行时，框架自动发现这些实现，并把它们加载进来。

一句话：

> **SPI 是“框架留给外部扩展的插槽”。你实现它，框架就能在运行时发现并使用你的实现。**

---

## SPI 和 API 的区别

很多人会把 SPI 和 API 混在一起。

|对比项|API|SPI|
|---|---|---|
|全称|Application Programming Interface|Service Provider Interface|
|面向对象|使用者|扩展者|
|谁调用谁|业务代码调用框架|框架调用你的实现|
|目的|使用已有能力|扩展框架能力|
|例子|调用 `JdbcTemplate.query()`|自定义 JDBC Driver|

可以粗略理解为：

```text
API：我调用别人
SPI：别人调用我
```

---

## 一个典型例子：JDBC Driver

你平时写 JDBC 连接数据库时，可能会这样：

```java
Connection connection = DriverManager.getConnection(
    "jdbc:mysql://localhost:3306/test",
    "root",
    "password"
);
```

你并没有手动 new 一个 MySQL Driver：

```java
new com.mysql.cj.jdbc.Driver()
```

那 `DriverManager` 是怎么知道要用 MySQL 驱动的？

原因是 MySQL JDBC 驱动包里有一个 SPI 配置文件：

```text
META-INF/services/java.sql.Driver
```

文件内容类似：

```text
com.mysql.cj.jdbc.Driver
```

JDK 会通过 `ServiceLoader` 机制发现这个实现类，然后注册到 `DriverManager` 中。

这就是 SPI。

---

## Java 原生 SPI 的基本结构

假设框架定义了一个接口：

```java
public interface PaymentProvider {
    boolean support(String channel);

    void pay(BigDecimal amount);
}
```

你写一个实现类：

```java
public class AliPayProvider implements PaymentProvider {
    @Override
    public boolean support(String channel) {
        return "alipay".equals(channel);
    }

    @Override
    public void pay(BigDecimal amount) {
        System.out.println("AliPay pay: " + amount);
    }
}
```

然后在资源目录下创建文件：

```text
src/main/resources/META-INF/services/com.example.PaymentProvider
```

文件内容写实现类全限定名：

```text
com.example.AliPayProvider
```

运行时可以这样加载：

```java
ServiceLoader<PaymentProvider> loader =
        ServiceLoader.load(PaymentProvider.class);

for (PaymentProvider provider : loader) {
    if (provider.support("alipay")) {
        provider.pay(new BigDecimal("100"));
    }
}
```

这就是 Java 标准 SPI 的最小模型。

---

## Spring 里的 SPI

Spring 也大量使用 SPI 思想，但它不完全依赖 JDK 的 `ServiceLoader`。

Spring 常见的扩展点包括：

|扩展点|作用|
|---|---|
|`BeanPostProcessor`|Bean 初始化前后增强|
|`BeanFactoryPostProcessor`|修改 BeanDefinition|
|`ImportSelector`|动态导入配置类|
|`EnvironmentPostProcessor`|在环境准备阶段修改配置|
|`ApplicationContextInitializer`|在容器刷新前初始化上下文|
|`AutoConfiguration.imports`|Spring Boot 自动配置加载|
|`FactoryBean`|自定义复杂 Bean 创建逻辑|
|`HandlerMethodArgumentResolver`|Spring MVC 自定义方法参数解析|
|`Converter` / `Formatter`|类型转换扩展|
|`Condition`|条件化装配|

这些都可以理解为 Spring 生态中的 SPI 扩展点。

---

## Spring Boot 自动配置也是 SPI 思想

Spring Boot 3.x 中，自动配置通常通过：

```text
META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
```

注册自动配置类，例如：

```text
com.example.demo.autoconfig.MyFeatureAutoConfiguration
```

然后你写一个自动配置类：

```java
@AutoConfiguration
@ConditionalOnClass(MyFeatureClient.class)
@EnableConfigurationProperties(MyFeatureProperties.class)
public class MyFeatureAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyFeatureClient myFeatureClient(MyFeatureProperties properties) {
        return new MyFeatureClient(properties.getEndpoint(), properties.getApiKey());
    }
}
```

当用户引入你的 starter 依赖后，Spring Boot 会自动发现这个配置类，并根据条件创建 Bean。

这就是典型的 Spring Boot Starter 扩展模式。

---

## MyBatis 里的 SPI / 插件机制

MyBatis 也有自己的扩展点，比如插件拦截器：

```java
@Intercepts({
    @Signature(
        type = Executor.class,
        method = "query",
        args = {
            MappedStatement.class,
            Object.class,
            RowBounds.class,
            ResultHandler.class
        }
    )
})
public class SqlLogInterceptor implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        long start = System.currentTimeMillis();

        try {
            return invocation.proceed();
        } finally {
            long cost = System.currentTimeMillis() - start;
            System.out.println("SQL cost: " + cost + "ms");
        }
    }
}
```

然后注册到 MyBatis 中：

```java
@Configuration
public class MyBatisConfig {

    @Bean
    public Interceptor sqlLogInterceptor() {
        return new SqlLogInterceptor();
    }
}
```

这种方式不是严格意义上的 JDK SPI，但属于**框架级 SPI 扩展机制**。

---

## Spring AI 里也有类似 SPI 的扩展思想

Spring AI 中你会接触到很多可扩展接口，例如：

|扩展点|作用|
|---|---|
|`ChatModel`|接入不同大模型供应商|
|`EmbeddingModel`|接入不同向量模型|
|`VectorStore`|接入不同向量数据库|
|`ChatMemory`|扩展对话记忆|
|`ToolCallback`|暴露工具调用能力|
|`Advisor`|对 ChatClient 调用链做增强|

例如你想接入一个自定义向量库，可以实现类似：

```java
public class CustomVectorStore implements VectorStore {

    @Override
    public void add(List<Document> documents) {
        // 写入自定义向量库
    }

    @Override
    public List<Document> similaritySearch(SearchRequest request) {
        // 执行相似度检索
        return List.of();
    }
}
```

然后注册成 Spring Bean：

```java
@Configuration
public class VectorStoreConfig {

    @Bean
    public VectorStore customVectorStore() {
        return new CustomVectorStore();
    }
}
```

业务层仍然依赖 `VectorStore` 接口，不关心底层到底是 PGVector、Redis、Milvus、Elasticsearch 还是你的自定义实现。

---

## 为什么简历里会写“基于 SPI 开发组件”？

这句话想表达的能力是：

> 不只是会调用框架 API，而是理解框架扩展点，能按照框架提供的接口、生命周期和装配机制，开发可复用组件。

比如：

### 低阶说法

```text
我会用 Spring Boot、MyBatis。
```

### 更强的说法

```text
我能基于 Spring Boot 自动配置机制封装 starter；
能基于 BeanPostProcessor / ImportSelector / Condition 等扩展点开发通用组件；
能基于 MyBatis Interceptor 开发 SQL 审计、分页、数据权限等插件。
```

差别很大。

前者是**框架使用者**。  
后者是**框架扩展者**。

---

## SPI 常见落地场景

在企业开发中，SPI 常用于这些场景：

|场景|示例|
|---|---|
|数据库驱动扩展|JDBC Driver|
|日志框架适配|SLF4J Binding|
|Spring Boot Starter|自动配置一个通用组件|
|ORM 插件|MyBatis 分页、审计、数据权限|
|支付渠道扩展|支付宝、微信、Stripe|
|消息通道扩展|Kafka、RabbitMQ、RocketMQ|
|AI 模型适配|OpenAI、Anthropic、DeepSeek、Qwen|
|向量数据库适配|PGVector、Milvus、Redis、Elasticsearch|
|文件存储扩展|本地、S3、MinIO、OSS|

---

## 面试里怎么讲 SPI？

可以这样回答：

> SPI 是 Service Provider Interface，本质是一种面向扩展的机制。框架定义接口和加载规则，第三方实现接口并通过配置文件、注解或容器注册方式暴露实现。运行时框架根据约定发现并加载这些实现，从而实现插件化和解耦。
> 
> API 是业务代码调用框架能力，而 SPI 是框架反向调用开发者提供的实现。典型例子是 JDBC Driver、Java `ServiceLoader`、Spring Boot 自动配置、MyBatis Interceptor。
> 
> 在实际项目中，我可以基于 Spring Boot 的自动配置机制封装 starter，也可以基于 MyBatis Interceptor 做 SQL 日志、数据权限、审计字段填充等通用组件。

---

## 你可以这样理解

```text
框架 = 主机
SPI 接口 = 插槽
你的实现类 = 插件
注册文件 / 注解 / Bean = 告诉主机插件在哪
运行时加载 = 主机把插件插上并调用
```

所以，简历里的这句话翻译成人话就是：

> 我不只是会用 Spring、Spring Boot、MyBatis、Spring AI，还能理解它们的扩展点，自己写 starter、插件、适配器、拦截器、模型接入组件、向量库接入组件等。