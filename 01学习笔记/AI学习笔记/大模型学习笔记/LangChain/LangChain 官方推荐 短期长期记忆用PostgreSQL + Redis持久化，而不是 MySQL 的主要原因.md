## LangChain 官方推荐 PostgreSQL + Redis，而不是 MySQL 的主要原因

1. **Agent Memory 以 JSON State 为核心，而不是关系型表结构，PostgreSQL 的 `jsonb` 更适合存储和查询复杂状态。**
2. **Checkpoint 写入频繁，PostgreSQL 在事务、一致性和高频更新场景表现更成熟。**
3. **LangGraph 支持 Interrupt/Resume，需要可靠的状态快照和恢复能力，PostgreSQL 更契合这种工作负载。**
4. **Agent State 会随着对话不断增长，PostgreSQL 对大 JSON 文档的存储和操作能力更强。**
5. **长期记忆通常需要语义检索，PostgreSQL 可以通过 `pgvector` 无缝支持向量数据库能力。**
6. **Redis 专注于缓存（Session、Prompt、Embedding、Tool Cache），而不是持久化 Memory。**
7. **MySQL 完全可以作为 Memory 存储，但需要自行实现对应的 Checkpointer/Store，官方没有提供成熟适配器。**
8. **官方选择 PostgreSQL 更多是生态和功能取向，并非 MySQL 无法胜任，而是 PostgreSQL 更符合 Agent Memory 的设计需求。**