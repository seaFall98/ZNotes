```
//  
// Source code recreated from a .class file by IntelliJ IDEA  
// (powered by FernFlower decompiler)  
//  
  
package org.springframework.amqp.rabbit.annotation;  
  
import java.lang.annotation.Documented;  
import java.lang.annotation.ElementType;  
import java.lang.annotation.Repeatable;  
import java.lang.annotation.Retention;  
import java.lang.annotation.RetentionPolicy;  
import java.lang.annotation.Target;  
import org.springframework.messaging.handler.annotation.MessageMapping;  
  
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.ANNOTATION_TYPE})  
@Retention(RetentionPolicy.RUNTIME)  
@MessageMapping  
@Documented  
@Repeatable(RabbitListeners.class)  
public @interface RabbitListener {  
    String id() default "";  
  
    String containerFactory() default "";  
  
    String[] queues() default {};  
  
    Queue[] queuesToDeclare() default {};  
  
    boolean exclusive() default false;  
  
    String priority() default "";  
  
    String admin() default "";  
  
    QueueBinding[] bindings() default {};  
  
    String group() default "";  
  
    String returnExceptions() default "";  
  
    String errorHandler() default "";  
  
    String concurrency() default "";  
  
    String autoStartup() default "";  
  
    String executor() default "";  
  
    String ackMode() default "";  
  
    String replyPostProcessor() default "";  
  
    String messageConverter() default "";  
  
    String replyContentType() default "";  
  
    String converterWinsContentType() default "true";  
}
```

# 结论先说：`@RabbitListener`不是“普通注解”，它本质是在声明一个**消息消费者容器**

你看到 `@RabbitListener` 属性很多，是因为它不只是标记“这个方法消费 RabbitMQ 消息”，而是在配置一套消费端运行时能力：

```text
监听哪个队列？
队列/交换机/绑定是否自动声明？
用哪个 ListenerContainer？
消费并发是多少？
怎么 ack？
怎么反序列化？
异常怎么处理？
是否自动启动？
消费方 id 是什么？
```

所以学习 `@RabbitListener`，不要按属性表死背。按实际开发可以分成 5 类：

| 类别    | 常用属性                                                      | 解决什么问题                    |
| ----- | --------------------------------------------------------- | ------------------------- |
| 监听目标  | `queues` / `queuesToDeclare` / `bindings`                 | 监听哪个队列，是否顺便声明队列、交换机、绑定    |
| 容器控制  | `containerFactory` / `concurrency` / `autoStartup` / `id` | 消费者容器、并发、启动控制             |
| 消息转换  | `messageConverter`                                        | JSON / 对象反序列化             |
| 异常与返回 | `errorHandler` / `returnExceptions` / `replyTo`           | 异常处理、RPC 模式响应             |
| 进阶元信息 | `group` / `admin` / `executor` / `ackMode`                | 分组管理、RabbitAdmin、线程池、确认模式 |

Spring AMQP 官方文档说明：`@RabbitListener` 用于把方法标记为 RabbitMQ 消息监听目标，可以通过 `queues` 或 `bindings` 指定监听队列；如果不指定 `containerFactory`，默认会寻找名为 `rabbitListenerContainerFactory` 的容器工厂。([Home](https://docs.spring.io/spring-amqp/api/org/springframework/amqp/rabbit/annotation/RabbitListener.html?utm_source=chatgpt.com "RabbitListener (Spring AMQP 4.0.2 API)"))

---

# 1. 最基础用法：监听一个已有队列

## 代码

```java
@Component
public class OrderMessageConsumer {

    @RabbitListener(queues = "order.created.queue")
    public void onOrderCreated(String message) {
        System.out.println("收到订单消息：" + message);
    }
}
```

## 这个注解做了什么？

```text
启动 Spring 容器
  ↓
扫描 @RabbitListener
  ↓
为这个方法创建 MessageListenerContainer
  ↓
容器连接 RabbitMQ
  ↓
订阅 order.created.queue
  ↓
收到消息后调用 onOrderCreated()
```

这里的核心属性是：

```java
queues = "order.created.queue"
```

表示：**监听 RabbitMQ 中已经存在的队列**。

注意：`queues` 通常要求队列已经提前创建好。队列可以由运维、RabbitMQ 控制台、Terraform、Helm Chart、Spring Bean 或其他配置创建。

---

# 2. 实际项目更常见：消费 JSON 对象

## 发送方消息

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderCreatedMessage {
    private Long orderId;
    private Long userId;
    private BigDecimal amount;
}
```

发送：

```java
@Service
@RequiredArgsConstructor
public class OrderProducer {

    private final RabbitTemplate rabbitTemplate;

    public void sendOrderCreated(OrderCreatedMessage message) {
        rabbitTemplate.convertAndSend(
                "order.exchange",
                "order.created",
                message
        );
    }
}
```

## 消费方

```java
@Component
@Slf4j
public class OrderConsumer {

    @RabbitListener(queues = "order.created.queue")
    public void handle(OrderCreatedMessage message) {
        log.info("收到订单创建消息：orderId={}, userId={}, amount={}",
                message.getOrderId(),
                message.getUserId(),
                message.getAmount());

        // 执行业务逻辑
        // 例如：初始化订单状态、发放积分、写入业务流水等
    }
}
```

## 需要配置 JSON 转换器

```java
@Configuration
public class RabbitConfig {

    @Bean
    public MessageConverter jacksonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public RabbitTemplate rabbitTemplate(
            ConnectionFactory connectionFactory,
            MessageConverter jacksonMessageConverter
    ) {
        RabbitTemplate rabbitTemplate = new RabbitTemplate(connectionFactory);
        rabbitTemplate.setMessageConverter(jacksonMessageConverter);
        return rabbitTemplate;
    }

    @Bean
    public SimpleRabbitListenerContainerFactory rabbitListenerContainerFactory(
            ConnectionFactory connectionFactory,
            MessageConverter jacksonMessageConverter
    ) {
        SimpleRabbitListenerContainerFactory factory =
                new SimpleRabbitListenerContainerFactory();

        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(jacksonMessageConverter);

        return factory;
    }
}
```

## 关键点

`@RabbitListener` 方法参数可以直接写业务对象：

```java
public void handle(OrderCreatedMessage message)
```

Spring AMQP 会通过 `MessageConverter` 把 RabbitMQ 原始消息转换成 Java 对象。Spring AMQP 本身提供消息驱动 POJO 支持，也就是可以让普通 Java 方法作为消息监听方法。([Home](https://docs.spring.io/spring-amqp/reference/index.html?utm_source=chatgpt.com "Spring AMQP"))

---

# 3. `queues`：监听已经存在的队列

这是最稳、最常用的写法。

```java
@RabbitListener(queues = "payment.success.queue")
public void handlePaymentSuccess(PaymentSuccessMessage message) {
    // 处理支付成功消息
}
```

适用场景：

```text
生产环境队列由基础设施统一创建
队列、交换机、绑定关系不希望散落在业务代码里
服务启动时不希望随便声明 RabbitMQ 资源
```

生产环境里，很多团队更喜欢这种方式：

```java
@Configuration
public class RabbitQueueConfig {

    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_CREATED_QUEUE = "order.created.queue";
    public static final String ORDER_CREATED_ROUTING_KEY = "order.created";

    @Bean
    public DirectExchange orderExchange() {
        return new DirectExchange(ORDER_EXCHANGE, true, false);
    }

    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder.durable(ORDER_CREATED_QUEUE)
                .build();
    }

    @Bean
    public Binding orderCreatedBinding() {
        return BindingBuilder
                .bind(orderCreatedQueue())
                .to(orderExchange())
                .with(ORDER_CREATED_ROUTING_KEY);
    }
}
```

然后消费者只写：

```java
@RabbitListener(queues = RabbitQueueConfig.ORDER_CREATED_QUEUE)
public void handle(OrderCreatedMessage message) {
    // 消费消息
}
```

这样结构更清晰：

```text
Rabbit 资源声明：RabbitQueueConfig
消费业务逻辑：OrderConsumer
```

---

# 4. `queuesToDeclare`：监听时顺便声明队列

## 示例

```java
@RabbitListener(
        queuesToDeclare = @Queue(
                name = "demo.queue",
                durable = "true"
        )
)
public void handle(String message) {
    System.out.println(message);
}
```

这表示：

```text
如果 demo.queue 不存在，Spring 帮你声明
然后监听 demo.queue
```

## 适用场景

适合：

```text
Demo
测试环境
小型内部系统
快速验证功能
```

不太适合大型生产系统，因为队列声明散落在监听方法上，后期不好统一治理。

---

# 5. `bindings`：在注解里声明队列、交换机、路由关系

这是 `@RabbitListener` 最“重”的写法。

## Direct Exchange 示例

```java
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(
                value = "order.created.queue",
                durable = "true"
        ),
        exchange = @Exchange(
                value = "order.exchange",
                type = ExchangeTypes.DIRECT,
                durable = "true"
        ),
        key = "order.created"
))
public void handle(OrderCreatedMessage message) {
    // 处理订单创建消息
}
```

这段代码同时表达了三件事：

```text
声明队列：order.created.queue
声明交换机：order.exchange
绑定关系：order.exchange + order.created -> order.created.queue
监听队列：order.created.queue
```

## Topic Exchange 示例

```java
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(
                value = "order.event.queue",
                durable = "true"
        ),
        exchange = @Exchange(
                value = "order.topic.exchange",
                type = ExchangeTypes.TOPIC,
                durable = "true"
        ),
        key = "order.*"
))
public void handleOrderEvent(OrderEventMessage message) {
    // 可以消费 order.created、order.paid、order.cancelled 等消息
}
```

## Fanout Exchange 示例

```java
@RabbitListener(bindings = @QueueBinding(
        value = @Queue(
                value = "order.audit.queue",
                durable = "true"
        ),
        exchange = @Exchange(
                value = "order.fanout.exchange",
                type = ExchangeTypes.FANOUT,
                durable = "true"
        )
))
public void audit(OrderCreatedMessage message) {
    // 订单审计消费
}
```

`bindings` 适合把 demo 跑起来，但生产项目里我更建议用 `@Bean` 单独声明 Exchange、Queue、Binding。

---

# 6. `containerFactory`：最重要的生产级属性之一

`@RabbitListener` 本身只是声明监听方法，真正负责消费的是 Listener Container。

常见容器工厂有：

```text
SimpleRabbitListenerContainerFactory
DirectRabbitListenerContainerFactory
```

多数 Spring Boot 项目默认用 `SimpleRabbitListenerContainerFactory`。

官方 API 文档说明，`containerFactory` 用于指定创建 Rabbit Listener Container 的工厂；如果没有设置，会默认查找 `rabbitListenerContainerFactory`。([Home](https://docs.spring.io/spring-amqp/api/org/springframework/amqp/rabbit/annotation/RabbitListener.html?utm_source=chatgpt.com "RabbitListener (Spring AMQP 4.0.2 API)"))

---

## 生产级配置示例：手动 ack + 并发 + 预取

```java
@Configuration
public class RabbitListenerConfig {

    @Bean
    public SimpleRabbitListenerContainerFactory orderListenerContainerFactory(
            ConnectionFactory connectionFactory,
            MessageConverter jacksonMessageConverter
    ) {
        SimpleRabbitListenerContainerFactory factory =
                new SimpleRabbitListenerContainerFactory();

        factory.setConnectionFactory(connectionFactory);

        // JSON 反序列化
        factory.setMessageConverter(jacksonMessageConverter);

        // 手动 ack
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);

        // 初始消费者数量
        factory.setConcurrentConsumers(3);

        // 最大消费者数量
        factory.setMaxConcurrentConsumers(10);

        // 每个消费者一次最多拿多少条未 ack 消息
        factory.setPrefetchCount(20);

        // 消费异常时是否默认重新入队
        factory.setDefaultRequeueRejected(false);

        return factory;
    }
}
```

使用：

```java
@Component
@Slf4j
public class OrderConsumer {

    @RabbitListener(
            queues = "order.created.queue",
            containerFactory = "orderListenerContainerFactory"
    )
    public void handle(
            OrderCreatedMessage message,
            Channel channel,
            Message rawMessage
    ) throws IOException {

        long deliveryTag = rawMessage.getMessageProperties().getDeliveryTag();

        try {
            log.info("处理订单创建消息：{}", message);

            // 1. 幂等校验
            // 2. 执行业务逻辑
            // 3. 落库 / 调用服务 / 更新状态

            channel.basicAck(deliveryTag, false);
        } catch (Exception e) {
            log.error("订单消息消费失败：{}", message, e);

            // false: 只 nack 当前消息
            // false: 不重新入队，通常配合死信队列
            channel.basicNack(deliveryTag, false, false);
        }
    }
}
```

这就是生产中非常常见的消费模型：

```text
收到消息
  ↓
反序列化
  ↓
执行业务
  ↓
成功：basicAck
失败：basicNack / basicReject
  ↓
失败消息进入 DLQ 或被丢弃，取决于队列配置
```

---

# 7. `concurrency`：直接在注解上配置并发

你可以不单独配置 factory，直接写：

```java
@RabbitListener(
        queues = "order.created.queue",
        concurrency = "3-10"
)
public void handle(OrderCreatedMessage message) {
    // 消费逻辑
}
```

含义：

```text
最少 3 个消费者线程
最多 10 个消费者线程
```

也可以写成配置项：

```java
@RabbitListener(
        queues = "order.created.queue",
        concurrency = "${rabbit.listener.order.concurrency}"
)
public void handle(OrderCreatedMessage message) {
}
```

```yaml
rabbit:
  listener:
    order:
      concurrency: 3-10
```

Spring AMQP 从 2.0 开始支持在 `@RabbitListener` 上配置 `concurrency`，并且支持 SpEL 和属性占位符；不同容器类型下该值含义略有差异。([docs4dev.com](https://www.docs4dev.com/docs/en/spring-amqp/2.1.2.RELEASE/reference/_reference.html?utm_source=chatgpt.com "Spring AMQP Reference"))

## 并发不是越高越好

你要结合业务看：

|场景|并发建议|
|---|---|
|CPU 密集型处理|不要太高，接近 CPU 核数|
|IO 密集型处理|可以适当提高|
|下游数据库压力大|并发要保守|
|消息要求顺序消费|不要随便提高并发|
|单条消息处理很慢|可以提高并发，但要配合限流|

RabbitMQ 的并发消费本质不是“一个队列变成多个队列”，而是多个消费者共同消费同一个队列：

```text
order.created.queue
      ↓
 ┌───────────┬───────────┬───────────┐
 consumer-1  consumer-2  consumer-3
```

---

# 8. `ackMode`：确认模式，决定消息什么时候算消费成功

有些版本的 `@RabbitListener` 支持在注解上指定 `ackMode`，也可以在容器工厂中配置。

常见模式：

|模式|含义|适用场景|
|---|---|---|
|`AUTO`|方法正常执行完自动 ack，抛异常则按容器策略处理|普通业务|
|`MANUAL`|业务代码手动 `basicAck/basicNack`|可靠性要求高|
|`NONE`|不确认，消息投递后即认为成功|低可靠场景，不推荐核心业务|

## AUTO 模式

```java
@RabbitListener(queues = "email.send.queue")
public void sendEmail(EmailMessage message) {
    emailService.send(message);

    // 方法正常结束后，Spring 自动 ack
}
```

问题是：

```text
如果 emailService.send() 抛异常，消息怎么处理？
```

这就取决于容器配置：

```java
factory.setDefaultRequeueRejected(false);
```

如果设置为 `false`，失败消息通常不重新入队，可进入死信队列。

---

## MANUAL 模式

```java
@RabbitListener(
        queues = "payment.success.queue",
        containerFactory = "manualAckContainerFactory"
)
public void handlePaymentSuccess(
        PaymentSuccessMessage message,
        Channel channel,
        Message rawMessage
) throws IOException {

    long deliveryTag = rawMessage.getMessageProperties().getDeliveryTag();

    try {
        paymentService.handleSuccess(message);

        channel.basicAck(deliveryTag, false);
    } catch (BizException e) {
        // 业务异常：不重试，进死信
        channel.basicNack(deliveryTag, false, false);
    } catch (Exception e) {
        // 系统异常：可以选择重新入队，但要小心死循环
        channel.basicNack(deliveryTag, false, true);
    }
}
```

## 不建议盲目 `requeue=true`

这个很容易造成毒丸消息反复消费：

```text
消息有问题
  ↓
消费失败
  ↓
重新入队
  ↓
再次消费
  ↓
再次失败
  ↓
无限循环
```

更稳的做法：

```text
失败 → 不重新入队 → 进入死信队列 → 后台补偿/人工处理/定时重试
```

---

# 9. `messageConverter`：给某个 Listener 单独指定转换器

全局配置：

```java
@Bean
public MessageConverter jacksonMessageConverter() {
    return new Jackson2JsonMessageConverter();
}
```

某个监听器指定：

```java
@RabbitListener(
        queues = "order.created.queue",
        messageConverter = "jacksonMessageConverter"
)
public void handle(OrderCreatedMessage message) {
}
```

适用场景：

```text
大部分队列用 JSON
少数队列用 String
少数队列用二进制协议
某些队列使用自定义序列化格式
```

不过在实际项目中，更常见的是在 `containerFactory` 上统一配置 converter，而不是每个 `@RabbitListener` 单独写。

---

# 10. `id`：给监听容器起名字，方便管理

```java
@RabbitListener(
        id = "orderCreatedListener",
        queues = "order.created.queue"
)
public void handle(OrderCreatedMessage message) {
}
```

这个 `id` 不是 RabbitMQ 队列名，而是 Spring 内部 Listener Container 的标识。

有什么用？

可以通过 `RabbitListenerEndpointRegistry` 动态控制监听器。

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/rabbit/listener")
public class RabbitListenerManageController {

    private final RabbitListenerEndpointRegistry registry;

    @PostMapping("/{id}/stop")
    public void stop(@PathVariable String id) {
        MessageListenerContainer container =
                registry.getListenerContainer(id);

        if (container != null) {
            container.stop();
        }
    }

    @PostMapping("/{id}/start")
    public void start(@PathVariable String id) {
        MessageListenerContainer container =
                registry.getListenerContainer(id);

        if (container != null) {
            container.start();
        }
    }
}
```

使用场景：

```text
临时暂停某类消息消费
发布期间先停止消费者
下游系统故障时暂停消费
后台管理系统控制消息消费开关
```

---

# 11. `autoStartup`：服务启动时是否自动开始消费

```java
@RabbitListener(
        id = "orderCreatedListener",
        queues = "order.created.queue",
        autoStartup = "false"
)
public void handle(OrderCreatedMessage message) {
}
```

含义：

```text
Spring Boot 启动时创建监听容器
但不自动开始消费
```

然后你可以在合适时机手动启动：

```java
@Component
@RequiredArgsConstructor
public class RabbitListenerStarter {

    private final RabbitListenerEndpointRegistry registry;

    public void startOrderListener() {
        MessageListenerContainer container =
                registry.getListenerContainer("orderCreatedListener");

        if (container != null) {
            container.start();
        }
    }
}
```

适合场景：

```text
系统初始化完成后再消费
缓存预热完成后再消费
依赖的下游服务可用后再消费
灰度发布时控制消费开关
```

---

# 12. `errorHandler`：监听方法异常处理

```java
@Component
public class OrderRabbitListenerErrorHandler implements RabbitListenerErrorHandler {

    @Override
    public Object handleError(
            Message amqpMessage,
            org.springframework.messaging.Message<?> message,
            ListenerExecutionFailedException exception
    ) throws Exception {

        // 这里可以记录日志、报警、埋点
        log.error("RabbitListener 消费失败，message={}", message, exception);

        // 继续抛出，让容器决定 nack / reject / retry
        throw exception;
    }
}
```

使用：

```java
@RabbitListener(
        queues = "order.created.queue",
        errorHandler = "orderRabbitListenerErrorHandler"
)
public void handle(OrderCreatedMessage message) {
    orderService.handle(message);
}
```

注意：

```text
errorHandler 不是万能重试器
它更适合做监听方法级别的异常兜底、日志增强、异常包装
真正的重试、死信、ack 策略通常还是放在容器工厂和 RabbitMQ 队列配置里
```

---

# 13. `replyTo`：RabbitMQ RPC 模式，不是普通异步消息的主流用法

`@RabbitListener` 方法可以有返回值：

```java
@RabbitListener(queues = "user.query.queue")
public UserInfoResponse queryUser(UserInfoRequest request) {
    return userService.query(request.getUserId());
}
```

发送方使用 `convertSendAndReceive`：

```java
UserInfoResponse response = (UserInfoResponse) rabbitTemplate.convertSendAndReceive(
        "user.exchange",
        "user.query",
        new UserInfoRequest(1001L)
);
```

这是一种 RPC 风格：

```text
发送请求消息
  ↓
消费者处理
  ↓
返回响应消息
  ↓
发送方阻塞等待结果
```

## 我的建议

普通业务不要滥用 RabbitMQ RPC。

更常见的选择：

```text
同步查询：HTTP / Dubbo / gRPC
异步解耦：RabbitMQ
```

RabbitMQ RPC 适合少量内部场景，但如果大量使用，系统会变得像“伪同步调用”，排查链路反而更复杂。

---

# 14. `@RabbitListener` 方法参数可以写哪些？

常见写法：

## 只拿业务对象

```java
@RabbitListener(queues = "order.created.queue")
public void handle(OrderCreatedMessage message) {
}
```

## 同时拿原始消息

```java
@RabbitListener(queues = "order.created.queue")
public void handle(OrderCreatedMessage message, Message rawMessage) {
    String messageId = rawMessage.getMessageProperties().getMessageId();
    String receivedRoutingKey = rawMessage.getMessageProperties().getReceivedRoutingKey();
}
```

## 手动 ack 时拿 Channel

```java
@RabbitListener(queues = "order.created.queue")
public void handle(
        OrderCreatedMessage message,
        Channel channel,
        Message rawMessage
) {
}
```

## 获取 Header

```java
@RabbitListener(queues = "order.created.queue")
public void handle(
        OrderCreatedMessage message,
        @Header("traceId") String traceId
) {
    log.info("traceId={}, message={}", traceId, message);
}
```

## 获取所有 Headers

```java
@RabbitListener(queues = "order.created.queue")
public void handle(
        OrderCreatedMessage message,
        @Headers Map<String, Object> headers
) {
    log.info("headers={}", headers);
}
```

实际项目里推荐：

```java
public void handle(
        OrderCreatedMessage message,
        Message rawMessage,
        Channel channel
)
```

这样你既能处理业务对象，也能拿到 deliveryTag、messageId、routingKey、headers 等元信息。

---

# 15. 一个更完整的生产级消费示例

## 队列配置：普通队列 + 死信队列

```java
@Configuration
public class OrderRabbitConfig {

    public static final String ORDER_EXCHANGE = "order.exchange";
    public static final String ORDER_CREATED_QUEUE = "order.created.queue";
    public static final String ORDER_CREATED_ROUTING_KEY = "order.created";

    public static final String ORDER_DLX = "order.dlx.exchange";
    public static final String ORDER_CREATED_DLQ = "order.created.dlq";
    public static final String ORDER_CREATED_DLK = "order.created.dead";

    @Bean
    public DirectExchange orderExchange() {
        return ExchangeBuilder
                .directExchange(ORDER_EXCHANGE)
                .durable(true)
                .build();
    }

    @Bean
    public DirectExchange orderDlxExchange() {
        return ExchangeBuilder
                .directExchange(ORDER_DLX)
                .durable(true)
                .build();
    }

    @Bean
    public Queue orderCreatedQueue() {
        return QueueBuilder
                .durable(ORDER_CREATED_QUEUE)
                // 消费失败后进入死信交换机
                .deadLetterExchange(ORDER_DLX)
                .deadLetterRoutingKey(ORDER_CREATED_DLK)
                .build();
    }

    @Bean
    public Queue orderCreatedDlq() {
        return QueueBuilder
                .durable(ORDER_CREATED_DLQ)
                .build();
    }

    @Bean
    public Binding orderCreatedBinding() {
        return BindingBuilder
                .bind(orderCreatedQueue())
                .to(orderExchange())
                .with(ORDER_CREATED_ROUTING_KEY);
    }

    @Bean
    public Binding orderCreatedDlqBinding() {
        return BindingBuilder
                .bind(orderCreatedDlq())
                .to(orderDlxExchange())
                .with(ORDER_CREATED_DLK);
    }
}
```

## 容器配置

```java
@Configuration
public class RabbitConsumerConfig {

    @Bean
    public MessageConverter jacksonMessageConverter() {
        return new Jackson2JsonMessageConverter();
    }

    @Bean
    public SimpleRabbitListenerContainerFactory reliableRabbitListenerContainerFactory(
            ConnectionFactory connectionFactory,
            MessageConverter jacksonMessageConverter
    ) {
        SimpleRabbitListenerContainerFactory factory =
                new SimpleRabbitListenerContainerFactory();

        factory.setConnectionFactory(connectionFactory);
        factory.setMessageConverter(jacksonMessageConverter);

        // 可靠消费：手动 ack
        factory.setAcknowledgeMode(AcknowledgeMode.MANUAL);

        // 并发消费
        factory.setConcurrentConsumers(3);
        factory.setMaxConcurrentConsumers(10);

        // 控制每个消费者一次拉取的未 ack 消息数量
        factory.setPrefetchCount(20);

        // 发生异常时不要默认重新入队，避免死循环
        factory.setDefaultRequeueRejected(false);

        return factory;
    }
}
```

## 消费者

```java
@Component
@Slf4j
@RequiredArgsConstructor
public class OrderCreatedConsumer {

    private final OrderApplicationService orderApplicationService;
    private final ProcessedMessageRepository processedMessageRepository;

    @RabbitListener(
            id = "orderCreatedListener",
            queues = OrderRabbitConfig.ORDER_CREATED_QUEUE,
            containerFactory = "reliableRabbitListenerContainerFactory",
            concurrency = "3-10"
    )
    public void handle(
            OrderCreatedMessage message,
            Message rawMessage,
            Channel channel
    ) throws IOException {

        long deliveryTag = rawMessage.getMessageProperties().getDeliveryTag();
        String messageId = rawMessage.getMessageProperties().getMessageId();

        try {
            log.info("收到订单创建消息，messageId={}, message={}", messageId, message);

            // 1. 幂等校验：防止重复消费
            if (processedMessageRepository.exists(messageId)) {
                log.warn("消息已处理，直接 ack，messageId={}", messageId);
                channel.basicAck(deliveryTag, false);
                return;
            }

            // 2. 执行业务逻辑
            orderApplicationService.handleOrderCreated(message);

            // 3. 记录消息已处理
            processedMessageRepository.save(messageId);

            // 4. 手动确认
            channel.basicAck(deliveryTag, false);

        } catch (BizNonRetryableException e) {
            log.error("不可重试业务异常，消息进入死信队列，messageId={}", messageId, e);

            // 不重新入队，进入 DLQ
            channel.basicNack(deliveryTag, false, false);

        } catch (Exception e) {
            log.error("系统异常，消息进入死信队列，messageId={}", messageId, e);

            // 生产中不建议无限 requeue=true
            // 这里选择进入死信队列，由后续补偿任务处理
            channel.basicNack(deliveryTag, false, false);
        }
    }
}
```

这个模型比简单的：

```java
@RabbitListener(queues = "xxx")
public void handle(String msg) {}
```

更接近真实生产环境。

---

# 16. `@RabbitListener` 和 RocketMQ Listener 的差异

你之前学过 RocketMQ，可以这样类比：

|维度|RocketMQ|RabbitMQ + Spring AMQP|
|---|---|---|
|消费注解|`@RocketMQMessageListener`|`@RabbitListener`|
|消费目标|Topic + ConsumerGroup|Queue|
|路由模型|Topic / Tag|Exchange / RoutingKey / Queue|
|消费组语义|原生强消费组概念|RabbitMQ 主要是 Queue 竞争消费|
|顺序消费|RocketMQ 支持更明确|RabbitMQ 需要通过队列/并发控制实现|
|重试机制|Broker/客户端体系内置较明显|常通过 nack、DLX、重试队列组合|
|注解职责|声明消费者|声明监听方法 + 可声明 Queue/Exchange/Binding|

RabbitMQ 的核心不是 Topic，而是：

```text
Producer → Exchange → Binding → Queue → Consumer
```

所以 `@RabbitListener` 更关注：

```text
监听哪个 Queue
容器怎么消费 Queue
失败消息怎么 ack/nack
```

而不是像 RocketMQ 那样围绕：

```text
Topic + ConsumerGroup + Tag
```

---

# 17. 实际开发中推荐的写法

## 推荐结构

```text
rabbit
├── config
│   ├── OrderRabbitConfig.java          # Exchange / Queue / Binding
│   └── RabbitConsumerConfig.java       # ListenerContainerFactory
├── message
│   └── OrderCreatedMessage.java        # 消息体 DTO
├── producer
│   └── OrderProducer.java              # 发送消息
└── consumer
    └── OrderCreatedConsumer.java       # @RabbitListener 消费
```

## 推荐注解写法

```java
@RabbitListener(
        id = "orderCreatedListener",
        queues = OrderRabbitConfig.ORDER_CREATED_QUEUE,
        containerFactory = "reliableRabbitListenerContainerFactory",
        concurrency = "${rabbit.listener.order-created.concurrency:3-10}",
        autoStartup = "${rabbit.listener.order-created.auto-startup:true}"
)
public void handle(
        OrderCreatedMessage message,
        Message rawMessage,
        Channel channel
) {
    // 消费逻辑
}
```

配置：

```yaml
rabbit:
  listener:
    order-created:
      concurrency: 3-10
      auto-startup: true
```

这种写法的优点：

```text
队列名集中管理
容器参数集中管理
并发和启动开关可配置
消费逻辑清晰
方便灰度和运维
```

---

# 18. 常见坑

## 坑 1：`queues` 写错，服务启动失败或一直消费不到

```java
@RabbitListener(queues = "order.create.queue")
```

但实际队列叫：

```text
order.created.queue
```

建议统一常量：

```java
public static final String ORDER_CREATED_QUEUE = "order.created.queue";
```

---

## 坑 2：没有配置 JSON Converter，导致对象反序列化失败

错误现象：

```text
Cannot convert from [java.lang.String] to [OrderCreatedMessage]
```

解决：

```java
@Bean
public MessageConverter jacksonMessageConverter() {
    return new Jackson2JsonMessageConverter();
}
```

---

## 坑 3：消费失败反复重入队，造成死循环

危险写法：

```java
channel.basicNack(deliveryTag, false, true);
```

如果消息本身有问题，会无限失败。

更推荐：

```java
channel.basicNack(deliveryTag, false, false);
```

然后配合死信队列。

---

## 坑 4：没有做幂等

RabbitMQ 和大部分 MQ 一样，业务上要按“至少一次投递”处理。

消费者必须假设：

```text
同一条消息可能被消费多次
```

常见幂等手段：

```text
messageId 去重表
业务唯一索引
状态机判断
Redis SETNX
订单号 + 事件类型唯一约束
```

---

## 坑 5：并发开太大，把数据库打爆

```java
concurrency = "20-100"
```

看起来吞吐高，实际可能导致：

```text
数据库连接池耗尽
下游接口超时
锁竞争加剧
消息大量失败
死信队列暴涨
```

并发应该根据下游承载能力压测出来。

---

# 19. 高频属性总结表

|属性|作用|实际建议|
|---|---|---|
|`queues`|监听已有队列|生产常用|
|`queuesToDeclare`|声明并监听队列|demo / 小项目可用|
|`bindings`|声明队列、交换机、绑定|demo 可用，生产慎用|
|`containerFactory`|指定监听容器工厂|生产强烈建议用|
|`concurrency`|配置消费者并发|常用，建议配置化|
|`ackMode`|配置 ack 模式|可靠消费用 MANUAL|
|`messageConverter`|指定消息转换器|多数场景放 factory 里|
|`id`|监听容器 id|需要动态启停时使用|
|`autoStartup`|是否自动启动|灰度/初始化场景有用|
|`errorHandler`|监听异常处理器|日志、告警、异常包装|
|`replyTo`|指定响应队列|RabbitMQ RPC 场景|
|`returnExceptions`|是否返回异常给调用方|RPC 场景才考虑|
|`admin`|指定 RabbitAdmin|多 RabbitMQ 实例时可能用|
|`group`|监听容器分组|管理多个 listener 时使用|

---

# 20. 面试加分项

你可以这样回答 `@RabbitListener`：

> `@RabbitListener` 不是简单的方法注解，它背后会注册 RabbitListenerEndpoint，并由 ListenerContainerFactory 创建消息监听容器。容器负责连接 RabbitMQ、订阅队列、拉取消息、线程并发、ack/nack、异常处理和消息转换。实际生产中我不会只写 `@RabbitListener(queues = "...")`，而是会单独配置 `SimpleRabbitListenerContainerFactory`，设置 JSON Converter、手动 ack、prefetch、并发、异常重入队策略，并配合死信队列和幂等表保证可靠消费。

再进一步：

> RabbitMQ 消费端可靠性重点不是注解本身，而是消费模型设计：手动 ack、业务幂等、死信队列、重试策略、并发控制、prefetch 控制、监控告警。`@RabbitListener` 只是 Spring AMQP 提供的声明式入口，真正的工程能力在 ListenerContainer 和 RabbitMQ 拓扑设计上。

---

# 关键词总结

```text
@RabbitListener
queues
queuesToDeclare
bindings
containerFactory
SimpleRabbitListenerContainerFactory
MessageConverter
Jackson2JsonMessageConverter
ackMode
MANUAL ACK
basicAck
basicNack
concurrency
prefetch
dead letter queue
idempotency
RabbitListenerEndpointRegistry
```

核心记忆：

```text
@RabbitListener = 声明消费入口
containerFactory = 定义消费运行时
Queue/Exchange/Binding = 定义 RabbitMQ 拓扑
ack/nack + DLQ + 幂等 = 定义可靠消费能力
```