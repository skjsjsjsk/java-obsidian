[第三章 LangChain进阶组件实操 | Easy-langent](https://easy-langent.datawhale.cc/guide/chapter3.html)

# 这一章讲了什么

第二章主要讲的是 LangChain 的基础调用链：

```
Prompt -> ChatOpenAI -> OutputParser
```

也就是先把用户输入组织成提示词，再交给大模型生成结果，最后用解析器把结果变成程序能处理的数据。

第三章开始进入更接近真实应用的部分，重点围绕两类能力展开：

1. **状态管理**：让模型记住历史对话，也就是 `Memory`。
2. **外部行动**：让模型调用外部工具，也就是 `Tools`。

这一章的核心目标是：从“单轮问答”升级到“带记忆、能调用工具、能完成任务的智能应用”。

本章涉及的核心组件包括：

- `ChatPromptTemplate`
- `MessagesPlaceholder`
- `BaseChatMessageHistory`
- `InMemoryChatMessageHistory`
- `RunnableWithMessageHistory`
- `RunnablePassthrough`
- `RunnableLambda`
- `@tool`
- `create_agent`
- `PythonREPLTool`
- `llm.bind_tools`
- `AIMessage`
- `ToolMessage`

---

# 1. Memory：让模型拥有记忆能力

LLM 原生是无状态的。

也就是说，每一次调用模型时，它只知道这一次请求里的内容，不会自动记住之前用户说过什么。

Memory 组件就是为了解决这个问题：把历史对话保存下来，并在下一轮对话时重新注入到 Prompt 中。

## 用途

Memory 主要解决多轮对话中的上下文问题：

- 记住用户身份，比如“我叫小明”。
- 记住用户偏好，比如“我喜欢编程”。
- 记住任务上下文，比如“刚才说的那个组件”。
- 支撑连续任务，比如先收集需求，再生成方案，再继续修改。

## 好处

有了 Memory，模型不再像每轮都重新开始：

- 对话更连续。
- 用户不用重复描述背景。
- 应用更像真正的助手。
- 可以支持客服、学习助手、文件助手、个人助理等长流程场景。

## 核心机制

Memory 的工作逻辑可以拆成两步：

1. **Save**：把用户输入和模型回复保存成历史消息。
2. **Load**：下一轮对话前，把历史消息取出来放进 Prompt。

在本章代码中，历史消息主要通过下面几个组件完成：

```
InMemoryChatMessageHistory
RunnableWithMessageHistory
MessagesPlaceholder
session_id
```

大致调用链是：

```
用户输入 -> 读取历史消息 -> 拼接 Prompt -> 调用 LLM -> 保存本轮消息
```

## 难点

Memory 的难点不在“保存历史”本身，而在于如何控制历史的长度和质量。

如果历史太少，模型会丢上下文；如果历史太多，Token 成本会越来越高，速度也会变慢。

所以本章重点讲了三种基础记忆模式：全量记忆、窗口记忆、摘要记忆。

---

# 2. 全量记忆：InMemoryChatMessageHistory

全量记忆就是完整保存所有历史对话。

本章示例中用一个字典保存不同会话的历史：

```
full_memory_store = {}

def get_full_memory_history(session_id: str):
    if session_id not in full_memory_store:
        full_memory_store[session_id] = InMemoryChatMessageHistory()
    return full_memory_store[session_id]
```

然后通过 `RunnableWithMessageHistory` 包装原始链：

```
full_memory_chain = RunnableWithMessageHistory(
    runnable=base_chain,
    get_session_history=get_full_memory_history,
    input_messages_key="user_input",
    history_messages_key="chat_history"
)
```

## 用途

全量记忆适合短对话，或者历史信息都很重要的场景。

比如：

- 用户刚刚说了自己的名字。
- 用户刚刚表达了偏好。
- 对话轮次不多，但每轮都和后续回答有关。

## 好处

全量记忆最大的优点是信息完整。

只要对话历史还在，模型就能看到之前所有内容，不容易因为截断而忘掉早期信息。

## 大概如何使用

一般步骤是：

1. 用 `ChatPromptTemplate` 定义对话模板。
2. 用 `MessagesPlaceholder(variable_name="chat_history")` 给历史消息留位置。
3. 用 `InMemoryChatMessageHistory` 保存消息。
4. 用 `RunnableWithMessageHistory` 自动处理历史注入和保存。
5. 调用时传入 `session_id`，区分不同用户或不同会话。

调用时的关键配置是：

```
config = {"configurable": {"session_id": "user_001"}}
```

`session_id` 很重要。  
它相当于一个聊天窗口编号，不同 `session_id` 的记忆互不影响。

## 难点

全量记忆的问题也很明显：对话越长，历史消息越多。

这会带来几个问题：

- Token 消耗越来越大。
- 请求速度变慢。
- 上下文超过模型限制后无法继续完整塞入。
- 无关历史太多时，模型反而可能被干扰。

所以全量记忆适合短对话，不适合长期对话。

---

# 3. 窗口记忆：只保留最近 N 轮

窗口记忆是在全量记忆基础上做截断：只保留最近几轮对话。

本章示例中设置：

```
WINDOW_SIZE = 2
```

表示只保留最近 2 轮对话。  
因为一轮对话通常包含一条用户消息和一条 AI 消息，所以实际保留的是最近 4 条消息：

```
history.messages = history.messages[-2 * WINDOW_SIZE:]
```

## 用途

窗口记忆适合中长对话，尤其是只关心最近上下文的场景。

比如：

- 闲聊助手。
- 学习问答。
- 临时任务协作。
- 只需要围绕最近问题继续追问的场景。

## 好处

窗口记忆比全量记忆更可控：

- Token 消耗稳定。
- 不会因为对话变长无限膨胀。
- 实现简单。
- 适合大多数普通多轮对话。

## 大概如何使用

窗口记忆和全量记忆的主体结构差不多，只是在获取历史时多了一步截断：

```
def get_window_memory_history(session_id: str):
    if session_id not in window_memory_store:
        window_memory_store[session_id] = InMemoryChatMessageHistory()

    history = window_memory_store[session_id]
    if len(history.messages) > 2 * WINDOW_SIZE:
        history.messages = history.messages[-2 * WINDOW_SIZE:]
    return history
```

然后同样交给 `RunnableWithMessageHistory`：

```
window_memory_chain = RunnableWithMessageHistory(
    runnable=window_base_chain,
    get_session_history=get_window_memory_history,
    input_messages_key="user_input",
    history_messages_key="chat_history"
)
```

## 难点

窗口记忆的核心难点是窗口大小怎么定。

如果 `WINDOW_SIZE` 太小，模型容易忘掉关键背景。  
比如第一轮用户说“我叫小红”，第六轮再问“我叫什么名字”，如果这条信息已经被窗口截掉，模型就回答不上来。

如果 `WINDOW_SIZE` 太大，又会接近全量记忆的问题，Token 成本重新变高。

所以窗口记忆本质是在做平衡：

```
上下文完整度 <-> Token 成本
```

---

# 4. 摘要记忆：用摘要替代完整历史

摘要记忆不是直接把完整历史塞给模型，而是先让 LLM 把历史对话压缩成摘要，再把摘要注入到 Prompt 中。

本章示例中先定义摘要链：

```
summary_prompt = ChatPromptTemplate.from_messages([
    ("system", "你是对话摘要助手，需简洁总结以下对话的核心信息..."),
    ("human", "对话历史：{chat_history_text}\n请生成摘要：")
])

summary_chain = summary_prompt | llm
```

然后在正式对话时，把历史消息转成文本并生成摘要：

```
chat_summary=lambda x: summary_chain.invoke(
    {
        "chat_history_text": "\n".join(
            [f"{msg.type}: {msg.content}" for msg in x["chat_history"]]
        )
    }
).content
```

## 用途

摘要记忆适合长对话，尤其是需要保留核心信息但不需要逐字保留全部历史的场景。

比如：

- 长期学习助手。
- 产品需求讨论。
- 咨询类对话。
- 多轮方案迭代。

## 好处

摘要记忆最大的好处是压缩上下文。

它不会把所有历史原文都放进 Prompt，而是只放一段摘要：

```
对话摘要：小李，产品经理，负责电商APP迭代，优化下单流程时遇用户流失率高问题，寻求建议。
```

这样既能保留关键背景，又能减少 Token 消耗。

## 大概如何使用

摘要记忆通常分成两条链：

1. 摘要链：负责把历史对话压缩成摘要。
2. 对话链：负责基于摘要和用户新输入生成回答。

整体流程是：

```
完整历史 -> 生成摘要 -> 摘要注入 Prompt -> LLM 回答 -> 保存新一轮历史
```

本章里用到了 `RunnablePassthrough.assign()`，它的作用是给输入动态增加一个字段：

```
chat_summary
```

然后 Prompt 中就可以使用：

```
("system", "对话摘要：{chat_summary}")
```

## 难点

摘要记忆的难点是摘要质量。

如果摘要写得太短，可能丢掉关键信息；如果摘要写得太长，又失去了压缩的意义。

另外，摘要是有损压缩，尤其容易丢：

- 数字。
- 人名。
- 具体约束。
- 用户语气。
- 早期但重要的决策。

所以摘要 Prompt 要写清楚，告诉模型保留什么。

例如本章要求摘要包含：

- 用户身份。
- 用户偏好。
- 关键问题。
- 核心上下文。

---

# 5. 三种 Memory 的选择

这三种记忆模式可以这样理解：

| 记忆方式 | 核心做法 | 优点 | 缺点 | 适合场景 |
|---|---|---|---|---|
| 全量记忆 | 保存所有历史 | 信息最完整 | Token 成本高 | 短对话 |
| 窗口记忆 | 只保留最近 N 轮 | 成本稳定 | 会忘早期信息 | 中长对话 |
| 摘要记忆 | 用摘要压缩历史 | 适合长对话 | 摘要可能丢细节 | 长期任务 |

简单选择原则：

- 对话短，选全量记忆。
- 对话较长，但只关心最近上下文，选窗口记忆。
- 对话很长，而且需要保留长期背景，选摘要记忆。

---

# 6. Tools：让模型调用外部工具

如果说 Memory 解决的是“模型记不住”的问题，那么 Tools 解决的是“模型做不了”的问题。

LLM 本身擅长生成文本，但它不能天然完成这些事情：

- 查询实时天气。
- 读取本地文件。
- 创建文件。
- 执行准确计算。
- 调用业务系统接口。

Tool 就是把外部能力包装成模型可以调用的函数。

## 用途

Tools 的作用是扩展模型能力。

例如本章中有几个工具案例：

- `weather_query`：查询天气。
- `temperature_converter`：温度转换。
- `PythonREPLTool`：执行数学计算。
- `list_files`：查看文件。
- `create_file`：创建文件。
- `write_file`：写入文件。
- `delete_file`：删除文件或空文件夹。

## 好处

工具调用让 LLM 从“只会回答”变成“可以行动”。

好处包括：

- 能调用实时数据。
- 能执行确定性计算。
- 能操作外部系统。
- 能和数据库、文件系统、API 结合。
- 可以把复杂能力封装成简单函数交给 Agent 使用。

---

# 7. `@tool`：自定义工具

本章首先使用 `@tool` 把普通 Python 函数包装成 LangChain 工具。

比如天气查询工具：

```
@tool
def weather_query(city: str) -> str:
    """查询指定城市天气"""
    weather_data = {
        "北京": "北京今日天气：晴，-2~8℃",
        "上海": "上海今日天气：多云，5~12℃",
        "广州": "广州今日天气：小雨，18~25℃",
    }
    return weather_data.get(city, f"暂无 {city} 数据")
```

## 用途

`@tool` 用来把一个函数暴露给 Agent。

模型看到工具名称、参数和说明后，可以决定是否调用它。

## 好处

`@tool` 的使用成本很低：

- 写法接近普通函数。
- 可以快速封装业务能力。
- 可以通过函数参数定义工具输入。
- 可以通过 docstring 告诉模型工具用途。

## 大概如何使用

基本流程是：

1. 写一个 Python 函数。
2. 用 `@tool` 装饰。
3. 写清楚函数参数类型。
4. 写清楚 docstring。
5. 放入 `tools` 列表。
6. 交给 `create_agent` 或 `llm.bind_tools`。

示例：

```
tools = [weather_query]

agent = create_agent(
    model=llm,
    tools=tools,
    debug=True
)
```

## 难点

自定义工具的难点是“让模型知道什么时候该用它”。

这里最关键的是工具描述。

如果 docstring 写得太模糊，模型可能不会调用工具，或者调用错工具。

比如：

```
"""查询指定城市天气"""
```

就比：

```
"""处理信息"""
```

更容易让模型判断出这个工具适合天气问题。

---

# 8. args_schema：给工具参数加约束

温度转换案例里，本章使用 Pydantic 定义工具参数：

```
class TemperatureConvertInput(BaseModel):
    temperature: float = Field(description="需要转换的温度值，例如37.0")
    from_unit: str = Field(description="原始温度单位，只能是celsius或fahrenheit")
```

然后传给 `@tool`：

```
@tool(args_schema=TemperatureConvertInput)
def temperature_converter(temperature: float, from_unit: str) -> str:
    """温度单位转换工具"""
```

## 用途

`args_schema` 用来明确工具参数结构。

它告诉模型：

- 这个工具需要哪些参数。
- 参数类型是什么。
- 每个参数代表什么含义。

## 好处

相比只靠函数签名，`args_schema` 更适合正式工具：

- 参数说明更清楚。
- 模型更容易正确传参。
- 可以做基础参数校验。
- 复杂工具更容易维护。

## 难点

工具调用失败很多时候不是工具函数本身的问题，而是模型传参不符合预期。

比如温度单位要求：

```
celsius
fahrenheit
```

但用户说的是“摄氏度”，模型需要把自然语言映射成工具参数。

所以工具的参数描述和系统提示词都要尽量明确。

---

# 9. create_agent：让模型自动决定是否调用工具

工具本身只是函数，真正负责决策的是 Agent。

本章使用：

```
agent = create_agent(
    model=llm,
    tools=tools,
    debug=True
)
```

## 用途

`create_agent` 用来创建一个会使用工具的智能体。

它可以根据用户输入判断：

- 是否需要调用工具。
- 调用哪个工具。
- 给工具传什么参数。
- 如何把工具结果组织成最终回答。

## 好处

Agent 把“工具选择”这件事自动化了。

开发者只需要提供工具列表，模型会在合适的时候调用。

`debug=True` 很有用，因为它能看到中间过程：

- 模型是否决定调用工具。
- 调用了哪个工具。
- 参数是什么。
- 工具返回了什么。

## 难点

Agent 的难点是可控性。

模型可能：

- 不调用工具。
- 调错工具。
- 参数提取错误。
- 对工具返回结果理解错误。

所以工具名称、docstring、参数 schema、system prompt 都会影响最终效果。

---

# 10. PythonREPLTool：用工具做精确计算

本章组合实践中使用了：

```
PythonREPLTool
```

它可以执行 Python 表达式，用来处理数学计算。

示例链路中先判断用户输入是否包含计算意图：

```
calc_pattern = r"(\+|\-|\×|\*|÷|/|=|计算|求和|求差|平方|立方|多少|等于)"
```

如果需要计算，就调用：

```
calc_result = calc_tool.run(calc_expr)
```

## 用途

`PythonREPLTool` 适合处理模型不擅长的精确计算。

比如：

- 四则运算。
- 公式计算。
- 简单数据处理。
- 需要确定性结果的问题。

## 好处

LLM 生成数学结果并不总是可靠。

把计算交给工具，可以让结果更准确，再让 LLM 负责解释结果。

这是一种常见模式：

```
工具负责确定性执行，LLM 负责自然语言表达
```

## 难点

`PythonREPLTool` 的难点是安全性和输入清洗。

如果直接执行用户输入，风险很高。  
本章示例做了一个简化处理：只提取数字和运算符。

但真实项目中还需要考虑：

- 禁止执行危险代码。
- 限制可用函数。
- 设置超时。
- 捕获异常。
- 控制工具权限。

---

# 11. Memory + Tools：带记忆的工具智能体

本章最后把 Memory 和 Tools 组合起来，做了一个文件操作智能体。

它既能记住上下文，又能调用文件工具。

核心结构是：

```
Prompt + chat_history + llm.bind_tools(tools)
```

代码中通过下面方式绑定工具：

```
agent = prompt | llm.bind_tools(tools)
```

再通过 `RunnableWithMessageHistory` 加上记忆：

```
agent_with_memory = RunnableWithMessageHistory(
    runnable=agent,
    get_session_history=get_session_history,
    input_messages_key="input",
    history_messages_key="chat_history"
)
```

## 用途

这种组合适合构建真正可用的智能助手。

比如：

- 文件管理助手。
- 数据分析助手。
- 代码执行助手。
- 带上下文的业务客服。
- 可以连续执行任务的个人助理。

## 好处

Memory + Tools 组合后，模型不只是“知道怎么回答”，还可以“知道刚才做了什么”。

例如：

1. 用户让它创建 `test.txt`。
2. 模型调用 `create_file`。
3. 下一轮用户问“刚才创建成功了吗？”
4. 模型可以结合历史和工具结果继续回答。

这就是带状态的 Agent。

## 大概如何使用

整体流程是：

1. 定义工具函数，比如 `list_files`、`create_file`、`write_file`。
2. 把工具放入 `tools` 列表。
3. 定义带 `MessagesPlaceholder` 的 Prompt。
4. 用 `llm.bind_tools(tools)` 让模型具备工具调用能力。
5. 用 `RunnableWithMessageHistory` 注入记忆。
6. 执行模型输出，如果发现 `tool_calls`，就手动调用对应工具。
7. 把工具执行结果用 `ToolMessage` 写回历史。

工具调用的核心判断是：

```
if isinstance(result, AIMessage) and result.tool_calls:
```

然后根据工具名找到对应工具：

```
tool_func = next(t for t in tools if t.name == tool_name)
observation = tool_func.invoke(tool_args)
```

最后把工具结果写回历史：

```
history.add_message(
    ToolMessage(
        tool_call_id=call["id"],
        content=str(observation)
    )
)
```

## 难点

这里的难点是理解：模型并不是直接执行工具。

模型做的是“提出工具调用请求”，真正执行工具的是程序。

完整过程是：

```
用户输入 -> 模型判断要调用工具 -> 程序执行工具 -> 工具结果写回历史 -> 模型基于结果继续回答
```

也就是说，Agent 应用不是只调用一次模型就结束，而是包含：

- 模型思考。
- 工具调用。
- 工具返回。
- 历史记录更新。
- 最终回答生成。

这也是智能体比普通链条复杂的地方。

---

# 12. 本章核心调用模式

第三章最重要的调用模式可以总结为两条。

第一条是带记忆的对话链：

```
Prompt -> LLM
        + RunnableWithMessageHistory
```

对应过程是：

```
用户输入 -> 加载历史 -> 拼接 Prompt -> 调用模型 -> 保存历史
```

第二条是带工具的智能体：

```
Prompt -> LLM.bind_tools(tools) -> tool_calls -> ToolMessage
```

对应过程是：

```
用户输入 -> 模型决定工具调用 -> 程序执行工具 -> 工具结果回写 -> 模型继续回答
```

如果把两者合起来，就是：

```
记忆负责状态，工具负责行动，模型负责决策和表达
```

---

# 13. 总结

第三章的核心是：从“会调用模型”进阶到“能构建带状态、能行动的智能应用”。

Memory 让模型拥有上下文连续性，Tools 让模型拥有外部行动能力。

这一章最重要的不是记住某一个 API，而是理解组件分工：

- `MessagesPlaceholder`：在 Prompt 中放历史消息。
- `InMemoryChatMessageHistory`：保存对话历史。
- `RunnableWithMessageHistory`：自动管理历史注入和保存。
- `@tool`：把普通函数包装成工具。
- `create_agent` / `llm.bind_tools`：让模型具备工具调用能力。
- `ToolMessage`：把工具执行结果写回对话流程。

掌握这些之后，就可以从简单问答应用，进一步搭建带记忆、能调用工具、能完成多步骤任务的 LangChain 智能体。
