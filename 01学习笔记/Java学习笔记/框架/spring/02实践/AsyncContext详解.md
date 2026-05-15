`AsyncContext` 是 **Servlet** **3.0** 引入的异步请求上下文，核心价值只有一句话：

> **把“一个 HTTP 请求必须一直占用一个容器线程”这件事打破。**

在传统同步 Servlet 模型里，请求进来后，Tomcat/Jetty 会分配一个工作线程一直阻塞到业务完成、响应写回；而 `AsyncContext` 允许你在 **启动异步后立刻释放容器线程**，把后续业务交给别的线程处理，等结果出来再写回响应。

从后端工程师视角看，它的价值主要体现在三类场景：

1. **长轮询 / 配置推送 / 消息等待**
    
2. **慢下游调用隔离**
    
3. **大流量下减少线程占用，提升系统吞吐**
    

---

## 1）它到底解决什么问题

### 传统同步模型

请求进来：

- Tomcat 分配一个 worker thread
    
- 线程执行业务逻辑
    
    - 线程阻塞等待 DB / RPC / IO
        
- 结果返回后结束请求
    

问题是：**线程是稀缺资源**。 如果你有大量请求在等待，比如：

- 长轮询 30 秒
    
- 下游 RPC 延迟抖动
    
- 某些请求需要等待配置变化 / 事件触发
    

那么线程会被大量占住，导致：

- 线程池耗尽
    
- 排队时间变长
    
- RT 抖动
    
- CPU 没满但服务已经“看起来卡死”
    

### AsyncContext 的价值

启动异步后：

- 容器线程立即归还
    
- 业务逻辑由自定义线程池处理
    
- 结果准备好后再完成响应
    
- 对容器线程占用更短
    

本质上是把模型从：

- **线程绑定请求生命周期**
    

变成：

- **线程只负责接入和收尾，真正等待由异步机制承担**
    

---

## 2）AsyncContext 的核心生命周期

一个典型流程如下：

### ① 请求进入 Servlet

容器线程开始执行 `doGet/doPost`

### ② 调用 `request.startAsync()`

进入异步模式，拿到 `AsyncContext`

### ③ 容器线程返回

请求线程不再阻塞等待业务完成

### ④ 后台线程处理业务

你可以自己提交到业务线程池，或者调用 `asyncContext.start(...)`

### ⑤ 业务完成后：

- `asyncContext.complete()`：直接结束请求
    
- `asyncContext.dispatch()`：重新派发回容器，让它继续走 Servlet / Filter / Controller 流程
    

---

## 3）最重要的几个 API

### `request.startAsync()`

开启异步处理，返回 `AsyncContext`

### `asyncContext.start(Runnable)`

让容器用异步线程执行任务

### `asyncContext.complete()`

结束异步请求，响应真正收尾

### `asyncContext.dispatch()`

把请求重新派发回容器线程，继续走原有链路

### `asyncContext.setTimeout(ms)`

设置超时时间，避免请求永久挂住

### `asyncContext.addListener(...)`

监听生命周期事件：

- `onStartAsync`
    
- `onTimeout`
    
- `onError`
    
- `onComplete`
    

---

## 4）阿里式业务场景：配置中心长轮询

[Nacos 长轮询机制深度解析](https://my.feishu.cn/docx/UTtOdUCObolMxNxBA9zc9XSInpb)

这是 `AsyncContext` 最经典的落地方式之一。

### 场景

客户端请求配置：

- 如果配置已变化，立刻返回
    
- 如果配置没变，不立即返回
    
- 服务端把这个请求挂起，等配置更新后再通知客户端
    

这就是很多长轮询配置中心的核心思路。

### 为什么不用普通同步阻塞

如果大量客户端都在等配置变化，直接同步阻塞会占满 Web 容器线程。

### 为什么适合 AsyncContext

因为“等待”本质上不是 CPU 计算，而是事件驱动。

这类请求特别适合：

- 挂起
    
- 等待事件
    
- 事件到来后统一唤醒
    

---

# 5）生产级示例：基于 Servlet + AsyncContext 的长轮询

下面给一版可以落地的简化实现，重点体现：

- 线程池隔离
    
- 超时控制
    
- 异常处理
    
- 请求注册与通知
    
- 防止资源泄漏
    

---

## 5.1 线程池配置

```Java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.*;

@Configuration
public class AsyncExecutorConfig {

    @Bean(name = "bizExecutor", destroyMethod = "shutdown")
    public ExecutorService bizExecutor() {
        return new ThreadPoolExecutor(
                8,
                32,
                60L,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(2000),
                new ThreadFactory() {
                    private final ThreadFactory defaultFactory = Executors.defaultThreadFactory();
                    @Override
                    public Thread newThread(Runnable r) {
                        Thread t = defaultFactory.newThread(r);
                        t.setName("biz-async-" + t.getId());
                        t.setDaemon(false);
                        return t;
                    }
                },
                new ThreadPoolExecutor.CallerRunsPolicy()
        );
    }
}
```

### 设计点

- **有界队列**：避免请求暴增时无限堆积
    
- **CallerRunsPolicy**：在极端情况下让提交方降速
    
- **线程命名**：便于线上排查
    

---

## 5.2 长轮询注册中心

```TypeScript
import jakarta.servlet.AsyncContext;
import jakarta.servlet.AsyncEvent;
import jakarta.servlet.AsyncListener;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;

/**
 * 长轮询注册中心：
 * 作用：维护“某个 key 对应的一组等待中的 HTTP 请求（AsyncContext）”，
 * 当数据发生变化时，统一唤醒这些请求并返回结果。
 */
@Component
public class LongPollingRegistry {

    /**
     * 核心数据结构：
     * key -> 等待该 key 变化的一组 AsyncContext（HTTP 长连接请求）
     *
     * ConcurrentHashMap：
     *  - 支持高并发读写
     *
     * CopyOnWriteArrayList：
     *  - 读多写少场景（长轮询典型特征）
     *  - 遍历时无锁，避免并发问题
     */
    private final ConcurrentHashMap<String, CopyOnWriteArrayList<AsyncContext>> waiters = new ConcurrentHashMap<>();

    /**
     * 注册一个长轮询请求
     *
     * @param key          订阅的 key（例如配置 key）
     * @param asyncContext Servlet 异步上下文（代表一个未完成的 HTTP 请求）
     */
    public void register(String key, AsyncContext asyncContext) {
        // 将当前请求加入等待队列
        waiters.computeIfAbsent(key, k -> new CopyOnWriteArrayList<>()).add(asyncContext);

        // 给当前请求绑定监听器（监听生命周期事件）
        asyncContext.addListener(new AsyncListener() {

            /**
             * 请求正常完成（调用 complete() 后触发）
             */
            @Override
            public void onComplete(AsyncEvent event) {
                // 从等待队列移除，避免内存泄漏
                remove(key, asyncContext);
            }

            /**
             * 请求超时（例如 30s 没有等到变更）
             */
            @Override
            public void onTimeout(AsyncEvent event) {
                // 从等待队列移除
                remove(key, asyncContext);

                // 返回“没有变更”的响应（HTTP 204）
                tryWriteAndComplete(asyncContext, 204, "{\"changed\":false,\"reason\":\"timeout\"}");
            }

            /**
             * 请求发生异常（客户端断开等）
             */
            @Override
            public void onError(AsyncEvent event) {
                // 从等待队列移除
                remove(key, asyncContext);
            }

            /**
             * 重新开始异步（一般很少用，这里不处理）
             */
            @Override
            public void onStartAsync(AsyncEvent event) {
                // no-op
            }
        });
    }

    /**
     * 发布变更（例如配置更新）
     *
     * @param key     被更新的 key
     * @param payload 返回给客户端的数据
     */
    public void publish(String key, String payload) {
        // 取出该 key 下所有等待的请求，并一次性移除（避免重复通知）
        CopyOnWriteArrayList<AsyncContext> contexts = waiters.remove(key);

        if (contexts == null || contexts.isEmpty()) {
            return;
        }

        // 遍历所有等待的请求，逐个返回结果
        for (AsyncContext ctx : contexts) {
            tryWriteAndComplete(ctx, 200, payload);
        }
    }

    /**
     * 从等待队列中移除一个 AsyncContext
     */
    private void remove(String key, AsyncContext asyncContext) {
        CopyOnWriteArrayList<AsyncContext> list = waiters.get(key);
        if (list != null) {
            list.remove(asyncContext);

            // 如果列表为空，顺便清理 map，避免 key 泄漏
            if (list.isEmpty()) {
                waiters.remove(key, list);
            }
        }
    }

    /**
     * 尝试写响应并结束请求
     *
     * @param asyncContext 当前请求上下文
     * @param status       HTTP 状态码
     * @param body         响应体
     */
    private void tryWriteAndComplete(AsyncContext asyncContext, int status, String body) {
        try {
            // 获取 HTTP 响应对象
            HttpServletResponse resp = (HttpServletResponse) asyncContext.getResponse();

            // 如果响应还没提交（还没写出）
            if (!resp.isCommitted()) {
                resp.setStatus(status);
                resp.setCharacterEncoding(StandardCharsets.UTF_8.name());
                resp.setContentType("application/json;charset=UTF-8");

                // 写入响应体
                resp.getWriter().write(body);
                resp.getWriter().flush();
            }
        } catch (IOException ignored) {
            // 实际生产中不要吞异常，应该打日志 + 上报监控
        } finally {
            try {
                // 结束异步请求（触发 onComplete）
                asyncContext.complete();
            } catch (IllegalStateException ignored) {
                // 如果已经 complete 或请求失效，会抛异常，这里忽略
            }
        }
    }
}
```

---

## 5.3 Servlet 入口

```Java
import jakarta.annotation.Resource;
import jakarta.servlet.AsyncContext;
import jakarta.servlet.ServletException;
import jakarta.servlet.annotation.WebServlet;
import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.util.Map;
import java.util.concurrent.ExecutorService;

@WebServlet(urlPatterns = "/config/poll", asyncSupported = true)
public class ConfigPollServlet extends HttpServlet {

    @Resource
    private LongPollingRegistry longPollingRegistry;

    @Resource(name = "bizExecutor")
    private ExecutorService bizExecutor;

    @Resource
    private ConfigService configService;

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        String dataId = req.getParameter("dataId");
        String clientVersion = req.getParameter("version");

        if (dataId == null || dataId.isBlank()) {
            resp.setStatus(400);
            resp.setContentType("application/json;charset=UTF-8");
            resp.getWriter().write("{\"error\":\"dataId required\"}");
            return;
        }

        // 1. 先做一次快速判断，能返回就直接返回
        String serverVersion = configService.getVersion(dataId);
        if (!serverVersion.equals(clientVersion)) {
            resp.setStatus(200);
            resp.setCharacterEncoding(StandardCharsets.UTF_8.name());
            resp.setContentType("application/json;charset=UTF-8");
            resp.getWriter().write(configService.getConfigJson(dataId));
            return;
        }

        // 2. 启动异步，释放容器线程
        AsyncContext asyncContext = req.startAsync();
        asyncContext.setTimeout(30000L);

        // 3. 注意：这里可以先设置基础响应头
        resp.setCharacterEncoding(StandardCharsets.UTF_8.name());
        resp.setContentType("application/json;charset=UTF-8");

        // 4. 注册等待
        longPollingRegistry.register(dataId, asyncContext);

        // 5. 可选：异步线程做一些附加逻辑
        bizExecutor.submit(() -> {
            try {
                // 模拟一些业务处理，实际项目中可能是查缓存、查元数据、记录埋点等
                Map<String, String> meta = configService.getMeta(dataId);
            } catch (Exception e) {
                try {
                    resp.setStatus(500);
                    resp.getWriter().write("{\"error\":\"internal error\"}");
                } catch (IOException ignored) {
                } finally {
                    asyncContext.complete();
                }
            }
        });
    }
}
```

---

## 5.4 配置更新时通知长轮询请求

```TypeScript
import org.springframework.stereotype.Service;

import java.util.concurrent.ConcurrentHashMap;

/**
 * 配置服务：
 * 作用：
 * 1. 存储配置内容（configMap）
 * 2. 存储配置版本号（versionMap）
 * 3. 提供查询能力
 * 4. 提供更新能力，并在更新时通知长轮询客户端（通过 LongPollingRegistry）
 */
@Service
public class ConfigService {

    /**
     * key: dataId（配置标识）
     * value: version（版本号）
     *
     * 用于客户端判断配置是否发生变化（长轮询核心字段）
     */
    private final ConcurrentHashMap<String, String> versionMap = new ConcurrentHashMap<>();

    /**
     * key: dataId（配置标识）
     * value: 配置内容（JSON 字符串）
     *
     * 实际配置存储
     */
    private final ConcurrentHashMap<String, String> configMap = new ConcurrentHashMap<>();

    /**
     * 获取配置版本号
     *
     * @param dataId 配置标识
     * @return 当前版本号（默认 "0" 表示初始版本）
     */
    public String getVersion(String dataId) {
        return versionMap.getOrDefault(dataId, "0");
    }

    /**
     * 获取配置内容（JSON）
     *
     * @param dataId 配置标识
     * @return 配置 JSON（如果不存在返回默认空配置）
     */
    public String getConfigJson(String dataId) {
        return configMap.getOrDefault(
                dataId,
                // 默认返回一个最简 JSON，避免客户端解析异常
                "{\"dataId\":\"" + dataId + "\",\"value\":\"\"}"
        );
    }

    /**
     * 获取元信息（用于扩展）
     *
     * @param dataId 配置标识
     * @return 简单元数据（这里只返回 dataId，可扩展为 group、tenant 等）
     */
    public java.util.Map<String, String> getMeta(String dataId) {
        return java.util.Map.of("dataId", dataId);
    }

    /**
     * 更新配置
     *
     * 核心流程：
     * 1. 更新内存中的配置内容
     * 2. 更新版本号
     * 3. 通知所有长轮询客户端（触发返回）
     *
     * @param dataId         配置标识
     * @param newConfigJson  新配置内容
     * @param newVersion     新版本号（通常是时间戳或递增版本）
     * @param registry       长轮询注册中心（用于通知客户端）
     */
    public void updateConfig(String dataId, String newConfigJson, String newVersion,
                             LongPollingRegistry registry) {

        // 更新配置内容（写内存）
        configMap.put(dataId, newConfigJson);

        // 更新版本号
        versionMap.put(dataId, newVersion);

        // 通知所有正在监听该 dataId 的客户端：
        // LongPollingRegistry 会唤醒所有挂起的请求并返回新配置
        registry.publish(dataId, newConfigJson);
    }
}
```

---

# 6）AsyncContext 的关键规则

这是面试和生产都很容易踩坑的部分。

---

## 6.1 必须开启 `asyncSupported=true`

如果 Servlet / Filter 不支持异步，调用 `startAsync()` 会报错。

### 注意

- Servlet 要支持
    
- Filter 也要支持
    
- 链路上任何一环不支持，都可能出问题
    

---

## 6.2 `complete()` 必须保证调用

这是最常见的泄漏点。

如果你：

- 超时没处理
    
- 异常没处理
    
- 业务成功了却忘了 complete
    

那么请求会一直挂着，造成连接泄漏、内存泄漏、请求堆积。

---

## 6.3 不要把 `AsyncContext` 当成无限期保存对象

它不是缓存，不是消息队列，不是数据库连接。

要控制：

- 超时
    
- 最大挂起数
    

实际意义：例如避免过多的Future对象撑爆服务端的heap（对应客户端信号量限流来解决）

- 每个 key 的等待队列大小
    
- 失败快速释放
    

---

## 6.4 写响应前检查状态

避免重复写入：

- `response.isCommitted()`
    
- `asyncContext.complete()` 是否已经执行
    

否则可能出现：

- `IllegalStateException`
    
- 重复提交响应
    
- 输出内容混乱
    

---

## 6.5 注意线程上下文传递

异步线程里常见上下文丢失：

- `ThreadLocal`
    
- `MDC`
    
- 登录态上下文
    
- 链路追踪上下文
    

生产上通常需要手工传递，或者用统一的上下文包装器。

例如：配合ContextDecorator手动快照上下文

---

# 7）AsyncContext、@Async、CompletableFuture、DeferredResult 的区别

很多人容易混。

---

## 7.1 AsyncContext

这是 **Servlet 容器层** 的异步能力，最底层。

适合：

- Servlet 原生开发
    
- 长轮询
    
- 事件通知
    
- 需要精确控制请求生命周期
    

---

## 7.2 `@Async`

这是 **Spring 方法异步执行**，本质是方法提交到线程池。

它不直接管理 HTTP 请求生命周期。

你用它做业务异步没问题，但它不是 Servlet 请求异步模型本身。

---

## 7.3 `CompletableFuture`

这是 **Java 并发编排工具**，便于组合多个异步任务。

它能和 `AsyncContext` 配合使用，但不是同一个层次的东西。

---

## 7.4 `DeferredResult`

这是 Spring MVC 对异步请求的高层封装。

底层依然依赖 Servlet Async 机制。

### 结论

如果你在 Spring MVC 里做接口异步，通常优先考虑：

- `DeferredResult`
    
- `WebAsyncTask`
    
- `SseEmitter`
    

如果你要更底层、更可控，才直接上 `AsyncContext`。

---

# 8）面试里最值得说的点

你回答 `AsyncContext` 时，最好不要停留在“异步处理请求”这种空话上。 高质量回答通常要包含这几个层次：

### ① 它解决的是线程占用问题

不是简单“提高并发”四个字就结束了。

### ② 它不是让业务更快，而是让线程更省

请求仍然可能等很久，但线程不再被占死。

### ③ 它适合事件驱动场景

尤其是长轮询、配置推送、消息等待。

### ④ 它有资源管理成本

必须考虑超时、取消、异常、complete、上下文传递。

### ⑤ 它和 Spring MVC 异步体系有关联

`DeferredResult/WebAsyncTask/SseEmitter` 本质上都和底层异步机制有关。

---

# 9）项目里真正要注意的工程化问题

这是企业生产中比 API 本身更重要的部分。

## 9.1 限流与背压

不能无限挂起请求。

要对每个 key、每个用户、每个实例设置上限。

## 9.2 监控指标

至少要有：

- 当前挂起请求数
    
- 超时数
    
- 异常数
    
- 平均挂起时长
    
- complete 失败数
    

## 9.3 取消语义

客户端断开后，服务端要及时清理等待对象。

## 9.4 容灾

实例重启时，挂起请求全丢失，客户端需要重试。

## 9.5 幂等

通知和 complete 可能重复触发，代码必须容忍重复调用。

---

# 10）一句话总结它的本质

`AsyncContext` 的本质不是“多线程”，而是 **“把 HTTP 请求的生命周期和容器线程****解耦****”**。 它让你可以在 **不占用请求线程** 的前提下，把“等待结果”这件事交给异步机制处理。

---

## 总结概括

`AsyncContext` 是 Servlet 异步模型的底层能力，核心作用是释放容器线程、降低阻塞等待带来的资源浪费，特别适合长轮询、配置推送、慢下游隔离这类场景。生产上使用时，重点不在“会不会用 API”，而在 **超时、complete、异常清理、****线程池****隔离、上下文传递、监控治理** 这些工程细节上。

## Key words

`Servlet 3.0`、`AsyncContext`、`startAsync`、`complete`、`dispatch`、`AsyncListener`、`长轮询`、`线程解耦`、`线程池隔离`、`超时控制`、`资源清理`、`配置中心`

## 可扩展的知识点（面试加分项）

1. `AsyncContext` 和 `DeferredResult/WebAsyncTask/SseEmitter` 的底层关系
    
2. 长轮询与 SSE 的区别，以及各自适合的业务场景
    
3. 如何设计一个高可用配置中心的长轮询通知链路
    
4. 线程上下文传递：MDC、TraceId、SecurityContext 如何在异步线程中保留
    
5. 异步请求的超时、取消、重试、幂等设计
    
6. Web 容器线程模型：Tomcat worker、NIO、连接复用
    
7. `dispatch()` 和 `complete()` 的语义区别与适用场景
    
8. 如何做异步请求的监控指标与告警
    
9. 如何避免异步场景下的内存泄漏和对象滞留
    
10. Reactor/Servlet Async/Netty 的模型差异与演进路径
    

下一条我可以继续把它展开成 **“AsyncContext** **源码****调用链 +** **Tomcat** **底层线程模型 + 面试标准回答模板”**。