
**文档中分块的核心原则：结构优先，保留原始语义，拒绝固定 token 数量限制。**

**第一点：我们根据内容类型动态调整。**  
FAQ类 → 一问一答作为一个chunk；  
政策文档类 → 每条条文为单位作为一个chunk；  
有章节或者标题的内容：以章节或者标题来主导 分块。

**第二点：在我之前做的项目中，都是一些产品和技术类的数据，有严谨的章节和标题层级。**我采用递归切片法 + 特殊模态分块，首先根据章节或者标题的层级递归分块，然后，判断分块的大小是否超过阈值，如果超过，则继续采用语义切割。langchain提供的semanticchunker类实现语义切割。需要指定一个嵌入模型，如BGE large 1.5。来计算相邻 符号间隔分块的语义相似度。比固定长度或字符分割更好地保持语义连贯性。断点阈值计算方法有：分位法、标准差、四分位、梯度法四种。我采用的是percentile(分位法)。因为它最稳定的方法。

还有一些表格，图片和公式数据，需要单独处理，组成一个独立的分块。比如：表格一般通过OCR模型转换成html标签数据或者json，

```python
# 初始化嵌入模型和语义分割器
embeddings = OpenAIEmbeddings()

text_splitter = SemanticChunker(
    embeddings=embeddings,
    breakpoint_threshold_type="percentile", # 使用百分位法
    breakpoint_threshold_amount=85,         # 将阈值设置在85%分位数
    sentence_split_regex=r'(?<=[。？！\n])', # 针对中文标点进行句子切分
    buffer_size=1,                          # 计算当前句子嵌入时，组合前后各1个句子
    min_chunk_size=50                       # 过滤掉字符数小于50的碎片化块
)
```

---
[ RAG文本分块策略全解析：七种主流方案评测与选型指南](https://developer.baidu.com/article/detail.html?id=8025647)