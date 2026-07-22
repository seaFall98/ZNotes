[16](https://xiaolinnote.com/ai/tools/16_llm_gateway.html#%E8%AF%AD%E4%B9%89%E7%BC%93%E5%AD%98-%E7%9C%81%E9%92%B1%E7%9A%84%E5%90%8C%E6%97%B6%E9%99%8D%E5%BB%B6%E8%BF%9F)

https://www.shengyayun.com/blog/tech-implementation-2026-04-21/# 技术热点落地：AI Gateway 语义缓存 + 智能路由，生产环境 LLM 成本削减 60% 实战

https://help.aliyun.com/zh/redis/tair-ai-gateway-semantic-caching/ 阿里云语义缓存网关概述

AI GATEWAY做什么
AI Gateway

├── Authentication
│
├── Authorization
│
├── Rate Limit
│
├── Model Router
│
├── Fallback
│
├── Semantic Cache
│
├── Prompt Management
│
├── Guardrails
│
├── Token Budget
│
├── Observability
│
└── Evaluation Hook

## 缓存问题

---
例1 "今天天气如何"和"今天天气怎么样" 缓存如何处理？

属于同一问题 缓存结果共享。

需要做语义识别 内容分类 选择不同的缓存策略

User Request

↓

Intent Classification

↓

Cache Policy

但是对于这种个性化或者强实时性的问答，甚至可以直接绕过缓存走 LLM。

---

例2：

```
公司的报销流程是什么？
```

判断：

```
Static Knowledge
```

策略：

```
TTL = 7天
```

甚至：

```
永久缓存
+
版本控制
```

---

企业不会只缓存：

```
question → answer
```

而是：

```
cache key =
{
 query embedding,

 model,

 system prompt version,

 user permission,

 tenant,

 knowledge version  #版本管理
}
```

AI Gateway 的语义缓存不是“相似问题返回相同答案”，而是“在语义相似、上下文一致、权限一致、数据仍有效的情况下复用模型结果”。实际非常复杂。。。