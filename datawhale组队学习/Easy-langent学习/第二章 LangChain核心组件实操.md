# 这一章讲了什么

第二章主要围绕 LangChain 中最常用的一条调用链展开：

```
ChatOpenAI -> PromptTemplate / FewShotPromptTemplate -> OutputParser
```

也就是：

1. 用 `ChatOpenAI` 初始化并调用大模型。
2. 用提示词模板组织输入，让模型更稳定地理解任务。
3. 用 FewShot 和 ExampleSelector 给模型提供示例，提升输出质量。
4. 用输出解析器把模型返回结果转换成字符串、JSON、Pydantic 对象或自定义结构，方便后续程序使用。

这一章涉及的核心组件包括：

- `ChatOpenAI`
- `PromptTemplate`
- `FewShotPromptTemplate`
- `BaseExampleSelector`
- `LengthBasedExampleSelector`
- `StrOutputParser`
- `JsonOutputParser`
- `PydanticOutputParser`
- `BaseOutputParser`

---

# 1. ChatOpenAI：模型调用入口

`ChatOpenAI` 是 LangChain 中调用聊天模型的核心封装。

本章代码中通过它连接大模型：

```
chat_model = ChatOpenAI(
    api_key=API_KEY,
    base_url=BASE_URL,
    model="deepseek-v4-flash",
    temperature=0.3,
    max_tokens=300
)
```

## 用途

`ChatOpenAI` 负责：

- 连接具体的大模型服务。
- 设置模型名称。
- 设置生成参数，如 `temperature`、`max_tokens`。
- 提供统一的 `.invoke()` 调用方式。

## 好处

它把不同模型服务封装成统一接口，后续可以和 Prompt、Parser 用管道符组合：

```
chain = prompt | llm | parser
```

这样代码从“手动拼接请求和解析响应”变成“声明式组装链条”。

## 大概如何使用

通常步骤是：

1. 从 `.env` 中读取 `API_KEY` 和 `BASE_URL`。
2. 初始化 `ChatOpenAI`。
3. 使用 `.invoke()` 传入字符串或消息列表。
4. 通过 `.content` 获取模型文本结果，或者交给输出解析器处理。

---

# 2. 提示词模板：PromptTemplate

`PromptTemplate` 用来把动态变量填充到提示词中。

例如 FewShot 示例中：

```
example_prompt = PromptTemplate(
    input_variables=["subject", "method"],
    template="学科：{subject}\n学习方法：{method}"
)
```

## 用途

它负责把结构化变量变成模型能读懂的自然语言提示词。

## 好处

相比直接手写字符串，模板有几个优势：

- 输入变量清晰。
- 方便复用。
- 不容易漏掉字段。
- 可以和 `FewShotPromptTemplate`、输出解析器组合。

---

# 3. FewShotPromptTemplate：少样本提示词

FewShot 的核心思想是：不要只告诉模型“你要做什么”，还要给它几个“参考答案”。

本章示例中先定义样本：

```
examples = [
    {
        "subject": "Python编程",
        "method": "核心目标：掌握基础语法..."
    },
    {
        "subject": "机器学习",
        "method": "核心目标：理解基础算法原理..."
    }
]
```

然后用 `FewShotPromptTemplate` 组装最终提示词：

```
few_shot_prompt = FewShotPromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    suffix="学科：{new_subject}\n请只输出学习方法，不要重复学科名和标题：",
    input_variables=["new_subject"]
)
```

## 用途

FewShot 用来引导模型模仿示例的：

- 内容结构
- 表达风格
- 输出格式
- 思考方向

比如前两个例子都是“核心目标、学习步骤、注意事项”，那么模型在回答 LangChain 学习方法时，也更容易按这个结构输出。

## 好处

FewShot 最大的价值是提升输出稳定性。

如果只写：

```
请给我 LangChain 学习方法
```

模型可能输出很散。

但如果给它几个标准示例，它更容易生成符合预期格式的结果。

## 大概如何使用

一般流程是：

1. 准备一组高质量示例。
2. 用 `PromptTemplate` 定义每个示例的展示格式。
3. 用 `FewShotPromptTemplate` 拼接示例和用户的新问题。
4. 调用 `.format()` 生成最终提示词。
5. 把提示词传给模型。

---

# 4. FewShot + ExampleSelector：动态选择示例

固定 FewShot 有一个问题：示例多了以后，提示词会变长，而且不是所有示例都适合当前问题。

所以本章又引入了 `ExampleSelector`。

示例里给了两种思路：

## 方案 A：LengthBasedExampleSelector

它按长度控制示例数量，避免提示词太长。

适合场景：

- 示例很多。
- 需要控制 token 成本。
- 不希望提示词超过模型上下文限制。

大致写法：

```
example_selector = LengthBasedExampleSelector(
    examples=examples,
    example_prompt=example_prompt,
    max_length=150,
    get_text_length=lambda x: len(x)
)
```

## 方案 B：自定义 BaseExampleSelector

本章重点实现了一个按难度筛选的选择器：

```
class DifficultyExampleSelector(BaseExampleSelector):
    def __init__(self, examples):
        self.examples = examples

    def add_example(self, example):
        self.examples.append(example)

    def select_examples(self, input_variables):
        target_difficulty = input_variables.get("difficulty", "easy")
        return [
            ex for ex in self.examples
            if ex.get("difficulty") == target_difficulty
        ]
```

然后在 `FewShotPromptTemplate` 中使用：

```
few_shot_prompt = FewShotPromptTemplate(
    example_selector=example_selector,
    example_prompt=example_prompt,
    suffix="参考以上示例，回答：\n学科：{new_subject}\n难度：{difficulty}\n学习方法：",
    input_variables=["new_subject", "difficulty"]
)
```

## 用途

`ExampleSelector` 用来动态决定“这次给模型看哪些示例”。

比如：

- 用户要入门级，就给 easy 示例。
- 用户要进阶级，就给 hard 示例。
- 输入很长时，就少给几个示例。
- 任务类型不同，就匹配不同任务样本。

## 好处

它比固定 FewShot 更工程化：

- 可以减少无关示例。
- 可以控制提示词长度。
- 可以提升示例和任务的匹配度。
- 方便从 JSON、数据库、向量库中维护示例。

## 难点

这里的难点是理解：`ExampleSelector` 不是直接生成答案，它只是决定“哪些例子进入提示词”。

真正的调用链仍然是：

```
用户输入 -> ExampleSelector 选示例 -> FewShotPromptTemplate 拼提示词 -> ChatOpenAI 生成结果
```

自定义选择器时，最关键的是实现：

```
select_examples(self, input_variables)
```

它接收当前用户输入变量，然后返回匹配的示例列表。

---

# 5. StrOutputParser：转成普通字符串

`StrOutputParser` 是最简单的输出解析器。

```
parser = StrOutputParser()
chain = llm | parser
result = chain.invoke("请简要介绍 LangChain 输出解析层的作用")
```

## 用途

它把模型返回的 `AIMessage` 对象转换成普通字符串。

## 好处

调用模型时，LangChain 默认返回的不是普通字符串，而是消息对象。  
如果后续程序只需要文本，用 `StrOutputParser` 可以直接得到 `str`。

## 适合场景

- 聊天回复。
- 文案生成。
- 摘要生成。
- 不需要结构化字段的任务。

---

# 6. JsonOutputParser：解析为 Python 字典

`JsonOutputParser` 用来让模型输出 JSON，并把结果解析成 Python 字典。

```
parser = JsonOutputParser()

prompt = PromptTemplate(
    template="请介绍1个LangChain开发工具，输出工具名和核心功能。{format_instructions}",
    input_variables=[],
    partial_variables={
        "format_instructions": parser.get_format_instructions()
    }
)

chain = prompt | llm | parser
result = chain.invoke({})
```

## 用途

它用于把模型输出转成结构化 JSON 数据。

最终可以这样访问字段：

```
result.get("tool_name")
```

## 好处

相比纯文本，JSON 更适合程序继续处理：

- 可以直接取字段。
- 可以存数据库。
- 可以传给前端。
- 可以作为下一个链条的输入。

## 难点

关键点是：

```
parser.get_format_instructions()
```

它会生成格式约束，告诉模型应该输出 JSON。

也就是说，`JsonOutputParser` 不只是“事后解析”，它还会“事前指导模型按 JSON 格式输出”。

但 JSON 解析仍依赖模型遵守格式。如果模型输出了多余解释、Markdown 代码块或非法 JSON，就可能解析失败。

---

# 7. PydanticOutputParser：带字段校验的结构化输出

`PydanticOutputParser` 比 `JsonOutputParser` 更严格。

它需要先定义一个 Pydantic 模型：

```
class ToolInfo(BaseModel):
    tool_name: str = Field(description="LangChain开发工具的名称，如 LangSmith")
    function: str = Field(description="工具的核心功能，30字以内")
    difficulty: str = Field(description="学习难度，仅可选：简单 / 中等 / 复杂")
```

然后创建解析器：

```
parser = PydanticOutputParser(pydantic_object=ToolInfo)
```

最终调用：

```
chain = prompt | llm | parser
result = chain.invoke({"user_input": "请介绍1个 Python 开发工具"})
```

## 用途

它用于把模型输出解析成一个 Pydantic 对象，并按字段规则做校验。

## 好处

相比 JSON，它更适合严肃的业务数据场景：

- 字段名固定。
- 字段类型明确。
- 字段描述清晰。
- 可以校验输出是否合法。
- 可以通过 `model_dump()` 转成字典。

例如：

```
result.difficulty
result.model_dump()
```

## 难点

难点在于：Pydantic 的字段约束越严格，模型越容易因为输出不符合要求而解析失败。

比如本章写了：

```
difficulty: str = Field(description="学习难度，仅可选：简单 / 中等 / 复杂")
```

这只是描述，不是强类型枚举约束。  
如果想更严格，可以使用 `Literal`：

```
from typing import Literal

difficulty: Literal["简单", "中等", "复杂"]
```

这样模型输出其他值时，Pydantic 会校验失败。

---

# 8. BaseOutputParser：自定义输出解析器

`BaseOutputParser` 用来实现自己的解析逻辑。

本章自定义了一个解析器，要求模型输出：

```
工具名@核心功能@学习难度
```

然后解析成字典：

```
class CustomToolParser(BaseOutputParser):
    def parse(self, text: str) -> dict:
        text = text.strip().replace("\n", "").replace(" ", "")
        parts = text.split("@")
        if len(parts) != 3:
            raise ValueError("输出格式错误")
        return {
            "tool_name": parts[0].strip(),
            "function": parts[1].strip(),
            "difficulty": parts[2].strip()
        }

    def get_format_instructions(self) -> str:
        return "请严格按照「工具名@核心功能@学习难度」格式输出，不添加多余内容。"
```

## 用途

当内置解析器不满足需求时，可以继承 `BaseOutputParser` 自定义解析规则。

## 好处

它非常灵活：

- 可以解析自定义分隔符格式。
- 可以清洗模型输出。
- 可以抛出明确错误。
- 可以适配已有系统的数据格式。

## 难点

自定义解析器的难点在于健壮性。

模型不一定完全按要求输出，所以 `parse()` 里需要考虑：

- 是否有多余换行。
- 是否有多余空格。
- 分隔符数量是否正确。
- 字段是否为空。
- 格式错误时如何报错或重试。

本章代码已经做了基础处理：

```
text.strip().replace("\n", "").replace(" ", "")
```

以及格式检查：

```
if len(parts) != 3:
    raise ValueError(...)
```

这就是自定义 Parser 的核心思想：把模型的不稳定文本输出，转换成程序可控的数据结构。

---

# 9. 本章核心调用模式

这一章最重要的是掌握 LangChain 的链式组合思想：

```
chain = prompt | llm | parser
```

它表示：

```
提示词模板 -> 大模型 -> 输出解析器
```

对应到实际开发就是：

1. Prompt 负责把用户输入组织成高质量指令。
2. LLM 负责生成结果。
3. Parser 负责把结果变成程序能用的数据。

---

# 10. 总结

第二章的核心是：从“能调用模型”进阶到“能稳定地控制模型输入和输出”。

- `ChatOpenAI`：负责调用模型。
- `PromptTemplate`：负责模板化提示词。
- `FewShotPromptTemplate`：通过示例提升输出稳定性。
- `ExampleSelector`：动态选择最合适的示例。
- `StrOutputParser`：把模型消息转成字符串。
- `JsonOutputParser`：把输出转成 JSON 字典。
- `PydanticOutputParser`：把输出转成带校验的数据模型。
- `BaseOutputParser`：实现完全自定义的解析逻辑。

这章的重点不是单个 API，而是建立一个工程化思维：

```
好的输入模板 + 合适的示例 + 明确的输出格式 = 更稳定、可维护、可集成的 LLM 应用
```