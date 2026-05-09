# 总述
在所有的模型都定义完成后，领域业务被串联了。那么接下来则是使用，而使用的方式可以包括；接口（http/rpc）、消息监听、定时任务等方式，这些方式统一被定义为触发动作。

由触发发起对编排功能的调用处理，如：定时任务做信贷的计息、开户成功消息通知返利优惠券、提供接口让外部调用授信逻辑等——这些都是触发动作。

# DDD 里的“触发器”是什么？

一句话：

> **触发器就是业务用例的入口。它负责接收外部动作，然后调用应用层编排能力。**

这里的“触发器”不是数据库 trigger，而是更广义的业务入口。

它可以是：

```text
HTTP 接口
RPC 接口
MQ 消息监听
定时任务
命令行任务
后台管理操作
第三方回调
事件订阅器
```

它们的共同点是：

```text
外部世界发生了一个动作
        ↓
触发器接收到这个动作
        ↓
转换成 Command / Event / Query
        ↓
调用 Application Service
        ↓
Application Service 编排领域能力
```

---

# 1. 触发器在 DDD 分层中的位置

常见结构：

```text
interfaces / trigger
├── controller
├── rpc
├── mq
├── scheduler
└── listener

application
├── service
├── command
├── dto
└── assembler

domain
├── model
├── service
├── repository
└── event

infrastructure
├── persistence
├── rpc-client
├── mq
├── cache
└── config
```

依赖方向：

```text
Trigger
   ↓
Application Service
   ↓
Domain
   ↓
Repository / Gateway 接口
   ↑
Infrastructure 实现
```

也就是说：

> **触发器不直接承载领域规则，它只是业务流程的入口适配层。**

---

# 2. 为什么需要“触发器”这个概念？

因为同一个业务能力，可能会被不同方式调用。

例如“授信审批”：

```text
方式 1：用户在 APP 上点击申请授信
方式 2：外部系统通过 RPC 调用授信接口
方式 3：后台运营手动触发授信复核
方式 4：定时任务扫描待审批用户
方式 5：风控系统发 MQ 消息触发重新评估
```

这些入口不同，但核心用例可能是同一个：

```java
creditApplicationService.evaluateCredit(command);
```

所以 DDD 会倾向于把入口和业务能力分开：

```text
触发器：负责“怎么被触发”
应用服务：负责“这个用例怎么执行”
领域层：负责“业务规则是什么”
```

---

# 3. 触发器不应该做什么？

触发器不应该写大量业务逻辑。

错误示例：

```java
@RestController
public class CreditController {

    @PostMapping("/credit/apply")
    public CreditResponse apply(@RequestBody CreditRequest request) {
        // 参数校验
        if (request.getUserId() == null) {
            throw new RuntimeException("用户不能为空");
        }

        // 查用户
        User user = userMapper.selectById(request.getUserId());

        // 查征信
        CreditReport report = creditRpcClient.queryReport(user.getId());

        // 计算额度
        BigDecimal amount = BigDecimal.ZERO;
        if (report.getScore() > 700) {
            amount = new BigDecimal("100000");
        } else {
            amount = new BigDecimal("10000");
        }

        // 写库
        creditMapper.insert(...);

        return new CreditResponse(amount);
    }
}
```

这会导致：

```text
Controller 变成业务 Service
HTTP 入口和业务规则强绑定
以后 MQ / 定时任务也要复制一遍逻辑
测试困难
复用困难
```

---

# 4. 正确的触发器写法

Controller 只做入口适配：

```java
@RestController
@RequestMapping("/credit")
public class CreditController {

    private final CreditApplicationService creditApplicationService;

    public CreditController(CreditApplicationService creditApplicationService) {
        this.creditApplicationService = creditApplicationService;
    }

    @PostMapping("/apply")
    public CreditResponse apply(@RequestBody CreditApplyRequest request) {
        CreditApplyCommand command = new CreditApplyCommand(
                request.userId(),
                request.productCode(),
                request.applyAmount()
        );

        CreditApplyResult result = creditApplicationService.apply(command);

        return CreditResponse.from(result);
    }
}
```

Controller 的职责是：

```text
接收 HTTP 请求
做基础参数转换
构造 Command
调用应用服务
转换 Response
```

不负责：

```text
额度计算
风控规则
状态流转
领域对象保存细节
RPC 技术调用细节
```

---

# 5. 触发器和应用服务的关系

可以这样理解：

```text
触发器 = 按钮
应用服务 = 按钮背后的流程
领域层 = 流程里的业务规则
基础设施层 = 流程用到的技术工具
```

例如：

```text
用户点击“提交订单”按钮
        ↓
OrderController
        ↓
SubmitOrderApplicationService
        ↓
Order / InventoryGateway / OrderRepository
```

或者：

```text
MQ 收到“支付成功”消息
        ↓
PaymentSuccessMessageListener
        ↓
HandlePaymentSuccessApplicationService
        ↓
Order.markPaid()
```

---

# 6. 触发器的常见类型

## 6.1 HTTP Controller

最常见的入口。

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    private final SubmitOrderApplicationService submitOrderApplicationService;

    @PostMapping("/{orderId}/submit")
    public SubmitOrderResponse submit(@PathVariable Long orderId) {
        SubmitOrderCommand command = new SubmitOrderCommand(orderId);
        SubmitOrderResult result = submitOrderApplicationService.submit(command);
        return SubmitOrderResponse.from(result);
    }
}
```

适合：

```text
前端页面调用
APP 调用
开放 API
后台管理接口
```

---

## 6.2 RPC Provider

比如 Dubbo / gRPC / OpenFeign Provider。

```java
@DubboService
public class CreditRpcProvider implements CreditRpcService {

    private final CreditApplicationService creditApplicationService;

    @Override
    public CreditRpcResponse apply(CreditRpcRequest request) {
        CreditApplyCommand command = new CreditApplyCommand(
                request.getUserId(),
                request.getProductCode(),
                request.getApplyAmount()
        );

        CreditApplyResult result = creditApplicationService.apply(command);

        return CreditRpcResponse.from(result);
    }
}
```

它和 Controller 本质一样：

```text
HTTP Controller 是 Web 触发器
RPC Provider 是 RPC 触发器
```

区别只是协议不同。

---

## 6.3 MQ 消息监听器

例如支付成功后，订单系统收到支付系统消息：

```java
@Component
public class PaymentSuccessMessageListener {

    private final PaymentSuccessApplicationService paymentSuccessApplicationService;

    @RabbitListener(queues = "payment.success.queue")
    public void onMessage(PaymentSuccessMessage message) {
        PaymentSuccessCommand command = new PaymentSuccessCommand(
                message.orderId(),
                message.paymentId(),
                message.paidAmount(),
                message.paidAt()
        );

        paymentSuccessApplicationService.handle(command);
    }
}
```

监听器只负责：

```text
接收消息
解析消息
构造 Command
调用应用服务
确认消费 / 异常处理
```

不应该在 Listener 里直接写：

```text
改订单状态
扣库存
发积分
发通知
```

这些应该在应用服务和领域层处理。

---

## 6.4 定时任务 Scheduler

例如信贷系统每天计息：

```java
@Component
public class LoanInterestScheduler {

    private final LoanInterestApplicationService loanInterestApplicationService;

    @Scheduled(cron = "0 0 1 * * ?")
    public void calculateDailyInterest() {
        loanInterestApplicationService.calculateDailyInterest();
    }
}
```

定时任务是典型触发器：

```text
不是用户触发
不是消息触发
而是时间触发
```

它适合：

```text
每日计息
订单超时取消
优惠券过期
积分结算
账单生成
数据对账
定期补偿
```

---

## 6.5 事件监听器

领域事件或集成事件也可以触发后续用例。

例如订单创建成功后，发放优惠券：

```java
@Component
public class AccountOpenedEventListener {

    private final CouponApplicationService couponApplicationService;

    public void onEvent(AccountOpenedEvent event) {
        IssueCouponCommand command = new IssueCouponCommand(
                event.userId(),
                "NEW_ACCOUNT_COUPON"
        );

        couponApplicationService.issue(command);
    }
}
```

这里：

```text
开户成功事件
        ↓
触发优惠券发放
        ↓
调用优惠券应用服务
```

---

# 7. 用一个例子串起来：信贷授信

假设有一个“授信评估”用例。

它可能被三个入口触发：

```text
1. HTTP 接口：用户主动申请授信
2. MQ 消息：开户成功后自动授信
3. 定时任务：每天扫描待评估用户
```

但核心应用服务只有一个：

```java
@Service
public class CreditEvaluationApplicationService {

    private final CreditRepository creditRepository;
    private final UserProfileGateway userProfileGateway;
    private final CreditReportGateway creditReportGateway;
    private final CreditPolicyDomainService creditPolicyDomainService;

    @Transactional
    public CreditEvaluationResult evaluate(CreditEvaluationCommand command) {
        UserProfile userProfile = userProfileGateway.getUserProfile(command.userId());
        CreditReport creditReport = creditReportGateway.getReport(command.userId());

        CreditAccount creditAccount = creditRepository.findOrCreate(command.userId());

        CreditLimit creditLimit = creditPolicyDomainService.evaluate(
                userProfile,
                creditReport
        );

        creditAccount.grant(creditLimit);

        creditRepository.save(creditAccount);

        return CreditEvaluationResult.of(creditAccount.id(), creditLimit);
    }
}
```

---

## HTTP 触发

```java
@RestController
@RequestMapping("/credit")
public class CreditController {

    private final CreditEvaluationApplicationService creditEvaluationApplicationService;

    @PostMapping("/evaluate")
    public CreditEvaluationResponse evaluate(@RequestBody CreditEvaluationRequest request) {
        CreditEvaluationCommand command = new CreditEvaluationCommand(
                request.userId(),
                "HTTP_REQUEST"
        );

        CreditEvaluationResult result = creditEvaluationApplicationService.evaluate(command);

        return CreditEvaluationResponse.from(result);
    }
}
```

---

## MQ 触发

```java
@Component
public class AccountOpenedMessageListener {

    private final CreditEvaluationApplicationService creditEvaluationApplicationService;

    @RabbitListener(queues = "account.opened.queue")
    public void onMessage(AccountOpenedMessage message) {
        CreditEvaluationCommand command = new CreditEvaluationCommand(
                message.userId(),
                "ACCOUNT_OPENED_MESSAGE"
        );

        creditEvaluationApplicationService.evaluate(command);
    }
}
```

---

## 定时任务触发

```java
@Component
public class CreditEvaluationScheduler {

    private final CreditEvaluationApplicationService creditEvaluationApplicationService;
    private final PendingCreditUserQueryService pendingCreditUserQueryService;

    @Scheduled(cron = "0 0 2 * * ?")
    public void evaluatePendingUsers() {
        List<UserId> userIds = pendingCreditUserQueryService.queryPendingUsers();

        for (UserId userId : userIds) {
            CreditEvaluationCommand command = new CreditEvaluationCommand(
                    userId,
                    "SCHEDULED_JOB"
            );

            creditEvaluationApplicationService.evaluate(command);
        }
    }
}
```

这就是触发器的价值：

```text
入口可以有多个
应用服务可以复用
领域规则只写一份
```

---

# 8. 触发器和适配器的关系

严格来说，触发器也是一种适配器。

在六边形架构里，适配器分两类：

```text
入站适配器 inbound adapter
出站适配器 outbound adapter
```

## 入站适配器

负责调用系统内部能力。

```text
HTTP Controller
RPC Provider
MQ Listener
Scheduler
CLI
```

也就是我们这里说的触发器。

---

## 出站适配器

负责系统内部调用外部能力。

```text
RepositoryImpl
RedisAdapter
PaymentGatewayImpl
InventoryRpcAdapter
MessagePublisher
FileStorageAdapter
```

所以关系是：

```text
触发器 = 入站适配器
仓储 / 网关实现 = 出站适配器
```

图示：

```text
外部请求 / 消息 / 时间
        ↓
┌────────────────────┐
│ Trigger             │
│ 入站适配器           │
└─────────┬──────────┘
          ↓
┌────────────────────┐
│ Application Service │
│ 用例编排             │
└─────────┬──────────┘
          ↓
┌────────────────────┐
│ Domain              │
│ 领域模型 / 领域服务   │
└─────────┬──────────┘
          ↓
┌────────────────────┐
│ Port 接口            │
│ Repository / Gateway │
└─────────┬──────────┘
          ↑
┌────────────────────┐
│ Outbound Adapter     │
│ MySQL / Redis / RPC  │
└────────────────────┘
```

---

# 9. 触发器层可以直接调用领域层吗？

可以，但只适合非常简单的场景。

例如：

```java
@PostMapping("/articles/{id}/publish")
public void publish(@PathVariable Long id) {
    Article article = articleRepository.find(new ArticleId(id));
    article.publish();
    articleRepository.save(article);
}
```

这种写法可以接受，前提是：

```text
只有一个入口
只操作一个聚合
没有复杂事务
没有外部系统调用
没有复用需求
没有 MQ / 定时任务等其他触发方式
```

一旦出现下面情况，就应该引入应用服务：

```text
同一个用例会被 HTTP、MQ、定时任务多种方式触发
需要事务控制
需要调用多个仓储
需要调用外部网关
需要发布事件
需要幂等处理
需要补偿和重试
需要权限、审计、日志、上下文追踪
```

否则这些逻辑会分散在多个触发器里。

---

# 10. 触发器层的职责边界

触发器应该做：

```text
协议适配
参数接收
DTO / Command 转换
基础参数校验
鉴权入口处理
调用应用服务
异常转换
返回结果
消息 ack / nack
任务调度
```

触发器不应该做：

```text
核心业务规则
复杂状态流转
价格计算
额度计算
库存扣减规则
直接调用 Mapper
直接操作 Redis
直接调用多个领域对象完成复杂流程
直接发 MQ 完成业务流转
```

简化成一句：

> **触发器负责“接进来”，应用服务负责“串起来”，领域层负责“算明白”。**

---

# 11. 不同触发器的特殊关注点

## 11.1 HTTP 触发器

关注：

```text
请求参数校验
认证鉴权
接口幂等
响应码转换
异常映射
DTO 转换
```

例如：

```text
DomainException → 400 / 409
ApplicationException → 业务错误码
SystemException → 500
```

---

## 11.2 MQ 触发器

关注：

```text
消息反序列化
消息幂等
重复消费
消费失败重试
死信队列
消息顺序
ack 时机
```

MQ Listener 特别不能随便吞异常。

否则会出现：

```text
消息消费失败但被确认
业务没处理成功
后续也不会重试
数据不一致
```

---

## 11.3 定时任务触发器

关注：

```text
分布式锁
分页扫描
批量大小
任务幂等
失败续跑
避免全表扫描
任务监控
补偿机制
```

例如订单超时取消：

```text
不能一次性扫全部订单
不能多个节点同时取消同一批订单
不能取消已经支付成功的订单
```

所以真正的取消规则仍然应该在领域对象里：

```java
order.cancelDueToTimeout(now);
```

而不是 Scheduler 自己判断一堆状态。

---

## 11.4 RPC 触发器

关注：

```text
接口契约稳定性
版本兼容
超时控制
异常转换
调用方隔离
参数兼容
```

RPC Provider 不应该直接暴露领域对象。

不要这样：

```java
Order getOrder(Long orderId);
```

更推荐返回 DTO：

```java
OrderRpcDTO getOrder(Long orderId);
```

原因：

```text
领域对象是内部模型
RPC DTO 是外部契约
二者不应该强绑定
```

---

# 12. 触发器和 Command 的关系

触发器通常不直接把 Request / Message 传给应用服务。

不推荐：

```java
creditApplicationService.apply(request);
```

更推荐：

```java
CreditApplyCommand command = CreditApplyCommand.from(request);
creditApplicationService.apply(command);
```

原因是：

```text
Request 属于 HTTP 协议
Message 属于 MQ 协议
Command 属于应用用例
```

同一个 Command 可以来自不同触发器：

```text
HTTP Request
        ↓
CreditApplyCommand

MQ Message
        ↓
CreditApplyCommand

Scheduler Job
        ↓
CreditApplyCommand
```

这样应用服务不会被具体触发方式污染。

---

# 13. 触发器和领域事件的区别

这两个也容易混。

## 触发器

触发器是入口。

```text
HTTP 请求来了
MQ 消息来了
定时任务到了
RPC 调用了
```

它负责调用应用服务。

---

## 领域事件

领域事件是领域内部发生了某个事实。

```text
OrderCreatedEvent
OrderPaidEvent
AccountOpenedEvent
CouponIssuedEvent
CreditGrantedEvent
```

领域事件通常由领域行为产生：

```java
order.markPaid(paymentId);
order.addDomainEvent(new OrderPaidEvent(order.id(), paymentId));
```

然后事件可以被发布出去，进一步触发其他流程。

所以链路可能是：

```text
HTTP Controller 触发
        ↓
PayOrderApplicationService
        ↓
Order.markPaid()
        ↓
OrderPaidEvent
        ↓
MQ Listener / Event Handler
        ↓
发积分 / 发通知 / 生成账单
```

触发器是“外部输入”。

领域事件是“内部事实”。

---

# 14. 触发器和编排层的关系

你前面学过领域编排，可以这样串起来：

```text
触发器层
    负责接收动作

应用编排层
    负责组织用例流程

领域层
    负责核心业务规则

基础设施层
    负责技术实现
```

例如“开户成功发放返利优惠券”：

```text
AccountOpenedMessageListener
    ↓ 接收到开户成功消息

CouponApplicationService.issueRebateCoupon()
    ↓ 编排优惠券发放流程

CouponPolicyDomainService
Coupon
    ↓ 执行优惠券领域规则

CouponRepository
MessagePublisher
    ↓ 保存优惠券，发布发放成功事件
```

代码结构：

```java
@Component
public class AccountOpenedMessageListener {

    private final CouponApplicationService couponApplicationService;

    public void onMessage(AccountOpenedMessage message) {
        IssueRebateCouponCommand command = new IssueRebateCouponCommand(
                message.userId(),
                message.accountId()
        );

        couponApplicationService.issueRebateCoupon(command);
    }
}
```

---

# 15. 触发器设计的核心原则

## 原则一：入口可以多，应用服务尽量复用

错误：

```text
HTTP Controller 写一套授信逻辑
MQ Listener 写一套授信逻辑
Scheduler 又写一套授信逻辑
```

正确：

```text
HTTP Controller
MQ Listener
Scheduler
        ↓
CreditApplicationService.evaluate()
```

---

## 原则二：触发器只做适配，不做规则

触发器可以知道：

```text
HTTP 怎么解析
MQ 怎么确认
定时任务怎么调度
RPC 怎么返回
```

但不应该知道：

```text
授信额度怎么算
订单状态怎么流转
优惠券是否可发放
库存是否可锁定
```

---

## 原则三：不要把协议模型传进领域层

不要让领域层依赖：

```text
HttpServletRequest
ResponseEntity
RabbitMQ Message
Dubbo Request
Cron Context
```

应该转换成：

```text
Command
Query
Domain Value Object
```

---

## 原则四：不同触发器可以有不同前置处理

例如同一个授信能力：

```text
HTTP 触发：需要用户鉴权
RPC 触发：需要调用方鉴权
MQ 触发：需要消息幂等
Scheduler 触发：需要分布式锁
```

这些是触发器层该关心的。

但进入应用服务后，应该尽量统一。

---

# 16. 一个完整链路图

```text
┌─────────────────────────────┐
│ 外部动作                     │
│ HTTP / RPC / MQ / Scheduler  │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Trigger 触发器               │
│ Controller / Listener / Job  │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Command / Query              │
│ 用例输入模型                  │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Application Service          │
│ 编排流程 / 事务 / 事件         │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Domain                       │
│ 聚合 / 实体 / 值对象 / 规则    │
└──────────────┬──────────────┘
               ↓
┌─────────────────────────────┐
│ Repository / Gateway Port    │
│ 领域需要的抽象能力             │
└──────────────┬──────────────┘
               ↑
┌─────────────────────────────┐
│ Infrastructure Adapter       │
│ MySQL / Redis / MQ / RPC      │
└─────────────────────────────┘
```

---

# 17. 用一句话把仓储、适配器、编排、触发器串起来

```text
触发器：谁来发起业务动作
编排层：业务流程怎么串起来
领域层：业务规则怎么表达
仓储：领域对象怎么保存和加载
适配器：外部系统怎么接入
基础设施层：具体技术怎么落地
```

放到完整调用链里：

```text
HTTP / MQ / Scheduler
        ↓
Trigger
        ↓
Application Service
        ↓
Domain Model / Domain Service
        ↓
Repository / Gateway Interface
        ↓
Infrastructure Implementation
```

---

# 18. 最后总结

DDD 中的触发器不是一个神秘概念，它就是**业务入口层**。

它包括：

```text
Controller
RPC Provider
MQ Listener
Scheduler
Event Listener
Command Handler
```

它的核心职责是：

```text
接收外部动作
转换输入模型
调用应用服务
处理协议级返回和异常
```

它不应该承担：

```text
核心业务规则
复杂流程编排
数据持久化细节
外部技术调用细节
```

最精简的一句话：

> **触发器负责把外部世界的动作接进系统，应用服务负责把这个动作编排成业务流程，领域层负责保证业务规则正确。**