[arthas idea plugin](https://github.com/WangJi92/arthas-idea-plugin)
[xfg arthas教程](https://bugstack.cn/md/road-map/arthas.html)
[arthas 官方手册](https://arthas.aliyun.com/doc/quick-start.html)
## 0. 先给结论

**Arthas 不是用来“学习 JVM 原理”的工具，而是 Java 后端线上排障工具。**

它最适合解决这些问题：

|场景|你想知道什么|常用命令|
|---|---|---|
|服务突然变慢|哪个线程、哪个方法慢|`dashboard`、`thread`、`trace`|
|接口返回异常|方法入参、返回值、异常是什么|`watch`|
|代码和预期不一致|线上到底加载的是哪个类|`sc`、`sm`、`jad`|
|CPU 飙高|哪些线程在消耗 CPU|`thread -n`|
|怀疑死锁|是否有线程死锁|`thread -b`|
|怀疑配置没生效|Spring Bean / class / method 是否存在|`sc`、`sm`、`ognl`|
|想临时验证方法调用|直接调用线上对象方法|`ognl`|
|想看 JVM 状态|GC、内存、线程、类加载情况|`jvm`、`memory`、`dashboard`|

Arthas 官方定位就是 Java 线上诊断工具，特点是不改代码、不重启 JVM，就能观察线程、内存、GC、方法参数、返回值、异常、类加载等信息。([Arthas](https://arthas.aliyun.com/doc/?utm_source=chatgpt.com "简介 - arthas"))

---

# 1. 安装和启动：先 attach 到目标 Java 进程

## 1.1 下载并启动

常见方式：

```bash
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
```

启动后会列出当前机器上的 Java 进程，例如：

```text
[1]: 12345 com.zblog.Application
[2]: 23456 org.elasticsearch.bootstrap.Elasticsearch
```

选择你的业务进程编号即可。

官方快速入门也是通过 `arthas-boot.jar` attach 到目标 Java 进程，然后使用 `dashboard`、`thread`、`jad`、`watch` 等命令进行诊断。([Arthas](https://arthas.aliyun.com/doc/quick-start.html?utm_source=chatgpt.com "快速入门 - arthas"))

---

## 1.2 进入后先看基础信息

```bash
dashboard
```

你会看到：

```text
ID    NAME                      GROUP       PRIORITY   STATE      %CPU
1     main                      main        5          RUNNABLE   0.12
23    http-nio-8080-exec-1      main        5          WAITING    0.00

Memory              used    total    max
heap                512M    1024M    2048M
gc.ps_scavenge.count 123
gc.ps_marksweep.count 2
```

重点看三类信息：

|指标|怎么看|
|---|---|
|CPU|是否某些线程长期占用高|
|Memory|heap 是否持续上涨|
|GC|full GC 是否频繁|
|Thread|是否大量 BLOCKED / WAITING / RUNNABLE|

`dashboard` 是 Arthas 官方推荐的实时系统运行状态观察命令。([Arthas](https://arthas.aliyun.com/en/?utm_source=chatgpt.com "arthas: Home"))

---

# 2. 场景一：接口突然变慢，怎么定位？

假设你的接口：

```text
GET /api/articles/1001
```

对应后端方法：

```java
ArticleDetailVO getArticleDetail(Long articleId)
```

类名：

```java
com.zblog.article.service.ArticleService
```

---

## 2.1 先看线程是否有明显异常

```bash
thread
```

查看所有线程。

如果 CPU 飙高，直接看最耗 CPU 的线程：

```bash
thread -n 5
```

含义：

```text
查看 CPU 使用率最高的 5 个线程
```

如果怀疑死锁：

```bash
thread -b
```

含义：

```text
查看是否存在阻塞其他线程的线程
```

---

## 2.2 再看方法内部哪一步慢

使用 `trace`：

```bash
trace com.zblog.article.service.ArticleService getArticleDetail
```

输出类似：

```text
`---ts=2026-05-19 10:20:11;thread_name=http-nio-8080-exec-7;id=52;is_daemon=true;priority=5;TCCL=...
    `---[128.421ms] com.zblog.article.service.ArticleService:getArticleDetail()
        +---[12.381ms] com.zblog.article.mapper.ArticleMapper:selectById()
        +---[95.742ms] com.zblog.comment.client.CommentClient:listByArticleId()
        `---[8.219ms] com.zblog.user.client.UserClient:getAuthorInfo()
```

这个结果说明：

```text
ArticleService#getArticleDetail 总耗时 128ms
其中 CommentClient#listByArticleId 耗时 95ms
真正慢的是评论服务调用
```

### 什么时候用 `trace`？

|场景|是否适合|
|---|---|
|想知道方法整体耗时|适合|
|想知道方法内部哪个调用最慢|适合|
|想看入参和返回值|不适合，用 `watch`|
|想反编译代码|不适合，用 `jad`|

Arthas 4.1.3 对 `watch` / `trace` 等核心命令仍有增强，说明它们依然是 Arthas 的核心排障命令。([GitHub](https://github.com/alibaba/arthas/releases?utm_source=chatgpt.com "Releases · alibaba/arthas"))

---

# 3. 场景二：接口返回结果不对，怎么查入参和返回值？

假设你怀疑这个方法返回了错误数据：

```java
public ArticleDetailVO getArticleDetail(Long articleId)
```

使用 `watch`：

```bash
watch com.zblog.article.service.ArticleService getArticleDetail '{params, returnObj}' -x 3
```

含义：

```text
观察 ArticleService#getArticleDetail 方法
打印方法入参 params 和返回值 returnObj
对象展开深度为 3
```

输出类似：

```text
method=com.zblog.article.service.ArticleService.getArticleDetail location=AtExit
ts=2026-05-19 10:31:02; [cost=32.517ms] result=@ArrayList[
    @Object[][
        @Long[1001],
    ],
    @ArticleDetailVO[
        id=@Long[1001],
        title=@String[Arthas 使用教程],
        status=@Integer[0],
    ],
]
```

你能直接看到：

```text
入参 articleId = 1001
返回对象 status = 0
```

如果这个 status 本来应该是 `1`，问题就可以继续往下追。

---

## 3.1 只观察异常

如果接口偶发报错：

```bash
watch com.zblog.article.service.ArticleService getArticleDetail '{params, throwExp}' -e -x 3
```

含义：

```text
只在方法抛异常时输出
打印入参和异常对象
```

输出可能是：

```text
throwExp=@NullPointerException[
    detailMessage=@String[Cannot invoke "User.getName()" because "user" is null]
]
```

这比你临时加日志、重新发版、等待复现高效得多。

---

## 3.2 只观察耗时超过 200ms 的调用

```bash
watch com.zblog.article.service.ArticleService getArticleDetail '{params, returnObj, #cost}' '#cost > 200' -x 3
```

含义：

```text
只打印耗时超过 200ms 的调用
```

这个非常适合线上排查慢请求，否则高频接口会刷屏。

---

# 4. 场景三：线上代码和你以为的不一样，怎么确认？

这是 Arthas 很实用的地方。

很多线上问题不是业务逻辑复杂，而是：

```text
你以为发布了 A 版本，线上实际跑的是 B 版本。
你以为加载的是这个类，实际被另一个 jar 覆盖了。
你以为方法存在，实际没有。
```

---

## 4.1 查类是否被加载

```bash
sc com.zblog.article.service.ArticleService
```

输出类似：

```text
com.zblog.article.service.ArticleService
```

如果你不确定完整类名，可以模糊查：

```bash
sc *ArticleService
```

---

## 4.2 查类从哪个 jar 加载

```bash
sc -d com.zblog.article.service.ArticleService
```

重点看：

```text
code-source    /app/zblog-server.jar
class-loader   org.springframework.boot.loader.LaunchedURLClassLoader
```

如果你看到它来自一个意外的 jar，例如：

```text
/app/lib/old-zblog-article-1.0.0.jar
```

那就说明 classpath 或依赖冲突了。

Arthas 官方也明确把“类冲突、类加载路径定位”作为核心使用场景之一。([Arthas](https://arthas.aliyun.com/en/?utm_source=chatgpt.com "arthas: Home"))

---

## 4.3 查某个类有哪些方法

```bash
sm com.zblog.article.service.ArticleService
```

查详细方法签名：

```bash
sm -d com.zblog.article.service.ArticleService getArticleDetail
```

输出类似：

```text
public ArticleDetailVO getArticleDetail(java.lang.Long)
```

如果线上没有你新增的方法，说明：

```text
代码没有发上去
jar 包不是最新
服务没重启到新版本
容器镜像不是你以为的那个版本
```

---

## 4.4 反编译线上代码

```bash
jad com.zblog.article.service.ArticleService
```

只看某个方法：

```bash
jad com.zblog.article.service.ArticleService getArticleDetail
```

这一步非常关键。

你不用猜线上代码是什么，直接看 JVM 里加载的 class 反编译结果。

Arthas 官方快速入门也把 `jad` 作为确认线上类内容的重要命令。([Arthas](https://arthas.aliyun.com/doc/quick-start.html?utm_source=chatgpt.com "快速入门 - arthas"))

---

# 5. 场景四：CPU 飙高，怎么查？

典型现象：

```text
接口变慢
机器 CPU 90%+
日志没明显异常
```

---

## 5.1 查看 CPU 最高的线程

```bash
thread -n 5
```

输出类似：

```text
"C2 CompilerThread0" cpuUsage=25.31%
"http-nio-8080-exec-18" cpuUsage=18.42%
"http-nio-8080-exec-21" cpuUsage=16.88%
```

如果业务线程 CPU 高，例如：

```text
http-nio-8080-exec-18
```

继续查看线程栈：

```bash
thread 18
```

输出可能是：

```text
at com.zblog.search.ArticleSearchService.buildIndex(ArticleSearchService.java:88)
at com.zblog.article.ArticleController.rebuildIndex(ArticleController.java:42)
```

结论：

```text
某个请求触发了 buildIndex，导致 CPU 飙高。
```

---

## 5.2 判断常见原因

|线程栈表现|可能原因|
|---|---|
|大量 JSON 序列化|返回对象太大、循环引用、日志打印大对象|
|大量正则匹配|正则写法低效|
|大量集合遍历|N² 循环、数据量过大|
|大量加密/压缩|CPU 密集型任务放在接口线程|
|大量 GC 线程|内存压力大，频繁 GC|

---

# 6. 场景五：接口卡住、线程阻塞，怎么查？

## 6.1 查看线程状态

```bash
thread
```

重点看：

```text
BLOCKED
WAITING
TIMED_WAITING
```

如果大量线程卡在数据库连接池：

```text
at com.zaxxer.hikari.pool.HikariPool.getConnection(...)
```

说明可能是：

```text
数据库慢
连接池太小
连接泄漏
事务长时间不释放
```

如果大量线程卡在 Redis：

```text
at io.lettuce.core.protocol.CommandHandler...
```

说明可能是：

```text
Redis 慢
网络抖动
连接池耗尽
大 key 操作
```

---

## 6.2 查死锁

```bash
thread -b
```

如果有死锁，Arthas 会指出阻塞其他线程的线程。

常见原因：

```java
synchronized (lockA) {
    synchronized (lockB) {
        // ...
    }
}
```

另一个线程：

```java
synchronized (lockB) {
    synchronized (lockA) {
        // ...
    }
}
```

生产环境里一旦查到死锁，通常不是 Arthas 里修，而是：

```text
保留线程栈证据
评估是否重启止血
回滚或修复锁顺序
补充并发测试
```

---

# 7. 场景六：想看 Spring Bean 或调用对象方法

这个场景要谨慎，但很有用。

比如你想确认某个配置值：

```java
@Component
public class AiConfig {
    private String model;
    private Double temperature;
}
```

可以用 `ognl` 调用 Spring Context。

前提是你能拿到 Spring ApplicationContext。

如果项目里有静态工具类：

```java
public class SpringContextHolder {
    public static ApplicationContext getApplicationContext() {
        return applicationContext;
    }
}
```

那么可以执行：

```bash
ognl '@com.zblog.common.SpringContextHolder@getApplicationContext().getBean("aiConfig")'
```

如果想调用某个 Bean 的方法：

```bash
ognl '@com.zblog.common.SpringContextHolder@getApplicationContext().getBean("articleService").getArticleDetail(1001L)'
```

## 注意

`ognl` 本质上是在线上 JVM 里执行表达式。

所以不要乱执行：

```bash
delete()
update()
send()
close()
clear()
```

这类有副作用的方法。

建议只执行：

```text
get
query
find
list
read
```

这类只读方法。

---

# 8. 场景七：临时增强日志，不发版观察方法调用

很多时候你只是想临时看一下：

```text
这个方法有没有被调用？
入参是什么？
返回值是什么？
耗时多少？
异常是什么？
```

不用加日志，不用重新部署。

---

## 8.1 看方法是否被调用

```bash
monitor com.zblog.article.service.ArticleService getArticleDetail
```

输出类似：

```text
timestamp            class                                  method             total  success  fail  avg-rt(ms)
2026-05-19 11:20:01  ArticleService                         getArticleDetail   20     20       0     12.33
```

适合判断：

```text
接口到底有没有打到这个方法？
方法调用频率是多少？
失败次数是多少？
平均耗时是多少？
```

---

## 8.2 看具体入参和返回值

```bash
watch com.zblog.article.service.ArticleService getArticleDetail '{params, returnObj, #cost}' -x 3
```

---

## 8.3 看调用路径耗时

```bash
trace com.zblog.article.service.ArticleService getArticleDetail
```

---

# 9. 场景八：想看 JVM 基本状态

## 9.1 JVM 信息

```bash
jvm
```

你能看到：

```text
JVM 版本
启动参数
类加载数量
GC 信息
内存区域
系统属性
```

适合确认：

```text
线上到底是不是 Java 21
JVM 参数有没有生效
最大堆内存是多少
GC 类型是什么
```

---

## 9.2 内存信息

```bash
memory
```

重点看：

```text
heap
nonheap
eden
survivor
old
metaspace
direct
```

如果 old 区持续上涨，需要警惕：

```text
缓存没有淘汰
集合静态持有
线程池队列堆积
大对象长期引用
本地缓存过大
```

---

## 9.3 系统环境变量

```bash
sysenv
```

适合确认：

```text
环境变量是否正确
容器里的配置是否符合预期
```

---

## 9.4 JVM 系统属性

```bash
sysprop
```

适合确认：

```text
-Dspring.profiles.active=prod
-Dfile.encoding=UTF-8
-Duser.timezone=Asia/Shanghai
```

---

# 10. Web Console：适合演示，不建议裸露到公网

Arthas attach 成功后，默认可以访问：

```text
http://127.0.0.1:8563/
```

通过浏览器使用 Web Console。官方文档说明，Arthas 默认只监听 `127.0.0.1`，如果要远程访问可以指定 `--target-ip`，但这存在安全风险；更推荐结合 tunnel 方案处理远程诊断。([Arthas](https://arthas.aliyun.com/en/doc/web-console.html?utm_source=chatgpt.com "Web Console - arthas"))

生产建议：

```text
不要直接把 Arthas Web Console 暴露到公网
不要在无认证环境中开放远程端口
不要长期驻留
排障完成后及时退出
```

退出 Arthas：

```bash
quit
```

完全关闭 Arthas server：

```bash
stop
```

---

# 11. 一套实战排障流程

## 11.1 接口慢

```bash
dashboard
thread -n 5
trace com.xxx.Service methodName
watch com.xxx.Service methodName '{params, returnObj, #cost}' '#cost > 200' -x 3
```

判断顺序：

```text
先看 JVM 是否整体异常
再看 CPU 线程
再看方法链路耗时
最后看慢调用的具体参数
```

---

## 11.2 接口报错

```bash
watch com.xxx.Service methodName '{params, throwExp}' -e -x 3
```

如果异常来自下游调用：

```bash
trace com.xxx.Service methodName
```

判断顺序：

```text
先抓异常
再看入参
再看异常来自哪个内部调用
```

---

## 11.3 线上代码不对

```bash
sc -d com.xxx.Service
sm -d com.xxx.Service methodName
jad com.xxx.Service methodName
```

判断顺序：

```text
类有没有加载
从哪个 jar 加载
方法签名是否正确
反编译结果是否符合预期
```

---

## 11.4 CPU 高

```bash
dashboard
thread -n 10
thread <线程ID>
```

判断顺序：

```text
先找高 CPU 线程
再看线程栈
再定位具体业务代码
```

---

## 11.5 线程阻塞

```bash
thread
thread -b
```

判断顺序：

```text
看 BLOCKED / WAITING 数量
看是否死锁
看是否卡在数据库、Redis、HTTP、锁竞争
```

---

# 12. 常用命令速查表

|命令|用途|高频程度|
|---|---|---|
|`dashboard`|查看 JVM 实时状态|很高|
|`thread`|查看线程|很高|
|`thread -n 5`|查看 CPU 最高线程|很高|
|`thread -b`|查死锁|中高|
|`watch`|看入参、返回值、异常|很高|
|`trace`|看方法内部调用耗时|很高|
|`monitor`|统计方法调用次数、成功率、耗时|中高|
|`sc`|查类是否加载、从哪加载|很高|
|`sm`|查类的方法|中高|
|`jad`|反编译线上 class|很高|
|`jvm`|查看 JVM 信息|中|
|`memory`|查看内存区域|中|
|`sysenv`|查看环境变量|中|
|`sysprop`|查看 JVM 属性|中|
|`ognl`|执行表达式 / 调用 Bean|高风险，谨慎用|
|`stop`|关闭 Arthas server|必须知道|

---

# 13. 生产环境使用原则

## 13.1 不要一上来就 watch 高频方法

例如：

```bash
watch com.xxx.UserService getUser '{params, returnObj}' -x 5
```

如果 `getUser` 是超高频方法，可能刷屏，也会增加额外开销。

更安全：

```bash
watch com.xxx.UserService getUser '{params, returnObj, #cost}' '#cost > 100' -x 2 -n 10
```

含义：

```text
只观察耗时超过 100ms 的调用
对象展开深度 2
最多输出 10 次
```

---

## 13.2 线上优先只读观察

推荐：

```text
dashboard
thread
trace
watch
monitor
sc
sm
jad
jvm
memory
```

谨慎：

```text
ognl
redefine
retransform
vmtool
```

不要在生产随便做：

```text
热更新 class
调用有副作用的方法
清空缓存
修改业务状态
```

---

## 13.3 排障结束要退出

```bash
stop
```

不要让 Arthas 长期无管理驻留在线上。

---

# 14. 最小学习路径

不需要把 Arthas 所有命令都背下来。

Java 后端实际工作中，先掌握这 8 个就够了：

```bash
dashboard
thread
thread -n 5
watch
trace
sc
sm
jad
```

对应能力：

```text
看 JVM 状态
看线程问题
看方法入参返回值
看方法耗时链路
看线上类和方法
看线上实际代码
```

这就是 Arthas 的核心价值：**当生产环境不能 debug、不能随便加日志、不能重启服务时，用最小侵入方式把 JVM 现场看清楚。**