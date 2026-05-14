[[02MySQL 建表规范 教学案例：订单表]]


> 范围：面向 Java 后端日常业务开发，覆盖订单、明细、支付、库存场景。  
> 风格：只给**场景 + SQL + 简要解释**，不展开 MySQL 原理。

---

# 1. 表结构

## 1.1 订单主表：`order_info`

```sql
DROP TABLE IF EXISTS `order_info`;

CREATE TABLE `order_info` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `order_no` varchar(32) NOT NULL COMMENT '业务订单号',
  `user_id` bigint unsigned NOT NULL COMMENT '用户ID',
  `user_name` varchar(64) NOT NULL DEFAULT '' COMMENT '下单时用户姓名快照',
  `user_mobile` varchar(20) NOT NULL DEFAULT '' COMMENT '下单时用户手机号快照',

  `total_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单总金额',
  `discount_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单优惠金额',
  `pay_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单应付金额',

  `order_status` tinyint unsigned NOT NULL DEFAULT 0 COMMENT '订单状态：0待支付，1已支付，2已取消，3已完成',
  `is_deleted` tinyint unsigned NOT NULL DEFAULT 0 COMMENT '是否删除：0否，1是',

  `client_ip` varbinary(16) DEFAULT NULL COMMENT '客户端IP，兼容IPv4和IPv6',
  `ext_data` json DEFAULT NULL COMMENT '扩展信息',

  `order_time` datetime NOT NULL COMMENT '下单时间',
  `pay_time` datetime DEFAULT NULL COMMENT '支付时间',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',

  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_order_no` (`order_no`),
  KEY `idx_user_order_time` (`user_id`, `is_deleted`, `order_time`),
  KEY `idx_status_order_time` (`order_status`, `order_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='订单主表';
```

---

## 1.2 订单明细表：`order_item`

```sql
DROP TABLE IF EXISTS `order_item`;

CREATE TABLE `order_item` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `order_no` varchar(32) NOT NULL COMMENT '业务订单号',
  `sku` varchar(64) NOT NULL COMMENT '商品SKU',
  `sku_name` varchar(128) NOT NULL DEFAULT '' COMMENT '商品名称快照',
  `quantity` int unsigned NOT NULL DEFAULT 1 COMMENT '购买数量',
  `unit_price` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '商品单价',
  `discount_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '商品优惠金额',
  `pay_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '商品实付金额',

  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',

  PRIMARY KEY (`id`),
  KEY `idx_order_no` (`order_no`),
  KEY `idx_sku_create_time` (`sku`, `create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='订单明细表';
```

---

## 1.3 支付流水表：`payment_record`

```sql
DROP TABLE IF EXISTS `payment_record`;

CREATE TABLE `payment_record` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `pay_no` varchar(32) NOT NULL COMMENT '支付流水号',
  `order_no` varchar(32) NOT NULL COMMENT '业务订单号',
  `user_id` bigint unsigned NOT NULL COMMENT '用户ID',
  `pay_channel` tinyint unsigned NOT NULL COMMENT '支付渠道：1微信，2支付宝，3银行卡',
  `pay_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '支付金额',
  `pay_status` tinyint unsigned NOT NULL DEFAULT 0 COMMENT '支付状态：0处理中，1成功，2失败',
  `third_trade_no` varchar(64) NOT NULL DEFAULT '' COMMENT '三方支付流水号',
  `notify_data` json DEFAULT NULL COMMENT '支付回调原始数据',

  `pay_time` datetime DEFAULT NULL COMMENT '支付完成时间',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',

  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_pay_no` (`pay_no`),
  UNIQUE KEY `uq_third_trade_no` (`third_trade_no`),
  KEY `idx_order_no` (`order_no`),
  KEY `idx_user_create_time` (`user_id`, `create_time`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='支付流水表';
```

---

## 1.4 商品库存表：`sku_stock`

```sql
DROP TABLE IF EXISTS `sku_stock`;

CREATE TABLE `sku_stock` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
  `sku` varchar(64) NOT NULL COMMENT '商品SKU',
  `stock` int unsigned NOT NULL DEFAULT 0 COMMENT '可用库存',
  `locked_stock` int unsigned NOT NULL DEFAULT 0 COMMENT '锁定库存',
  `version` int unsigned NOT NULL DEFAULT 0 COMMENT '乐观锁版本号',

  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',

  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_sku` (`sku`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci COMMENT='商品库存表';
```

---

# 2. 基础写入

## 2.1 新增订单

```sql
INSERT INTO order_info (
  order_no, user_id, user_name, user_mobile,
  total_amount, discount_amount, pay_amount,
  order_status, client_ip, ext_data, order_time
) VALUES (
  'ORD202605150001', 10001, '张三', '13800000000',
  199.00, 20.00, 179.00,
  0, INET6_ATON('127.0.0.1'),
  JSON_OBJECT('channel', 'wechat', 'device', 'ios'),
  NOW()
);
```

说明：新增数据必须显式写字段名，不依赖字段顺序，不插入自增主键。

---

## 2.2 新增订单明细

```sql
INSERT INTO order_item (
  order_no, sku, sku_name, quantity,
  unit_price, discount_amount, pay_amount
) VALUES (
  'ORD202605150001', 'SKU10001', 'Java实战课程', 1,
  199.00, 20.00, 179.00
);
```

说明：订单主表存订单级信息，订单明细表存商品级信息。

---

## 2.3 批量新增订单明细

```sql
INSERT INTO order_item (
  order_no, sku, sku_name, quantity,
  unit_price, discount_amount, pay_amount
) VALUES
  ('ORD202605150002', 'SKU10001', 'Java实战课程', 1, 199.00, 20.00, 179.00),
  ('ORD202605150002', 'SKU10002', 'MySQL实战课程', 1, 99.00, 0.00, 99.00);
```

说明：批量插入比循环单条插入更高效，但单批数量要控制。

---

## 2.4 幂等创建支付流水

```sql
INSERT INTO payment_record (
  pay_no, order_no, user_id, pay_channel, pay_amount, pay_status
) VALUES (
  'PAY202605150001', 'ORD202605150001', 10001, 1, 179.00, 0
)
ON DUPLICATE KEY UPDATE
  update_time = CURRENT_TIMESTAMP;
```

说明：基于 `uq_pay_no` 唯一索引防止重复创建支付流水。

---

# 3. 基础查询

## 3.1 根据主键查询订单

```sql
SELECT
  id, order_no, user_id, user_name, user_mobile,
  pay_amount, order_status, order_time
FROM order_info
WHERE id = 1;
```

说明：主键查询是最直接的单行查询方式。

---

## 3.2 根据订单号查询订单

```sql
SELECT
  id, order_no, user_id, pay_amount, order_status, pay_time
FROM order_info
WHERE order_no = 'ORD202605150001';
```

说明：命中 `uq_order_no` 唯一索引，适合业务系统按订单号查详情。

---

## 3.3 判断订单是否存在

```sql
SELECT 1
FROM order_info
WHERE order_no = 'ORD202605150001'
LIMIT 1;
```

说明：存在性判断不需要查整行数据。

---

## 3.4 查询用户订单列表

```sql
SELECT
  order_no, pay_amount, order_status, order_time
FROM order_info
WHERE user_id = 10001
  AND is_deleted = 0
ORDER BY order_time DESC
LIMIT 20;
```

说明：典型 C 端订单列表查询，匹配 `idx_user_order_time`。

---

## 3.5 后台按状态查询订单

```sql
SELECT
  order_no, user_id, pay_amount, order_status, order_time
FROM order_info
WHERE order_status = 1
  AND order_time >= '2026-05-01 00:00:00'
  AND order_time < '2026-06-01 00:00:00'
ORDER BY order_time DESC
LIMIT 50;
```

说明：适合后台运营、客服、财务按状态和时间筛选订单。

---

# 4. 关联查询

## 4.1 查询订单详情和明细

```sql
SELECT
  o.order_no,
  o.user_id,
  o.user_name,
  o.pay_amount AS order_pay_amount,
  o.order_status,
  i.sku,
  i.sku_name,
  i.quantity,
  i.unit_price,
  i.pay_amount AS item_pay_amount
FROM order_info o
JOIN order_item i ON o.order_no = i.order_no
WHERE o.order_no = 'ORD202605150001';
```

说明：订单主表查订单级信息，明细表查商品级信息。

---

## 4.2 查询用户订单及支付信息

```sql
SELECT
  o.order_no,
  o.user_id,
  o.pay_amount,
  o.order_status,
  p.pay_no,
  p.pay_channel,
  p.pay_status,
  p.pay_time
FROM order_info o
LEFT JOIN payment_record p ON o.order_no = p.order_no
WHERE o.user_id = 10001
  AND o.is_deleted = 0
ORDER BY o.order_time DESC
LIMIT 20;
```

说明：`LEFT JOIN` 保留没有支付流水的订单。

---

## 4.3 查询有支付成功记录的订单

```sql
SELECT
  o.order_no,
  o.user_id,
  o.pay_amount,
  o.order_status
FROM order_info o
WHERE EXISTS (
  SELECT 1
  FROM payment_record p
  WHERE p.order_no = o.order_no
    AND p.pay_status = 1
);
```

说明：`EXISTS` 适合判断关联表中是否存在满足条件的数据。

---

# 5. 分页查询

## 5.1 普通分页

```sql
SELECT
  order_no, user_id, pay_amount, order_status, order_time
FROM order_info
WHERE order_status = 1
ORDER BY order_time DESC
LIMIT 0, 20;
```

说明：适合浅分页，例如第 1 页、第 2 页。

---

## 5.2 深分页优化：基于主键游标

```sql
SELECT
  id, order_no, user_id, pay_amount, order_status
FROM order_info
WHERE id > 100000
ORDER BY id ASC
LIMIT 20;
```

说明：避免大 offset 扫描，适合后台批处理、数据同步。

---

## 5.3 用户订单游标分页

```sql
SELECT
  id, order_no, pay_amount, order_status, order_time
FROM order_info
WHERE user_id = 10001
  AND is_deleted = 0
  AND (
    order_time < '2026-05-15 12:00:00'
    OR (order_time = '2026-05-15 12:00:00' AND id < 5000)
  )
ORDER BY order_time DESC, id DESC
LIMIT 20;
```

说明：适合 App 订单列表无限下拉，`order_time + id` 保证排序稳定。

---

# 6. 更新与状态流转

## 6.1 修改订单状态

```sql
UPDATE order_info
SET order_status = 2
WHERE order_no = 'ORD202605150001'
  AND user_id = 10001;
```

说明：用户侧操作建议带 `user_id`，防止越权更新。

---

## 6.2 支付成功更新订单

```sql
UPDATE order_info
SET
  order_status = 1,
  pay_time = NOW()
WHERE order_no = 'ORD202605150001'
  AND order_status = 0;
```

说明：状态流转必须带旧状态条件，防止重复支付回调反复更新。

---

## 6.3 更新支付流水为成功

```sql
UPDATE payment_record
SET
  pay_status = 1,
  third_trade_no = 'WX202605150001',
  pay_time = NOW(),
  notify_data = JSON_OBJECT('trade_state', 'SUCCESS', 'channel', 'wechat')
WHERE pay_no = 'PAY202605150001'
  AND pay_status = 0;
```

说明：支付回调更新也要带旧状态条件，实现幂等处理。

---

## 6.4 批量关闭超时未支付订单

```sql
UPDATE order_info
SET order_status = 2
WHERE order_status = 0
  AND order_time < DATE_SUB(NOW(), INTERVAL 30 MINUTE)
LIMIT 500;
```

说明：大批量更新要分批执行，避免一次影响过多数据。

---

# 7. 删除与清理

## 7.1 逻辑删除订单

```sql
UPDATE order_info
SET is_deleted = 1
WHERE order_no = 'ORD202605150001'
  AND user_id = 10001
  AND is_deleted = 0;
```

说明：业务系统优先逻辑删除，不直接物理删除核心数据。

---

## 7.2 删除测试订单明细

```sql
DELETE FROM order_item
WHERE order_no = 'ORD202605150001';
```

说明：物理删除必须带明确条件，生产核心表慎用。

---

## 7.3 清空测试表

```sql
TRUNCATE TABLE order_item;
```

说明：只适合测试环境。生产业务表不要随意使用 `TRUNCATE`。

---

# 8. 聚合统计

## 8.1 统计用户订单数

```sql
SELECT
  COUNT(*) AS order_count
FROM order_info
WHERE user_id = 10001
  AND is_deleted = 0;
```

说明：统计行数使用 `COUNT(*)`。

---

## 8.2 统计用户消费金额

```sql
SELECT
  COALESCE(SUM(pay_amount), 0.00) AS total_pay_amount
FROM order_info
WHERE user_id = 10001
  AND order_status IN (1, 3)
  AND is_deleted = 0;
```

说明：`SUM()` 没有匹配数据时可能返回 `NULL`，用 `COALESCE` 兜底。

---

## 8.3 按订单状态统计数量

```sql
SELECT
  order_status,
  COUNT(*) AS order_count
FROM order_info
WHERE order_time >= '2026-05-01 00:00:00'
  AND order_time < '2026-06-01 00:00:00'
GROUP BY order_status;
```

说明：适合后台看板统计不同状态订单数量。

---

## 8.4 按 SKU 统计销量

```sql
SELECT
  sku,
  SUM(quantity) AS total_quantity,
  SUM(pay_amount) AS total_pay_amount
FROM order_item
WHERE create_time >= '2026-05-01 00:00:00'
  AND create_time < '2026-06-01 00:00:00'
GROUP BY sku;
```

说明：商品维度统计应从订单明细表统计。

---

## 8.5 查询销售额大于 10000 的 SKU

```sql
SELECT
  sku,
  SUM(pay_amount) AS total_pay_amount
FROM order_item
WHERE create_time >= '2026-05-01 00:00:00'
  AND create_time < '2026-06-01 00:00:00'
GROUP BY sku
HAVING SUM(pay_amount) > 10000.00;
```

说明：`WHERE` 过滤明细行，`HAVING` 过滤聚合结果。

---

# 9. 库存扣减

## 9.1 查询 SKU 库存

```sql
SELECT
  sku, stock, locked_stock
FROM sku_stock
WHERE sku = 'SKU10001';
```

说明：命中 `uq_sku` 唯一索引。

---

## 9.2 下单锁定库存

```sql
UPDATE sku_stock
SET
  stock = stock - 1,
  locked_stock = locked_stock + 1
WHERE sku = 'SKU10001'
  AND stock >= 1;
```

说明：通过 `stock >= 1` 防止库存扣成负数；业务代码根据影响行数判断是否扣减成功。

---

## 9.3 支付成功扣减锁定库存

```sql
UPDATE sku_stock
SET locked_stock = locked_stock - 1
WHERE sku = 'SKU10001'
  AND locked_stock >= 1;
```

说明：支付成功后释放锁定库存，不再加回可用库存。

---

## 9.4 取消订单释放库存

```sql
UPDATE sku_stock
SET
  stock = stock + 1,
  locked_stock = locked_stock - 1
WHERE sku = 'SKU10001'
  AND locked_stock >= 1;
```

说明：取消订单时把锁定库存还原为可用库存。

---

## 9.5 乐观锁更新库存

```sql
UPDATE sku_stock
SET
  stock = stock - 1,
  version = version + 1
WHERE sku = 'SKU10001'
  AND stock >= 1
  AND version = 10;
```

说明：带 `version` 条件，适合需要显式并发控制的库存更新场景。

---

# 10. 事务场景

## 10.1 创建订单事务

```sql
START TRANSACTION;

INSERT INTO order_info (
  order_no, user_id, user_name, user_mobile,
  total_amount, discount_amount, pay_amount,
  order_status, order_time
) VALUES (
  'ORD202605150003', 10001, '张三', '13800000000',
  199.00, 20.00, 179.00,
  0, NOW()
);

INSERT INTO order_item (
  order_no, sku, sku_name, quantity,
  unit_price, discount_amount, pay_amount
) VALUES (
  'ORD202605150003', 'SKU10001', 'Java实战课程', 1,
  199.00, 20.00, 179.00
);

UPDATE sku_stock
SET
  stock = stock - 1,
  locked_stock = locked_stock + 1
WHERE sku = 'SKU10001'
  AND stock >= 1;

COMMIT;
```

说明：创建订单、写入明细、锁定库存应该保证事务一致性。

---

## 10.2 支付成功事务

```sql
START TRANSACTION;

UPDATE payment_record
SET
  pay_status = 1,
  third_trade_no = 'WX202605150003',
  pay_time = NOW()
WHERE pay_no = 'PAY202605150003'
  AND pay_status = 0;

UPDATE order_info
SET
  order_status = 1,
  pay_time = NOW()
WHERE order_no = 'ORD202605150003'
  AND order_status = 0;

UPDATE sku_stock
SET locked_stock = locked_stock - 1
WHERE sku = 'SKU10001'
  AND locked_stock >= 1;

COMMIT;
```

说明：支付流水、订单状态、库存确认需要在一个事务中处理；业务代码要检查每条 SQL 的影响行数。

---

## 10.3 悲观锁查询订单

```sql
START TRANSACTION;

SELECT
  id, order_no, order_status, pay_amount
FROM order_info
WHERE order_no = 'ORD202605150003'
FOR UPDATE;

UPDATE order_info
SET order_status = 1, pay_time = NOW()
WHERE order_no = 'ORD202605150003'
  AND order_status = 0;

COMMIT;
```

说明：`FOR UPDATE` 必须放在事务中，用于锁定待修改订单。

---

# 11. JSON 字段

## 11.1 查询 JSON 字段

```sql
SELECT
  order_no,
  ext_data ->> '$.channel' AS channel,
  ext_data ->> '$.device' AS device
FROM order_info
WHERE order_no = 'ORD202605150001';
```

说明：`->>` 返回字符串值，适合直接展示或条件判断。

---

## 11.2 根据 JSON 条件查询

```sql
SELECT
  order_no, user_id, pay_amount
FROM order_info
WHERE ext_data ->> '$.channel' = 'wechat'
  AND order_time >= '2026-05-01 00:00:00'
  AND order_time < '2026-06-01 00:00:00';
```

说明：低频查询可以这样写；高频查询字段建议单独建列。

---

## 11.3 更新 JSON 字段

```sql
UPDATE order_info
SET ext_data = JSON_SET(ext_data, '$.campaignId', 'ACT202605')
WHERE order_no = 'ORD202605150001';
```

说明：`JSON_SET` 可以更新或新增 JSON 中的某个 key。

---

## 11.4 删除 JSON 字段

```sql
UPDATE order_info
SET ext_data = JSON_REMOVE(ext_data, '$.campaignId')
WHERE order_no = 'ORD202605150001';
```

说明：删除扩展字段中的某个属性，不影响其他 JSON 内容。

---

# 12. IP 字段

## 12.1 插入 IPv4

```sql
INSERT INTO order_info (
  order_no, user_id, user_name, user_mobile,
  total_amount, discount_amount, pay_amount,
  order_status, client_ip, order_time
) VALUES (
  'ORD202605150004', 10002, '李四', '13900000000',
  99.00, 0.00, 99.00,
  0, INET6_ATON('192.168.1.10'), NOW()
);
```

说明：`INET6_ATON()` 可以处理 IPv4 和 IPv6。

---

## 12.2 查询 IP 字符串

```sql
SELECT
  order_no,
  INET6_NTOA(client_ip) AS client_ip
FROM order_info
WHERE order_no = 'ORD202605150004';
```

说明：数据库存二进制，展示时转成字符串。

---

# 13. 模糊查询

## 13.1 手机号前缀查询

```sql
SELECT
  order_no, user_id, user_mobile, order_time
FROM order_info
WHERE user_mobile LIKE '138%'
ORDER BY order_time DESC
LIMIT 20;
```

说明：前缀匹配有机会利用普通索引；如果高频查询，应给 `user_mobile` 加索引。

---

## 13.2 用户名模糊查询

```sql
SELECT
  order_no, user_id, user_name, order_time
FROM order_info
WHERE user_name LIKE CONCAT('%', '张', '%')
ORDER BY order_time DESC
LIMIT 20;
```

说明：前后模糊匹配通常不适合大表高频查询，复杂搜索应考虑搜索引擎。

---

# 14. 索引与执行计划

## 14.1 查看执行计划

```sql
EXPLAIN
SELECT
  order_no, pay_amount, order_status, order_time
FROM order_info
WHERE user_id = 10001
  AND is_deleted = 0
ORDER BY order_time DESC
LIMIT 20;
```

说明：用 `EXPLAIN` 检查是否命中预期索引。

---

## 14.2 避免对索引字段使用函数

```sql
-- 不推荐
SELECT
  order_no, pay_amount, order_time
FROM order_info
WHERE DATE(order_time) = '2026-05-15';

-- 推荐
SELECT
  order_no, pay_amount, order_time
FROM order_info
WHERE order_time >= '2026-05-15 00:00:00'
  AND order_time < '2026-05-16 00:00:00';
```

说明：时间条件推荐使用左闭右开范围查询。

---

## 14.3 避免隐式类型转换

```sql
-- 不推荐
SELECT
  order_no, user_id, pay_amount
FROM order_info
WHERE order_no = 202605150001;

-- 推荐
SELECT
  order_no, user_id, pay_amount
FROM order_info
WHERE order_no = '202605150001';
```

说明：字符串字段要传字符串参数，避免隐式转换影响索引使用。

---

## 14.4 覆盖索引查询

```sql
SELECT
  user_id, is_deleted, order_time
FROM order_info
WHERE user_id = 10001
  AND is_deleted = 0
ORDER BY order_time DESC
LIMIT 20;
```

说明：查询字段都在 `idx_user_order_time` 中，可以减少回表。

---

# 15. 安全 SQL 规范

## 15.1 更新前先确认影响范围

```sql
SELECT
  COUNT(*) AS affected_count
FROM order_info
WHERE order_status = 0
  AND order_time < DATE_SUB(NOW(), INTERVAL 30 MINUTE);
```

说明：批量更新或删除前，先用相同条件查询影响范围。

---

## 15.2 禁止无条件更新

```sql
-- 危险 SQL，不要执行
UPDATE order_info
SET order_status = 2;
```

说明：生产更新必须带明确 `WHERE` 条件。

---

## 15.3 禁止无条件删除

```sql
-- 危险 SQL，不要执行
DELETE FROM order_info;
```

说明：生产核心表禁止无条件删除。

---

## 15.4 大批量删除分批执行

```sql
DELETE FROM payment_record
WHERE pay_status = 2
  AND create_time < '2026-01-01 00:00:00'
LIMIT 500;
```

说明：大批量删除应分批执行，避免长事务和大范围锁影响线上业务。

---

# 16. 常用 SQL 模板总结

## 单行查询

```sql
SELECT
  id, order_no, user_id, pay_amount, order_status
FROM order_info
WHERE order_no = ?;
```

## 列表查询

```sql
SELECT
  order_no, pay_amount, order_status, order_time
FROM order_info
WHERE user_id = ?
  AND is_deleted = 0
ORDER BY order_time DESC
LIMIT ?;
```

## 条件更新

```sql
UPDATE order_info
SET order_status = ?
WHERE order_no = ?
  AND order_status = ?;
```

## 防超卖扣库存

```sql
UPDATE sku_stock
SET stock = stock - ?
WHERE sku = ?
  AND stock >= ?;
```

## 聚合统计

```sql
SELECT
  order_status,
  COUNT(*) AS order_count,
  SUM(pay_amount) AS total_pay_amount
FROM order_info
WHERE order_time >= ?
  AND order_time < ?
GROUP BY order_status;
```

## 事务模板

```sql
START TRANSACTION;

-- SQL 1
-- SQL 2
-- SQL 3

COMMIT;
```

---

# 17. 最后记忆版

这份手册覆盖的是 Java 后端最常用的 MySQL SQL 场景：

> **新增、批量新增、幂等写入、主键查询、唯一键查询、列表查询、分页、关联查询、状态流转、逻辑删除、聚合统计、库存扣减、支付事务、JSON、IP、EXPLAIN、索引失效、安全更新。**

核心原则：

> **SQL 不要脱离业务场景写；索引不要脱离查询条件建；更新不要脱离状态机做；删除不要脱离安全边界执行。**