- LangChain就是一条链, 比如它会把prompt, 模型, 解析器, 检索器串成一条
- 而LangGraph则是一个流程图, 通过定义state, node, edge得到.
# LangChain

==官方定位==: 用于构建 `agents` 和 `LLM` 应用的框架
- LangChain就是一个框架, 它适合**线性任务**. 比如你想做一个AI应用, 你就需要自己做: 
  `用户提问->改写问题->调用大模型->检索资料->大模型组织答案->返回结果`
  每做一次都需要**重复**写一次该流程
- 而有了LangChain, 它帮你把这些常见的重复步骤封装为了一个个的组件, 让你可以按需调用, 把它们像搭积木一样一个一个串起来

# LangGraph
- 处理有分支, 有状态, 有循环的AI工作流
## 核心
- 就三个: 
	- State: 状态
		- ```
		  {
		    "messages": [],          # 对话历史
		    "question": "我的订单在哪？",
		    "intent": "查订单",
		    "order_id": "123456",
		    "answer": ""
		  }
		  ```
	- Node: 节点
	- Edge: 边
![](assets/第一章%20LangChain与LangGraph框架认知/file-20260616150639160.png)
 