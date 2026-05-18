[xfg原文](https://bugstack.cn/md/road-map/fastjson.html#_5-tostring-%E5%A4%84%E7%90%86)
## 1. 先给结论

`toString2Bean` 不是 fastjson 的标准 API，也不是生产主流程应该依赖的序列化方案。

它更像一个**日志排查辅助工具**：

```text
线上日志里只有对象的 toString() 文本
    ↓
本地想快速还原成 Java 对象
    ↓
用自定义工具把 toString 字符串解析成 Bean
    ↓
再用 fastjson 转成 JSON
    ↓
方便格式化、调试、构造测试用例
```

它适合：

|场景|是否适合|
|---|---|
|本地调试线上日志|适合|
|快速把 `toString()` 文本转 JSON|适合|
|复现简单对象状态|适合|
|生产接口入参解析|不适合|
|MQ 消息反序列化|不适合|
|Redis 缓存反序列化|不适合|
|复杂嵌套对象恢复|不建议|
|不可信外部文本解析|不建议|

一句话：

> **它不是 JSON 序列化方案，而是“日志文本 → Java 对象 → JSON”的临时调试工具。**

---

# 2. 为什么会有这个需求？

实际项目里经常会打印日志：

```java
log.info("创建订单请求 request={}", request);
```

如果 `request` 重写了 `toString()`，日志可能是这样的：

```text
OrderCreateRequest(orderId=1001, userId=9527, productName=Java实战课, amount=99.90, couponId=3001)
```

或者是这样的：

```text
OrderCreateRequest{orderId=1001, userId=9527, productName='Java实战课', amount=99.90, couponId=3001}
```

这种日志对人看还可以，但对程序不友好：

- 不能直接复制到 Postman 当 JSON。
    
- 不能直接反序列化成 Java 对象。
    
- 不能直接格式化。
    
- 本地单测复现时要手动重新 new 对象。
    
- 字段多的时候，手动构造非常麻烦。
    

所以可以写一个工具：

```java
OrderCreateRequest request = ToStringBeanUtils.toObject(logText, OrderCreateRequest.class);

String json = JSON.toJSONString(request);
```

这样就能把日志内容转换成：

```json
{
  "orderId": 1001,
  "userId": 9527,
  "productName": "Java实战课",
  "amount": 99.90,
  "couponId": 3001
}
```

---

# 3. 案例目标

我们实现一个小工具，支持把这种 `toString()` 文本：

```text
OrderCreateRequest{orderId=1001, userId=9527, productName='Java实战课', amount=99.90, couponId=3001}
```

转换成 Java 对象：

```java
OrderCreateRequest request
```

再用 fastjson2 输出 JSON：

```java
String json = JSON.toJSONString(request);
```

---

# 4. 准备依赖

这里用 fastjson2。

```xml
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2</artifactId>
    <version>2.0.62</version>
</dependency>
```

注意：项目里不要同时混用 fastjson1 和 fastjson2 的 `JSONObject` / `JSONArray`，历史上确实出现过 1.x API 和 2.x 类型混用导致的转换问题。实际项目里建议明确统一版本和包路径。([GitHub](https://github.com/alibaba/fastjson2/issues/2665?utm_source=chatgpt.com "class com.alibaba.fastjson.JSONObject cannot be cast to ..."))

---

# 6. 实现 `ToStringBeanUtils`

# 5. 定义一个业务对象

这里模拟一个订单创建请求。

```java
import java.math.BigDecimal;

public class OrderCreateRequest {

    private Long orderId;

    private Long userId;

    private String productName;

    private BigDecimal amount;

    private Long couponId;

    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    public String getProductName() {
        return productName;
    }

    public void setProductName(String productName) {
        this.productName = productName;
    }

    public BigDecimal getAmount() {
        return amount;
    }

    public void setAmount(BigDecimal amount) {
        this.amount = amount;
    }

    public Long getCouponId() {
        return couponId;
    }

    public void setCouponId(Long couponId) {
        this.couponId = couponId;
    }

    @Override
    public String toString() {
        return "OrderCreateRequest{" +
                "orderId=" + orderId +
                ", userId=" + userId +
                ", productName='" + productName + '\'' +
                ", amount=" + amount +
                ", couponId=" + couponId +
                '}';
    }
}
```

这个对象的 `toString()` 输出类似：

```text
OrderCreateRequest{orderId=1001, userId=9527, productName='Java实战课', amount=99.90, couponId=3001}
```

---

## 6.1 第一版：支持简单对象

```java
import java.lang.reflect.Field;
import java.math.BigDecimal;

public final class ToStringBeanUtils {

    private ToStringBeanUtils() {
        // 工具类不允许实例化
    }

    public static <T> T toObject(String text, Class<T> targetClass) {
        if (text == null || text.isBlank()) {
            throw new IllegalArgumentException("toString 文本不能为空");
        }

        try {
            // 1. 创建目标对象，要求目标类必须有无参构造器
            T target = targetClass.getDeclaredConstructor().newInstance();

            // 2. 找到大括号内容，例如：
            // OrderCreateRequest{orderId=1001, userId=9527}
            int start = text.indexOf("{");
            int end = text.lastIndexOf("}");

            if (start < 0 || end <= start) {
                throw new IllegalArgumentException("非法 toString 格式：" + text);
            }

            // 3. 截取大括号内部内容：
            // orderId=1001, userId=9527, productName='Java实战课'
            String body = text.substring(start + 1, end);

            if (body.isBlank()) {
                return target;
            }

            // 4. 简单按 ", " 拆分字段
            // 注意：这个版本只适合简单对象，字段值中不能包含逗号
            String[] fieldItems = body.split(", ");

            for (String item : fieldItems) {
                // 5. 每个字段按第一个 "=" 拆分
                // 使用 split("=", 2)，避免字段值里偶然出现 "=" 时被拆坏
                String[] pair = item.split("=", 2);

                if (pair.length != 2) {
                    continue;
                }

                String fieldName = pair[0].trim();
                String rawValue = trimQuote(pair[1].trim());

                // 6. 根据字段名找到 Java 字段
                Field field = targetClass.getDeclaredField(fieldName);
                field.setAccessible(true);

                // 7. 根据字段类型做基础类型转换
                Object convertedValue = convertValue(rawValue, field.getType());

                // 8. 通过反射设置字段值
                field.set(target, convertedValue);
            }

            return target;
        } catch (Exception ex) {
            throw new IllegalArgumentException("toString 转 Bean 失败，text=" + text, ex);
        }
    }

    private static String trimQuote(String value) {
        // 处理字符串字段的单引号，例如：'Java实战课'
        if (value.length() >= 2 && value.startsWith("'") && value.endsWith("'")) {
            return value.substring(1, value.length() - 1);
        }

        // 处理字符串字段的双引号，例如："Java实战课"
        if (value.length() >= 2 && value.startsWith("\"") && value.endsWith("\"")) {
            return value.substring(1, value.length() - 1);
        }

        return value;
    }

    private static Object convertValue(String value, Class<?> targetType) {
        if ("null".equals(value)) {
            return null;
        }

        if (targetType == String.class) {
            return value;
        }

        if (targetType == Long.class || targetType == long.class) {
            return Long.valueOf(value);
        }

        if (targetType == Integer.class || targetType == int.class) {
            return Integer.valueOf(value);
        }

        if (targetType == Boolean.class || targetType == boolean.class) {
            return Boolean.valueOf(value);
        }

        if (targetType == BigDecimal.class) {
            // 金额字段不要转 Double，避免精度问题
            return new BigDecimal(value);
        }

        throw new IllegalArgumentException("暂不支持字段类型：" + targetType.getName());
    }
}
```

---

# 7. 使用 fastjson 把还原后的对象转成 JSON

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.JSONWriter;

public class ToStringBeanDemo {

    public static void main(String[] args) {
        String logText = "OrderCreateRequest{" +
                "orderId=1001, " +
                "userId=9527, " +
                "productName='Java实战课', " +
                "amount=99.90, " +
                "couponId=3001" +
                "}";

        // 1. 从日志中的 toString 文本恢复成 Java 对象
        OrderCreateRequest request = ToStringBeanUtils.toObject(
                logText,
                OrderCreateRequest.class
        );

        // 2. 再用 fastjson2 转成 JSON，方便格式化、调试、构造测试请求
        String json = JSON.toJSONString(
                request,
                JSONWriter.Feature.WriteNulls
        );

        System.out.println(json);
    }
}
```

输出结果类似：

```json
{
  "amount": 99.90,
  "couponId": 3001,
  "orderId": 1001,
  "productName": "Java实战课",
  "userId": 9527
}
```

---

# 8. 这个工具实际解决什么问题？

## 8.1 线上日志快速转 JSON

你从日志平台复制到一段：

```text
OrderCreateRequest{orderId=1001, userId=9527, productName='Java实战课', amount=99.90, couponId=3001}
```

不用手动改成：

```json
{
  "orderId": 1001,
  "userId": 9527,
  "productName": "Java实战课",
  "amount": 99.90,
  "couponId": 3001
}
```

而是直接：

```java
OrderCreateRequest request = ToStringBeanUtils.toObject(logText, OrderCreateRequest.class);

System.out.println(JSON.toJSONString(request));
```

这对本地复现 bug 很有用。

---

## 8.2 快速构造单测数据

例如你线上看到一个异常订单：

```text
OrderCreateRequest{orderId=1001, userId=9527, productName='Java实战课', amount=99.90, couponId=3001}
```

可以在单测中直接还原：

```java
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

public class OrderCreateRequestTest {

    @Test
    void should_parse_order_request_from_to_string_log() {
        String logText = "OrderCreateRequest{" +
                "orderId=1001, " +
                "userId=9527, " +
                "productName='Java实战课', " +
                "amount=99.90, " +
                "couponId=3001" +
                "}";

        OrderCreateRequest request = ToStringBeanUtils.toObject(
                logText,
                OrderCreateRequest.class
        );

        // 验证核心字段是否恢复正确
        assertThat(request.getOrderId()).isEqualTo(1001L);
        assertThat(request.getUserId()).isEqualTo(9527L);
        assertThat(request.getProductName()).isEqualTo("Java实战课");
    }
}
```

这样你可以基于线上日志快速构造回归测试。

---

# 9. 进一步优化：支持日期类型

实际业务里经常有时间字段。

新增对象：

```java
import java.math.BigDecimal;
import java.time.LocalDateTime;

public class PaymentNotifyRequest {

    private Long paymentId;

    private Long orderId;

    private BigDecimal payAmount;

    private LocalDateTime payTime;

    public Long getPaymentId() {
        return paymentId;
    }

    public void setPaymentId(Long paymentId) {
        this.paymentId = paymentId;
    }

    public Long getOrderId() {
        return orderId;
    }

    public void setOrderId(Long orderId) {
        this.orderId = orderId;
    }

    public BigDecimal getPayAmount() {
        return payAmount;
    }

    public void setPayAmount(BigDecimal payAmount) {
        this.payAmount = payAmount;
    }

    public LocalDateTime getPayTime() {
        return payTime;
    }

    public void setPayTime(LocalDateTime payTime) {
        this.payTime = payTime;
    }

    @Override
    public String toString() {
        return "PaymentNotifyRequest{" +
                "paymentId=" + paymentId +
                ", orderId=" + orderId +
                ", payAmount=" + payAmount +
                ", payTime='" + payTime + '\'' +
                '}';
    }
}
```

工具类中增加日期转换：

```java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
```

修改 `convertValue`：

```java
private static Object convertValue(String value, Class<?> targetType) {
    if ("null".equals(value)) {
        return null;
    }

    if (targetType == String.class) {
        return value;
    }

    if (targetType == Long.class || targetType == long.class) {
        return Long.valueOf(value);
    }

    if (targetType == Integer.class || targetType == int.class) {
        return Integer.valueOf(value);
    }

    if (targetType == Boolean.class || targetType == boolean.class) {
        return Boolean.valueOf(value);
    }

    if (targetType == BigDecimal.class) {
        return new BigDecimal(value);
    }

    if (targetType == LocalDateTime.class) {
        // 兼容 2026-05-18T20:30:00 这种默认 LocalDateTime.toString() 格式
        return LocalDateTime.parse(value, DateTimeFormatter.ISO_LOCAL_DATE_TIME);
    }

    throw new IllegalArgumentException("暂不支持字段类型：" + targetType.getName());
}
```

测试：

```java
import com.alibaba.fastjson2.JSON;

public class PaymentNotifyDemo {

    public static void main(String[] args) {
        String logText = "PaymentNotifyRequest{" +
                "paymentId=9001, " +
                "orderId=1001, " +
                "payAmount=99.90, " +
                "payTime='2026-05-18T20:30:00'" +
                "}";

        PaymentNotifyRequest request = ToStringBeanUtils.toObject(
                logText,
                PaymentNotifyRequest.class
        );

        System.out.println(JSON.toJSONString(request));
    }
}
```

---

# 10. 这个方案的核心问题

这个工具能用，但边界必须说清楚。

## 10.1 字段值里有逗号会出问题

例如：

```text
OrderCreateRequest{productName='Java, Spring, Redis', amount=99.90}
```

我们用的是：

```java
String[] fieldItems = body.split(", ");
```

这会把 `productName` 错误拆开。

所以这个工具只适合简单日志。

---

## 10.2 嵌套对象不适合

例如：

```text
OrderCreateRequest{orderId=1001, user=UserDTO{id=1, name='z'}}
```

这个解析会复杂很多，因为大括号里面还有大括号。

这种场景不建议继续增强工具，而是应该直接打印 JSON 日志。

---

## 10.3 List / Map 不适合

例如：

```text
OrderCreateRequest{orderId=1001, skuIds=[1, 2, 3]}
```

`[1, 2, 3]` 里面也有逗号，简单 split 会失败。

---

## 10.4 强依赖 `toString()` 格式

如果你原来是：

```text
OrderCreateRequest{orderId=1001}
```

后来 Lombok 改成：

```text
OrderCreateRequest(orderId=1001)
```

工具就可能失效。

所以它不是稳定协议。

---

# 11. 更工程化的建议

## 11.1 生产日志优先打印 JSON

更推荐这样：

```java
log.info("创建订单请求 request={}", JsonLogUtils.toSafeJson(request));
```

而不是：

```java
log.info("创建订单请求 request={}", request);
```

其中 `JsonLogUtils` 可以统一做：

- JSON 序列化
    
- 敏感字段脱敏
    
- 超长日志截断
    
- 异常兜底
    

---

## 11.2 `toString2Bean` 只作为补救工具

它主要解决的是这种历史遗留情况：

```text
之前日志没有打印 JSON，只打印了对象 toString()
现在排查问题时，需要把日志还原成对象
```

也就是说，它是**补救手段**，不是最佳实践。

---

# 12. 最终完整代码版本

```java
import java.lang.reflect.Field;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public final class ToStringBeanUtils {

    private ToStringBeanUtils() {
        // 工具类禁止实例化
    }

    public static <T> T toObject(String text, Class<T> targetClass) {
        if (text == null || text.isBlank()) {
            throw new IllegalArgumentException("toString 文本不能为空");
        }

        try {
            // 创建目标对象，要求目标类必须存在无参构造器
            T target = targetClass.getDeclaredConstructor().newInstance();

            int start = text.indexOf("{");
            int end = text.lastIndexOf("}");

            if (start < 0 || end <= start) {
                throw new IllegalArgumentException("非法 toString 格式：" + text);
            }

            String body = text.substring(start + 1, end);

            if (body.isBlank()) {
                return target;
            }

            // 当前实现只处理简单字段，不处理嵌套对象、List、Map
            String[] fieldItems = body.split(", ");

            for (String item : fieldItems) {
                String[] pair = item.split("=", 2);

                if (pair.length != 2) {
                    continue;
                }

                String fieldName = pair[0].trim();
                String rawValue = trimQuote(pair[1].trim());

                Field field = targetClass.getDeclaredField(fieldName);
                field.setAccessible(true);

                Object convertedValue = convertValue(rawValue, field.getType());
                field.set(target, convertedValue);
            }

            return target;
        } catch (Exception ex) {
            throw new IllegalArgumentException("toString 转 Bean 失败，text=" + text, ex);
        }
    }

    private static String trimQuote(String value) {
        if (value.length() >= 2 && value.startsWith("'") && value.endsWith("'")) {
            return value.substring(1, value.length() - 1);
        }

        if (value.length() >= 2 && value.startsWith("\"") && value.endsWith("\"")) {
            return value.substring(1, value.length() - 1);
        }

        return value;
    }

    private static Object convertValue(String value, Class<?> targetType) {
        if ("null".equals(value)) {
            return null;
        }

        if (targetType == String.class) {
            return value;
        }

        if (targetType == Long.class || targetType == long.class) {
            return Long.valueOf(value);
        }

        if (targetType == Integer.class || targetType == int.class) {
            return Integer.valueOf(value);
        }

        if (targetType == Boolean.class || targetType == boolean.class) {
            return Boolean.valueOf(value);
        }

        if (targetType == BigDecimal.class) {
            return new BigDecimal(value);
        }

        if (targetType == LocalDateTime.class) {
            return LocalDateTime.parse(value, DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        }

        throw new IllegalArgumentException("暂不支持字段类型：" + targetType.getName());
    }
}
```

---

# 13. 文章总结

`toString2Bean` 这个案例的价值不在于 fastjson API，而在于它体现了一个真实的工程问题：

> **线上日志里的对象状态，如何低成本还原成本地可调试的数据？**

完整链路是：

```text
线上日志 toString 文本
    ↓
自定义反射解析
    ↓
Java Bean
    ↓
fastjson 转 JSON
    ↓
本地调试 / 单测复现 / Postman 构造请求
```

但边界也很明确：

```text
它适合简单对象的调试恢复；
不适合生产链路；
不适合复杂嵌套结构；
不能替代标准 JSON 日志；
不能解析不可信外部输入。
```

最终建议：

> **生产系统最好直接打印安全脱敏后的 JSON 日志；`toString2Bean` 只作为历史日志、临时排查、本地复现问题时的辅助工具。**