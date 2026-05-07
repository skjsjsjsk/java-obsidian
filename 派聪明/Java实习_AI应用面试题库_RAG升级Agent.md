# Java 实习 / AI 应用实习面试题库：RAG 升级 Agent

标签：#Java实习 #AI应用开发 #RAG #Agent #派聪明 #面试

## 使用定位

不要把项目说成“主导企业级 Agent 平台”，更适合的实习生人设是：

> 我是 Java 后端方向，项目是自己在原 RAG 知识库基础上做的 Agent 化改造。我重点做的是工程落地：ES 混合检索、DeepSeek OpenAI 兼容工具调用、WebClient 流式消费、Spring WebSocket 推送、Redis 生成态缓存、MinIO 文档处理，以及超时、降级、引用校验这些脏活。

面试回答建议遵循：

- 总起：先直接抛结论。
- 细节：必须结合自己的技术栈，比如 Spring Boot 3.4、WebClient、Flux 流式消费、Spring WebSocket、Elasticsearch DSL/knn、Redis Key 过期策略、MinIO、Kafka、DeepSeek tool_choice。
- 问题：讲一个自己测试时发现的小问题。
- 总结：回到“RAG 升级 Agent”的主线。

## 一、动机与选型

### 1. 为什么要从 RAG 升级到 Agent？

**考察点：** 你是不是为了蹭 Agent 概念，还是确实理解 RAG 的边界。

**话术：**  
我的理解是，RAG 更适合“一次检索 + 一次生成”的知识问答，但我测试项目时发现，用户问题稍微复杂一点，比如跨多个文档、需要总结知识库、查看知识库规模，或者检索结果为空时，普通 RAG 就比较被动。所以我在原来的链路上加了 ReAct 风格的 Agent，让模型可以先判断是否需要工具，再调用 `search_knowledge`、`generate_summary`、`knowledge_stats` 或 `submit_feedback`，然后根据 Observation 继续推理。这里我没有完全抛弃 RAG，而是把 RAG 变成 Agent 的一个工具。当前实现仍然会先进入 Agent 回合，通过 prompt 白名单减少无意义检索，后续我会考虑再加简单问题的 RAG 快路径。总结来说，我做 Agent 主要是为了解决“动态决策”和“检索失败后继续调整”，不是单纯换个名词。

### 2. 为什么用 ES 8.10 做 RAG 底座，而不是 Milvus 或 FAISS？

**考察点：** 你是否会结合项目成本选型，而不是背组件优缺点。

**话术：**  
我当时选 ES 8.10，主要是因为这个项目本身就是 Java 后端知识库场景，不只是做纯向量相似度搜索。ES 可以同时支持 DSL 关键词查询、过滤条件、权限字段、文档元数据，以及 `dense_vector` 的 `knn` 向量检索。比如一个 chunk 里我会存 `fileMd5`、`chunkId`、页码、`anchorText`、权限标签、原文片段和向量字段，检索时既可以用关键词 `match`，也可以用 `knn`，还可以用 filter 做用户、组织和公开文档范围限制。Milvus 更偏专门的向量库，FAISS 更偏本地向量索引，对我这个实习项目来说还要额外维护元数据和检索服务。ES 的优势是工程集成简单，Spring Boot 里接入也方便。缺点是极大规模向量检索可能不如专门向量库，但我的项目规模下 ES 更合适。

### 3. 为什么引入 Spring WebFlux / WebClient，同时又保留 Spring WebSocket？

**考察点：** 你是否真的用到了响应式，而不是简历堆技术栈。

**话术：**  
我这里不是把整个后端改成纯 WebFlux，而是主要用 WebFlux 里的 `WebClient` 去消费 DeepSeek、Embedding、Rerank 这些 OpenAI 兼容接口。Agent 链路里外部 IO 很多，DeepSeek 流式响应可以通过 `bodyToFlux(String.class)` 一边接收一边处理；而浏览器侧仍然用 Spring WebSocket 的 `TextWebSocketHandler` 推送 `chunk`、`tool_call`、`completion` 这些事件。也就是说，WebClient 负责非阻塞地接模型流，Spring WebSocket 负责把结果推给前端。文档解析、MinIO 文件合并、Kafka 消费这些阻塞或耗时任务，我没有强行写成响应式，而是放在文件处理链路或线程池里。总结来说，我用 WebFlux 的点主要是模型 API 流式消费，不会把它包装成全链路响应式架构。

### 4. 为什么选 DeepSeek API，而不是自己部署模型？

**考察点：** 你是否理解实习项目资源边界和 API 工程化。

**话术：**  
我没有选择自己部署大模型，主要是因为我的项目重点不是训练或推理优化，而是 Java 后端里的 AI 应用落地。DeepSeek 的 OpenAI 兼容接口对我来说更适合验证 Agent 流程，比如多轮对话、流式输出、工具调用和 JSON 参数生成。我在后端做了 `LlmProviderRouter`，把 `messages`、`tools`、`tool_choice=auto`、stream 参数收敛到一层，后面也能切到 Qwen、智谱这类兼容 Provider。自己部署模型虽然可控，但需要显存、推理框架、并发服务、量化和监控，实习项目很容易把重点带偏。当然 API 也有问题，比如网络超时、限流、成本和不可控输出，所以我在外层加了 120 秒生成超时、`StreamHandle.cancel()`、Redis 限流和 Token 配额。总结就是，我优先把“应用链路”做完整，而不是把精力放在模型部署上。

## 二、RAG 到 Agent 演进

### 5. 你这个 Agent 的 ReAct 流程具体怎么跑？

**考察点：** 能不能讲出执行闭环，而不是只会说 Thought/Action/Observation。

**话术：**  
我实现的 ReAct 流程可以简单理解成循环：用户问题进来后，先构造 system prompt 和工具列表，让 DeepSeek 判断下一步是直接回答还是调用工具。如果返回 `tool_calls`，后端解析函数名和参数，先做参数校验，再执行项目里注册过的工具。当前工具白名单主要是 `search_knowledge`、`generate_summary`、`knowledge_stats` 和 `submit_feedback`，其中知识检索最终会走 ES 混合检索。工具执行结果会包装成 Observation，再作为 tool message 放回上下文，继续请求模型。这个循环在代码里有限制，比如最多 4 轮 ReAct、最多 8 次工具调用，避免模型一直绕圈。这里关键是：模型只负责“决定调用什么工具和生成参数”，真正执行工具的是 Java 后端。总结来说，我做的是一个受控的 ReAct 执行器，不是让模型无限自由发挥。

### 6. Agent 决定调用 ES 检索时，Prompt 怎么设计？

**考察点：** 你是否理解工具描述、边界约束和参数 Schema。

**话术：**  
我在 prompt 里没有只写“你可以检索知识库”，而是把工具使用条件写得比较明确。比如用户问项目文档、上传文件、制度内容、历史知识时，默认优先调用 `search_knowledge`；如果是严格命中白名单的闲聊、格式调整、解释上一轮答案，才允许跳过检索。实际工具参数我保持得比较克制，`search_knowledge` 只有 `query` 和 `topK`，权限范围不交给模型传参，而是在 `HybridSearchService.searchWithPermission` 里根据 userId、orgTag、isPublic 这类字段拼 ES filter。DeepSeek 请求里我目前用的是 `tool_choice=auto`，不是强制 function，主要靠系统 prompt 和工具 description 约束模型。总结就是，Prompt 在这里不是文案，而是“什么时候必须查知识库、什么时候可以不查”的工具路由规则。

### 7. ES 里关键词检索和向量检索怎么结合？

**考察点：** 是否真的做过 ES 查询，而不是只写“向量数据库”。

**话术：**  
我的 ES 检索主要是混合检索思路。关键词部分用 Query DSL，比如对正文 chunk 做 `match`，语义部分用向量字段做 `knn`，根据用户问题生成 query embedding 后查相似 chunk。项目里还会根据 `topK` 扩大候选集，再通过 BM25 rescore 和 Rerank 做二次排序。ES 文档里每个 chunk 都带元数据，比如 `fileMd5`、`chunkId`、页码、`anchorText`、userId、orgTag、isPublic，这样可以在 DSL 里加 filter，避免越权检索。刚开始只靠向量检索会对专有名词、编号、接口名不稳定，所以关键词 DSL 是必要补充。总结来说，向量检索解决语义相似，关键词检索解决精确词和实体命中，Agent 拿到的是融合后的证据。

### 8. 如果 ES 检索结果为空，Agent 怎么自我纠错？

**考察点：** 你是否把 Self-Correction 落成规则，而不是让模型自由发挥。

**话术：**  
我处理检索为空时，不会让模型直接编答案，而是把它作为 Observation 放回 ReAct 上下文。当前项目没有写一个很重的独立 query rewrite 模块，主要是通过系统 prompt 引导模型在检索失败后换一个更聚焦的 `query` 再调用 `search_knowledge`。检索服务本身也有兜底：如果 embedding 或向量检索链路失败，会退到 text-only 的 DSL 检索。为了避免模型一直重复查，后端设置了 ReAct 最大 4 轮、最多 8 次工具调用；预算耗尽后会要求模型基于已有证据总结，或者明确说明知识库没找到依据。总结来说，我现在做的是“Prompt 约束 + 检索降级 + 轮次预算”的自我纠错，后续可以再补规则化 query rewrite 和低分阈值判断。

### 9. 文档分块 chunk 是怎么设计的？

**考察点：** 是否理解 RAG 质量不只靠模型，更多靠数据处理。

**话术：**  
我分块时没有简单按固定字符硬切，因为这样容易把一个概念切断。项目里会结合 Tika/PDFBox 做文档解析，PDF 会尽量保留页码，然后按语义边界、段落和长度做切分，并保留一定 overlap，避免上下文断裂。每个 chunk 会生成 embedding，然后写入 ES 的向量字段，同时保存原文、页码、文件名、`fileMd5`、`chunkId` 和 `anchorText`。这样 Agent 回答时可以把引用定位到具体片段，而不是只说“来自某个文档”。我测试时遇到过一个问题：chunk 太短，召回很多碎片，模型难总结；chunk 太长，又浪费 Token，还容易把无关内容塞进上下文。所以我会用测试问题去调 chunk 大小和 topK。总结来说，分块是 RAG 到 Agent 的底层质量基础，不能只靠 Prompt 补救。

### 10. 你怎么防止 Agent 编造引用？

**考察点：** 企业知识库场景下的可信度控制。

**话术：**  
我做了一个比较朴素但有效的限制：不要让模型自己凭空生成文件名，而是把 `search_knowledge` 返回的结果编号成 `[1]...[K]`，每条证据带文件名、页码、chunkId 和内容片段，Prompt 要求答案按这些编号引用。后端会维护本轮 generationId 下的引用映射，并把映射保存到 Redis 生成态和最终历史里，前端展示引用时可以按编号找到真实 chunk。当前项目更偏“引用可追溯”，还没有做很重的生成后引用 ID 强校验，所以面试里不能说我做了完整 citation validator。我测试时确实遇到过模型编造来源的问题，所以后来把引用从自然语言文件名改成结构化证据编号。总结就是，引用可信度要靠后端证据映射兜住，不能完全相信模型自觉。

## 三、WebFlux / WebSocket / 并发

### 11. Agent 多轮工具调用时，怎么保证打字机效果不卡？

**考察点：** 是否理解流式输出不等于最终一次性返回。

**话术：**  
我把 Agent 的输出拆成两类事件：一类是模型 token 流，一类是 Agent 工具状态。模型 API 这边用 WebClient 的 `bodyToFlux(String.class)` 消费 DeepSeek 流式响应，每收到一段内容就追加到 Redis 生成态，并通过 `ChatSessionRegistry.sendJsonToUser` 推给前端 `chunk` 事件。如果模型中间返回 `tool_calls`，后端会先推 `tool_call` 状态，比如正在检索知识库、工具完成或失败，这样前端不会空等。这里要说清楚：我的 WebSocket 用的是 Spring `TextWebSocketHandler`，不是 WebFlux 响应式 WebSocket 那套接口。我测试时发现，如果只在最终完成时返回，用户会以为系统卡住，所以中间状态必须推出来。总结来说，不卡顿不是靠模型更快，而是靠流式 token 和工具事件拆开推。

### 12. WebFlux 里有没有遇到阻塞问题？

**考察点：** 是否知道响应式项目里最常见的坑是乱用阻塞调用。

**话术：**  
有遇到。这个项目不是纯 WebFlux 架构，容易讲错的点是：我主要用 WebClient 的 `Flux` 消费模型 API 流，ReAct 主循环是在后端工作线程里跑，WebSocket 还是传统 Spring WebSocket。所以我不会说“全链路响应式”。真正要注意的是不要在 WebSocket 处理线程上等 90 秒以上的模型流，代码里把 ReAct 决策循环放到了单独执行器里；文档解析、MinIO 大文件处理、Kafka 消费这些本来就是偏阻塞或耗时任务，也不放到模型流式消费链路里。项目里确实还有一些非流式接口会用到 `block(Duration)` 或同步等待，但它们不在 Netty event loop 里。总结来说，我用 WebFlux 的重点是外部模型 IO 的流式消费，而不是为了把所有方法都改成 `Mono`。

### 13. 多个工具能不能并发调用？怎么保证顺序？

**考察点：** 是否会区分可并发任务和有依赖任务。

**话术：**  
我的处理比较保守，当前 ReAct 主流程基本是串行执行工具。原因是很多步骤有推理依赖，比如先 `search_knowledge` 查知识库，再根据 Observation 决定要不要总结或继续检索；如果为了并发把多个工具同时跑起来，反而容易让模型拿到互相冲突的上下文。项目里真正需要优化的是减少无效工具调用和限制轮次，而不是盲目并发。代码里也有最大 4 轮 ReAct、最多 8 次工具调用这类预算控制。面试官如果追问并发优化，我会说后续可以对互不依赖的只读工具做并发，比如知识库统计和检索可以拆开跑，但当前为了可控性先串行。总结来说，我的原则是：Agent 决策链路先保证可解释和可控，再考虑并发。

### 14. WebSocket 断开了怎么办？

**考察点：** 是否考虑真实前端连接问题。

**话术：**  
我没有把 WebSocket 当成绝对可靠的通道。每次生成会有 generationId，后端会把生成中的 meta、content、refs 存到 Redis，比如 `chat:generation:{id}:meta`、`chat:generation:{id}:content`、`chat:generation:{id}:refs`，还有 `chat:user:{userId}:active_generation`，TTL 大概 30 分钟。前端断线或刷新后，可以根据活跃 generation 拉取快照，恢复已经生成的内容和引用映射。对于已经完成的会话，最终还是以 MySQL 的历史记录为准，Redis 只是生成过程中的短期兜底。这个项目里我没有做复杂的分布式消息重放，但至少避免了刷新页面就完全丢失正在生成的答案。总结来说，WebSocket 只负责实时推送，关键生成态不能只存在内存里。

## 四、文件、Redis、缓存与一致性

### 15. MinIO 文档怎么解析并灌入 ES？

**考察点：** 是否知道文件入库链路，而不是只会上传文件。

**话术：**  
我的链路是：用户上传文件后，文件本体分片上传并合并到 MinIO，数据库保存文件元数据，比如 fileMd5、文件名、上传人、组织标签、是否公开和解析状态。然后通过 Kafka 文件处理任务读取 MinIO 对象，根据文件类型用 Tika/PDFBox 解析文本，做清洗、分段、embedding，再写入 ES。ES 里不是存整个文件，而是按 chunk 存，每个 chunk 有 `fileMd5`、`chunkId`、页码、`anchorText`、原文片段、向量字段和权限字段。解析成功后更新数据库状态；如果失败，记录失败原因，方便前端提示。这个过程中我踩过一个坑：文件上传成功不代表知识库可检索，所以状态机要把上传、解析、索引区分开。总结来说，MinIO 管原始文件，ES 管可检索片段，数据库管状态和元数据。

### 16. Redis 在 Agent 里具体缓存了什么？

**考察点：** 是否能说清 Redis Key 设计和过期策略。

**话术：**  
我在这个项目里没有把 Redis 包装成“长期记忆库”，它主要做短期状态。第一类是生成态，比如 `chat:generation:{id}:meta/content/refs`，存生成状态、已输出内容和引用映射，TTL 大概 30 分钟；还有 `chat:user:{userId}:active_generation`，用于前端刷新后恢复正在生成的回答。第二类是会话短期上下文，比如 `conversation:{conversationId}` 和 `user:{userId}:current_conversation`，TTL 大概 7 天，长期历史还是落 MySQL。第三类是工程辅助，比如上传分片 bitmap：`upload:{userId}:{fileMd5}`，以及聊天限流、Token 配额、反馈 hash 等。Redis Key 都会设置合理过期，也就是 Key 过期策略，避免临时状态堆积。总结来说，Redis 在这里是短期生成态、会话状态和工程辅助，不是工具结果缓存。

### 17. 知识库更新时，怎么避免 ES 数据不一致？

**考察点：** 是否考虑文档重建、幂等和旧数据清理。

**话术：**  
我处理知识库更新时，会以 `fileMd5` 作为文件维度的核心标识，每个 chunk 带 `fileMd5`、`chunkId`、页码和模型版本。重新解析文档时，不能只看 MinIO 里有没有文件，还要看数据库解析状态和 ES chunk 是否已经写入成功。比较稳的做法是让解析任务幂等：同一个 fileMd5 重复处理时，生成的 chunkId 可追踪，失败时记录状态，不要让前端误以为“上传成功就一定可检索”。实习项目里我没有做双索引灰度或 alias 切换，面试里不能夸大；如果追问后续优化，我会说可以按版本字段或索引 alias 做更完整的重建切换。总结来说，知识库更新的重点是 fileMd5、状态机、幂等和失败可恢复。

### 18. 怎么避免 Agent 重复调用同一个工具？

**考察点：** 是否处理 Agent 循环和成本浪费。

**话术：**  
我现在主要靠流程预算来避免 Agent 重复调用工具，而不是 Redis 参数级去重。代码里有 `MAX_REACT_ROUNDS=4` 和 `MAX_REACT_TOOL_CALLS=8`，超过后会把“工具预算已用尽”的约束放回上下文，让模型基于已有 Observation 输出最终回答。每次工具失败也不会直接抛异常，而是作为结构化 tool message 返回给模型，让它决定换 query 还是结束。我测试时发现，模型在检索失败时确实可能换个说法继续查，所以这个预算很重要。这里不能夸大成“我已经做了工具结果去重缓存”，因为当前代码没有。后续如果优化，我会再按 userId、toolName、query hash 做短 TTL 工具结果缓存。总结来说，当前已落地的是轮次和工具次数限制，去重缓存是后续优化点。

## 五、DeepSeek 工具调用与容错

### 19. DeepSeek 的 Function Calling 你怎么接的？

**考察点：** 是否知道工具调用是模型产出参数，后端执行工具。

**话术：**  
我对接时是按 OpenAI 兼容 Chat Completion 的 tools 格式来做。后端有 `AgentToolRegistry`，把工具注册成 name、description 和 JSON Schema，比如 `search_knowledge` 只需要 `query` 和 `topK`，`generate_summary` 需要 `topic` 和 `maxDocs`。请求 DeepSeek 时，`LlmProviderRouter` 会带上 `tools`，并设置 `tool_choice=auto`，由模型判断是否返回 `tool_calls`。模型返回后，我不会直接让它接触 ES，而是在 Java 后端解析函数名和 arguments，再执行白名单工具；未知工具名会直接报“未注册的工具”。执行结果会作为 role=tool 的 message，带 `tool_call_id` 回传给模型继续生成。总结来说，Function Calling 不是模型真的调用函数，而是模型生成结构化调用意图，真正执行权在后端。

### 20. DeepSeek 超时、限流或者不调用工具怎么办？

**考察点：** 是否有真实 API 容错经验。

**话术：**  
我遇到过两类问题：一种是 API 网络超时或返回慢，另一种是明明该查知识库，模型却直接回答。超时这块，当前代码不是在工具层单独包响应式超时，而是在 `ChatHandler` 里做 120 秒生成截止，等待流式 ReAct 回合时会短轮询 `CompletableFuture`，超时或用户点停止就调用 `StreamHandle.cancel()`。限流和 Token 不足会通过配额服务、错误事件和 WebSocket 返回给前端。模型不调用工具的问题，我现在主要靠 prompt 强约束：除严格白名单外，问题默认要先 `search_knowledge`；`tool_choice` 当前仍是 `auto`，还没有落地强制工具调用。总结来说，大模型 API 要按不稳定外部服务处理，同时不能夸大当前实现。

### 21. 工具参数不合法怎么办？

**考察点：** 是否知道 LLM 输出不能直接进后端逻辑。

**话术：**  
我不会把模型生成的工具参数直接拿去执行。DeepSeek tools 里有 JSON Schema，后端 Java 层还会再校验一遍。比如 `search_knowledge` 的 `query` 是必填，`topK` 默认 5，并限制在 1 到 20；`submit_feedback` 的 rating 只能是 good 或 bad；未知工具名直接走未注册异常。权限范围也不让模型传参控制，而是在后端执行 `searchWithPermission` 时根据当前 userId、组织标签和公开状态做 ES filter。参数缺失或非法时，工具执行异常会被捕获，包装成 tool message 返回给模型，而不是让 WebSocket 直接断掉。总结来说，Agent 工具调用必须“模型建议、后端裁决”，不能让模型绕过业务规则。

### 22. 如果模型生成了错误答案，你怎么定位是哪一步错了？

**考察点：** 是否有可观测性和 Trace 思维。

**话术：**  
我会看完整 Agent 执行链，而不是只看最终回答。一次请求里我重点看用户问题、system prompt、模型返回的 `tool_calls`、工具名和参数、`search_knowledge` 返回的 chunk、引用映射、工具状态事件、生成耗时和 Token 估算。比如答案错了，我先判断是 ES 没召回、召回了但 Rerank 排序差、检索结果为空后模型硬答，还是最终生成没有按引用作答。当前项目没有做完整的可视化 Trace 平台，所以面试里我会说“结构化日志 + WebSocket 工具事件 + Redis 生成态快照”能辅助复现，而不是说有专业观测系统。总结来说，Agent 问题必须按链路拆，不然很难定位。

## 六、评估、成本与线上质量

### 23. 怎么评估 Agent 是否比原 RAG 更好？

**考察点：** 是否会量化，不是只说“感觉更智能”。

**话术：**  
我会用自己构造的评测集来对比，而不是只看几个 demo。评测集可以分成简单知识问答、多文档总结、检索失败、知识库统计、无答案问题几类，分别跑原 RAG 和 Agent，看召回命中率、引用正确率、平均响应时间、平均工具调用次数和失败原因。比如知识库统计这类问题，原 RAG 很难准确回答，Agent 可以调用 `knowledge_stats`；总结类问题可以调用 `generate_summary`。简单问题如果 Agent 链路明显更慢，就说明后续要做路由优化。当前项目还没有独立 RAG 快路径，所以面试时要说“这是优化方向”，不能说已经完整落地。总结来说，Agent 应该在复杂任务上提升，但不能默认所有问题都更好。

### 24. Agent 链路变长，延迟怎么优化？

**考察点：** 是否理解延迟来自模型、检索、工具和串行步骤。

**话术：**  
我主要从已落地和待优化两块讲。已落地的是流式和预算控制：DeepSeek 流式响应用 WebClient 的 `Flux` 消费，再通过 Spring WebSocket 推 `chunk` 和 `tool_call`，用户能看到中间状态；ReAct 有最大 4 轮、最多 8 次工具调用，生成整体有 120 秒截止。检索层会控制 `topK`，并用 ES 初召回加 Rerank，而不是把大量 chunk 都塞给模型。还没落地的是简单问题的 RAG 快路径和工具结果短期缓存，所以面试里要说这是后续优化，不能当成现有成果。总结来说，延迟优化不是只让模型快，而是流式感知、减少无效工具调用、控制召回和限制轮次。

### 25. Token 成本暴涨怎么解决？

**考察点：** 是否知道 Agent 的真实成本问题。

**话术：**  
Agent 比 RAG 更容易烧 Token，因为它会多轮调用模型，每次还要带历史上下文、工具描述和 Observation。我现在主要做了三件事。第一是请求前做 Token 估算和配额预占，`UsageQuotaService` 会估算 prompt、tools 和 max completion 的消耗，避免无预算时继续调用。第二是控制输出长度，DeepSeek 请求会带 `max_tokens`，ReAct 每轮也有最大 completion token。第三是工具结果结构化，只把检索 chunk 的编号、文件名、页码和关键片段给模型，不传整个文件。当前还没有 Redis 工具结果缓存，也没有完整问题路由快路径，这两块是后续成本优化方向。总结来说，成本控制的核心是少传无效上下文、限制轮次和提前做配额控制。

### 26. Agent 输出不可控，怎么做安全边界？

**考察点：** 是否知道 AI 应用上线风险。

**话术：**  
我主要做了几类边界。第一是工具白名单，Agent 只能调用 `AgentToolRegistry` 里注册的 `search_knowledge`、`generate_summary`、`knowledge_stats`、`submit_feedback`，不能自己构造任意接口。第二是参数校验，比如 `topK` 范围、rating 枚举、必填 query 都在后端校验。第三是权限过滤，知识检索必须走 `searchWithPermission`，由 ES filter 控制 userId、orgTag、isPublic。第四是最大轮次和 120 秒超时，避免无限循环。第五是引用映射，回答中的来源要能追到本轮检索 chunk。作为实习项目，我没有做高风险写操作工具。总结来说，Agent 的自由度必须被后端规则框住，尤其不能让模型决定权限边界。

## 七、Java 后端能力深挖

### 27. 这个项目哪里体现了 Java 后端能力？

**考察点：** 防止你变成“调 API 工程师”。

**话术：**  
我觉得主要体现在几个地方。第一是后端链路分层，`ChatHandler` 负责 ReAct 循环，`AgentToolRegistry` 负责工具注册和执行，`LlmProviderRouter` 负责 OpenAI 兼容请求，`HybridSearchService` 负责 ES 混合检索。第二是异步和流式处理，模型 API 用 WebClient 的 `Flux` 消费流，前端通过 Spring WebSocket 接收 chunk 和 tool_call。第三是中间件落地，ES 做 DSL/`knn` 检索，Redis 做生成态和 TTL，MinIO/Kafka 做文件上传后的异步解析链路。第四是异常和状态处理，超时、取消、工具失败都会变成可恢复状态。总结来说，这个项目不是只会调用 DeepSeek，而是把 AI 能力接进 Java 后端工程链路里。

### 28. 模型流式链路里异常怎么统一处理？

**考察点：** 是否知道流式调用、工具执行和 WebSocket 推送的错误不能只靠最终兜底。

**话术：**  
这个问题我会先纠正一下：我的项目不是纯 WebFlux 项目。模型流式调用这一层确实是 WebClient/`Flux`，但 Agent 工具执行和 WebSocket 推送主要是 Java 同步方法加线程池。我的处理方式是把异常尽量转成可理解的状态：工具执行失败会捕获异常并包装成 tool message，比如“工具 search_knowledge 执行失败”；生成超时会标记 Redis generation 为 failed，并通过 WebSocket 推 error/completion；用户取消会调用 `StreamHandle.cancel()` 并标记 cancelled。同步参数校验则直接抛业务异常。总结来说，核心不是套响应式写法，而是让失败可见、可恢复、可落状态。

### 29. 你怎么处理权限过滤？

**考察点：** 企业知识库绕不开多用户和越权问题。

**话术：**  
我的思路是权限过滤必须在 ES 查询阶段做，而不是检索回来后再让模型判断。每个 chunk 写入 ES 时会带上权限相关元数据，比如 userId、orgTag、isPublic。Agent 调 `search_knowledge` 时不会让模型传权限范围，而是后端用当前登录 userId 调 `HybridSearchService.searchWithPermission`，在 DSL/`knn` 查询里拼权限 filter，只返回当前用户能看的 chunk。这样模型拿到的 Observation 本身就是过滤后的结果。DeepSeek 不知道用户权限，也不能让它决定能不能看某个文件。当前没有 Redis 工具结果缓存，所以也不存在跨用户复用工具缓存；如果后续做缓存，Key 一定要带 userId 或 orgTag。总结来说，权限要在后端和 ES 层解决，不能交给大模型。

### 30. 如果面试官问你和 Dify / LangChain 有什么区别，怎么答？

**考察点：** 是否了解行业工具，也能解释自己项目价值。

**话术：**  
我会先承认 Dify、LangChain 这类框架更成熟，内置了很多 Agent、RAG、工作流能力。我这个项目不是要重复造一个完整平台，而是为了学习和落地 Java 后端里的 Agent 核心链路。我自己实现的部分更偏底层工程：工具注册怎么设计、DeepSeek 的 `tool_choice=auto` 和 `tool_calls` 怎么解析、ES 的 DSL 和 `knn` 怎么融合、WebClient 流式响应怎么转成 Spring WebSocket 事件、Redis 怎么存 generation 快照。用现成框架当然更快，但很多细节容易变成配置黑盒。自己改一遍后，我能讲清楚一次 Agent 从用户问题到工具调用、再到 Observation 和最终回答的完整过程。总结来说，我不是说自己比框架强，而是这个项目能证明我理解 AI 应用的后端落地细节。

## 八、复盘与实习生人设题

### 31. 从 RAG 到 Agent，你踩过最大的坑是什么？

**考察点：** 是否有真实实践，不是包装项目。

**话术：**  
我踩过最大的坑是，一开始我以为 Agent 只要接上 tools 就会稳定变强，后来测试发现模型有时会跳过检索直接回答，或者检索失败后反复换说法调用 `search_knowledge`，延迟和 Token 都会上去。我的解决方式不是继续堆 prompt，而是加边界：系统 prompt 明确“知识库优先”和跳过检索白名单，后端限制最大 4 轮 ReAct、最多 8 次工具调用，超时后取消流，工具失败也作为 Observation 返回。Redis 只负责生成态恢复，不负责工具结果缓存。这个坑让我意识到，Agent 不是替代 RAG，而是把 RAG 工具化后再加决策能力。总结来说，我现在会更强调边界控制，而不是所有问题都 Agent 化。

### 32. 如果让你继续优化这个项目，你会优先做什么？

**考察点：** 是否有后续思考和工程判断。

**话术：**  
如果继续优化，我会优先做评测集和 Trace 可视化。现在很多 AI 项目容易只看 demo，真正面试或上线时说不清效果。我会把问题分成简单问答、多文档总结、检索失败、无答案、知识库统计几类，分别统计引用正确率、平均延迟、平均工具调用次数和失败原因。Trace 可视化则是为了看每次 Agent 为什么调用某个工具、ES 查了什么 DSL 或 `knn`、返回了哪些 chunk、最终答案引用了哪些证据。第二步我会做问题路由快路径和工具结果短 TTL 缓存，避免简单问题也走完整 ReAct。总结来说，我下一步不会急着加更多工具，而是先把效果评估和问题定位做好。

## 备用追问清单

### Agent / RAG 主线

- RAG 和 Agent 的边界是什么？
- 为什么说 Agent 不是替代 RAG，而是把 RAG 工具化？
- ReAct 的 Action 和 Observation 在你项目里分别对应什么？
- Agent 最大 step 设置多少？为什么？
- Agent 怎么判断是否该追问用户？
- 你怎么避免模型在检索失败后硬答？
- 你的 Agent 有没有 Memory？短期 Memory 和长期 Memory 怎么区分？
- 你有没有做 Query Rewrite？怎么触发？
- Agent 检索失败后重试几次？为什么不能无限重试？

### Elasticsearch / 检索

- ES 的索引 mapping 怎么设计？
- `dense_vector` 字段怎么存？
- `knn` 查询和普通 Query DSL 怎么组合？
- `topK` 取多少？太大太小有什么问题？
- ES 返回分数低时怎么处理？
- 关键词命中和向量命中冲突时怎么排序？
- 你怎么处理专有名词、接口名、编号这类精确查询？
- chunk 过大或过小分别有什么问题？
- 文档删除或更新后，ES 里的旧 chunk 怎么处理？

### WebFlux / WebSocket

- `Mono` 和 `Flux` 在项目里分别表示什么？
- WebFlux 里为什么不能随便 `block()`？
- WebSocket 推送 token 流时，后端事件格式怎么设计？
- 如果 DeepSeek 流式返回中断，前端会看到什么？
- 工具调用期间前端如何显示状态？
- 多个工具能不能并发执行？
- WebSocket 断开重连后怎么恢复状态？
- WebFlux 和 Spring MVC 在线程模型上有什么差别？

### Redis / MinIO / 文件链路

- Redis 里哪些 Key 设置 TTL？为什么？
- 当前为什么没有做工具结果缓存？如果后续做，Key 里为什么必须带 userId 或 orgTag？
- Redis 里的短期上下文和 MySQL 的历史记录有什么区别？
- MinIO 存原文件，ES 存 chunk，MySQL 存什么？
- 文件上传成功但解析失败怎么办？
- 解析任务是同步还是异步？为什么？
- 如果用户上传大文件，怎么避免请求阻塞？
- 文档解析状态有哪些？

### DeepSeek / 大模型 API

- 你当前为什么只用 `tool_choice=auto`，没有强制指定工具？
- Function Calling 是模型执行函数吗？
- 工具参数为什么还要后端校验？
- DeepSeek 超时怎么降级？
- 模型不调用工具直接回答怎么办？
- 模型返回 JSON 不合法怎么办？
- 如何限制模型输出长度和 Token 成本？
- 如何记录一次模型调用的输入、输出和 Token？

### 项目复盘

- 这个项目最难的点是什么？
- 你最能体现 Java 能力的代码在哪里？
- 如果面试官要求现场画架构图，你怎么画？
- 项目里哪些地方还不够成熟？
- 如果用户量扩大 10 倍，最先会瓶颈在哪里？
- 如果让你接入 LangChain4j，你会改哪里？
- 如果让你上线到真实企业环境，最先补什么？

## 面试时不要这么说

- 不要说“我们团队主导了 Agent 平台”，因为这是个人项目，容易被追问崩。
- 不要说“准确率提升 80%”，除非你有真实评测集和统计口径。
- 不要把 Redis 说成长期记忆库或工具结果缓存，Redis 在这里主要是生成态、短期会话、上传分片和限流配额。
- 不要说 WebFlux 一定比 Spring MVC 快，应该说你主要用 WebClient/Flux 消费模型 API 流式响应。
- 不要说 Function Calling 是模型调用函数，正确说法是模型生成工具调用意图，后端执行。
- 不要把 Agent 包装成万能，应该强调路由、边界、降级和成本控制。

## 推荐准备材料

- 准备 30 到 100 条自测问题，分成简单问答、多文档问答、检索失败、无答案、工具调用几类。
- 记录原 RAG 和 Agent 的对比：命中率、引用正确率、平均延迟、平均工具调用次数。
- 准备一张手绘架构图：前端 WebSocket、后端 Agent Executor、DeepSeek、Tool Registry、ES、Redis、MinIO、MySQL。
- 准备 2 到 3 条真实 bug 复盘：比如重复调用工具、检索为空硬答、WebSocket 等待卡顿、chunk 切分不合理。

## 参考来源

- 牛客：腾讯 RAG / Agent 面经整理  
  https://www.nowcoder.com/discuss/878945851924627456?sourceSSR=post
- 牛客：小红书校招面经列表，含 AI 后端 / Agent 项目追问  
  https://www.nowcoder.com/enterprise/715/interview
- 小红书社区 Agent 开发实习生 JD，含 Coding Agent、代码库 RAG、Eval 要求  
  https://campus.niuqizp.com/job-vwl5tZzCt.html
- 小红书 AI 研发效能与智能编程工具实习 JD，含 RAG 工程优化、评测集、Agent 插件  
  https://www.mianshima.com/job/15/18558
- Spring WebFlux 官方文档  
  https://docs.spring.io/spring-framework/reference/web/webflux.html
- Spring WebSocket 官方文档  
  https://docs.spring.io/spring-framework/reference/web/websocket.html
- Elasticsearch kNN Query 官方文档  
  https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-knn-query
- DeepSeek Function Calling 官方文档  
  https://api-docs.deepseek.com/guides/function_calling
- DeepSeek Chat Completion / tool_choice 官方文档  
  https://api-docs.deepseek.com/api/create-chat-completion
- Redis EXPIRE 官方文档  
  https://redis.io/docs/latest/commands/expire/
