[Guava官方wiki](https://github.com/google/guava/wiki)
[xfg](https://bugstack.cn/md/road-map/guava.html)
## 1. 先给结论



### 1.1 新项目还推荐使用 Guava 吗？

**结论：可以用，但不要把 Guava 当成“万能工具包”。**

在 Java 17 / Java 21 项目里，Guava 的定位应该是：

> **集合增强 + 少量实用工具 + 老项目兼容工具库。**

它不再是 Java 项目里“必选”的基础依赖。因为 JDK 8 之后，`Optional`、`Stream`、`CompletableFuture`、`Map.of`、`List.of`、`Objects`、`StringJoiner`、`Files` 等能力已经覆盖了 Guava 早期解决的一部分问题。OpenRewrite 甚至已经提供了面向 Java 21 的 Guava 替换迁移规则，说明部分 Guava API 已经可以被标准库替代。([OpenRewrite Docs](https://docs.openrewrite.org/recipes/java/migrate/guava/noguavajava21?utm_source=chatgpt.com "Prefer the Java 21 standard library instead of Guava"))

但 Guava 仍然有价值，尤其是：

| 场景                                  | Guava 是否仍有价值              |
| ----------------------------------- | ------------------------- |
| 不可变集合                               | 有价值，表达力强                  |
| Multimap / Multiset / BiMap / Table | 很有价值，JDK 没有直接替代           |
| Preconditions                       | 有价值，但可被 Spring / JDK 部分替代 |
| Joiner / Splitter                   | 有价值，尤其是复杂字符串拆分            |
| Guava Cache                         | 老项目可维护，新项目更推荐 Caffeine    |
| ListenableFuture                    | 不建议新项目优先使用                |
| Optional                            | 不建议优先使用 Guava Optional    |
| Files / Resources / ByteStreams     | 可用，但很多场景 JDK 已够用          |

Guava 当前仍然是一个活跃项目，官方仓库说明其 Maven 坐标为 `com.google.guava:guava`，并且区分 JRE 与 Android 两种 flavor；JRE flavor 面向 Java 8+。([GitHub](https://github.com/google/guava?utm_source=chatgpt.com "Guava: Google Core Libraries for Java"))

---

### 1.2 JDK 17 / 21 之后，Guava 还有哪些值得用？

**最值得用的是 Guava 的增强集合。**

这些东西 JDK 没有直接给你：

```java
Multimap<String, Long> categoryProductMap;
Multiset<String> wordCounter;
BiMap<Long, String> userIdUsernameMap;
Table<String, String, BigDecimal> priceTable;
ImmutableMap<String, Integer> fixedConfigMap;
```

这些类型在后端项目里很实用，因为它们不是“工具方法”，而是**更准确的数据结构表达**。

比如：

```java
Map<Long, List<Long>> userRoleMap = new HashMap<>();
```

这当然能写，但你要自己处理：

```java
userRoleMap.computeIfAbsent(userId, k -> new ArrayList<>()).add(roleId);
```

换成 Guava：

```java
Multimap<Long, Long> userRoleMap = ArrayListMultimap.create();
userRoleMap.put(userId, roleId);
```

这不是炫技，而是代码语义更直接：**一个用户对应多个角色**。

---

### 1.3 哪些 Guava 功能不建议新项目优先使用？

|Guava 功能|不优先推荐原因|建议替代|
|---|---|---|
|`com.google.common.base.Optional`|JDK 8 已有 `java.util.Optional`|`java.util.Optional`|
|`ListenableFuture`|Java 8+ 已有 `CompletableFuture`|`CompletableFuture`|
|`FluentIterable`|Stream API 已成熟|`Stream`|
|`Throwables.propagate`|异常处理风格过时|明确抛出业务异常|
|`Guava Cache`|官方文档也建议优先考虑 Caffeine|Caffeine|
|`Files` 部分方法|JDK `java.nio.file.Files` 已经很强|JDK Files|
|`Objects.equal/hashCode`|JDK `Objects` 已覆盖|`java.util.Objects`|

Guava 的 `CacheBuilder` API 文档明确写到，Guava caching API 的继任者是 Caffeine，并说明 Caffeine 提供更好的性能、更多特性，包括异步加载和更少 bug。([Guava](https://guava.dev/releases/31.0-jre/api/docs/com/google/common/cache/CacheBuilder.html?utm_source=chatgpt.com "Class CacheBuilder<K,​V>"))

---

# 2. 常用集合实践

## 2.1 ImmutableList / ImmutableMap / ImmutableSet：配置类、枚举映射、只读结果

### 解决什么问题？

后端项目里经常有一些数据**初始化后不应该被修改**：

- 状态机允许流转关系
    
- 白名单字段
    
- 固定业务类型映射
    
- 默认配置项
    
- 接口返回的只读集合
    

如果你用普通 `ArrayList` / `HashMap`，别人拿到引用后可能误改。

---

### 示例：订单状态允许流转关系

```java
import com.google.common.collect.ImmutableMap;
import com.google.common.collect.ImmutableSet;

import java.util.Set;

public class OrderStatusRule {

    /**
     * 固定的订单状态流转规则。
     * 使用 ImmutableMap / ImmutableSet 的目的：
     * 1. 初始化后不可修改
     * 2. 避免业务代码误操作规则表
     * 3. 表达“这是配置规则，不是运行时数据”
     */
    private static final ImmutableMap<String, ImmutableSet<String>> ALLOWED_TRANSITIONS =
            ImmutableMap.<String, ImmutableSet<String>>builder()
                    .put("CREATED", ImmutableSet.of("PAID", "CANCELLED"))
                    .put("PAID", ImmutableSet.of("DELIVERED", "REFUNDED"))
                    .put("DELIVERED", ImmutableSet.of("FINISHED"))
                    .put("CANCELLED", ImmutableSet.of())
                    .put("FINISHED", ImmutableSet.of())
                    .build();

    public boolean canTransit(String currentStatus, String nextStatus) {
        Set<String> allowedNextStatus = ALLOWED_TRANSITIONS.getOrDefault(
                currentStatus,
                ImmutableSet.of()
        );

        return allowedNextStatus.contains(nextStatus);
    }
}
```

### 什么时候用？

适合：

- 固定配置
    
- 只读规则
    
- 常量集合
    
- 不希望调用方修改的返回值
    

不适合：

- 高频增删改集合
    
- 大量动态数据
    
- 需要并发写入的数据结构
    

---

## 2.2 Multimap：一个 key 对多个 value

### 解决什么问题？

典型业务场景：

- 一个用户多个角色
    
- 一个分类多个商品
    
- 一个订单多个优惠券
    
- 一个标签多个文章
    
- 一个部门多个员工
    

不用 Guava 的写法：

```java
Map<Long, List<Long>> categoryProductMap = new HashMap<>();

categoryProductMap
        .computeIfAbsent(categoryId, key -> new ArrayList<>())
        .add(productId);
```

不是不能写，而是样板代码多。

---

### 示例：商品按照分类分组

```java
import com.google.common.collect.ArrayListMultimap;
import com.google.common.collect.Multimap;

import java.util.Collection;
import java.util.List;

public class ProductGroupService {

    public Multimap<Long, Long> groupProductIdsByCategory(List<ProductDTO> products) {
        Multimap<Long, Long> categoryProductMap = ArrayListMultimap.create();

        for (ProductDTO product : products) {
            // 一个 categoryId 可以对应多个 productId
            categoryProductMap.put(product.categoryId(), product.productId());
        }

        return categoryProductMap;
    }

    public Collection<Long> getProductIdsByCategory(Multimap<Long, Long> categoryProductMap,
                                                    Long categoryId) {
        // 即使 key 不存在，也会返回空集合，而不是 null
        return categoryProductMap.get(categoryId);
    }

    public record ProductDTO(Long productId, Long categoryId, String productName) {
    }
}
```

### 什么时候用？

适合：

- `Map<K, List<V>>`
    
- `Map<K, Set<V>>`
    
- 按 key 聚合数据
    

不适合：

- 需要复杂查询条件
    
- value 需要分页
    
- 数据量非常大，需要放数据库或 Redis
    

---

## 2.3 Multiset：统计元素出现次数

### 解决什么问题？

`Multiset` 可以理解为“带计数的 Set”。

典型场景：

- 统计标签出现次数
    
- 统计接口错误码出现次数
    
- 统计搜索关键词频次
    
- 统计用户行为事件次数
    

---

### 示例：统计接口错误码出现次数

```java
import com.google.common.collect.HashMultiset;
import com.google.common.collect.Multiset;

import java.util.List;

public class ErrorCodeStatisticsService {

    public Multiset<String> countErrorCodes(List<ApiLog> logs) {
        Multiset<String> errorCodeCounter = HashMultiset.create();

        for (ApiLog log : logs) {
            if (log.success()) {
                continue;
            }

            // add 一次，计数 +1
            errorCodeCounter.add(log.errorCode());
        }

        return errorCodeCounter;
    }

    public int getErrorCount(Multiset<String> counter, String errorCode) {
        // 不存在返回 0，不需要手动处理 null
        return counter.count(errorCode);
    }

    public record ApiLog(boolean success, String errorCode) {
    }
}
```

### 和普通 Map 的对比

不用 Guava：

```java
Map<String, Integer> counter = new HashMap<>();

counter.merge(errorCode, 1, Integer::sum);
```

用 Guava：

```java
Multiset<String> counter = HashMultiset.create();
counter.add(errorCode);
```

### 什么时候用？

适合：

- 简单频次统计
    
- 内存内聚合
    
- 日志、标签、关键词统计
    

不适合：

- 大规模实时统计
    
- 分布式计数
    
- 精确持久化统计
    

这些应该交给 Redis、ClickHouse、Flink、数据库聚合等系统。

---

## 2.4 BiMap：双向映射

### 解决什么问题？

普通 `Map<K, V>` 只能从 key 查 value。

但有些场景需要双向查询：

- 用户 ID ↔ 用户名
    
- 编码 ↔ 枚举名
    
- 权限码 ↔ 权限 ID
    
- 国家编码 ↔ 国家名称
    

---

### 示例：业务状态码和状态名称互查

```java
import com.google.common.collect.BiMap;
import com.google.common.collect.HashBiMap;

public class StatusCodeService {

    private final BiMap<Integer, String> statusMap = HashBiMap.create();

    public StatusCodeService() {
        statusMap.put(1, "CREATED");
        statusMap.put(2, "PAID");
        statusMap.put(3, "DELIVERED");
        statusMap.put(4, "FINISHED");
    }

    public String getStatusName(Integer code) {
        return statusMap.get(code);
    }

    public Integer getStatusCode(String statusName) {
        // inverse() 返回反向视图：statusName -> code
        return statusMap.inverse().get(statusName);
    }
}
```

### 注意点

`BiMap` 的 value 也必须唯一。

```java
statusMap.put(1, "CREATED");
statusMap.put(2, "CREATED"); // 会抛异常，因为 value 重复
```

需要覆盖时可以用：

```java
statusMap.forcePut(2, "CREATED");
```

但生产代码里要谨慎，避免隐藏数据冲突。

---

## 2.5 Table：二维映射

### 解决什么问题？

`Table<R, C, V>` 可以理解为：

```java
Map<RowKey, Map<ColumnKey, Value>>
```

典型场景：

- 城市 + 业务类型 → 价格
    
- 用户 + 权限类型 → 是否拥有
    
- 商品 + 渠道 → 售价
    
- 日期 + 指标名 → 指标值
    

---

### 示例：不同渠道下的商品价格

```java
import com.google.common.collect.HashBasedTable;
import com.google.common.collect.Table;

import java.math.BigDecimal;

public class ChannelPriceService {

    /**
     * rowKey: productId
     * columnKey: channel
     * value: price
     */
    private final Table<Long, String, BigDecimal> priceTable = HashBasedTable.create();

    public void initPrice() {
        priceTable.put(1001L, "APP", new BigDecimal("99.00"));
        priceTable.put(1001L, "MINI_PROGRAM", new BigDecimal("95.00"));
        priceTable.put(1002L, "APP", new BigDecimal("199.00"));
    }

    public BigDecimal getPrice(Long productId, String channel) {
        return priceTable.get(productId, channel);
    }
}
```

### 什么时候用？

适合：

- 二维关系清晰
    
- 数据量不大
    
- 内存中临时计算
    

不适合：

- 三维以上复杂关系
    
- 数据量大
    
- 需要数据库查询优化
    

---

# 3. 字符串与参数处理

## 3.1 Strings：空字符串、null、默认值处理

### 常用方法

```java
Strings.isNullOrEmpty(value);
Strings.nullToEmpty(value);
Strings.emptyToNull(value);
```

### 示例：接口参数清洗

```java
import com.google.common.base.Strings;

public class UserQueryService {

    public UserQueryCommand buildCommand(String keyword, String status) {
        return new UserQueryCommand(
                // null 转为空字符串，避免后续 trim 空指针
                Strings.nullToEmpty(keyword).trim(),

                // 空字符串转 null，方便 MyBatis 动态 SQL 判断
                Strings.emptyToNull(Strings.nullToEmpty(status).trim())
        );
    }

    public record UserQueryCommand(String keyword, String status) {
    }
}
```

### 什么时候用？

适合：

- Controller 入参清洗
    
- DTO 转 QueryCommand
    
- 兼容前端传 `""` 和 `null`
    

不适合：

- 复杂校验
    
- Bean Validation 已经能覆盖的场景
    

复杂参数校验应该优先用：

```java
@NotBlank
@NotNull
@Size
@Pattern
```

---

## 3.2 Joiner：集合拼接字符串

### 解决什么问题？

常见场景：

- 日志输出 ID 列表
    
- 拼接导出字段
    
- 拼接错误信息
    
- 拼接 SQL 调试片段
    

---

### 示例：拼接批量操作日志

```java
import com.google.common.base.Joiner;

import java.util.List;

public class OperationLogService {

    public String buildDeleteLog(List<Long> productIds) {
        return "批量删除商品，productIds="
                + Joiner.on(",")
                        .skipNulls()
                        .join(productIds);
    }
}
```

### 和 String.join 的区别

`String.join` 只适合 `CharSequence`：

```java
String.join(",", List.of("A", "B", "C"));
```

`Joiner` 对 null 处理更灵活：

```java
Joiner.on(",").skipNulls().join(values);
Joiner.on(",").useForNull("UNKNOWN").join(values);
```

---

## 3.3 Splitter：字符串拆分，比 String.split 更适合后端参数解析

### String.split 的问题

```java
String[] arr = "1, 2,,3 ".split(",");
```

你还要自己处理：

- trim
    
- 空字符串
    
- 类型转换
    
- 异常值
    

---

### 示例：解析前端传来的 ID 字符串

```java
import com.google.common.base.Splitter;

import java.util.List;

public class IdParser {

    private static final Splitter COMMA_SPLITTER = Splitter.on(",")
            .trimResults()
            .omitEmptyStrings();

    public List<Long> parseIdList(String rawIds) {
        return COMMA_SPLITTER.splitToStream(rawIds)
                .map(Long::valueOf)
                .toList();
    }
}
```

输入：

```text
"1001, 1002,, 1003 "
```

输出：

```java
[1001L, 1002L, 1003L]
```

### 什么时候用？

适合：

- URL 参数解析
    
- 配置项解析
    
- 后台管理系统批量 ID 输入
    
- 简单 CSV 风格字符串处理
    

不适合：

- 真正的 CSV 文件解析
    
- 含引号、转义、换行的复杂格式
    

复杂 CSV 应该用 OpenCSV、Apache Commons CSV。

---

## 3.4 Preconditions：参数校验和状态校验

### 常用方法

```java
Preconditions.checkArgument(condition, "参数错误");
Preconditions.checkNotNull(object, "对象不能为空");
Preconditions.checkState(condition, "状态错误");
```

### 示例：Service 层业务前置校验

```java
import com.google.common.base.Preconditions;

import java.math.BigDecimal;

public class ProductPriceService {

    public void updatePrice(Long productId, BigDecimal price) {
        Preconditions.checkNotNull(productId, "productId 不能为空");
        Preconditions.checkNotNull(price, "price 不能为空");

        Preconditions.checkArgument(
                price.compareTo(BigDecimal.ZERO) > 0,
                "商品价格必须大于 0"
        );

        // 这里才是真正的业务逻辑
        doUpdatePrice(productId, price);
    }

    private void doUpdatePrice(Long productId, BigDecimal price) {
        // 调用 Repository 更新数据库
    }
}
```

### Controller 层是否应该大量用 Preconditions？

不建议。

Controller 入参校验优先：

```java
@Validated
@PostMapping("/products")
public void create(@Valid @RequestBody CreateProductRequest request) {
    productService.create(request);
}
```

Service 层可以用 `Preconditions` 做内部防御性校验。

---

# 4. 缓存实践

## 4.1 Guava Cache 适合什么？

Guava Cache 是**本地内存缓存**。

它适合：

- 缓存少量热点配置
    
- 缓存字典数据
    
- 缓存权限元数据
    
- 缓存短时间不变的远程接口结果
    
- 单机应用或一致性要求不高的场景
    

它不适合：

- 分布式共享缓存
    
- 强一致性数据
    
- 库存、余额、优惠券这类关键数据
    
- 多节点之间必须实时同步的数据
    

Guava 官方对 Cache 的解释是：它类似 `ConcurrentMap`，但核心区别是 Cache 通常会配置自动驱逐，以限制内存占用；而 `ConcurrentMap` 会一直保留元素，直到显式移除。([GitHub](https://github.com/google/guava/wiki/cachesexplained?utm_source=chatgpt.com "CachesExplained · google/guava Wiki"))

---

## 4.2 expireAfterWrite / expireAfterAccess / maximumSize 怎么选？

|配置|含义|适合场景|
|---|---|---|
|`expireAfterWrite`|写入后固定时间过期|配置、字典、商品分类|
|`expireAfterAccess`|访问后续期|会话、本地热点对象|
|`maximumSize`|限制最大条目数|防止内存无限膨胀|
|`refreshAfterWrite`|写入一段时间后刷新|Guava 支持有限，Caffeine 更强|

实践建议：

```java
CacheBuilder.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(10, TimeUnit.MINUTES);
```

不要只设置过期时间，不设置最大容量。

---

## 4.3 LoadingCache 示例：商品分类本地缓存

```java
import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

public class CategoryLocalCache {

    private final CategoryRepository categoryRepository;

    private final LoadingCache<Long, CategoryDTO> categoryCache;

    public CategoryLocalCache(CategoryRepository categoryRepository) {
        this.categoryRepository = categoryRepository;

        this.categoryCache = CacheBuilder.newBuilder()
                // 最多缓存 5000 个分类，避免内存无限增长
                .maximumSize(5_000)

                // 写入 10 分钟后过期，适合变化不频繁的分类数据
                .expireAfterWrite(10, TimeUnit.MINUTES)

                // 构建 LoadingCache，缓存 miss 时自动加载
                .build(new CacheLoader<>() {
                    @Override
                    public CategoryDTO load(Long categoryId) {
                        return categoryRepository.findById(categoryId);
                    }
                });
    }

    public CategoryDTO getCategory(Long categoryId) {
        try {
            return categoryCache.get(categoryId);
        } catch (ExecutionException e) {
            // 不建议把底层异常直接抛给上层，转换为业务异常更清晰
            throw new IllegalStateException("加载商品分类缓存失败，categoryId=" + categoryId, e);
        }
    }

    public void invalidate(Long categoryId) {
        // 分类更新后，主动删除本地缓存
        categoryCache.invalidate(categoryId);
    }

    public record CategoryDTO(Long id, String name) {
    }

    public interface CategoryRepository {
        CategoryDTO findById(Long categoryId);
    }
}
```

---

## 4.4 为什么新项目更推荐 Caffeine？

Caffeine 是 Guava Cache 的事实继任者。Caffeine 官方说明它提供了受 Guava API 启发的内存缓存，并在设计上基于 Guava Cache 和 ConcurrentLinkedHashMap 的经验改进。([GitHub](https://github.com/ben-manes/caffeine?utm_source=chatgpt.com "ben-manes/caffeine: A high performance caching library for ..."))

Guava `CacheBuilder` API 文档也直接建议优先使用 Caffeine，理由包括更好的性能、更多功能、异步加载和更少 bug。([Guava](https://guava.dev/releases/31.0-jre/api/docs/com/google/common/cache/CacheBuilder.html?utm_source=chatgpt.com "Class CacheBuilder<K,​V>"))

### Guava Cache 和 Caffeine 的取舍

|场景|建议|
|---|---|
|老项目已经用了 Guava Cache|可以继续维护|
|新项目要做本地缓存|优先 Caffeine|
|Spring Boot 项目|优先 Spring Cache + Caffeine|
|只做简单学习 demo|Guava Cache 可以|
|生产高并发热点缓存|Caffeine 更合适|

---

# 5. 并发工具

## 5.1 ListenableFuture 基本用法

Guava 在 Java 8 之前提供了 `ListenableFuture`，用来给异步任务添加回调。

```java
import com.google.common.util.concurrent.*;

import java.util.concurrent.Executors;

public class AsyncTaskService {

    private final ListeningExecutorService executorService =
            MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(4));

    public void submitTask() {
        ListenableFuture<String> future = executorService.submit(() -> {
            // 模拟远程调用
            return callRemoteService();
        });

        Futures.addCallback(
                future,
                new FutureCallback<>() {
                    @Override
                    public void onSuccess(String result) {
                        // 异步任务成功后的处理
                        System.out.println("远程调用成功：" + result);
                    }

                    @Override
                    public void onFailure(Throwable throwable) {
                        // 异步任务失败后的处理
                        System.err.println("远程调用失败：" + throwable.getMessage());
                    }
                },
                executorService
        );
    }

    private String callRemoteService() {
        return "OK";
    }
}
```

---

## 5.2 和 CompletableFuture 的关系

Java 8 之后，`CompletableFuture` 已经覆盖了大部分异步编排场景：

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureService {

    public CompletableFuture<String> queryUserProfile(Long userId) {
        return CompletableFuture
                .supplyAsync(() -> callUserService(userId))
                .thenApply(profile -> "用户资料：" + profile)
                .exceptionally(ex -> {
                    // 异常兜底，避免异常直接吞掉
                    return "默认用户资料";
                });
    }

    private String callUserService(Long userId) {
        return "user-" + userId;
    }
}
```

### 现在是否推荐 ListenableFuture？

**新项目不优先推荐。**

建议：

|场景|选择|
|---|---|
|新业务代码|`CompletableFuture`|
|Spring 异步任务|`@Async` + `CompletableFuture`|
|Reactor 项目|`Mono` / `Flux`|
|老项目已有 Guava 并发工具|继续维护即可|
|需要兼容老框架返回值|可以保留 `ListenableFuture`|

---

# 6. I/O、Hash、RateLimiter 等实用工具

## 6.1 Files / Resources / ByteStreams

### Resources：读取 classpath 文件

适合读取：

- 初始化 SQL
    
- JSON 模板
    
- 本地测试数据
    
- classpath 下的配置模板
    

```java
import com.google.common.io.Resources;

import java.io.IOException;
import java.net.URL;
import java.nio.charset.StandardCharsets;

public class TemplateLoader {

    public String loadTemplate(String path) {
        try {
            // 从 classpath 加载资源文件
            URL url = Resources.getResource(path);

            // 读取为字符串，适合读取小文件模板
            return Resources.toString(url, StandardCharsets.UTF_8);
        } catch (IOException e) {
            throw new IllegalStateException("读取模板文件失败，path=" + path, e);
        }
    }
}
```

### ByteStreams：流拷贝

```java
import com.google.common.io.ByteStreams;

import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;

public class FileCopyService {

    public void copy(InputStream inputStream, OutputStream outputStream) {
        try {
            // 适合简单流复制，避免手写 buffer 循环
            ByteStreams.copy(inputStream, outputStream);
        } catch (IOException e) {
            throw new IllegalStateException("文件流复制失败", e);
        }
    }
}
```

### 生产边界

这些工具可以用，但 Java 17 / 21 下 JDK I/O 已经足够强：

```java
java.nio.file.Files.readString(path);
java.nio.file.Files.copy(source, target);
java.nio.file.Files.newInputStream(path);
```

所以新项目不要为了读文件强行引入 Guava。

---

## 6.2 Hashing：哈希工具

### 典型场景

- 生成缓存 key
    
- 数据分片
    
- 简单一致性哈希
    
- 文件摘要
    
- 幂等 key 辅助生成
    

### 示例：生成业务幂等 key

```java
import com.google.common.hash.Hashing;

import java.nio.charset.StandardCharsets;

public class IdempotentKeyService {

    public String buildOrderCreateKey(Long userId, Long productId, String requestNo) {
        String rawKey = userId + ":" + productId + ":" + requestNo;

        // murmur3_128 适合非加密哈希，速度快，碰撞率低
        return Hashing.murmur3_128()
                .hashString(rawKey, StandardCharsets.UTF_8)
                .toString();
    }
}
```

### 注意

`Hashing.murmur3_128()` 不是加密算法。

不适合：

- 密码存储
    
- 签名验签
    
- 安全摘要
    
- 防篡改
    

安全场景应该用：

- SHA-256
    
- HMAC-SHA256
    
- BCrypt
    
- Argon2
    
- JDK `MessageDigest`
    
- Spring Security Crypto
    

---

## 6.3 RateLimiter：简单单机限流

### 解决什么问题？

Guava `RateLimiter` 是单机令牌桶限流。

适合：

- 单机后台任务限速
    
- 防止本服务压垮下游服务
    
- 批处理任务平滑调用第三方接口
    
- 简单管理后台按钮防抖
    

---

### 示例：调用第三方接口前限速

```java
import com.google.common.util.concurrent.RateLimiter;

public class SmsSendService {

    /**
     * 每秒最多发 20 个请求。
     * 注意：这是单机限流，多实例部署时总 QPS = 实例数 * 20。
     */
    private final RateLimiter rateLimiter = RateLimiter.create(20.0);

    private final SmsClient smsClient;

    public SmsSendService(SmsClient smsClient) {
        this.smsClient = smsClient;
    }

    public void sendSms(String phone, String content) {
        // acquire 会阻塞，直到拿到令牌
        rateLimiter.acquire();

        smsClient.send(phone, content);
    }

    public interface SmsClient {
        void send(String phone, String content);
    }
}
```

### 生产边界

`RateLimiter` 不适合做：

- 分布式限流
    
- API 网关限流
    
- 用户级精细化限流
    
- 秒杀库存保护
    
- 多租户全局限流
    

这些场景建议：

|场景|推荐方案|
|---|---|
|网关限流|Spring Cloud Gateway / Sentinel|
|分布式限流|Redis + Lua / Sentinel|
|服务治理|Sentinel / Resilience4j|
|Nginx 层限流|Nginx limit_req|
|高并发秒杀|Redis 原子扣减 + 队列削峰|

---

# 7. Spring Boot 项目实战：商品分类本地缓存 + 标签分组 + 参数校验

## 7.1 Maven 引入

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>33.4.8-jre</version>
</dependency>
```

官方仓库当前示例中 Maven 坐标为 `com.google.guava:guava`，并区分 `-jre` 和 `-android` flavor；普通 Java 后端项目应该选择 `-jre`。([GitHub](https://github.com/google/guava?utm_source=chatgpt.com "Guava: Google Core Libraries for Java"))

---

## 7.2 案例目标

做一个商品查询接口：

```text
GET /products/search?categoryId=1&tagIds=10,20,30
```

要求：

1. 校验 `categoryId` 不能为空。
    
2. `tagIds` 支持逗号分隔。
    
3. 商品分类使用本地缓存。
    
4. 查询出的商品按标签分组。
    
5. 返回结构适合前端展示。
    

---

## 7.3 DTO

```java
import java.math.BigDecimal;
import java.util.List;

public record ProductSearchRequest(
        Long categoryId,
        String tagIds
) {
}

public record ProductView(
        Long productId,
        String productName,
        Long categoryId,
        String categoryName,
        List<Long> tagIds,
        BigDecimal price
) {
}

public record ProductSearchResponse(
        Long categoryId,
        String categoryName,
        List<TagProductGroup> groups
) {
}

public record TagProductGroup(
        Long tagId,
        List<ProductView> products
) {
}

public record CategoryDTO(
        Long categoryId,
        String categoryName
) {
}
```

---

## 7.4 Controller

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class ProductController {

    private final ProductQueryService productQueryService;

    public ProductController(ProductQueryService productQueryService) {
        this.productQueryService = productQueryService;
    }

    @GetMapping("/products/search")
    public ProductSearchResponse search(ProductSearchRequest request) {
        // Controller 保持薄，参数解析和业务规则交给 Service
        return productQueryService.search(request);
    }
}
```

---

## 7.5 本地分类缓存组件

```java
import com.google.common.cache.CacheBuilder;
import com.google.common.cache.CacheLoader;
import com.google.common.cache.LoadingCache;
import org.springframework.stereotype.Component;

import java.util.concurrent.ExecutionException;
import java.util.concurrent.TimeUnit;

@Component
public class CategoryCache {

    private final LoadingCache<Long, CategoryDTO> categoryCache;

    public CategoryCache(CategoryRepository categoryRepository) {
        this.categoryCache = CacheBuilder.newBuilder()
                // 分类数量通常有限，设置最大容量避免异常数据撑爆内存
                .maximumSize(10_000)

                // 分类名称偶尔变化，允许最多 10 分钟缓存延迟
                .expireAfterWrite(10, TimeUnit.MINUTES)

                .build(new CacheLoader<>() {
                    @Override
                    public CategoryDTO load(Long categoryId) {
                        return categoryRepository.findById(categoryId);
                    }
                });
    }

    public CategoryDTO get(Long categoryId) {
        try {
            return categoryCache.get(categoryId);
        } catch (ExecutionException e) {
            throw new IllegalStateException("获取商品分类缓存失败，categoryId=" + categoryId, e);
        }
    }

    public void evict(Long categoryId) {
        // 分类被后台修改后，主动清理缓存
        categoryCache.invalidate(categoryId);
    }
}
```

---

## 7.6 Repository 接口

```java
import java.util.List;

public interface CategoryRepository {

    CategoryDTO findById(Long categoryId);
}

public interface ProductRepository {

    List<ProductView> searchByCategoryAndTags(Long categoryId, List<Long> tagIds);
}
```

实际项目中这里可以是：

- MyBatis Mapper
    
- MyBatis-Plus Service
    
- JPA Repository
    
- RPC Client
    
- 防腐层 Gateway
    

---

## 7.7 Service：Guava 综合实践

```java
import com.google.common.base.Preconditions;
import com.google.common.base.Splitter;
import com.google.common.collect.ArrayListMultimap;
import com.google.common.collect.Multimap;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class ProductQueryService {

    private static final Splitter TAG_ID_SPLITTER = Splitter.on(",")
            .trimResults()
            .omitEmptyStrings();

    private final CategoryCache categoryCache;
    private final ProductRepository productRepository;

    public ProductQueryService(CategoryCache categoryCache,
                               ProductRepository productRepository) {
        this.categoryCache = categoryCache;
        this.productRepository = productRepository;
    }

    public ProductSearchResponse search(ProductSearchRequest request) {
        // Service 层做业务前置校验，避免无意义查询打到数据库
        Preconditions.checkNotNull(request.categoryId(), "categoryId 不能为空");

        List<Long> tagIds = parseTagIds(request.tagIds());

        CategoryDTO category = categoryCache.get(request.categoryId());

        List<ProductView> products = productRepository.searchByCategoryAndTags(
                request.categoryId(),
                tagIds
        );

        Multimap<Long, ProductView> tagProductMap = groupByTag(products);

        List<TagProductGroup> groups = tagProductMap.keySet()
                .stream()
                .map(tagId -> new TagProductGroup(
                        tagId,
                        List.copyOf(tagProductMap.get(tagId))
                ))
                .toList();

        return new ProductSearchResponse(
                category.categoryId(),
                category.categoryName(),
                groups
        );
    }

    private List<Long> parseTagIds(String rawTagIds) {
        if (rawTagIds == null || rawTagIds.isBlank()) {
            return List.of();
        }

        return TAG_ID_SPLITTER.splitToStream(rawTagIds)
                .map(Long::valueOf)
                .toList();
    }

    private Multimap<Long, ProductView> groupByTag(List<ProductView> products) {
        Multimap<Long, ProductView> tagProductMap = ArrayListMultimap.create();

        for (ProductView product : products) {
            for (Long tagId : product.tagIds()) {
                // 一个 tagId 下可以挂多个商品
                tagProductMap.put(tagId, product);
            }
        }

        return tagProductMap;
    }
}
```

---

## 7.8 这个案例里 Guava 的价值

|代码点|使用 Guava|价值|
|---|---|---|
|参数校验|`Preconditions`|Service 层防御性校验|
|字符串拆分|`Splitter`|处理 trim、空值更舒服|
|分类缓存|`LoadingCache`|简单本地缓存|
|标签分组|`Multimap`|替代 `Map<Long, List<ProductView>>`|
|只读返回|`List.copyOf`|这里用 JDK 即可，不必强行 Guava|

注意最后一点：**不是用了 Guava，就所有地方都要用 Guava。**

Java 17 / 21 里，很多只读集合可以直接用：

```java
List.of(...)
Map.of(...)
Set.of(...)
List.copyOf(...)
```

---

# 8. 常见坑和最佳实践

## 8.1 不要滥用 Guava Optional

不要在新代码里写：

```java
com.google.common.base.Optional<User> user;
```

优先使用：

```java
java.util.Optional<User> user;
```

尤其不要在 DTO、Entity 字段中使用 Optional：

```java
public class UserDTO {
    private Optional<String> nickname; // 不推荐
}
```

更好的方式：

```java
public class UserDTO {
    private String nickname; // 可以为 null，由序列化和校验层处理
}
```

`Optional` 更适合作为方法返回值，而不是字段类型。

---

## 8.2 不可变集合不是普通集合

下面代码会抛异常：

```java
ImmutableList<String> names = ImmutableList.of("Tom", "Jerry");
names.add("Spike"); // UnsupportedOperationException
```

不可变集合适合表达：

> 这个集合初始化后不允许修改。

不要把它用在需要增删改的业务流程中。

---

## 8.3 Guava Cache 不是分布式缓存

假设你有 3 个服务实例：

```text
product-service-1
product-service-2
product-service-3
```

每个实例都有自己的本地缓存。

当后台修改分类名称后：

```text
实例 1 缓存更新了
实例 2 可能还是旧数据
实例 3 可能还是旧数据
```

所以本地缓存天然有一致性问题。

解决方案：

|一致性要求|方案|
|---|---|
|允许短暂不一致|设置较短过期时间|
|修改后尽快生效|更新后主动 invalidate|
|多实例同步失效|MQ / Redis PubSub 广播失效|
|强一致|不要用本地缓存，直接查 DB / Redis|
|高并发共享热点|Redis / Caffeine + Redis 二级缓存|

---

## 8.4 本地缓存不要缓存大对象

不要缓存：

- 大列表
    
- 大 JSON
    
- 大文件内容
    
- 用户维度海量数据
    
- 频繁变化的数据
    

错误示例：

```java
Cache<Long, List<OrderDTO>> userOrderCache;
```

如果一个用户有几万订单，这个缓存很容易把 JVM 内存打爆。

---

## 8.5 Guava 版本冲突问题

Guava 是很多 Java 生态库的传递依赖。容易出现：

```text
A 依赖 guava 18
B 依赖 guava 31
你的项目直接依赖 guava 33
```

可能导致：

- `NoSuchMethodError`
    
- `ClassNotFoundException`
    
- 运行时行为不一致
    
- 依赖树混乱
    

建议在 Maven 里统一版本：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.google.guava</groupId>
            <artifactId>guava</artifactId>
            <version>33.4.8-jre</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

排查命令：

```bash
mvn dependency:tree | grep guava
```

---

## 8.6 和 Apache Commons、Hutool、JDK 的取舍

### JDK 优先

Java 17 / 21 已有能力时，优先 JDK。

```java
Objects.requireNonNull(value);
List.of("A", "B");
Map.of("k1", "v1");
Files.readString(path);
String.join(",", values);
```

### Spring 项目优先 Spring 工具

Spring 项目里很多场景可以用：

```java
StringUtils.hasText(value);
CollectionUtils.isEmpty(list);
Assert.notNull(value, "不能为空");
```

### Hutool 谨慎使用

Hutool 很方便，但在严肃企业项目中要控制边界：

- 不要让 Hutool 替代业务建模
    
- 不要到处静态工具类乱飞
    
- 不要引入大量隐式行为
    
- 不要为了一个小工具引入过多依赖面
    

### Apache Commons

Apache Commons 适合：

- `StringUtils`
    
- `CollectionUtils`
    
- `DigestUtils`
    
- `FileUtils`
    
- `IOUtils`
    

但它和 Guava、JDK、Spring 工具类会重叠。项目里最好有统一规范。

---

# 总结表：Guava 模块 / 典型场景 / 是否推荐 / 替代方案

|Guava 模块|典型场景|是否推荐|替代方案|
|---|---|--:|---|
|`ImmutableList` / `ImmutableMap` / `ImmutableSet`|固定配置、只读规则、常量集合|推荐|JDK `List.of` / `Map.of` / `Set.of`|
|`Multimap`|一个 key 对多个 value，如用户角色、分类商品|推荐|`Map<K, List<V>>`|
|`Multiset`|频次统计，如错误码、标签、关键词|推荐|`Map<T, Integer>` / Redis|
|`BiMap`|双向映射，如 code ↔ name|可推荐|两个 Map|
|`Table`|二维映射，如商品 + 渠道 → 价格|可推荐|`Map<R, Map<C, V>>`|
|`Strings`|null / 空字符串处理|可用|Spring `StringUtils` / JDK|
|`Joiner`|集合拼接字符串，处理 null 更方便|可用|`String.join` / Stream|
|`Splitter`|参数拆分、trim、过滤空值|推荐|`String.split` + Stream|
|`Preconditions`|Service 层参数与状态校验|可推荐|Spring `Assert` / JDK `Objects`|
|`Guava Cache`|简单本地缓存、老项目维护|老项目可用，新项目不优先|Caffeine|
|`ListenableFuture`|老异步代码回调|不推荐新项目优先用|`CompletableFuture` / Reactor|
|`RateLimiter`|单机限流、批任务限速|可用，但边界清楚|Sentinel / Redis / Gateway|
|`Hashing`|非安全哈希、缓存 key、分片 key|可用|JDK `MessageDigest` / Apache Commons Codec|
|`Files` / `Resources` / `ByteStreams`|小文件读取、classpath 资源、流复制|可用|JDK `java.nio.file.Files`|
|`Optional`|老项目兼容|不推荐|`java.util.Optional`|

---

## 最终建议

Guava 在今天的 Java 后端项目里，最值得保留的是：

```text
Immutable Collections
Multimap
Multiset
BiMap
Table
Splitter
Preconditions
RateLimiter
```

最需要谨慎的是：

```text
Guava Cache
ListenableFuture
Optional
I/O 工具类
```

一句话判断：

> **Guava 的集合增强仍然有工程价值；Guava 的缓存、并发、Optional 等能力在新项目里要优先考虑 JDK、Caffeine、Spring 生态的替代方案。**