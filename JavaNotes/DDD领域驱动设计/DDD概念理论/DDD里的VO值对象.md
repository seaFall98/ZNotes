# 总述

VO这个对象在领域服务方法的生命周期过程内是不可变对象，也没有唯一标识。它通常是配合实体对象使用。如为实体对象提供对象属性值的描述，比如：一个公司雇员的级别值对象，一个下单的商品收货的四级地址信息对象。所以在开发值对象的时候，通常不会提供 setter 方法，而是提供构造函数或者 Builder 方法来实例化对象。这个对象通常不会独立作为方法的入参对象，但做可以独立作为出参对象使用。

- 概念：值对象是由一组属性组成的，它们共同描述了一个领域概念。与实体（Entity）不同，值对象不需要有一个唯一的标识符来区分它们。值对象通常是不可变的，这意味着一旦创建，它们的状态就不应该改变。
- 特征：
    - **不可变性（Immutability）**：值对象一旦被创建，它的状态就不应该发生变化。这有助于保证领域模型的一致性和线程安全性。
    - **等价性（Equality）**：值对象的等价性不是基于身份或引用，而是基于对象的属性值。如果两个值对象的所有属性值都相等，那么这两个对象就被认为是等价的。
    - **替换性（Replaceability）**：由于值对象是不可变的，任何需要改变值对象的操作都会导致创建一个新的值对象实例，而不是修改现有的实例。
    - **侧重于描述事物的状态**：值对象通常用来描述事物的状态，而不是事物的唯一身份。
    - **可复用性（Reusability）**：值对象可以在不同的领域实体或其他值对象中重复使用。
- 用途：
    - 金额和货币（如价格、工资、费用等）
    - 度量和数据（如重量、长度、体积等）
    - 范围或区间（如日期范围、温度区间等）
    - 复杂的数学模型（如坐标、向量等）
    - 任何其他需要封装的属性集合
- 实现手段：
    - **定义不可变类**：确保类的所有属性都是私有的，并且只能通过构造函数来设置。
    - **重写equals和hashCode方法**：这样可以确保值对象的等价性是基于它们的属性值，而不是对象的引用。
    - **提供只读访问器**：只提供获取属性值的方法，不提供修改属性值的方法。
    - **使用工厂方法或构造函数创建实例**：这有助于确保值对象的有效性和一致性。
    - **考虑序列化支持**：如果值对象需要在网络上传输或存储到数据库中，需要提供序列化和反序列化的支持。


# MVC 和 DDD 里的 VO，不是同一个东西

在 Java 项目里，**VO** 这个缩写最容易混乱，因为它在不同语境下含义不同：

|语境|VO 全称|中文常见叫法|核心作用|
|---|---|---|---|
|MVC / Web 分层|View Object / Value Object，有时也叫 VO|视图对象 / 页面展示对象|给前端返回数据|
|DDD|Value Object|值对象|表达领域概念，没有独立身份|

这两个 VO **名字像，但职责完全不同**。

---

# 一、MVC 里的 VO

## 1. MVC 里的 VO 是什么？

在传统 Java Web / Spring Boot 项目里，VO 通常指：

> **View Object：面向页面展示的数据对象。**

也就是后端接口返回给前端的数据结构。

例如：

```java
public class ArticleVO {
    private Long id;
    private String title;
    private String authorName;
    private String summary;
    private String coverUrl;
    private Integer viewCount;
    private String createdAt;
}
```

这个 `ArticleVO` 不是数据库表，也不是领域模型，而是：

> 前端页面需要什么字段，我就组装什么字段。

---

## 2. MVC 里的常见对象分工

在普通 MVC 项目中，经常会看到这些对象：

```text
Controller
  ↓
Service
  ↓
Mapper / Repository
  ↓
Database
```

对应的数据对象可能有：

|对象|作用|
|---|---|
|Entity / DO / PO|数据库表映射对象|
|DTO|服务之间、接口之间传输数据|
|BO|业务处理过程对象|
|VO|返回给前端展示|
|Query / Req|前端请求参数对象|

例如：

```text
ArticleEntity   -> 数据库文章表
ArticleCreateReq -> 创建文章请求
ArticleVO       -> 文章详情返回给前端
```

---

## 3. MVC 里的 VO 示例

假设数据库表是：

```java
public class ArticleEntity {
    private Long id;
    private Long authorId;
    private String title;
    private String content;
    private Integer status;
    private LocalDateTime createdAt;
}
```

但是前端文章详情页需要：

```json
{
  "id": 1001,
  "title": "DDD 入门",
  "authorName": "z",
  "contentHtml": "<p>...</p>",
  "statusText": "已发布",
  "createdAt": "2026-05-09 22:30"
}
```

那么可以设计：

```java
public class ArticleDetailVO {
    private Long id;
    private String title;
    private String authorName;
    private String contentHtml;
    private String statusText;
    private String createdAt;
}
```

这里的 `ArticleDetailVO` 明显是**面向前端展示**的。

它可以包含：

```text
数据库没有的字段：authorName、statusText、contentHtml
格式化后的字段：createdAt 字符串
隐藏掉的字段：authorId、status、deleted、version
```

---

## 4. MVC 里的 VO 主要解决什么问题？

### 第一，避免直接暴露数据库 Entity

不推荐这样写：

```java
@GetMapping("/{id}")
public ArticleEntity detail(@PathVariable Long id) {
    return articleService.getById(id);
}
```

问题是：

```text
1. 数据库字段可能泄露给前端
2. 前端字段格式和数据库字段格式不一致
3. Entity 一改，接口响应结构也跟着变
4. 很难做字段聚合、翻译、脱敏、格式化
```

更好的方式：

```java
@GetMapping("/{id}")
public ArticleDetailVO detail(@PathVariable Long id) {
    return articleService.getArticleDetail(id);
}
```

---

### 第二，隔离前端展示结构和后端数据结构

数据库里可能是：

```java
private Integer status; // 0 草稿，1 发布
```

前端需要的是：

```java
private String statusText; // 草稿 / 已发布
```

这时候 VO 就很合适。

---

### 第三，服务多个页面场景

同一个 Article，可以有多个 VO：

```text
ArticleListVO     -> 文章列表页
ArticleDetailVO   -> 文章详情页
ArticleAdminVO    -> 后台管理页
ArticleSearchVO   -> 搜索结果页
```

它们字段不同，服务不同页面。

---

## 5. MVC 里的 VO 应该放在哪？

常见目录：

```text
controller
service
mapper
entity
dto
vo
```

例如：

```text
com.example.blog
  ├── controller
  ├── service
  ├── entity
  ├── mapper
  ├── dto
  ├── vo
  └── converter
```

或者按业务模块划分：

```text
article
  ├── controller
  ├── service
  ├── repository
  ├── entity
  ├── dto
  ├── vo
  └── converter
```

---

# 二、DDD 里的 VO

## 1. DDD 里的 VO 是什么？

DDD 里的 VO 通常明确指：

> **Value Object：值对象。**

它不是“给前端看的对象”，而是**领域模型的一部分**。

DDD 里的 Value Object 用来表达一个业务概念：

```text
金额 Money
地址 Address
邮箱 Email
手机号 PhoneNumber
时间范围 DateRange
坐标 Coordinate
文章标题 ArticleTitle
文章内容 ArticleContent
```

它的特点是：

> 没有独立身份，只通过属性值判断相等。

---

## 2. DDD 里的 VO 和 Entity 的区别

这是 DDD 里的核心区别。

|对象|是否有身份 ID|是否可变|判断相等的方式|例子|
|---|--:|--:|---|---|
|Entity|有|通常可变|根据 ID 判断|User、Order、Article|
|Value Object|没有|通常不可变|根据所有属性值判断|Money、Address、Email|

---

## 3. DDD Value Object 示例：Money

```java
public record Money(BigDecimal amount, Currency currency) {

    public Money {
        if (amount == null) {
            throw new IllegalArgumentException("金额不能为空");
        }
        if (currency == null) {
            throw new IllegalArgumentException("币种不能为空");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("金额不能为负数");
        }
    }

    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("不同币种不能相加");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

这个 `Money` 就是典型 DDD VO。

它不是为了返回给前端，而是表达领域规则：

```text
金额不能为负
币种不能为空
不同币种不能直接相加
金额相加后产生新的 Money
```

---

## 4. DDD Value Object 示例：Email

```java
public record Email(String value) {

    public Email {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("邮箱不能为空");
        }

        if (!value.matches("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+$")) {
            throw new IllegalArgumentException("邮箱格式不合法");
        }
    }
}
```

在领域模型里，不要直接写：

```java
private String email;
```

可以写成：

```java
private Email email;
```

这样一来，邮箱的合法性由 `Email` 自己保证，而不是到处散落：

```java
if (email == null || !email.contains("@")) {
    ...
}
```

---

## 5. DDD Value Object 示例：ArticleTitle

结合你之前的 DevWiki / Blog 场景，可以这样设计：

```java
public record ArticleTitle(String value) {

    private static final int MAX_LENGTH = 100;

    public ArticleTitle {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("文章标题不能为空");
        }

        if (value.length() > MAX_LENGTH) {
            throw new IllegalArgumentException("文章标题不能超过 100 个字符");
        }
    }
}
```

然后领域实体：

```java
public class Article {

    private ArticleId id;
    private ArticleTitle title;
    private ArticleContent content;
    private ArticleStatus status;

    public void rename(ArticleTitle newTitle) {
        this.title = newTitle;
    }

    public void publish() {
        if (content.isEmpty()) {
            throw new IllegalStateException("空文章不能发布");
        }
        this.status = ArticleStatus.PUBLISHED;
    }
}
```

这里的 `ArticleTitle` 是 DDD Value Object。

它不是返回给前端的 `ArticleVO`，而是领域内部的概念。

---

# 三、两个 VO 的核心区别

## 1. MVC VO：面向展示

MVC 里的 VO 关注：

```text
接口返回什么字段？
前端页面需要什么结构？
字段要不要格式化？
字段要不要脱敏？
字段要不要合并？
```

例如：

```java
public class ArticleDetailVO {
    private Long id;
    private String title;
    private String authorName;
    private String statusText;
    private String createdAt;
}
```

它是**出站数据模型**。

---

## 2. DDD VO：面向领域

DDD 里的 VO 关注：

```text
这个业务概念是什么？
它有哪些不变量？
它是否有身份？
它有哪些行为？
它如何保证自身合法？
```

例如：

```java
public record ArticleTitle(String value) {
    public ArticleTitle {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("文章标题不能为空");
        }
    }
}
```

它是**领域模型的一部分**。

---

# 四、一个完整例子：文章详情接口

## 1. DDD 领域层

```java
public class Article {

    private ArticleId id;
    private ArticleTitle title;
    private ArticleContent content;
    private AuthorId authorId;
    private ArticleStatus status;
    private LocalDateTime createdAt;

    public void publish() {
        if (content.isBlank()) {
            throw new IllegalStateException("文章内容为空，不能发布");
        }
        this.status = ArticleStatus.PUBLISHED;
    }

    public ArticleTitle getTitle() {
        return title;
    }

    public ArticleContent getContent() {
        return content;
    }
}
```

这里的：

```java
ArticleTitle
ArticleContent
ArticleId
AuthorId
```

都可以是 DDD Value Object。

---

## 2. DDD Value Object

```java
public record ArticleId(Long value) {
    public ArticleId {
        if (value == null || value <= 0) {
            throw new IllegalArgumentException("文章 ID 不合法");
        }
    }
}
```

```java
public record ArticleContent(String value) {
    public ArticleContent {
        if (value == null) {
            throw new IllegalArgumentException("文章内容不能为 null");
        }
    }

    public boolean isBlank() {
        return value.isBlank();
    }
}
```

---

## 3. 应用层组装返回对象

```java
public class ArticleAppService {

    private final ArticleRepository articleRepository;
    private final AuthorQueryService authorQueryService;

    public ArticleDetailVO getArticleDetail(Long articleId) {
        Article article = articleRepository.findById(new ArticleId(articleId));

        String authorName = authorQueryService.getAuthorName(article.getAuthorId());

        return new ArticleDetailVO(
            article.getId().value(),
            article.getTitle().value(),
            article.getContent().value(),
            authorName,
            article.getStatus().getText(),
            format(article.getCreatedAt())
        );
    }
}
```

---

## 4. MVC 返回给前端的 VO

```java
public record ArticleDetailVO(
    Long id,
    String title,
    String content,
    String authorName,
    String statusText,
    String createdAt
) {
}
```

这个 `ArticleDetailVO` 是 MVC/Web 层的 VO，不是 DDD Value Object。

---

# 五、包结构上怎么区分？

建议不要都叫 `vo` 包，否则一定会乱。

## 普通 MVC 项目可以这样：

```text
article
  ├── controller
  ├── service
  ├── entity
  ├── mapper
  ├── dto
  └── vo
```

这里的 `vo` 多半是 View Object。

---

## DDD 项目建议这样：

```text
article
  ├── domain
  │   ├── model
  │   │   ├── Article.java
  │   │   ├── ArticleId.java
  │   │   ├── ArticleTitle.java
  │   │   └── ArticleContent.java
  │   └── repository
  │       └── ArticleRepository.java
  │
  ├── application
  │   └── ArticleAppService.java
  │
  ├── interfaces
  │   ├── controller
  │   ├── request
  │   └── response
  │       └── ArticleDetailResponse.java
  │
  └── infrastructure
      ├── persistence
      └── mapper
```

这里我更建议：

```text
不要把前端返回对象叫 ArticleVO
而是叫 ArticleDetailResponse
```

这样可以减少歧义。

---

# 六、实战命名建议

## 如果是普通 MVC 项目

可以接受：

```text
ArticleVO
ArticleListVO
ArticleDetailVO
UserVO
OrderVO
```

但最好有清晰边界：

```text
controller.vo
controller.request
controller.response
```

---

## 如果是 DDD 项目

建议避免：

```text
domain.vo
interfaces.vo
```

因为两个地方都叫 VO，会很混乱。

更推荐：

### 领域层

```text
domain.model.valueobject
```

或者直接放在：

```text
domain.model.article
```

类名直接表达业务概念：

```text
ArticleTitle
ArticleContent
Email
Money
Address
DateRange
```

不要叫：

```text
ArticleTitleVO
EmailVO
MoneyVO
```

DDD 里通常不需要在类名后面加 `VO`。

---

### 接口层

返回给前端的对象建议叫：

```text
ArticleDetailResponse
ArticleListResponse
UserProfileResponse
OrderDetailResponse
```

请求对象叫：

```text
CreateArticleRequest
UpdateArticleRequest
PublishArticleRequest
```

这样比 `VO / DTO / BO` 满天飞更清楚。

---

# 七、最关键的判断标准

你可以用这个标准判断一个 VO 到底是哪种：

## 1. 它是为了接口返回给前端吗？

例如：

```java
public class ArticleDetailVO {
    private Long id;
    private String title;
    private String authorName;
    private String createdAt;
}
```

这是 **MVC VO / View Object**。

核心问题是：

```text
前端需要什么？
接口返回什么？
```

---

## 2. 它是在表达业务概念和业务规则吗？

例如：

```java
public record ArticleTitle(String value) {
    public ArticleTitle {
        if (value == null || value.isBlank()) {
            throw new IllegalArgumentException("标题不能为空");
        }
    }
}
```

这是 **DDD Value Object**。

核心问题是：

```text
这个业务概念是否有独立身份？
它有哪些不变量？
它如何保证自身合法？
```

---

# 八、总结一句话

**MVC 里的 VO，多数是 View Object，是给前端看的返回对象。**

**DDD 里的 VO，是 Value Object，是领域模型的一部分，用来表达没有独立身份、但有业务含义和规则的值。**

所以：

```text
ArticleDetailVO        -> MVC 展示对象
ArticleTitle           -> DDD 值对象
Money                  -> DDD 值对象
Email                  -> DDD 值对象
Address                -> DDD 值对象
UserProfileResponse    -> 更清晰的接口返回对象命名
```

在 DDD 项目里，我建议你尽量少用 `XXXVO` 这个名字，尤其不要让接口返回对象和领域值对象都叫 VO。最稳的做法是：

```text
接口层：Request / Response
应用层：Command / Query / DTO
领域层：Entity / Value Object / Aggregate
基础设施层：PO / DO / DataObject
```