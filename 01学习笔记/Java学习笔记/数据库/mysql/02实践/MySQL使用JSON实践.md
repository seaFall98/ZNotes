# prompt
```
请讲解 MySQL 8.x 的 JSON 功能。

要求：
1. 只讲具体使用方式，不介绍历史背景和概念铺垫。
2. 重点给出实际 SQL 代码案例，包括建表、插入、查询、更新、删除、索引优化等。
3. 每个案例配简短说明，说明控制在 1-2 句话。
4. 内容面向 Java 后端开发者，突出实际业务场景中的用法。
5. 控制篇幅，避免长篇理论，整体结构清晰、可快速复制使用。

请按以下结构输出：
- JSON 字段建表
- 插入 JSON 数据
- 查询 JSON 字段
- 修改 JSON 字段
- 删除 JSON 字段
- JSON 条件查询
- JSON 数组处理
- JSON 索引优化
- Java 后端使用注意点
```

---

# MySQL 8.x JSON 功能：Java 后端实用版

## 1. JSON 字段建表

### 场景：订单扩展信息、用户偏好、商品规格

```sql
CREATE TABLE user_profile (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    base_info JSON NOT NULL,
    preferences JSON NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);
```

说明：  
JSON 字段适合存放**结构灵活、查询频率不高、字段变化快**的数据，例如用户配置、扩展属性、前端表单快照。

---

### 示例：商品 SKU 扩展属性

```sql
CREATE TABLE product_sku (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    sku_code VARCHAR(64) NOT NULL,
    product_name VARCHAR(128) NOT NULL,
    specs JSON NOT NULL,
    ext_attrs JSON NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE KEY uk_sku_code (sku_code)
);
```

```json
{
  "color": "black",
  "size": "XL",
  "material": "cotton"
}
```

说明：  
商品规格字段经常因品类不同而变化，用 JSON 比不停加列更灵活。

---

## 2. 插入 JSON 数据

### 插入对象类型 JSON

```sql
INSERT INTO user_profile (user_id, base_info, preferences)
VALUES (
    1001,
    JSON_OBJECT(
        'nickname', 'zhangsan',
        'age', 25,
        'city', 'Shanghai'
    ),
    JSON_OBJECT(
        'theme', 'dark',
        'language', 'zh-CN',
        'notify', true
    )
);
```

说明：  
`JSON_OBJECT()` 可以在 SQL 中构造 JSON 对象，适合后端拼接参数后插入。

---

### 直接插入 JSON 字符串

```sql
INSERT INTO product_sku (sku_code, product_name, specs, ext_attrs)
VALUES (
    'SKU_10001',
    '程序员卫衣',
    '{
        "color": "black",
        "size": "XL",
        "material": "cotton"
    }',
    '{
        "tags": ["hot", "new"],
        "sales": 128,
        "visible": true
    }'
);
```

说明：  
MySQL 会校验 JSON 格式，不合法会插入失败。

---

### 插入 JSON 数组

```sql
CREATE TABLE article (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    title VARCHAR(128) NOT NULL,
    tags JSON NOT NULL
);

INSERT INTO article (title, tags)
VALUES (
    'MySQL JSON 实战',
    JSON_ARRAY('mysql', 'json', 'backend')
);
```

说明：  
数组适合存储标签、权限点、配置项列表，但不要滥用来替代多对多关系表。

---

## 3. 查询 JSON 字段

### 查询整个 JSON 字段

```sql
SELECT id, user_id, base_info, preferences
FROM user_profile
WHERE user_id = 1001;
```

说明：  
后端可以直接把 JSON 字段映射为 `String`、`Map`、DTO 或 `JsonNode`。

---

### 查询 JSON 中的某个字段

```sql
SELECT 
    user_id,
    JSON_EXTRACT(base_info, '$.nickname') AS nickname
FROM user_profile;
```

结果：

```text
"zhangsan"
```

说明：  
`JSON_EXTRACT(json_col, '$.path')` 会返回 JSON 类型，所以字符串结果会带双引号。

---

### 查询并去掉双引号

```sql
SELECT 
    user_id,
    JSON_UNQUOTE(JSON_EXTRACT(base_info, '$.nickname')) AS nickname
FROM user_profile;
```

也可以简写：

```sql
SELECT 
    user_id,
    base_info->>'$.nickname' AS nickname
FROM user_profile;
```

说明：  
`->` 返回 JSON，`->>` 返回去引号后的字符串，Java 后端查询展示时通常用 `->>`。

---

### 查询嵌套字段

```sql
SELECT 
    id,
    ext_attrs->>'$.manufacturer.name' AS manufacturer_name
FROM product_sku;
```

说明：  
JSON Path 使用 `$.a.b.c` 表示嵌套对象字段路径。

---

## 4. 修改 JSON 字段

### 更新某个 JSON 字段

```sql
UPDATE user_profile
SET preferences = JSON_SET(preferences, '$.theme', 'light')
WHERE user_id = 1001;
```

说明：  
`JSON_SET()` 会新增或覆盖指定路径的值。

---

### 新增 JSON 字段

```sql
UPDATE user_profile
SET preferences = JSON_SET(preferences, '$.fontSize', 16)
WHERE user_id = 1001;
```

说明：  
如果 `$.fontSize` 不存在，`JSON_SET()` 会自动新增。

---

### 只在字段不存在时插入

```sql
UPDATE user_profile
SET preferences = JSON_INSERT(preferences, '$.timezone', 'Asia/Shanghai')
WHERE user_id = 1001;
```

说明：  
`JSON_INSERT()` 只在路径不存在时插入，已存在则不覆盖。

---

### 只替换已存在字段

```sql
UPDATE user_profile
SET preferences = JSON_REPLACE(preferences, '$.language', 'en-US')
WHERE user_id = 1001;
```

说明：  
`JSON_REPLACE()` 只更新已存在的路径，不存在则不处理。

---

### 同时修改多个字段

```sql
UPDATE product_sku
SET specs = JSON_SET(
    specs,
    '$.color', 'white',
    '$.size', 'L'
)
WHERE sku_code = 'SKU_10001';
```

说明：  
`JSON_SET()` 支持一次更新多个 JSON Path。

---

## 5. 删除 JSON 字段

### 删除对象中的某个字段

```sql
UPDATE user_profile
SET preferences = JSON_REMOVE(preferences, '$.notify')
WHERE user_id = 1001;
```

说明：  
`JSON_REMOVE()` 删除 JSON 中指定路径，不影响其他字段。

---

### 删除嵌套字段

```sql
UPDATE product_sku
SET ext_attrs = JSON_REMOVE(ext_attrs, '$.manufacturer.address')
WHERE sku_code = 'SKU_10001';
```

说明：  
适合删除废弃的扩展配置字段。

---

### 删除数组中的元素

```sql
UPDATE article
SET tags = JSON_REMOVE(tags, '$[1]')
WHERE id = 1;
```

说明：  
`$[1]` 表示数组下标为 1 的元素，JSON 数组下标从 0 开始。

---

## 6. JSON 条件查询

### 根据 JSON 字段值查询

```sql
SELECT *
FROM user_profile
WHERE base_info->>'$.city' = 'Shanghai';
```

说明：  
这是后端最常见的 JSON 查询方式，但如果数据量大，需要配合索引优化。

---

### 数字比较

```sql
SELECT *
FROM user_profile
WHERE CAST(base_info->>'$.age' AS UNSIGNED) >= 18;
```

说明：  
`->>` 取出来是字符串，做数值比较时要显式转换类型。

---

### 布尔值查询

```sql
SELECT *
FROM user_profile
WHERE JSON_EXTRACT(preferences, '$.notify') = true;
```

或者：

```sql
SELECT *
FROM user_profile
WHERE preferences->>'$.notify' = 'true';
```

说明：  
布尔值查询时要注意 JSON 类型和字符串类型的差异。

---

### 判断字段是否存在

```sql
SELECT *
FROM user_profile
WHERE JSON_CONTAINS_PATH(preferences, 'one', '$.theme');
```

说明：  
`one` 表示任意一个路径存在即可，`all` 表示所有路径都必须存在。

---

### 判断 JSON 是否包含指定对象

```sql
SELECT *
FROM product_sku
WHERE JSON_CONTAINS(specs, JSON_OBJECT('color', 'black'));
```

说明：  
适合查询 JSON 对象里是否包含某个键值对。

---

## 7. JSON 数组处理

### 查询数组中是否包含某个值

```sql
SELECT *
FROM article
WHERE JSON_CONTAINS(tags, JSON_QUOTE('mysql'));
```

说明：  
`JSON_QUOTE('mysql')` 会转成合法 JSON 字符串 `"mysql"`。

---

### 查询数组长度

```sql
SELECT 
    id,
    title,
    JSON_LENGTH(tags) AS tag_count
FROM article;
```

说明：  
适合统计标签数量、配置项数量等。

---

### 向数组追加元素

```sql
UPDATE article
SET tags = JSON_ARRAY_APPEND(tags, '$', 'database')
WHERE id = 1;
```

说明：  
`$` 表示当前 JSON 数组本身，`JSON_ARRAY_APPEND()` 会在数组末尾追加元素。

---

### 按下标查询数组元素

```sql
SELECT 
    id,
    title,
    tags->>'$[0]' AS first_tag
FROM article;
```

说明：  
数组路径使用 `$[index]`，index 从 0 开始。

---

### 将 JSON 数组展开成行

```sql
SELECT 
    a.id,
    a.title,
    jt.tag
FROM article a,
JSON_TABLE(
    a.tags,
    '$[*]' COLUMNS (
        tag VARCHAR(64) PATH '$'
    )
) jt;
```

说明：  
`JSON_TABLE()` 可以把 JSON 数组转成关系型结果集，适合报表、搜索、数据清洗场景。

---

## 8. JSON 索引优化

## 8.1 使用生成列索引

### 场景：经常按城市查询用户

```sql
ALTER TABLE user_profile
ADD COLUMN city VARCHAR(64)
GENERATED ALWAYS AS (base_info->>'$.city') STORED,
ADD INDEX idx_city (city);
```

查询：

```sql
SELECT *
FROM user_profile
WHERE city = 'Shanghai';
```

说明：  
生产中推荐把高频查询的 JSON 字段提取为**生成列**，再给生成列加普通索引。

---

### 场景：按商品颜色筛选

```sql
ALTER TABLE product_sku
ADD COLUMN color VARCHAR(32)
GENERATED ALWAYS AS (specs->>'$.color') STORED,
ADD INDEX idx_color (color);
```

查询：

```sql
SELECT *
FROM product_sku
WHERE color = 'black';
```

说明：  
电商筛选条件如果频繁出现在 WHERE 中，不要每次直接扫 JSON 字段。

---

## 8.2 使用函数索引

MySQL 8.x 支持表达式索引，可以直接对 JSON 表达式建索引。

```sql
CREATE INDEX idx_user_city 
ON user_profile ((CAST(base_info->>'$.city' AS CHAR(64))));
```

查询：

```sql
SELECT *
FROM user_profile
WHERE CAST(base_info->>'$.city' AS CHAR(64)) = 'Shanghai';
```

说明：  
表达式索引要求查询条件中的表达式尽量和建索引时保持一致，否则可能无法命中索引。

---

## 8.3 数值字段索引

### 场景：按用户年龄筛选

```sql
ALTER TABLE user_profile
ADD COLUMN age INT
GENERATED ALWAYS AS (
    CAST(base_info->>'$.age' AS UNSIGNED)
) STORED,
ADD INDEX idx_age (age);
```

查询：

```sql
SELECT *
FROM user_profile
WHERE age >= 18;
```

说明：  
JSON 中的数字字段用于范围查询时，建议用生成列转成明确的数值类型再建索引。

---

## 8.4 查看是否命中索引

```sql
EXPLAIN
SELECT *
FROM user_profile
WHERE city = 'Shanghai';
```

说明：  
实际开发中不要只靠猜，JSON 查询是否命中索引必须用 `EXPLAIN` 验证。

---

## 9. Java 后端使用注意点

## 9.1 MyBatis 中用 String 接收 JSON

```java
@Data
public class UserProfileDO {

    private Long id;

    private Long userId;

    /**
     * JSON 字符串，例如：
     * {"nickname":"zhangsan","age":25,"city":"Shanghai"}
     */
    private String baseInfo;

    private String preferences;
}
```

说明：  
最简单的方式是数据库 JSON 字段在 Java 中用 `String` 接收，由 Jackson 手动转换。

---

## 9.2 Jackson 转 DTO

```java
@Data
public class BaseInfoDTO {
    private String nickname;
    private Integer age;
    private String city;
}
```

```java
ObjectMapper objectMapper = new ObjectMapper();

BaseInfoDTO baseInfo = objectMapper.readValue(
        userProfileDO.getBaseInfo(),
        BaseInfoDTO.class
);
```

说明：  
业务代码里不要到处手写 JSON Path，建议转换成 DTO 后使用。

---

## 9.3 写入前先序列化

```java
BaseInfoDTO baseInfo = new BaseInfoDTO();
baseInfo.setNickname("zhangsan");
baseInfo.setAge(25);
baseInfo.setCity("Shanghai");

String json = objectMapper.writeValueAsString(baseInfo);
```

```java
userProfileMapper.insert(userId, json);
```

说明：  
不要手动拼接 JSON 字符串，容易产生转义、格式错误和注入风险。

---

## 9.4 MyBatis 插入示例

```java
@Mapper
public interface UserProfileMapper {

    @Insert("""
        INSERT INTO user_profile (user_id, base_info, preferences)
        VALUES (#{userId}, CAST(#{baseInfo} AS JSON), CAST(#{preferences} AS JSON))
        """)
    int insert(@Param("userId") Long userId,
               @Param("baseInfo") String baseInfo,
               @Param("preferences") String preferences);
}
```

说明：  
`CAST(... AS JSON)` 可以让语义更清晰，但实际是否需要取决于驱动和 SQL 写法。

---

## 9.5 MyBatis 查询 JSON 字段

```java
@Select("""
    SELECT 
        id,
        user_id,
        base_info,
        preferences
    FROM user_profile
    WHERE user_id = #{userId}
    """)
UserProfileDO selectByUserId(@Param("userId") Long userId);
```

说明：  
常规查询可以直接把 JSON 字段当字符串取回。

---

## 9.6 查询 JSON 内部字段

```java
@Select("""
    SELECT 
        user_id,
        base_info->>'$.nickname' AS nickname,
        base_info->>'$.city' AS city
    FROM user_profile
    WHERE user_id = #{userId}
    """)
UserProfileView selectViewByUserId(@Param("userId") Long userId);
```

```java
@Data
public class UserProfileView {
    private Long userId;
    private String nickname;
    private String city;
}
```

说明：  
如果接口只需要少数字段，可以直接在 SQL 层提取 JSON 属性，避免返回整个 JSON。

---

# 最后总结

|场景|推荐做法|
|---|---|
|扩展属性、配置、表单快照|可以用 JSON|
|高频查询字段|抽成普通列或生成列并建索引|
|范围查询字段|生成列转成数值类型再索引|
|多对多关系|不要用 JSON 数组替代关系表|
|Java 接收 JSON|`String` / `Map` / DTO / `JsonNode`|
|业务代码处理|推荐 Jackson 转 DTO|
|大数据量查询 JSON|必须 `EXPLAIN` 验证索引|

**一句话：MySQL JSON 适合做“灵活扩展字段”，不适合替代正常表结构；一旦进入高频查询、排序、聚合，就应该抽字段、建生成列或回归关系模型。**