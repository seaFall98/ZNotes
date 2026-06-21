## 结论

**Spring AI 的 memory 主要靠三层实现：**

|层级|作用|常用实现|
|---|---|---|
|`ChatMemory`|记忆抽象，负责管理一段对话历史|`MessageWindowChatMemory`|
|`ChatMemoryRepository`|存储层，决定 memory 存在哪里|`InMemoryChatMemoryRepository`、`JdbcChatMemoryRepository`|
|`ChatMemoryAdvisor`|接入 `ChatClient`，在请求前注入历史，在响应后写回 memory|`MessageChatMemoryAdvisor`、`PromptChatMemoryAdvisor`、`VectorStoreChatMemoryAdvisor`|

Spring AI 官方文档把它定义为用于“跨多轮交互存储和检索信息”的聊天记忆能力；LLM 本身是无状态的，Spring AI 通过 Chat Memory 把历史上下文重新塞回 prompt 或 message 列表里。([Home](https://docs.spring.io/spring-ai/reference/api/chat-memory.html?utm_source=chatgpt.com "Chat Memory :: Spring AI Reference"))

---

## 1. 最常用：`MessageWindowChatMemory`

这是目前最核心、最常用的实现。

它的作用是：

> 维护最近 N 条消息，超过窗口后丢弃旧消息，控制上下文长度和 token 成本。

示意：

```java
@Bean
ChatMemory chatMemory(ChatMemoryRepository repository) {
    return MessageWindowChatMemory.builder()
            .chatMemoryRepository(repository)
            .maxMessages(20) // 只保留最近 20 条消息
            .build();
}
```

官方文档中也提到，`MessageWindowChatMemory` 会维护指定大小的消息窗口，默认窗口大小是 20 条消息。([Spring中文网](https://www.spring-doc.cn/spring-ai/1.0.0-M8/api_chat-memory.en.html?utm_source=chatgpt.com "Chat Memory"))

---

## 2. 存储层：`ChatMemoryRepository`

`ChatMemory` 不一定自己负责持久化，真正落库通常靠 `ChatMemoryRepository`。

### 常见实现

|实现|适合场景|
|---|---|
|`InMemoryChatMemoryRepository`|Demo、本地测试、单机临时会话|
|`JdbcChatMemoryRepository`|生产项目常用，MySQL/PostgreSQL 等关系型数据库持久化|
|自定义 Repository|Redis、MongoDB、分库分表、多租户隔离等|

官方文档说明，默认情况下如果没有配置其他 repository，Spring AI 会自动配置 `InMemoryChatMemoryRepository`，它底层使用 `ConcurrentHashMap` 存储消息。JDBC 版本则使用关系型数据库持久化聊天记忆。([Spring Pleiades](https://spring.pleiades.io/spring-ai/reference/api/chat-memory.html?utm_source=chatgpt.com "チャットメモリ :: Spring AI リファレンス"))

生产项目里，一般不要只用内存版，因为服务重启、扩容、多实例都会丢上下文或上下文不一致。

---

## 3. 接入 ChatClient：Advisor

Spring AI 不是让你手动每次查历史、拼 prompt，而是通过 **Advisor** 挂到 `ChatClient` 上。

常见有三个：

|Advisor|注入方式|适合场景|
|---|---|---|
|`MessageChatMemoryAdvisor`|把历史作为 message 列表加入请求|标准聊天、多轮对话|
|`PromptChatMemoryAdvisor`|把历史拼进 system prompt 文本|模型不太支持结构化 message 历史时|
|`VectorStoreChatMemoryAdvisor`|从向量库检索相关记忆，拼进 system text|长期记忆、语义召回|

官方 Advisors 文档明确列出这些 Chat Memory Advisors：`MessageChatMemoryAdvisor`、`PromptChatMemoryAdvisor`、`VectorStoreChatMemoryAdvisor`。其中 `MessageChatMemoryAdvisor` 会把历史作为 message 集合加入 prompt；`PromptChatMemoryAdvisor` 会把记忆合并进 system text；`VectorStoreChatMemoryAdvisor` 会从 VectorStore 检索 memory。([Home](https://docs.spring.io/spring-ai/reference/api/advisors.html?utm_source=chatgpt.com "Advisors API :: Spring AI Reference"))

示意代码：

```java
@Bean
ChatClient chatClient(ChatClient.Builder builder, ChatMemory chatMemory) {
    return builder
            .defaultAdvisors(
                    MessageChatMemoryAdvisor.builder(chatMemory).build()
            )
            .build();
}
```

调用时一般需要传入 conversation id：

```java
String answer = chatClient.prompt()
        .user("我叫张三，后面请记住")
        .advisors(advisor -> advisor.param(
                ChatMemory.CONVERSATION_ID,
                "user-123-session-001"
        ))
        .call()
        .content();
```

后续同一个 `conversationId` 的请求，就能拿到对应历史。

---

## 4. 工程上怎么选？

### Demo / 学习项目

用：

```text
MessageWindowChatMemory
+ InMemoryChatMemoryRepository
+ MessageChatMemoryAdvisor
```

特点：简单，但不持久化。

---

### 普通生产聊天助手

用：

```text
MessageWindowChatMemory
+ JdbcChatMemoryRepository
+ MessageChatMemoryAdvisor
```

适合：

- 客服助手
    
- 内部知识库问答
    
- AI 编程助手
    
- 普通多轮聊天
    

这是 Java 后端项目里最稳妥的第一版。

---

### Agent / 长期用户画像记忆

不要只靠 `MessageWindowChatMemory`。

更合理的是分两类 memory：

|类型|说明|技术实现|
|---|---|---|
|短期记忆|最近几轮对话上下文|`MessageWindowChatMemory` + JDBC/Redis|
|长期记忆|用户偏好、事实、历史任务、项目背景|向量库 / 结构化表 / KV / Redis|

例如：

```text
短期 memory:
最近 20 条聊天消息

长期 memory:
- 用户常用 Java + Spring Boot
- 用户正在做 RAG 项目
- 用户偏好中文技术文章
- 用户项目使用 PostgreSQL
```

长期记忆更像 **用户画像 + 语义检索 + 事实库**，不能简单等同于聊天历史。

---

## 5. 阿里系后端视角：不要把 memory 理解成“数据库里存聊天记录”

更准确的工程理解是：

```text
Memory = 会话上下文管理 + 存储策略 + 检索策略 + 注入策略 + 隔离策略
```

生产级 memory 至少要考虑：

|问题|说明|
|---|---|
|conversationId 设计|按用户、租户、业务场景、会话拆分|
|token 成本|不能无限塞历史，需要窗口、摘要、裁剪|
|多实例一致性|内存版不适合分布式部署|
|数据隔离|userId、tenantId、conversationId 必须严格隔离|
|敏感信息|需要脱敏、权限控制、可删除|
|过期策略|临时会话和长期记忆要分开|
|质量控制|不是所有内容都该记住|
|RAG 区分|memory 是用户/会话上下文，RAG 是外部知识检索|

---

## 6. 推荐实现组合

你如果做 **Java + AI 应用 / RAG / Agent 项目**，建议这样设计：

```text
第一阶段：
Spring AI ChatClient
+ MessageWindowChatMemory
+ JdbcChatMemoryRepository
+ MessageChatMemoryAdvisor

第二阶段：
增加 conversation 表、message 表、user_session 表
把 Spring AI memory 和业务会话体系打通

第三阶段：
增加长期记忆：
- 用户偏好表
- 项目上下文表
- 向量库 memory collection
- 摘要压缩任务

第四阶段：
做 memory governance：
- 可查看
- 可删除
- 可过期
- 可审计
- 可禁用
```

---

## 一句话总结

**Spring AI 的 memory 本质上是用 `ChatMemory` 管理对话历史，用 `ChatMemoryRepository` 存储历史，再通过 `ChatMemoryAdvisor` 自动把历史注入 `ChatClient` 请求。**

面试或项目里可以这样说：

> Spring AI 的 memory 不是模型真的“记住了”，而是在应用层维护 conversationId 对应的消息历史，并在每次调用模型前通过 Advisor 把相关历史重新注入上下文。短期记忆通常用 `MessageWindowChatMemory`，存储层可用 `InMemoryChatMemoryRepository` 或 `JdbcChatMemoryRepository`；生产环境一般选择 JDBC 或自定义 Redis/数据库实现，并结合窗口裁剪、摘要压缩、用户隔离和长期向量记忆。