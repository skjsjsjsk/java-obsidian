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
我把 Agent 的输出拆成两类流：一类是模型 token 流，一类是 Agent step 事件流。比如模型正在生成时，通过 `Flux` 不断发 `answer_chunk`；如果中间需要调用 ES，就先发 `tool_call_start`，工具完成后再发 `tool_call_result`，这样前端不会空等。后端 WebSocket 使用 WebFlux 的 `WebSocketSession.send(Publisher)` 思路，把事件转换成 text message 推出去。单个工具调用用 `Mono` 包装，并加 timeout 和 `onErrorResume`。我测试时发现，如果等所有工具都执行完再统一让模型总结，用户会感觉卡住，所以我把中间状态也推给前端。总结来说，不卡顿不是靠模型更快，而是靠事件流设计，让用户知道系统正在查什么、做到哪一步了。

### 12. WebFlux 里有没有遇到阻塞问题？

**考察点：** 是否知道响应式项目里最常见的坑是乱用阻塞调用。

**话术：**  
有遇到。刚开始我有些地方为了省事直接调用阻塞方法，比如读取文件内容、等待外部 API 返回，甚至调试时用过类似 `block()` 的写法。后来我发现这样会破坏 WebFlux 的非阻塞优势，严重时会让事件循环线程被占住。我的处理方式是：能用响应式客户端的地方尽量返回 `Mono` 或 `Flux`，比如模型 API、Redis、WebSocket 推送；确实阻塞的任务，比如文档解析或 MinIO 大文件处理，就放到单独线程池或异步任务里，不放在 Netty event loop 上。工具调用层统一返回 `Mono<ToolResult>`，这样 Agent 执行器可以用 `flatMap`、`concatMap` 编排。总结来说，WebFlux 不是把返回值改成 Mono 就完事，关键是链路里不能偷偷阻塞。

### 13. 多个工具能不能并发调用？怎么保证顺序？

**考察点：** 是否会区分可并发任务和有依赖任务。

**话术：**  
我的处理比较保守，不是所有工具都并发。像“同时查知识库和文件标题”这种互不依赖的工具，可以用 `Flux.merge` 或 `Mono.zip` 并发查，最后合并 Observation。但 ReAct 主流程里很多步骤是有依赖的，比如先检索文档，再根据结果决定是否调用外部 API，这种就不能并发乱跑，我会用 `concatMap` 或显式 step 循环保证顺序。面试里我会说清楚，我作为实习项目更关注可控性，不会为了并发把 Agent 行为搞复杂。并发工具调用还要注意超时和失败隔离，比如一个工具失败不能拖死整个 Agent，可以返回结构化错误，让模型或后端选择降级。总结来说，我的原则是：无依赖可并发，有推理依赖必须串行。

### 14. WebSocket 断开了怎么办？

**考察点：** 是否考虑真实前端连接问题。

**话术：**  
我没有把 WebSocket 当成绝对可靠的通道。每次会话会有 sessionId，Agent 每一步状态会短期存到 Redis，比如 `agent:step:{sessionId}`，Key 设置过期策略。前端断线重连后，会带 sessionId 请求最近的状态，后端可以把已经完成的 step 和最终答案恢复出来。对于正在执行的任务，我会区分两种情况：如果后端任务已经完成，只是前端断开，就从 Redis 或数据库读结果；如果任务还在跑，可以让前端重新订阅同一个 session 的事件流。这个项目里我没有做非常复杂的分布式消息恢复，但至少避免了刷新页面就完全丢失上下文。总结来说，WebSocket 只负责实时推送，关键状态不能只存在内存里，要有 Redis 短期兜底。

## 四、文件、Redis、缓存与一致性

### 15. MinIO 文档怎么解析并灌入 ES？

**考察点：** 是否知道文件入库链路，而不是只会上传文件。

**话术：**  
我的链路是：用户上传文件后，文件本体存到 MinIO，数据库保存文件元数据，比如 bucket、objectName、文件名、上传人、解析状态。然后后台任务读取 MinIO 对象，根据文件类型解析文本，做清洗、分段、embedding，再写入 ES。ES 里不是存整个文件，而是按 chunk 存，每个 chunk 有 `docId`、`chunkId`、页码、原文片段、向量字段和权限字段。解析成功后更新数据库状态；如果失败，记录失败原因，方便前端提示。这个过程中我踩过一个坑：文件上传成功不代表知识库可检索，所以我后来把状态区分成 uploaded、parsing、indexed、failed。总结来说，MinIO 管原始文件，ES 管可检索片段，数据库管状态和元数据。

### 16. Redis 在 Agent 里具体缓存了什么？

**考察点：** 是否能说清 Redis Key 设计和过期策略。

**话术：**  
我在 Agent 里主要用了三类 Redis 缓存。第一类是会话短期上下文，比如 `agent:session:{sessionId}`，存最近几轮对话摘要和当前 step，过期时间大概 30 分钟。第二类是工具结果缓存，比如 `agent:tool:search:{hash}`，同一会话里相同 query 的 ES 检索结果 3 到 5 分钟内复用，防止模型重复问同一个问题导致重复查 ES。第三类是执行状态，比如当前调用到了第几步、是否超时、失败原因，方便 WebSocket 断线恢复。Redis Key 都会设置 TTL，也就是 Key 过期策略，避免 Agent 长链路把 Redis 堆满。我没有把长期历史存在 Redis，持久数据还是进 MySQL 或 ES。总结来说，Redis 在这里是短期状态和去重，不是长期记忆库。

### 17. 知识库更新时，怎么避免 ES 数据不一致？

**考察点：** 是否考虑文档重建、幂等和旧数据清理。

**话术：**  
我处理知识库更新时，会以 `docId` 作为文档维度的核心标识，每个 chunk 带 `docId` 和版本号。重新解析文档时，不是简单覆盖原文件，而是生成新的 chunk 列表，写入 ES 时保证 chunkId 可追踪。比较稳的做法是先写新版本 chunk，确认成功后再把数据库里的文档版本切到新版本，旧 chunk 再清理或标记不可用。实习项目里我没有做很复杂的双索引灰度，但至少保证失败时不会把原来的可检索数据直接删掉。面试官如果追问，我会说可以进一步用 alias 做索引切换，实现更接近不停服重建。总结来说，知识库更新的重点是幂等、状态机和失败回滚，而不是简单“删了重建”。

### 18. 怎么避免 Agent 重复调用同一个工具？

**考察点：** 是否处理 Agent 循环和成本浪费。

**话术：**  
我做了两层限制。第一层是流程限制，Agent 执行器有最大 step 数，比如最多 3 次工具调用，超过就降级总结已有信息或提示用户补充问题。第二层是 Redis 去重，我会把 toolName、query、docScope 这些参数做 hash，生成类似 `agent:tool:dedup:{sessionId}:{hash}` 的 Key，设置几分钟过期。如果同一轮会话里模型又调用了相同工具和相同参数，就直接复用上次 Observation，或者告诉模型这个查询已经执行过。这样能减少 ES 重复查询和 DeepSeek 重复消耗 Token。我测试时发现，模型在检索失败时容易换个说法重复查，所以去重不能只比原始 query，还要配合 query rewrite 次数限制。总结来说，重复调用要靠 step 限制和缓存去重一起控制。

## 五、DeepSeek 工具调用与容错

### 19. DeepSeek 的 Function Calling 你怎么接的？

**考察点：** 是否知道工具调用是模型产出参数，后端执行工具。

**话术：**  
我对接时是按 Chat Completion 的 tools 格式来做。后端会把可用工具注册成列表，每个工具有 name、description 和 JSON Schema 参数，比如 `knowledge_search` 需要 `query`、`topK`、`searchMode`。请求 DeepSeek 时带上 `tools`，`tool_choice` 默认用 `auto`，让模型判断是否调用工具；如果我判断这轮必须查知识库，就指定具体 function。模型返回 `tool_calls` 后，我不会直接信任参数，而是在 Java 后端做字段校验、默认值补齐和权限检查，然后执行 ES 或文件搜索。执行结果再作为 tool message 带 `tool_call_id` 回传给模型，让它生成最终回答。总结来说，Function Calling 不是模型真的调用函数，而是模型生成结构化调用意图，真正执行权在后端。

### 20. DeepSeek 超时、限流或者不调用工具怎么办？

**考察点：** 是否有真实 API 容错经验。

**话术：**  
我遇到过两类问题：一种是 API 网络超时或返回慢，另一种是明明该查知识库，模型却直接回答。超时这块，我在 WebFlux 里用 `Mono.timeout` 控制单次调用时间，配合有限重试和 `onErrorResume` 降级。如果是限流或连续失败，就返回一个明确的 fallback，比如“当前模型服务繁忙，可以先基于检索结果给出简要回答”。模型不调用工具的问题，我主要用两种方式处理：第一是 prompt 写清楚必须检索的场景；第二是对需要引用的问题，把 `tool_choice` 从 `auto` 改成指定工具或 `required`。如果最终答案没有引用，我会拦截并要求重新检索。总结来说，大模型 API 要按不稳定外部服务处理，不能假设它每次都听话。

### 21. 工具参数不合法怎么办？

**考察点：** 是否知道 LLM 输出不能直接进后端逻辑。

**话术：**  
我不会把模型生成的工具参数直接拿去执行。每个工具都有参数 Schema，比如 `topK` 只能在一个范围内，`searchMode` 只能是 keyword、vector、hybrid，docScope 只能是用户有权限的文档范围。DeepSeek 的 tools 里可以用 JSON Schema 约束，后端还会再做一次 Java 层校验，因为模型输出仍然可能不符合预期。如果参数缺失，我会补默认值；如果参数越权或格式错误，就返回结构化错误 Observation，比如 `INVALID_ARGUMENT`，让 Agent 选择修正或结束。对于外部 API 这类工具，还会加白名单和超时。总结来说，Agent 工具调用必须“模型建议、后端裁决”，不能让模型绕过业务规则。

### 22. 如果模型生成了错误答案，你怎么定位是哪一步错了？

**考察点：** 是否有可观测性和 Trace 思维。

**话术：**  
我会看完整 Agent Trace，而不是只看最终回答。一次请求里我会记录用户问题、system prompt 版本、模型返回的 tool_calls、每次工具参数、ES DSL 或 `knn` 查询、返回 chunk、工具耗时、Token 使用量、最终引用 ID。如果答案错了，我先判断是检索没召回、召回了但排序错、上下文太长被模型忽略，还是模型总结时幻觉。比如 ES 没返回相关 chunk，那问题在检索或分块；如果返回了正确 chunk 但答案没引用，那可能是 prompt 或生成阶段问题。作为实习项目，我没有做特别完整的观测平台，但至少把这些日志结构化打出来，能复现一次 Agent 执行。总结来说，Agent 问题必须按链路拆，不然很难定位。

## 六、评估、成本与线上质量

### 23. 怎么评估 Agent 是否比原 RAG 更好？

**考察点：** 是否会量化，不是只说“感觉更智能”。

**话术：**  
我会用自己构造的评测集来对比，而不是只看几个 demo。评测集可以分成简单知识问答、多文档问题、检索失败问题、需要工具调用的问题、无答案问题。每类准备一些样例，分别跑原 RAG 和 Agent，看命中率、引用正确率、平均响应时间、平均工具调用次数和失败原因。比如简单问题，如果 Agent 比 RAG 慢很多但答案没提升，那就不应该走 Agent。复杂问题则重点看是否能通过多步检索或 query rewrite 找到答案。评价时我不会只用大模型打分，还会人工检查引用 chunk 是否真的支持答案。总结来说，Agent 不是所有指标都赢，它应该在复杂任务上提升，而简单任务要保持 RAG 快路径。

### 24. Agent 链路变长，延迟怎么优化？

**考察点：** 是否理解延迟来自模型、检索、工具和串行步骤。

**话术：**  
我主要从四个方向优化。第一是路由，简单问题不进 Agent，直接走 RAG；只有需要多步推理或工具调用才进入 Agent。第二是流式输出，用 WebFlux 的 `Flux` 和 WebSocket 先推 token 或 step 状态，减少用户体感等待。第三是工具层优化，比如 ES 检索控制 `topK` 和 `num_candidates`，Redis 缓存短期检索结果，避免重复查。第四是限制 Agent step，比如最多 3 轮工具调用，单个工具用 `Mono.timeout` 控制时间。刚开始我以为优化模型速度最重要，后来发现无效工具调用和重复检索更浪费。总结来说，延迟优化不是单点优化，而是路由、流式、缓存、超时和 step 控制一起做。

### 25. Token 成本暴涨怎么解决？

**考察点：** 是否知道 Agent 的真实成本问题。

**话术：**  
Agent 比 RAG 更容易烧 Token，因为它会多轮调用模型，每次还要带历史上下文和工具结果。我做的第一件事是上下文裁剪，不把完整历史和完整文档都塞给模型，只传最近几轮摘要和 ES 返回的 top chunk。第二是工具结果结构化，比如只传 chunk 摘要、docId、页码和关键片段，不传整个文件。第三是路由，简单问题直接 RAG，不让 Agent 过度规划。第四是 Redis 缓存短期工具结果，同一个 session 里相同检索不重复消耗。DeepSeek 请求里也会设置合理的 `max_tokens`，避免无限生成。我测试时发现，Token 成本很多时候不是答案长，而是上下文塞太多。总结来说，成本控制的核心是少走 Agent、少传无效上下文、少重复调用。

### 26. Agent 输出不可控，怎么做安全边界？

**考察点：** 是否知道 AI 应用上线风险。

**话术：**  
我主要做了几类边界。第一是工具白名单，Agent 只能调用后端注册过的工具，不能自己构造任意接口。第二是参数校验，比如 searchMode、topK、docScope 都要校验，防止越权查文档。第三是最大 step 和超时控制，避免无限循环。第四是引用校验，知识库问题必须基于 ES chunk 回答，没有证据就不输出确定结论。第五是降级策略，DeepSeek 调用失败或者工具失败时，可以回到普通 RAG 或提示用户稍后重试。作为实习项目，我没有做高风险写操作工具，主要是只读查询类工具。总结来说，Agent 的自由度必须被后端规则框住，尤其是企业知识库场景，不能让模型决定安全边界。

## 七、Java 后端能力深挖

### 27. 这个项目哪里体现了 Java 后端能力？

**考察点：** 防止你变成“调 API 工程师”。

**话术：**  
我觉得主要体现在几个地方。第一是后端链路设计，Agent 执行器、Tool 接口、模型客户端、ES 检索服务、Redis 状态服务都做了分层，不是 Controller 里堆逻辑。第二是异步和流式处理，用 WebFlux 的 `Mono`、`Flux` 编排模型调用、工具调用和 WebSocket 推送。第三是中间件落地，ES 里设计 chunk 索引和 DSL/`knn` 查询，Redis 设计 Key 过期策略，MinIO 负责原始文件存储。第四是异常处理，外部 API 超时、工具失败、检索为空都有结构化错误和 fallback。第五是日志和 Trace，能定位一次 Agent 失败在哪一步。总结来说，这个项目不是只会调用 DeepSeek，而是把 AI 能力接进 Java 后端工程链路里。

### 28. WebFlux 项目里异常怎么统一处理？

**考察点：** 是否知道响应式异常不能只靠 try-catch。

**话术：**  
在 WebFlux 里，很多异常发生在异步链路中，普通 try-catch 不一定能覆盖。所以我在工具调用和模型调用层会用 `onErrorResume` 把异常转换成统一的结果，比如 `ToolResult.failed(code, message)`。比如 DeepSeek 超时就是 `MODEL_TIMEOUT`，ES 查询异常是 `SEARCH_ERROR`，参数错误是 `INVALID_ARGUMENT`。这样 Agent 执行器拿到的是结构化 Observation，而不是直接抛异常导致 WebSocket 断掉。Controller 或 WebSocket 层也会把错误事件推给前端，而不是让连接莫名结束。对于同步参数校验，还是可以提前抛业务异常。总结来说，响应式异常处理要沿着 `Mono`/`Flux` 链路传递，让失败变成 Agent 可理解的状态。

### 29. 你怎么处理权限过滤？

**考察点：** 企业知识库绕不开多用户和越权问题。

**话术：**  
我的思路是权限过滤必须在 ES 查询阶段做，而不是检索回来后再让模型判断。每个 chunk 写入 ES 时会带上权限相关元数据，比如用户、组织、知识库 ID 或文档可见范围。检索时，DSL 或 `knn` query 里会加 filter，只允许查当前用户有权限的文档。这样 Agent 拿到的 Observation 本身就是过滤后的结果。DeepSeek 不知道用户权限，也不能让它决定能不能看某个文件。Redis 缓存工具结果时，Key 里也要带 userId 或 orgId，避免不同用户复用同一个检索缓存造成越权。我测试时特别注意这个点，因为 Agent 比普通搜索更容易把多个工具结果组合起来。总结来说，权限要在后端和 ES 层解决，不能交给大模型。

### 30. 如果面试官问你和 Dify / LangChain 有什么区别，怎么答？

**考察点：** 是否了解行业工具，也能解释自己项目价值。

**话术：**  
我会先承认 Dify、LangChain 这类框架更成熟，内置了很多 Agent、RAG、工作流能力。我这个项目不是要重复造一个完整平台，而是为了学习和落地 Java 后端里的 Agent 核心链路。我自己实现的部分更偏底层工程：Tool 接口怎么抽象、DeepSeek 的 `tool_choice` 怎么接、ES 的 DSL 和 `knn` 怎么融合、WebFlux/WebSocket 怎么流式推送、Redis 怎么存短期 step 状态。用现成框架当然更快，但很多细节容易变成配置黑盒。自己改一遍后，我能讲清楚一次 Agent 从用户问题到工具调用、再到 Observation 和最终回答的完整过程。总结来说，我不是说自己比框架强，而是这个项目能证明我理解 AI 应用的后端落地细节。

## 八、复盘与实习生人设题

### 31. 从 RAG 到 Agent，你踩过最大的坑是什么？

**考察点：** 是否有真实实践，不是包装项目。

**话术：**  
我踩过最大的坑是，一开始我以为 Agent 一定比 RAG 强，所以把很多问题都交给 Agent 处理。后来测试发现，简单问题反而变慢了，因为 Agent 会先分析、再决定工具、再总结，链路比普通 RAG 长很多，而且 Token 消耗也上去了。后来我做了一个比较实用的改动：先做问题路由，能一次 ES 检索回答的就走 RAG；只有跨文档、检索失败、需要外部工具时才进入 Agent。同时我限制最大 step 数，并用 Redis 缓存短期工具结果，减少重复检索。这个坑让我意识到，Agent 不是替代 RAG，而是补充 RAG 的复杂任务能力。总结来说，我现在会更强调边界控制，而不是所有问题都 Agent 化。

### 32. 如果让你继续优化这个项目，你会优先做什么？

**考察点：** 是否有后续思考和工程判断。

**话术：**  
如果继续优化，我会优先做评测集和 Trace 可视化。现在很多 AI 项目容易只看 demo，真正面试或上线时说不清效果。我会把问题分成简单问答、多文档、多跳推理、检索失败、无答案问题几类，分别统计 RAG 和 Agent 的准确率、引用正确率、平均延迟、平均工具调用次数。Trace 可视化则是为了看每次 Agent 为什么调用某个工具、ES 查了什么 DSL 或 `knn`、返回了哪些 chunk、最终答案引用了哪些证据。第二步我会优化重排，比如在 ES 初召回后加一个 rerank 或轻量打分规则。总结来说，我下一步不会急着加更多工具，而是先把效果评估和问题定位做好。

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
- 工具结果缓存 Key 怎么设计，如何避免不同用户串数据？
- Redis 里的短期上下文和 MySQL 的历史记录有什么区别？
- MinIO 存原文件，ES 存 chunk，MySQL 存什么？
- 文件上传成功但解析失败怎么办？
- 解析任务是同步还是异步？为什么？
- 如果用户上传大文件，怎么避免请求阻塞？
- 文档解析状态有哪些？

### DeepSeek / 大模型 API

- `tool_choice=auto` 和指定工具有什么区别？
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
- 不要把 Redis 说成长期记忆库，Redis 在这里更适合短期上下文、状态和去重。
- 不要说 WebFlux 一定比 Spring MVC 快，应该说它更适合 IO 多、长连接、流式返回的场景。
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
- Spring WebFlux WebSocket 官方文档  
  https://docs.spring.io/spring-framework/reference/web/webflux-websocket.html
- Elasticsearch kNN Query 官方文档  
  https://www.elastic.co/docs/reference/query-languages/query-dsl/query-dsl-knn-query
- DeepSeek Function Calling 官方文档  
  https://api-docs.deepseek.com/guides/function_calling
- DeepSeek Chat Completion / tool_choice 官方文档  
  https://api-docs.deepseek.com/api/create-chat-completion
- Redis EXPIRE 官方文档  
  https://redis.io/docs/latest/commands/expire/
