# prompt
```
你是一名资深 Java 后端架构师和技术面试官。请以“教学 + 面试准备 + 生产实践”的方式，系统讲解 UUID。

我的目标：
1. 掌握 UUID 的基础概念和核心原理；
2. 知道 Java 项目中 UUID 的典型使用方式；
3. 能写出生产级代码；
4. 能识别常见坑点；
5. 能应对面试中的高频问题。

请按以下结构输出：

## 1. UUID 是什么
用通俗但准确的方式解释 UUID 的定义、格式、长度、组成方式，以及它解决什么问题。
不要像百科一样展开历史背景，只讲 Java 后端开发必须知道的内容。

## 2. UUID 的核心原理
重点解释：
- UUID 为什么能做到“近似全局唯一”；
- UUID v1、v3、v4、v5、v7 的差异；
- Java `UUID.randomUUID()` 属于哪一种；
- UUID 是否真的绝对唯一；
- UUID 和数据库自增 ID、雪花算法 ID 的区别。

要求：
- 不要堆概念；
- 每个版本只讲实际开发中需要知道的重点；
- 用对比表总结。

## 3. Java 中如何使用 UUID
结合 Java 代码讲解：
- 生成 UUID；
- 去掉中划线；
- UUID 与 String 的互转；
- 校验 UUID 格式；
- 作为业务唯一标识；
- 作为请求追踪 traceId；
- 作为幂等键 idempotencyKey；
- 作为数据库主键时的写法。

代码要求：
- 使用现代 Java 写法；
- 包含必要的异常处理；
- 不要只给 demo，要给可用于项目中的工具类或服务类；
- 代码要有简短注释。

## 4. 具体业务案例
请给出至少 3 个 Java 后端项目中的真实使用场景，例如：
- 用户注册生成 userId；
- 订单请求生成幂等键；
- 分布式调用生成 traceId；
- 文件上传生成唯一文件名；
- 数据库主键设计。

每个案例需要包含：
- 业务背景；
- 为什么适合或不适合用 UUID；
- 生产级 Java 示例代码；
- 数据库存储建议；
- 可能的坑点。

## 5. UUID 在数据库中的使用
重点讲：
- UUID 作为主键的优缺点；
- MySQL 中 `CHAR(36)`、`CHAR(32)`、`BINARY(16)` 的区别；
- UUID 对索引、页分裂、插入性能的影响；
- 为什么随机 UUID 不适合高并发写入表的聚簇主键；
- 什么情况下可以用 UUID 做主键；
- 什么情况下更推荐雪花算法、数据库自增 ID 或 UUID v7。

请给出 Java + MySQL 的推荐存储方案和示例 SQL。

## 6. 生产级最佳实践
总结 Java 项目中使用 UUID 的推荐规范：
- 是否保留中划线；
- 是否统一小写；
- 是否暴露给前端；
- 是否适合做排序字段；
- 是否适合做日志 traceId；
- 是否适合作为安全 token；
- 是否需要加密或脱敏；
- 如何封装工具类。

要求给出明确建议，而不是“视情况而定”。

## 7. 常见坑点
列出并解释 Java 后端中 UUID 的常见坑点，例如：
- 把 UUID 当成有序 ID；
- 用随机 UUID 做 MySQL 聚簇主键；
- 误以为 UUID 绝对不会冲突；
- 用 UUID 替代安全 token；
- 在日志链路中格式不统一；
- `replace("-", "")` 滥用；
- 大小写不统一；
- UUID 和业务编码混用。

每个坑点都要说明：
- 错在哪里；
- 会造成什么后果；
- 正确做法。

## 8. 面试高频问题
请整理 Java 后端面试中关于 UUID 的高频问题，并给出标准回答。
问题包括但不限于：
- UUID 是什么？
- Java 的 `UUID.randomUUID()` 是怎么生成的？
- UUID 会不会重复？
- UUID 能不能作为数据库主键？
- UUID 和雪花算法有什么区别？
- 为什么 UUID 不适合作为 MySQL 聚簇索引主键？
- UUID 适合做分布式 ID 吗？
- UUID 能不能做 token？
- UUID v4 和 v7 有什么区别？
- 你在项目中怎么用 UUID？

回答要求：
- 面试风格；
- 简洁但有技术深度；
- 能体现生产经验。

## 9. 最终总结
最后用一张表总结：
- 使用场景；
- 是否推荐使用 UUID；
- 推荐 UUID 版本；
- Java 实现方式；
- 数据库存储方式；
- 替代方案。

整体要求：
- 面向 Java 后端开发者；
- 不要百科式罗列；
- 重点放在项目使用、生产实践和面试表达；
- 代码要能落地；
- 解释要循序渐进；
- 避免空泛结论；
- 对关键点给出明确判断。
```

---

# 0. 先给结论

对 Java 后端开发来说，UUID 不是“一个万能 ID 方案”，而是一个**去中心化生成唯一标识的工具**。

最重要的工程判断如下：

|场景|结论|
|---|---|
|普通业务唯一标识|可以用 UUID v4|
|日志 traceId|可以用 UUID，但建议统一为 `32位小写无中划线`|
|幂等键|可以用 UUID，但必须配合数据库唯一索引 / Redis 原子写入|
|MySQL 高并发大表聚簇主键|不推荐随机 UUID v4|
|分布式数据库主键|优先雪花 ID / UUID v7 / 数据库自增 + 业务唯一键|
|安全 token|不要直接把 UUID 当安全 token|
|文件名去重|UUID 很适合|
|前端可见资源 ID|可以用 UUID，但注意枚举风险和权限校验|

UUID 标准本质上是 128 bit 标识符，RFC 9562 明确说明 UUID 固定为 128 bit，并建议数据库中可行时优先按底层 128 bit 二进制存储，而不是冗长文本。([RFC 编辑器](https://www.rfc-editor.org/rfc/rfc9562.html "RFC 9562: Universally Unique IDentifiers (UUIDs)")) Java `UUID.randomUUID()` 生成的是 **UUID v4**，Oracle JDK 文档明确说明它是“type 4 pseudo randomly generated UUID”，并使用强伪随机数生成器。([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/util/UUID.html "UUID (Java Platform SE 8 )"))

---

# 1. UUID 是什么

## 1.1 一句话定义

**UUID，Universally Unique Identifier，通用唯一标识符。**

它的目标是：  
在没有中心化发号器、没有数据库自增 ID、没有 Redis 计数器的情况下，应用程序自己就能生成一个**极低概率冲突**的全局唯一 ID。

典型格式：

```text
550e8400-e29b-41d4-a716-446655440000
```

它有几个关键特征：

|项目|说明|
|---|---|
|本质|128 bit 二进制值|
|标准字符串长度|36 个字符|
|十六进制字符数|32 个 hex 字符|
|中划线数量|4 个|
|常见格式|`8-4-4-4-12`|
|是否全局唯一|工程上近似唯一，不是数学绝对唯一|
|Java 类型|`java.util.UUID`|
|常见生成方式|`UUID.randomUUID()`|

标准字符串结构：

```text
xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx
```

其中：

- `M` 表示 UUID 版本；
    
- `N` 表示 UUID variant；
    
- 剩下的大部分内容由时间、随机数、哈希、节点信息等组成，取决于 UUID 版本。
    

Java 文档中也明确了 UUID 字符串格式是由 5 段十六进制数字和中划线组成。([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/util/UUID.html "UUID (Java Platform SE 8 )"))

---

## 1.2 UUID 解决什么问题

UUID 主要解决三个问题：

### 问题 1：分布式系统中不想依赖中心发号器

例如多个服务实例同时创建文件、订单草稿、导入任务、异步消息，不想每次都访问数据库拿 ID。

```java
String importTaskId = UUID.randomUUID().toString();
```

### 问题 2：外部可见 ID 不想暴露数据库自增规律

自增 ID 有枚举风险：

```text
/users/10001
/users/10002
/users/10003
```

别人可能猜到系统规模和资源路径。

UUID 更适合做外部资源标识：

```text
/users/550e8400-e29b-41d4-a716-446655440000
```

但注意：**UUID 不能替代权限校验。**

### 问题 3：跨系统传递唯一标识

例如：

- traceId；
    
- requestId；
    
- fileId；
    
- idempotencyKey；
    
- messageId；
    
- importBatchId；
    
- callbackId。
    

---

# 2. UUID 的核心原理

## 2.1 UUID 为什么能做到“近似全局唯一”

UUID 的唯一性来自不同机制：

|类型|唯一性来源|
|---|---|
|时间型 UUID|时间戳 + 节点信息 + 时钟序列|
|名称型 UUID|namespace + name 的哈希|
|随机型 UUID|足够大的随机空间|
|时间有序型 UUID|时间戳 + 随机数 / 计数器|

以 Java 最常用的 UUID v4 为例，它主要靠随机数。

UUID 总长度是 128 bit，但 v4 中有固定的 version 和 variant 位，真正随机部分大约是 122 bit。RFC 9562 对 UUID v4 的定义就是基于真随机或伪随机数生成，并设置版本位和变体位。([RFC 编辑器](https://www.rfc-editor.org/rfc/rfc9562.html "RFC 9562: Universally Unique IDentifiers (UUIDs)"))

122 bit 空间有多大？

```text
2^122 ≈ 5.3 × 10^36
```

这个空间极大，所以在正常工程规模下冲突概率可以忽略。

但关键点是：

> UUID 是概率唯一，不是绝对唯一。

生产系统中只要它落库成为唯一业务标识，就应该有唯一约束兜底。

---

## 2.2 UUID v1、v3、v4、v5、v7 的区别

|版本|生成方式|是否随机|是否有序|Java 标准库支持|适合场景|主要问题|
|---|--:|--:|--:|--:|---|---|
|v1|时间戳 + 节点信息 + 时钟序列|部分随机|大体有序，但字符串排序不友好|可解析，不直接生成|老系统、需要时间相关 ID|可能暴露机器信息；隐私风险|
|v3|namespace + name + MD5|否|否|`UUID.nameUUIDFromBytes()`|同样输入生成同样 ID|MD5 已不适合安全场景|
|v4|随机数|是|否|`UUID.randomUUID()`|Java 最常用；文件名、traceId、幂等键、外部 ID|数据库聚簇主键性能差|
|v5|namespace + name + SHA-1|否|否|标准库没有直接 v5 工厂方法|稳定映射，如外部编码转 UUID|SHA-1 安全性不适合安全用途|
|v7|Unix 毫秒时间戳 + 随机数|部分随机|是，适合按生成时间排序|JDK 25 仍未作为常规稳定 API直接暴露；JDK 26 已规划/源码出现 `ofEpochMillis`|数据库主键、事件 ID、分布式有序 ID|标准库生态仍在过渡|

RFC 9562 明确定义了 UUID v7：前 48 bit 是 Unix Epoch 毫秒时间戳，剩余 74 bit 在排除 version 和 variant 后由随机位填充，因此它具备时间有序特征。([RFC 编辑器](https://www.rfc-editor.org/rfc/rfc9562.html "RFC 9562: Universally Unique IDentifiers (UUIDs)")) OpenJDK 相关议题也说明，JDK 当前长期以来主要暴露 v3 和 v4 工厂方法，v7 支持是新增方向。([OpenJDK Bug Tracker](https://bugs.openjdk.org/browse/JDK-8357251?utm_source=chatgpt.com "[JDK-8357251] Add Support for UUID Version 7 (UUIDv7) ..."))

---

## 2.3 Java `UUID.randomUUID()` 属于哪一种

Java：

```java
UUID uuid = UUID.randomUUID();
```

生成的是：

```text
UUID v4
```

Oracle 文档明确说明：

```text
randomUUID() -> type 4 pseudo randomly generated UUID
```

并且使用 cryptographically strong pseudo random number generator。([Oracle Docs](https://docs.oracle.com/javase/8/docs/api/java/util/UUID.html "UUID (Java Platform SE 8 )"))

所以你在项目中说：

> Java 标准库的 `UUID.randomUUID()` 生成的是 v4 随机 UUID，不是时间有序 UUID，也不是雪花 ID。

这是面试中的高频点。

---

## 2.4 UUID 是否绝对唯一

不是。

更准确的表达：

> UUID 的设计目标是让冲突概率极低，工程上可以认为唯一，但不是数学上绝对不可能重复。

原因：

- v4 靠随机数，随机空间很大，但仍是概率模型；
    
- 生成器实现可能有 bug；
    
- 随机源可能质量不足；
    
- 系统可能错误复用 ID；
    
- 数据迁移、测试数据、手工导入可能制造冲突。
    

生产做法：

```sql
UNIQUE KEY uk_xxx_id (xxx_id)
```

也就是说：

> UUID 负责降低冲突概率，数据库唯一约束负责最终兜底。

---

## 2.5 UUID、自增 ID、雪花 ID 的区别

|维度|UUID v4|UUID v7|数据库自增 ID|雪花算法 ID|
|---|--:|--:|--:|--:|
|是否中心化|否|否|是，依赖 DB|否，但依赖机器号分配|
|是否有序|否|是|是|大体有序|
|长度|128 bit|128 bit|通常 64 bit|通常 64 bit|
|可读性|差|差|好|一般|
|MySQL 聚簇索引友好|差|较好|很好|较好|
|暴露业务规模|不明显|不明显|明显|部分明显|
|生成成本|低|低|依赖 DB|低|
|跨服务生成|方便|方便|不方便|方便|
|典型用途|traceId、文件名、幂等键、外部 ID|数据库 ID、事件 ID|单体系统主键|分布式业务主键|

工程建议：

- **内部数据库大表主键**：优先自增 ID / 雪花 ID / UUID v7；
    
- **外部暴露 ID**：UUID v4 / UUID v7；
    
- **日志 traceId**：UUID v4 或更短的 traceId；
    
- **幂等键**：UUID v4 可以，但必须有唯一约束；
    
- **安全 token**：不要直接用 UUID，使用 `SecureRandom` 生成足够熵的 token。
    

---

# 3. Java 中如何使用 UUID

## 3.1 基础用法

```java
import java.util.UUID;

public class UuidBasicDemo {

    public static void main(String[] args) {
        // 标准 UUID，36 位，带中划线
        String uuid = UUID.randomUUID().toString();
        System.out.println(uuid);

        // 32 位，无中划线，常用于 traceId、文件名
        String compactUuid = uuid.replace("-", "");
        System.out.println(compactUuid);

        // String -> UUID
        UUID parsed = UUID.fromString(uuid);

        // UUID -> String
        String backToString = parsed.toString();
    }
}
```

注意：

```java
UUID.fromString("invalid")
```

会抛出：

```java
IllegalArgumentException
```

不要在 controller 里不处理，直接让异常穿透成 500。

---

## 3.2 生产可用 UUID 工具类

建议项目中统一封装，不要到处散落：

```java
UUID.randomUUID().toString().replace("-", "")
```

生产工具类：

```java
package com.example.common.id;

import java.nio.ByteBuffer;
import java.util.Locale;
import java.util.Objects;
import java.util.Optional;
import java.util.UUID;
import java.util.regex.Pattern;

public final class UuidUtils {

    /**
     * 标准 UUID 格式：
     * 550e8400-e29b-41d4-a716-446655440000
     */
    private static final Pattern STANDARD_UUID_PATTERN =
            Pattern.compile("^[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}$");

    /**
     * 32 位无中划线格式：
     * 550e8400e29b41d4a716446655440000
     */
    private static final Pattern COMPACT_UUID_PATTERN =
            Pattern.compile("^[0-9a-fA-F]{32}$");

    private UuidUtils() {
        throw new UnsupportedOperationException("Utility class");
    }

    /**
     * 生成标准 UUID，36 位，带中划线，小写。
     */
    public static String standardUuid() {
        return UUID.randomUUID().toString();
    }

    /**
     * 生成紧凑 UUID，32 位，无中划线，小写。
     * 适合 traceId、文件名、幂等键等场景。
     */
    public static String compactUuid() {
        return UUID.randomUUID().toString().replace("-", "");
    }

    /**
     * 校验标准 UUID。
     */
    public static boolean isStandardUuid(String value) {
        return value != null && STANDARD_UUID_PATTERN.matcher(value).matches();
    }

    /**
     * 校验 32 位紧凑 UUID。
     */
    public static boolean isCompactUuid(String value) {
        return value != null && COMPACT_UUID_PATTERN.matcher(value).matches();
    }

    /**
     * 接受标准格式或紧凑格式，统一转为 Java UUID 对象。
     */
    public static Optional<UUID> parseUuid(String value) {
        if (value == null || value.isBlank()) {
            return Optional.empty();
        }

        String normalized = value.trim();

        try {
            if (isStandardUuid(normalized)) {
                return Optional.of(UUID.fromString(normalized.toLowerCase(Locale.ROOT)));
            }

            if (isCompactUuid(normalized)) {
                String standard = toStandardFormat(normalized);
                return Optional.of(UUID.fromString(standard));
            }

            return Optional.empty();
        } catch (IllegalArgumentException ex) {
            return Optional.empty();
        }
    }

    /**
     * 将 32 位 UUID 转为标准 36 位 UUID。
     */
    public static String toStandardFormat(String compactUuid) {
        Objects.requireNonNull(compactUuid, "compactUuid must not be null");

        String value = compactUuid.trim().toLowerCase(Locale.ROOT);
        if (!isCompactUuid(value)) {
            throw new IllegalArgumentException("Invalid compact UUID: " + compactUuid);
        }

        return value.substring(0, 8) + "-"
                + value.substring(8, 12) + "-"
                + value.substring(12, 16) + "-"
                + value.substring(16, 20) + "-"
                + value.substring(20);
    }

    /**
     * 标准格式或紧凑格式统一转为 32 位小写格式。
     */
    public static String toCompactFormat(String uuid) {
        UUID parsed = parseUuid(uuid)
                .orElseThrow(() -> new IllegalArgumentException("Invalid UUID: " + uuid));

        return parsed.toString().replace("-", "");
    }

    /**
     * UUID 转 BINARY(16) 对应的 byte[]。
     */
    public static byte[] toBytes(UUID uuid) {
        Objects.requireNonNull(uuid, "uuid must not be null");

        ByteBuffer buffer = ByteBuffer.allocate(16);
        buffer.putLong(uuid.getMostSignificantBits());
        buffer.putLong(uuid.getLeastSignificantBits());
        return buffer.array();
    }

    /**
     * BINARY(16) byte[] 转 UUID。
     */
    public static UUID fromBytes(byte[] bytes) {
        Objects.requireNonNull(bytes, "bytes must not be null");
        if (bytes.length != 16) {
            throw new IllegalArgumentException("UUID bytes length must be 16");
        }

        ByteBuffer buffer = ByteBuffer.wrap(bytes);
        long mostSigBits = buffer.getLong();
        long leastSigBits = buffer.getLong();

        return new UUID(mostSigBits, leastSigBits);
    }
}
```

---

## 3.3 作为业务唯一标识

示例：创建用户时生成外部可见 `userNo`。

```java
public record RegisterUserCommand(
        String username,
        String password
) {
}
```

```java
public class User {

    private Long id;          // 数据库内部主键
    private String userNo;    // 外部业务唯一标识
    private String username;

    public static User register(RegisterUserCommand command) {
        User user = new User();
        user.userNo = UuidUtils.compactUuid();
        user.username = command.username();
        return user;
    }
}
```

推荐设计：

```text
id      BIGINT       内部主键
user_no CHAR(32)     外部业务 ID，唯一索引
```

不要把所有地方都只靠 UUID 主键。高并发 MySQL 大表中，`BIGINT` 自增主键 + UUID 业务唯一键通常更稳。

---

## 3.4 作为请求追踪 traceId

```java
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.slf4j.MDC;

import java.io.IOException;

public class TraceIdFilter implements Filter {

    private static final String TRACE_ID_HEADER = "X-Trace-Id";
    private static final String MDC_TRACE_ID = "traceId";

    @Override
    public void doFilter(ServletRequest request,
                         ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;

        String traceId = httpRequest.getHeader(TRACE_ID_HEADER);
        if (!UuidUtils.isCompactUuid(traceId)) {
            traceId = UuidUtils.compactUuid();
        }

        try {
            MDC.put(MDC_TRACE_ID, traceId);
            chain.doFilter(request, response);
        } finally {
            // 防止线程复用导致 MDC 污染
            MDC.remove(MDC_TRACE_ID);
        }
    }
}
```

日志格式：

```properties
logging.pattern.level=%5p [%X{traceId}]
```

建议：

```text
traceId 统一 32 位小写无中划线
```

不要一部分服务用 36 位，一部分用 32 位，一部分大写。

---

## 3.5 作为幂等键 idempotencyKey

核心原则：

> UUID 只是幂等键的值，不是幂等机制本身。真正防重靠唯一约束、Redis 原子操作或事务控制。

```java
public record CreateOrderRequest(
        Long userId,
        Long productId,
        Integer quantity,
        String idempotencyKey
) {
}
```

```java
@Service
public class OrderAppService {

    private final OrderRepository orderRepository;
    private final IdempotencyRepository idempotencyRepository;

    public OrderAppService(OrderRepository orderRepository,
                           IdempotencyRepository idempotencyRepository) {
        this.orderRepository = orderRepository;
        this.idempotencyRepository = idempotencyRepository;
    }

    @Transactional
    public Long createOrder(CreateOrderRequest request) {
        String key = normalizeIdempotencyKey(request.idempotencyKey());

        boolean firstRequest = idempotencyRepository.trySaveKey(key, "CREATE_ORDER");
        if (!firstRequest) {
            return idempotencyRepository.findResultId(key)
                    .orElseThrow(() -> new IllegalStateException("Duplicate request is still processing"));
        }

        Long orderId = orderRepository.createOrder(
                request.userId(),
                request.productId(),
                request.quantity()
        );

        idempotencyRepository.saveResult(key, orderId);
        return orderId;
    }

    private String normalizeIdempotencyKey(String key) {
        if (!UuidUtils.isCompactUuid(key)) {
            throw new IllegalArgumentException("Invalid idempotencyKey");
        }
        return key.toLowerCase();
    }
}
```

数据库兜底：

```sql
CREATE TABLE idempotency_record (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    idempotency_key CHAR(32) NOT NULL,
    biz_type VARCHAR(64) NOT NULL,
    result_id BIGINT NULL,
    status VARCHAR(32) NOT NULL,
    created_at DATETIME(3) NOT NULL,
    updated_at DATETIME(3) NOT NULL,
    UNIQUE KEY uk_idem_key_biz (idempotency_key, biz_type)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## 3.6 作为数据库主键时的写法

不推荐高并发大表直接这样：

```sql
id CHAR(36) PRIMARY KEY
```

更推荐：

```sql
id BINARY(16) PRIMARY KEY
```

Java Entity 可以这样处理：

```java
public class FileObject {

    /**
     * 对应 MySQL BINARY(16)
     */
    private byte[] id;

    private String originalName;

    public static FileObject create(String originalName) {
        FileObject file = new FileObject();
        file.id = UuidUtils.toBytes(UUID.randomUUID());
        file.originalName = originalName;
        return file;
    }

    public String idAsString() {
        return UuidUtils.fromBytes(id).toString();
    }
}
```

但再次强调：

> 如果是 MySQL InnoDB 高并发写入大表，不建议随机 UUID v4 直接做聚簇主键。优先 `BIGINT` 自增、雪花 ID 或 UUID v7。

---

# 4. 具体业务案例

# 案例 1：用户注册生成 userId / userNo

## 业务背景

用户注册后，需要一个对外展示或 API 使用的用户唯一标识。

不建议直接暴露数据库自增 ID：

```text
/user/10001
/user/10002
```

更推荐：

```text
/user/0f8fad5bd9cb469fa16570867728950e
```

## 是否适合用 UUID

适合，但建议作为**业务唯一标识**，不一定作为数据库主键。

推荐表结构：

```sql
CREATE TABLE user_account (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_no CHAR(32) NOT NULL,                    ## uuid 32位 去掉了中划线
    username VARCHAR(64) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at DATETIME(3) NOT NULL,
    updated_at DATETIME(3) NOT NULL,
    UNIQUE KEY uk_user_no (user_no),
    UNIQUE KEY uk_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 生产级代码

```java
@Service
public class UserRegisterService {

    private final UserAccountMapper userAccountMapper;
    private final PasswordEncoder passwordEncoder;

    public UserRegisterService(UserAccountMapper userAccountMapper,
                               PasswordEncoder passwordEncoder) {
        this.userAccountMapper = userAccountMapper;
        this.passwordEncoder = passwordEncoder;
    }

    @Transactional
    public String register(RegisterUserCommand command) {
        if (command.username() == null || command.username().isBlank()) {
            throw new IllegalArgumentException("username must not be blank");
        }
		//生成UUID	
        String userNo = UuidUtils.compactUuid();

        UserAccountPO po = new UserAccountPO();
        po.setUserNo(userNo);
        po.setUsername(command.username());
        po.setPasswordHash(passwordEncoder.encode(command.password()));
        po.setCreatedAt(LocalDateTime.now());
        po.setUpdatedAt(LocalDateTime.now());

        try {
            userAccountMapper.insert(po);
        } catch (DuplicateKeyException ex) {
            // 极小概率 UUID 冲突，更多时候是 username 冲突
            throw new IllegalStateException("User already exists or userNo conflict", ex);
        }

        return userNo;
    }
}
```

## 坑点

|坑点|后果|正确做法|
|---|---|---|
|只生成 UUID，不加唯一索引|极端情况下脏数据|`UNIQUE KEY uk_user_no`|
|把 UUID 当权限凭证|被拿到 ID 就能访问资源|必须做鉴权|
|直接用 `CHAR(36)`|索引更大|外部 ID 可用 `CHAR(32)`，主键更推荐 `BINARY(16)` 或 BIGINT|

---

# 案例 2：订单创建接口生成幂等键

## 业务背景

下单接口经常遇到：

- 用户重复点击；
    
- 前端重试；
    
- 网关超时重试；
    
- App 网络抖动；
    
- MQ 重复投递。
    

如果没有幂等控制，可能重复创建订单。

## 是否适合用 UUID

适合。客户端或服务端可以生成 UUID 作为幂等键。

但核心不是 UUID，而是：

```text
idempotencyKey + 唯一约束 + 状态机
```

## 表结构

```sql
CREATE TABLE order_request_idempotency (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    idempotency_key CHAR(32) NOT NULL,
    user_id BIGINT NOT NULL,
    order_id BIGINT NULL,
    status VARCHAR(32) NOT NULL COMMENT 'PROCESSING/SUCCESS/FAILED',
    created_at DATETIME(3) NOT NULL,
    updated_at DATETIME(3) NOT NULL,
    UNIQUE KEY uk_user_idem_key (user_id, idempotency_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 生产级代码

```java
@Service
public class OrderCommandService {

    private final OrderMapper orderMapper;
    private final OrderIdempotencyMapper idempotencyMapper;

    public OrderCommandService(OrderMapper orderMapper,
                               OrderIdempotencyMapper idempotencyMapper) {
        this.orderMapper = orderMapper;
        this.idempotencyMapper = idempotencyMapper;
    }

    @Transactional
    public Long createOrder(CreateOrderRequest request) {
        validateRequest(request);

        String key = request.idempotencyKey().toLowerCase();

        boolean locked = tryCreateIdempotencyRecord(request.userId(), key);
        if (!locked) {
            return queryExistingOrderId(request.userId(), key);
        }

        Long orderId = orderMapper.insertOrder(
                request.userId(),
                request.productId(),
                request.quantity()
        );

        idempotencyMapper.markSuccess(request.userId(), key, orderId);

        return orderId;
    }

    private void validateRequest(CreateOrderRequest request) {
        if (request.userId() == null) {
            throw new IllegalArgumentException("userId must not be null");
        }
        if (!UuidUtils.isCompactUuid(request.idempotencyKey())) {
            throw new IllegalArgumentException("invalid idempotencyKey");
        }
    }

    private boolean tryCreateIdempotencyRecord(Long userId, String key) {
        try {
            return idempotencyMapper.insertProcessing(userId, key) == 1;
        } catch (DuplicateKeyException ex) {
            return false;
        }
    }

    private Long queryExistingOrderId(Long userId, String key) {
        IdempotencyRecord record = idempotencyMapper.findByUserIdAndKey(userId, key)
                .orElseThrow(() -> new IllegalStateException("Idempotency record not found"));

        if ("SUCCESS".equals(record.status())) {
            return record.orderId();
        }

        throw new IllegalStateException("Request is processing, please retry later");
    }
}
```

## 坑点

|坑点|后果|正确做法|
|---|---|---|
|每次服务端都重新生成 idempotencyKey|根本无法防重|同一次业务重试必须使用同一个 key|
|只靠 Redis 不落库|Redis 丢失后可能重复|核心交易建议数据库兜底|
|幂等键不带用户维度|不同用户可能互相影响|唯一键使用 `(user_id, idempotency_key)`|

---

# 案例 3：分布式调用生成 traceId

## 业务背景

一次请求经过：

```text
Gateway -> User Service -> Order Service -> Payment Service -> MQ -> Consumer
```

如果没有统一 traceId，排查问题很困难。

## 是否适合用 UUID

适合。

traceId 的核心要求：

- 足够唯一；
    
- 生成快；
    
- 全链路传递；
    
- 日志格式统一；
    
- 不用于排序；
    
- 不当安全 token。
    

## Spring Boot Filter

```java
@Component
public class TraceIdFilter implements Filter {

    public static final String TRACE_ID_HEADER = "X-Trace-Id";
    public static final String MDC_KEY = "traceId";

    @Override
    public void doFilter(ServletRequest request,
                         ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {

        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        String incomingTraceId = httpRequest.getHeader(TRACE_ID_HEADER);
        String traceId = UuidUtils.isCompactUuid(incomingTraceId)
                ? incomingTraceId.toLowerCase()
                : UuidUtils.compactUuid();

        try {
            MDC.put(MDC_KEY, traceId);
            httpResponse.setHeader(TRACE_ID_HEADER, traceId);
            chain.doFilter(request, response);
        } finally {
            MDC.remove(MDC_KEY);
        }
    }
}
```

## Feign 拦截器传递 traceId

```java
@Configuration
public class FeignTraceConfig {

    @Bean
    public RequestInterceptor traceIdRequestInterceptor() {
        return template -> {
            String traceId = MDC.get(TraceIdFilter.MDC_KEY);
            if (traceId != null && !traceId.isBlank()) {
                template.header(TraceIdFilter.TRACE_ID_HEADER, traceId);
            }
        };
    }
}
```

## 坑点

|坑点|后果|正确做法|
|---|---|---|
|线程池中 MDC 不传递|异步日志丢 traceId|使用 TTL MDC 或手动包装任务|
|格式不统一|日志检索困难|全公司统一 32 位小写|
|traceId 进入业务表做排序|没意义|traceId 只做排查|

---

# 案例 4：文件上传生成唯一文件名

## 业务背景

用户上传文件：

```text
avatar.png
avatar.png
avatar.png
```

如果直接用原文件名存储，会覆盖。

## 是否适合用 UUID

非常适合。

推荐文件名：

```text
2026/05/13/4f23a1c5c0b94d36a0b52b3e4cc5f1e8.png
```

## 生产级代码

```java
@Service
public class FileNameService {

    private static final Set<String> ALLOWED_EXTENSIONS =
            Set.of("jpg", "jpeg", "png", "pdf", "xlsx", "docx");

    public String generateObjectKey(String originalFilename) {
        String extension = extractExtension(originalFilename);

        if (!ALLOWED_EXTENSIONS.contains(extension)) {
            throw new IllegalArgumentException("Unsupported file extension: " + extension);
        }

        LocalDate today = LocalDate.now();

        return "%d/%02d/%02d/%s.%s".formatted(
                today.getYear(),
                today.getMonthValue(),
                today.getDayOfMonth(),
                UuidUtils.compactUuid(),
                extension
        );
    }

    private String extractExtension(String originalFilename) {
        if (originalFilename == null || originalFilename.isBlank()) {
            throw new IllegalArgumentException("originalFilename must not be blank");
        }

        int dotIndex = originalFilename.lastIndexOf('.');
        if (dotIndex < 0 || dotIndex == originalFilename.length() - 1) {
            throw new IllegalArgumentException("file extension is required");
        }

        return originalFilename.substring(dotIndex + 1).toLowerCase(Locale.ROOT);
    }
}
```

## 数据库存储建议

```sql
CREATE TABLE file_object (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    file_no CHAR(32) NOT NULL,
    bucket VARCHAR(128) NOT NULL,
    object_key VARCHAR(512) NOT NULL,
    original_name VARCHAR(255) NOT NULL,
    content_type VARCHAR(128) NOT NULL,
    size_bytes BIGINT NOT NULL,
    created_at DATETIME(3) NOT NULL,
    UNIQUE KEY uk_file_no (file_no),
    UNIQUE KEY uk_object_key (object_key)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

## 坑点

|坑点|后果|正确做法|
|---|---|---|
|信任原始文件名|覆盖、路径穿越风险|自己生成 objectKey|
|只看扩展名|伪造文件类型|结合 MIME、魔数校验|
|UUID 文件名不分目录|对象存储/文件系统管理困难|按日期、业务类型分片|

---

# 5. UUID 在数据库中的使用

## 5.1 UUID 作为主键的优缺点

### 优点

|优点|说明|
|---|---|
|应用侧生成|不依赖数据库|
|分布式友好|多服务、多节点可独立生成|
|不暴露规模|不像自增 ID 容易被猜|
|数据合并方便|多库多表合并冲突概率低|
|适合外部引用|URL、API、回调场景方便|

### 缺点

|缺点|说明|
|---|---|
|占用空间大|文本 UUID 比 BIGINT 大很多|
|随机写入差|v4 对 B+Tree 不友好|
|可读性差|排查问题不如数字 ID|
|不适合排序|v4 没有时间顺序|
|二级索引膨胀|InnoDB 二级索引会携带主键值|

---

## 5.2 `CHAR(36)`、`CHAR(32)`、`BINARY(16)` 对比

|存储方式|示例|空间|可读性|索引效率|推荐程度|
|---|---|--:|--:|--:|--:|
|`CHAR(36)`|`550e8400-e29b-41d4-a716-446655440000`|36 字符|好|差|一般不推荐做主键|
|`CHAR(32)`|`550e8400e29b41d4a716446655440000`|32 字符|一般|一般|可用于业务唯一键|
|`BINARY(16)`|16 字节二进制|16 字节|差|最好|推荐用于 UUID 主键/索引|

RFC 9562 也明确指出，数据库中把 UUID 存成文本会很冗长；可行时应该以底层 128 bit 二进制形式存储。([RFC 编辑器](https://www.rfc-editor.org/rfc/rfc9562.html "RFC 9562: Universally Unique IDentifiers (UUIDs)"))

---

## 5.3 UUID 对索引、页分裂、插入性能的影响

以 MySQL InnoDB 为例：

- InnoDB 表数据按聚簇索引组织；
    
- 主键就是聚簇索引；
    
- 叶子节点存整行数据；
    
- 如果主键递增，插入大多发生在 B+Tree 右侧；
    
- 如果主键随机，插入会分散到各个页；
    
- 页满时触发页分裂；
    
- 页分裂会带来更多 IO、锁竞争、页碎片和缓存命中下降。
    

随机 UUID v4 的问题：

```text
新数据不是追加写，而是随机插入 B+Tree 的不同位置。
```

所以高并发写入大表中：

```sql
id CHAR(36) PRIMARY KEY
```

通常是糟糕设计。

---

## 5.4 为什么随机 UUID 不适合高并发写入表的聚簇主键

原因不是“UUID 长”，而是两个问题叠加：

### 问题 1：长

`BIGINT` 是 8 字节。  
`BINARY(16)` 是 16 字节。  
`CHAR(36)` 更大。

InnoDB 二级索引叶子节点会保存主键值，主键越大，二级索引越大。

### 问题 2：随机

随机写入导致 B+Tree 页分裂和缓存局部性差。

所以真正的问题是：

```text
随机 + 大字段 + 聚簇索引
```

---

## 5.5 什么时候可以用 UUID 做主键

可以用的场景：

|场景|是否可以|
|---|---|
|小表、配置表|可以|
|写入量低的业务表|可以|
|多系统数据合并|可以|
|离线同步数据|可以|
|PostgreSQL 使用原生 UUID 类型|可以|
|MySQL 中使用 UUID v7 + BINARY(16)|可以考虑|
|高并发交易订单表使用 UUID v4 聚簇主键|不推荐|

---

## 5.6 什么时候更推荐雪花、自增 ID、UUID v7

|场景|推荐方案|
|---|---|
|单体系统、单库单表|数据库自增 ID|
|分布式订单、支付、库存流水|雪花算法 ID|
|希望 ID 全局唯一且趋势有序|UUID v7|
|外部暴露资源 ID|UUID v4 / v7|
|日志 traceId|UUID v4 / OpenTelemetry traceId|
|文件对象名|UUID v4|
|多库数据同步合并|UUID v4 / v7|

---

## 5.7 Java + MySQL 推荐方案

### 方案 A：最稳妥的业务系统方案

适合绝大多数 Java 后端系统：

```sql
CREATE TABLE user_account (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_no CHAR(32) NOT NULL,
    username VARCHAR(64) NOT NULL,
    created_at DATETIME(3) NOT NULL,
    updated_at DATETIME(3) NOT NULL,
    UNIQUE KEY uk_user_no (user_no),
    UNIQUE KEY uk_username (username)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

特点：

```text
BIGINT 自增主键负责数据库性能
UUID user_no 负责外部唯一标识
```

这是最推荐的组合。

---

### 方案 B：UUID 必须做主键时

```sql
CREATE TABLE file_object (
    id BINARY(16) PRIMARY KEY,
    file_no CHAR(32) GENERATED ALWAYS AS (LOWER(HEX(id))) STORED,
    object_key VARCHAR(512) NOT NULL,
    original_name VARCHAR(255) NOT NULL,
    created_at DATETIME(3) NOT NULL,
    UNIQUE KEY uk_object_key (object_key),
    UNIQUE KEY uk_file_no (file_no)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Java 插入：

```java
UUID id = UUID.randomUUID();
byte[] idBytes = UuidUtils.toBytes(id);
```

MyBatis TypeHandler：

```java
@MappedTypes(UUID.class)
@MappedJdbcTypes(JdbcType.BINARY)
public class UuidBinaryTypeHandler extends BaseTypeHandler<UUID> {

    @Override
    public void setNonNullParameter(PreparedStatement ps,
                                    int i,
                                    UUID parameter,
                                    JdbcType jdbcType) throws SQLException {
        ps.setBytes(i, UuidUtils.toBytes(parameter));
    }

    @Override
    public UUID getNullableResult(ResultSet rs, String columnName) throws SQLException {
        byte[] bytes = rs.getBytes(columnName);
        return bytes == null ? null : UuidUtils.fromBytes(bytes);
    }

    @Override
    public UUID getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        byte[] bytes = rs.getBytes(columnIndex);
        return bytes == null ? null : UuidUtils.fromBytes(bytes);
    }

    @Override
    public UUID getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        byte[] bytes = cs.getBytes(columnIndex);
        return bytes == null ? null : UuidUtils.fromBytes(bytes);
    }
}
```

---

### 方案 C：MySQL 函数转换

MySQL 提供 `UUID_TO_BIN()` 和 `BIN_TO_UUID()`。官方文档说明，`UUID_TO_BIN()` 可以把字符串 UUID 转为二进制 UUID；带 `swap_flag=1` 时会交换时间相关部分，以改善 v1 UUID 的索引效率，但该优化假设 UUID 是 v1，对其他格式没有收益。([MySQL开发者区](https://dev.mysql.com/doc/en/miscellaneous-functions.html "MySQL :: MySQL 9.7 Reference Manual :: 14.24 Miscellaneous Functions"))

```sql
CREATE TABLE event_record (
    id BINARY(16) PRIMARY KEY,
    event_type VARCHAR(64) NOT NULL,
    payload JSON NOT NULL,
    created_at DATETIME(3) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

插入：

```sql
INSERT INTO event_record (id, event_type, payload, created_at)
VALUES (UUID_TO_BIN(UUID()), 'USER_REGISTERED', JSON_OBJECT('userId', 1001), NOW(3));
```

查询：

```sql
SELECT BIN_TO_UUID(id) AS id, event_type, payload, created_at
FROM event_record;
```

注意：

```sql
UUID_TO_BIN(uuid, 1)
```

主要针对 v1 时间型 UUID。不要盲目对 Java `UUID.randomUUID()` 生成的 v4 使用 `swap_flag=1`，MySQL 文档明确说明非 v1 不受益。([MySQL开发者区](https://dev.mysql.com/doc/en/miscellaneous-functions.html "MySQL :: MySQL 9.7 Reference Manual :: 14.24 Miscellaneous Functions"))

---

# 6. 生产级最佳实践

## 6.1 是否保留中划线

明确建议：

|场景|建议|
|---|---|
|API 对外展示|标准 UUID 可保留中划线|
|traceId|不保留，使用 32 位小写|
|文件名|不保留|
|数据库业务键 `CHAR`|不保留，使用 `CHAR(32)`|
|数据库主键|不存字符串，使用 `BINARY(16)`|
|配置、人工排查|可以保留中划线|

工程规范建议：

```text
内部系统传递：32 位小写无中划线
外部标准 API：36 位标准格式也可以
数据库主键：BINARY(16)
```

---

## 6.2 是否统一小写

明确建议：

```text
统一小写。
```

原因：

- Java `UUID.toString()` 默认是小写；
    
- 日志检索更统一；
    
- 数据库比较更稳定；
    
- 避免同一个 UUID 出现多种表现形式。
    

---

## 6.3 是否暴露给前端

明确建议：

```text
可以暴露，但不能把 UUID 当权限。
```

适合暴露：

- `userNo`
    
- `fileNo`
    
- `orderNo`
    
- `taskId`
    
- `requestId`
    

不适合：

- 仅凭 UUID 下载私有文件；
    
- 仅凭 UUID 查看订单；
    
- 仅凭 UUID 访问后台资源。
    

正确做法：

```text
UUID 标识资源，权限系统决定能不能访问资源。
```

---

## 6.4 是否适合做排序字段

明确建议：

```text
UUID v4 不适合排序。
UUID v7 可以按生成时间趋势排序。
```

如果你需要排序：

```sql
ORDER BY created_at DESC
```

不要：

```sql
ORDER BY uuid DESC
```

除非你明确使用 UUID v7，并且存储顺序与排序语义一致。

---

## 6.5 是否适合做日志 traceId

明确建议：

```text
适合。
```

但要规范：

```text
32 位小写无中划线，全链路统一 header 名。
```

例如：

```text
X-Trace-Id: 82e69f44d8a34c9f9f0974bc65e7b3e1
```

---

## 6.6 是否适合作为安全 token

明确建议：

```text
不要直接把 UUID 当安全 token。
```

RFC 9562 安全章节明确提醒：实现者不应该假设 UUID 难以猜测，也不能把 UUID 作为“持有即授权”的安全能力。([RFC 编辑器](https://www.rfc-editor.org/rfc/rfc9562.html "RFC 9562: Universally Unique IDentifiers (UUIDs)"))

更推荐：

```java
public final class TokenUtils {

    private static final SecureRandom SECURE_RANDOM = new SecureRandom();

    private TokenUtils() {
    }

    public static String secureToken(int byteLength) {
        if (byteLength < 32) {
            throw new IllegalArgumentException("token byteLength should be at least 32");
        }

        byte[] bytes = new byte[byteLength];
        SECURE_RANDOM.nextBytes(bytes);

        return Base64.getUrlEncoder()
                .withoutPadding()
                .encodeToString(bytes);
    }
}
```

使用：

```java
String resetPasswordToken = TokenUtils.secureToken(32);
```

---

## 6.7 是否需要加密或脱敏

明确建议：

|场景|建议|
|---|---|
|普通业务 UUID|不需要加密|
|日志 traceId|不需要脱敏|
|用户外部 ID|通常不需要|
|下载链接 token|不要用 UUID，使用安全随机 token|
|内部数据库主键|不应直接暴露|
|敏感业务资源|UUID + 权限校验，而不是 UUID 加密|

UUID 本身一般不是敏感数据，但它可能能定位某个资源，所以必须结合权限控制。

---

## 6.8 如何封装工具类

建议项目中统一提供：

```text
UuidUtils.standardUuid()
UuidUtils.compactUuid()
UuidUtils.isStandardUuid()
UuidUtils.isCompactUuid()
UuidUtils.parseUuid()
UuidUtils.toBytes()
UuidUtils.fromBytes()
```

禁止业务代码里到处写：

```java
UUID.randomUUID().toString().replace("-", "")
```

因为后续如果规范调整、格式统一、traceId 改造，会很难收敛。

---

# 7. 常见坑点

## 坑 1：把 UUID 当成有序 ID

错误做法：

```sql
ORDER BY id DESC
```

如果 `id` 是 UUID v4，这个排序没有业务意义。

后果：

- 结果不是按创建时间排序；
    
- 分页可能混乱；
    
- 面试中暴露对 UUID 理解不清。
    

正确做法：

```sql
ORDER BY created_at DESC
```

或者使用 UUID v7 / 雪花 ID。

---

## 坑 2：用随机 UUID 做 MySQL 聚簇主键

错误做法：

```sql
CREATE TABLE orders (
    id CHAR(36) PRIMARY KEY,
    ...
);
```

后果：

- 主键索引大；
    
- 二级索引膨胀；
    
- 插入随机分布；
    
- 页分裂增加；
    
- 高并发写入性能下降。
    

正确做法：

```sql
id BIGINT PRIMARY KEY AUTO_INCREMENT,
order_no CHAR(32) NOT NULL UNIQUE
```

或者：

```sql
id BIGINT PRIMARY KEY -- 雪花 ID
```

或者在合适场景使用：

```sql
id BINARY(16) PRIMARY KEY -- UUID v7 更合适
```

---

## 坑 3：误以为 UUID 绝对不会冲突

错误认知：

```text
UUID 永远不会重复，所以不用唯一索引。
```

后果：

- 极端情况下数据重复；
    
- 手工导入、测试数据、迁移时可能冲突；
    
- 业务唯一性没有数据库保障。
    

正确做法：

```sql
UNIQUE KEY uk_biz_id (biz_id)
```

---

## 坑 4：用 UUID 替代安全 token

错误做法：

```java
String resetToken = UUID.randomUUID().toString();
```

后果：

- token 熵和生成方式不一定满足安全设计；
    
- 容易产生“拿到 ID 就有权限”的错误模型；
    
- 不利于设置过期、绑定用户、绑定场景。
    

正确做法：

```java
String resetToken = TokenUtils.secureToken(32);
```

并存储：

```sql
token_hash
expires_at
used_at
user_id
scene
```

---

## 坑 5：日志链路中格式不统一

错误现象：

```text
服务 A：550e8400-e29b-41d4-a716-446655440000
服务 B：550E8400E29B41D4A716446655440000
服务 C：trace-550e8400
```

后果：

- 日志平台难查；
    
- 链路追踪断裂；
    
- 排障成本上升。
    

正确做法：

```text
统一 32 位小写无中划线
统一 header：X-Trace-Id
统一 MDC key：traceId
```

---

## 坑 6：`replace("-", "")` 滥用

错误做法：

```java
String id = input.replace("-", "");
```

问题是：  
你没有校验它是不是合法 UUID。

比如：

```text
abc---xyz
```

replace 后也能变成一个字符串。

正确做法：

```java
String compact = UuidUtils.toCompactFormat(input);
```

先解析，再转换。

---

## 坑 7：大小写不统一

错误现象：

```text
550E8400E29B41D4A716446655440000
550e8400e29b41d4a716446655440000
```

后果：

- 日志检索不一致；
    
- 某些大小写敏感系统中可能被当成不同值；
    
- 前后端校验规则不一致。
    

正确做法：

```java
uuid.toLowerCase(Locale.ROOT)
```

---

## 坑 8：UUID 和业务编码混用

错误做法：

```text
订单号：550e8400-e29b-41d4-a716-446655440000
```

业务上订单号通常需要：

- 可读；
    
- 可客服沟通；
    
- 可按日期分段；
    
- 可定位业务线；
    
- 可做风控分析。
    

UUID 不适合作为“展示型订单编号”。

正确做法：

```text
id:        BIGINT / UUID / 雪花 ID    技术主键
order_no: 202605130001234567         业务订单号
request_id: UUID                     请求幂等键
trace_id: UUID                       链路追踪 ID
```

不同 ID 不要混用。

---

# 8. 面试高频问题

## Q1：UUID 是什么？

标准回答：

> UUID 是通用唯一标识符，本质是一个 128 bit 的标识值，常见字符串格式是 36 位，形如 `8-4-4-4-12`。它的核心价值是应用程序可以在不依赖中心发号器的情况下生成近似全局唯一 ID。Java 中常用 `UUID.randomUUID()` 生成 UUID v4，适合 traceId、文件名、幂等键、外部资源 ID 等场景。

---

## Q2：Java 的 `UUID.randomUUID()` 是怎么生成的？

标准回答：

> Java 的 `UUID.randomUUID()` 生成的是 UUID v4，也就是随机 UUID。它使用强伪随机数生成器生成随机位，然后设置 UUID 的 version 和 variant 位。它不是时间有序 ID，也不是雪花 ID，所以不适合拿它做排序字段或高并发 MySQL 聚簇主键。

---

## Q3：UUID 会不会重复？

标准回答：

> 理论上会，工程上概率极低。比如 UUID v4 有约 122 bit 随机空间，正常业务规模下可以认为不会冲突。但生产系统不能只靠概率，凡是作为业务唯一标识落库，都应该加唯一索引兜底。

---

## Q4：UUID 能不能作为数据库主键？

标准回答：

> 能，但要看数据库和写入规模。小表、低频写入、跨系统合并数据可以用。MySQL InnoDB 高并发大表不建议用随机 UUID v4 做聚簇主键，因为它随机写入 B+Tree，会导致页分裂、索引膨胀和缓存局部性差。如果一定要用 UUID，建议用 `BINARY(16)` 存储，或者考虑 UUID v7。多数业务更推荐 `BIGINT` 自增主键或雪花 ID，再配一个 UUID 业务唯一键对外暴露。

---

## Q5：UUID 和雪花算法有什么区别？

标准回答：

> UUID 是 128 bit，常见 v4 靠随机数保证近似唯一，不依赖中心节点，但无序、较长。雪花算法通常是 64 bit，由时间戳、机器号、序列号组成，趋势递增，更适合数据库主键和分布式业务 ID。但雪花算法需要管理机器号，并处理时钟回拨问题。简单说，UUID 更偏通用唯一标识，雪花 ID 更偏高性能分布式主键。

---

## Q6：为什么 UUID 不适合作为 MySQL 聚簇索引主键？

标准回答：

> 主要是随机性和长度。InnoDB 的表按聚簇索引组织，主键随机意味着插入不是追加写，而是分散插入到 B+Tree 的不同位置，容易导致页分裂和页碎片。同时 UUID 比 BIGINT 大，二级索引也会携带主键值，导致索引体积变大。因此随机 UUID v4 不适合高并发写入大表的聚簇主键。

---

## Q7：UUID 适合做分布式 ID 吗？

标准回答：

> 适合某些分布式 ID 场景，比如 traceId、文件 ID、请求 ID、外部资源 ID，因为它可以本地生成，不需要中心服务。但如果这个 ID 要作为数据库主键，并且要求趋势递增、查询排序、高并发写入，那 UUID v4 不是最佳选择，更推荐雪花 ID 或 UUID v7。

---

## Q8：UUID 能不能做 token？

标准回答：

> 不建议直接把 UUID 当安全 token。UUID 是标识符，不是安全凭证。安全 token 应该使用 `SecureRandom` 生成足够长度的随机字节，再用 Base64URL 编码，并且服务端存储 hash、过期时间、使用状态和场景。UUID 可以作为 requestId，但不要作为“持有即授权”的 token。

---

## Q9：UUID v4 和 v7 有什么区别？

标准回答：

> UUID v4 主要由随机数生成，唯一性好，但无序。UUID v7 把 Unix 毫秒时间戳放在高位，再加随机位，因此具备时间有序特征，更适合数据库索引、事件 ID 和按生成时间排序的场景。简单说，v4 适合随机标识，v7 更适合需要趋势有序的分布式 ID。

---

## Q10：你在项目中怎么用 UUID？

标准回答：

> 我一般不会直接把 UUID v4 作为 MySQL 大表主键。常见做法是内部主键用 `BIGINT` 自增或雪花 ID，对外暴露的 `userNo`、`fileNo`、`requestId` 用 UUID。traceId 使用 32 位小写无中划线格式，放到 MDC 和请求 header 中。幂等键也可以用 UUID，但一定配合唯一索引或 Redis 原子操作。数据库如果必须存 UUID，会优先用 `BINARY(16)`，而不是 `CHAR(36)`。

---

# 9. 最终总结表

|使用场景|是否推荐 UUID|推荐 UUID 版本|Java 实现方式|数据库存储方式|替代方案|
|---|--:|---|---|---|---|
|日志 traceId|推荐|v4|`UUID.randomUUID().toString().replace("-", "")`，封装工具类|一般不入业务库；日志字段用字符串|OpenTelemetry traceId|
|文件唯一名|推荐|v4|`UuidUtils.compactUuid()`|`file_no CHAR(32)` 或 object_key 唯一|雪花 ID、对象存储 ETag|
|幂等键|推荐|v4|客户端或服务端生成 UUID|`CHAR(32)` + 唯一索引|业务请求号、Redis token|
|用户外部 ID|推荐|v4 / v7|注册时生成 `userNo`|`CHAR(32)` + 唯一索引|雪花 ID、业务编码|
|MySQL 大表主键|不推荐 v4|v7 可考虑|标准库 v7 支持仍需看 JDK 版本/第三方库|`BINARY(16)`|BIGINT 自增、雪花 ID|
|订单主键|不推荐 v4|v7 可考虑|不建议直接 v4|BIGINT / BIGINT 雪花更常见|雪花 ID|
|订单展示编号|不推荐|不建议|不用 UUID|`order_no VARCHAR`|日期 + 序列号|
|安全 token|不推荐|不建议|使用 `SecureRandom`|存 token hash|JWT、Opaque Token|
|多系统数据合并|推荐|v4 / v7|各系统独立生成|`BINARY(16)` 或 `CHAR(32)`|统一发号服务|
|事件 ID|推荐|v7 更好|v7 库 / 自定义生成器|`BINARY(16)`|雪花 ID、ULID|

---

# 记忆版总结

UUID 在 Java 后端中最常见的是 `UUID.randomUUID()`，它生成 UUID v4，优点是本地生成、分布式友好、冲突概率极低；缺点是随机、无序、较长，不适合直接作为 MySQL 高并发大表的聚簇主键。

生产实践中建议：

```text
1. traceId、fileNo、idempotencyKey：可以用 UUID v4，统一 32 位小写无中划线。
2. MySQL 大表主键：不要用随机 UUID v4，优先 BIGINT 自增、雪花 ID 或 UUID v7。
3. UUID 落库要有唯一索引兜底。
4. UUID 不等于安全 token，安全 token 用 SecureRandom。
5. 数据库存 UUID 时，主键/索引优先 BINARY(16)，业务展示可用 CHAR(32)。
```

面试时最关键的一句话：

> UUID 适合解决“分布式环境下无需中心协调生成唯一标识”的问题，但它不是万能主键方案。尤其在 MySQL InnoDB 中，随机 UUID v4 作为聚簇主键会带来随机写入、页分裂和索引膨胀问题；生产上通常用 BIGINT/雪花 ID 做内部主键，用 UUID 做外部唯一标识、traceId、幂等键或文件名。