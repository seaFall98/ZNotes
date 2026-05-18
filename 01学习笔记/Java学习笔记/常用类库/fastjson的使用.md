[fastjson](https://github.com/alibaba/fastjson)

# 1. 结论：新项目还推荐 fastjson 吗？

**结论先说：**

|场景|建议|
|---|---|
|**新项目**|优先用 **Jackson**，因为 Spring Boot 默认就是 Jackson，生态兼容最好|
|**已有项目大量使用 fastjson 1.x**|不建议继续扩展 1.x，新代码优先迁移到 **fastjson2**|
|**明确需要 fastjson 的性能、JSONPath、JSONB、兼容历史代码**|可以用 **fastjson2**|
|**接口 JSON 序列化 / 反序列化**|Spring Boot 项目中一般让 Jackson 处理，fastjson 更适合工具类、缓存、日志、内部转换|
|**涉及外部输入反序列化**|禁止随意开启 AutoType，尤其不要反序列化成任意 Object|

fastjson 官方仓库已经明确提示：**FASTJSON 2.0.x 已发布，更快、更安全，推荐升级到最新版**。fastjson2 官方定位是 fastjson 的下一代 JSON 库，支持 JSON/JSONB、JSONPath、服务端、Android、大数据、Kotlin、GraalVM 等场景。([GitHub](https://github.com/alibaba/fastjson?utm_source=chatgpt.com "alibaba/fastjson"))

截至当前可查信息，`com.alibaba.fastjson2:fastjson2` 在 Maven Repository 上已有 `2.0.62`，发布时间为 2026-05-05；fastjson 1.x 常见版本 `1.2.83` 则是 2022 年发布，并且历史漏洞数量非常高。([Maven Repository](https://mvnrepository.com/artifact/com.alibaba/fastjson?utm_source=chatgpt.com "com.alibaba » fastjson"))

所以实际建议是：

> **如果是你现在做新项目，不建议把 fastjson1 当核心 JSON 方案。要么沿用 Spring Boot 默认 Jackson，要么局部引入 fastjson2 作为工具库。**

---

# 2. Maven 依赖：优先 fastjson2

## 2.1 推荐依赖

```xml
<!-- fastjson2 核心依赖 -->
<dependency>
    <groupId>com.alibaba.fastjson2</groupId>
    <artifactId>fastjson2</artifactId>
    <version>2.0.62</version>
</dependency>
```

如果公司有统一依赖管理，建议放到父工程：

```xml
<properties>
    <!-- 统一版本，避免多个模块 fastjson2 版本不一致 -->
    <fastjson2.version>2.0.62</fastjson2.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.fastjson2</groupId>
            <artifactId>fastjson2</artifactId>
            <version>${fastjson2.version}</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

## 2.2 不建议新项目继续这样引入 fastjson1

```xml
<!-- 不建议新项目继续使用 fastjson 1.x -->
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>fastjson</artifactId>
    <version>1.2.83</version>
</dependency>
```

fastjson1 不是不能用，而是**新项目没有必要主动背它的历史包袱**。除非你维护的是老项目，并且项目中已经大量使用了 `com.alibaba.fastjson.JSON`、`JSONObject`、`JSONArray`。

---

# 3. 基础使用：对象、List、Map、嵌套对象互转

下面全部以 **fastjson2** 为主。

## 3.1 准备几个业务对象

假设有一个博客系统中的文章对象：

```java
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

public class ArticleDTO {

    private Long id;

    private String title;

    private String author;

    private BigDecimal rewardAmount;

    private List<String> tags;

    private AuthorDTO authorInfo;

    private LocalDateTime createTime;

    // 实际项目中建议用 Lombok，这里手写 getter/setter 便于示例完整
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public String getAuthor() {
        return author;
    }

    public void setAuthor(String author) {
        this.author = author;
    }

    public BigDecimal getRewardAmount() {
        return rewardAmount;
    }

    public void setRewardAmount(BigDecimal rewardAmount) {
        this.rewardAmount = rewardAmount;
    }

    public List<String> getTags() {
        return tags;
    }

    public void setTags(List<String> tags) {
        this.tags = tags;
    }

    public AuthorDTO getAuthorInfo() {
        return authorInfo;
    }

    public void setAuthorInfo(AuthorDTO authorInfo) {
        this.authorInfo = authorInfo;
    }

    public LocalDateTime getCreateTime() {
        return createTime;
    }

    public void setCreateTime(LocalDateTime createTime) {
        this.createTime = createTime;
    }
}
```

```java
public class AuthorDTO {

    private Long userId;

    private String nickname;

    public Long getUserId() {
        return userId;
    }

    public void setUserId(Long userId) {
        this.userId = userId;
    }

    public String getNickname() {
        return nickname;
    }

    public void setNickname(String nickname) {
        this.nickname = nickname;
    }
}
```

---

## 3.2 Java 对象转 JSON 字符串

```java
import com.alibaba.fastjson2.JSON;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

public class FastJsonBasicDemo {

    public static void main(String[] args) {
        ArticleDTO article = new ArticleDTO();
        article.setId(1001L);
        article.setTitle("Spring AI 实战入门");
        article.setAuthor("z");
        article.setRewardAmount(new BigDecimal("19.90"));
        article.setTags(List.of("Java", "Spring AI", "RAG"));
        article.setCreateTime(LocalDateTime.now());

        AuthorDTO author = new AuthorDTO();
        author.setUserId(1L);
        author.setNickname("seaFall");
        article.setAuthorInfo(author);

        // 对象序列化为 JSON 字符串，常用于日志、缓存、HTTP 请求体构造等场景
        String json = JSON.toJSONString(article);

        System.out.println(json);
    }
}
```

实际解决的问题：

- 把 Java 对象写入 Redis。
    
- 打印结构化日志。
    
- 调用第三方接口时构造 JSON 请求体。
    
- MQ 消息体序列化。
    

---

## 3.3 JSON 字符串转 Java 对象

```java
import com.alibaba.fastjson2.JSON;

public class FastJsonParseObjectDemo {

    public static void main(String[] args) {
        String json = """
                {
                  "id": 1001,
                  "title": "Spring AI 实战入门",
                  "author": "z",
                  "rewardAmount": 19.90,
                  "tags": ["Java", "Spring AI", "RAG"],
                  "authorInfo": {
                    "userId": 1,
                    "nickname": "seaFall"
                  },
                  "createTime": "2026-05-18 20:30:00"
                }
                """;

        // JSON 字符串反序列化为 Java 对象
        ArticleDTO article = JSON.parseObject(json, ArticleDTO.class);

        System.out.println(article.getTitle());
        System.out.println(article.getAuthorInfo().getNickname());
    }
}
```

这个场景很常见：

- Redis 中取出的 JSON 还原成对象。
    
- MQ 消息体消费时还原成业务对象。
    
- 调用第三方 HTTP 接口后，把响应 JSON 转成 DTO。
    
- 从数据库 `json` 字段读取配置对象。
    

---

## 3.4 List 转 JSON 与反序列化

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.TypeReference;

import java.util.List;

public class FastJsonListDemo {

    public static void main(String[] args) {
        ArticleDTO article1 = new ArticleDTO();
        article1.setId(1L);
        article1.setTitle("Redis 缓存设计");

        ArticleDTO article2 = new ArticleDTO();
        article2.setId(2L);
        article2.setTitle("RocketMQ 事务消息");

        List<ArticleDTO> articles = List.of(article1, article2);

        // List<ArticleDTO> 序列化为 JSON 数组字符串
        String jsonArray = JSON.toJSONString(articles);

        // 泛型 List 反序列化，推荐使用 TypeReference
        List<ArticleDTO> parsedList = JSON.parseObject(
                jsonArray,
                new TypeReference<List<ArticleDTO>>() {}
        );

        System.out.println(parsedList.get(0).getTitle());
    }
}
```

**重点：不要这样写：**

```java
// 不推荐：只能拿到原始 List，泛型信息不完整
List list = JSON.parseObject(jsonArray, List.class);
```

这种写法容易导致里面的元素变成 `JSONObject`，后续强转成 `ArticleDTO` 时出问题。

---

## 3.5 Map 转 JSON 与反序列化

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.TypeReference;

import java.util.Map;

public class FastJsonMapDemo {

    public static void main(String[] args) {
        Map<String, Object> cacheData = Map.of(
                "articleId", 1001L,
                "title", "fastjson2 实战",
                "viewCount", 128
        );

        // Map 序列化，常用于灵活结构的缓存、日志、扩展字段
        String json = JSON.toJSONString(cacheData);

        // Map 反序列化时，也建议用 TypeReference 保留泛型信息
        Map<String, Object> parsedMap = JSON.parseObject(
                json,
                new TypeReference<Map<String, Object>>() {}
        );

        System.out.println(parsedMap.get("title"));
    }
}
```

实际项目里，`Map<String, Object>` 常用于：

- 扩展字段。
    
- 审计日志。
    
- 动态配置。
    
- 第三方接口原始响应。
    
- 埋点事件参数。
    

但注意：**核心业务对象不要长期用 Map 代替 DTO**，否则可维护性会变差。

---

# 4. 进阶使用：泛型、字段名、日期、空值、字段过滤

## 4.1 泛型反序列化：TypeReference 是重点

实际项目中经常有统一响应结构：

```java
public class ApiResponse<T> {

    private Integer code;

    private String message;

    private T data;

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }

    public String getMessage() {
        return message;
    }

    public void setMessage(String message) {
        this.message = message;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }
}
```

第三方接口返回：

```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1001,
    "title": "fastjson2 实战"
  }
}
```

正确写法：

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.TypeReference;

public class FastJsonGenericDemo {

    public static void main(String[] args) {
        String responseJson = """
                {
                  "code": 200,
                  "message": "success",
                  "data": {
                    "id": 1001,
                    "title": "fastjson2 实战"
                  }
                }
                """;

        // 正确：保留 ApiResponse<ArticleDTO> 的完整泛型信息
        ApiResponse<ArticleDTO> response = JSON.parseObject(
                responseJson,
                new TypeReference<ApiResponse<ArticleDTO>>() {}
        );

        ArticleDTO article = response.getData();
        System.out.println(article.getTitle());
    }
}
```

错误写法：

```java
// 错误示例：运行时 T 的类型信息丢失，data 可能变成 JSONObject
ApiResponse response = JSON.parseObject(responseJson, ApiResponse.class);
```

**工程经验：**

凡是遇到：

```java
Result<T>
PageResult<T>
List<T>
Map<String, T>
ApiResponse<List<T>>
```

都不要直接用 `Class` 反序列化，要用 `TypeReference`。

---

## 4.2 字段名不一致：使用 `@JSONField`

第三方接口返回字段可能是下划线：

```json
{
  "article_id": 1001,
  "article_title": "fastjson2 实战"
}
```

Java 里一般是驼峰：

```java
import com.alibaba.fastjson2.annotation.JSONField;

public class ThirdPartyArticleDTO {

    @JSONField(name = "article_id")
    private Long articleId;

    @JSONField(name = "article_title")
    private String articleTitle;

    public Long getArticleId() {
        return articleId;
    }

    public void setArticleId(Long articleId) {
        this.articleId = articleId;
    }

    public String getArticleTitle() {
        return articleTitle;
    }

    public void setArticleTitle(String articleTitle) {
        this.articleTitle = articleTitle;
    }
}
```

使用：

```java
import com.alibaba.fastjson2.JSON;

public class FastJsonFieldNameDemo {

    public static void main(String[] args) {
        String json = """
                {
                  "article_id": 1001,
                  "article_title": "fastjson2 实战"
                }
                """;

        // @JSONField 会把 article_id 映射到 articleId
        ThirdPartyArticleDTO dto = JSON.parseObject(json, ThirdPartyArticleDTO.class);

        System.out.println(dto.getArticleId());
        System.out.println(dto.getArticleTitle());
    }
}
```

实际解决的问题：

- 对接第三方接口字段风格不一致。
    
- 老系统字段命名不规范。
    
- JSON 字段需要兼容前端已有协议。
    

---

## 4.3 日期格式：不要让日期格式散落在业务代码里

### DTO 上指定格式

```java
import com.alibaba.fastjson2.annotation.JSONField;

import java.time.LocalDateTime;

public class ArticleTimeDTO {

    private Long id;

    private String title;

    @JSONField(format = "yyyy-MM-dd HH:mm:ss")
    private LocalDateTime createTime;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public LocalDateTime getCreateTime() {
        return createTime;
    }

    public void setCreateTime(LocalDateTime createTime) {
        this.createTime = createTime;
    }
}
```

使用：

```java
import com.alibaba.fastjson2.JSON;

import java.time.LocalDateTime;

public class FastJsonDateDemo {

    public static void main(String[] args) {
        ArticleTimeDTO dto = new ArticleTimeDTO();
        dto.setId(1001L);
        dto.setTitle("日期格式处理");
        dto.setCreateTime(LocalDateTime.now());

        // createTime 会按照 @JSONField 指定格式输出
        String json = JSON.toJSONString(dto);

        System.out.println(json);
    }
}
```

**生产建议：**

- 内部系统优先使用 ISO-8601 或统一格式。
    
- API 层日期格式必须有统一规范。
    
- 不要一个接口 `yyyy-MM-dd HH:mm:ss`，另一个接口时间戳，另一个接口 ISO 字符串。
    
- 涉及时区时，不要只用 `LocalDateTime`，考虑 `OffsetDateTime`、`ZonedDateTime` 或统一 UTC 时间戳。
    

---

## 4.4 空值处理：要明确“空字段是否输出”

默认情况下，某些空字段可能不输出。实际业务里要根据场景选择。

例如前端希望字段稳定存在：

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.JSONWriter;

public class FastJsonNullDemo {

    public static void main(String[] args) {
        ArticleDTO dto = new ArticleDTO();
        dto.setId(1001L);
        dto.setTitle(null);

        // WriteNulls：输出 null 字段，保证 JSON 字段结构稳定
        String json = JSON.toJSONString(dto, JSONWriter.Feature.WriteNulls);

        System.out.println(json);
    }
}
```

常见场景：

|场景|建议|
|---|---|
|对外 API 响应|通常建议字段结构稳定，必要时输出 null|
|Redis 缓存|可以不输出 null，减少体积|
|日志打印|可以输出 null，便于排查|
|MQ 消息|建议结构稳定，消费者更容易兼容|

fastjson2 的行为可以通过 `JSONWriter.Feature`、`JSONReader.Feature` 调整，官方文档也列出了大量序列化和反序列化特性配置。([Alibaba Open Source](https://alibaba.github.io/fastjson2/features_cn.html?utm_source=chatgpt.com "通过Features配置序列化和反序列化的行为| fastjson2"))

---

## 4.5 字段过滤：日志、脱敏、缓存都能用

假设用户对象里有敏感字段：

```java
public class UserDTO {

    private Long id;

    private String username;

    private String password;

    private String phone;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    // 敏感字段，不应该直接输出到日志或前端
    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    // 手机号也属于敏感信息，日志中通常需要脱敏
    public String getPhone() {
        return phone;
    }

    public void setPhone(String phone) {
        this.phone = phone;
    }
}
```

用过滤器排除密码：

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.filter.PropertyFilter;

public class FastJsonFilterDemo {

    public static void main(String[] args) {
        UserDTO user = new UserDTO();
        user.setId(1L);
        user.setUsername("z");
        user.setPassword("123456");
        user.setPhone("13800000000");

        // 字段过滤器：password 字段不参与序列化
        PropertyFilter filter = (object, name, value) -> !"password".equals(name);

        String json = JSON.toJSONString(user, filter);

        System.out.println(json);
    }
}
```

更推荐生产中这样做：

- 对外响应 DTO 不放 `password` 字段。
    
- 日志打印使用统一脱敏工具。
    
- 不要靠开发人员每次手动记得过滤。
    
- 敏感信息过滤要平台化，而不是散落在业务代码里。
    

---

# 5. Spring Boot 实战

## 5.1 Controller 中是否应该显式使用 fastjson？

通常不建议这样写：

```java
@PostMapping("/articles")
public String createArticle(@RequestBody String body) {
    ArticleDTO dto = JSON.parseObject(body, ArticleDTO.class);
    return JSON.toJSONString(dto);
}
```

更推荐：

```java
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/articles")
public class ArticleController {

    private final ArticleService articleService;

    public ArticleController(ArticleService articleService) {
        this.articleService = articleService;
    }

    @PostMapping
    public ArticleDTO createArticle(@RequestBody ArticleDTO request) {
        // Spring Boot 默认由 Jackson 完成 HTTP JSON 到 Java 对象的转换
        // Controller 里只关心业务语义，不直接操作 JSON 字符串
        return articleService.createArticle(request);
    }
}
```

原因：

- Spring Boot 默认 JSON 处理链路是 Jackson。
    
- Controller 里手动 `parseObject` 会破坏框架约定。
    
- 参数校验、异常处理、OpenAPI 文档生成都会变麻烦。
    
- 以后切换 JSON 库成本更高。
    

**结论：**

> HTTP 接口层让 Spring MVC/Jackson 处理；fastjson 更适合在 Service、Client、Cache、Log、MQ 等内部场景使用。

---

## 5.2 Service 中调用第三方接口：fastjson 适合处理非标准响应

例如第三方接口返回 JSON：

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.JSONObject;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class ThirdPartyArticleClient {

    private final RestTemplate restTemplate;

    public ThirdPartyArticleClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public ThirdPartyArticleDTO getArticle(Long articleId) {
        String url = "https://example.com/api/articles/" + articleId;

        // 第三方接口返回原始 JSON 字符串
        String responseJson = restTemplate.getForObject(url, String.class);

        // 先解析为 JSONObject，适合处理字段不稳定的第三方响应
        JSONObject root = JSON.parseObject(responseJson);

        Integer code = root.getInteger("code");
        if (code == null || code != 200) {
            throw new IllegalStateException("第三方文章接口调用失败：" + responseJson);
        }

        // data 部分再转成强类型 DTO
        JSONObject data = root.getJSONObject("data");
        return data.toJavaObject(ThirdPartyArticleDTO.class);
    }
}
```

这种写法适合：

- 第三方接口字段不稳定。
    
- 外层响应结构固定，内层 data 类型不同。
    
- 需要先判断 `code`、`message`，再解析业务数据。
    

但如果接口协议稳定，优先直接反序列化：

```java
ApiResponse<ThirdPartyArticleDTO> response = JSON.parseObject(
        responseJson,
        new TypeReference<ApiResponse<ThirdPartyArticleDTO>>() {}
);
```

---

## 5.3 Redis 缓存中使用 fastjson

### 错误倾向：直接缓存 Java 对象

很多初学者喜欢直接把对象丢给 RedisTemplate，用 JDK 序列化。这会产生几个问题：

- Redis 中内容不可读。
    
- 跨语言不友好。
    
- 类结构变化后容易反序列化失败。
    
- 体积偏大。
    

### 推荐：业务对象转 JSON 字符串

```java
import com.alibaba.fastjson2.JSON;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.stereotype.Repository;

import java.time.Duration;

@Repository
public class ArticleCacheRepository {

    private final StringRedisTemplate stringRedisTemplate;

    public ArticleCacheRepository(StringRedisTemplate stringRedisTemplate) {
        this.stringRedisTemplate = stringRedisTemplate;
    }

    public void saveArticle(ArticleDTO article) {
        String key = buildArticleKey(article.getId());

        // 对象转 JSON，存入 Redis，便于排查和跨语言读取
        String value = JSON.toJSONString(article);

        // 设置过期时间，避免缓存无限膨胀
        stringRedisTemplate.opsForValue().set(key, value, Duration.ofMinutes(30));
    }

    public ArticleDTO getArticle(Long articleId) {
        String key = buildArticleKey(articleId);
        String value = stringRedisTemplate.opsForValue().get(key);

        if (value == null || value.isBlank()) {
            return null;
        }

        // 从 Redis JSON 字符串还原成业务对象
        return JSON.parseObject(value, ArticleDTO.class);
    }

    private String buildArticleKey(Long articleId) {
        return "article:detail:" + articleId;
    }
}
```

实际解决的问题：

- Redis 中 value 可读。
    
- 排查线上问题更方便。
    
- 不强绑定 Java 序列化机制。
    
- 适合多语言服务读取。
    

但要注意：**缓存中的 JSON 是一种隐形协议**。DTO 字段变更时，要考虑兼容性。

---

## 5.4 MQ 消息体中使用 fastjson

定义消息：

```java
import java.time.LocalDateTime;

public class ArticlePublishedEvent {

    private Long articleId;

    private Long authorId;

    private String title;

    private LocalDateTime publishedAt;

    public Long getArticleId() {
        return articleId;
    }

    public void setArticleId(Long articleId) {
        this.articleId = articleId;
    }

    public Long getAuthorId() {
        return authorId;
    }

    public void setAuthorId(Long authorId) {
        this.authorId = authorId;
    }

    public String getTitle() {
        return title;
    }

    public void setTitle(String title) {
        this.title = title;
    }

    public LocalDateTime getPublishedAt() {
        return publishedAt;
    }

    public void setPublishedAt(LocalDateTime publishedAt) {
        this.publishedAt = publishedAt;
    }
}
```

生产者：

```java
import com.alibaba.fastjson2.JSON;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;

@Service
public class ArticleEventPublisher {

    public void publishArticleCreated(Long articleId, Long authorId, String title) {
        ArticlePublishedEvent event = new ArticlePublishedEvent();
        event.setArticleId(articleId);
        event.setAuthorId(authorId);
        event.setTitle(title);
        event.setPublishedAt(LocalDateTime.now());

        // MQ 消息体一般使用 JSON，便于跨服务、跨语言消费
        String messageBody = JSON.toJSONString(event);

        // 示例：这里替换成 RocketMQTemplate / KafkaTemplate / RabbitTemplate
        sendMessage("article.published", messageBody);
    }

    private void sendMessage(String topic, String body) {
        // 实际项目中这里调用 MQ SDK
        System.out.println("topic = " + topic);
        System.out.println("body = " + body);
    }
}
```

消费者：

```java
import com.alibaba.fastjson2.JSON;
import org.springframework.stereotype.Component;

@Component
public class ArticleEventConsumer {

    public void onMessage(String messageBody) {
        ArticlePublishedEvent event;

        try {
            // 消费 MQ 时必须考虑脏数据、旧版本消息、字段缺失
            event = JSON.parseObject(messageBody, ArticlePublishedEvent.class);
        } catch (Exception ex) {
            // 生产中这里应该记录原始消息，并进入死信队列或告警
            throw new IllegalArgumentException("文章发布事件反序列化失败，body=" + messageBody, ex);
        }

        // 后续执行业务逻辑，例如刷新缓存、同步搜索索引、发通知
        handleArticlePublished(event);
    }

    private void handleArticlePublished(ArticlePublishedEvent event) {
        System.out.println("处理文章发布事件：" + event.getArticleId());
    }
}
```

MQ 场景要特别注意：

- 消息结构要版本化。
    
- 不要随便删除字段。
    
- 消费者要兼容旧消息。
    
- 反序列化失败要进入死信/告警，不要无限重试打爆系统。
    

---

## 5.5 日志打印：不要直接输出整个对象

常见错误：

```java
// 不推荐：可能把 password、token、手机号、身份证等敏感信息打进日志
log.info("user={}", JSON.toJSONString(user));
```

更好的做法：

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.filter.PropertyFilter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class SafeLogDemo {

    private static final Logger log = LoggerFactory.getLogger(SafeLogDemo.class);

    public void printUserLog(UserDTO user) {
        // 统一过滤敏感字段，避免敏感信息进入日志系统
        PropertyFilter safeLogFilter = (object, name, value) ->
                !"password".equals(name)
                        && !"token".equals(name)
                        && !"idCard".equals(name);

        String safeJson = JSON.toJSONString(user, safeLogFilter);

        log.info("用户信息：{}", safeJson);
    }
}
```

生产环境更推荐做成工具类：

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.filter.PropertyFilter;

public final class JsonLogUtils {

    private JsonLogUtils() {
    }

    private static final PropertyFilter SENSITIVE_FIELD_FILTER = (object, name, value) ->
            !"password".equalsIgnoreCase(name)
                    && !"token".equalsIgnoreCase(name)
                    && !"accessToken".equalsIgnoreCase(name)
                    && !"refreshToken".equalsIgnoreCase(name)
                    && !"idCard".equalsIgnoreCase(name);

    public static String toSafeJson(Object object) {
        if (object == null) {
            return "null";
        }

        // 日志序列化失败不能影响主流程，所以这里兜底
        try {
            return JSON.toJSONString(object, SENSITIVE_FIELD_FILTER);
        } catch (Exception ex) {
            return "[JSON_SERIALIZE_ERROR: " + ex.getMessage() + "]";
        }
    }
}
```

使用：

```java
log.info("创建文章请求 request={}", JsonLogUtils.toSafeJson(request));
```

---

# 6. 生产坑点

## 6.1 泛型丢失

错误写法：

```java
ApiResponse response = JSON.parseObject(json, ApiResponse.class);

// 这里 data 可能不是 ArticleDTO，而是 JSONObject
ArticleDTO article = (ArticleDTO) response.getData();
```

正确写法：

```java
ApiResponse<ArticleDTO> response = JSON.parseObject(
        json,
        new TypeReference<ApiResponse<ArticleDTO>>() {}
);
```

**判断标准：**

只要目标类型里有 `<T>`，基本就该用 `TypeReference`。

---

## 6.2 AutoType 安全风险

AutoType 的本质是：JSON 里可以带类型信息，让反序列化时按指定类型创建对象。

问题在于：

> 如果允许外部输入决定反序列化成什么 Java 类型，就可能带来严重安全风险。

fastjson2 官方资料中也强调了安全默认策略，例如 AutoType 默认关闭、支持 SafeMode 等。([Gitee](https://gitee.com/alibaba/fastjson2?utm_source=chatgpt.com "Alibaba/fastjson2"))

生产建议：

```java
// 不要为了“方便”在全局打开 AutoType
// 尤其不要对外部 HTTP 请求、MQ 消息、Redis 数据、第三方响应直接开启 AutoType
```

更安全的做法是：

```java
// 明确指定目标类型
ArticleDTO article = JSON.parseObject(json, ArticleDTO.class);
```

或者：

```java
ApiResponse<ArticleDTO> response = JSON.parseObject(
        json,
        new TypeReference<ApiResponse<ArticleDTO>>() {}
);
```

**核心原则：**

> 反序列化目标类型应该由服务端代码决定，而不是由 JSON 数据决定。

---

## 6.3 BigDecimal 精度问题

金额、积分、比例等字段不要用 `Double`。

错误示例：

```java
public class OrderDTO {
    // 不推荐：金额不能用 Double，容易出现精度问题
    private Double amount;
}
```

正确示例：

```java
import java.math.BigDecimal;

public class OrderDTO {

    // 金额字段使用 BigDecimal
    private BigDecimal amount;

    public BigDecimal getAmount() {
        return amount;
    }

    public void setAmount(BigDecimal amount) {
        this.amount = amount;
    }
}
```

使用：

```java
import com.alibaba.fastjson2.JSON;

import java.math.BigDecimal;

public class BigDecimalDemo {

    public static void main(String[] args) {
        String json = """
                {
                  "amount": 19.90
                }
                """;

        OrderDTO order = JSON.parseObject(json, OrderDTO.class);

        // 业务计算时继续使用 BigDecimal，不要转 double
        BigDecimal amount = order.getAmount();

        System.out.println(amount);
    }
}
```

生产建议：

- 金额统一 `BigDecimal`。
    
- 数据库存储用 `DECIMAL`。
    
- 前端展示再格式化。
    
- 不要在 JSON 转换链路中转成 `double`。
    

---

## 6.4 日期与时区问题

`LocalDateTime` 没有时区信息。

在单体项目里可能没问题，但在分布式系统、跨地区部署、日志追踪、跨语言服务里就容易出问题。

建议：

|场景|推荐|
|---|---|
|数据库存储|UTC 时间或明确业务时区|
|API 输出|ISO-8601 或统一格式|
|内部事件|时间戳 / OffsetDateTime|
|用户展示|根据用户时区格式化|

示例：

```java
import java.time.OffsetDateTime;

public class EventDTO {

    private Long eventId;

    // 比 LocalDateTime 多了时区偏移信息，更适合跨系统传输
    private OffsetDateTime eventTime;

    public Long getEventId() {
        return eventId;
    }

    public void setEventId(Long eventId) {
        this.eventId = eventId;
    }

    public OffsetDateTime getEventTime() {
        return eventTime;
    }

    public void setEventTime(OffsetDateTime eventTime) {
        this.eventTime = eventTime;
    }
}
```

---

## 6.5 循环引用问题

例如：

```java
public class DepartmentDTO {

    private Long id;

    private String name;

    private UserWithDeptDTO manager;

    // getter/setter 省略
}
```

```java
public class UserWithDeptDTO {

    private Long id;

    private String username;

    private DepartmentDTO department;

    // getter/setter 省略
}
```

如果 `department.manager.department.manager...` 互相引用，就可能导致序列化结果复杂，甚至出现异常或非预期输出。

生产建议：

- API DTO 不要直接复用 Entity。
    
- 不要在响应对象里放完整双向引用。
    
- 用轻量 DTO 打断引用链。
    

例如：

```java
public class UserViewDTO {

    private Long id;

    private String username;

    // 只放部门摘要，不放完整 Department 对象
    private DepartmentSimpleDTO department;

    // getter/setter 省略
}
```

```java
public class DepartmentSimpleDTO {

    private Long id;

    private String name;

    // 不再反向引用 users 或 manager，避免循环引用
    // getter/setter 省略
}
```

**关键点：**

> 循环引用不是 JSON 库的问题，本质是你的 API 模型设计不干净。

---

## 6.6 反序列化失败：不要让异常直接打穿主流程

错误写法：

```java
ArticleDTO article = JSON.parseObject(json, ArticleDTO.class);
```

如果这是 MQ 消费、Redis 缓存、第三方接口响应，直接抛异常可能影响主流程。

更好的写法：

```java
import com.alibaba.fastjson2.JSON;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class JsonUtils {

    private static final Logger log = LoggerFactory.getLogger(JsonUtils.class);

    private JsonUtils() {
    }

    public static <T> T parseObjectSafely(String json, Class<T> targetClass) {
        if (json == null || json.isBlank()) {
            return null;
        }

        try {
            return JSON.parseObject(json, targetClass);
        } catch (Exception ex) {
            // 注意：生产环境日志可能需要截断 json，避免超长日志或敏感信息泄露
            log.warn("JSON 反序列化失败，targetClass={}, json={}",
                    targetClass.getName(), json, ex);
            return null;
        }
    }
}
```

使用：

```java
ArticleDTO article = JsonUtils.parseObjectSafely(json, ArticleDTO.class);

if (article == null) {
    // 根据业务场景决定：返回缓存未命中、走降级、进入死信队列、抛业务异常
    throw new IllegalStateException("文章缓存数据格式异常");
}
```

---

# 7. 一个更接近企业项目的 JSON 工具类

不要在项目里到处写：

```java
JSON.toJSONString(xxx)
JSON.parseObject(xxx, Xxx.class)
```

建议封装一个轻量工具类，统一异常处理、日志策略、泛型处理。

```java
import com.alibaba.fastjson2.JSON;
import com.alibaba.fastjson2.TypeReference;
import com.alibaba.fastjson2.JSONWriter;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public final class JsonUtils {

    private static final Logger log = LoggerFactory.getLogger(JsonUtils.class);

    private JsonUtils() {
        // 工具类禁止实例化
    }

    public static String toJson(Object object) {
        if (object == null) {
            return null;
        }

        try {
            return JSON.toJSONString(object);
        } catch (Exception ex) {
            // 序列化失败一般说明对象结构有问题，例如循环引用、非法 getter 等
            log.warn("对象序列化 JSON 失败，class={}", object.getClass().getName(), ex);
            return null;
        }
    }

    public static String toJsonWithNulls(Object object) {
        if (object == null) {
            return null;
        }

        try {
            // 输出 null 字段，适合对外 API、结构化日志等需要字段稳定的场景
            return JSON.toJSONString(object, JSONWriter.Feature.WriteNulls);
        } catch (Exception ex) {
            log.warn("对象序列化 JSON 失败，class={}", object.getClass().getName(), ex);
            return null;
        }
    }

    public static <T> T parseObject(String json, Class<T> targetClass) {
        if (json == null || json.isBlank()) {
            return null;
        }

        try {
            return JSON.parseObject(json, targetClass);
        } catch (Exception ex) {
            // 不建议直接吞异常。这里返回 null 是一种策略，业务层必须判断
            log.warn("JSON 反序列化失败，targetClass={}, json={}",
                    targetClass.getName(), limit(json), ex);
            return null;
        }
    }

    public static <T> T parseObject(String json, TypeReference<T> typeReference) {
        if (json == null || json.isBlank()) {
            return null;
        }

        try {
            // 用于 Result<T>、List<T>、Map<String,T> 等泛型场景
            return JSON.parseObject(json, typeReference);
        } catch (Exception ex) {
            log.warn("JSON 泛型反序列化失败，type={}, json={}",
                    typeReference.getType(), limit(json), ex);
            return null;
        }
    }

    private static String limit(String json) {
        // 防止超长 JSON 打爆日志系统
        int maxLength = 2000;
        if (json.length() <= maxLength) {
            return json;
        }
        return json.substring(0, maxLength) + "...";
    }
}
```

使用：

```java
String json = JsonUtils.toJson(article);

ArticleDTO parsed = JsonUtils.parseObject(json, ArticleDTO.class);

ApiResponse<ArticleDTO> response = JsonUtils.parseObject(
        responseJson,
        new TypeReference<ApiResponse<ArticleDTO>>() {}
);
```

这个工具类解决几个实际问题：

- 统一 JSON 库入口。
    
- 后续从 fastjson2 切到 Jackson/Gson 时影响面更小。
    
- 统一异常处理。
    
- 防止超长 JSON 打爆日志。
    
- 泛型场景写法统一。
    

---

# 8. 是否要把 fastjson2 接入 Spring Boot 的全局 HTTP 序列化？

可以，但**不建议作为默认动作**。

Spring Boot 默认使用 Jackson。你强行替换成 fastjson2，会影响：

- `@RequestBody`
    
- `@ResponseBody`
    
- Spring MVC 消息转换器
    
- 日期格式
    
- SpringDoc / Swagger
    
- 第三方 Starter
    
- 参数校验异常响应
    
- Spring Security 相关响应
    

如果只是为了“项目用了 fastjson”，没必要替换全局 JSON 转换器。

更合理的策略：

```text
HTTP API 层：继续使用 Spring Boot 默认 Jackson
内部工具层：局部使用 fastjson2
Redis/MQ/第三方接口：根据场景使用 fastjson2
```

只有在这些情况下才考虑替换全局转换器：

- 公司基础框架统一要求。
    
- 历史项目已经基于 fastjson。
    
- 你明确测试过所有接口、日期、空值、泛型、异常响应。
    
- 有完整回归测试。
    

---

# 9. fastjson1 到 fastjson2 的代码迁移思路

fastjson1 常见写法：

```java
import com.alibaba.fastjson.JSON;

String json = JSON.toJSONString(article);
ArticleDTO article = JSON.parseObject(json, ArticleDTO.class);
```

fastjson2 写法：

```java
import com.alibaba.fastjson2.JSON;

String json = JSON.toJSONString(article);
ArticleDTO article = JSON.parseObject(json, ArticleDTO.class);
```

看起来只是包名变了：

```text
com.alibaba.fastjson.JSON
↓
com.alibaba.fastjson2.JSON
```

但真实迁移不能只改 import。

要重点回归：

|迁移点|为什么要测|
|---|---|
|日期格式|1.x 和 2.x 默认行为可能不同|
|空字段输出|前端可能依赖字段存在|
|JSONObject/JSONArray 使用|API 有差异|
|AutoType|安全策略不同|
|自定义序列化器|迁移成本较高|
|字段注解|包路径变了|
|泛型反序列化|历史代码可能存在隐患|
|Redis 老数据|老 JSON 是否能被新代码正确读取|
|MQ 老消息|消费者是否兼容历史消息|

---

# 10. 生产环境使用建议清单

## 10.1 选型建议

|问题|建议|
|---|---|
|新 Spring Boot 项目默认 JSON 库|Jackson|
|需要 fastjson|用 fastjson2，不用 fastjson1|
|老项目 fastjson1|制定迁移计划，不要继续扩展|
|对外接口 JSON|优先框架默认转换|
|内部缓存/MQ/日志|可以局部使用 fastjson2|
|安全敏感系统|禁止随意开启 AutoType|

---

## 10.2 代码实践建议

1. **不要到处散落 `JSON.toJSONString` 和 `JSON.parseObject`**  
    用 `JsonUtils` 统一封装。
    
2. **泛型反序列化必须用 `TypeReference`**  
    尤其是 `Result<T>`、`List<T>`、`Page<T>`、`Map<String, T>`。
    
3. **金额字段必须用 `BigDecimal`**  
    不要用 `Double`。
    
4. **DTO 和 Entity 分离**  
    不要直接把数据库 Entity 序列化给前端或 MQ。
    
5. **日期格式统一管理**  
    不要每个接口各写各的格式。
    
6. **日志序列化必须脱敏**  
    password、token、手机号、身份证、邮箱等都要谨慎。
    
7. **反序列化异常要有兜底策略**  
    MQ 进入死信，缓存走降级，第三方接口抛业务异常。
    
8. **Redis JSON 是隐形协议**  
    DTO 字段变更要考虑兼容老缓存。
    
9. **MQ 消息要版本化**  
    不要随便删除字段，消费者要兼容旧消息。
    
10. **不要全局开启 AutoType**  
    外部输入永远不能决定服务端反序列化类型。
    

---

# 11. 最后给一个工程化结论

fastjson 的正确使用姿势不是“会几个 API”：

```java
JSON.toJSONString(obj);
JSON.parseObject(json, Xxx.class);
```

这只是入门。

真正的生产级使用重点是：

```text
JSON 是系统之间的数据协议。
序列化 / 反序列化不是简单工具调用，而是边界治理问题。
```

在 Java 后端项目里，fastjson2 适合放在这些位置：

```text
第三方接口适配
Redis 缓存值转换
MQ 消息体转换
结构化日志输出
动态配置解析
JSON 字段处理
历史 fastjson 项目迁移
```

不适合滥用在这些位置：

```text
替代所有 DTO 设计
让 Controller 手动 parse JSON
把 Entity 直接暴露成 JSON
反序列化外部任意 Object
全局打开 AutoType
用 Map 逃避类型建模
```

一句话总结：

> **新项目默认 Jackson，局部场景用 fastjson2；老项目逐步从 fastjson1 迁移到 fastjson2；生产环境的重点不是 API，而是泛型、安全、日期、金额、日志脱敏、缓存兼容和消息协议稳定性。**