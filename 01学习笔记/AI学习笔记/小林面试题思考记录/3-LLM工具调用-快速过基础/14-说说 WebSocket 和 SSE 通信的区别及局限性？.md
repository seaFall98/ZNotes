
我觉得最核心的区别是通信方向：SSE是服务端单向推，客户端只能接收，想发消息只能另起一个HTTP请求；WebSocket是全双工，双方都可以随时主动发消息。

对于LLM流式输出这种「模型一直在推token、用户只是看」的场景，SSE完全够用，而且轻量、HTTP原生支持、运维简单，OpenAl和Anthropic的APl用的都是SSE。

WebSocket的复杂性只有在真正需要双向实时交互的时候才值得引入，比如用户要在模型说话过程中随时打断。

两者各有局限：

WebSocket 的优势是双向性和支持二进制数据，但连接管理、心跳、重连、负载均衡和扩展成本更高。
SSE 的优势是实现简单、天然兼容 HTTP、浏览器支持自动重连，但只能服务端到客户端、主要传输文本，并且受浏览器连接数、代理超时和 HTTP 长连接配置影响。


---

现代AI应用的图文回复流程，在工程上做了很巧妙的设计，而不是“硬传”图片文件。根据一些技术文档对ChatGPT上游通信协议的分析，它生成图片的流程大概是这样的[](https://github.com/basketikun/chatgpt2api/blob/main/docs/upstream-sse-conversation.md)：

1. **先发文字指令**：ChatGPT服务器通过SSE协议，给你发送一串描述图片信息的**文本数据**（比如一个图片的ID或访问链接）。
    
2. **客户端负责展示**：你的浏览器或App收到这个“文本链接”后，会**单独发起一个新的HTTP请求**去下载这张图片，然后把它显示在聊天界面上[](https://github.com/basketikun/chatgpt2api/blob/main/docs/upstream-sse-conversation.md)。
    

所以，从你的视角看，就是收到了“文字+图片”；但从技术底层看，服务器通过SSE推送的始终是文本。这种方式完美绕过了SSE无法传输二进制数据的限制。

---
