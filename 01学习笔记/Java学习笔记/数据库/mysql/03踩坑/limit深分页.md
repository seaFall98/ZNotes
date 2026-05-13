```
观点：分页查询，当limit起点较高时，可先用过滤条件进行过滤。如 `select a,b,c from t1 limit 10000,20`; 优化为 `select a,b,c from t1 where id>10000 limit 20`;
```

---
## 结论：不完全正确

这个说法**方向是对的**，但太模糊。

分页查询里真正的问题不是 `LIMIT` 本身，而是：

> 当 `LIMIT offset, size` 的 `offset` 很大时，MySQL 仍然需要先扫描/排序/跳过前面大量数据，再返回后面的 `size` 条记录。

所以更准确的说法是：

> 深分页时，应避免直接使用大的 `LIMIT offset, size`。可以通过索引条件、上一页游标、主键范围过滤、延迟关联等方式，先缩小扫描范围，再取目标页数据。

---

# 1. 问题 SQL：深分页

假设订单表：

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    status TINYINT NOT NULL,
    create_time DATETIME NOT NULL,
    amount DECIMAL(10,2) NOT NULL,

    KEY idx_create_time (create_time),
    KEY idx_user_time (user_id, create_time, id)
);
```

常见分页：

```sql
SELECT *
FROM orders
ORDER BY create_time DESC
LIMIT 100000, 20;
```

这条 SQL 的问题是：

```text
MySQL 不是直接跳到第 100001 条。
它通常要先找到前 100020 条记录，
然后丢弃前 100000 条，
最后返回 20 条。
```

所以 `offset` 越大，扫描和丢弃的数据越多。

---

# 2. 正确优化方式一：用“上一页最后一条记录”做过滤

假设上一页最后一条记录是：

```text
create_time = '2026-05-01 10:00:00'
id = 888888
```

不要这样查：

```sql
SELECT *
FROM orders
ORDER BY create_time DESC, id DESC
LIMIT 100000, 20;
```

改成游标分页：

```sql
SELECT *
FROM orders
WHERE (create_time < '2026-05-01 10:00:00')
   OR (create_time = '2026-05-01 10:00:00' AND id < 888888)
ORDER BY create_time DESC, id DESC
LIMIT 20;
```

配套索引：

```sql
KEY idx_time_id (create_time, id)
```

这样 MySQL 可以从上一页结束位置继续往后扫，而不是重新跳过前 10 万条。

面试点评：

> 深分页最常见的优化方案是“游标分页”或“基于上一次最大/最小 ID 的范围查询”。

---

# 3. 正确优化方式二：按主键范围过滤

如果分页按 `id` 递增：

```sql
SELECT *
FROM orders
ORDER BY id ASC
LIMIT 100000, 20;
```

可以改成：

```sql
SELECT *
FROM orders
WHERE id > 100000
ORDER BY id ASC
LIMIT 20;
```

前提是：

```text
id 大体连续，并且业务可以接受按 id 翻页。
```

这类写法能直接利用主键索引：

```sql
PRIMARY KEY (id)
```

但注意：

> `id > 100000` 不等价于第 100001 条记录。  
> 如果中间有删除、过滤条件、数据不连续，结果会有偏差。

所以它更适合“加载更多”“滚动加载”，不适合严格跳页。

---

# 4. 正确优化方式三：延迟关联

深分页如果必须支持跳到第 N 页，可以先只查主键，再回表查完整数据。

低效写法：

```sql
SELECT *
FROM orders
ORDER BY create_time DESC
LIMIT 100000, 20;
```

优化写法：

```sql
SELECT o.*
FROM orders o
JOIN (
    SELECT id
    FROM orders
    ORDER BY create_time DESC
    LIMIT 100000, 20
) t ON o.id = t.id;
```

如果有索引：

```sql
KEY idx_create_time_id (create_time, id)
```

子查询可以尽量走覆盖索引：

```sql
SELECT id
FROM orders
ORDER BY create_time DESC
LIMIT 100000, 20;
```

这样比直接 `SELECT *` 好，因为前 100020 条扫描过程中不需要反复回表读取完整行。

面试点评：

> 延迟关联的核心是：先用覆盖索引拿到目标页主键，再根据主键回表查完整数据。

---

# 5. “先用过滤条件进行过滤”什么时候对？

比如你原来查：

```sql
SELECT *
FROM orders
ORDER BY create_time DESC
LIMIT 100000, 20;
```

如果业务上可以增加条件：

```sql
SELECT *
FROM orders
WHERE status = 1
  AND create_time >= '2026-01-01'
ORDER BY create_time DESC
LIMIT 1000, 20;
```

这确实可能更快。

但关键不是“随便加过滤条件”，而是：

```text
过滤条件必须有业务意义；
过滤条件最好能命中索引；
过滤后能显著减少候选数据量。
```

对应索引可以是：

```sql
KEY idx_status_time (status, create_time, id)
```

否则：

```sql
WHERE status = 1
```

如果 `status = 1` 占全表 90%，过滤效果很差，优化意义有限。

---

# 6. 面试中容易被追问的点

|追问|要点|
|---|---|
|为什么深分页慢？|MySQL 要扫描并丢弃 offset 前面的数据|
|`LIMIT 100000, 20` 是不是直接定位？|不是，一般不是直接跳到第 100001 条|
|怎么优化？|游标分页、主键范围、延迟关联、增加有效过滤条件|
|`id > xxx` 一定等价于深分页吗？|不等价，只适合特定排序和业务场景|
|延迟关联为什么快？|先走覆盖索引查 id，减少大量回表|
|加过滤条件一定有用吗？|不一定，取决于选择性和索引|

---

# 7. 更准确的表述

你的原话可以改成：

> 分页查询在 `LIMIT offset, size` 的 `offset` 很大时，会产生深分页问题。MySQL 通常需要先扫描并跳过大量记录，再返回目标数据，导致性能下降。优化时可以通过业务过滤条件、游标分页、主键范围查询或延迟关联等方式缩小扫描范围。过滤条件必须具备较高选择性，并配合合适索引，否则效果有限。