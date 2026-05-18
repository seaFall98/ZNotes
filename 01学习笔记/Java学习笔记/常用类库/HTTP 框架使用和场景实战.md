[xfg的教学案例](https://bugstack.cn/md/road-map/http.html)

# HTTP 框架使用和场景实战：从“能调接口”到“生产级 HTTP 集成体系”


```mermaid
mindmap
  root((HTTP 生产级实战))
    协议基础
      Method
      Header
      Status Code
      Body
      Cookie
      HTTPS
    客户端选型
      RestClient
      WebClient
      OkHttp
      Apache HttpClient
      OpenFeign
      HTTP Interface
    工程封装
      Client层
      Gateway层
      DTO隔离
      Mapper转换
      配置中心
    稳定性治理
      超时
      重试
      熔断
      限流
      隔舱
      降级
    安全
      Token
      签名
      验签
      HTTPS
      日志脱敏
      Secret管理
    可观测性
      Log
      Metrics
      Trace
      Alert
      Prometheus
      OpenTelemetry
    典型场景
      支付
      短信
      AI服务
      Webhook
      文件上传
      API聚合
    测试
      单元测试
      MockServer
      契约测试
      集成测试
      压测
```

---


## 0. 结论先行

在企业级 Java 后端里，**HTTP 调用不是简单的 `发请求 -> 拿响应`**。真正的生产级 HTTP 集成要解决的是：

> 如何把外部 HTTP 服务调用，做成**可配置、可观测、可限流、可降级、可重试、可治理、可演进、可测试**的一套工程能力。

小项目里你可以用 `OkHttp`、`HttpClient`、`RestTemplate` 随手写一个请求；但在生产系统里，HTTP 调用通常会变成以下问题：

|问题|低质量写法|生产级写法|
|---|---|---|
|接口散落|每个 Service 里拼 URL、拼 Header|独立 Client 层 / Gateway Client 层|
|超时缺失|默认无限等待或超时过长|连接超时、读超时、整体超时分层配置|
|重试混乱|捕获异常后 while 重试|只对幂等请求重试，使用 Resilience4j 策略|
|熔断缺失|上游挂了拖垮自己|熔断、隔离、降级、限流|
|日志粗糙|只打印异常|traceId、接口名、耗时、状态码、错误码|
|安全风险|明文存 token/cookie|配置中心、密钥管理、脱敏日志|
|可测试性差|真实请求第三方|MockServer / WireMock / 契约测试|
|代码膨胀|JSON、Header、错误处理混在一起|DTO、Client、Adapter、Domain 分层|

Spring 生态现在的主流选择是：

- **同步 HTTP 调用**：优先考虑 `RestClient`。Spring Framework 官方文档说明，`RestClient` 是同步 HTTP 客户端，提供 fluent API，并抽象底层 HTTP 库与对象转换。([Home](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html?utm_source=chatgpt.com "REST Clients :: Spring Framework"))
    
- **响应式/高并发 I/O 调用**：使用 `WebClient`。
    
- **声明式 HTTP 客户端**：Spring 6 的 HTTP Interface，适合把远程 HTTP API 定义成 Java 接口。
    
- **容错治理**：Resilience4j，提供 Retry、CircuitBreaker、RateLimiter、Bulkhead 等组件。Resilience4j 官方说明它是 Java 的轻量级容错库，可以通过装饰器叠加熔断、限流、重试、隔舱等能力。([GitHub](https://github.com/resilience4j/resilience4j?utm_source=chatgpt.com "Resilience4j is a fault tolerance library designed for Java8 ..."))
    
- **可观测性**：Micrometer + OpenTelemetry。OpenTelemetry 官方将其定义为云原生软件的开源可观测性框架，用于采集 traces、metrics、logs。([OpenTelemetry](https://opentelemetry.io/?utm_source=chatgpt.com "OpenTelemetry"))
    

---

# 1. HTTP 在后端系统里的真实定位

## 1.1 HTTP 不是“低级协议”，而是系统边界协议

很多 Java 后端开发者一开始会有一个误解：

> 微服务内部用 Dubbo/gRPC 才高级，HTTP 只是对外接口。

这个理解不完整。

在真实企业系统里，HTTP 经常出现在这些位置：

```mermaid
flowchart LR
    A[前端 Web / App / 小程序] -->|HTTP/HTTPS| B[API Gateway]
    B -->|HTTP / RPC| C[业务服务]
    C -->|HTTP| D[支付平台]
    C -->|HTTP| E[短信平台]
    C -->|HTTP| F[AI 模型服务]
    C -->|HTTP| G[第三方 SaaS]
    C -->|HTTP| H[内部异构系统]
```

HTTP 的价值是：

1. **跨语言**：Java、Go、Python、Node 都能接。
    
2. **跨组织**：对接微信支付、支付宝、OpenAI、DeepSeek、飞书、企业微信，基本都靠 HTTP。
    
3. **生态成熟**：网关、代理、CDN、WAF、监控、日志、链路追踪都天然支持 HTTP。
    
4. **调试成本低**：curl、Postman、Apifox、浏览器 DevTools 都能直接调试。
    
5. **适合开放 API**：REST、SSE、Webhook、OpenAPI 规范都建立在 HTTP 上。
    

所以，HTTP 是现代后端系统的**边界协议**和**集成协议**。

---

# 2. HTTP 调用的工程复杂度在哪里？

## 2.1 初级代码：能跑，但不可维护

很多人第一次写 HTTP 调用，大概是这样：

```java
public String callRemote(String userId) {
    RestTemplate restTemplate = new RestTemplate();

    String url = "https://api.example.com/users/" + userId;

    HttpHeaders headers = new HttpHeaders();
    headers.add("Authorization", "Bearer xxx");

    HttpEntity<Void> entity = new HttpEntity<>(headers);

    ResponseEntity<String> response = restTemplate.exchange(
            url,
            HttpMethod.GET,
            entity,
            String.class
    );

    return response.getBody();
}
```

问题很多：

- `RestTemplate` 临时创建，没有连接池治理。
    
- URL 写死。
    
- token 写死。
    
- 没有超时。
    
- 没有错误码解析。
    
- 没有重试策略。
    
- 没有日志与耗时统计。
    
- 没有 traceId 传递。
    
- 没有测试替身。
    
- 调用逻辑和业务逻辑混在一起。
    

这类代码在 Demo 里没问题，在生产中就是事故隐患。

---

## 2.2 生产级 HTTP 调用要关心 9 件事

|维度|生产要求|
|---|---|
|协议|GET/POST/PUT/DELETE、Header、Query、Body、Status Code|
|序列化|JSON、表单、文件上传、流式响应|
|超时|connect timeout、read timeout、response timeout、business timeout|
|连接池|最大连接数、每路由连接数、空闲连接回收|
|容错|重试、熔断、限流、降级、隔离|
|安全|HTTPS、签名、token、密钥、日志脱敏|
|可观测性|日志、指标、链路追踪、告警|
|幂等|请求幂等键、重试安全性、重复提交控制|
|测试|单测、Mock、契约测试、压测|

---

# 3. Java HTTP 客户端选型：不要再只盯着 RestTemplate

## 3.1 常见 HTTP 客户端对比

|客户端|适用场景|优点|注意点|
|---|---|---|---|
|JDK HttpClient|简单调用、无额外依赖|JDK 原生、支持 HTTP/2|企业治理能力要自己封装|
|Apache HttpClient|高级连接池、底层控制|成熟、配置能力强|API 相对重|
|OkHttp|轻量、高性能|易用、连接池好|Spring 生态集成需自己封装|
|RestTemplate|老项目维护|使用广泛|新项目不建议作为首选|
|RestClient|Spring Boot 3.x 同步调用|现代 fluent API，适合替代 RestTemplate|Spring 6.1+|
|WebClient|响应式、高并发 I/O|非阻塞、适合流式和并发请求|学习成本高，阻塞混用易出问题|
|OpenFeign|声明式 HTTP|微服务调用体验好|复杂治理依赖 Spring Cloud 生态|
|Spring HTTP Interface|声明式接口客户端|Spring 原生、清晰|适合 Spring 6+ 项目|

Spring 官方在 Spring 6.1 引入 `RestClient`，定位就是新的同步 HTTP 客户端，API 风格接近 `WebClient`，但共享 `RestTemplate` 的基础设施。([Home](https://spring.io/blog/2023/07/13/new-in-spring-6-1-restclient?utm_source=chatgpt.com "New in Spring 6.1: RestClient"))

---

## 3.2 推荐选型

### 场景一：普通 Spring Boot 后端调用第三方 API

优先：

```text
RestClient + Resilience4j + Micrometer
```

适合：

- 支付接口
    
- 短信接口
    
- AI 模型同步调用
    
- 企业微信/飞书接口
    
- 内部低并发 HTTP API
    

---

### 场景二：高并发 I/O 聚合服务

优先：

```text
WebClient + Reactor Netty + Resilience4j + OpenTelemetry
```

适合：

- API 聚合层
    
- BFF 层
    
- 同时调用多个下游服务
    
- 流式接口
    
- SSE 响应
    
- AI streaming 响应
    

---

### 场景三：接口很多，想提升可维护性

优先：

```text
Spring HTTP Interface / OpenFeign
```

适合：

- 对接平台型 API
    
- 多接口 SDK 封装
    
- 内部服务之间 HTTP 调用
    
- 需要强类型接口定义
    

Spring Boot 官方文档也明确说明，`RestClient.Builder` 和 `WebClient.Builder` 可以配置 API versioning，用于远程 API 演进。([Home](https://docs.spring.io/spring-boot/reference/io/rest-client.html?utm_source=chatgpt.com "Calling REST Services :: Spring Boot"))

---

# 4. 生产级案例：订单系统对接支付平台

下面设计一个接近生产环境的 HTTP 调用案例：

> 电商订单系统调用外部支付平台，创建支付单、查询支付状态、接收支付回调。

## 4.1 架构分层

```mermaid
flowchart TD
    A[OrderController] --> B[OrderApplicationService]
    B --> C[PaymentDomainService]
    C --> D[PaymentGateway]
    D --> E[PaymentHttpClient]
    E --> F[External Payment Provider]

    D --> G[DTO Mapper]
    E --> H[RestClient]
    E --> I[Resilience4j]
    E --> J[Metrics / Trace / Log]
```

建议分层：

|层|职责|
|---|---|
|Controller|接收用户请求|
|Application Service|编排业务流程|
|Domain Service|处理支付领域逻辑|
|Gateway|定义外部系统抽象|
|HttpClient|具体 HTTP 调用|
|DTO|外部接口入参出参|
|Mapper|内外部模型转换|

核心原则：

> 业务层不要直接感知 HTTP 细节，HTTP 只是基础设施实现。

---

# 5. Maven 依赖设计

以 Spring Boot 3.x 为例：

```xml
<dependencies>
    <!-- Web 基础能力：Controller、JSON、校验等 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- 参数校验：@NotBlank、@Valid 等 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Resilience4j：熔断、重试、限流、隔舱 -->
    <dependency>
        <groupId>io.github.resilience4j</groupId>
        <artifactId>resilience4j-spring-boot3</artifactId>
    </dependency>

    <!-- Actuator：暴露健康检查、指标等端点 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Micrometer Prometheus：导出 Prometheus 指标 -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Lombok：简化 DTO 代码，生产项目可按团队规范决定是否使用 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

# 6. 配置设计：不要把 URL、Token、超时写死在代码里

## 6.1 application.yml

```yaml
payment:
  provider:
    base-url: https://api.payment-example.com
    merchant-id: M123456
    app-id: app-demo
    # 生产环境不要直接写在 yml，应该来自配置中心、K8s Secret、Vault 等
    secret-key: ${PAYMENT_SECRET_KEY}
    connect-timeout-ms: 1000
    read-timeout-ms: 3000
    max-retry-attempts: 2

resilience4j:
  retry:
    instances:
      paymentClient:
        max-attempts: 2
        wait-duration: 200ms
        retry-exceptions:
          - java.net.SocketTimeoutException
          - org.springframework.web.client.ResourceAccessException

  circuitbreaker:
    instances:
      paymentClient:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 50
        failure-rate-threshold: 50
        slow-call-rate-threshold: 50
        slow-call-duration-threshold: 2s
        minimum-number-of-calls: 20
        wait-duration-in-open-state: 10s

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
```

注意：

- 支付创建接口通常不能随便重试，除非有**幂等号**。
    
- 查询接口可以重试。
    
- 支付回调处理必须幂等。
    
- secret 不应写入 Git。
    
- HTTP 日志不能打印完整 token、cookie、身份证、手机号、银行卡号。
    

---

## 6.2 配置类

```java
package com.example.payment.config;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.validation.annotation.Validated;

/**
 * 支付平台 HTTP 配置。
 *
 * 这类配置不要散落在业务代码中，统一收口后：
 * 1. 便于环境隔离：dev/test/prod 使用不同配置；
 * 2. 便于超时调优；
 * 3. 便于密钥治理；
 * 4. 便于在启动阶段做参数校验。
 */
@Data
@Validated
@ConfigurationProperties(prefix = "payment.provider")
public class PaymentProviderProperties {

    @NotBlank
    private String baseUrl;

    @NotBlank
    private String merchantId;

    @NotBlank
    private String appId;

    @NotBlank
    private String secretKey;

    @Min(100)
    private int connectTimeoutMs = 1000;

    @Min(100)
    private int readTimeoutMs = 3000;

    @Min(1)
    private int maxRetryAttempts = 2;
}
```

---

# 7. RestClient 生产级封装

## 7.1 RestClient Bean 配置

```java
package com.example.payment.config;

import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.JdkClientHttpRequestFactory;
import org.springframework.web.client.RestClient;

import java.net.http.HttpClient;
import java.time.Duration;

/**
 * 支付平台 HTTP 客户端配置。
 *
 * 这里使用 JDK HttpClient 作为底层实现。
 * 生产环境也可以替换为 Apache HttpClient，以获得更细粒度连接池能力。
 */
@Configuration
@EnableConfigurationProperties(PaymentProviderProperties.class)
public class PaymentHttpClientConfig {

    @Bean
    public RestClient paymentRestClient(PaymentProviderProperties properties) {
        HttpClient httpClient = HttpClient.newBuilder()
                // 连接超时：TCP 建连阶段最大等待时间
                .connectTimeout(Duration.ofMillis(properties.getConnectTimeoutMs()))
                .version(HttpClient.Version.HTTP_2)
                .build();

        JdkClientHttpRequestFactory requestFactory = new JdkClientHttpRequestFactory(httpClient);

        // 读超时：请求发出后等待响应数据的最大时间
        requestFactory.setReadTimeout(Duration.ofMillis(properties.getReadTimeoutMs()));

        return RestClient.builder()
                .baseUrl(properties.getBaseUrl())
                .requestFactory(requestFactory)
                .defaultHeader("Content-Type", "application/json")
                .build();
    }
}
```

这段代码解决了几个生产问题：

- `RestClient` 作为 Bean 复用，不要每次 new。
    
- 统一 baseUrl。
    
- 统一超时。
    
- 统一底层 requestFactory。
    
- 后续可以加入拦截器、日志、traceId。
    

---

# 8. 外部接口 DTO 设计

## 8.1 创建支付单请求

```java
package com.example.payment.client.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Builder;
import lombok.Data;

import java.math.BigDecimal;

/**
 * 外部支付平台创建支付单请求。
 *
 * 注意：
 * 1. DTO 表示“外部协议模型”，不等同于内部领域模型；
 * 2. 字段命名应尽量贴近外部接口文档；
 * 3. 内部 Order 不要直接暴露给第三方。
 */
@Data
@Builder
public class CreatePaymentRequest {

    @NotBlank
    private String merchantId;

    @NotBlank
    private String outTradeNo;

    @NotNull
    private BigDecimal amount;

    @NotBlank
    private String currency;

    @NotBlank
    private String subject;

    @NotBlank
    private String notifyUrl;

    /**
     * 幂等键。
     * 对创建类接口非常关键，用于避免网络超时后重复创建支付单。
     */
    @NotBlank
    private String idempotencyKey;

    /**
     * 请求签名。
     * 生产环境中通常需要按平台规范进行 HMAC/RSA 签名。
     */
    @NotBlank
    private String sign;
}
```

---

## 8.2 创建支付单响应

```java
package com.example.payment.client.dto;

import lombok.Data;

import java.time.OffsetDateTime;

/**
 * 外部支付平台创建支付单响应。
 */
@Data
public class CreatePaymentResponse {

    private String code;

    private String message;

    private PaymentData data;

    public boolean success() {
        return "SUCCESS".equalsIgnoreCase(code);
    }

    @Data
    public static class PaymentData {
        private String paymentNo;
        private String outTradeNo;
        private String payUrl;
        private String status;
        private OffsetDateTime expireTime;
    }
}
```

---

# 9. Client 层：统一封装 HTTP 细节

```java
package com.example.payment.client;

import com.example.payment.client.dto.CreatePaymentRequest;
import com.example.payment.client.dto.CreatePaymentResponse;
import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatusCode;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

/**
 * 支付平台 HTTP Client。
 *
 * 这一层只做外部 HTTP 调用，不承载订单业务规则。
 * 业务含义应该放在 PaymentGateway / DomainService 中。
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentHttpClient {

    private final RestClient paymentRestClient;

    /**
     * 创建支付单。
     *
     * 注意：
     * 1. 创建类接口原则上不应盲目重试；
     * 2. 如果要重试，必须依赖幂等键；
     * 3. 熔断可以防止支付平台异常时拖垮订单服务。
     */
    @Retry(name = "paymentClient", fallbackMethod = "createPaymentFallback")
    @CircuitBreaker(name = "paymentClient", fallbackMethod = "createPaymentFallback")
    public CreatePaymentResponse createPayment(CreatePaymentRequest request) {
        long start = System.currentTimeMillis();

        try {
            CreatePaymentResponse response = paymentRestClient.post()
                    .uri("/v1/payments")
                    // 幂等键建议同时放 Header，便于网关和平台侧识别
                    .header("Idempotency-Key", request.getIdempotencyKey())
                    .body(request)
                    .retrieve()
                    .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
                        // 4xx 通常代表请求参数、鉴权、签名错误，不应重试
                        throw new PaymentClientException("PAYMENT_4XX", "支付平台返回客户端错误");
                    })
                    .onStatus(HttpStatusCode::is5xxServerError, (req, res) -> {
                        // 5xx 通常可能是平台临时故障，可根据策略重试/熔断
                        throw new PaymentClientException("PAYMENT_5XX", "支付平台返回服务端错误");
                    })
                    .body(CreatePaymentResponse.class);

            long cost = System.currentTimeMillis() - start;
            log.info("create payment success, outTradeNo={}, cost={}ms",
                    maskOutTradeNo(request.getOutTradeNo()), cost);

            return response;
        } catch (Exception ex) {
            long cost = System.currentTimeMillis() - start;
            log.warn("create payment failed, outTradeNo={}, cost={}ms, error={}",
                    maskOutTradeNo(request.getOutTradeNo()), cost, ex.toString());
            throw ex;
        }
    }

    /**
     * 熔断/重试失败后的降级逻辑。
     *
     * 生产中不建议伪造“支付成功”。
     * 更合理的做法是返回“支付创建中/待确认”，由补偿任务后续查询。
     */
    public CreatePaymentResponse createPaymentFallback(CreatePaymentRequest request, Throwable ex) {
        log.error("payment create fallback, outTradeNo={}, reason={}",
                maskOutTradeNo(request.getOutTradeNo()), ex.toString());

        throw new PaymentClientException(
                "PAYMENT_UNAVAILABLE",
                "支付服务暂不可用，请稍后重试"
        );
    }

    private String maskOutTradeNo(String outTradeNo) {
        if (outTradeNo == null || outTradeNo.length() <= 6) {
            return "******";
        }
        return outTradeNo.substring(0, 3) + "******" + outTradeNo.substring(outTradeNo.length() - 3);
    }
}
```

自定义异常：

```java
package com.example.payment.client;

/**
 * 支付平台调用异常。
 *
 * 用业务错误码封装底层 HTTP/网络异常，
 * 避免上层业务直接依赖 RestClientException、SocketTimeoutException 等技术异常。
 */
public class PaymentClientException extends RuntimeException {

    private final String code;

    public PaymentClientException(String code, String message) {
        super(message);
        this.code = code;
    }

    public PaymentClientException(String code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public String getCode() {
        return code;
    }
}
```

---

# 10. Gateway 层：把 HTTP 细节翻译成业务语义

```java
package com.example.payment.gateway;

import com.example.payment.client.PaymentHttpClient;
import com.example.payment.client.dto.CreatePaymentRequest;
import com.example.payment.client.dto.CreatePaymentResponse;
import com.example.payment.config.PaymentProviderProperties;
import com.example.payment.domain.PaymentCreateCommand;
import com.example.payment.domain.PaymentCreateResult;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.UUID;

/**
 * 支付网关实现。
 *
 * Gateway 是领域层访问外部系统的防腐层：
 * 1. 屏蔽 HTTP DTO；
 * 2. 屏蔽第三方错误码；
 * 3. 把外部系统语义转换成内部业务语义。
 */
@Component
@RequiredArgsConstructor
public class PaymentGateway {

    private final PaymentHttpClient paymentHttpClient;
    private final PaymentProviderProperties properties;
    private final PaymentSigner paymentSigner;

    public PaymentCreateResult createPayment(PaymentCreateCommand command) {
        String idempotencyKey = buildIdempotencyKey(command);

        CreatePaymentRequest request = CreatePaymentRequest.builder()
                .merchantId(properties.getMerchantId())
                .outTradeNo(command.getOrderNo())
                .amount(command.getAmount())
                .currency("CNY")
                .subject(command.getSubject())
                .notifyUrl(command.getNotifyUrl())
                .idempotencyKey(idempotencyKey)
                .build();

        // 签名必须基于最终请求参数生成
        request.setSign(paymentSigner.sign(request));

        CreatePaymentResponse response = paymentHttpClient.createPayment(request);

        if (response == null || !response.success() || response.getData() == null) {
            throw new IllegalStateException("创建支付单失败");
        }

        return PaymentCreateResult.builder()
                .paymentNo(response.getData().getPaymentNo())
                .orderNo(response.getData().getOutTradeNo())
                .payUrl(response.getData().getPayUrl())
                .status(response.getData().getStatus())
                .expireTime(response.getData().getExpireTime())
                .build();
    }

    private String buildIdempotencyKey(PaymentCreateCommand command) {
        // 推荐格式：业务类型 + 业务单号 + 操作类型
        // UUID 不适合作为创建支付单的幂等键，因为重试时会变化
        return "payment:create:" + command.getOrderNo();
    }
}
```

---

# 11. 签名设计：HTTP 集成里的关键安全点

支付、开放平台、企业 SaaS 对接经常要求签名。

## 11.1 简化版 HMAC 签名器

```java
package com.example.payment.gateway;

import com.example.payment.client.dto.CreatePaymentRequest;
import com.example.payment.config.PaymentProviderProperties;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import java.nio.charset.StandardCharsets;
import java.util.HexFormat;

/**
 * 支付请求签名器。
 *
 * 示例使用 HMAC-SHA256。
 * 真实项目必须以第三方支付平台文档为准：
 * 1. 参数排序规则；
 * 2. 空值是否参与签名；
 * 3. 时间戳/nonce 是否参与签名；
 * 4. RSA/HMAC/国密算法要求；
 * 5. 签名字段大小写规则。
 */
@Component
@RequiredArgsConstructor
public class PaymentSigner {

    private final PaymentProviderProperties properties;

    public String sign(CreatePaymentRequest request) {
        String plainText = buildPlainText(request);
        return hmacSha256(plainText, properties.getSecretKey());
    }

    private String buildPlainText(CreatePaymentRequest request) {
        // 示例：按固定字段拼接。生产中建议使用 TreeMap 按字典序排序。
        return "merchantId=" + request.getMerchantId()
                + "&outTradeNo=" + request.getOutTradeNo()
                + "&amount=" + request.getAmount()
                + "&currency=" + request.getCurrency()
                + "&idempotencyKey=" + request.getIdempotencyKey();
    }

    private String hmacSha256(String plainText, String secretKey) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            SecretKeySpec keySpec = new SecretKeySpec(
                    secretKey.getBytes(StandardCharsets.UTF_8),
                    "HmacSHA256"
            );
            mac.init(keySpec);
            byte[] digest = mac.doFinal(plainText.getBytes(StandardCharsets.UTF_8));
            return HexFormat.of().formatHex(digest);
        } catch (Exception ex) {
            throw new IllegalStateException("支付请求签名失败", ex);
        }
    }
}
```

## 11.2 安全要求

|风险|处理方式|
|---|---|
|secret 泄露|不进 Git，不打日志，使用 Secret 管理|
|请求被篡改|签名校验|
|请求重放|timestamp + nonce + 幂等键|
|回调伪造|验签 + IP 白名单 + HTTPS|
|敏感数据日志泄露|日志脱敏|
|token 过期|token 自动刷新和缓存|

---

# 12. 超时设计：HTTP 调用最容易被忽略的事故源

## 12.1 超时不是一个参数

生产中至少要区分：

|超时类型|含义|
|---|---|
|连接超时|TCP 建连最大等待时间|
|读超时|等待响应数据最大时间|
|连接池获取超时|从连接池拿连接的最大等待时间|
|整体调用超时|一次业务调用允许的最大时间|
|上游业务超时|第三方服务内部处理超时|

错误配置示例：

```yaml
read-timeout-ms: 30000
```

这看似安全，实际可能导致：

- Tomcat 工作线程被占满。
    
- 连接池被耗尽。
    
- 下游慢导致上游雪崩。
    
- 重试叠加后请求时间远超用户可接受范围。
    

---

## 12.2 推荐经验值

|场景|连接超时|读超时|是否重试|
|---|--:|--:|---|
|内网服务|100~300ms|500~1500ms|可谨慎重试|
|支付创建|500~1000ms|2000~5000ms|必须幂等才重试|
|支付查询|500~1000ms|1000~3000ms|可重试|
|短信发送|500~1000ms|2000~5000ms|谨慎重试|
|AI 模型同步调用|1000~3000ms|30s~120s|通常不简单重试|
|AI 流式调用|1000~3000ms|按流式协议设计|不按普通 HTTP 处理|

---

# 13. 重试设计：不是失败就重试

Resilience4j 的 Retry 模块支持创建 RetryRegistry，并可配置等待间隔、自定义 backoff 等策略。([resilience4j](https://resilience4j.readme.io/docs/retry?utm_source=chatgpt.com "Retry"))

但工程上要先判断：**这个请求能不能重试？**

## 13.1 可以重试的场景

|场景|原因|
|---|---|
|GET 查询接口|通常无副作用|
|幂等 POST|有 Idempotency-Key|
|502/503/504|可能是临时网关/服务异常|
|连接超时|请求可能未到达服务端|
|读超时|谨慎，服务端可能已经处理成功|

## 13.2 不应盲目重试的场景

|场景|原因|
|---|---|
|创建订单|可能重复创建|
|扣库存|可能重复扣减|
|支付扣款|可能重复支付|
|发送短信|用户可能收到多条|
|非幂等写接口|副作用不可控|

## 13.3 判断口诀

> 查询可以重试，写入先问幂等；  
> 网络异常可重试，业务失败别重试；  
> 读超时最危险，因为服务端可能已经成功。

---

# 14. 熔断设计：保护自己，而不是修复别人

Resilience4j CircuitBreaker 使用滑动窗口聚合调用结果，可以选择基于调用次数或基于时间的窗口。([resilience4j](https://resilience4j.readme.io/docs/circuitbreaker?utm_source=chatgpt.com "CircuitBreaker"))

熔断的本质：

```text
当下游持续失败或持续变慢时，主动停止调用一段时间，避免把自己的线程、连接、CPU 全拖死。
```

## 14.1 熔断状态

```mermaid
stateDiagram-v2
    [*] --> CLOSED
    CLOSED --> OPEN: 失败率/慢调用率超过阈值
    OPEN --> HALF_OPEN: 等待窗口结束
    HALF_OPEN --> CLOSED: 探测调用成功
    HALF_OPEN --> OPEN: 探测调用失败
```

## 14.2 支付场景里的熔断策略

支付创建接口熔断时，不应该返回“支付成功”。

正确做法：

1. 返回“支付服务暂不可用”。
    
2. 或者创建本地 `PAYMENT_PENDING` 状态。
    
3. 后续由补偿任务查询支付平台。
    
4. 用户侧展示“支付处理中，请稍后刷新”。
    

---

# 15. 限流与隔舱：避免第三方拖垮核心链路

## 15.1 为什么 HTTP Client 也要限流？

假设支付平台故障，用户疯狂点击“支付”：

```text
用户请求 -> 订单服务 -> 支付 HTTP 调用 -> 全部阻塞/超时
```

如果没有限流：

- 订单服务线程被占满。
    
- 数据库连接被占满。
    
- HTTP 连接池被占满。
    
- JVM 堆积大量等待对象。
    
- 最终不是支付平台挂，而是你的订单服务先挂。
    

## 15.2 隔舱思想

不同外部依赖要有不同资源池：

```mermaid
flowchart TD
    A[订单服务] --> B[支付调用线程池/连接池]
    A --> C[短信调用线程池/连接池]
    A --> D[AI服务调用线程池/连接池]
    A --> E[物流调用线程池/连接池]
```

不要让 AI 服务慢调用占满支付链路资源。

---

# 16. 可观测性：HTTP 调用必须能被定位

## 16.1 最少要记录什么？

|字段|说明|
|---|---|
|traceId|链路追踪 ID|
|spanId|当前调用跨度 ID|
|clientName|调用哪个外部系统|
|apiName|调用哪个接口|
|method|GET/POST|
|urlPattern|不记录完整敏感 URL|
|statusCode|HTTP 状态码|
|bizCode|第三方业务错误码|
|costMs|耗时|
|retryCount|重试次数|
|fallback|是否降级|
|exceptionType|异常类型|

OpenTelemetry 提供标准化的 API、SDK 与 Collector 来采集和导出 metrics、logs、traces，适合解决分布式系统中跨服务定位问题。([OpenTelemetry](https://opentelemetry.io/?utm_source=chatgpt.com "OpenTelemetry"))

---

## 16.2 日志示例

好的日志：

```text
payment_http_call client=paymentProvider api=createPayment method=POST status=200 bizCode=SUCCESS costMs=286 traceId=abc123 outTradeNo=ORD***789
```

差的日志：

```text
调用失败了
```

更差的日志：

```text
Authorization: Bearer eyJhbGciOiJIUzI1Ni...
Cookie: SESSION=xxx
cardNo=6222020202020202020
```

---

# 17. 统一 HTTP 错误模型

## 17.1 HTTP 状态码和业务错误码要分开

很多第三方 API 会这样返回：

```json
{
  "code": "INVALID_SIGN",
  "message": "签名错误",
  "data": null
}
```

HTTP 状态码可能是 `200`，但业务失败。

所以判断成功不能只看 HTTP 200。

```java
if (response == null || !response.success()) {
    throw new PaymentClientException("PAYMENT_BIZ_FAILED", "支付平台业务处理失败");
}
```

## 17.2 推荐错误分层

|类型|示例|处理方式|
|---|---|---|
|网络错误|连接失败、DNS 失败|可重试/熔断|
|超时错误|connect/read timeout|可重试/熔断|
|HTTP 4xx|401、403、400|多数不重试|
|HTTP 5xx|500、502、503|可重试/熔断|
|业务失败|余额不足、签名错误|不重试，按业务处理|
|解析失败|JSON 格式变更|告警，兼容处理|

---

# 18. WebClient 场景：并发聚合多个 HTTP 服务

如果你做的是 BFF 或 API 聚合层，例如一个首页接口要同时调用：

- 用户服务
    
- 推荐服务
    
- 优惠券服务
    
- 订单服务
    
- 内容服务
    

同步串行调用会很慢。

```mermaid
sequenceDiagram
    participant BFF
    participant User
    participant Recommend
    participant Coupon
    participant Order

    BFF->>User: 查询用户
    BFF->>Recommend: 查询推荐
    BFF->>Coupon: 查询优惠券
    BFF->>Order: 查询订单摘要
```

串行耗时：

```text
100ms + 200ms + 150ms + 180ms = 630ms
```

并发后理论耗时：

```text
max(100, 200, 150, 180) = 200ms 左右
```

## 18.1 WebClient 并发聚合示例

```java
package com.example.bff;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

/**
 * 首页聚合服务。
 *
 * WebClient 适合 I/O 密集型并发调用。
 * 注意：不要在响应式链路中随意 block()，否则会破坏非阻塞模型。
 */
@Service
@RequiredArgsConstructor
public class HomeAggregationService {

    private final WebClient webClient;

    public Mono<HomeView> getHome(String userId) {
        Mono<UserView> userMono = webClient.get()
                .uri("http://user-service/users/{userId}", userId)
                .retrieve()
                .bodyToMono(UserView.class);

        Mono<RecommendView> recommendMono = webClient.get()
                .uri("http://recommend-service/recommendations?userId={userId}", userId)
                .retrieve()
                .bodyToMono(RecommendView.class);

        Mono<CouponView> couponMono = webClient.get()
                .uri("http://coupon-service/coupons?userId={userId}", userId)
                .retrieve()
                .bodyToMono(CouponView.class);

        // zip 会并发等待多个 Mono 完成，然后组合结果
        return Mono.zip(userMono, recommendMono, couponMono)
                .map(tuple -> HomeView.builder()
                        .user(tuple.getT1())
                        .recommend(tuple.getT2())
                        .coupon(tuple.getT3())
                        .build());
    }
}
```

## 18.2 WebClient 常见坑

|坑|说明|
|---|---|
|到处 `block()`|把非阻塞又变成阻塞|
|连接池不配置|高并发下连接耗尽|
|不设超时|慢请求拖垮链路|
|异常处理混乱|Mono 错误传播不熟悉|
|ThreadLocal 丢失|traceId、用户上下文需要专门传播|
|阻塞操作混入 event loop|例如 JDBC、文件 IO、慢 CPU 任务|

---

# 19. 声明式 HTTP Interface：让远程 API 像 Java 接口

Spring Framework 支持用 `@HttpExchange` 等注解将 HTTP 服务定义为 Java 接口，再通过代理工厂创建客户端。官方 REST Clients 文档将 `RestClient`、`WebClient`、`RestTemplate` 和 HTTP Interface 放在 REST 客户端体系下。([Home](https://docs.spring.io/spring-framework/reference/integration/rest-clients.html?utm_source=chatgpt.com "REST Clients :: Spring Framework"))

## 19.1 接口定义

```java
package com.example.payment.client;

import com.example.payment.client.dto.CreatePaymentRequest;
import com.example.payment.client.dto.CreatePaymentResponse;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.service.annotation.HttpExchange;
import org.springframework.web.service.annotation.PostExchange;

/**
 * 声明式支付 HTTP API。
 *
 * 优点：
 * 1. 接口清晰；
 * 2. 减少样板代码；
 * 3. 适合 SDK 化封装；
 * 4. 更接近 OpenFeign 的使用体验。
 */
@HttpExchange("/v1")
public interface PaymentApi {

    @PostExchange("/payments")
    CreatePaymentResponse createPayment(@RequestBody CreatePaymentRequest request);
}
```

## 19.2 代理 Bean 配置

```java
package com.example.payment.config;

import com.example.payment.client.PaymentApi;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestClient;
import org.springframework.web.client.support.RestClientAdapter;
import org.springframework.web.service.invoker.HttpServiceProxyFactory;

/**
 * Spring HTTP Interface 配置。
 */
@Configuration
public class PaymentApiProxyConfig {

    @Bean
    public PaymentApi paymentApi(RestClient paymentRestClient) {
        RestClientAdapter adapter = RestClientAdapter.create(paymentRestClient);

        HttpServiceProxyFactory factory = HttpServiceProxyFactory
                .builderFor(adapter)
                .build();

        return factory.createClient(PaymentApi.class);
    }
}
```

适合场景：

- 外部 API 接口数量多。
    
- 团队希望接口定义清晰。
    
- 想减少重复 `.uri().body().retrieve()` 样板代码。
    
- 做内部 HTTP SDK。
    

不适合场景：

- 每个请求都要非常复杂的动态逻辑。
    
- 需要底层连接、拦截器、错误处理极度定制。
    
- 团队对 Spring 6 HTTP Interface 不熟。
    

---

# 20. AI 服务 HTTP 调用：和普通业务接口不一样

现在 Java 后端越来越多会对接 AI 模型服务，例如：

- OpenAI API
    
- DeepSeek API
    
- 智谱 GLM
    
- 通义千问
    
- Claude Router
    
- 私有化模型网关
    

AI HTTP 调用和传统 HTTP 调用有明显区别。

## 20.1 AI HTTP 调用的特点

|特点|影响|
|---|---|
|响应慢|超时要明显更长|
|输出不稳定|要做结果校验|
|token 成本|要记录输入/输出 token|
|流式响应|SSE/WebFlux 更合适|
|上下文大|请求体可能很大|
|限流严格|要处理 429|
|供应商差异|最好做模型网关适配层|

## 20.2 AI Client 分层建议

```mermaid
flowchart TD
    A[业务服务] --> B[AiApplicationService]
    B --> C[ModelGateway]
    C --> D[OpenAiClient]
    C --> E[DeepSeekClient]
    C --> F[ZhipuClient]
    C --> G[LocalModelClient]
```

不要在业务代码里直接写：

```java
webClient.post()
    .uri("https://api.openai.com/v1/chat/completions")
```

应该抽象成：

```java
modelGateway.chat(command);
```

这样后续可以替换模型、加缓存、加限流、加审计、加成本统计。

---

# 21. Webhook 回调：HTTP 服务端也要生产级

HTTP 不只是你调用别人，也包括别人回调你。

支付回调是典型场景。

## 21.1 支付回调 Controller

```java
package com.example.payment.webhook;

import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

/**
 * 支付平台回调入口。
 *
 * 生产要求：
 * 1. 必须验签；
 * 2. 必须幂等；
 * 3. 不能在回调线程里做过重业务；
 * 4. 返回结果要符合支付平台规范；
 * 5. 原始报文建议安全落库，便于审计和问题追踪。
 */
@RestController
@RequestMapping("/webhooks/payment")
@RequiredArgsConstructor
public class PaymentWebhookController {

    private final PaymentWebhookService paymentWebhookService;

    @PostMapping("/notify")
    public String notify(@RequestBody @Valid PaymentNotifyRequest request,
                         @RequestHeader("X-Payment-Sign") String sign) {

        paymentWebhookService.handleNotify(request, sign);

        // 很多支付平台要求返回固定字符串
        return "SUCCESS";
    }
}
```

## 21.2 回调处理服务

```java
package com.example.payment.webhook;

import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

/**
 * 支付回调处理。
 */
@Service
@RequiredArgsConstructor
public class PaymentWebhookService {

    private final PaymentNotifyVerifier verifier;
    private final PaymentNotifyRepository notifyRepository;
    private final OrderPaymentService orderPaymentService;

    @Transactional
    public void handleNotify(PaymentNotifyRequest request, String sign) {
        // 1. 验签，防止伪造回调
        verifier.verify(request, sign);

        // 2. 幂等判断：同一个 paymentNo 的成功回调只能处理一次
        if (notifyRepository.existsByPaymentNoAndStatus(request.getPaymentNo(), request.getTradeStatus())) {
            return;
        }

        // 3. 保存回调记录，便于审计和排查
        notifyRepository.save(PaymentNotifyRecord.from(request));

        // 4. 推进订单支付状态
        if ("SUCCESS".equals(request.getTradeStatus())) {
            orderPaymentService.markPaid(request.getOutTradeNo(), request.getPaymentNo());
        }
    }
}
```

## 21.3 Webhook 关键点

|点|要求|
|---|---|
|验签|必须|
|幂等|必须|
|事务|本地状态更新要一致|
|快速响应|不要让第三方长时间等待|
|异步化|重业务可丢 MQ 后处理|
|审计|原始请求、签名、处理结果要可追踪|

---

# 22. 文件上传下载：HTTP 里最容易写脏的部分

## 22.1 文件上传注意点

|问题|生产要求|
|---|---|
|文件大小|限制最大大小|
|文件类型|MIME + 后缀 + 文件头校验|
|存储|不建议直接存应用服务器|
|安全|防止木马、脚本、路径穿越|
|超时|上传下载要单独设置超时|
|断点续传|大文件场景需要|
|病毒扫描|企业内部系统可能要求|

## 22.2 文件下载注意点

- 不要一次性把大文件全部读入内存。
    
- 使用流式写出。
    
- 设置正确的 `Content-Type`。
    
- 设置 `Content-Disposition`。
    
- 防止任意文件读取漏洞。
    

---

# 23. HTTP 接口版本管理

生产系统接口会不断演进。

## 23.1 常见版本方式

|方式|示例|评价|
|---|---|---|
|URL Path|`/api/v1/orders`|清晰，常见|
|Header|`API-Version: 1`|更优雅，但调试略麻烦|
|Query|`?version=1`|简单，但不够规范|
|Content-Type|`application/vnd.xxx.v1+json`|成熟 API 设计中常见，但复杂|

## 23.2 推荐

对普通企业项目：

```text
/api/v1/xxx
/api/v2/xxx
```

就足够。

对开放平台：

```text
Header + OpenAPI 文档 + SDK 版本
```

更适合。

---

# 24. HTTP 接口测试体系

## 24.1 不要只靠 Postman 手测

生产级 HTTP Client 至少要有：

|测试类型|目的|
|---|---|
|单元测试|参数构造、签名、错误处理|
|Mock HTTP 测试|模拟第三方响应|
|契约测试|保证接口格式不破坏|
|集成测试|跑真实测试环境|
|压测|验证连接池、超时、熔断|
|故障注入|模拟超时、5xx、慢响应|

## 24.2 Mock 测试示意

```java
/**
 * 示例思路：
 * 真实项目可使用 MockWebServer、WireMock 或 Spring MockRestServiceServer。
 */
class PaymentHttpClientTest {

    // 重点测试：
    // 1. 正常 200 + SUCCESS；
    // 2. HTTP 500；
    // 3. HTTP 200 但业务 code 失败；
    // 4. 超时；
    // 5. 返回 JSON 字段缺失；
    // 6. 幂等键是否正确传递。
}
```

---

# 25. 生产部署：HTTP 调用和网关、K8s 的关系

## 25.1 典型链路

```mermaid
flowchart LR
    A[Client] --> B[CDN/WAF]
    B --> C[API Gateway]
    C --> D[Service A]
    D --> E[Service B]
    D --> F[External SaaS]
```

## 25.2 每层职责

|层|职责|
|---|---|
|CDN/WAF|防攻击、缓存静态内容、安全防护|
|API Gateway|鉴权、路由、限流、灰度、协议转换|
|Service|业务处理|
|Client Layer|出站 HTTP 调用治理|
|Observability|日志、指标、链路追踪|

不要把所有问题都丢给网关。

例如：

- 网关可以做入站限流。
    
- 但服务内部调用第三方支付，也需要自己的出站限流。
    
- 网关可以做统一鉴权。
    
- 但业务服务仍要校验资源权限。
    
- 网关可以记录访问日志。
    
- 但业务服务仍要记录业务关键日志。
    

---

# 26. 企业级 HTTP Client 模板

一个成熟项目里，HTTP 集成可以抽象成以下结构：

```text
infrastructure
└── client
    ├── common
    │   ├── HttpClientProperties.java
    │   ├── ClientErrorDecoder.java
    │   ├── ClientLogInterceptor.java
    │   ├── ClientMetricsInterceptor.java
    │   └── SensitiveDataMasker.java
    │
    ├── payment
    │   ├── PaymentHttpClient.java
    │   ├── PaymentApi.java
    │   ├── PaymentSigner.java
    │   ├── dto
    │   │   ├── CreatePaymentRequest.java
    │   │   └── CreatePaymentResponse.java
    │   └── config
    │       └── PaymentClientConfig.java
    │
    ├── sms
    │   ├── SmsHttpClient.java
    │   └── dto
    │
    └── ai
        ├── ModelGateway.java
        ├── OpenAiClient.java
        ├── DeepSeekClient.java
        └── dto
```

核心原则：

> 每个外部系统一个 Client 包，公共能力沉淀到 common。

---

# 27. 常见反模式

## 27.1 Controller 直接调第三方 HTTP

错误：

```java
@PostMapping("/pay")
public String pay() {
    return restClient.post()
            .uri("https://xxx/pay")
            .retrieve()
            .body(String.class);
}
```

问题：

- Controller 变厚。
    
- 无法复用。
    
- 无法测试。
    
- 异常处理混乱。
    
- 外部协议污染业务入口。
    

---

## 27.2 HTTP DTO 直接当领域对象用

错误：

```java
Order order = restClient.get()
        .uri("/third-party/order")
        .retrieve()
        .body(Order.class);
```

问题：

- 第三方字段变化影响内部领域模型。
    
- 外部错误语义侵入内部系统。
    
- 后续更换供应商代价很高。
    

---

## 27.3 所有异常 catch 后返回 null

错误：

```java
try {
    return call();
} catch (Exception e) {
    return null;
}
```

问题：

- 上层不知道是超时、签名错、业务失败还是解析失败。
    
- 空指针延迟爆炸。
    
- 监控无法识别真实故障。
    

---

## 27.4 不区分 4xx 和 5xx

错误：

```java
catch (Exception e) {
    retry();
}
```

问题：

- 401 鉴权失败重试没有意义。
    
- 400 参数错误重试没有意义。
    
- 签名错误重试会放大流量。
    
- 只有部分网络错误、5xx、429 适合策略性重试。
    

---

# 28. 面试表达：怎么讲 HTTP 框架实战？

可以这样回答：

> 我理解 HTTP 调用在生产系统中不是简单发请求，而是外部系统集成能力。实际落地时，我会把 HTTP 调用从业务逻辑中剥离出来，放到独立 Client 或 Gateway 层。
> 
> 技术选型上，如果是 Spring Boot 3.x 的同步调用，我会优先使用 RestClient；如果是高并发聚合、流式响应或响应式链路，会使用 WebClient；如果接口很多，会考虑 Spring HTTP Interface 或 OpenFeign。
> 
> 生产治理上，我会统一配置 baseUrl、超时、连接池、鉴权、签名、日志脱敏、错误解码，并接入 Resilience4j 做重试、熔断、限流和隔舱。同时通过 Micrometer/OpenTelemetry 采集 HTTP 调用的耗时、状态码、异常、traceId，保证故障可定位。
> 
> 对写接口尤其是支付、扣库存、发券这类接口，我不会盲目重试，而是先设计幂等键和状态机。对于 Webhook 回调，我会重点处理验签、幂等、事务和审计日志。

这比单纯说“我用过 RestTemplate/OkHttp”高级很多。

---

# 29. 关键词总结

**核心关键词：**

- HTTP Client
    
- RestClient
    
- WebClient
    
- HTTP Interface
    
- OpenFeign
    
- Timeout
    
- Retry
    
- CircuitBreaker
    
- RateLimiter
    
- Bulkhead
    
- Idempotency-Key
    
- Webhook
    
- Signature
    
- Observability
    
- Micrometer
    
- OpenTelemetry
    
- DTO 隔离
    
- Gateway 防腐层
    
- API Versioning
    
- MockServer
    
- Contract Test
    

**一句话总结：**

> 生产级 HTTP 框架实战的核心，不是学会某个 HTTP 工具类，而是建立一套“外部系统集成治理能力”：调用要规范，失败要可控，问题要可查，接口要可演进，业务要不被第三方拖垮。