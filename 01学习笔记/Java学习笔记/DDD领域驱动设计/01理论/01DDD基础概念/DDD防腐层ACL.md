## ACL 是什么？

在 DDD 里，**ACL = Anti-Corruption Layer，防腐层**。

它的核心作用是：

> **当你的领域模型需要和外部系统、其他限界上下文、遗留系统交互时，用一层隔离层保护自己的领域模型不被外部模型“污染”。**

这里的“腐”不是代码烂，而是指：

> 外部系统的概念、字段、状态、接口风格、数据结构，直接侵入了你自己的领域模型。

---

## 一个直观例子

假设你在做电商的 **订单上下文 Order Context**。

你的领域模型里有：

```java
Order
OrderItem
Money
Address
OrderStatus
```

但是你需要调用第三方支付系统。

第三方支付返回的数据可能是这样的：

```json
{
  "trade_no": "202605100001",
  "pay_state": "SUCCESS",
  "amount_cent": 9900,
  "channel": "ALI_PAY",
  "buyer_account": "138****8888"
}
```

如果你在订单领域里直接使用这些字段：

```java
order.setTradeNo(response.getTradeNo());
order.setPayState(response.getPayState());
order.setAmountCent(response.getAmountCent());
```

问题来了：

- `trade_no` 是支付系统的概念，不一定是订单领域概念
    
- `pay_state` 的状态值是支付系统定义的
    
- `amount_cent` 是支付系统的数据表达方式
    
- `channel`、`buyer_account` 可能跟订单核心业务无关
    
- 如果支付系统字段变化，订单领域代码也跟着改
    

这就是**外部模型污染内部模型**。

---

## 加上 ACL 后

你会在订单上下文里加一个防腐层：

```text
Order Context
    |
    | 调用
    v
Payment ACL
    |
    | 调用
    v
第三方支付系统
```

防腐层负责：

```text
第三方支付返回模型
        ↓ 转换
订单上下文能理解的模型
```

例如：

```java
public class PaymentAcl {

    private final ThirdPartyPaymentClient paymentClient;

    public PaymentResult pay(PaymentCommand command) {
        ThirdPartyPayRequest request = toThirdPartyRequest(command);

        ThirdPartyPayResponse response = paymentClient.pay(request);

        return toPaymentResult(response);
    }

    private ThirdPartyPayRequest toThirdPartyRequest(PaymentCommand command) {
        return new ThirdPartyPayRequest(
            command.orderId().value(),
            command.amount().toCent(),
            command.payChannel().code()
        );
    }

    private PaymentResult toPaymentResult(ThirdPartyPayResponse response) {
        return new PaymentResult(
            new PaymentId(response.getTradeNo()),
            convertStatus(response.getPayState())
        );
    }

    private PaymentStatus convertStatus(String payState) {
        return switch (payState) {
            case "SUCCESS" -> PaymentStatus.PAID;
            case "FAILED" -> PaymentStatus.FAILED;
            case "PROCESSING" -> PaymentStatus.PROCESSING;
            default -> PaymentStatus.UNKNOWN;
        };
    }
}
```

订单领域只认识自己的模型：

```java
PaymentResult result = paymentAcl.pay(command);

order.pay(result.paymentId());
```

而不是直接认识第三方支付的 `trade_no`、`pay_state`、`amount_cent`。

---

## ACL 防什么？

主要防 4 类污染。

### 1. 防字段污染

外部系统字段直接进入你的领域对象。

```java
// 不推荐
order.setTradeNo(thirdPartyResponse.getTradeNo());
```

更好的方式：

```java
PaymentId paymentId = paymentAcl.convertToPaymentId(response);
order.pay(paymentId);
```

---

### 2. 防状态污染

外部系统的状态枚举直接进入你的领域。

比如支付系统状态：

```text
SUCCESS
FAIL
PROCESSING
CLOSED
REFUNDING
```

订单领域状态可能是：

```text
CREATED
PAID
CANCELLED
DELIVERED
```

不能简单让订单系统直接依赖支付系统状态。

应该做转换：

```text
支付 SUCCESS -> 订单 PAID
支付 FAIL -> 订单仍然待支付或支付失败
支付 PROCESSING -> 订单保持 CREATED
```

---

### 3. 防语义污染

不同上下文里，同一个词可能意思不同。

比如 `Customer`：

|上下文|Customer 含义|
|---|---|
|订单上下文|下单人|
|营销上下文|可投放用户|
|售后上下文|售后申请人|
|CRM 上下文|客户资产对象|

如果订单上下文直接复用 CRM 的 `CustomerDTO`，就会让订单模型被 CRM 语义污染。

---

### 4. 防变化污染

外部系统改接口、改字段、改状态值，不应该导致你的领域核心大面积修改。

ACL 的价值是把变化挡在边界外：

```text
外部系统变了
    ↓
只改 ACL
    ↓
领域模型尽量不动
```

---

## ACL 放在哪里？

通常放在 **基础设施层 Infrastructure** 或者 **接口适配层 Adapter**。

典型结构：

```text
order
├── domain
│   ├── Order.java
│   ├── OrderItem.java
│   ├── Money.java
│   └── OrderStatus.java
│
├── application
│   └── OrderAppService.java
│
├── infrastructure
│   └── acl
│       └── PaymentAcl.java
│
└── interfaces
    └── OrderController.java
```

也可以这样理解：

|层|作用|
|---|---|
|Domain|核心业务模型|
|Application|编排用例|
|Infrastructure / ACL|对接外部系统并转换模型|
|Interfaces|接收 HTTP、RPC、MQ 请求|

---

## ACL 和 Adapter 有什么区别？

两者很接近，但关注点不同。

|概念|重点|
|---|---|
|Adapter 适配器|技术适配，比如 HTTP、RPC、MQ、数据库|
|ACL 防腐层|语义隔离，防止外部模型污染领域模型|

举例：

```text
PaymentHttpClient
```

更偏技术适配，负责发 HTTP 请求。

```text
PaymentAcl
```

更偏领域边界，负责把支付系统模型翻译成订单上下文能理解的模型。

实际项目中，这两个经常一起出现：

```text
OrderAppService
    -> PaymentAcl
        -> PaymentHttpClient
            -> 第三方支付系统
```

---

## ACL 和 DTO Converter 有什么区别？

普通 DTO Converter 只是字段转换。

ACL 不只是字段转换，而是**上下文之间的语义翻译**。

例如：

```java
SUCCESS -> PAID
amount_cent -> Money
trade_no -> PaymentId
channel -> PayChannel
```

这不只是 `copyProperties`，而是把外部系统语言翻译成当前领域语言。

所以 ACL 更像：

> **上下文边界上的翻译官。**

---

## 一句话总结

**ACL 防腐层就是两个模型世界之间的翻译层。**

它的职责是：

```text
外部系统模型 / 其他上下文模型
        ↓
防腐层翻译、隔离、适配
        ↓
当前上下文自己的领域模型
```

它解决的问题是：

> **不要让别人的模型、字段、状态、接口风格，侵入你的核心领域模型。**

在 DDD 里，只要跨限界上下文、跨外部系统、跨遗留系统，尤其是模型语义不一致时，就应该考虑 ACL。