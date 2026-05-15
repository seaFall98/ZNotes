```
小傅哥这篇mysql连接池教学案例

https://bugstack.cn/md/road-map/connection-pool.html

给到的压测数据有一个地方一些疑问：总数据量是100万，但uq_uuid这个索引对应的区分度（Cardinality）不到100万，只有97万多。这是UUID发生了重复的问题吗？
```
## 结论

**不是。`Cardinality` 只有 97 万多，不能说明 UUID 发生了重复。**

在 MySQL 里，`SHOW INDEX` / `information_schema.statistics` 看到的 `Cardinality` 是**优化器统计信息中的“估算值”**，不是精确的 `COUNT(DISTINCT uuid)`。MySQL 官方文档明确说，`Cardinality` 是索引中唯一值数量的**估计值**，并且因为统计信息以整数形式保存，所以即使小表也不一定精确。InnoDB 更新统计信息时，会抽样索引页来估算基数。([MySQL开发者专区](https://dev.mysql.com/doc/en/show-index.html?utm_source=chatgpt.com "15.7.7.24 SHOW INDEX Statement"))

小傅哥这篇案例本身是在 100 万订单数据基础上做连接池、索引字段、查询更新性能压测，文章也提到初始化 100 万数据和相关压测场景。([BugStack](https://bugstack.cn/md/road-map/connection-pool.html "ConnectionPool | 小傅哥 bugstack 虫洞栈"))

---

## 1. 为什么 100 万数据，Cardinality 只有 97 万多？

因为它大概率来自类似：

```sql
SHOW INDEX FROM order_table;
```

或者：

```sql
SELECT *
FROM information_schema.statistics
WHERE table_schema = 'xxx'
  AND table_name = 'xxx';
```

里面的 `Cardinality` 字段。

这个字段的含义不是：

```sql
SELECT COUNT(DISTINCT uuid) FROM table;
```

而是 MySQL 优化器用于估算索引选择性的统计值。

所以：

```text
总行数：1,000,000
uq_uuid Cardinality：977,xxx
```

只能说明：

```text
MySQL 估算这个索引大约有 97 万多个不同值
```

不能推出：

```text
UUID 真实重复了 2 万多个
```

---

## 2. 如果 `uq_uuid` 真的是唯一索引，UUID 能重复插入吗？

要分情况。

### 情况一：`uuid` 是 `NOT NULL`，且 `uq_uuid` 是完整唯一索引

例如：

```sql
uuid varchar(64) NOT NULL,
UNIQUE KEY uq_uuid (uuid)
```

这种情况下，**重复 UUID 插不进去**。

MySQL 的 `UNIQUE` 索引会约束索引值必须唯一，如果插入相同 key，会报重复键错误。([MySQL开发者专区](https://dev.mysql.com/doc/en/create-index.html?utm_source=chatgpt.com "15.1.18 CREATE INDEX Statement"))

也就是说，如果已经成功插入 100 万行，并且 `uuid` 字段是 `NOT NULL + UNIQUE`，那么真实数据里不应该存在重复 UUID。

---

### 情况二：`uuid` 允许为 `NULL`

例如：

```sql
uuid varchar(64) DEFAULT NULL,
UNIQUE KEY uq_uuid (uuid)
```

这时要小心：**MySQL 的唯一索引允许多个 `NULL` 值**。官方文档明确说明，`UNIQUE` 索引要求非空值唯一，但可空列允许多个 `NULL`。([MySQL开发者专区](https://dev.mysql.com/doc/en/create-index.html?utm_source=chatgpt.com "15.1.18 CREATE INDEX Statement"))

所以如果字段允许 `NULL`，下面这种情况是可能的：

```text
uuid = NULL
uuid = NULL
uuid = NULL
```

它们不会违反唯一索引。

生产上如果 UUID 是业务唯一标识，建议明确：

```sql
uuid varchar(64) NOT NULL,
UNIQUE KEY uq_uuid (uuid)
```

---

### 情况三：唯一索引是前缀索引

例如：

```sql
UNIQUE KEY uq_uuid (uuid(16))
```

这约束的是 UUID 前 16 个字符唯一，不是完整 UUID 唯一。官方文档也提到，如果对 `UNIQUE` 索引指定前缀长度，那么唯一性约束发生在该前缀范围内。([MySQL开发者专区](https://dev.mysql.com/doc/en/create-index.html?utm_source=chatgpt.com "15.1.18 CREATE INDEX Statement"))

不过从名字 `uq_uuid` 看，通常应该是完整唯一索引，具体还要看建表 SQL。

---

## 3. 怎么确认 UUID 到底有没有重复？

不要看 `Cardinality`，直接查真实数据。

### 3.1 查总数和去重数

```sql
SELECT 
    COUNT(*) AS total_count,
    COUNT(uuid) AS not_null_uuid_count,
    COUNT(DISTINCT uuid) AS distinct_uuid_count
FROM order_table;
```

解释：

```text
total_count           总行数
not_null_uuid_count   uuid 非 NULL 的行数
distinct_uuid_count   uuid 去重后的数量
```

如果：

```text
not_null_uuid_count = distinct_uuid_count
```

说明非空 UUID 没有重复。

---

### 3.2 查具体重复 UUID

```sql
SELECT 
    uuid,
    COUNT(*) AS cnt
FROM order_table
WHERE uuid IS NOT NULL
GROUP BY uuid
HAVING COUNT(*) > 1
ORDER BY cnt DESC
LIMIT 20;
```

如果查不出结果，说明非空 UUID 没重复。

---

### 3.3 查是否存在多个 NULL

```sql
SELECT COUNT(*) AS null_uuid_count
FROM order_table
WHERE uuid IS NULL;
```

如果这个值大于 0，说明表里存在未生成 UUID 的数据。

---

### 3.4 刷新统计信息后再看 Cardinality

```sql
ANALYZE TABLE order_table;

SHOW INDEX FROM order_table;
```

`ANALYZE TABLE` 会更新 key distribution / cardinality 统计信息，MySQL 文档也说明可以通过 `ANALYZE TABLE` 更新这类统计值。([MySQL开发者专区](https://dev.mysql.com/doc/en/analyze-table.html?utm_source=chatgpt.com "15.7.3.1 ANALYZE TABLE Statement"))

但注意：刷新后也仍然是统计估算，不是业务级精确校验。

---

## 4. 实际业务能容忍 UUID 重复吗？

### 结论：关键业务唯一标识，不能容忍。

比如这些字段：

```text
order_id
payment_id
refund_id
transaction_id
request_id
idempotent_token
biz_uuid
```

都不能靠“UUID 理论上几乎不会重复”来裸奔。工程上必须加：

```sql
UNIQUE KEY uq_xxx (xxx)
```

因为生产系统里真正的问题不一定是 UUID 算法碰撞，而是：

```text
代码 bug
重试逻辑错误
批量导入重复
多环境数据合并
手工修数据
消息重复消费
测试数据污染
并发请求重复提交
```

UUID 算法本身碰撞概率极低，但业务系统不能只相信概率，要靠数据库唯一约束兜底。

---

## 5. 生产建议

如果 `uuid` 是业务唯一字段，推荐这样设计：

```sql
CREATE TABLE order_record (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    uuid CHAR(36) NOT NULL COMMENT '业务唯一UUID',
    order_id VARCHAR(64) NOT NULL COMMENT '订单号',
    user_id BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    create_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
    update_time DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
    PRIMARY KEY (id),
    UNIQUE KEY uq_uuid (uuid),
    UNIQUE KEY uq_order_id (order_id),
    KEY idx_user_id_create_time (user_id, create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

Java 插入时也要处理重复键异常：

```java
try {
    orderMapper.insert(order);
} catch (DuplicateKeyException e) {
    // 说明唯一约束冲突：可能是重复请求、幂等重放、数据生成异常
    throw new BizException("订单重复提交或唯一编号冲突", e);
}
```

---

## 6. 面试/工程表达

可以这样说：

> `SHOW INDEX` 里的 `Cardinality` 不是精确去重数量，而是 MySQL 优化器基于统计信息估算出来的索引基数。InnoDB 会通过抽样索引页估算 cardinality，所以 100 万行数据下唯一索引显示 97 万多，并不能说明 UUID 发生了重复。要判断是否重复，应该用 `COUNT(DISTINCT uuid)` 或 `GROUP BY uuid HAVING COUNT(*) > 1` 查真实数据。如果 `uuid` 是 `NOT NULL` 且有完整唯一索引，重复值理论上插不进去；如果字段允许 `NULL`，MySQL 唯一索引允许多个 `NULL`，这点生产设计要特别注意。业务唯一字段不能容忍重复，必须用数据库唯一约束兜底。