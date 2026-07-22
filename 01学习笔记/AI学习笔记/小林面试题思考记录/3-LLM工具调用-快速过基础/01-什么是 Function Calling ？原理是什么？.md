https://xiaolinnote.com/ai/tools/1_function_calling.html

面试回答这道题，有几个点必须说到：工具定义用JSON schema描述，description字段是模型判断是否调用的核心依据；运行时是「两轮对话+中间执行」的闭环流程；模型通过finish_reason字段为toolcalls来明确告知需要工具帮助；以及模型支持一次返回多个toolcalls实现并行调用，也能判断工具调用依赖关系进行串行调用。

把这几个点讲清楚，再强调模型决策、代码执行」的分工原则，这道题就稳了。