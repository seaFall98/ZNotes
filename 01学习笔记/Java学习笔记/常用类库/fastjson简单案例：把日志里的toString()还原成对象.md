
[xfg原文](https://bugstack.cn/md/road-map/fastjson.html#_5-tostring-%E5%A4%84%E7%90%86)
## 1. 问题场景

线上日志里经常看到这种内容：

```text
OrderCreateRequest{orderId=1001, userId=9527, productName='Java实战课', amount=99.90}
```

它是 Java 对象的 `toString()` 输出。

问题是：

- 它不是 JSON，不能直接丢给 Postman。
    
- 不能直接用 fastjson 反序列化。
    
- 本地复现 bug 时，需要手动重新 new 对象，很麻烦。
    

我们想做的是：

```text
toString 日志文本
    ↓
还原成 Java Bean
    ↓
再转成 JSON
```

---

## 2. 如何解决

思路很简单：

1. 截取 `{}` 中间的内容。
    
2. 按 `,` 拆成一个个字段。
    
3. 每个字段按 `=` 拆成字段名和值。
    
4. 通过反射找到 Java 对象里的字段。
    
5. 根据字段类型，把字符串转换成 `Long`、`String`、`BigDecimal` 等。
    
6. 设置到对象里。
    
7. 最后用 fastjson 转成 JSON。
    

这就是 `toString2Bean` 的核心。

注意：  
它不是生产序列化方案，只是**本地调试工具**。

---

## 3. 代码

### 3.1 业务对象

```java
import java.math.BigDecimal;

public class OrderCreateRequest {

    private Long orderId;
    private Long userId;
    private String productName;
    private BigDecimal amount;

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

    @Override
    public String toString() {
        return "OrderCreateRequest{" +
                "orderId=" + orderId +
                ", userId=" + userId +
                ", productName='" + productName + '\'' +
                ", amount=" + amount +
                '}';
    }
}
```

---

### 3.2 `ToStringBeanUtils`

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
            // 创建目标对象，要求目标类有无参构造方法
            T target = targetClass.getDeclaredConstructor().newInstance();

            int start = text.indexOf("{");
            int end = text.lastIndexOf("}");

            if (start < 0 || end <= start) {
                throw new IllegalArgumentException("非法 toString 格式：" + text);
            }

            // 取出大括号中的内容：
            // orderId=1001, userId=9527, productName='Java实战课', amount=99.90
            String body = text.substring(start + 1, end);

            if (body.isBlank()) {
                return target;
            }

            // 简化处理：按 ", " 拆字段
            String[] fieldItems = body.split(", ");

            for (String item : fieldItems) {
                // 按第一个 "=" 拆成字段名和值
                String[] pair = item.split("=", 2);
                if (pair.length != 2) {
                    continue;
                }

                String fieldName = pair[0].trim();
                String rawValue = removeQuote(pair[1].trim());

                // 通过反射找到字段
                Field field = targetClass.getDeclaredField(fieldName);
                field.setAccessible(true);

                // 把字符串值转换成字段真实类型
                Object value = convertValue(rawValue, field.getType());

                // 设置字段值
                field.set(target, value);
            }

            return target;
        } catch (Exception e) {
            throw new IllegalArgumentException("toString 转 Bean 失败：" + text, e);
        }
    }

    private static String removeQuote(String value) {
        // 处理 'Java实战课'
        if (value.length() >= 2 && value.startsWith("'") && value.endsWith("'")) {
            return value.substring(1, value.length() - 1);
        }

        // 处理 "Java实战课"
        if (value.length() >= 2 && value.startsWith("\"") && value.endsWith("\"")) {
            return value.substring(1, value.length() - 1);
        }

        return value;
    }

    private static Object convertValue(String value, Class<?> type) {
        if ("null".equals(value)) {
            return null;
        }

        if (type == String.class) {
            return value;
        }

        if (type == Long.class || type == long.class) {
            return Long.valueOf(value);
        }

        if (type == Integer.class || type == int.class) {
            return Integer.valueOf(value);
        }

        if (type == BigDecimal.class) {
            return new BigDecimal(value);
        }

        throw new IllegalArgumentException("暂不支持的字段类型：" + type.getName());
    }
}
```

---

### 3.3 使用 fastjson 转成 JSON

```java
import com.alibaba.fastjson2.JSON;

public class ToStringBeanDemo {

    public static void main(String[] args) {
        String logText = "OrderCreateRequest{" +
                "orderId=1001, " +
                "userId=9527, " +
                "productName='Java实战课', " +
                "amount=99.90" +
                "}";

        // 1. 把 toString 文本还原成 Java 对象
        OrderCreateRequest request = ToStringBeanUtils.toObject(
                logText,
                OrderCreateRequest.class
        );

        // 2. 再用 fastjson 转成 JSON
        String json = JSON.toJSONString(request);

        System.out.println(json);
    }
}
```

输出：

```json
{"amount":99.90,"orderId":1001,"productName":"Java实战课","userId":9527}
```

---

## 4. 总结

`toString2Bean` 的作用很简单：

```text
把日志里的 toString 文本，临时还原成 Java 对象。
```

它适合：

- 根据线上日志快速复现问题；
    
- 把 `toString()` 文本转成 JSON；
    
- 辅助本地调试；
    
- 快速构造测试数据。
    

它不适合：

- 生产主流程；
    
- 复杂嵌套对象；
    
- List / Map；
    
- 字段值里包含逗号的情况；
    
- 替代 JSON 序列化。
    

最核心的一句话：

> **正常项目里应该优先打印 JSON 日志；`toString2Bean` 只是当日志里已经只有 `toString()` 时，用来补救和调试的小工具。**