```
我发现一个问题，
阿里的《Java开发手册》本身也推荐数据库字段用 `is_xxx` 格式（如 `is_deleted`），
同时强制规定Java布尔属性不能用 `is` 开头（应写成 `deleted`），
实际企业代码里，是怎么解决这个矛盾的？
```

---
## 结论

企业里通常**不解决成同名**，而是**分层命名 + 显式映射**：

|层|推荐命名|示例|
|---|---|---|
|MySQL 字段|`is_xxx`|`is_deleted`|
|Java 实体属性|去掉 `is`|`deleted`|
|Getter/Setter|JavaBean 规范|`getDeleted()` / `setDeleted()`|
|MyBatis / JPA 映射|显式映射|`is_deleted -> deleted`|

阿里手册英文版也明确写了：Java POJO 布尔属性不能以 `is` 开头，而数据库列名应该以 `is` 开头，需要建立属性和列的映射。([Alibaba Open Source](https://alibaba.github.io/Alibaba-Java-Coding-Guidelines/?utm_source=chatgpt.com "Table of Contents | Alibaba-Java-Coding-Guidelines"))

---

## 1. 标准企业写法

### 数据库

```sql
CREATE TABLE article (
    id BIGINT PRIMARY KEY,
    title VARCHAR(128) NOT NULL,
    is_deleted TINYINT UNSIGNED NOT NULL DEFAULT 0 COMMENT '是否删除：1是，0否'
);
```

### Java 实体

```java
@Data
public class ArticleDO {

    private Long id;

    private String title;

    /**
     * 对应数据库字段 is_deleted
     */
    private Boolean deleted;
}
```

### MyBatis 映射

```xml
<resultMap id="ArticleResultMap" type="com.example.ArticleDO">
    <id column="id" property="id"/>
    <result column="title" property="title"/>
    <result column="is_deleted" property="deleted"/>
</resultMap>
```

或者 MyBatis-Plus：

```java
@TableName("article")
public class ArticleDO {

    private Long id;

    @TableField("is_deleted")
    private Boolean deleted;
}
```

---

## 2. 为什么 Java 里不建议写 `isDeleted`

因为 JavaBean 规范、Lombok、Jackson、MyBatis、Spring BeanUtils、RPC 框架对布尔属性的识别可能不一致。

比如：

```java
private boolean isDeleted;
```

可能生成或识别成：

```java
public boolean isDeleted()
public void setDeleted(boolean deleted)
```

这时框架可能认为属性名是：

```java
deleted
```

而不是：

```java
isDeleted
```

于是 JSON、反射、对象拷贝、RPC 序列化、ORM 映射都可能出现奇怪问题。

所以企业里一般避免：

```java
private Boolean isDeleted; // 不推荐
```

而是写：

```java
private Boolean deleted; // 推荐
```

---

## 3. 真实项目中的常见落地方式

### 方式一：DO/Entity 层直接映射

最常见。

```java
@TableField("is_deleted")
private Boolean deleted;
```

优点是简单、稳定、符合规范。

---

### 方式二：数据库还是 `is_deleted`，接口返回也叫 `isDeleted`

有些前端喜欢看到：

```json
{
  "isDeleted": true
}
```

这时不要把 Java 字段写成 `isDeleted`，而是用序列化注解控制：

```java
@JsonProperty("isDeleted")
private Boolean deleted;
```

这样 Java 内部仍然是：

```java
private Boolean deleted;
```

但 API 输出可以是：

```json
{
  "isDeleted": true
}
```

---

### 方式三：DTO/VO 和 DO 分开

生产项目更推荐：

```java
public class ArticleDO {
    @TableField("is_deleted")
    private Boolean deleted;
}
```

```java
public class ArticleVO {
    @JsonProperty("isDeleted")
    private Boolean deleted;
}
```

数据库、Java 领域对象、前端 API 各自按自己的规则命名。

---

## 4. 面试回答版

可以这样说：

> 这个不是矛盾，而是分层规范不同。数据库字段用 `is_deleted` 是为了表达“是否”语义，并且配合 `tinyint(1)` 或 `tinyint unsigned` 表示 0/1 状态；Java POJO 不建议用 `isDeleted`，是因为 JavaBean、Lombok、Jackson、RPC、ORM 等框架对 boolean getter 的解析可能把属性识别成 `deleted`，导致序列化或映射异常。
> 
> 企业里一般数据库字段仍然叫 `is_deleted`，Java 属性叫 `deleted`，然后通过 MyBatis `resultMap`、MyBatis-Plus `@TableField("is_deleted")`、JPA `@Column(name = "is_deleted")` 做显式映射。如果接口层想返回 `isDeleted`，再通过 DTO/VO 或 `@JsonProperty("isDeleted")` 控制 JSON 字段名。核心原则是：数据库命名服务于数据语义，Java 属性命名服务于 JavaBean 规范，中间靠 ORM/序列化映射解耦。