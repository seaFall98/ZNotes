
还是围绕自己的项目是怎么做的

deepagent是怎么做的 offload，summarize内置+自定义中间件，memoryupdate中间件和/userid/memory.md(可以提 关键信息结构化抽取存到关系型数据库 但是md文件非常灵活)

memory.md可以就存到用户本机，云服务才需要存mongodb

注意滑动窗口这种老方案 注意信息时效性

---

prompt caching 可以提

记忆压缩在「信息层」工作，决定哪些内容值得被保留在对话历史里；PromptCaching在「计算层」工作，对已经决定要带进去的内容减少重复计算的开销。两者解决的不是同一个问题，可以同时使用，是互补关系，不是替代。

systemprompt加上长期记忆注入的部分在多轮对话中基本不变，天然适合被缓存，收益非常可观。

deepseek v4 当时缓存命中超高 大幅降低成本 现在各家都在跟上了

在ai gateway比如sub2api 有专门的prompt caching处理和统计

gpt 5.6之后缓存创建和读取要收费了，虽然宣称模型本身5.6比5.5便宜，但是总花费还是比5.5高，体现了记忆压缩和prompt caching乃至上下文工程的重要性和服务端存储的压力

---

