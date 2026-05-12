```
你是一名资深 Java 微服务架构师，请系统讲解 Apache Dubbo 中的一致性哈希负载均衡机制。

请按以下结构回答：

1. 背景引入
   - 什么是负载均衡
   - 为什么普通哈希取模在节点变化时会产生大量请求迁移
   - 一致性哈希要解决什么问题

2. 一致性哈希原理
   - 哈希环是什么
   - 请求 key 如何映射到服务节点
   - 节点新增、删除时为什么只影响局部请求
   - 虚拟节点的作用是什么

3. Dubbo 中的一致性哈希实现
   - Dubbo 的 ConsistentHashLoadBalance 的核心思路
   - 它默认使用哪些参数作为 hash key
   - `hash.arguments` 的作用
   - `hash.nodes` 的作用
   - 虚拟节点在 Dubbo 中如何体现

4. 结合源码讲解
   - 以 Dubbo 3.x 的实现为主
   - 解释核心类、核心方法和关键数据结构
   - 用伪代码或简化 Java 代码说明流程

5. 示例说明
   - 假设有 3 个 Provider：A、B、C
   - 请求参数为 userId
   - 展示相同 userId 如何稳定路由到同一个 Provider
   - 展示新增 Provider D 后，哪些请求可能会迁移

6. 配置示例
   - XML 配置
   - 注解配置
   - application.yml / properties 配置
   - 展示如何指定按某个参数做一致性哈希

7. 适用场景
   - 会话黏滞
   - 缓存命中优化
   - 有状态服务路由
   - 用户维度分片

8. 注意事项
   - Provider 数量变化时仍会有少量迁移
   - 参数选择不合理会导致热点
   - 一致性哈希不等于绝对均匀
   - 和随机、轮询、最少活跃数负载均衡的区别

9. 面试视角总结
   - 用 3 到 5 句话总结 Dubbo 一致性哈希
   - 给出常见面试题和参考答案

请用中文回答，要求：
- 先讲原理，再讲 Dubbo 实现
- 用 Java 后端开发者能理解的方式说明
- 多用图示化描述、类比和代码
- 不要只停留在概念，要结合 Dubbo 配置和源码流程


```

# 0. 结论先行

Dubbo 的一致性哈希负载均衡，本质是：

> **把“请求参数”而不是“请求次数”作为路由依据，让相同业务 key 的请求稳定落到同一个 Provider 上。**

它解决的不是“绝对平均分配流量”，而是：

- 相同 `userId` / `tenantId` / `sessionId` 尽量打到同一台机器；
    
- Provider 上下线时，只让少量 key 迁移；
    
- 通过虚拟节点减少流量倾斜；
    
- 适合会话黏滞、缓存命中、有状态路由、用户分片等场景。
    

Dubbo 官方文档也明确说明：ConsistentHash 的特点是**相同参数的请求总是发到同一提供者**；当某台 Provider 挂掉时，原本发往该 Provider 的请求会基于虚拟节点平摊到其他 Provider，避免剧烈变动；默认只对第一个参数 Hash，默认虚拟节点数是 160。([Apache Dubbo](https://dubbo.apache.org/zh-cn/docs/advanced/loadbalance/ "负载均衡 | Apache Dubbo"))

---

# 1. 背景引入：为什么需要一致性哈希

## 1.1 什么是负载均衡

Dubbo 是典型的 RPC 框架。一个 Consumer 调用某个接口时，注册中心可能返回多个 Provider：

```text
UserService
  ├── Provider A: 10.0.0.1:20880
  ├── Provider B: 10.0.0.2:20880
  └── Provider C: 10.0.0.3:20880
```

**负载均衡**要解决的问题是：

> Consumer 这次 RPC 调用，到底应该发给哪一个 Provider？

Dubbo 的负载均衡是在 **Consumer 侧**完成的，也就是客户端拿到 Provider 列表后，由 Consumer 本地选择目标 Provider。Dubbo 官方文档也说明，Dubbo 提供的是客户端负载均衡，由 Consumer 通过负载均衡算法决定请求提交到哪个 Provider 实例。([Apache Dubbo](https://dubbo.apache.org/zh-cn/docs/advanced/loadbalance/ "负载均衡 | Apache Dubbo"))

常见策略包括：

|策略|目标|
|---|---|
|Random|随机分配，长期看比较均匀|
|RoundRobin|按顺序轮询|
|LeastActive|优先选择当前活跃调用数少的 Provider|
|ShortestResponse|优先选择响应时间短的 Provider|
|ConsistentHash|相同参数稳定路由到同一个 Provider|

Dubbo 当前文档列出的内置负载均衡算法包括 Random、RoundRobin、LeastActive、ShortestResponse、ConsistentHash。([Apache Dubbo](https://dubbo.apache.org/zh-cn/docs/advanced/loadbalance/ "负载均衡 | Apache Dubbo"))

---

## 1.2 普通哈希取模的问题

最朴素的做法是：

```java
int index = hash(userId) % providerList.size();
Provider provider = providerList.get(index);
```

假设现在有 3 个 Provider：

```text
providers = [A, B, C]
index = hash(userId) % 3
```

例如：

|userId|hash(userId)|`% 3`|Provider|
|---|--:|--:|---|
|101|101|2|C|
|102|102|0|A|
|103|103|1|B|
|104|104|2|C|

现在新增一个 Provider D：

```text
providers = [A, B, C, D]
index = hash(userId) % 4
```

结果变成：

|userId|hash(userId)|原来 `% 3`|新的 `% 4`|是否迁移|
|---|--:|--:|--:|---|
|101|101|2 -> C|1 -> B|迁移|
|102|102|0 -> A|2 -> C|迁移|
|103|103|1 -> B|3 -> D|迁移|
|104|104|2 -> C|0 -> A|迁移|

问题在于：

> **节点数从 3 变成 4 后，几乎所有 key 的取模结果都可能变化。**

这会导致：

- 用户会话从 A 跑到 C；
    
- 本地缓存命中率大幅下降；
    
- 有状态 Provider 上的数据关联失效；
    
- 下游状态服务被打穿；
    
- 扩容、缩容、上下线 Provider 时产生大量请求迁移。
    

---

## 1.3 一致性哈希要解决什么问题

一致性哈希的目标不是“完全不迁移”，而是：

> **当 Provider 数量变化时，只让一小部分 key 迁移，其他 key 保持原来的路由结果。**

对比一下：

|方案|节点变化时的影响|
|---|---|
|`hash(key) % N`|大量 key 重新分布|
|一致性哈希|只影响新增/删除节点附近的一段 key|

所以一致性哈希特别适合：

```text
相同 userId     -> 尽量固定 Provider
相同 tenantId   -> 尽量固定 Provider
相同 sessionId  -> 尽量固定 Provider
相同 shopId     -> 尽量固定 Provider
```

---

# 2. 一致性哈希原理

## 2.1 哈希环是什么

一致性哈希会把哈希空间想象成一个环。

以 32 位无符号整数为例：

```text
0 ~ 2^32 - 1
```

首尾相接后形成一个环：

```text
                 0
                 |
      [C]        |        [A]
                 |
  2^32-1 --------+-------- 中间值
                 |
      [B]        |
                 |
```

更抽象一点：

```text
             0
             |
       C     |      A
             |
             |
  -----------+-----------
             |
             |
       B     |
             |
```

Provider 节点也会被 hash 到这个环上：

```java
hash("10.0.0.1:20880") -> 900
hash("10.0.0.2:20880") -> 2100
hash("10.0.0.3:20880") -> 3300
```

得到：

```text
hash ring:

0
|
|---- A(900) ---- B(2100) ---- C(3300) ---- 回到 0
```

---

## 2.2 请求 key 如何映射到服务节点

请求也会根据业务 key 做 hash：

```java
hash(userId)
```

然后在环上顺时针找第一个 Provider。

例如：

```text
0 ---- A(900) ---- B(2100) ---- C(3300) ---- 回到 0
        ^            ^             ^
        |            |             |
      Provider A   Provider B    Provider C
```

请求 key：

```text
userId=1001 -> hash=1000
```

顺时针找第一个节点：

```text
1000 后面第一个 Provider 是 B
```

所以：

```text
userId=1001 -> Provider B
```

再例如：

```text
userId=1002 -> hash=3400
```

3400 后面没有节点，就回到环的起点：

```text
3400 -> 回到 0 -> A
```

所以：

```text
userId=1002 -> Provider A
```

核心规则：

```text
请求 key hash 后，顺时针找第一个 Provider。
找不到就回到环的第一个 Provider。
```

---

## 2.3 节点新增、删除时为什么只影响局部请求

### 新增节点

原来：

```text
0 ---- A ---- B ---- C ---- 回到 0
```

新增 D，D 落在 A 和 B 之间：

```text
0 ---- A ---- D ---- B ---- C ---- 回到 0
```

原来 A 到 B 之间的请求都归 B：

```text
A ---- key1 ---- key2 ---- B
```

新增 D 后：

```text
A ---- key1 ---- D ---- key2 ---- B
```

只有这一小段会变化：

```text
A ~ D 之间的 key -> D
D ~ B 之间的 key -> 仍然 B
```

其他区间完全不变。

### 删除节点

原来：

```text
0 ---- A ---- B ---- C ---- 回到 0
```

删除 B：

```text
0 ---- A -------- C ---- 回到 0
```

原来归 B 的那一段请求，会顺时针迁移给 C：

```text
A ~ B 之间的 key：原来给 B，现在给 C
```

其他区间不变。

所以一致性哈希的迁移范围是局部的。

---

## 2.4 虚拟节点的作用

如果每个真实 Provider 只在环上放一个点，会出现严重倾斜。

例如：

```text
0 ---- A ---------------- B -- C ---- 回到 0
```

A 到 B 这段特别长，意味着 B 承接了很大范围的 key。

为了解决这个问题，引入**虚拟节点**。

真实节点 A 不只放一个点，而是放多个点：

```text
A#0, A#1, A#2, A#3 ...
B#0, B#1, B#2, B#3 ...
C#0, C#1, C#2, C#3 ...
```

环上变成：

```text
0 -- A#1 -- C#3 -- B#2 -- A#4 -- C#1 -- B#0 -- A#2 -- ...
```

每个虚拟节点都指向真实 Provider：

```text
A#1 -> A
A#2 -> A
A#3 -> A
```

这样做的作用：

|作用|解释|
|---|---|
|减少倾斜|真实节点在环上分布更多位置|
|提升均匀性|key 更容易均匀落到不同 Provider|
|降低节点变化冲击|新增/删除节点后，影响被分散到多个小区间|
|支持权重思路|理论上可以给高权重机器更多虚拟节点|

---

# 3. Dubbo 中的一致性哈希实现

## 3.1 Dubbo 的核心思路

Dubbo 的 `ConsistentHashLoadBalance` 大致流程是：

```text
Consumer 发起 RPC
        |
        v
Cluster Invoker 拿到当前 Provider 列表
        |
        v
LoadBalance.select()
        |
        v
ConsistentHashLoadBalance.doSelect()
        |
        v
为当前 service + method 构建 / 复用一致性哈希选择器
        |
        v
从 invocation 参数里取 hash key
        |
        v
MD5(hash key)
        |
        v
在 TreeMap 维护的哈希环中找顺时针第一个虚拟节点
        |
        v
返回虚拟节点对应的真实 Invoker
```

Dubbo 官方文档说明，启用一致性哈希可以通过 `@DubboReference(loadbalance = "consistenthash")`、`referenceConfig.setLoadBalance("consistenthash")`、`dubbo.reference.loadbalance=consistenthash`、`<dubbo:reference loadbalance="consistenthash" />` 等方式配置。([Apache Dubbo](https://dubbo.apache.org/en/docs3-v2/java-sdk/advanced-features-and-usage/service/consistent-hash/ "Consistent Hash Site Selection | Apache Dubbo"))

---

## 3.2 默认使用哪些参数作为 hash key

Dubbo 默认使用**第一个方法参数**作为 hash key。

也就是说，如果接口是：

```java
UserProfile getProfile(Long userId, String traceId);
```

默认 hash key 是：

```text
args[0] = userId
```

所以相同 `userId` 会稳定路由到同一个 Provider。

Dubbo 官方文档明确说明：默认使用第一个参数作为 hash key；如需切换参数，可以指定 `hash.arguments` 属性。([Apache Dubbo](https://dubbo.apache.org/en/docs3-v2/java-sdk/advanced-features-and-usage/service/consistent-hash/ "Consistent Hash Site Selection | Apache Dubbo"))

---

## 3.3 `hash.arguments` 的作用

`hash.arguments` 用来指定：

> 方法参数列表中，哪些参数参与一致性哈希。

参数下标从 `0` 开始。

例如：

```java
OrderDTO queryOrder(Long userId, Long orderId, String traceId);
```

参数位置：

```text
0 -> userId
1 -> orderId
2 -> traceId
```

如果配置：

```text
hash.arguments=0
```

表示按 `userId` 做一致性哈希。

如果配置：

```text
hash.arguments=1
```

表示按 `orderId` 做一致性哈希。

如果配置：

```text
hash.arguments=0,1
```

表示把 `userId + orderId` 拼起来做 hash key。

Dubbo 源码里会把这些参数值 append 到一个 `StringBuilder` 中：

```java
private String toKey(Object[] args) {
    StringBuilder buf = new StringBuilder();
    for (int i : argumentIndex) {
        if (i >= 0 && args != null && i < args.length) {
            buf.append(args[i]);
        }
    }
    return buf.toString();
}
```

Dubbo 3.3.0 的 `ConsistentHashLoadBalance` 源码中，`HASH_ARGUMENTS` 常量名为 `"hash.arguments"`，默认值是 `"0"`，并通过 `url.getMethodParameter(methodName, HASH_ARGUMENTS, "0")` 读取。([GitHub](https://raw.githubusercontent.com/apache/dubbo/dubbo-3.3.0/dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/loadbalance/ConsistentHashLoadBalance.java "raw.githubusercontent.com"))

---

## 3.4 `hash.nodes` 的作用

`hash.nodes` 用来指定：

> 每个真实 Provider 对应多少个虚拟节点。

Dubbo 默认值是：

```text
hash.nodes=160
```

也就是说：

```text
Provider A -> 160 个虚拟节点
Provider B -> 160 个虚拟节点
Provider C -> 160 个虚拟节点
```

Dubbo 官方文档说明，ConsistentHash 缺省使用 160 份虚拟节点，可以通过 `hash.nodes` 修改。([Apache Dubbo](https://dubbo.apache.org/zh-cn/docs/advanced/loadbalance/ "负载均衡 | Apache Dubbo"))

---

## 3.5 虚拟节点在 Dubbo 中如何体现

Dubbo 不是显式创建一个 `VirtualNode` 类，而是用 `TreeMap<Long, Invoker<?>>` 表示哈希环：

```java
private final TreeMap<Long, Invoker<T>> virtualInvokers;
```

其中：

```text
key   -> 虚拟节点在哈希环上的位置
value -> 真实 Provider Invoker
```

例如：

```text
1001 -> Invoker A
2088 -> Invoker A
3901 -> Invoker A
4567 -> Invoker B
6988 -> Invoker C
```

Dubbo 3.3.0 源码中，每个 Provider 会根据地址和序号生成 MD5，`replicaNumber / 4` 次循环，每次 MD5 摘要切成 4 段，生成 4 个 long 值，因此总数约等于 `replicaNumber`。([GitHub](https://raw.githubusercontent.com/apache/dubbo/dubbo-3.3.0/dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/loadbalance/ConsistentHashLoadBalance.java "raw.githubusercontent.com"))

简化理解：

```java
for (Invoker invoker : invokers) {
    String address = invoker.getUrl().getAddress();

    for (int i = 0; i < replicaNumber / 4; i++) {
        byte[] digest = md5(address + i);

        for (int h = 0; h < 4; h++) {
            long virtualNodeHash = hash(digest, h);
            virtualInvokers.put(virtualNodeHash, invoker);
        }
    }
}
```

如果 `replicaNumber = 160`：

```text
160 / 4 = 40 次 MD5
每次 MD5 生成 4 个 hash 点
共 160 个虚拟节点
```

---

# 4. 结合 Dubbo 3.x 源码讲解

## 4.1 核心类

Dubbo 3.x 中核心类是：

```java
org.apache.dubbo.rpc.cluster.loadbalance.ConsistentHashLoadBalance
```

它继承：

```java
AbstractLoadBalance
```

并实现核心选择逻辑：

```java
protected <T> Invoker<T> doSelect(
    List<Invoker<T>> invokers,
    URL url,
    Invocation invocation
)
```

Dubbo 的负载均衡扩展点接口是 `org.apache.dubbo.rpc.cluster.LoadBalance`，官方扩展点文档也列出了 `ConsistentHashLoadBalance` 作为已知扩展实现之一。([Apache Dubbo](https://dubbo.apache.org/en/overview/mannual/java-sdk/reference-manual/spi/description/load-balance/ "Load Balance Extension | Apache Dubbo"))

---

## 4.2 核心数据结构

### `selectors`

```java
private final ConcurrentMap<String, ConsistentHashSelector<?>> selectors
    = new ConcurrentHashMap<>();
```

作用：

> 缓存不同 service + method 对应的哈希选择器。

key 类似：

```text
com.demo.UserService.getProfile
```

为什么要缓存？

因为构建哈希环是有成本的：

```text
Provider 列表 -> 生成虚拟节点 -> 放入 TreeMap
```

不应该每次请求都重新构建。

---

### `ConsistentHashSelector`

这是内部选择器，核心字段包括：

```java
private final TreeMap<Long, Invoker<T>> virtualInvokers;
private final int replicaNumber;
private final int identityHashCode;
private final int[] argumentIndex;
```

含义：

|字段|作用|
|---|---|
|`virtualInvokers`|哈希环，key 是虚拟节点 hash，value 是真实 Invoker|
|`replicaNumber`|虚拟节点数量，默认 160|
|`identityHashCode`|Provider 列表 hash，用于判断 Provider 列表是否变化|
|`argumentIndex`|哪些参数参与 hash，由 `hash.arguments` 决定|

Dubbo 3.3.0 源码中，`ConsistentHashSelector` 内部使用 `TreeMap<Long, Invoker<T>> virtualInvokers` 维护虚拟节点环，使用 `replicaNumber` 读取 `hash.nodes`，使用 `argumentIndex` 保存 `hash.arguments` 解析结果。([GitHub](https://raw.githubusercontent.com/apache/dubbo/dubbo-3.3.0/dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/loadbalance/ConsistentHashLoadBalance.java "raw.githubusercontent.com"))

---

## 4.3 `doSelect()` 方法流程

Dubbo 3.3.0 源码核心逻辑可以简化成：

```java
@Override
protected <T> Invoker<T> doSelect(
        List<Invoker<T>> invokers,
        URL url,
        Invocation invocation) {

    // 1. 获取当前调用的方法名
    String methodName = RpcUtils.getMethodName(invocation);

    // 2. 使用 serviceKey + methodName 作为选择器缓存 key
    String key = invokers.get(0).getUrl().getServiceKey() + "." + methodName;

    // 3. 计算当前 Provider 列表的 hash
    int invokersHashCode = invokers.hashCode();

    // 4. 从缓存中取一致性哈希选择器
    ConsistentHashSelector<T> selector =
        (ConsistentHashSelector<T>) selectors.get(key);

    // 5. 如果没有选择器，或者 Provider 列表变化了，则重建哈希环
    if (selector == null || selector.identityHashCode != invokersHashCode) {
        selectors.put(
            key,
            new ConsistentHashSelector<T>(
                invokers,
                methodName,
                invokersHashCode
            )
        );
        selector = (ConsistentHashSelector<T>) selectors.get(key);
    }

    // 6. 用选择器选择具体 Invoker
    return selector.select(invocation);
}
```

重点：

```text
service + method 维度缓存哈希环
Provider 列表变化时重建哈希环
请求进来后，根据参数选择 Invoker
```

Dubbo 3.3.0 的实现使用 `invokers.hashCode()` 计算 Provider 列表 hash，并在 selector 为空或 hash 变化时新建 `ConsistentHashSelector`。([GitHub](https://raw.githubusercontent.com/apache/dubbo/dubbo-3.3.0/dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/loadbalance/ConsistentHashLoadBalance.java "raw.githubusercontent.com"))

---

## 4.4 构造哈希环

简化代码：

```java
ConsistentHashSelector(
        List<Invoker<T>> invokers,
        String methodName,
        int identityHashCode) {

    this.virtualInvokers = new TreeMap<>();
    this.identityHashCode = identityHashCode;

    URL url = invokers.get(0).getUrl();

    // 1. 读取 hash.nodes，默认 160
    this.replicaNumber =
        url.getMethodParameter(methodName, "hash.nodes", 160);

    // 2. 读取 hash.arguments，默认 "0"
    String[] index = COMMA_SPLIT_PATTERN.split(
        url.getMethodParameter(methodName, "hash.arguments", "0")
    );

    // 3. 解析参数下标
    argumentIndex = new int[index.length];
    for (int i = 0; i < index.length; i++) {
        argumentIndex[i] = Integer.parseInt(index[i]);
    }

    // 4. 为每个真实 Provider 创建虚拟节点
    for (Invoker<T> invoker : invokers) {
        String address = invoker.getUrl().getAddress();

        for (int i = 0; i < replicaNumber / 4; i++) {
            byte[] digest = md5(address + i);

            for (int h = 0; h < 4; h++) {
                long m = hash(digest, h);
                virtualInvokers.put(m, invoker);
            }
        }
    }
}
```

这个环的结构可以理解成：

```text
TreeMap<Long, Invoker>

100023      -> A
289012      -> B
920113      -> A
1033921     -> C
1982333     -> B
...
```

`TreeMap` 的意义是：

> 可以快速找到“大于等于请求 hash 的第一个虚拟节点”。

---

## 4.5 选择 Provider

源码核心：

```java
public Invoker<T> select(Invocation invocation) {
    String key = toKey(RpcUtils.getArguments(invocation));
    byte[] digest = Bytes.getMD5(key);
    return selectForKey(hash(digest, 0));
}
```

流程：

```text
Invocation
   |
   v
取参数数组 args
   |
   v
根据 hash.arguments 拼接 hash key
   |
   v
MD5(hash key)
   |
   v
取 MD5 的一段转成 long
   |
   v
在 TreeMap 中找 ceilingEntry(hash)
   |
   v
返回 entry.value，也就是真实 Invoker
```

`selectForKey()` 简化如下：

```java
private Invoker<T> selectForKey(long hash) {
    Map.Entry<Long, Invoker<T>> entry = virtualInvokers.ceilingEntry(hash);

    if (entry == null) {
        entry = virtualInvokers.firstEntry();
    }

    return entry.getValue();
}
```

图示：

```text
hash ring in TreeMap:

1000 -> A
2300 -> C
3600 -> B
4800 -> A

请求 hash = 2500

ceilingEntry(2500) = 3600 -> B

所以请求路由到 B
```

Dubbo 3.3.0 源码中，选择逻辑就是对请求 key 做 MD5，然后通过 `virtualInvokers.ceilingEntry(hash)` 找到环上顺时针第一个节点；如果不存在，则回到 `firstEntry()`。([GitHub](https://raw.githubusercontent.com/apache/dubbo/dubbo-3.3.0/dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/loadbalance/ConsistentHashLoadBalance.java "raw.githubusercontent.com"))

---

# 5. 示例说明：A、B、C 三个 Provider

假设接口：

```java
public interface UserSessionService {
    SessionInfo getSession(Long userId);
}
```

Consumer 配置：

```text
loadbalance = consistenthash
hash.arguments = 0
```

Provider：

```text
A = 10.0.0.1:20880
B = 10.0.0.2:20880
C = 10.0.0.3:20880
```

---

## 5.1 构建哈希环

为了方便说明，假设简化后的虚拟节点位置如下：

```text
0 ---- A1 ---- C1 ---- B1 ---- A2 ---- C2 ---- B2 ---- 回到 0
```

展开：

```text
100  -> A
300  -> C
500  -> B
700  -> A
850  -> C
950  -> B
```

---

## 5.2 相同 userId 稳定路由

请求：

```java
getSession(10001L)
```

Dubbo 取第 0 个参数：

```text
hash key = "10001"
```

假设：

```text
hash("10001") = 420
```

在环上找顺时针第一个节点：

```text
420 后面第一个节点是 500 -> B
```

所以：

```text
userId=10001 -> B
```

下一次还是：

```java
getSession(10001L)
```

只要 Provider 列表没有变化：

```text
hash("10001") 仍然是 420
ceilingEntry(420) 仍然是 500 -> B
```

所以仍然路由到 B。

这就是：

```text
相同参数 -> 相同 Provider
```

---

## 5.3 不同 userId 分散到不同 Provider

假设：

|userId|hash|顺时针节点|Provider|
|---|--:|--:|---|
|10001|420|500|B|
|10002|120|300|C|
|10003|760|850|C|
|10004|910|950|B|
|10005|50|100|A|

注意：

> 一致性哈希不是按请求次数轮询，而是按 key 的 hash 分布。

所以短时间内不一定均匀。

---

## 5.4 新增 Provider D 后，哪些请求会迁移

新增 D 后，D 的虚拟节点插入环中：

原来：

```text
100(A) ---- 300(C) ---- 500(B) ---- 700(A) ---- 850(C) ---- 950(B)
```

新增：

```text
100(A) ---- 300(C) ---- 400(D) ---- 500(B) ---- 700(A) ---- 850(C) ---- 950(B)
```

原来区间：

```text
300 ~ 500 之间的请求 -> B
```

新增 D 后：

```text
300 ~ 400 之间的请求 -> D
400 ~ 500 之间的请求 -> B
```

所以只有落在这个局部区间的 key 会迁移。

例如：

|userId|hash|原 Provider|新 Provider|是否迁移|
|---|--:|---|---|---|
|10001|420|B|B|不迁移|
|10006|350|B|D|迁移|
|10007|390|B|D|迁移|
|10008|760|C|C|不迁移|
|10009|50|A|A|不迁移|

这就是一致性哈希相比普通取模的核心价值：

```text
节点新增 / 删除，只影响环上相邻区间。
```

---

# 6. 配置示例

下面示例以接口为例：

```java
public interface UserSessionService {

    SessionInfo getSession(Long userId);

    OrderInfo queryOrder(Long userId, Long orderId);
}
```

---

## 6.1 XML 配置：按第一个参数 userId 哈希

```xml
<dubbo:reference
        id="userSessionService"
        interface="com.demo.api.UserSessionService"
        loadbalance="consistenthash">

    <!-- 默认就是 0，这里显式写出来 -->
    <dubbo:parameter key="hash.arguments" value="0"/>

    <!-- 默认 160，这里显式写出来 -->
    <dubbo:parameter key="hash.nodes" value="160"/>
</dubbo:reference>
```

含义：

```text
getSession(Long userId)
             ^
             第 0 个参数参与 hash
```

---

## 6.2 XML 配置：方法级别指定参数

如果接口里多个方法需要不同 hash 参数：

```xml
<dubbo:reference
        id="userSessionService"
        interface="com.demo.api.UserSessionService"
        loadbalance="consistenthash">

    <dubbo:method name="getSession">
        <dubbo:parameter key="hash.arguments" value="0"/>
        <dubbo:parameter key="hash.nodes" value="160"/>
    </dubbo:method>

    <dubbo:method name="queryOrder">
        <!-- queryOrder(Long userId, Long orderId)，按 orderId 哈希 -->
        <dubbo:parameter key="hash.arguments" value="1"/>
        <dubbo:parameter key="hash.nodes" value="160"/>
    </dubbo:method>

</dubbo:reference>
```

Dubbo 文档中也给出过类似思路：可以配置全局 `hash.arguments`，也可以配置方法级别的 `sayHello.hash.arguments`。([Apache Dubbo](https://dubbo.apache.org/en/docs3-v2/java-sdk/advanced-features-and-usage/service/consistent-hash/ "Consistent Hash Site Selection | Apache Dubbo"))

---

## 6.3 注解配置：基础用法

```java
@Component
public class UserController {

    @DubboReference(loadbalance = "consistenthash")
    private UserSessionService userSessionService;

    public SessionInfo getSession(Long userId) {
        return userSessionService.getSession(userId);
    }
}
```

这种方式会启用一致性哈希，默认使用第一个参数。

---

## 6.4 注解配置：指定 hash.arguments / hash.nodes

Dubbo 注解里可以通过 `parameters` 传递扩展参数。示例：

```java
@DubboReference(
    loadbalance = "consistenthash",
    parameters = {
        "hash.arguments", "0",
        "hash.nodes", "160"
    }
)
private UserSessionService userSessionService;
```

如果要按第二个参数 `orderId`：

```java
@DubboReference(
    loadbalance = "consistenthash",
    parameters = {
        "hash.arguments", "1",
        "hash.nodes", "160"
    }
)
private UserSessionService userSessionService;
```

---

## 6.5 application.properties 配置

全局 Reference 默认负载均衡：

```properties
dubbo.reference.loadbalance=consistenthash
```

Dubbo 官方一致性哈希文档给出的 properties 配置方式就是：

```properties
dubbo.reference.loadbalance=consistenthash
```

([Apache Dubbo](https://dubbo.apache.org/en/docs3-v2/java-sdk/advanced-features-and-usage/service/consistent-hash/ "Consistent Hash Site Selection | Apache Dubbo"))

如果要配置参数，常见做法是结合注解 `parameters` 或 XML `dubbo:parameter`，因为 `hash.arguments` / `hash.nodes` 本质是 URL 参数，最终要进入 Dubbo URL 的 method parameter 中。

---

## 6.6 application.yml 配置

基础启用：

```yaml
dubbo:
  reference:
    loadbalance: consistenthash
```

如果你使用 Spring Boot + 注解方式，建议把关键参数直接放在 `@DubboReference(parameters = ...)` 上，语义更明确：

```java
@DubboReference(
    loadbalance = "consistenthash",
    parameters = {
        "hash.arguments", "0",
        "hash.nodes", "320"
    }
)
private UserSessionService userSessionService;
```

---

## 6.7 实战建议：按业务 key 配，而不是按技术参数配

假设方法：

```java
OrderInfo queryOrder(Long userId, Long orderId, String traceId);
```

不要按 `traceId`：

```java
parameters = {
    "hash.arguments", "2"
}
```

因为每次请求 `traceId` 都不同，会导致一致性哈希失去“稳定路由”的意义。

更合理：

```java
// 按用户维度黏滞
parameters = {
    "hash.arguments", "0"
}
```

或者：

```java
// 按订单维度黏滞
parameters = {
    "hash.arguments", "1"
}
```

---

# 7. 适用场景

## 7.1 会话黏滞

例如：

```java
SessionInfo getSession(Long userId);
```

如果 Provider 本地维护了用户 session 缓存：

```text
userId=10001 -> Provider B
```

那么后续请求继续到 B，可以减少跨节点查找。

适合：

```text
用户会话
购物车状态
游戏房间状态
临时登录态
```

注意：

> 真正生产系统中，核心状态仍然建议放 Redis / DB / 分布式缓存，不建议强依赖 Provider 本地内存。

一致性哈希只能提高命中率，不应该成为唯一可靠性手段。

---

## 7.2 缓存命中优化

例如 Provider 本地有 Caffeine 缓存：

```java
ProductDetail getProduct(Long productId);
```

按 `productId` 做一致性哈希：

```text
productId=888 -> Provider A
```

这样 Provider A 的本地缓存更容易命中。

适合：

```text
商品详情缓存
用户画像缓存
权限规则缓存
风控规则缓存
配置数据缓存
```

---

## 7.3 有状态服务路由

有些服务不是纯无状态的。

例如：

```text
WebSocket 网关
长连接网关
游戏房间服务
实时协作服务
本地状态计算服务
```

如果请求必须尽量回到持有状态的节点，一致性哈希有意义。

但要注意：

> Provider 宕机时，状态仍然会丢。必须配合状态迁移、持久化、重连、补偿机制。

---

## 7.4 用户维度分片

例如：

```java
UserRiskResult checkRisk(Long userId, RiskContext context);
```

可以让同一个用户的风控请求稳定落到同一个 Provider：

```text
userId=10001 -> A
userId=10002 -> B
userId=10003 -> C
```

好处：

- 用户维度缓存更容易命中；
    
- 用户行为上下文更集中；
    
- 降低跨节点状态同步成本；
    
- 可以做局部热点治理。
    

---

# 8. 注意事项

## 8.1 Provider 数量变化时仍然会有迁移

一致性哈希不是“不迁移”。

它只是把：

```text
大规模迁移
```

降低为：

```text
局部迁移
```

Provider 新增、删除、重启、注册中心推送地址变化，都会可能触发哈希环重建。

Dubbo 源码中会根据 Provider 列表的 hash 判断是否需要重建 `ConsistentHashSelector`。Provider 列表变化后，新的哈希环会影响部分 key 的路由结果。([GitHub](https://raw.githubusercontent.com/apache/dubbo/dubbo-3.3.0/dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/loadbalance/ConsistentHashLoadBalance.java "raw.githubusercontent.com"))

---

## 8.2 参数选择不合理会导致热点

错误示例：

```java
getByTenant(Long tenantId, Long userId)
```

如果按 `tenantId` 哈希：

```text
hash.arguments=0
```

但是某个大租户流量极大：

```text
tenantId=1 占 80% 请求
```

那么 80% 请求可能落到同一台 Provider 上。

更合理的选择可能是：

```text
hash.arguments=1   // 按 userId
```

或者：

```text
hash.arguments=0,1 // tenantId + userId
```

但注意 Dubbo 是直接 append 参数值：

```text
tenantId=1, userId=23 -> "123"
tenantId=12, userId=3 -> "123"
```

存在拼接歧义风险。虽然实际业务里不一定常见，但严谨场景下要注意。

如果你非常在意这个问题，可以考虑：

- 把业务 key 封装成一个明确的字符串参数；
    
- 自定义 LoadBalance；
    
- 在参数对象的 `toString()` 中保证稳定且无歧义；
    
- 或使用明确分隔符生成 routingKey。
    

---

## 8.3 一致性哈希不等于绝对均匀

一致性哈希的均匀性取决于：

```text
hash 函数质量
虚拟节点数量
Provider 数量
key 分布
业务流量分布
```

即使虚拟节点很多，如果业务 key 本身高度倾斜，也会出现热点。

例如：

```text
userId=10001 是超级大客户
```

那这个 key 对应的 Provider 仍然会被打爆。

一致性哈希解决的是：

```text
key 到节点的稳定映射
```

不是：

```text
热点 key 自动拆散
```

---

## 8.4 `hash.nodes` 不是越大越好

提高 `hash.nodes` 可以改善均匀性，但也会增加哈希环规模。

例如：

```text
Provider 数量 = 100
hash.nodes = 160
虚拟节点总数 = 16,000
```

一般可以接受。

但如果：

```text
Provider 数量很多
hash.nodes 配得过大
Provider 列表频繁变化
```

就会增加：

- 哈希环构建成本；
    
- TreeMap 存储成本；
    
- Consumer 侧 CPU 成本；
    
- 地址变化时的重建成本。
    

常规业务先用默认 160。确实观察到分布不均，再调到 320 或更高。

---

## 8.5 和随机、轮询、最少活跃数的区别

|策略|选择依据|是否稳定路由|是否关注负载状态|适合场景|
|---|---|--:|--:|---|
|Random|随机数 + 权重|否|否|默认通用场景|
|RoundRobin|轮询顺序 + 权重|否|否|请求耗时接近的场景|
|LeastActive|活跃调用数|否|是|Provider 性能差异明显|
|ShortestResponse|响应时间|否|是|追求低延迟|
|ConsistentHash|请求参数 hash|是|否|会话黏滞、缓存命中、有状态路由|

Dubbo 文档中对 LeastActive 的说明是：活跃数越低越优先调用，活跃数表示请求发送数与响应返回数的差，慢 Provider 会因活跃数较高而收到更少请求。([Apache Dubbo](https://dubbo.apache.org/zh-cn/docs/advanced/loadbalance/ "负载均衡 | Apache Dubbo"))

一句话区别：

```text
Random / RoundRobin 关心“怎么把请求分出去”
LeastActive / ShortestResponse 关心“谁现在更空 / 更快”
ConsistentHash 关心“这个 key 上次去哪，这次还去哪”
```

---

# 9. 面试视角总结

## 9.1 3 到 5 句话总结 Dubbo 一致性哈希

1. Dubbo 的一致性哈希负载均衡会根据请求参数计算 hash，让相同参数的请求稳定路由到同一个 Provider。
    
2. 它内部通过 `TreeMap<Long, Invoker>` 维护哈希环，Provider 会被映射成多个虚拟节点，默认每个 Provider 160 个虚拟节点。
    
3. 默认使用第一个参数作为 hash key，可以通过 `hash.arguments` 指定参与 hash 的参数，通过 `hash.nodes` 调整虚拟节点数量。
    
4. 当 Provider 新增或删除时，只会影响哈希环上相邻区间的一部分请求，避免普通取模造成的大规模请求迁移。
    
5. 它适合会话黏滞、本地缓存命中、有状态服务、用户维度分片，但不保证绝对均匀，也不能自动解决热点 key 问题。
    

---

## 9.2 常见面试题与参考答案

### Q1：Dubbo 一致性哈希负载均衡解决什么问题？

**参考答案：**

它主要解决相同业务参数的请求稳定路由问题。比如相同 `userId` 的请求尽量打到同一个 Provider，从而提高本地缓存命中率或支持一定程度的会话黏滞。同时相比普通 `hash % 节点数`，在 Provider 上下线时，一致性哈希只迁移局部 key，避免大面积请求重新分布。

---

### Q2：Dubbo 一致性哈希默认按哪个参数做 hash？

**参考答案：**

默认按第一个方法参数，也就是参数下标 `0`。如果要指定其他参数，可以通过 `hash.arguments` 配置，例如：

```xml
<dubbo:parameter key="hash.arguments" value="1"/>
```

表示按第二个参数做一致性哈希。

Dubbo 官方文档明确说明，默认使用第一个参数作为 hash key，可以通过 `hash.arguments` 切换参数。([Apache Dubbo](https://dubbo.apache.org/en/docs3-v2/java-sdk/advanced-features-and-usage/service/consistent-hash/ "Consistent Hash Site Selection | Apache Dubbo"))

---

### Q3：`hash.nodes` 是什么？

**参考答案：**

`hash.nodes` 表示每个真实 Provider 对应的虚拟节点数量，Dubbo 默认是 160。虚拟节点越多，Provider 在哈希环上的分布越分散，一般有助于减少流量倾斜。但它不是越大越好，过大可能增加哈希环构建和维护成本。

---

### Q4：Dubbo 一致性哈希底层用什么数据结构？

**参考答案：**

核心是 `TreeMap<Long, Invoker<T>>`。`Long` 表示虚拟节点在哈希环上的位置，`Invoker` 表示真实 Provider。请求进来后，Dubbo 根据参数生成 hash，然后用 `TreeMap.ceilingEntry(hash)` 找到顺时针方向第一个虚拟节点；如果找不到，就回到 `firstEntry()`。

---

### Q5：一致性哈希能不能保证负载绝对均匀？

**参考答案：**

不能。一致性哈希只能尽量让 key 分布更平滑，尤其依赖虚拟节点改善均匀性。但如果业务 key 本身倾斜，例如某个大用户或大租户占了大量请求，那么这些请求仍然可能集中到一个 Provider 上。它解决的是稳定路由和降低节点变化时的迁移量，不是热点自动拆分。

---

### Q6：一致性哈希和 LeastActive 怎么选？

**参考答案：**

如果目标是“相同 key 稳定路由”，选 ConsistentHash；如果目标是“让慢节点少接请求，让空闲节点多接请求”，选 LeastActive。ConsistentHash 不感知 Provider 当前负载，LeastActive 不保证相同参数落到同一节点。两者解决的问题不同。

---

# 10. 最后用一张图收束

```text
普通 hash 取模：

hash(userId) % 3 -> A/B/C

Provider 数量变成 4：

hash(userId) % 4 -> 大量 userId 路由结果变化


一致性哈希：

userId -> hash -> 哈希环 -> 顺时针第一个虚拟节点 -> Provider

新增 / 删除 Provider：

只影响新节点 / 删除节点附近的一小段 key
```

Dubbo 里的具体落点是：

```text
ConsistentHashLoadBalance
  ├── selectors: ConcurrentMap<String, ConsistentHashSelector>
  │
  └── ConsistentHashSelector
       ├── virtualInvokers: TreeMap<Long, Invoker>
       ├── replicaNumber: hash.nodes，默认 160
       ├── argumentIndex: hash.arguments，默认 0
       ├── select(invocation)
       ├── toKey(args)
       └── selectForKey(hash)
```

作为 Java 后端开发者，记住一句话就够了：

> **Dubbo 一致性哈希不是为了普通流量均衡，而是为了“按业务 key 稳定路由”；它牺牲了一部分动态负载感知能力，换来了更好的缓存命中、会话黏滞和节点变化时的低迁移成本。**