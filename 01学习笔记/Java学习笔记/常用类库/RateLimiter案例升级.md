[xfgRateLimiter限流 —— 通过切面对单个用户进行限流和黑名单处理](https://bugstack.cn/md/road-map/ratelimiter.html)
# Spring Boot 3 + Redis Lua：单用户限流 + 自动黑名单教学案例

## 0. 场景

参考的原案例核心是：**通过 AOP 自定义注解，对接口按浏览器指纹 ID 维度限流；当某个 ID 频繁触发限流后，自动加入黑名单**。原文里也明确提到使用 `key`、`permitsPerSecond`、`blacklistCount`、`fallbackMethod` 这些配置项，并通过切面拦截使用注解的方法。([BugStack](https://bugstack.cn/md/road-map/ratelimiter.html "RateLimiter | 小傅哥 bugstack 虫洞栈"))

但新项目不建议继续用 Guava `RateLimiter` 做这个案例，原因很直接：

|旧方案|问题|
|---|---|
|Guava `RateLimiter`|单 JVM 内存限流，多实例部署时不共享|
|Guava `Cache` 记录黑名单|也是本地内存，多节点不一致|
|AOP 方案本身|可以保留，适合做教学和业务级限流|

新版本设计：

```text
Spring Boot 3.x
+ Java 21
+ AOP 自定义注解
+ Redis Lua 原子脚本
+ 按用户 / 设备 / IP / 参数限流
+ 自动黑名单
+ 统一返回 429
```

---

# 1. 方案设计

## 1.1 业务目标

以登录验证码接口为例：

```http
POST /api/sms/send
Header: X-Device-Id: browser-fingerprint-id
Body:
{
  "phone": "13800000000"
}
```

限制规则：

```text
同一个 deviceId：
10 秒内最多请求 3 次；
如果连续触发限流 5 次；
自动拉黑 30 分钟。
```

---

## 1.2 为什么用 Redis Lua？

因为限流本质上需要三个能力：

|能力|说明|
|---|---|
|计数原子性|并发请求下不能计数错乱|
|多实例共享|多个 Spring Boot 实例要看到同一份限流状态|
|自动过期|限流窗口和黑名单都要自动释放|

Redis + Lua 可以把这些操作放在一个脚本里执行，避免：

```text
GET -> 判断 -> INCR -> EXPIRE
```

这种多命令并发不一致问题。

---

# 2. 工程依赖

```xml
<dependencies>
    <!-- Web 接口 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- AOP 切面 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>

    <!-- Redis 客户端，底层默认 Lettuce -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- 参数校验 -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Lombok 可选，不想用可以删掉 -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

# 3. application.yml

```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      database: 0
      timeout: 2s
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 2

server:
  port: 8080
```

---

# 4. 自定义注解：`@AccessLimit`

这个注解放在 Controller 方法上。

```java
package com.example.ratelimit.annotation;

import java.lang.annotation.*;

/**
 * 接口访问限流注解。
 *
 * 设计目标：
 * 1. 通过 key 表达式指定限流维度，例如请求头、用户 ID、手机号、IP。
 * 2. 支持窗口期限流，例如 10 秒内最多 3 次。
 * 3. 支持触发限流多次后自动加入黑名单。
 * 4. 支持指定 fallback 方法，方便返回业务友好的降级结果。
 */
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface AccessLimit {

    /**
     * 限流 key 表达式。
     *
     * 这里使用 Spring SpEL。
     * 示例：
     * 1. "#request.deviceId"
     * 2. "#request.phone"
     * 3. "#deviceId"
     */
    String key();

    /**
     * 业务场景名称。
     *
     * 用于拼 Redis key，避免不同接口之间互相影响。
     */
    String biz();

    /**
     * 窗口期内允许的最大请求次数。
     */
    int permits();

    /**
     * 限流窗口秒数。
     *
     * 例如 windowSeconds = 10，permits = 3：
     * 表示 10 秒内最多访问 3 次。
     */
    int windowSeconds();

    /**
     * 触发限流多少次后加入黑名单。
     *
     * 0 表示不启用黑名单。
     */
    int blacklistThreshold() default 0;

    /**
     * 黑名单有效期，单位秒。
     */
    int blacklistSeconds() default 1800;

    /**
     * 降级方法名。
     *
     * 方法需要和原方法在同一个类中。
     * 参数可以和原方法一致。
     */
    String fallbackMethod() default "";
}
```

---

# 5. Redis Lua 脚本

## 5.1 脚本返回约定

```text
0 = 允许访问
1 = 被限流
2 = 黑名单拦截
```

## 5.2 Lua 脚本

```java
package com.example.ratelimit.infrastructure;

public final class RateLimitLuaScript {

    private RateLimitLuaScript() {
    }

    /**
     * KEYS[1] = limitKey      限流计数 key
     * KEYS[2] = blacklistKey  黑名单 key
     * KEYS[3] = punishKey     限流处罚计数 key
     *
     * ARGV[1] = permits              窗口期最大访问次数
     * ARGV[2] = windowSeconds        限流窗口秒数
     * ARGV[3] = blacklistThreshold   触发多少次限流后加入黑名单
     * ARGV[4] = blacklistSeconds     黑名单秒数
     *
     * 返回：
     * 0 = 通过
     * 1 = 限流
     * 2 = 黑名单
     */
    public static final String SCRIPT = """
            -- 1. 先判断是否已经在黑名单中
            if redis.call('exists', KEYS[2]) == 1 then
                return 2
            end
            
            local permits = tonumber(ARGV[1])
            local windowSeconds = tonumber(ARGV[2])
            local blacklistThreshold = tonumber(ARGV[3])
            local blacklistSeconds = tonumber(ARGV[4])
            
            -- 2. 对限流 key 做自增计数
            local current = redis.call('incr', KEYS[1])
            
            -- 3. 第一次访问时设置窗口过期时间
            if current == 1 then
                redis.call('expire', KEYS[1], windowSeconds)
            end
            
            -- 4. 没超过阈值，允许访问
            if current <= permits then
                return 0
            end
            
            -- 5. 超过阈值，说明本次请求被限流
            if blacklistThreshold > 0 then
                local punish = redis.call('incr', KEYS[3])
            
                -- 处罚计数使用同一个窗口周期，避免永久累加
                if punish == 1 then
                    redis.call('expire', KEYS[3], windowSeconds)
                end
            
                -- 6. 处罚次数达到阈值，加入黑名单
                if punish >= blacklistThreshold then
                    redis.call('set', KEYS[2], '1', 'EX', blacklistSeconds)
                    return 2
                end
            end
            
            return 1
            """;
}
```

---

# 6. 限流执行器：封装 Redis 调用

```java
package com.example.ratelimit.infrastructure;

import lombok.RequiredArgsConstructor;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.stereotype.Component;

import java.util.List;

/**
 * Redis 限流执行器。
 *
 * 这个类只负责和 Redis 交互，不关心 AOP、不关心 Controller。
 * 这样后续要换成 Bucket4j、Sentinel、网关限流时，替换成本较低。
 */
@Component
@RequiredArgsConstructor
public class RedisRateLimitExecutor {

    private final StringRedisTemplate stringRedisTemplate;

    private static final DefaultRedisScript<Long> RATE_LIMIT_SCRIPT;

    static {
        RATE_LIMIT_SCRIPT = new DefaultRedisScript<>();
        RATE_LIMIT_SCRIPT.setScriptText(RateLimitLuaScript.SCRIPT);
        RATE_LIMIT_SCRIPT.setResultType(Long.class);
    }

    public RateLimitResult tryAcquire(RateLimitCommand command) {
        String limitKey = buildLimitKey(command.biz(), command.identity());
        String blacklistKey = buildBlacklistKey(command.biz(), command.identity());
        String punishKey = buildPunishKey(command.biz(), command.identity());

        Long result = stringRedisTemplate.execute(
                RATE_LIMIT_SCRIPT,
                List.of(limitKey, blacklistKey, punishKey),
                String.valueOf(command.permits()),
                String.valueOf(command.windowSeconds()),
                String.valueOf(command.blacklistThreshold()),
                String.valueOf(command.blacklistSeconds())
        );

        if (result == null) {
            // Redis 执行异常时，不建议默认放行还是默认拦截要看业务。
            // 登录、验证码、支付等安全接口建议 fail-close。
            return RateLimitResult.LIMITED;
        }

        return switch (result.intValue()) {
            case 0 -> RateLimitResult.ALLOWED;
            case 1 -> RateLimitResult.LIMITED;
            case 2 -> RateLimitResult.BLACKLISTED;
            default -> RateLimitResult.LIMITED;
        };
    }

    private String buildLimitKey(String biz, String identity) {
        return "access-limit:" + biz + ":counter:" + identity;
    }

    private String buildBlacklistKey(String biz, String identity) {
        return "access-limit:" + biz + ":blacklist:" + identity;
    }

    private String buildPunishKey(String biz, String identity) {
        return "access-limit:" + biz + ":punish:" + identity;
    }

    public record RateLimitCommand(
            String biz,
            String identity,
            int permits,
            int windowSeconds,
            int blacklistThreshold,
            int blacklistSeconds
    ) {
    }

    public enum RateLimitResult {
        ALLOWED,
        LIMITED,
        BLACKLISTED
    }
}
```

---

# 7. SpEL 解析器：从方法参数中取限流 key

比如注解里写：

```java
@AccessLimit(key = "#request.deviceId", ...)
```

就要能从方法参数里解析出 `request.deviceId`。

```java
package com.example.ratelimit.support;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.core.DefaultParameterNameDiscoverer;
import org.springframework.expression.ExpressionParser;
import org.springframework.expression.spel.standard.SpelExpressionParser;
import org.springframework.expression.spel.support.StandardEvaluationContext;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

/**
 * SpEL 解析工具。
 *
 * 作用：
 * 将注解中的 key 表达式解析成真实业务值。
 *
 * 示例：
 * key = "#request.deviceId"
 * 方法参数：
 * sendSms(SendSmsRequest request)
 * 最终解析为：
 * request.getDeviceId()
 */
@Component
public class RateLimitKeyResolver {

    private final ExpressionParser expressionParser = new SpelExpressionParser();

    private final DefaultParameterNameDiscoverer parameterNameDiscoverer =
            new DefaultParameterNameDiscoverer();

    public String resolve(String expression, ProceedingJoinPoint joinPoint) {
        Method method = ((MethodSignature) joinPoint.getSignature()).getMethod();
        Object[] args = joinPoint.getArgs();

        String[] parameterNames = parameterNameDiscoverer.getParameterNames(method);
        if (parameterNames == null || parameterNames.length == 0) {
            throw new IllegalArgumentException("无法解析方法参数名，请确认编译参数开启 -parameters");
        }

        StandardEvaluationContext context = new StandardEvaluationContext();

        // 把方法参数名注册到 SpEL 上下文中
        // 例如：sendSms(SendSmsRequest request)
        // 则可以在表达式中使用 #request
        for (int i = 0; i < parameterNames.length; i++) {
            context.setVariable(parameterNames[i], args[i]);
        }

        Object value = expressionParser.parseExpression(expression).getValue(context);
        if (value == null) {
            throw new IllegalArgumentException("限流 key 解析结果为空，expression=" + expression);
        }

        return String.valueOf(value);
    }
}
```

> Maven 编译建议加 `-parameters`，否则运行时可能拿不到方法参数名。

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <source>21</source>
                <target>21</target>
                <parameters>true</parameters>
            </configuration>
        </plugin>
    </plugins>
</build>
```

---

# 8. AOP 切面：核心拦截逻辑

```java
package com.example.ratelimit.aspect;

import com.example.ratelimit.annotation.AccessLimit;
import com.example.ratelimit.infrastructure.RedisRateLimitExecutor;
import com.example.ratelimit.support.RateLimitKeyResolver;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;

import static com.example.ratelimit.infrastructure.RedisRateLimitExecutor.*;

/**
 * 接口访问限流切面。
 *
 * 职责：
 * 1. 拦截带有 @AccessLimit 的方法。
 * 2. 解析限流身份，例如 deviceId、userId、phone、ip。
 * 3. 调用 Redis Lua 做原子限流判断。
 * 4. 被限流或黑名单时，走 fallback。
 */
@Slf4j
@Aspect
@Component
@RequiredArgsConstructor
public class AccessLimitAspect {

    private final RateLimitKeyResolver keyResolver;
    private final RedisRateLimitExecutor rateLimitExecutor;

    @Around("@annotation(accessLimit)")
    public Object around(ProceedingJoinPoint joinPoint, AccessLimit accessLimit) throws Throwable {
        String identity = keyResolver.resolve(accessLimit.key(), joinPoint);

        RateLimitCommand command = new RateLimitCommand(
                accessLimit.biz(),
                identity,
                accessLimit.permits(),
                accessLimit.windowSeconds(),
                accessLimit.blacklistThreshold(),
                accessLimit.blacklistSeconds()
        );

        RateLimitResult result = rateLimitExecutor.tryAcquire(command);

        if (result == RateLimitResult.ALLOWED) {
            return joinPoint.proceed();
        }

        log.warn(
                "接口访问被拦截，biz={}, identity={}, result={}",
                accessLimit.biz(),
                identity,
                result
        );

        if (!accessLimit.fallbackMethod().isBlank()) {
            return invokeFallback(joinPoint, accessLimit.fallbackMethod(), result);
        }

        throw new AccessLimitedException(result);
    }

    private Object invokeFallback(ProceedingJoinPoint joinPoint,
                                  String fallbackMethod,
                                  RateLimitResult result) throws Exception {
        Object target = joinPoint.getTarget();
        Object[] args = joinPoint.getArgs();

        Method method = findFallbackMethod(target.getClass(), fallbackMethod, args);

        // 这里保持 fallback 参数和原方法一致，简单直观。
        // 如果要把 result 也传进去，可以扩展 fallback 方法签名。
        return method.invoke(target, args);
    }

    private Method findFallbackMethod(Class<?> targetClass,
                                      String fallbackMethod,
                                      Object[] args) throws NoSuchMethodException {
        Class<?>[] parameterTypes = new Class<?>[args.length];

        for (int i = 0; i < args.length; i++) {
            parameterTypes[i] = args[i].getClass();
        }

        Method method = targetClass.getMethod(fallbackMethod, parameterTypes);
        method.setAccessible(true);
        return method;
    }

    public static class AccessLimitedException extends RuntimeException {

        private final RateLimitResult result;

        public AccessLimitedException(RateLimitResult result) {
            super("接口访问过于频繁：" + result);
            this.result = result;
        }

        public RateLimitResult getResult() {
            return result;
        }
    }
}
```

---

# 9. 统一返回对象

```java
package com.example.common;

public record ApiResponse<T>(
        int code,
        String message,
        T data
) {

    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(0, "success", data);
    }

    public static <T> ApiResponse<T> fail(int code, String message) {
        return new ApiResponse<>(code, message, null);
    }
}
```

---

# 10. 全局异常处理

```java
package com.example.common;

import com.example.ratelimit.aspect.AccessLimitAspect.AccessLimitedException;
import com.example.ratelimit.infrastructure.RedisRateLimitExecutor.RateLimitResult;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

/**
 * 全局异常处理。
 *
 * 被限流时统一返回 HTTP 429。
 * Spring Cloud Gateway 的 RequestRateLimiter 默认也是在不允许请求时返回 HTTP 429。
 * 这样和常见网关限流语义保持一致。
 */
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ResponseStatus(HttpStatus.TOO_MANY_REQUESTS)
    @ExceptionHandler(AccessLimitedException.class)
    public ApiResponse<Void> handleAccessLimited(AccessLimitedException ex) {
        if (ex.getResult() == RateLimitResult.BLACKLISTED) {
            return ApiResponse.fail(42901, "访问过于频繁，已被临时限制");
        }

        return ApiResponse.fail(42900, "请求过于频繁，请稍后再试");
    }

    @ResponseStatus(HttpStatus.BAD_REQUEST)
    @ExceptionHandler(IllegalArgumentException.class)
    public ApiResponse<Void> handleIllegalArgument(IllegalArgumentException ex) {
        return ApiResponse.fail(40000, ex.getMessage());
    }
}
```

Spring Cloud Gateway 的 `RequestRateLimiter` 在请求不允许通过时默认返回 `HTTP 429 - Too Many Requests`，所以业务接口限流也建议统一使用 429 语义。([Home](https://docs.spring.io/spring-cloud-gateway/reference/spring-cloud-gateway-server-webflux/gatewayfilter-factories/requestratelimiter-factory.html?utm_source=chatgpt.com "RequestRateLimiter GatewayFilter Factory"))

---

# 11. 业务案例：短信验证码接口

## 11.1 请求对象

```java
package com.example.sms.api;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Pattern;

/**
 * 发送短信验证码请求。
 *
 * deviceId 可以来自浏览器指纹、App 设备 ID、客户端生成的匿名 ID。
 * 注意：deviceId 不能作为唯一安全凭证，只能作为风控维度之一。
 */
public record SendSmsRequest(

        @NotBlank(message = "手机号不能为空")
        @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
        String phone,

        @NotBlank(message = "设备 ID 不能为空")
        String deviceId
) {
}
```

---

## 11.2 Controller

```java
package com.example.sms.api;

import com.example.common.ApiResponse;
import com.example.ratelimit.annotation.AccessLimit;
import com.example.sms.application.SmsService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/sms")
@RequiredArgsConstructor
public class SmsController {

    private final SmsService smsService;

    /**
     * 规则：
     * 1. 按 deviceId 限流。
     * 2. 10 秒内最多请求 3 次。
     * 3. 如果连续触发限流 5 次，加入黑名单 30 分钟。
     *
     * 注意：
     * 这里限制的是 deviceId，不是 phone。
     * 生产环境可以组合维度：
     * deviceId + phone + ip + userAgent。
     */
    @PostMapping("/send")
    @AccessLimit(
            biz = "sms-send",
            key = "#request.deviceId",
            permits = 3,
            windowSeconds = 10,
            blacklistThreshold = 5,
            blacklistSeconds = 1800,
            fallbackMethod = "sendSmsFallback"
    )
    public ApiResponse<Void> sendSms(@Valid @RequestBody SendSmsRequest request) {
        smsService.sendLoginCode(request.phone(), request.deviceId());
        return ApiResponse.success(null);
    }

    /**
     * fallback 方法。
     *
     * 为了教学简单，这里参数和原方法保持一致。
     * 实际生产可以直接抛异常交给全局异常处理，
     * 不一定每个接口都写 fallback。
     */
    public ApiResponse<Void> sendSmsFallback(SendSmsRequest request) {
        return ApiResponse.fail(42900, "验证码发送过于频繁，请稍后再试");
    }
}
```

---

## 11.3 Service

```java
package com.example.sms.application;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class SmsService {

    public void sendLoginCode(String phone, String deviceId) {
        // 1. 生成验证码
        String code = generateCode();

        // 2. 存储验证码
        // 实际项目中建议存 Redis：
        // key = sms:login:code:{phone}
        // value = code
        // ttl = 5 minutes
        saveCode(phone, code);

        // 3. 调用短信服务商
        sendToProvider(phone, code);

        log.info("发送登录验证码成功，phone={}, deviceId={}", maskPhone(phone), deviceId);
    }

    private String generateCode() {
        // 教学示例，生产中建议使用更规范的随机数工具
        return String.valueOf((int) ((Math.random() * 9 + 1) * 100000));
    }

    private void saveCode(String phone, String code) {
        // 这里省略 Redis 写入逻辑
        log.info("保存验证码，phone={}, code={}", maskPhone(phone), code);
    }

    private void sendToProvider(String phone, String code) {
        // 这里省略真实短信供应商调用逻辑
        log.info("调用短信供应商，phone={}", maskPhone(phone));
    }

    private String maskPhone(String phone) {
        if (phone == null || phone.length() < 7) {
            return "unknown";
        }
        return phone.substring(0, 3) + "****" + phone.substring(7);
    }
}
```

---

# 12. 测试请求

```bash
curl -X POST http://localhost:8080/api/sms/send \
  -H "Content-Type: application/json" \
  -d '{"phone":"13800000000","deviceId":"browser-fp-001"}'
```

快速执行 4 次。

前 3 次：

```json
{
  "code": 0,
  "message": "success",
  "data": null
}
```

第 4 次：

```json
{
  "code": 42900,
  "message": "验证码发送过于频繁，请稍后再试",
  "data": null
}
```

继续触发多次后：

```json
{
  "code": 42901,
  "message": "访问过于频繁，已被临时限制",
  "data": null
}
```

---

# 13. Redis Key 观察

```bash
redis-cli keys "access-limit:*"
```

你会看到类似：

```text
access-limit:sms-send:counter:browser-fp-001
access-limit:sms-send:punish:browser-fp-001
access-limit:sms-send:blacklist:browser-fp-001
```

查看 TTL：

```bash
redis-cli ttl access-limit:sms-send:blacklist:browser-fp-001
```

---

# 14. 为什么不用 Guava RateLimiter？

## 14.1 Guava RateLimiter 是单机限流

```text
user -> Nginx -> app-1
              -> app-2
              -> app-3
```

如果每个实例都允许：

```text
3 次 / 10 秒
```

那 3 个实例总共就是：

```text
9 次 / 10 秒
```

这不是用户维度全局限流。

---

## 14.2 Guava Cache 黑名单也是本地的

如果用户在 `app-1` 被拉黑：

```text
app-1 黑名单：有
app-2 黑名单：无
app-3 黑名单：无
```

用户只要打到其他实例，仍然可能继续访问。

---

## 14.3 新项目更合理的选择

|场景|推荐方案|
|---|---|
|单体应用、教学 Demo|AOP + Redis Lua|
|多实例业务服务|AOP + Redis Lua / Bucket4j Redis|
|网关统一限流|Spring Cloud Gateway RequestRateLimiter|
|服务熔断、重试、隔离、限流组合|Resilience4j|
|高并发风控规则|Redis + Lua + 风控策略服务|
|企业级服务治理|Sentinel / 网关 / WAF / 风控系统|

Bucket4j 是一个基于 token bucket 的 Java 限流库，官方文档强调它不只是简单令牌桶实现，还支持多限制、透支等扩展模型；它也支持集群 JVM 场景下的分布式限流能力。([Bucket4j](https://bucket4j.com/8.7.0/toc.html?utm_source=chatgpt.com "Bucket4j 8.7.0 Reference"))

Resilience4j 则更偏服务韧性治理，它提供 CircuitBreaker、RateLimiter、Retry、Bulkhead 等可组合能力，适合保护远程调用或下游资源。([GitHub](https://github.com/resilience4j/resilience4j?utm_source=chatgpt.com "Resilience4j is a fault tolerance library designed for Java8 ..."))

---

# 15. 生产级改造建议

## 15.1 限流维度不要只依赖 deviceId

浏览器指纹可以伪造，不能单独作为安全依据。

建议组合：

```text
deviceId
+ phone
+ ip
+ userAgent
+ userId
+ endpoint
```

比如：

```java
@AccessLimit(
    biz = "sms-send",
    key = "#request.deviceId + ':' + #request.phone",
    permits = 3,
    windowSeconds = 10,
    blacklistThreshold = 5,
    blacklistSeconds = 1800
)
```

---

## 15.2 不同接口用不同策略

|接口|建议策略|
|---|---|
|登录验证码|严格限流 + 黑名单|
|登录提交|IP + 账号 + 设备组合限流|
|查询接口|宽松限流|
|支付接口|不建议简单限流，应该做风控|
|管理后台导出|用户级限流 + 异步任务|
|AI 接口|用户级 quota + token 级计量|

---

## 15.3 Redis 异常时怎么处理？

这是生产里必须明确的问题。

|业务|Redis 异常策略|
|---|---|
|登录验证码|fail-close，默认拒绝|
|普通查询接口|fail-open，默认放行|
|支付、提现|fail-close|
|后台管理接口|看安全等级|

当前代码里选择的是：

```java
return RateLimitResult.LIMITED;
```

也就是 **fail-close**。

---

## 15.4 fallback 不要滥用

教学案例保留了 `fallbackMethod`，是为了对齐“注解 + 切面 + fallback”的案例结构。

但真实项目里更推荐：

```text
切面抛 AccessLimitedException
GlobalExceptionHandler 统一返回 429
```

否则每个接口都写 fallback，维护成本会变高。

---

# 16. 这个案例的工程价值

这个版本比 Guava `RateLimiter + Cache` 更接近生产，因为它解决了：

|问题|新方案|
|---|---|
|单机限流|Redis 共享状态|
|多实例不一致|Redis key 全局一致|
|并发计数不准|Lua 原子执行|
|黑名单丢失|Redis TTL 黑名单|
|配置不灵活|注解参数化|
|和业务强耦合|AOP 横切处理|
|返回不统一|全局异常处理|

---

# 17. 最终代码结构

```text
src/main/java/com/example
├── common
│   ├── ApiResponse.java
│   └── GlobalExceptionHandler.java
│
├── ratelimit
│   ├── annotation
│   │   └── AccessLimit.java
│   ├── aspect
│   │   └── AccessLimitAspect.java
│   ├── infrastructure
│   │   ├── RedisRateLimitExecutor.java
│   │   └── RateLimitLuaScript.java
│   └── support
│       └── RateLimitKeyResolver.java
│
└── sms
    ├── api
    │   ├── SendSmsRequest.java
    │   └── SmsController.java
    └── application
        └── SmsService.java
```

---

# 18. 总结

这个案例真正要学的不是“限流 API 怎么调”，而是这套工程结构：

```text
注解定义限流规则
AOP 识别受保护接口
SpEL 解析限流维度
Redis Lua 保证原子计数
TTL 管理窗口和黑名单
统一异常返回 429
```

一句话：

> **Guava RateLimiter 适合理解单机令牌桶思想；新项目做用户级接口限流，更推荐 Spring Boot 3 + Redis Lua / Bucket4j / 网关限流。业务接口上的精细化限流，用 AOP + Redis Lua 是一个非常适合教学和落地的方案。**