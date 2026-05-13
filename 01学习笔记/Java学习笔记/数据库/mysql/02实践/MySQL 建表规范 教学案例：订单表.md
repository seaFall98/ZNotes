[[01mysql开发规范和使用约束]]
[xfg](https://bugstack.cn/md/road-map/mysql.html)
# MySQL 建表规范满分教学案例：订单表

## 1. 教学目标

用一张**简化版订单表**讲清楚 Java 后端必须掌握的 MySQL 建表规范：

- 主键为什么用自增 `BIGINT`
    
- 手机号、订单号、金额、状态字段怎么设计
    
- `NULL`、默认值、时间字段怎么处理
    
- 唯一索引、普通索引、联合索引怎么设计
    
- 哪些字段适合单独建列，哪些字段适合放 JSON
    

---

# 2. 场景设定

假设我们要设计一张订单主表，支持这些查询：

```sql
-- 1. 根据订单号查订单
SELECT * FROM order_info WHERE order_no = ?;

-- 2. 查询某个用户的订单列表
SELECT * 
FROM order_info 
WHERE user_id = ? AND is_deleted = 0
ORDER BY order_time DESC
LIMIT 20;

-- 3. 后台按状态和时间范围查询订单
SELECT *
FROM order_info
WHERE order_status = ?
  AND order_time >= ?
  AND order_time < ?;
```

---

# 3. 建表 SQL

```sql
DROP TABLE IF EXISTS `order_info`;

CREATE TABLE `order_info` (
  `id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',

  `order_no` varchar(32) NOT NULL COMMENT '业务订单号',
  `user_id` bigint unsigned NOT NULL COMMENT '用户ID',
  `user_name` varchar(64) NOT NULL DEFAULT '' COMMENT '下单时用户姓名快照',
  `user_mobile` varchar(20) NOT NULL DEFAULT '' COMMENT '下单时用户手机号快照',

  `total_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单实付金额',
  `discount_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单优惠金额',
  `pay_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单应付金额',

  `order_status` tinyint unsigned NOT NULL DEFAULT 0 COMMENT '订单状态：0待支付，1已支付，2已取消，3已完成',
  `is_deleted` tinyint unsigned NOT NULL DEFAULT 0 COMMENT '是否删除：0否，1是',

  `client_ip` varbinary(16) DEFAULT NULL COMMENT '客户端IP，兼容IPv4和IPv6',
  `ext_data` json DEFAULT NULL COMMENT '扩展信息，如设备、渠道、活动参数',

  `order_time` datetime NOT NULL COMMENT '下单时间',
  `pay_time` datetime DEFAULT NULL COMMENT '支付时间',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',

  PRIMARY KEY (`id`),
  UNIQUE KEY `uq_order_no` (`order_no`),
  KEY `idx_user_order_time` (`user_id`, `is_deleted`, `order_time`),
  KEY `idx_status_order_time` (`order_status`, `order_time`)
) ENGINE=InnoDB
  DEFAULT CHARSET=utf8mb4
  COLLATE=utf8mb4_0900_ai_ci
  COMMENT='订单主表';
```

---

# 4. 逐点讲解

## 4.1 主键：用自增 `BIGINT`

```sql
`id` bigint unsigned NOT NULL AUTO_INCREMENT COMMENT '自增主键',
PRIMARY KEY (`id`)
```

InnoDB 的数据按主键组织。自增主键写入更顺序，页分裂更少，适合作为聚簇索引。

不要轻易用随机 UUID、随机订单号做主键，否则插入位置随机，可能导致更多页分裂和随机 I/O。

---

## 4.2 订单号：业务唯一，用唯一索引

```sql
`order_no` varchar(32) NOT NULL COMMENT '业务订单号',
UNIQUE KEY `uq_order_no` (`order_no`)
```

`id` 是数据库内部主键，`order_no` 是对外暴露的业务编号。

典型查询：

```sql
SELECT * FROM order_info WHERE order_no = 'ORD202605130001';
```

所以必须加唯一索引，既保证业务唯一，也提升查询性能。

---

## 4.3 用户信息：保存下单快照

```sql
`user_id` bigint unsigned NOT NULL COMMENT '用户ID',
`user_name` varchar(64) NOT NULL DEFAULT '' COMMENT '下单时用户姓名快照',
`user_mobile` varchar(20) NOT NULL DEFAULT '' COMMENT '下单时用户手机号快照',
```

订单表里保存用户姓名、手机号，是为了保留下单当时的快照。

否则用户后续修改手机号，历史订单展示就会被影响。

手机号用 `varchar(20)`，不要用数字类型。手机号不是用来计算的，而且可能包含国家区号、前导 0、`+86` 等形式。

---

## 4.4 金额：必须用 `DECIMAL`

```sql
`total_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单实付金额',
`discount_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单优惠金额',
`pay_amount` decimal(10,2) NOT NULL DEFAULT 0.00 COMMENT '订单应付金额',
```

金额字段不要用 `float` / `double`，因为它们是近似值，可能出现精度误差。

电商、支付、结算类字段必须用 `decimal`。

金额字段建议设置默认值 `0.00`，减少空值处理。

---

## 4.5 状态：用 `tinyint unsigned`，不用 `enum`

```sql
`order_status` tinyint unsigned NOT NULL DEFAULT 0 COMMENT '订单状态：0待支付，1已支付，2已取消，3已完成',
```

状态字段推荐用 `tinyint unsigned`。

不推荐用 `enum`，因为状态值变更需要改表结构，不利于 Java 代码和数据库之间的长期维护。

Java 里可以这样定义：

```java
public enum OrderStatus {
    WAIT_PAY(0, "待支付"),
    PAID(1, "已支付"),
    CANCELED(2, "已取消"),
    FINISHED(3, "已完成");

    private final int code;
    private final String desc;
}
```

---

## 4.6 逻辑删除：数据库字段用 `is_deleted`

```sql
`is_deleted` tinyint unsigned NOT NULL DEFAULT 0 COMMENT '是否删除：0否，1是',
```

数据库字段可以用 `is_deleted`，表达“是否已删除”。

Java 实体类里建议写成：

```java
private Boolean deleted;
```

不要写成：

```java
private Boolean isDeleted;
```

否则部分序列化、反射、Bean 规范场景下容易出现字段映射问题。

---

## 4.7 IP 地址：教学案例里推荐 `VARBINARY(16)`

```sql
`client_ip` varbinary(16) DEFAULT NULL COMMENT '客户端IP，兼容IPv4和IPv6',
```

`VARBINARY(16)` 可以同时兼容 IPv4 和 IPv6。

插入时：

```sql
INSERT INTO order_info(client_ip) 
VALUES (INET6_ATON('2001:0db8:85a3::8a2e:0370:7334'));
```

查询时：

```sql
SELECT INET6_NTOA(client_ip) FROM order_info;
```

如果系统只考虑 IPv4，也可以用：

```sql
client_ipv4 int unsigned
```

但现在新系统更建议兼容 IPv6。

---

## 4.8 JSON：只放扩展信息

```sql
`ext_data` json DEFAULT NULL COMMENT '扩展信息，如设备、渠道、活动参数',
```

JSON 适合存低频、扩展、不稳定字段，例如：

```json
{
  "device": "iPhone 15",
  "channel": "wechat",
  "campaignId": "ACT202605"
}
```

但核心查询字段不要放 JSON，比如：

```text
user_id
order_status
order_time
amount
sku
```

这些字段应该单独建列，否则索引、查询、统计都会变复杂。

---

## 4.9 时间字段：业务时间和系统时间分开

```sql
`order_time` datetime NOT NULL COMMENT '下单时间',
`pay_time` datetime DEFAULT NULL COMMENT '支付时间',
`create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
`update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
```

这里有两类时间：

|字段|含义|
|---|---|
|`order_time`|用户下单的业务时间|
|`pay_time`|用户支付的业务时间|
|`create_time`|数据插入时间|
|`update_time`|数据更新时间|

不要混用业务时间和系统记录时间。

---

# 5. 索引设计讲解

## 5.1 唯一索引：订单号唯一

```sql
UNIQUE KEY `uq_order_no` (`order_no`)
```

用于支持：

```sql
SELECT * FROM order_info WHERE order_no = ?;
```

同时防止重复订单号写入。

---

## 5.2 用户订单列表索引

```sql
KEY `idx_user_order_time` (`user_id`, `is_deleted`, `order_time`)
```

用于支持：

```sql
SELECT *
FROM order_info
WHERE user_id = ?
  AND is_deleted = 0
ORDER BY order_time DESC
LIMIT 20;
```

索引顺序的原因：

```text
user_id：先定位某个用户
is_deleted：过滤未删除订单
order_time：支持按下单时间排序
```

这是最典型的 C 端订单列表查询。

---

## 5.3 后台状态查询索引

```sql
KEY `idx_status_order_time` (`order_status`, `order_time`)
```

用于支持：

```sql
SELECT *
FROM order_info
WHERE order_status = 1
  AND order_time >= '2026-05-01'
  AND order_time < '2026-06-01';
```

适合后台运营、客服、财务按状态和时间范围筛选订单。

---

# 6. 这份案例可以讲出的面试点

|面试问题|回答方向|
|---|---|
|为什么不用 UUID 做主键？|随机写入可能导致页分裂，自增主键更适合 InnoDB 聚簇索引|
|为什么订单号还要唯一索引？|业务唯一 + 快速查询|
|手机号为什么不用 BIGINT？|本质是字符串标识，不参与计算，可能有区号和前导 0|
|金额为什么不用 double？|浮点数有精度误差|
|状态为什么不用 enum？|扩展和维护成本高|
|JSON 字段什么时候用？|低频、扩展、不稳定字段|
|联合索引怎么设计？|根据真实 SQL 的过滤、排序、范围条件设计|

---

# 7. 教学总结

这份案例的核心不是“订单系统完整建模”，而是借订单表讲清楚 MySQL 建表规范。

一句话总结：

> **主键要稳定顺序，业务号要唯一索引，金额用 DECIMAL，手机号用字符串，状态用 TINYINT，核心字段单独建列，扩展信息放 JSON，索引必须围绕真实查询 SQL 设计。**