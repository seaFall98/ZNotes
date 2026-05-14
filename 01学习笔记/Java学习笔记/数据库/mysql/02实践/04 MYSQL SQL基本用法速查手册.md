

> 约定：
> 
> - 示例基于 MySQL 8.x。
>     
> - 所有 SQL 仅示范核心写法，实际执行前必须结合业务表结构、索引、数据量和事务边界评估。
>     
> - 生产环境默认禁止 `SELECT *`、无条件 `UPDATE / DELETE`、无限制大批量操作。
>     

---

## 1. 查询类 SQL

### 1.1 按主键查询

**场景**

订单详情页根据订单主键查询订单基础信息。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    total_amount,
    pay_amount,
    created_at,
    updated_at
FROM order_info
WHERE id = 10000001
LIMIT 1;
```

**注意事项**

- 主键查询必须带 `LIMIT 1`。
    
- 查询字段明确列出，不使用 `SELECT *`。
    
- Java 代码中主键类型要与数据库字段类型一致。
    

---

### 1.2 按唯一键查询

**场景**

根据订单号查询订单。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    pay_status,
    total_amount,
    pay_amount,
    created_at
FROM order_info
WHERE order_no = 'ORD202605150001'
LIMIT 1;
```

**注意事项**

- `order_no` 应建立唯一索引。
    
- 适合订单号、支付单号、用户账号、手机号加密哈希等唯一业务键查询。
    

---

### 1.3 按普通索引查询

**场景**

查询某个用户最近的订单列表。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    pay_amount,
    created_at
FROM order_info
WHERE user_id = 10001
  AND is_deleted = 0
ORDER BY id DESC
LIMIT 20;
```

**注意事项**

- 推荐索引：`idx_user_deleted_id(user_id, is_deleted, id)`。
    
- 列表查询必须限制返回条数。
    
- 排序字段尽量与索引配合。
    

---

### 1.4 多条件查询

**场景**

后台订单列表按用户、状态、时间范围筛选。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    pay_status,
    pay_amount,
    created_at
FROM order_info
WHERE user_id = 10001
  AND order_status = 20
  AND created_at >= '2026-05-01 00:00:00'
  AND created_at <  '2026-06-01 00:00:00'
  AND is_deleted = 0
ORDER BY id DESC
LIMIT 50;
```

**注意事项**

- 推荐索引：`idx_user_status_time(user_id, order_status, created_at, id)`。
    
- 时间范围使用左闭右开。
    
- 后台查询应限制最大时间跨度。
    

---

### 1.5 范围查询

**场景**

查询指定时间段内已支付订单。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    pay_amount,
    paid_at
FROM order_info
WHERE pay_status = 1
  AND paid_at >= '2026-05-01 00:00:00'
  AND paid_at <  '2026-05-02 00:00:00'
ORDER BY paid_at ASC, id ASC
LIMIT 1000;
```

**注意事项**

- 推荐索引：`idx_pay_status_paid_at_id(pay_status, paid_at, id)`。
    
- 避免 `DATE(paid_at) = '2026-05-01'`。
    
- 大范围查询要分页处理。
    

---

### 1.6 IN 查询

**场景**

根据一批订单 ID 查询订单信息。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    pay_amount
FROM order_info
WHERE id IN (100001, 100002, 100003, 100004);
```

**注意事项**

- `IN` 列表不宜过大，建议单批 500～1000 个以内。
    
- 大批量 ID 查询应分批。
    
- `IN` 字段类型必须与入参类型一致。
    

---

### 1.7 LIKE 查询

**场景**

后台按用户昵称前缀搜索。

**推荐 SQL**

```sql
SELECT
    id,
    user_no,
    nickname,
    mobile_masked,
    created_at
FROM user_info
WHERE nickname LIKE '张%'
  AND is_deleted = 0
ORDER BY id DESC
LIMIT 50;
```

**注意事项**

- `LIKE 'xxx%'` 可利用普通索引。
    
- 避免 `LIKE '%xxx%'`，必要时使用 Elasticsearch / OpenSearch。
    
- 后台模糊查询必须限制返回条数。
    

---

### 1.8 EXISTS 查询

**场景**

查询存在未支付订单的用户。

**推荐 SQL**

```sql
SELECT
    u.id,
    u.user_no,
    u.nickname,
    u.created_at
FROM user_info u
WHERE u.is_deleted = 0
  AND EXISTS (
      SELECT 1
      FROM order_info o
      WHERE o.user_id = u.id
        AND o.pay_status = 0
        AND o.is_deleted = 0
      LIMIT 1
  )
ORDER BY u.id DESC
LIMIT 100;
```

**注意事项**

- 子查询中必须使用可命中索引的条件。
    
- 推荐索引：`order_info.idx_user_pay_deleted(user_id, pay_status, is_deleted)`。
    
- 只判断存在性时使用 `SELECT 1`。
    

---

### 1.9 JOIN 查询

**场景**

后台订单列表展示用户昵称。

**推荐 SQL**

```sql
SELECT
    o.id,
    o.order_no,
    o.user_id,
    u.nickname,
    o.order_status,
    o.pay_amount,
    o.created_at
FROM order_info o
INNER JOIN user_info u ON u.id = o.user_id
WHERE o.created_at >= '2026-05-01 00:00:00'
  AND o.created_at <  '2026-05-02 00:00:00'
  AND o.is_deleted = 0
ORDER BY o.id DESC
LIMIT 100;
```

**注意事项**

- JOIN 字段必须有索引。
    
- 大表 JOIN 前先缩小主表数据范围。
    
- 高并发核心接口避免复杂 JOIN，可通过冗余字段或异步宽表解决。
    

---

### 1.10 聚合查询

**场景**

统计某天支付成功金额。

**推荐 SQL**

```sql
SELECT
    COALESCE(SUM(pay_amount), 0) AS total_pay_amount
FROM order_info
WHERE pay_status = 1
  AND paid_at >= '2026-05-01 00:00:00'
  AND paid_at <  '2026-05-02 00:00:00';
```

**注意事项**

- 金额字段使用 `DECIMAL`，不要用 `DOUBLE / FLOAT`。
    
- 大数据量统计建议走离线表、汇总表或 OLAP。
    
- `SUM()` 结果可能为 `NULL`，使用 `COALESCE`。
    

---

### 1.11 分组统计

**场景**

按订单状态统计订单数量。

**推荐 SQL**

```sql
SELECT
    order_status,
    COUNT(*) AS order_count
FROM order_info
WHERE created_at >= '2026-05-01 00:00:00'
  AND created_at <  '2026-06-01 00:00:00'
  AND is_deleted = 0
GROUP BY order_status;
```

**注意事项**

- 分组维度应是低基数字段，例如状态、类型、渠道。
    
- 大表分组统计建议走汇总表。
    
- 不要在核心接口实时统计大范围数据。
    

---

### 1.12 排序查询

**场景**

查询销量最高的商品。

**推荐 SQL**

```sql
SELECT
    id,
    product_no,
    product_name,
    sale_count,
    created_at
FROM product_info
WHERE product_status = 1
  AND is_deleted = 0
ORDER BY sale_count DESC, id DESC
LIMIT 20;
```

**注意事项**

- 推荐索引：`idx_status_sale_id(product_status, sale_count, id)`。
    
- 排序字段需结合业务过滤条件建立索引。
    
- 避免无条件全表排序。
    

---

### 1.13 游标分页

**场景**

App 查询用户订单列表，使用上一页最后一条 ID 继续翻页。

**推荐 SQL**

第一页：

```sql
SELECT
    id,
    order_no,
    order_status,
    pay_amount,
    created_at
FROM order_info
WHERE user_id = 10001
  AND is_deleted = 0
ORDER BY id DESC
LIMIT 20;
```

下一页：

```sql
SELECT
    id,
    order_no,
    order_status,
    pay_amount,
    created_at
FROM order_info
WHERE user_id = 10001
  AND id < 998800
  AND is_deleted = 0
ORDER BY id DESC
LIMIT 20;
```

**注意事项**

- 推荐用于 App 信息流、订单列表、消息列表。
    
- 游标字段要稳定、递增、可排序。
    
- 常用 `id` 或 `(created_at, id)` 作为游标。
    

---

### 1.14 深分页替代写法

**场景**

后台订单查询第 1000 页，避免 `LIMIT 999000, 1000`。

**不推荐**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    created_at
FROM order_info
WHERE is_deleted = 0
ORDER BY id DESC
LIMIT 999000, 1000;
```

**推荐 SQL**

```sql
SELECT
    o.id,
    o.order_no,
    o.user_id,
    o.order_status,
    o.created_at
FROM order_info o
INNER JOIN (
    SELECT
        id
    FROM order_info
    WHERE is_deleted = 0
    ORDER BY id DESC
    LIMIT 999000, 1000
) t ON t.id = o.id
ORDER BY o.id DESC;
```

更推荐游标分页：

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    created_at
FROM order_info
WHERE is_deleted = 0
  AND id < 90000000
ORDER BY id DESC
LIMIT 1000;
```

**注意事项**

- 深分页优先改成交互方式，不允许用户无限跳页。
    
- 导出场景必须按游标分批。
    
- 管理后台可限制最大页数。
    

---

### 1.15 查询最近 N 条记录

**场景**

查询用户最近 10 条登录日志。

**推荐 SQL**

```sql
SELECT
    id,
    user_id,
    login_ip,
    login_device,
    login_at
FROM user_login_log
WHERE user_id = 10001
ORDER BY id DESC
LIMIT 10;
```

**注意事项**

- 推荐索引：`idx_user_id_id(user_id, id)`。
    
- 日志表建议按时间归档或分区。
    
- 不要一次查询全部历史日志。
    

---

### 1.16 查询是否存在

**场景**

判断手机号是否已注册。

**推荐 SQL**

```sql
SELECT
    1
FROM user_info
WHERE mobile_hash = 'e10adc3949ba59abbe56e057f20f883e'
  AND is_deleted = 0
LIMIT 1;
```

**注意事项**

- 只查存在性，不要查完整行。
    
- 手机号建议存密文或哈希索引字段。
    
- 推荐唯一索引：`uk_mobile_hash(mobile_hash)`。
    

---

## 2. 新增类 SQL

### 2.1 单条插入

**场景**

创建用户。

**推荐 SQL**

```sql
INSERT INTO user_info (
    user_no,
    nickname,
    mobile_hash,
    mobile_masked,
    user_status,
    is_deleted,
    created_at,
    updated_at
) VALUES (
    'U202605150001',
    '张三',
    'e10adc3949ba59abbe56e057f20f883e',
    '138****8888',
    1,
    0,
    NOW(3),
    NOW(3)
);
```

**注意事项**

- 明确字段名。
    
- 不依赖字段默认顺序。
    
- 敏感字段避免明文存储。
    

---

### 2.2 批量插入

**场景**

批量写入优惠券发放记录。

**推荐 SQL**

```sql
INSERT INTO coupon_grant_record (
    grant_no,
    user_id,
    coupon_id,
    grant_status,
    source_type,
    created_at,
    updated_at
) VALUES
    ('GRANT202605150001', 10001, 20001, 1, 10, NOW(3), NOW(3)),
    ('GRANT202605150002', 10002, 20001, 1, 10, NOW(3), NOW(3)),
    ('GRANT202605150003', 10003, 20001, 1, 10, NOW(3), NOW(3));
```

**注意事项**

- 单批数量建议控制在 500～1000 条。
    
- 批量插入失败要有重试和错误记录。
    
- 关键业务应有幂等唯一键。
    

---

### 2.3 插入时忽略重复

**场景**

用户领取优惠券，重复领取时忽略。

**推荐 SQL**

```sql
INSERT IGNORE INTO user_coupon (
    user_id,
    coupon_id,
    coupon_status,
    received_at,
    created_at,
    updated_at
) VALUES (
    10001,
    20001,
    1,
    NOW(3),
    NOW(3),
    NOW(3)
);
```

**注意事项**

- 需要唯一索引：`uk_user_coupon(user_id, coupon_id)`。
    
- `INSERT IGNORE` 会忽略部分错误，业务上要确认可接受。
    
- 更严谨场景可使用 `ON DUPLICATE KEY UPDATE`。
    

---

### 2.4 插入或更新

**场景**

更新用户购物车商品数量，没有则插入。

**推荐 SQL**

```sql
INSERT INTO shopping_cart (
    user_id,
    product_id,
    sku_id,
    quantity,
    selected,
    created_at,
    updated_at
) VALUES (
    10001,
    30001,
    40001,
    2,
    1,
    NOW(3),
    NOW(3)
)
ON DUPLICATE KEY UPDATE
    quantity = quantity + VALUES(quantity),
    selected = VALUES(selected),
    updated_at = NOW(3);
```

**注意事项**

- 需要唯一索引：`uk_user_sku(user_id, sku_id)`。
    
- 并发下可能出现锁冲突，Java 侧要捕获异常并重试。
    
- 不适合复杂状态机更新。
    

---

### 2.5 根据查询结果插入

**场景**

给符合条件的老用户初始化会员权益。

**推荐 SQL**

```sql
INSERT INTO user_member_benefit (
    user_id,
    benefit_type,
    benefit_status,
    start_at,
    end_at,
    created_at,
    updated_at
)
SELECT
    u.id AS user_id,
    10 AS benefit_type,
    1 AS benefit_status,
    NOW(3) AS start_at,
    '2026-12-31 23:59:59.999' AS end_at,
    NOW(3) AS created_at,
    NOW(3) AS updated_at
FROM user_info u
WHERE u.register_at >= '2026-01-01 00:00:00'
  AND u.register_at <  '2026-02-01 00:00:00'
  AND u.user_status = 1
  AND u.is_deleted = 0
LIMIT 1000;
```

**注意事项**

- 大批量初始化必须分批执行。
    
- 目标表建议有唯一键防重复。
    
- 执行前先 `SELECT` 验证数据范围。
    

---

### 2.6 初始化配置数据

**场景**

初始化系统参数配置。

**推荐 SQL**

```sql
INSERT INTO system_config (
    config_key,
    config_value,
    config_desc,
    config_status,
    created_at,
    updated_at
) VALUES
    ('order.timeout.minutes', '30', '订单超时时间，单位分钟', 1, NOW(3), NOW(3)),
    ('coupon.max.receive.count', '1', '单用户最大领取次数', 1, NOW(3), NOW(3))
ON DUPLICATE KEY UPDATE
    config_value = VALUES(config_value),
    config_desc = VALUES(config_desc),
    config_status = VALUES(config_status),
    updated_at = NOW(3);
```

**注意事项**

- `config_key` 必须建立唯一索引。
    
- 初始化 SQL 应可重复执行。
    
- 生产配置变更建议走审批。
    

---

### 2.7 写入审计日志

**场景**

管理员修改用户状态后记录审计日志。

**推荐 SQL**

```sql
INSERT INTO admin_audit_log (
    operator_id,
    operator_name,
    operation_type,
    biz_type,
    biz_id,
    before_content,
    after_content,
    client_ip,
    created_at
) VALUES (
    90001,
    'admin_zhangsan',
    'UPDATE_USER_STATUS',
    'USER',
    10001,
    JSON_OBJECT('user_status', 1),
    JSON_OBJECT('user_status', 2),
    '10.10.1.20',
    NOW(3)
);
```

**注意事项**

- 审计日志只增不改。
    
- 关键字段明确保存，JSON 用于扩展。
    
- 审计日志表建议独立归档。
    

---

## 3. 更新类 SQL

### 3.1 按主键更新

**场景**

修改用户昵称。

**推荐 SQL**

```sql
UPDATE user_info
SET
    nickname = '李四',
    updated_at = NOW(3)
WHERE id = 10001
  AND is_deleted = 0;
```

**注意事项**

- 必须带主键或明确唯一条件。
    
- 更新字段明确列出。
    
- 不要更新无变化字段。
    

---

### 3.2 条件更新

**场景**

冻结存在风控风险的用户。

**推荐 SQL**

```sql
UPDATE user_info
SET
    user_status = 2,
    freeze_reason = 'RISK_CONTROL',
    updated_at = NOW(3)
WHERE id = 10001
  AND user_status = 1
  AND is_deleted = 0;
```

**注意事项**

- 条件中带当前状态，避免重复更新。
    
- Java 根据影响行数判断是否更新成功。
    
- 状态字段要使用明确枚举值。
    

---

### 3.3 状态流转更新

**场景**

订单从待支付变为已取消。

**推荐 SQL**

```sql
UPDATE order_info
SET
    order_status = 90,
    cancel_reason = 'PAY_TIMEOUT',
    canceled_at = NOW(3),
    updated_at = NOW(3)
WHERE order_no = 'ORD202605150001'
  AND order_status = 10
  AND pay_status = 0
  AND is_deleted = 0;
```

**注意事项**

- 状态流转必须带前置状态。
    
- 影响行数为 1 才代表成功。
    
- 不允许直接无条件覆盖状态。
    

---

### 3.4 乐观锁更新

**场景**

用户资料编辑防止并发覆盖。

**推荐 SQL**

```sql
UPDATE user_profile
SET
    avatar_url = 'https://cdn.example.com/avatar/10001.png',
    profile_desc = 'Java Backend Developer',
    version = version + 1,
    updated_at = NOW(3)
WHERE user_id = 10001
  AND version = 8;
```

**注意事项**

- Java 侧更新失败后提示重试或重新读取。
    
- 适合低冲突业务。
    
- `version` 字段不允许客户端随意修改。
    

---

### 3.5 库存扣减

**场景**

用户下单扣减 SKU 库存。

**推荐 SQL**

```sql
UPDATE sku_stock
SET
    available_stock = available_stock - 2,
    locked_stock = locked_stock + 2,
    updated_at = NOW(3)
WHERE sku_id = 40001
  AND available_stock >= 2;
```

**注意事项**

- 必须在 SQL 条件中判断库存充足。
    
- Java 根据影响行数判断扣减是否成功。
    
- 不要先查库存再无条件更新。
    

---

### 3.6 金额余额更新

**场景**

用户支付扣减账户余额。

**推荐 SQL**

```sql
UPDATE user_account
SET
    available_balance = available_balance - 99.90,
    frozen_balance = frozen_balance + 99.90,
    updated_at = NOW(3)
WHERE user_id = 10001
  AND available_balance >= 99.90
  AND account_status = 1;
```

**注意事项**

- 金额字段使用 `DECIMAL(18, 2)` 或更高精度。
    
- 扣款条件必须包含余额充足。
    
- 账户流水必须与余额更新在同一事务内。
    

---

### 3.7 批量更新

**场景**

批量关闭超时未支付订单。

**推荐 SQL**

```sql
UPDATE order_info
SET
    order_status = 90,
    cancel_reason = 'PAY_TIMEOUT',
    canceled_at = NOW(3),
    updated_at = NOW(3)
WHERE order_status = 10
  AND pay_status = 0
  AND created_at < DATE_SUB(NOW(3), INTERVAL 30 MINUTE)
ORDER BY id ASC
LIMIT 500;
```

**注意事项**

- 批量更新必须带 `LIMIT`。
    
- 定时任务循环分批执行。
    
- 每批更新后记录影响行数。
    

---

### 3.8 CASE WHEN 批量更新

**场景**

批量更新商品价格。

**推荐 SQL**

```sql
UPDATE product_sku
SET
    sale_price = CASE sku_id
        WHEN 40001 THEN 99.90
        WHEN 40002 THEN 129.90
        WHEN 40003 THEN 159.90
    END,
    updated_at = NOW(3)
WHERE sku_id IN (40001, 40002, 40003);
```

**注意事项**

- 适合小批量不同值更新。
    
- `WHERE` 条件必须限制更新范围。
    
- 大批量建议临时表关联更新。
    

---

### 3.9 只更新非空字段的常见写法

**场景**

后台编辑用户信息，传入字段为空则保持原值。

**推荐 SQL**

```sql
UPDATE user_info
SET
    nickname = COALESCE(NULLIF('新昵称', ''), nickname),
    email = COALESCE(NULLIF('new_email@example.com', ''), email),
    updated_at = NOW(3)
WHERE id = 10001
  AND is_deleted = 0;
```

**注意事项**

- 更推荐 Java 动态 SQL 只更新传入字段。
    
- 不要把空字符串误更新为有效业务值。
    
- 敏感字段修改要记录审计日志。
    

---

### 3.10 更新时间字段维护

**场景**

修改订单备注，同时维护更新时间。

**推荐 SQL**

```sql
UPDATE order_info
SET
    buyer_remark = '请尽快发货',
    updated_at = NOW(3)
WHERE order_no = 'ORD202605150001'
  AND user_id = 10001
  AND is_deleted = 0;
```

**注意事项**

- 业务更新统一维护 `updated_at`。
    
- 建议使用 `DATETIME(3)`。
    
- 不建议完全依赖数据库自动更新，避免行为不透明。
    

---

## 4. 删除类 SQL

### 4.1 按主键删除

**场景**

删除测试环境中的指定配置。

**推荐 SQL**

```sql
DELETE FROM system_config
WHERE id = 10001
  AND config_key = 'test.config.key'
LIMIT 1;
```

**注意事项**

- 生产业务数据优先软删除。
    
- 物理删除必须带明确条件。
    
- 删除前先执行对应 `SELECT` 验证。
    

---

### 4.2 条件删除

**场景**

删除指定用户未提交的草稿。

**推荐 SQL**

```sql
DELETE FROM article_draft
WHERE user_id = 10001
  AND draft_status = 0
  AND updated_at < '2026-05-01 00:00:00'
LIMIT 100;
```

**注意事项**

- 条件删除必须带 `LIMIT`。
    
- 大批量删除必须分批。
    
- 删除任务要记录执行日志。
    

---

### 4.3 软删除

**场景**

用户删除收货地址。

**推荐 SQL**

```sql
UPDATE user_address
SET
    is_deleted = 1,
    deleted_at = NOW(3),
    updated_at = NOW(3)
WHERE id = 50001
  AND user_id = 10001
  AND is_deleted = 0;
```

**注意事项**

- 用户侧删除通常使用软删除。
    
- 查询时统一加 `is_deleted = 0`。
    
- 唯一索引设计要考虑软删除。
    

---

### 4.4 批量软删除

**场景**

批量下架违规商品。

**推荐 SQL**

```sql
UPDATE product_info
SET
    is_deleted = 1,
    deleted_at = NOW(3),
    updated_at = NOW(3)
WHERE id IN (30001, 30002, 30003)
  AND product_status = 2
  AND is_deleted = 0;
```

**注意事项**

- `IN` 列表不宜过大。
    
- 操作前备份目标数据。
    
- 批量操作应记录操作人和原因。
    

---

### 4.5 物理删除历史数据

**场景**

删除 180 天前的接口调用日志。

**推荐 SQL**

```sql
DELETE FROM api_access_log
WHERE created_at < DATE_SUB(NOW(3), INTERVAL 180 DAY)
ORDER BY id ASC
LIMIT 1000;
```

**注意事项**

- 日志表物理删除必须分批。
    
- 大日志表优先按月归档或分区。
    
- 避免一次性删除百万级数据。
    

---

### 4.6 删除前备份

**场景**

删除异常优惠券发放记录前备份。

**推荐 SQL**

```sql
CREATE TABLE coupon_grant_record_bak_20260515 AS
SELECT
    id,
    grant_no,
    user_id,
    coupon_id,
    grant_status,
    source_type,
    created_at,
    updated_at
FROM coupon_grant_record
WHERE source_type = 99
  AND created_at >= '2026-05-15 00:00:00'
  AND created_at <  '2026-05-16 00:00:00';
```

删除：

```sql
DELETE FROM coupon_grant_record
WHERE source_type = 99
  AND created_at >= '2026-05-15 00:00:00'
  AND created_at <  '2026-05-16 00:00:00'
LIMIT 1000;
```

**注意事项**

- 备份表名包含日期。
    
- 先确认备份数量与待删除数量一致。
    
- 删除后保留备份表一段时间。
    

---

### 4.7 分批删除大表数据

**场景**

循环删除历史消息。

**推荐 SQL**

```sql
DELETE FROM user_message
WHERE id < 90000000
  AND message_status = 3
  AND created_at < '2026-01-01 00:00:00'
ORDER BY id ASC
LIMIT 1000;
```

**注意事项**

- Java / 脚本循环执行，直到影响行数为 0。
    
- 每批之间可短暂 sleep，降低主库压力。
    
- 避免长事务和大锁等待。
    

---

## 5. 分页与列表查询

### 5.1 普通分页

**场景**

后台用户列表，页数较浅。

**推荐 SQL**

```sql
SELECT
    id,
    user_no,
    nickname,
    mobile_masked,
    user_status,
    created_at
FROM user_info
WHERE is_deleted = 0
ORDER BY id DESC
LIMIT 0, 20;
```

**注意事项**

- 只适合浅分页。
    
- 后台最大页数应有限制。
    
- 返回字段要控制。
    

---

### 5.2 游标分页

**场景**

用户消息列表。

**推荐 SQL**

```sql
SELECT
    id,
    user_id,
    message_title,
    message_type,
    read_status,
    created_at
FROM user_message
WHERE user_id = 10001
  AND id < 80000000
  AND is_deleted = 0
ORDER BY id DESC
LIMIT 20;
```

**注意事项**

- App 列表优先使用游标分页。
    
- 下一页传上一页最后一条 `id`。
    
- 不支持随意跳页，但性能稳定。
    

---

### 5.3 ID 翻页

**场景**

批量扫描订单表做数据同步。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    updated_at
FROM order_info
WHERE id > 10000000
ORDER BY id ASC
LIMIT 1000;
```

**注意事项**

- 适合数据迁移、补偿任务、导出。
    
- 每次记录本批最大 `id`。
    
- 不受新增数据影响太大。
    

---

### 5.4 时间游标分页

**场景**

按支付时间导出支付订单。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    pay_amount,
    paid_at
FROM order_info
WHERE pay_status = 1
  AND paid_at > '2026-05-01 10:00:00.000'
ORDER BY paid_at ASC
LIMIT 1000;
```

**注意事项**

- 时间字段可能重复。
    
- 仅用时间游标可能漏数据或重复。
    
- 更推荐复合游标。
    

---

### 5.5 复合游标分页

**场景**

按支付时间和 ID 稳定导出订单。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    pay_amount,
    paid_at
FROM order_info
WHERE pay_status = 1
  AND (
      paid_at > '2026-05-01 10:00:00.000'
      OR (
          paid_at = '2026-05-01 10:00:00.000'
          AND id > 10000001
      )
  )
ORDER BY paid_at ASC, id ASC
LIMIT 1000;
```

**注意事项**

- 推荐索引：`idx_pay_paid_id(pay_status, paid_at, id)`。
    
- 适合导出、同步、补偿任务。
    
- 每批记录最后一条的 `paid_at` 和 `id`。
    

---

### 5.6 后台管理列表查询

**场景**

后台订单列表按多条件筛选。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    pay_status,
    pay_amount,
    created_at
FROM order_info
WHERE created_at >= '2026-05-01 00:00:00'
  AND created_at <  '2026-05-15 00:00:00'
  AND order_status = 20
  AND is_deleted = 0
ORDER BY id DESC
LIMIT 100;
```

**注意事项**

- 后台查询必须限制时间范围。
    
- 复杂多条件查询需要针对高频条件设计索引。
    
- 非高频条件可走搜索引擎或报表库。
    

---

### 5.7 App 信息流查询

**场景**

App 首页查询推荐文章流。

**推荐 SQL**

```sql
SELECT
    id,
    article_title,
    cover_url,
    author_id,
    publish_at,
    like_count
FROM article_info
WHERE publish_status = 1
  AND id < 70000000
  AND is_deleted = 0
ORDER BY id DESC
LIMIT 20;
```

**注意事项**

- 信息流不使用深分页。
    
- 可以结合推荐服务生成候选 ID。
    
- 热门数据建议缓存。
    

---

### 5.8 导出场景分页查询

**场景**

导出某月支付订单。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    pay_amount,
    pay_channel,
    paid_at
FROM order_info
WHERE pay_status = 1
  AND paid_at >= '2026-05-01 00:00:00'
  AND paid_at <  '2026-06-01 00:00:00'
  AND id > 10000000
ORDER BY id ASC
LIMIT 2000;
```

**注意事项**

- 导出必须异步化。
    
- 分批查询，分批写文件。
    
- 不允许一次性加载全部数据到内存。
    

---

## 6. 事务与并发控制 SQL

### 6.1 显式事务

**场景**

创建订单和订单明细必须同时成功。

**推荐 SQL**

```sql
START TRANSACTION;

INSERT INTO order_info (
    order_no,
    user_id,
    order_status,
    pay_status,
    total_amount,
    pay_amount,
    created_at,
    updated_at
) VALUES (
    'ORD202605150001',
    10001,
    10,
    0,
    199.80,
    199.80,
    NOW(3),
    NOW(3)
);

INSERT INTO order_item (
    order_no,
    product_id,
    sku_id,
    product_name,
    quantity,
    sale_price,
    created_at,
    updated_at
) VALUES (
    'ORD202605150001',
    30001,
    40001,
    'Java 实战课程',
    2,
    99.90,
    NOW(3),
    NOW(3)
);

COMMIT;
```

**注意事项**

- 事务内 SQL 尽量短。
    
- 不要在事务中调用远程接口。
    
- 异常时必须回滚。
    

---

### 6.2 SELECT ... FOR UPDATE

**场景**

支付前锁定订单，防止并发支付。

**推荐 SQL**

```sql
START TRANSACTION;

SELECT
    id,
    order_no,
    user_id,
    order_status,
    pay_status,
    pay_amount
FROM order_info
WHERE order_no = 'ORD202605150001'
  AND is_deleted = 0
FOR UPDATE;

UPDATE order_info
SET
    pay_status = 1,
    order_status = 20,
    paid_at = NOW(3),
    updated_at = NOW(3)
WHERE order_no = 'ORD202605150001'
  AND pay_status = 0
  AND order_status = 10;

COMMIT;
```

**注意事项**

- `FOR UPDATE` 必须命中索引。
    
- 锁范围越小越好。
    
- 事务中不要做耗时操作。
    

---

### 6.3 乐观锁版本号

**场景**

并发更新商品信息。

**推荐 SQL**

```sql
UPDATE product_info
SET
    product_name = '新版商品名称',
    version = version + 1,
    updated_at = NOW(3)
WHERE id = 30001
  AND version = 5
  AND is_deleted = 0;
```

**注意事项**

- 更新失败由 Java 侧判断影响行数。
    
- 适合低冲突场景。
    
- 高冲突库存扣减优先使用条件更新。
    

---

### 6.4 防止重复提交

**场景**

用户重复点击提交订单。

**推荐 SQL**

```sql
INSERT INTO request_idempotent (
    request_no,
    user_id,
    biz_type,
    biz_status,
    created_at,
    updated_at
) VALUES (
    'REQ202605150001',
    10001,
    'CREATE_ORDER',
    0,
    NOW(3),
    NOW(3)
);
```

**注意事项**

- `request_no` 建唯一索引。
    
- 插入成功才继续执行业务。
    
- 重复键异常直接返回处理中或已完成结果。
    

---

### 6.5 幂等插入

**场景**

支付回调重复通知，只处理一次。

**推荐 SQL**

```sql
INSERT INTO payment_notify_record (
    notify_no,
    payment_no,
    order_no,
    notify_status,
    raw_content,
    created_at,
    updated_at
) VALUES (
    'NOTIFY202605150001',
    'PAY202605150001',
    'ORD202605150001',
    0,
    JSON_OBJECT('channel', 'ALIPAY', 'trade_status', 'SUCCESS'),
    NOW(3),
    NOW(3)
)
ON DUPLICATE KEY UPDATE
    updated_at = updated_at;
```

**注意事项**

- `notify_no` 或 `payment_no` 建唯一索引。
    
- 重复回调不应重复加钱、发货、改状态。
    
- 后续业务更新也必须带状态条件。
    

---

### 6.6 库存扣减事务

**场景**

下单时扣减库存并生成库存流水。

**推荐 SQL**

```sql
START TRANSACTION;

UPDATE sku_stock
SET
    available_stock = available_stock - 1,
    locked_stock = locked_stock + 1,
    updated_at = NOW(3)
WHERE sku_id = 40001
  AND available_stock >= 1;

INSERT INTO stock_flow (
    flow_no,
    sku_id,
    change_type,
    change_quantity,
    biz_type,
    biz_no,
    created_at
) VALUES (
    'STOCK202605150001',
    40001,
    'LOCK',
    -1,
    'ORDER',
    'ORD202605150001',
    NOW(3)
);

COMMIT;
```

**注意事项**

- Java 必须判断库存更新影响行数。
    
- 库存流水需要幂等唯一键：`uk_biz_sku(biz_type, biz_no, sku_id)`。
    
- 事务内不要调用外部服务。
    

---

### 6.7 订单状态流转事务

**场景**

支付成功后订单改为已支付并写状态日志。

**推荐 SQL**

```sql
START TRANSACTION;

UPDATE order_info
SET
    order_status = 20,
    pay_status = 1,
    paid_at = NOW(3),
    updated_at = NOW(3)
WHERE order_no = 'ORD202605150001'
  AND order_status = 10
  AND pay_status = 0
  AND is_deleted = 0;

INSERT INTO order_status_log (
    order_no,
    from_status,
    to_status,
    change_reason,
    operator_type,
    created_at
) VALUES (
    'ORD202605150001',
    10,
    20,
    'PAY_SUCCESS',
    'SYSTEM',
    NOW(3)
);

COMMIT;
```

**注意事项**

- 订单状态更新失败时，不应写成功日志。
    
- 状态日志只增不改。
    
- 建议 Java 根据影响行数控制日志写入。
    

---

### 6.8 转账事务

**场景**

账户 A 向账户 B 转账。

**推荐 SQL**

```sql
START TRANSACTION;

UPDATE user_account
SET
    available_balance = available_balance - 100.00,
    updated_at = NOW(3)
WHERE user_id = 10001
  AND available_balance >= 100.00
  AND account_status = 1;

UPDATE user_account
SET
    available_balance = available_balance + 100.00,
    updated_at = NOW(3)
WHERE user_id = 10002
  AND account_status = 1;

INSERT INTO account_flow (
    flow_no,
    from_user_id,
    to_user_id,
    amount,
    flow_type,
    biz_no,
    created_at
) VALUES (
    'FLOW202605150001',
    10001,
    10002,
    100.00,
    'TRANSFER',
    'TRANSFER202605150001',
    NOW(3)
);

COMMIT;
```

**注意事项**

- 扣款失败必须回滚。
    
- 转账流水必须有唯一业务号。
    
- 多账户加锁时保持固定顺序，减少死锁。
    

---

### 6.9 任务抢占 SQL

**场景**

多个 worker 抢占待执行任务。

**推荐 SQL**

```sql
UPDATE async_task
SET
    task_status = 1,
    worker_id = 'worker-01',
    locked_at = NOW(3),
    updated_at = NOW(3)
WHERE task_status = 0
  AND execute_at <= NOW(3)
ORDER BY priority DESC, id ASC
LIMIT 1;
```

随后查询自己抢到的任务：

```sql
SELECT
    id,
    task_no,
    task_type,
    task_payload,
    retry_count
FROM async_task
WHERE worker_id = 'worker-01'
  AND task_status = 1
ORDER BY locked_at DESC
LIMIT 1;
```

**注意事项**

- 任务状态更新要原子化。
    
- 超时任务需要重置机制。
    
- 推荐索引：`idx_status_execute_priority(task_status, execute_at, priority, id)`。
    

---

## 7. 索引友好写法

### 7.1 等值查询

**场景**

根据用户 ID 查询账户。

**推荐 SQL**

```sql
SELECT
    id,
    user_id,
    available_balance,
    frozen_balance,
    account_status
FROM user_account
WHERE user_id = 10001
LIMIT 1;
```

**注意事项**

- `user_id` 建唯一索引或普通索引。
    
- 入参类型必须与字段类型一致。
    
- 等值字段适合放联合索引前面。
    

---

### 7.2 最左前缀匹配

**场景**

联合索引 `(user_id, order_status, created_at)` 查询用户订单。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    created_at
FROM order_info
WHERE user_id = 10001
  AND order_status = 20
ORDER BY created_at DESC
LIMIT 20;
```

**注意事项**

- 使用联合索引时，从最左列开始匹配。
    
- 不要跳过联合索引前导列。
    
- 高频查询条件决定索引顺序。
    

---

### 7.3 范围条件写法

**场景**

查询某天订单。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    created_at
FROM order_info
WHERE created_at >= '2026-05-15 00:00:00'
  AND created_at <  '2026-05-16 00:00:00'
ORDER BY id ASC
LIMIT 1000;
```

**注意事项**

- 使用左闭右开。
    
- 不使用 `DATE(created_at)`。
    
- 范围字段后面的索引列利用受限，索引设计要谨慎。
    

---

### 7.4 避免函数作用在索引列

**场景**

按注册日期查询用户。

**不推荐**

```sql
SELECT
    id,
    user_no,
    nickname,
    created_at
FROM user_info
WHERE DATE(created_at) = '2026-05-15';
```

**推荐 SQL**

```sql
SELECT
    id,
    user_no,
    nickname,
    created_at
FROM user_info
WHERE created_at >= '2026-05-15 00:00:00'
  AND created_at <  '2026-05-16 00:00:00';
```

**注意事项**

- 索引列不要包函数。
    
- 时间范围由 Java 或 SQL 边界表达。
    
- 统计场景可使用生成列或汇总表。
    

---

### 7.5 避免隐式类型转换

**场景**

根据用户编号查询用户。

**不推荐**

```sql
SELECT
    id,
    user_no,
    nickname
FROM user_info
WHERE user_no = 202605150001;
```

**推荐 SQL**

```sql
SELECT
    id,
    user_no,
    nickname
FROM user_info
WHERE user_no = '202605150001';
```

**注意事项**

- 字符串字段必须使用字符串参数。
    
- Java 参数类型要与数据库字段一致。
    
- 隐式转换可能导致索引失效。
    

---

### 7.6 避免前置通配符 LIKE

**场景**

根据商品名称搜索。

**不推荐**

```sql
SELECT
    id,
    product_name,
    sale_price
FROM product_info
WHERE product_name LIKE '%手机%'
LIMIT 20;
```

**推荐 SQL**

```sql
SELECT
    id,
    product_name,
    sale_price
FROM product_info
WHERE product_name LIKE '手机%'
LIMIT 20;
```

**更推荐**

```sql
SELECT
    id,
    product_name,
    sale_price
FROM product_search_keyword
WHERE keyword = '手机'
ORDER BY score DESC
LIMIT 20;
```

**注意事项**

- 包含搜索使用搜索引擎。
    
- MySQL 适合前缀匹配，不适合全文检索大流量场景。
    
- 商品搜索不要硬扛在业务库上。
    

---

### 7.7 ORDER BY 利用索引

**场景**

查询某用户最近订单。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    order_status,
    pay_amount,
    created_at
FROM order_info
WHERE user_id = 10001
ORDER BY id DESC
LIMIT 20;
```

**注意事项**

- 推荐索引：`idx_user_id_id(user_id, id)`。
    
- 排序字段尽量放入联合索引。
    
- 避免大结果集 filesort。
    

---

### 7.8 覆盖索引查询

**场景**

订单列表只展示轻量字段。

**推荐 SQL**

```sql
SELECT
    user_id,
    order_status,
    id,
    created_at
FROM order_info
WHERE user_id = 10001
  AND order_status = 20
ORDER BY id DESC
LIMIT 20;
```

**注意事项**

- 推荐索引：`idx_user_status_id_time(user_id, order_status, id, created_at)`。
    
- 查询字段都在索引中，可减少回表。
    
- 不要为了覆盖索引把大字段放进索引。
    

---

### 7.9 联合索引查询示例

**场景**

后台按商户、状态、时间查询支付单。

**推荐 SQL**

```sql
SELECT
    id,
    payment_no,
    merchant_id,
    payment_status,
    pay_amount,
    created_at
FROM payment_order
WHERE merchant_id = 80001
  AND payment_status = 1
  AND created_at >= '2026-05-01 00:00:00'
  AND created_at <  '2026-06-01 00:00:00'
ORDER BY created_at DESC, id DESC
LIMIT 100;
```

**注意事项**

- 推荐索引：`idx_merchant_status_time_id(merchant_id, payment_status, created_at, id)`。
    
- 等值条件在前，范围和排序字段在后。
    
- 根据最高频查询设计索引，而不是盲目堆索引。
    

---

## 8. 统计分析 SQL

### 8.1 COUNT 查询

**场景**

统计某天订单数量。

**推荐 SQL**

```sql
SELECT
    COUNT(*) AS order_count
FROM order_info
WHERE created_at >= '2026-05-15 00:00:00'
  AND created_at <  '2026-05-16 00:00:00'
  AND is_deleted = 0;
```

**注意事项**

- 大表实时 `COUNT` 成本高。
    
- 后台统计建议限制时间范围。
    
- 高频统计建议维护汇总表。
    

---

### 8.2 条件计数

**场景**

统计订单中待支付、已支付、已取消数量。

**推荐 SQL**

```sql
SELECT
    COUNT(*) AS total_count,
    SUM(CASE WHEN order_status = 10 THEN 1 ELSE 0 END) AS pending_pay_count,
    SUM(CASE WHEN order_status = 20 THEN 1 ELSE 0 END) AS paid_count,
    SUM(CASE WHEN order_status = 90 THEN 1 ELSE 0 END) AS canceled_count
FROM order_info
WHERE created_at >= '2026-05-01 00:00:00'
  AND created_at <  '2026-06-01 00:00:00'
  AND is_deleted = 0;
```

**注意事项**

- 适合小范围聚合。
    
- 大范围统计走报表表。
    
- 条件枚举要与代码一致。
    

---

### 8.3 按天统计

**场景**

统计每日支付金额。

**推荐 SQL**

```sql
SELECT
    DATE(paid_at) AS stat_date,
    COUNT(*) AS order_count,
    COALESCE(SUM(pay_amount), 0) AS total_pay_amount
FROM order_info
WHERE pay_status = 1
  AND paid_at >= '2026-05-01 00:00:00'
  AND paid_at <  '2026-06-01 00:00:00'
GROUP BY DATE(paid_at)
ORDER BY stat_date ASC;
```

**注意事项**

- `WHERE` 条件不要对索引列使用函数。
    
- `SELECT / GROUP BY` 中可用于统计展示。
    
- 大数据量按天统计建议写入日汇总表。
    

---

### 8.4 按小时统计

**场景**

统计当天每小时支付订单数。

**推荐 SQL**

```sql
SELECT
    DATE_FORMAT(paid_at, '%Y-%m-%d %H:00:00') AS stat_hour,
    COUNT(*) AS paid_order_count,
    COALESCE(SUM(pay_amount), 0) AS paid_amount
FROM order_info
WHERE pay_status = 1
  AND paid_at >= '2026-05-15 00:00:00'
  AND paid_at <  '2026-05-16 00:00:00'
GROUP BY DATE_FORMAT(paid_at, '%Y-%m-%d %H:00:00')
ORDER BY stat_hour ASC;
```

**注意事项**

- 限制单日或较小时间范围。
    
- 实时大盘建议异步汇总。
    
- 不要在核心交易链路实时跑该 SQL。
    

---

### 8.5 用户留存类统计

**场景**

统计 5 月 1 日注册用户在 5 月 2 日是否登录。

**推荐 SQL**

```sql
SELECT
    COUNT(DISTINCT r.user_id) AS register_user_count,
    COUNT(DISTINCT l.user_id) AS next_day_login_user_count
FROM (
    SELECT
        id AS user_id
    FROM user_info
    WHERE register_at >= '2026-05-01 00:00:00'
      AND register_at <  '2026-05-02 00:00:00'
      AND is_deleted = 0
) r
LEFT JOIN user_login_log l ON l.user_id = r.user_id
    AND l.login_at >= '2026-05-02 00:00:00'
    AND l.login_at <  '2026-05-03 00:00:00';
```

**注意事项**

- 留存统计建议离线计算。
    
- 登录日志表需要索引：`idx_user_login_at(user_id, login_at)`。
    
- 大数据量不要直接在业务库高峰期执行。
    

---

### 8.6 Top N 查询

**场景**

查询销量 Top 10 商品。

**推荐 SQL**

```sql
SELECT
    product_id,
    product_name,
    SUM(quantity) AS sale_quantity
FROM order_item
WHERE created_at >= '2026-05-01 00:00:00'
  AND created_at <  '2026-06-01 00:00:00'
GROUP BY product_id, product_name
ORDER BY sale_quantity DESC
LIMIT 10;
```

**注意事项**

- 商品名称可能变更，严谨场景按商品 ID 统计后再查商品表。
    
- 大表 Top N 建议使用汇总表。
    
- 不要在高峰期执行大范围聚合排序。
    

---

### 8.7 去重统计

**场景**

统计当日支付用户数。

**推荐 SQL**

```sql
SELECT
    COUNT(DISTINCT user_id) AS paid_user_count
FROM order_info
WHERE pay_status = 1
  AND paid_at >= '2026-05-15 00:00:00'
  AND paid_at <  '2026-05-16 00:00:00';
```

**注意事项**

- `COUNT(DISTINCT)` 在大表上成本较高。
    
- 高频指标建议预聚合。
    
- 可按小时分桶后再汇总。
    

---

### 8.8 多指标聚合

**场景**

运营看板统计订单数、支付数、取消数、支付金额。

**推荐 SQL**

```sql
SELECT
    COUNT(*) AS order_count,
    SUM(CASE WHEN pay_status = 1 THEN 1 ELSE 0 END) AS paid_order_count,
    SUM(CASE WHEN order_status = 90 THEN 1 ELSE 0 END) AS canceled_order_count,
    COALESCE(SUM(CASE WHEN pay_status = 1 THEN pay_amount ELSE 0 END), 0) AS paid_amount
FROM order_info
WHERE created_at >= '2026-05-15 00:00:00'
  AND created_at <  '2026-05-16 00:00:00'
  AND is_deleted = 0;
```

**注意事项**

- 适合小范围实时看板。
    
- 多指标聚合比多次查询更稳定。
    
- 大规模看板应使用汇总表或数仓。
    

---

### 8.9 窗口函数示例

**场景**

查询每个用户最近一笔订单。

**推荐 SQL**

```sql
SELECT
    t.user_id,
    t.order_no,
    t.order_status,
    t.pay_amount,
    t.created_at
FROM (
    SELECT
        user_id,
        order_no,
        order_status,
        pay_amount,
        created_at,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY created_at DESC, id DESC
        ) AS row_num
    FROM order_info
    WHERE created_at >= '2026-05-01 00:00:00'
      AND created_at <  '2026-06-01 00:00:00'
      AND is_deleted = 0
) t
WHERE t.row_num = 1;
```

**注意事项**

- MySQL 8.x 支持窗口函数。
    
- 大数据量窗口函数建议在分析库执行。
    
- 生产核心链路慎用复杂窗口查询。
    

---

## 9. 数据修复与运维 SQL

### 9.1 查询异常数据

**场景**

查询已支付但支付时间为空的订单。

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    pay_status,
    paid_at,
    updated_at
FROM order_info
WHERE pay_status = 1
  AND paid_at IS NULL
ORDER BY id ASC
LIMIT 100;
```

**注意事项**

- 先查异常范围，再修复。
    
- 每次只查看有限数量。
    
- 修复前确认异常原因。
    

---

### 9.2 修复状态字段

**场景**

修复已支付但订单状态仍为待支付的数据。

**推荐 SQL**

```sql
UPDATE order_info
SET
    order_status = 20,
    updated_at = NOW(3)
WHERE pay_status = 1
  AND order_status = 10
  AND paid_at IS NOT NULL
ORDER BY id ASC
LIMIT 500;
```

**注意事项**

- 分批修复。
    
- 修复前备份异常数据。
    
- 修复后再次查询校验。
    

---

### 9.3 修复冗余字段

**场景**

订单表冗余的用户昵称与用户表不一致。

**推荐 SQL**

```sql
UPDATE order_info o
INNER JOIN user_info u ON u.id = o.user_id
SET
    o.user_nickname = u.nickname,
    o.updated_at = NOW(3)
WHERE o.user_nickname <> u.nickname
  AND o.created_at >= '2026-05-01 00:00:00'
  AND o.created_at <  '2026-06-01 00:00:00'
LIMIT 500;
```

**注意事项**

- MySQL 对 `UPDATE JOIN LIMIT` 的行为要结合版本验证。
    
- 更稳妥方式是先查出 ID，再按 ID 分批更新。
    
- 冗余字段修复要确认是否允许历史快照变化。
    

---

### 9.4 数据备份表

**场景**

修复前备份异常订单。

**推荐 SQL**

```sql
CREATE TABLE order_info_fix_bak_20260515 AS
SELECT
    id,
    order_no,
    user_id,
    order_status,
    pay_status,
    paid_at,
    updated_at
FROM order_info
WHERE pay_status = 1
  AND order_status = 10
  AND paid_at IS NOT NULL;
```

**注意事项**

- 备份表名带业务名和日期。
    
- 备份字段覆盖回滚所需字段。
    
- 大表备份不要无条件全表复制。
    

---

### 9.5 回滚备份数据

**场景**

根据备份表恢复订单状态。

**推荐 SQL**

```sql
UPDATE order_info o
INNER JOIN order_info_fix_bak_20260515 b ON b.id = o.id
SET
    o.order_status = b.order_status,
    o.pay_status = b.pay_status,
    o.paid_at = b.paid_at,
    o.updated_at = NOW(3)
WHERE o.id = b.id;
```

**注意事项**

- 回滚前先确认备份表数据准确。
    
- 回滚也要分批执行。
    
- 回滚后保留操作日志。
    

---

### 9.6 批量补数据

**场景**

给历史用户补默认积分账户。

**推荐 SQL**

```sql
INSERT INTO user_point_account (
    user_id,
    available_points,
    frozen_points,
    account_status,
    created_at,
    updated_at
)
SELECT
    u.id AS user_id,
    0 AS available_points,
    0 AS frozen_points,
    1 AS account_status,
    NOW(3) AS created_at,
    NOW(3) AS updated_at
FROM user_info u
LEFT JOIN user_point_account a ON a.user_id = u.id
WHERE a.user_id IS NULL
  AND u.is_deleted = 0
ORDER BY u.id ASC
LIMIT 1000;
```

**注意事项**

- 目标表 `user_id` 建唯一索引。
    
- 补数据脚本可重复执行。
    
- 大批量补数据按 ID 分段。
    

---

### 9.7 分批更新大表

**场景**

给历史订单补充渠道字段。

**推荐 SQL**

```sql
UPDATE order_info
SET
    order_source = 10,
    updated_at = NOW(3)
WHERE id > 10000000
  AND id <= 10001000
  AND order_source IS NULL;
```

**注意事项**

- 按主键区间分批。
    
- 每批控制在 500～5000 行。
    
- 避免一次性更新全表。
    

---

### 9.8 校验数据一致性

**场景**

校验订单金额是否等于订单明细金额之和。

**推荐 SQL**

```sql
SELECT
    o.id,
    o.order_no,
    o.total_amount,
    t.item_total_amount
FROM order_info o
INNER JOIN (
    SELECT
        order_no,
        SUM(quantity * sale_price) AS item_total_amount
    FROM order_item
    WHERE order_no IN (
        'ORD202605150001',
        'ORD202605150002',
        'ORD202605150003'
    )
    GROUP BY order_no
) t ON t.order_no = o.order_no
WHERE o.total_amount <> t.item_total_amount;
```

**注意事项**

- 金额比较注意精度。
    
- 大范围校验应分批。
    
- 差异数据先落表，再人工复核。
    

---

### 9.9 查找重复数据

**场景**

查找重复领取优惠券的用户。

**推荐 SQL**

```sql
SELECT
    user_id,
    coupon_id,
    COUNT(*) AS duplicate_count
FROM user_coupon
WHERE received_at >= '2026-05-01 00:00:00'
  AND received_at <  '2026-06-01 00:00:00'
GROUP BY user_id, coupon_id
HAVING COUNT(*) > 1
ORDER BY duplicate_count DESC
LIMIT 100;
```

**注意事项**

- 重复数据查询要带时间范围。
    
- 查出后再定位具体记录。
    
- 后续必须补唯一索引或幂等逻辑。
    

---

### 9.10 删除重复数据

**场景**

保留最早一条用户优惠券记录，删除重复记录。

**推荐 SQL**

先备份：

```sql
CREATE TABLE user_coupon_duplicate_bak_20260515 AS
SELECT
    uc.id,
    uc.user_id,
    uc.coupon_id,
    uc.created_at
FROM user_coupon uc
INNER JOIN (
    SELECT
        user_id,
        coupon_id,
        MIN(id) AS keep_id
    FROM user_coupon
    GROUP BY user_id, coupon_id
    HAVING COUNT(*) > 1
) d ON d.user_id = uc.user_id
   AND d.coupon_id = uc.coupon_id
WHERE uc.id <> d.keep_id;
```

再删除：

```sql
DELETE uc
FROM user_coupon uc
INNER JOIN user_coupon_duplicate_bak_20260515 b ON b.id = uc.id;
```

**注意事项**

- 删除前必须备份。
    
- 保留规则必须明确，例如保留最早或最新。
    
- 删除后添加唯一索引防止再次发生。
    

---

## 10. 表结构与字段维护 SQL

### 10.1 创建业务表

**场景**

创建订单表。

**推荐 SQL**

```sql
CREATE TABLE order_info (
    id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键ID',
    order_no VARCHAR(64) NOT NULL COMMENT '订单号',
    user_id BIGINT UNSIGNED NOT NULL COMMENT '用户ID',
    order_status TINYINT NOT NULL COMMENT '订单状态：10待支付，20已支付，90已取消',
    pay_status TINYINT NOT NULL COMMENT '支付状态：0未支付，1已支付',
    total_amount DECIMAL(18, 2) NOT NULL COMMENT '订单总金额',
    pay_amount DECIMAL(18, 2) NOT NULL COMMENT '实付金额',
    is_deleted TINYINT NOT NULL DEFAULT 0 COMMENT '是否删除：0否，1是',
    created_at DATETIME(3) NOT NULL COMMENT '创建时间',
    updated_at DATETIME(3) NOT NULL COMMENT '更新时间',
    PRIMARY KEY (id),
    UNIQUE KEY uk_order_no (order_no),
    KEY idx_user_id_id (user_id, id),
    KEY idx_status_created_at (order_status, created_at)
) ENGINE = InnoDB
  DEFAULT CHARSET = utf8mb4
  COLLATE = utf8mb4_0900_ai_ci
  COMMENT = '订单表';
```

**注意事项**

- 必须有主键。
    
- 金额使用 `DECIMAL`。
    
- 字段、索引、表必须有清晰语义。
    

---

### 10.2 添加字段

**场景**

订单表新增订单来源字段。

**推荐 SQL**

```sql
ALTER TABLE order_info
ADD COLUMN order_source TINYINT NOT NULL DEFAULT 0 COMMENT '订单来源：0未知，10App，20Web'
AFTER pay_status;
```

**注意事项**

- 大表变更需评估锁表风险。
    
- 添加字段优先设置默认值。
    
- 生产建议使用在线 DDL 工具或低峰执行。
    

---

### 10.3 修改字段

**场景**

扩大商品名称长度。

**推荐 SQL**

```sql
ALTER TABLE product_info
MODIFY COLUMN product_name VARCHAR(256) NOT NULL COMMENT '商品名称';
```

**注意事项**

- 修改字段前确认兼容历史数据。
    
- 缩短字段长度风险较高。
    
- 生产执行前必须在预发验证。
    

---

### 10.4 添加索引

**场景**

订单表新增支付时间查询索引。

**推荐 SQL**

```sql
ALTER TABLE order_info
ADD INDEX idx_pay_status_paid_at (pay_status, paid_at);
```

**注意事项**

- 索引名表达字段含义。
    
- 添加索引前先确认慢 SQL 和查询频率。
    
- 不要为低频查询随意加索引。
    

---

### 10.5 删除索引

**场景**

删除无用索引。

**推荐 SQL**

```sql
ALTER TABLE order_info
DROP INDEX idx_old_status;
```

**注意事项**

- 删除前确认没有 SQL 依赖。
    
- 观察线上执行计划和慢 SQL。
    
- 先在预发环境验证。
    

---

### 10.6 创建唯一索引

**场景**

用户优惠券防止重复领取。

**推荐 SQL**

```sql
ALTER TABLE user_coupon
ADD UNIQUE KEY uk_user_coupon (user_id, coupon_id);
```

**注意事项**

- 添加前必须清理重复数据。
    
- 唯一索引是幂等写入的重要保障。
    
- 软删除场景要考虑唯一约束设计。
    

---

### 10.7 创建联合索引

**场景**

后台订单按商户、状态、时间查询。

**推荐 SQL**

```sql
ALTER TABLE payment_order
ADD INDEX idx_merchant_status_time_id (
    merchant_id,
    payment_status,
    created_at,
    id
);
```

**注意事项**

- 联合索引字段顺序要匹配查询条件。
    
- 等值条件通常放前面。
    
- 索引不是越多越好，会影响写入性能。
    

---

### 10.8 查看表结构

**场景**

确认订单表字段定义。

**推荐 SQL**

```sql
SHOW CREATE TABLE order_info;
```

或：

```sql
DESC order_info;
```

**注意事项**

- 以 `SHOW CREATE TABLE` 为准。
    
- 上线前确认字符集、索引、字段类型。
    
- 表结构变更要纳入版本管理。
    

---

### 10.9 查看索引

**场景**

确认订单表索引。

**推荐 SQL**

```sql
SHOW INDEX FROM order_info;
```

**注意事项**

- 关注索引字段顺序。
    
- 检查是否有重复索引。
    
- 慢 SQL 优化前先看现有索引。
    

---

### 10.10 安全变更表结构示例

**场景**

给大表新增可为空字段，再逐步补数据。

**推荐 SQL**

第一步：新增字段。

```sql
ALTER TABLE order_info
ADD COLUMN invoice_status TINYINT NULL COMMENT '开票状态：0未开票，1已开票';
```

第二步：分批补数据。

```sql
UPDATE order_info
SET
    invoice_status = 0,
    updated_at = NOW(3)
WHERE id > 10000000
  AND id <= 10005000
  AND invoice_status IS NULL;
```

第三步：确认无空值。

```sql
SELECT
    COUNT(*) AS null_count
FROM order_info
WHERE invoice_status IS NULL;
```

第四步：改为非空。

```sql
ALTER TABLE order_info
MODIFY COLUMN invoice_status TINYINT NOT NULL DEFAULT 0 COMMENT '开票状态：0未开票，1已开票';
```

**注意事项**

- 大表变更采用分阶段策略。
    
- 先兼容代码，再补数据，最后收紧约束。
    
- 不要一次性完成高风险 DDL。
    

---

## 11. 常见反例与推荐写法

### 11.1 SELECT *

**场景**

查询订单详情。

**反例**

```sql
SELECT *
FROM order_info
WHERE order_no = 'ORD202605150001';
```

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    order_status,
    pay_status,
    pay_amount,
    created_at
FROM order_info
WHERE order_no = 'ORD202605150001'
LIMIT 1;
```

**注意事项**

- 避免返回无用字段。
    
- 防止表结构变更影响代码。
    
- 有利于覆盖索引。
    

---

### 11.2 无 WHERE 更新

**场景**

错误更新所有用户状态。

**反例**

```sql
UPDATE user_info
SET user_status = 2;
```

**推荐 SQL**

```sql
UPDATE user_info
SET
    user_status = 2,
    updated_at = NOW(3)
WHERE id = 10001
  AND user_status = 1
  AND is_deleted = 0;
```

**注意事项**

- 生产禁止无条件更新。
    
- 更新前先 `SELECT` 确认范围。
    
- 执行工具应开启安全更新模式。
    

---

### 11.3 无 WHERE 删除

**场景**

误删整张表。

**反例**

```sql
DELETE FROM order_info;
```

**推荐 SQL**

```sql
DELETE FROM order_info
WHERE id = 10000001
  AND order_no = 'ORD202605150001'
LIMIT 1;
```

**注意事项**

- 生产禁止无条件删除。
    
- 优先软删除。
    
- 重要表删除前必须备份。
    

---

### 11.4 深分页

**场景**

后台订单列表翻到很深页码。

**反例**

```sql
SELECT
    id,
    order_no,
    user_id,
    created_at
FROM order_info
ORDER BY id DESC
LIMIT 1000000, 20;
```

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id,
    created_at
FROM order_info
WHERE id < 80000000
ORDER BY id DESC
LIMIT 20;
```

**注意事项**

- App 和导出使用游标分页。
    
- 后台限制最大页码。
    
- 深分页不是靠加机器硬扛。
    

---

### 11.5 索引列上使用函数

**场景**

按日期查订单。

**反例**

```sql
SELECT
    id,
    order_no,
    created_at
FROM order_info
WHERE DATE(created_at) = '2026-05-15';
```

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    created_at
FROM order_info
WHERE created_at >= '2026-05-15 00:00:00'
  AND created_at <  '2026-05-16 00:00:00';
```

**注意事项**

- 条件中的索引列保持原样。
    
- 日期边界由程序计算更清晰。
    
- 统计字段可单独冗余 `stat_date`。
    

---

### 11.6 隐式类型转换

**场景**

根据订单号查询订单。

**反例**

```sql
SELECT
    id,
    order_no,
    user_id
FROM order_info
WHERE order_no = 202605150001;
```

**推荐 SQL**

```sql
SELECT
    id,
    order_no,
    user_id
FROM order_info
WHERE order_no = '202605150001';
```

**注意事项**

- 字符串字段必须传字符串。
    
- Java DTO 类型要与数据库一致。
    
- 参数绑定不要偷懒。
    

---

### 11.7 前置模糊查询

**场景**

搜索商品。

**反例**

```sql
SELECT
    id,
    product_name
FROM product_info
WHERE product_name LIKE '%键盘%'
LIMIT 20;
```

**推荐 SQL**

```sql
SELECT
    id,
    product_name
FROM product_info
WHERE product_name LIKE '键盘%'
LIMIT 20;
```

**更推荐**

```sql
SELECT
    product_id,
    product_name
FROM product_search_index
WHERE keyword = '键盘'
ORDER BY score DESC
LIMIT 20;
```

**注意事项**

- 包含式搜索应使用搜索系统。
    
- MySQL 业务库不适合承担复杂搜索。
    
- 后台低频模糊查询也要限制范围。
    

---

### 11.8 大事务

**场景**

一次性更新 100 万条历史订单。

**反例**

```sql
UPDATE order_info
SET
    order_source = 10,
    updated_at = NOW(3)
WHERE order_source IS NULL;
```

**推荐 SQL**

```sql
UPDATE order_info
SET
    order_source = 10,
    updated_at = NOW(3)
WHERE id > 10000000
  AND id <= 10005000
  AND order_source IS NULL;
```

**注意事项**

- 大事务拆小批。
    
- 每批提交。
    
- 避免长时间持锁。
    

---

### 11.9 一次性大批量更新

**场景**

批量关闭超时订单。

**反例**

```sql
UPDATE order_info
SET
    order_status = 90,
    updated_at = NOW(3)
WHERE order_status = 10
  AND created_at < DATE_SUB(NOW(3), INTERVAL 30 MINUTE);
```

**推荐 SQL**

```sql
UPDATE order_info
SET
    order_status = 90,
    cancel_reason = 'PAY_TIMEOUT',
    canceled_at = NOW(3),
    updated_at = NOW(3)
WHERE order_status = 10
  AND pay_status = 0
  AND created_at < DATE_SUB(NOW(3), INTERVAL 30 MINUTE)
ORDER BY id ASC
LIMIT 500;
```

**注意事项**

- 定时任务循环执行。
    
- 控制每批影响行数。
    
- 记录每批执行结果。
    

---

### 11.10 非幂等写入

**场景**

支付回调重复通知导致重复入账。

**反例**

```sql
UPDATE user_account
SET
    available_balance = available_balance + 99.90,
    updated_at = NOW(3)
WHERE user_id = 10001;
```

**推荐 SQL**

先插入幂等记录：

```sql
INSERT INTO payment_process_record (
    payment_no,
    order_no,
    process_status,
    created_at,
    updated_at
) VALUES (
    'PAY202605150001',
    'ORD202605150001',
    0,
    NOW(3),
    NOW(3)
);
```

再处理账户：

```sql
UPDATE user_account
SET
    available_balance = available_balance + 99.90,
    updated_at = NOW(3)
WHERE user_id = 10001
  AND account_status = 1;
```

最后更新处理状态：

```sql
UPDATE payment_process_record
SET
    process_status = 1,
    updated_at = NOW(3)
WHERE payment_no = 'PAY202605150001'
  AND process_status = 0;
```

**注意事项**

- `payment_no` 必须唯一。
    
- 重复请求不能重复执行业务副作用。
    
- 金额、库存、权益发放必须做幂等。
    

---

# 生产 SQL 编写检查清单

## 查询 SQL

-  是否禁止了 `SELECT *`？
    
-  是否只查询业务需要字段？
    
-  `WHERE` 条件是否命中索引？
    
-  字段类型和入参类型是否一致？
    
-  是否避免在索引列上使用函数？
    
-  是否避免 `LIKE '%xxx%'`？
    
-  是否限制了返回条数？
    
-  是否避免深分页？
    
-  排序字段是否有合适索引？
    
-  大范围统计是否避开核心业务库？
    

## 新增 SQL

-  是否明确写出插入字段？
    
-  是否包含 `created_at / updated_at`？
    
-  关键业务是否有唯一键防重复？
    
-  批量插入是否控制单批数量？
    
-  是否避免明文写入敏感数据？
    
-  初始化 SQL 是否可重复执行？
    

## 更新 SQL

-  是否有明确 `WHERE` 条件？
    
-  是否禁止无条件更新？
    
-  状态流转是否带前置状态？
    
-  库存扣减是否在 SQL 中判断余量？
    
-  余额扣减是否在 SQL 中判断余额充足？
    
-  是否维护 `updated_at`？
    
-  批量更新是否使用分批策略？
    
-  Java 是否检查影响行数？
    

## 删除 SQL

-  是否优先软删除？
    
-  物理删除前是否备份？
    
-  是否有明确条件和 `LIMIT`？
    
-  是否禁止无条件删除？
    
-  大表删除是否分批？
    
-  删除后是否有校验 SQL？
    

## 事务 SQL

-  事务是否足够短？
    
-  事务内是否避免远程调用？
    
-  加锁查询是否命中索引？
    
-  多资源操作是否有固定顺序？
    
-  是否有幂等记录？
    
-  是否有流水表记录资金、库存、状态变化？
    
-  异常时是否明确回滚？
    

## 索引与性能

-  是否根据高频查询设计联合索引？
    
-  联合索引顺序是否匹配查询条件？
    
-  是否存在重复索引？
    
-  是否避免低选择性字段单独建索引？
    
-  是否避免为了覆盖索引加入大字段？
    
-  上线前是否执行 `EXPLAIN` 验证？
    

## 数据修复与运维

-  是否先 `SELECT` 确认影响范围？
    
-  是否创建备份表？
    
-  是否按主键或时间分批？
    
-  是否记录修复 SQL、执行人、执行时间？
    
-  是否提供回滚 SQL？
    
-  是否在低峰期执行？
    
-  是否修复后再次校验数据一致性？