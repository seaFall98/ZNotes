**ChatGPT 里你看到的是一个“产品入口”，但 OpenAI API 背后是一套模型矩阵**。

[[大模型的分类]]
你可以把 OpenAI 的大模型体系理解成下面这张图：

```text
OpenAI 模型体系
├── GPT 主模型：负责文本、多模态理解、推理、代码、Agent
├── Realtime / Audio：负责实时语音、语音转写、语音输出
├── Embedding：负责向量化、语义检索、RAG
├── Image：负责图片生成与编辑
├── Video：负责视频生成
├── Moderation / Safety：负责内容安全审核
├── Tool / Agent 能力：函数调用、文件搜索、Web 搜索、代码解释器、MCP
└── Fine-tuning / Batch / Responses API：负责工程化调用方式
```

下面按工程视角展开。

---

# 1. GPT 主模型：OpenAI 的“通用大脑”

你说的 **GPT-5.5** 属于这个层级。

OpenAI 官方模型页把 GPT-5.5 定位为面向复杂专业工作的前沿模型，支持推理能力，并且有很大的上下文窗口；GPT-5.5 Pro 则是使用更多计算资源、面向更难问题的版本。([OpenAI开发者](https://developers.openai.com/api/docs/models/gpt-5.5?utm_source=chatgpt.com "GPT-5.5 Model | OpenAI API"))

## 主要模型层级

|模型|定位|适合场景|
|---|---|---|
|**GPT-5.5**|当前前沿通用模型|复杂问答、代码、架构分析、Agent、专业写作|
|**GPT-5.5 Pro**|更强推理、更高成本|高难度研究、复杂代码审查、深度分析|
|**GPT-5.x / GPT-5 mini**|上一代或轻量版本|成本敏感、高并发、低延迟任务|
|**GPT-5-Codex**|偏代码/Agentic Coding|代码生成、仓库理解、自动修复|

你可以把它理解成：

```text
GPT-5.5
= OpenAI 当前的通用高级大脑

GPT-5.5 Pro
= 更慢、更贵、更适合疑难复杂任务的大脑

GPT-5 mini / 轻量模型
= 更便宜、更快，适合批量简单任务
```

---

# 2. GPT 不只是“聊天模型”，它已经是多模态模型

很多人以为 GPT 只会处理文本，但现在的 GPT 主模型通常可以处理：

|输入|是否常见|
|---|---|
|文本|是|
|图片|是|
|文件|是，通过文件/工具能力|
|音频|部分模型支持|
|视频|依具体模型/接口能力而定|
|工具结果|是|

OpenAI 的 GPT-5.5 Pro 模型页明确列出：文本支持输入和输出，图像支持输入；并支持函数调用、结构化输出、Web search、File search、Image generation、Code interpreter、MCP 等工具能力。([OpenAI开发者](https://developers.openai.com/api/docs/models/gpt-5.5-pro?utm_source=chatgpt.com "GPT-5.5 Pro Model | OpenAI API"))

所以你可以这样理解：

```text
早期 GPT：
文本输入 → 文本输出

现在 GPT：
文本 / 图片 / 文件 / 工具结果
        ↓
推理、规划、调用工具
        ↓
文本答案 / 结构化 JSON / 工具动作 / 图片生成请求
```

---

# 3. 推理模型：GPT 主模型里的“深度思考模式”

OpenAI 现在不是简单区分“GPT”和“非 GPT”，而是很多 GPT 模型带有 **reasoning effort** 参数。

比如官方模型页显示，GPT-5.5 支持 `none、low、medium、high、xhigh` 等推理强度；GPT-5.5 Pro 支持更高强度的推理设置。([OpenAI开发者](https://developers.openai.com/api/docs/models/gpt-5.5?utm_source=chatgpt.com "GPT-5.5 Model | OpenAI API"))

## 这是什么意思？

同一个模型，可以根据任务调整“思考预算”：

|reasoning effort|适合场景|
|---|---|
|none / low|简单问答、分类、改写|
|medium|普通代码、技术解释、业务分析|
|high|复杂 Debug、架构设计、数学推理|
|xhigh|高难研究、复杂规划、多约束问题|

类比一下：

```text
同一个人回答问题：

low：快速回答
medium：认真分析
high：拿纸笔推演
xhigh：做深度研究和反复校验
```

所以 **GPT-5.5 不是一个固定“聊天机器人”**，它可以根据任务变成快速助手、推理模型、代码分析模型或 Agent 控制器。

---

# 4. 代码模型：Codex 不是另一个聊天产品，而是代码专项能力

OpenAI 体系里还有偏代码的模型，例如 **GPT-5-Codex**。官方模型页显示 GPT-5-Codex 支持图像输入、文本输入输出，并出现在专门模型列表中。([OpenAI开发者](https://developers.openai.com/api/docs/models/gpt-5-codex?utm_source=chatgpt.com "GPT-5-Codex Model | OpenAI API"))

你平时用的 **Codex / Codex 插件 / AI Coding** 可以理解为：

```text
强代码模型
+ 仓库文件读取
+ 终端执行
+ Git diff
+ 测试运行
+ 工具调用
+ 任务规划
= Agentic Coding 系统
```

不是只有一个模型在“聊天”，而是一个工程系统在做：

|能力|例子|
|---|---|
|读代码|分析项目结构|
|改代码|修改 Controller、Service、前端组件|
|跑命令|`mvn test`、`npm run build`|
|看报错|根据日志修复|
|做 diff|生成补丁|
|代码审查|找 bug、找坏味道、找未实现接口|

---

# 5. Embedding 模型：RAG 检索用的“向量化模型”

这类模型不是 GPT-5.5，也不是用来聊天的。

它的职责是：

```text
文本 → 向量
```

例如：

```text
"Redis 如何解决缓存穿透？"
        ↓
[0.021, -0.392, 0.884, ...]
```

OpenAI 官方文档把 embeddings 定义为把文本转换成数字向量，用于搜索、聚类、推荐、异常检测、分类等任务。([OpenAI开发者](https://developers.openai.com/api/docs/models/all?utm_source=chatgpt.com "All models | OpenAI API"))

## 在 RAG 中的位置

```text
用户问题
  ↓
OpenAI Embedding 模型
  ↓
向量库检索：Milvus / pgvector / Elasticsearch / Pinecone
  ↓
召回相关文档
  ↓
GPT-5.5 生成答案
```

所以一个 OpenAI RAG 应用通常不是只用 GPT-5.5，而是：

```text
Embedding 模型负责“找资料”
GPT-5.5 负责“读资料并回答”
```

---

# 6. Realtime / Audio 模型：语音助手体系

OpenAI 不是只有文本模型，还有一组语音模型。

官方模型列表里有 **GPT-Realtime-2、GPT-Realtime-Translate、GPT-Realtime-Whisper、GPT-4o Transcribe、TTS-1、Whisper** 等实时、语音和音频工作流模型。([OpenAI开发者](https://developers.openai.com/api/docs/models/all?utm_source=chatgpt.com "All models | OpenAI API"))

## 可以分成几类

|类型|OpenAI 例子|用途|
|---|---|---|
|实时语音对话|GPT-Realtime-2|语音进、语音出，做实时助手|
|实时翻译|GPT-Realtime-Translate|语音到语音翻译|
|语音转文本|GPT-Realtime-Whisper、GPT-4o Transcribe、Whisper|会议转写、字幕、语音输入|
|文本转语音|TTS-1、TTS-1 HD|播报、客服、语音回复|
|说话人识别/分离|GPT-4o Transcribe Diarize|区分谁在说话|

GPT-Realtime-2 官方说明是面向实时语音交互的推理模型，支持 speech-to-speech、可配置 reasoning effort、更强指令遵循和更可靠的工具使用。([OpenAI开发者](https://developers.openai.com/api/docs/models/gpt-realtime-2?utm_source=chatgpt.com "GPT-Realtime-2 Model | OpenAI API"))

## 语音 Agent 的真实链路

```text
用户说话
  ↓
Realtime / Transcribe 模型
  ↓
GPT 理解意图
  ↓
调用工具：查订单 / 查日程 / 查数据库
  ↓
GPT 组织答案
  ↓
TTS / Realtime 语音输出
```

所以语音模型不是“GPT 的一个小功能”，而是一整套实时交互模型。

---

# 7. Image 模型：图片生成和图片编辑

OpenAI 也有专门的图像生成模型。

官方模型页显示 **GPT Image 2** 属于图像生成模型；GPT Image 1 是原生多模态语言模型，接受文本和图片输入，生成图片输出；DALL·E 3 则是上一代图像生成模型。([OpenAI开发者](https://developers.openai.com/api/docs/models/gpt-image-2?utm_source=chatgpt.com "GPT Image 2 Model | OpenAI API"))

## 图像模型能做什么？

|能力|例子|
|---|---|
|文生图|“生成一张科技感产品海报”|
|图生图|上传图片，让模型改风格|
|局部编辑|换背景、移除物体、改服装|
|设计辅助|生成 Logo、Banner、插画|
|商品图|电商主图、广告图|

它和 GPT-5.5 的关系可以理解成：

```text
GPT-5.5：负责理解需求、写 prompt、规划图像内容
GPT Image：负责真正生成图片
```

---

# 8. Video 模型：视频生成

OpenAI API 模型页已经把 **Videos / v1/videos** 作为模型端点之一列出。([OpenAI开发者](https://developers.openai.com/api/docs/models/gpt-image-2?utm_source=chatgpt.com "GPT Image 2 Model | OpenAI API"))

这一类模型面向：

|输入|输出|
|---|---|
|文本 prompt|视频|
|图片|动态视频|
|视频片段|编辑后视频|

实际产品上你可以理解为 Sora / video generation 这一类能力。

工程场景：

```text
产品宣传视频
教学动画
短视频素材
概念片
广告分镜
图生视频
```

---

# 9. Moderation / Safety 模型：安全审核模型

这类模型经常被忽视，但生产系统里很重要。

OpenAI API 模型能力列表中包含 `v1/moderations` 端点，用于安全审核相关能力。([OpenAI开发者](https://developers.openai.com/api/docs/models/gpt-5.5-pro?utm_source=chatgpt.com "GPT-5.5 Pro Model | OpenAI API"))

## 它不是负责回答问题，而是负责判断风险

|检查对象|例子|
|---|---|
|用户输入|是否包含违法、暴力、自伤、仇恨等内容|
|模型输出|是否生成不安全内容|
|文件内容|是否包含敏感或违规内容|
|工具调用前|是否允许执行这个动作|

典型链路：

```text
用户输入
  ↓
Moderation / Guardrail
  ↓
安全则进入 GPT
  ↓
GPT 输出
  ↓
再次安全检查
  ↓
返回用户
```

生产系统里，尤其是客服、教育、医疗、金融、未成年人产品，这一层很关键。

---

# 10. Tool / Agent 能力：OpenAI 不只是模型，还有工具系统

这是最容易被忽略的一点。

现在 OpenAI API 的核心不只是：

```text
prompt → model → answer
```

而是：

```text
用户任务
  ↓
GPT 理解和规划
  ↓
调用工具
  ↓
读取工具结果
  ↓
继续推理
  ↓
返回最终结果
```

GPT-5.5 Pro 官方模型页列出的工具能力包括 Web search、File search、Image generation、Code interpreter、Hosted shell、MCP 等；同时支持 function calling 和 structured outputs。([OpenAI开发者](https://developers.openai.com/api/docs/models/gpt-5.5-pro?utm_source=chatgpt.com "GPT-5.5 Pro Model | OpenAI API"))

## 常见工具能力

|能力|作用|
|---|---|
|Function Calling|调用你自己定义的函数/API|
|Structured Outputs|稳定输出 JSON / schema|
|File Search|在文件知识库中检索|
|Web Search|查网页|
|Code Interpreter|执行代码、分析数据|
|Image Generation|调图像模型生成图片|
|MCP|接入外部工具生态|
|Hosted Shell|执行受控 shell 任务|

这就是为什么现在说：

> GPT 不只是“生成文本的模型”，它越来越像一个“任务控制器”。

---

# 11. Fine-tuning / Batch / Responses API：工程化调用方式

这不是模型类别，但它决定你怎么用模型。

## 主要工程接口

|能力|作用|
|---|---|
|Responses API|新一代统一调用入口|
|Chat Completions|传统聊天接口|
|Batch API|批量离线处理，成本更低|
|Fine-tuning|对特定任务做微调|
|Realtime API|实时语音/音频交互|
|Embeddings API|向量化|
|Images API|图片生成/编辑|
|Audio API|语音转写/合成|
|Moderation API|安全审核|

所以你做项目时，不是“调用 GPT-5.5 就完了”，而是要选：

```text
我是在做聊天？
做 RAG？
做语音助手？
做图片生成？
做批量数据处理？
做安全审核？
做 Agent 工具调用？
```

不同场景选不同 API 和模型。

---

# 一张表看懂 OpenAI 模型体系

|大类|代表模型 / 能力|作用|
|---|---|---|
|GPT 主模型|GPT-5.5、GPT-5.5 Pro、GPT-5 mini|通用问答、推理、代码、Agent|
|代码模型|GPT-5-Codex|仓库级代码生成、修复、审查|
|实时语音模型|GPT-Realtime-2|实时语音对话、语音 Agent|
|语音转文本|Whisper、GPT-4o Transcribe、Realtime Whisper|ASR、会议转写、字幕|
|文本转语音|TTS-1、TTS-1 HD|播报、语音回复|
|Embedding|OpenAI Embedding 模型|向量化、RAG、语义检索|
|图像生成|GPT Image 2、GPT Image 1、DALL·E 3|文生图、图像编辑|
|视频生成|Video generation / v1/videos|文生视频、图生视频|
|安全审核|Moderation|内容安全、风险判断|
|工具系统|Function Calling、File Search、Web Search、Code Interpreter、MCP|让模型接入真实业务系统|
|工程接口|Responses、Batch、Fine-tuning、Realtime|不同调用方式|

---

# 用一个 RAG 项目理解 OpenAI 的模型组合

假设你要做一个企业知识库问答系统。

不是只用 GPT-5.5，而是这样：

```text
用户上传 PDF / Word
  ↓
文件解析 / OCR / 文档切分
  ↓
Embedding 模型向量化
  ↓
向量库保存
  ↓
用户提问
  ↓
Embedding 模型把问题向量化
  ↓
File Search / 向量检索找资料
  ↓
GPT-5.5 阅读资料并生成答案
  ↓
Moderation / Evaluator 检查安全和质量
```

其中：

|环节|用什么|
|---|---|
|最终回答|GPT-5.5|
|复杂推理|GPT-5.5 Pro|
|语义检索|Embedding 模型|
|文件检索|File Search|
|安全审核|Moderation|
|结构化输出|Structured Outputs|
|外部业务接口|Function Calling / MCP|

---

# 用一个语音 Agent 理解 OpenAI 的模型组合

例如你做一个“语音客服”。

```text
用户说：帮我查一下订单 123 的物流
  ↓
GPT-Realtime-2 接收语音
  ↓
模型理解意图
  ↓
Function Calling 调用订单 API
  ↓
拿到物流状态
  ↓
GPT 生成自然语言回复
  ↓
语音输出给用户
```

这里涉及：

|环节|模型/能力|
|---|---|
|语音输入|Realtime / Transcribe|
|意图理解|GPT|
|工具调用|Function Calling|
|业务查询|你的订单系统 API|
|语音输出|Realtime / TTS|

这已经不是单纯“聊天模型”了，而是 **语音 + GPT + 工具调用 + 业务系统**。

---

# 用一句话概括 OpenAI 体系

> OpenAI 不是只有一个 GPT-5.5，而是有一个以 GPT 为核心的模型矩阵：GPT 负责理解、推理、生成和工具调用；Embedding 负责语义检索；Realtime/Audio 负责语音交互；Image/Video 负责内容生成；Moderation 负责安全；Responses API、Function Calling、File Search、Code Interpreter、MCP 等负责把模型接入真实工程系统。

---

# 你现在应该怎么记

不用一开始记所有模型名。先记这 6 类就够了：

|你要记的类别|代表作用|
|---|---|
|**GPT-5.5**|通用大脑|
|**GPT-5.5 Pro**|更强推理|
|**Embedding**|RAG 检索向量化|
|**Realtime / Audio**|语音助手|
|**Image / Video**|图像、视频生成|
|**Moderation + Tools**|安全和业务接入|

最核心的工程认知是：

```text
ChatGPT 是产品。
GPT-5.5 是核心模型。
OpenAI API 是一整套模型与工具平台。
真实应用通常不是只调用一个 GPT，而是组合多个模型和工具能力。
```