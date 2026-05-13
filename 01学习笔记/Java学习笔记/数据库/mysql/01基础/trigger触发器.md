## 1. 先给结论

**MySQL 触发器 Trigger，本质是“绑定在表上的自动执行 SQL 逻辑”。**

当某张表发生 `INSERT`、`UPDATE`、`DELETE` 时，MySQL 可以自动执行一段预先定义好的 SQL。它常用于：

- 自动记录审计日志；
    
- 自动记录数据变更轨迹；
    
- 做简单的数据同步；
    
- 做数据库层面的兜底约束。
    

但在真实项目中，**触发器要谨慎使用**。因为它是**隐式执行**的，Java 代码里看不见，容易导致排查困难、写入变慢、锁冲突扩大、业务逻辑分散。  
所以生产经验一般是：

> **简单、稳定、审计类、兜底类逻辑可以考虑触发器；复杂业务流程、外部系统调用、高并发核心链路，优先放在 Java Service 层。**

---

# 2. 用订单状态流转日志讲清触发器

## 2.1 业务场景

订单表 `t_order` 中有一个订单状态字段 `status`。

当订单状态从：

- `CREATED` → `PAID`
    
- `PAID` → `SHIPPED`
    
- `SHIPPED` → `FINISHED`
    

发生变化时，希望自动记录一条状态流转日志。

这个需求很适合用触发器做教学案例，因为它是典型的：

> **主表状态变化 → 自动写一条变更日志。**

---

## 2.2 建表 SQL

```sql
CREATE TABLE t_order (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_no VARCHAR(64) NOT NULL,
    user_id BIGINT NOT NULL,
    status VARCHAR(32) NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

CREATE TABLE t_order_status_log (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    order_no VARCHAR(64) NOT NULL,
    old_status VARCHAR(32) NOT NULL,
    new_status VARCHAR(32) NOT NULL,
    changed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    remark VARCHAR(255) DEFAULT NULL
);
```

插入一条订单：

```sql
INSERT INTO t_order (order_no, user_id, status, amount)
VALUES ('ORDER_1001', 10001, 'CREATED', 199.00);
```

---

## 2.3 创建触发器 SQL

```sql
DELIMITER $$

CREATE TRIGGER trg_order_status_change_log
AFTER UPDATE ON t_order
FOR EACH ROW
BEGIN
    IF OLD.status <> NEW.status THEN
        INSERT INTO t_order_status_log (
            order_id,
            order_no,
            old_status,
            new_status,
            changed_at,
            remark
        )
        VALUES (
            NEW.id,
            NEW.order_no,
            OLD.status,
            NEW.status,
            NOW(),
            'order status changed automatically by trigger'
        );
    END IF;
END$$

DELIMITER ;
```

这里有几个关键点：

```sql
AFTER UPDATE ON t_order
```

表示在 `t_order` 表执行 `UPDATE` 之后触发。

```sql
FOR EACH ROW
```

表示每更新一行，触发器执行一次。

```sql
OLD.status
NEW.status
```

表示更新前后的字段值。

```sql
IF OLD.status <> NEW.status THEN
```

表示只有状态真的发生变化时，才写日志。

---

## 2.4 更新订单状态的测试 SQL

```sql
UPDATE t_order
SET status = 'PAID'
WHERE order_no = 'ORDER_1001';
```

再更新一次：

```sql
UPDATE t_order
SET status = 'SHIPPED'
WHERE order_no = 'ORDER_1001';
```

查询订单：

```sql
SELECT id, order_no, user_id, status, amount
FROM t_order
WHERE order_no = 'ORDER_1001';
```

结果类似：

```text
id | order_no   | user_id | status  | amount
---+------------+---------+---------+--------
1  | ORDER_1001 | 10001   | SHIPPED | 199.00
```

查询状态流转日志：

```sql
SELECT order_id, order_no, old_status, new_status, changed_at, remark
FROM t_order_status_log
WHERE order_no = 'ORDER_1001'
ORDER BY id;
```

结果类似：

```text
order_id | order_no   | old_status | new_status | changed_at          | remark
---------+------------+------------+------------+---------------------+--------------------------------------------
1        | ORDER_1001 | CREATED    | PAID       | 2026-05-13 23:20:00 | order status changed automatically by trigger
1        | ORDER_1001 | PAID       | SHIPPED    | 2026-05-13 23:21:00 | order status changed automatically by trigger
```

---

## 2.5 这个案例里触发器发挥了什么作用？

触发器把“订单状态变化后写日志”这件事从 Java 代码里抽到了数据库层。

好处是：

```text
无论是 Java Service 更新订单，
还是后台 SQL 手工修复订单，
还是其他系统直接修改订单表，
只要 t_order.status 发生变化，
状态流转日志都会自动记录。
```

所以触发器适合做这种**数据变更感知型、审计型、兜底型**逻辑。

但注意：

> 它适合记录日志，不适合承载完整订单状态机。

订单能不能从 `CREATED` 改成 `SHIPPED`，这种复杂业务规则，最好还是放在 Java Service 层做校验。

---

# 3. 触发器适合解决什么问题

## 3.1 审计日志

比如用户信息、订单金额、账户状态发生变化后，自动记录修改前后的值。

适合原因：

> 审计日志强依赖数据库变更，而且通常逻辑简单、稳定，适合做数据库层兜底。

---

## 3.2 数据变更记录

比如商品价格变化后，自动记录价格变更历史。

适合原因：

> 只要核心字段发生变化，就记录一条历史数据，不依赖复杂业务流程。

---

## 3.3 状态流转记录

比如订单状态、审批状态、工单状态发生变化后，记录从旧状态到新状态的流转过程。

适合原因：

> 状态字段变化本身就是数据库事件，触发器可以保证不同入口修改数据时都能留下轨迹。

---

## 3.4 简单数据同步

比如主表插入一条数据后，自动向影子表、统计表、历史表插入一条记录。

适合原因：

> 同步逻辑非常简单，并且只发生在数据库内部，不涉及 MQ、RPC、HTTP 等外部调用。

---

## 3.5 兜底的数据一致性约束

比如删除主记录时，自动清理某些强关联的辅助表数据。

适合原因：

> 可以作为数据库层的最后一道防线，但不建议把复杂级联删除逻辑都堆到触发器里。

---

# 4. 什么时候适合用触发器

## 4.1 逻辑强依赖数据库数据变更

比如：

```text
只要订单状态变了，就必须记录日志。
只要商品价格变了，就必须记录历史价格。
只要用户等级变了，就必须记录等级变更记录。
```

这种逻辑的触发条件不是某个接口，而是**数据本身发生变化**。

适合触发器。

---

## 4.2 希望无论哪个入口修改数据，都能统一生效

如果数据可能来自多个入口：

- Java 后台管理系统；
    
- 定时任务；
    
- 数据修复 SQL；
    
- 第三方系统写库；
    
- 运维脚本。
    

那么只在 Java Service 里写日志可能不够稳。

触发器的优势是：

> 只要最终修改了这张表，它就能执行。

---

## 4.3 主要用于审计、日志、兜底约束

触发器最适合做这种低侵入的辅助逻辑：

```text
主业务：更新订单状态
辅助逻辑：记录状态变更日志
```

而不是：

```text
主业务：订单支付完成后，发优惠券、发 MQ、扣库存、通知仓库、调用风控
```

后者必须放业务代码。

---

## 4.4 逻辑简单、稳定、低频变化

适合触发器的逻辑一般应该满足：

```text
逻辑短；
SQL 简单；
不经常变；
没有复杂分支；
不依赖外部系统；
不会产生大规模额外写入。
```

比如：

```sql
IF OLD.status <> NEW.status THEN
    INSERT INTO log_table ...
END IF;
```

这种可以接受。

如果触发器里开始写大量 `IF ELSE`、多表查询、多表更新，就要警惕了。

---

## 4.5 不涉及外部系统调用

MySQL 触发器不能也不应该做这些事：

```text
发 MQ；
调 HTTP 接口；
调 RPC 服务；
写 Redis；
调用第三方支付接口；
触发异步任务。
```

这些都应该放在 Java 业务代码里。

---

# 5. 什么时候应该用业务代码，而不是触发器

## 5.1 复杂业务流程应该放在 Service 层

例如订单支付完成：

```text
校验订单状态；
校验支付金额；
更新订单状态；
扣减库存；
生成账单；
发送 MQ；
通知仓库；
记录操作日志。
```

这种流程涉及多个步骤、多个系统、异常处理和补偿机制。

应该放在 Java Service 层：

```java
@Transactional(rollbackFor = Exception.class)
public void paySuccess(String orderNo, BigDecimal paidAmount) {
    Order order = orderMapper.selectByOrderNo(orderNo);

    if (!"CREATED".equals(order.getStatus())) {
        throw new BusinessException("订单状态不允许支付");
    }

    if (order.getAmount().compareTo(paidAmount) != 0) {
        throw new BusinessException("支付金额不一致");
    }

    orderMapper.updateStatus(order.getId(), "PAID");

    orderStatusLogMapper.insert(new OrderStatusLog(
            order.getId(),
            order.getStatus(),
            "PAID",
            "支付成功"
    ));

    // 后续通过 MQ 或事务消息处理
    messagePublisher.publishOrderPaidEvent(order.getId());
}
```

这类代码虽然比触发器“显式”一些，但它的优势是：

- 逻辑清晰；
    
- 可测试；
    
- 可打日志；
    
- 可监控；
    
- 可灰度；
    
- 可重试；
    
- 可做异常处理。
    

---

## 5.2 需要调用 MQ、RPC、HTTP 接口时，不能放触发器

比如订单支付后要发消息：

```java
messagePublisher.publishOrderPaidEvent(orderId);
```

这不能放触发器里。

触发器只能执行数据库内部 SQL。即使某些数据库能力可以间接做外部调用，也不建议这么设计。

原因是：

> 数据库应该负责数据存储和简单约束，不应该承担业务编排和系统集成职责。

---

## 5.3 需要清晰事务边界、异常处理、重试机制时，应放业务代码

触发器和当前 SQL 处在同一个事务里。

例如：

```sql
UPDATE t_order SET status = 'PAID' WHERE id = 1;
```

如果触发器插入日志失败，那么这个 `UPDATE` 也会失败。

这在某些兜底场景是好事，但在复杂业务中会变成风险：

```text
主流程更新失败，到底是订单更新失败？
还是触发器写日志失败？
Java 代码不一定直观看出来。
```

业务代码更适合控制：

- 哪些异常需要回滚；
    
- 哪些失败可以降级；
    
- 哪些操作可以异步重试；
    
- 哪些日志失败不能影响主链路。
    

---

## 5.4 需要测试、灰度、监控、日志追踪时，应放业务代码

Service 层代码可以做：

```java
log.info("订单状态变更, orderId={}, oldStatus={}, newStatus={}", orderId, oldStatus, newStatus);
```

也可以接入：

- 单元测试；
    
- 集成测试；
    
- 链路追踪；
    
- 灰度发布；
    
- 指标监控；
    
- 告警系统。
    

但触发器藏在数据库里，Java 代码层面很难感知。

所以：

> 越是需要工程化治理的逻辑，越应该放业务代码。

---

## 5.5 高并发核心链路应避免触发器增加隐式开销

比如秒杀扣库存：

```sql
UPDATE stock
SET available_stock = available_stock - 1
WHERE sku_id = 1001
  AND available_stock > 0;
```

如果这个表上还有触发器，每次扣库存都额外写库存流水、更新统计表、查询其他表，就会增加写入耗时。

高并发场景下，这种额外开销会放大成：

```text
TPS 下降；
锁持有时间变长；
慢 SQL 增加；
死锁概率升高。
```

这种核心链路通常应该尽量短，把非核心逻辑放到异步消息里。

---

## 5.6 频繁变化的业务规则不适合写进触发器

例如营销规则：

```text
新用户首单送券；
会员等级不同返积分不同；
不同渠道订单走不同审核逻辑；
节假日活动规则临时调整。
```

这类规则经常变，不适合写在数据库触发器中。

原因很简单：

> 业务规则频繁变化，就应该放在应用层，方便发布、测试、回滚和灰度。

---

# 6. 触发器的生产风险

## 6.1 隐式执行，排查困难

Java 代码里只看到：

```java
orderMapper.updateStatus(orderId, "PAID");
```

但数据库里可能实际发生了：

```text
更新订单表；
插入订单状态日志；
更新统计表；
修改其他关联表。
```

如果开发人员不知道有触发器，很容易误判问题。

典型事故：

> 明明代码里只更新一张表，结果线上写入很慢，排查半天才发现表上挂了多个触发器。

---

## 6.2 增加写入耗时

原本一条 SQL：

```sql
UPDATE t_order SET status = 'PAID' WHERE id = 1;
```

如果触发器里还执行：

```sql
INSERT INTO t_order_status_log ...
```

那么一次更新就变成了：

```text
更新主表 + 写日志表
```

写入耗时自然会上升。

如果触发器里还有复杂查询或多表更新，影响更明显。

---

## 6.3 可能扩大锁冲突

假设更新订单表时，触发器还更新统计表：

```sql
UPDATE t_order_stat
SET paid_count = paid_count + 1
WHERE stat_date = CURRENT_DATE;
```

高并发下，很多订单更新都会争抢同一行统计记录。

这会导致：

```text
订单更新锁等待；
统计表热点行竞争；
事务持锁时间变长；
死锁概率增加。
```

这类统计逻辑更适合异步 MQ 或离线计算。

---

## 6.4 和事务回滚绑定

触发器和触发它的 SQL 在同一个事务里。

例如：

```sql
START TRANSACTION;

UPDATE t_order
SET status = 'PAID'
WHERE id = 1;

ROLLBACK;
```

如果 `UPDATE` 回滚，触发器插入的日志也会回滚。

这点要特别注意。

如果你希望保留“尝试修改但失败”的审计日志，触发器不一定合适，因为它会随着主事务一起回滚。

---

## 6.5 业务逻辑分散在数据库中

如果一部分逻辑在 Java：

```text
OrderService
PaymentService
InventoryService
```

另一部分逻辑在数据库触发器：

```text
trg_order_status_change_log
trg_stock_change_log
trg_user_level_change
```

系统理解成本会变高。

新人接手时，很容易只看代码，不看数据库对象，导致遗漏关键逻辑。

---

## 6.6 数据迁移、批量修复时容易误触发

比如运维执行：

```sql
UPDATE t_order
SET status = 'CLOSED'
WHERE created_at < '2025-01-01'
  AND status = 'CREATED';
```

如果影响 100 万行，触发器可能也会插入 100 万条日志。

结果可能是：

```text
批量修复变慢；
日志表暴增；
binlog 暴增；
主从延迟；
磁盘压力上升。
```

所以有触发器的表，做批量操作前必须先评估触发器影响。

---

## 6.7 ORM / MyBatis 代码层面不容易感知

MyBatis 里可能只是：

```xml
<update id="updateStatus">
    UPDATE t_order
    SET status = #{status}
    WHERE id = #{id}
</update>
```

从代码看，只有一条 `UPDATE`。

但实际数据库层还自动写了日志表。

这会带来两个问题：

1. 代码阅读者不知道副作用；
    
2. 性能评估时容易低估真实写入成本。
    

所以触发器一定要配套文档和命名规范。

---

# 7. 面试回答模板

可以这样回答：

> MySQL 触发器是绑定在表上的自动执行逻辑，当表发生 `INSERT`、`UPDATE`、`DELETE` 时，可以自动执行一段 SQL。
> 
> 在实际项目里，我不会把复杂业务流程放进触发器。触发器更适合做审计日志、数据变更记录、状态流转日志、简单同步或者兜底约束。比如订单状态从 `CREATED` 变成 `PAID` 时，可以用 `AFTER UPDATE` 触发器自动写一条订单状态流转日志，这样无论是 Java 服务、后台系统还是运维 SQL 修改订单状态，都能统一留下记录。
> 
> 但触发器要慎用。它是隐式执行的，Java 代码层面不容易感知，会增加写入耗时，也可能扩大锁冲突。触发器和当前 SQL 在同一个事务里，如果触发器失败，主 SQL 也会失败；如果事务回滚，触发器写入的数据也会回滚。
> 
> 所以我的取舍是：简单、稳定、审计型、兜底型逻辑可以考虑触发器；涉及复杂业务流程、MQ、RPC、HTTP 调用、异常处理、重试、灰度、监控、高并发核心链路的逻辑，应该放在 Java Service 层，而不是放在数据库触发器里。

---

# 总结

|问题|判断|
|---|---|
|触发器是什么？|表数据变化时自动执行的数据库逻辑|
|适合做什么？|审计日志、状态流转、变更记录、简单同步、兜底约束|
|不适合做什么？|复杂业务流程、外部系统调用、高并发核心逻辑|
|最大优点|不依赖 Java 入口，只要数据变更就能统一生效|
|最大缺点|隐式执行，排查困难，性能和锁风险不直观|
|生产建议|能不用就不用；要用就限制在简单、稳定、低副作用场景|

**关键词：**

```text
隐式执行
审计日志
状态流转
事务绑定
锁冲突
业务代码优先
数据库兜底
Service 层编排
```