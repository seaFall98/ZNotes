```
减少使用or语句 or有可能被 mysq l优化为支持索引，但也要损耗 mysql 的 cpu 性能。可将or语句优化为union，然后在各个where条件上建立索引。
```

## 结论先行

这句话**方向是对的，但表述不够严谨**。

更准确的说法应该是：

> `OR` 不一定会导致索引失效。MySQL 优化器在某些情况下可以对 `OR` 条件使用索引，例如 `index_merge`、多个索引扫描后合并结果等。  
> 但 `OR` 会增加优化器判断成本和执行成本，尤其当多个条件分布在不同字段、不同索引、选择性较差、数据量较大时，可能导致扫描范围变大、回表次数增加、CPU 消耗升高。  
> 在某些场景下，可以将复杂 `OR` 拆成多个 `SELECT ... WHERE ...`，再用 `UNION` 或 `UNION ALL` 合并结果，使每个子查询都能稳定命中合适索引，从而提升可控性和执行效率。

核心不是“看到 `OR` 就改成 `UNION`”，而是：

> **当 `OR` 让 MySQL 无法很好地利用索引，或者执行计划不可控时，才考虑拆成 `UNION`。**

---

# 1. `OR` 为什么可能有问题？

假设有一张订单表：

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL,
    order_no VARCHAR(64) NOT NULL,
    status TINYINT NOT NULL,
    create_time DATETIME NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    KEY idx_user_id (user_id),
    KEY idx_order_no (order_no),
    KEY idx_status (status),
    KEY idx_create_time (create_time)
);
```

现在有一个查询：

```sql
SELECT *
FROM orders
WHERE user_id = 10001
   OR order_no = 'O202605130001';
```

从业务角度看很自然：

> 查这个用户的订单，或者查这个订单号对应的订单。

但从 MySQL 执行角度看，这个查询有两个条件：

```sql
user_id = 10001
order_no = 'O202605130001'
```

它们分别对应两个不同索引：

```sql
idx_user_id
idx_order_no
```

MySQL 可能有几种选择：

1. 走 `idx_user_id`，再过滤 `order_no`
    
2. 走 `idx_order_no`，再过滤 `user_id`
    
3. 走 `index_merge`，分别扫描两个索引，再合并结果
    
4. 干脆全表扫描
    

问题就在这里：

> `OR` 会让优化器需要在多个执行策略之间做选择，执行计划更复杂，也更容易出现不稳定。

---

# 2. `OR` 一定会导致索引失效吗？

不会。

这是一个常见误区。

例如：

```sql
SELECT *
FROM orders
WHERE user_id = 10001
   OR user_id = 10002;
```

这个查询只涉及同一个字段 `user_id`。

MySQL 很可能可以正常使用 `idx_user_id`。

甚至这个 SQL 更推荐写成：

```sql
SELECT *
FROM orders
WHERE user_id IN (10001, 10002);
```

这类查询通常没必要改成 `UNION`。

---

再看这个：

```sql
SELECT *
FROM orders
WHERE order_no = 'O202605130001'
   OR order_no = 'O202605130002';
```

也可以改成：

```sql
SELECT *
FROM orders
WHERE order_no IN ('O202605130001', 'O202605130002');
```

这仍然可以很好利用 `idx_order_no`。

所以不能简单说：

> `OR` 会导致索引失效。

更准确是：

> **跨字段、跨索引、低选择性、大范围条件混合的 `OR`，更容易导致索引使用效果变差。**

---

# 3. 真正危险的是这种 `OR`

## 场景一：不同字段上的 `OR`

```sql
SELECT *
FROM orders
WHERE user_id = 10001
   OR status = 1;
```

这里的问题是：

```sql
user_id = 10001
```

选择性可能很高，只查一个用户。

但：

```sql
status = 1
```

选择性可能很低，比如 80% 的订单都是 `status = 1`。

这时就算 `status` 上有索引，MySQL 也可能认为：

> 走 `idx_status` 还不如全表扫描。

因为它要查出大量数据，还要回表。

这种情况下，`OR` 把一个高选择性的条件和一个低选择性的条件绑在了一起，执行计划就容易变差。

---

## 场景二：一个条件有索引，另一个条件没索引

```sql
SELECT *
FROM orders
WHERE order_no = 'O202605130001'
   OR amount > 1000;
```

假设只有 `order_no` 有索引，`amount` 没有索引。

那么：

```sql
order_no = 'O202605130001'
```

本来可以很快查到一条记录。

但：

```sql
amount > 1000
```

没有索引，可能需要扫描大量数据。

两个条件被 `OR` 连在一起之后，MySQL 可能无法只靠 `order_no` 索引解决问题，最终执行计划可能退化。

---

## 场景三：范围条件和等值条件混在一起

```sql
SELECT *
FROM orders
WHERE order_no = 'O202605130001'
   OR create_time >= '2026-05-01';
```

`order_no` 是高选择性等值查询。

`create_time` 是范围查询。

如果 `create_time >= '2026-05-01'` 命中大量数据，那么这个 `OR` 查询整体成本仍然可能很高。

---

# 4. 为什么 `UNION` 可能更好？

把原来的 SQL：

```sql
SELECT *
FROM orders
WHERE user_id = 10001
   OR order_no = 'O202605130001';
```

改成：

```sql
SELECT *
FROM orders
WHERE user_id = 10001

UNION

SELECT *
FROM orders
WHERE order_no = 'O202605130001';
```

这样做的好处是：

## 每个子查询可以独立选择最优索引

第一个子查询：

```sql
SELECT *
FROM orders
WHERE user_id = 10001;
```

可以稳定走：

```sql
idx_user_id
```

第二个子查询：

```sql
SELECT *
FROM orders
WHERE order_no = 'O202605130001';
```

可以稳定走：

```sql
idx_order_no
```

这样 MySQL 不需要在一个复杂的 `OR` 条件里做折中判断，而是让每个查询各走各的索引。

本质是：

> 用多个简单查询，替代一个复杂查询。

---

# 5. `UNION` 和 `UNION ALL` 怎么选？

这是关键。

## `UNION` 会去重

```sql
SELECT *
FROM orders
WHERE user_id = 10001

UNION

SELECT *
FROM orders
WHERE order_no = 'O202605130001';
```

`UNION` 默认会去重。

如果同一条订单同时满足：

```sql
user_id = 10001
order_no = 'O202605130001'
```

那么结果只返回一条。

但是去重有成本。

MySQL 需要做临时结果集、排序或哈希去重，数据量大时开销不小。

---

## `UNION ALL` 不去重，性能通常更好

```sql
SELECT *
FROM orders
WHERE user_id = 10001

UNION ALL

SELECT *
FROM orders
WHERE order_no = 'O202605130001';
```

`UNION ALL` 直接合并结果，不去重。

性能通常比 `UNION` 好。

但是如果两边可能查出同一条数据，就会重复返回。

---

# 6. 如何避免 `UNION ALL` 重复数据？

假设原始语义是：

```sql
WHERE user_id = 10001
   OR order_no = 'O202605130001'
```

如果改成 `UNION ALL`，为了避免重复，可以这样写：

```sql
SELECT *
FROM orders
WHERE user_id = 10001

UNION ALL

SELECT *
FROM orders
WHERE order_no = 'O202605130001'
  AND user_id <> 10001;
```

第二个查询主动排除已经被第一个查询覆盖的数据。

这样既避免重复，又避免 `UNION` 的去重成本。

这在高性能 SQL 优化里很常见。

---

# 7. 对比一下执行逻辑

## 原始 `OR`

```sql
SELECT *
FROM orders
WHERE user_id = 10001
   OR order_no = 'O202605130001';
```

可能的执行逻辑：

```text
MySQL 优化器判断：
- 是否走 idx_user_id？
- 是否走 idx_order_no？
- 是否用 index_merge？
- 是否全表扫描？
- 两个条件怎么合并？
- 结果怎么过滤？
```

执行计划可能不稳定。

---

## 改成 `UNION ALL`

```sql
SELECT *
FROM orders
WHERE user_id = 10001

UNION ALL

SELECT *
FROM orders
WHERE order_no = 'O202605130001'
  AND user_id <> 10001;
```

执行逻辑更清楚：

```text
子查询 1：
- 走 idx_user_id
- 查 user_id = 10001 的订单

子查询 2：
- 走 idx_order_no
- 查 order_no = xxx 的订单
- 排除 user_id = 10001 的重复记录

最后直接合并结果。
```

可控性更强。

---

# 8. 但不是所有 `OR` 都应该改成 `UNION`

这是面试和生产实践里最需要强调的点。

## 不需要改的情况

### 1. 同一个字段多个等值条件

```sql
WHERE status = 1 OR status = 2
```

改成：

```sql
WHERE status IN (1, 2)
```

更自然。

---

### 2. 同一个索引前缀上的条件

例如有联合索引：

```sql
KEY idx_user_status (user_id, status)
```

查询：

```sql
SELECT *
FROM orders
WHERE user_id = 10001
  AND (status = 1 OR status = 2);
```

可以写成：

```sql
SELECT *
FROM orders
WHERE user_id = 10001
  AND status IN (1, 2);
```

不必用 `UNION`。

---

### 3. 数据量很小

如果表本身只有几千条，或者查询频率很低，改写意义不大。

过度优化只会增加 SQL 复杂度。

---

### 4. `OR` 条件本身已经被优化器处理得很好

MySQL 可能已经通过 `index_merge` 或范围扫描解决得不错。

这时强行改成 `UNION` 不一定更快。

最终还是要看：

```sql
EXPLAIN
EXPLAIN ANALYZE
```

---

# 9. `OR` 优化成 `UNION` 的典型适用场景

比较适合改写的情况：

|场景|是否适合改成 UNION|
|---|--:|
|不同字段都有高选择性索引|适合|
|一个条件走 `idx_a`，另一个条件走 `idx_b`|适合|
|`OR` 导致全表扫描|很适合|
|`OR` 导致 `index_merge` 成本很高|适合|
|条件之间数据量差异很大|可以考虑|
|同字段多个等值条件|不建议，优先 `IN`|
|表数据很小|不建议|
|查询不在核心链路|不建议过度优化|

---

# 10. 一个更完整的例子

## 原始 SQL

```sql
SELECT id, user_id, order_no, status, create_time
FROM orders
WHERE user_id = 10001
   OR order_no = 'O202605130001';
```

可能的问题：

```text
1. user_id 和 order_no 是两个不同索引
2. MySQL 可能走 index_merge
3. 如果数据量大，合并索引结果、回表、过滤都有成本
4. 执行计划可能随着数据分布变化而变化
```

---

## 优化版本一：使用 UNION

```sql
SELECT id, user_id, order_no, status, create_time
FROM orders
WHERE user_id = 10001

UNION

SELECT id, user_id, order_no, status, create_time
FROM orders
WHERE order_no = 'O202605130001';
```

特点：

```text
优点：自动去重，语义安全
缺点：去重有额外成本
```

---

## 优化版本二：使用 UNION ALL + 手动排重

```sql
SELECT id, user_id, order_no, status, create_time
FROM orders
WHERE user_id = 10001

UNION ALL

SELECT id, user_id, order_no, status, create_time
FROM orders
WHERE order_no = 'O202605130001'
  AND user_id <> 10001;
```

特点：

```text
优点：避免 UNION 去重成本
缺点：SQL 稍微复杂，需要保证排重逻辑正确
```

---

# 11. 索引应该怎么建？

如果你改成了：

```sql
SELECT id, user_id, order_no, status, create_time
FROM orders
WHERE user_id = 10001

UNION ALL

SELECT id, user_id, order_no, status, create_time
FROM orders
WHERE order_no = 'O202605130001'
  AND user_id <> 10001;
```

那么至少需要：

```sql
KEY idx_user_id (user_id),
KEY idx_order_no (order_no)
```

如果想减少回表，可以考虑覆盖索引：

```sql
KEY idx_user_query (user_id, id, order_no, status, create_time),
KEY idx_order_no_query (order_no, id, user_id, status, create_time)
```

但是这要看查询频率和字段数量。

不要为了一个低频查询建立过宽索引。

生产中更推荐先做：

```sql
EXPLAIN ANALYZE
```

观察实际扫描行数、耗时、是否回表，再决定是否加覆盖索引。

---

# 12. 面试时怎么回答最稳？

可以这样说：

> `OR` 不一定会导致索引失效。MySQL 优化器在一些场景下可以使用索引，比如同字段的 `OR` 可以等价优化成范围查询，跨字段的 `OR` 也可能使用 `index_merge`。  
> 但是 `OR` 会让执行计划更复杂，尤其是不同字段、不同索引、选择性差异大的条件混在一起时，可能导致全表扫描、索引合并成本高、回表次数多、CPU 消耗增加。  
> 如果发现 `OR` 查询执行计划不好，可以把它拆成多个子查询，用 `UNION` 或 `UNION ALL` 合并结果，让每个子查询独立命中最合适的索引。  
> 如果能保证结果不重复，优先用 `UNION ALL`；如果可能重复，可以用 `UNION` 去重，或者用 `UNION ALL` 加排重条件手动避免重复。  
> 但这不是固定规则，最终要通过 `EXPLAIN` 或 `EXPLAIN ANALYZE` 验证。

---

# 13. 最终压缩版

原来的说法可以优化成这版：

> `OR` 不一定会让索引失效，MySQL 在部分场景下可以通过索引范围扫描或 `index_merge` 使用索引。但复杂 `OR` 会增加优化器选择成本和执行成本，尤其是多个条件分布在不同字段、不同索引、选择性差异较大时，可能导致扫描行数变多、回表增多、CPU 消耗上升，甚至退化成全表扫描。  
> 对于这种情况，可以将 `OR` 拆成多个独立查询，用 `UNION` 或 `UNION ALL` 合并，让每个 `WHERE` 条件都稳定命中自己的索引。若结果可能重复，用 `UNION` 去重；若能手动排除重复，优先使用 `UNION ALL`，性能更好。最终是否优化，应以 `EXPLAIN / EXPLAIN ANALYZE` 的执行计划和实际耗时为准。

---

## 关键词

`OR`、`UNION`、`UNION ALL`、`index_merge`、索引选择性、回表、全表扫描、执行计划、覆盖索引、EXPLAIN ANALYZE。