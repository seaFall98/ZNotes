# 1. 场景：订单表分 2 库 × 2 表

假设有一个订单系统，订单量持续增长，单表 `t_order` 未来可能到几千万、上亿级。

我们先做一个简单方案：

```text
逻辑表：t_order

物理库：
  ds_0 = order_db_0
  ds_1 = order_db_1

物理表：
  order_db_0.t_order_0
  order_db_0.t_order_1
  order_db_1.t_order_0
  order_db_1.t_order_1
```

分片规则：

```text
按 order_id 分库：ds_${order_id % 2}
按 order_id 分表：t_order_${order_id % 2}
```

也就是：

```text
order_id = 1001 → ds_1.t_order_1
order_id = 1002 → ds_0.t_order_0
```

这里选择 **order_id** 做分片键，而不是 `user_id`，原因是订单系统里最常见的核心查询是：

```sql
select * from t_order where order_id = ?
```

如果你的业务更常见的是“查询某个用户的订单列表”，也可以按 `user_id` 分片，但需要接受“按订单号查订单”可能无法精准路由的问题。

---

# 2. 中间件选择：ShardingSphere-JDBC

这里选 **ShardingSphere-JDBC**，不选 MyCat。

原因很简单：

|对比点|ShardingSphere-JDBC|MyCat|
|---|---|---|
|接入方式|Java 应用内嵌 JDBC 层|独立数据库代理|
|Spring Boot 集成|更自然|需要额外部署代理|
|运维复杂度|较低|较高|
|适合场景|Java 后端项目内部分库分表|多语言、统一代理、DB 网关场景|

ShardingSphere 官方文档也明确支持 **Java API / YAML** 两种配置方式，Spring Boot 中可以通过 `ShardingSphereDriver` 加载 YAML 配置文件；官方 YAML 分片配置里也支持 `actualDataNodes`、`databaseStrategy`、`tableStrategy`、`INLINE` 算法等核心配置。([Apache ShardingSphere](https://shardingsphere.apache.org/document/5.5.3/en/quick-start/shardingsphere-jdbc-quick-start/ "ShardingSphere-JDBC :: ShardingSphere"))

---

# 3. 表结构：每个物理库都建相同结构

在 `order_db_0` 和 `order_db_1` 中，都执行下面 SQL。

```sql
CREATE TABLE t_order_0 (
    id BIGINT NOT NULL AUTO_INCREMENT,
    order_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    order_no VARCHAR(64) NOT NULL,
    amount DECIMAL(18, 2) NOT NULL,
    status TINYINT NOT NULL DEFAULT 0,
    create_time DATETIME NOT NULL,
    update_time DATETIME NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_order_id (order_id),
    KEY idx_user_id_create_time (user_id, create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE t_order_1 LIKE t_order_0;
```

注意：

```text
逻辑表叫 t_order
物理表叫 t_order_0 / t_order_1
```

业务代码里永远写：

```sql
select * from t_order where order_id = ?
```

不要在 Java 代码里手写：

```sql
select * from t_order_0 where order_id = ?
```

分片路由由 ShardingSphere-JDBC 接管。

---

# 4. Spring Boot 接入方式

## 4.1 Maven 依赖

以 Spring Boot + MyBatis 为例：

```xml
<dependencies>
    <!-- ShardingSphere JDBC -->
    <dependency>
        <groupId>org.apache.shardingsphere</groupId>
        <artifactId>shardingsphere-jdbc</artifactId>
        <version>5.5.3</version>
    </dependency>

    <!-- MySQL Driver -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
    </dependency>

    <!-- MyBatis Spring Boot -->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>3.0.4</version>
    </dependency>
</dependencies>
```

官方快速开始文档中，ShardingSphere-JDBC 可以通过引入 `shardingsphere-jdbc`，再用 `spring.datasource.driver-class-name=org.apache.shardingsphere.driver.ShardingSphereDriver` 和 `jdbc:shardingsphere:classpath:xxx.yaml` 来加载配置。([Apache ShardingSphere](https://shardingsphere.apache.org/document/5.5.3/en/quick-start/shardingsphere-jdbc-quick-start/ "ShardingSphere-JDBC :: ShardingSphere"))

---

## 4.2 `application.yml`

```yaml
spring:
  datasource:
    driver-class-name: org.apache.shardingsphere.driver.ShardingSphereDriver
    url: jdbc:shardingsphere:classpath:shardingsphere-order.yaml

mybatis:
  mapper-locations: classpath:/mapper/*.xml
  type-aliases-package: com.example.order.domain
```

---

## 4.3 `shardingsphere-order.yaml`

```yaml
databaseName: order_logic_db

dataSources:
  ds_0:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/order_db_0?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: root

  ds_1:
    dataSourceClassName: com.zaxxer.hikari.HikariDataSource
    driverClassName: com.mysql.cj.jdbc.Driver
    jdbcUrl: jdbc:mysql://localhost:3306/order_db_1?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
    username: root
    password: root

rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds_${0..1}.t_order_${0..1}

        databaseStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: order_database_inline

        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: order_table_inline

    shardingAlgorithms:
      order_database_inline:
        type: INLINE
        props:
          algorithm-expression: ds_${order_id % 2}

      order_table_inline:
        type: INLINE
        props:
          algorithm-expression: t_order_${order_id % 2}

props:
  sql-show: true
```

核心解释：

```yaml
actualDataNodes: ds_${0..1}.t_order_${0..1}
```

表示逻辑表 `t_order` 实际对应：

```text
ds_0.t_order_0
ds_0.t_order_1
ds_1.t_order_0
ds_1.t_order_1
```

```yaml
algorithm-expression: ds_${order_id % 2}
algorithm-expression: t_order_${order_id % 2}
```

表示根据 `order_id` 取模决定落在哪个库、哪张表。

官方 YAML 示例中也使用了类似的 `ds_${0..1}.t_order_${0..1}`、`INLINE`、`algorithm-expression` 配置方式。([Apache ShardingSphere](https://shardingsphere.apache.org/document/5.5.1/en/user-manual/shardingsphere-jdbc/yaml-config/rules/sharding/ "Sharding :: ShardingSphere"))

---

# 5. Java 代码

**业务代码不用关心数据落到哪个库、哪张表；但必须围绕分片键设计 SQL，否则中间件只能广播查询。**

## 5.1 Entity

```java
@Data
public class OrderDO {

    private Long id;

    private Long orderId;

    private Long userId;

    private String orderNo;

    private BigDecimal amount;

    private Integer status;

    private LocalDateTime createTime;

    private LocalDateTime updateTime;
}
```

---

## 5.2 Mapper 接口

```java
@Mapper
public interface OrderMapper {

    int insert(OrderDO order);

    OrderDO selectByOrderId(@Param("orderId") Long orderId);

    List<OrderDO> selectByUserId(@Param("userId") Long userId);

    int updateStatusByOrderId(@Param("orderId") Long orderId,
                              @Param("status") Integer status);
}
```

---

## 5.3 Mapper XML

```xml
<mapper namespace="com.example.order.mapper.OrderMapper">

    <insert id="insert">
        INSERT INTO t_order (
            order_id,
            user_id,
            order_no,
            amount,
            status,
            create_time,
            update_time
        ) VALUES (
            #{orderId},
            #{userId},
            #{orderNo},
            #{amount},
            #{status},
            #{createTime},
            #{updateTime}
        )
    </insert>

    <select id="selectByOrderId" resultType="com.example.order.domain.OrderDO">
        SELECT
            id,
            order_id AS orderId,
            user_id AS userId,
            order_no AS orderNo,
            amount,
            status,
            create_time AS createTime,
            update_time AS updateTime
        FROM t_order
        WHERE order_id = #{orderId}
    </select>

    <select id="selectByUserId" resultType="com.example.order.domain.OrderDO">
        SELECT
            id,
            order_id AS orderId,
            user_id AS userId,
            order_no AS orderNo,
            amount,
            status,
            create_time AS createTime,
            update_time AS updateTime
        FROM t_order
        WHERE user_id = #{userId}
        ORDER BY create_time DESC
    </select>

    <update id="updateStatusByOrderId">
        UPDATE t_order
        SET status = #{status},
            update_time = NOW()
        WHERE order_id = #{orderId}
    </update>

</mapper>
```

注意这里全部写的是逻辑表：

```sql
t_order
```

不是物理表：

```sql
t_order_0
t_order_1
```

---

## 5.4 Service 代码

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderMapper orderMapper;

    public Long createOrder(Long userId, BigDecimal amount) {
        long orderId = IdGenerator.nextId();

        OrderDO order = new OrderDO();
        order.setOrderId(orderId);
        order.setUserId(userId);
        order.setOrderNo("ORD" + orderId);
        order.setAmount(amount);
        order.setStatus(0);
        order.setCreateTime(LocalDateTime.now());
        order.setUpdateTime(LocalDateTime.now());

        orderMapper.insert(order);

        return orderId;
    }

    public OrderDO getByOrderId(Long orderId) {
        return orderMapper.selectByOrderId(orderId);
    }

    public void paySuccess(Long orderId) {
        int updated = orderMapper.updateStatusByOrderId(orderId, 1);
        if (updated == 0) {
            throw new IllegalArgumentException("订单不存在：" + orderId);
        }
    }
}
```

`IdGenerator` 可以先用雪花算法，示意如下：

```java
public class IdGenerator {

    private static final AtomicLong SEQUENCE = new AtomicLong(100000000000L);

    public static long nextId() {
        return SEQUENCE.incrementAndGet();
    }
}
```

生产中不要用这个简化版，应该接入：

```text
雪花算法
Leaf
美团 Leaf
Redis incr
数据库号段
ShardingSphere keyGenerator
```

---

# 6. 查询效果

开启：

```yaml
props:
  sql-show: true
```

插入订单：

```java
orderService.createOrder(10001L, new BigDecimal("99.00"));
```

假设生成的：

```text
order_id = 100000000001
```

那么：

```text
100000000001 % 2 = 1
```

实际会路由到：

```text
ds_1.t_order_1
```

执行查询：

```java
orderService.getByOrderId(100000000001L);
```

业务 SQL：

```sql
SELECT * FROM t_order WHERE order_id = ?
```

ShardingSphere-JDBC 实际路由：

```sql
SELECT * FROM ds_1.t_order_1 WHERE order_id = ?
```

这就是分库分表中间件的核心价值：

```text
Java 代码面对逻辑表
中间件负责路由到真实库表
```

---

# 7. 常见坑

## 坑 1：查询条件没有分片键，会全路由

比如：

```sql
SELECT * FROM t_order WHERE user_id = 10001;
```

当前规则按 `order_id` 分片，但 SQL 里没有 `order_id`。

结果是 ShardingSphere 不知道该去哪张表查，只能广播到：

```text
ds_0.t_order_0
ds_0.t_order_1
ds_1.t_order_0
ds_1.t_order_1
```

这叫全路由。

少量数据还能接受，大数据量下就是灾难。

解决方式：

|查询模式|建议|
|---|---|
|高频按订单号查询|用 `order_id` 分片|
|高频查用户订单列表|用 `user_id` 分片|
|两种都高频|建冗余索引表、ES、宽表、订单查询中心|
|后台管理多条件查询|走异构查询库，不要直接扫分片表|

---

## 坑 2：分片键不能随便更新

错误示例：

```sql
UPDATE t_order SET order_id = 20002 WHERE order_id = 10001;
```

`order_id` 决定数据在哪个库、哪张表。

如果改了分片键，本质上可能需要：

```text
从旧分片删除
插入新分片
保证事务一致性
```

生产中一般规定：

```text
分片键一旦写入，不允许修改
```

---

## 坑 3：范围查询容易跨库跨表

比如：

```sql
SELECT * FROM t_order
WHERE order_id BETWEEN 100000000000 AND 100000999999;
```

即使带了 `order_id`，范围查询也可能覆盖多个分片。

这类 SQL 在分库分表后要谨慎。

更推荐：

```sql
SELECT * FROM t_order
WHERE order_id = ?
```

或者业务上分页查询时按明确的分片键查询。

---

## 坑 4：分页查询成本变高

比如：

```sql
SELECT * FROM t_order
ORDER BY create_time DESC
LIMIT 10000, 20;
```

分库分表后，这不是简单查一张表。

它可能变成：

```text
每个分片都查一批数据
中间件归并排序
再截取 LIMIT
```

深分页会非常重。

解决方式：

```sql
SELECT * FROM t_order
WHERE user_id = ?
  AND create_time < ?
ORDER BY create_time DESC
LIMIT 20;
```

也就是使用游标分页，而不是深分页。

---

## 坑 5：分布式事务不要默认当成本地事务

单库单表时：

```java
@Transactional
public void createOrderAndPay() {
    createOrder();
    createPayRecord();
}
```

事务边界相对简单。

分库分表后，如果订单表和支付表落在不同库，本地事务就不够了。

常见处理方式：

|场景|方案|
|---|---|
|强一致要求高|尽量让相关数据按同一分片键落同库|
|支付、订单状态流转|本地事务 + 可靠消息 + 幂等消费|
|跨库强事务|XA / Seata，但复杂度高|
|大多数互联网业务|最终一致性|

不要一上来就幻想“分库分表后事务还像单库一样简单”。

---

## 坑 6：唯一索引只能保证单个物理表内唯一

比如每张物理表都有：

```sql
UNIQUE KEY uk_order_no (order_no)
```

它只能保证：

```text
当前这张物理表内唯一
```

不能天然保证：

```text
所有库表全局唯一
```

所以 `order_no`、`order_id` 这种关键业务字段，要么由全局 ID 生成器保证唯一，要么设计成带业务前缀、时间、机器号、序列号的全局唯一编码。

---

## 坑 7：分片键选择比中间件配置更重要

中间件只是帮你路由 SQL。

真正决定系统好不好用的是：

```text
你用什么字段分片
你的核心查询是否都带分片键
你的事务边界是否围绕分片键设计
你的扩容方案是否提前考虑
```

订单系统里最常见的选择：

|分片键|优点|缺点|
|---|---|---|
|`order_id`|按订单号查询非常快；订单详情、支付回调友好|查用户订单列表可能跨分片|
|`user_id`|查用户订单列表很自然|按订单号查可能需要索引表|
|`buyer_id`|电商买家侧查询友好|商家侧、后台查询麻烦|
|`merchant_id`|商家维度查询友好|买家侧查询麻烦|

没有完美分片键，只有最贴合主链路的分片键。

---

# 8. 最小可运行链路总结

你真正接入时，核心就 5 步：

```text
1. 建物理库：
   order_db_0
   order_db_1

2. 建物理表：
   t_order_0
   t_order_1

3. Spring Boot 配置 ShardingSphereDriver

4. shardingsphere-order.yaml 配置：
   dataSources
   actualDataNodes
   databaseStrategy
   tableStrategy
   shardingAlgorithms

5. Java/MyBatis 代码只操作逻辑表：
   t_order
```

一句话总结：

> ShardingSphere-JDBC 的工程使用方式不是让你在代码里判断该查哪张表，而是让业务代码继续写 `t_order`，由中间件根据 `order_id` 或 `user_id` 自动路由到真实库表。真正的难点不是配置，而是分片键选择、查询模式约束、分页、事务、全局唯一 ID 和扩容方案。