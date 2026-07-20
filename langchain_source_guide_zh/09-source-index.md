# 09. 源码符号速查表

阅读方法：先搜“公共入口”，再搜“模板方法”，最后进入一个“具体实现”。

## Runnable / LCEL

| 主题 | 公共入口 | 关键实现 |
|---|---|---|
| 统一协议 | [`Runnable`](../libs/core/langchain_core/runnables/base.py) | `invoke/ainvoke/batch/stream` |
| `|` 组合 | [`Runnable.__or__`](../libs/core/langchain_core/runnables/base.py) | `RunnableSequence.__init__/__or__` |
| 类型适配 | [`coerce_to_runnable`](../libs/core/langchain_core/runnables/base.py) | callable→`RunnableLambda`，mapping→`RunnableParallel` |
| 顺序 | [`RunnableSequence`](../libs/core/langchain_core/runnables/base.py) | `invoke/_transform/batch` |
| 并行 | [`RunnableParallel`](../libs/core/langchain_core/runnables/base.py) | `invoke/ainvoke/transform` |
| 函数包装 | [`RunnableLambda`](../libs/core/langchain_core/runnables/base.py) | `invoke/_invoke` |
| 透传/字段扩展 | [`runnables/passthrough.py`](../libs/core/langchain_core/runnables/passthrough.py) | `RunnablePassthrough/Assign/Pick` |
| 配置 | [`RunnableConfig`](../libs/core/langchain_core/runnables/config.py) | `ensure_config/patch_config/merge_configs` |
| 动态配置 | [`runnables/configurable.py`](../libs/core/langchain_core/runnables/configurable.py) | `RunnableConfigurableFields/Alternatives` |
| 重试/fallback | [`retry.py`](../libs/core/langchain_core/runnables/retry.py) | [`fallbacks.py`](../libs/core/langchain_core/runnables/fallbacks.py) |
| 图展示 | [`runnables/graph.py`](../libs/core/langchain_core/runnables/graph.py) | [`graph_mermaid.py`](../libs/core/langchain_core/runnables/graph_mermaid.py) |

## Prompt / Message / Output

| 主题 | 抽象/数据 | 具体实现 |
|---|---|---|
| Prompt 基类 | [`BasePromptTemplate`](../libs/core/langchain_core/prompts/base.py) | `invoke/_validate_input/format_prompt` |
| 文本 Prompt | [`PromptTemplate`](../libs/core/langchain_core/prompts/prompt.py) | `format/from_template` |
| 聊天 Prompt | [`ChatPromptTemplate`](../libs/core/langchain_core/prompts/chat.py) | `format_messages/from_messages` |
| Prompt 值 | [`prompt_values.py`](../libs/core/langchain_core/prompt_values.py) | `StringPromptValue/ChatPromptValue` |
| Message 基类 | [`BaseMessage`](../libs/core/langchain_core/messages/base.py) | content/additional_kwargs/metadata |
| AI 消息 | [`AIMessage`](../libs/core/langchain_core/messages/ai.py) | tool_calls/usage_metadata |
| Tool 消息 | [`ToolMessage`](../libs/core/langchain_core/messages/tool.py) | tool_call_id/status/artifact |
| 消息转换 | [`messages/utils.py`](../libs/core/langchain_core/messages/utils.py) | `convert_to_messages/message_chunk_to_message` |
| 生成结果 | [`outputs`](../libs/core/langchain_core/outputs) | `Generation/ChatGeneration/ChatResult/LLMResult` |

## Model / Provider

| 主题 | 公共入口 | 模板/具体实现 |
|---|---|---|
| 语言模型基类 | [`BaseLanguageModel`](../libs/core/langchain_core/language_models/base.py) | token/序列化/通用属性 |
| Chat 模型 | [`BaseChatModel`](../libs/core/langchain_core/language_models/chat_models.py) | `invoke/generate/_generate_with_cache/_generate` |
| 流式聚合 | [`chat_models.py`](../libs/core/langchain_core/language_models/chat_models.py) | `stream/generate_from_stream` |
| 模型初始化 | [`init_chat_model`](../libs/langchain_v1/langchain/chat_models/base.py) | `_parse_model/_init_chat_model_helper` |
| OpenAI 模型 | [`BaseChatOpenAI/ChatOpenAI`](../libs/partners/openai/langchain_openai/chat_models/base.py) | `_generate/_stream/_get_request_payload/_create_chat_result` |
| OpenAI embeddings | [`embeddings/base.py`](../libs/partners/openai/langchain_openai/embeddings/base.py) | `OpenAIEmbeddings` |

## Parser / Structured Output

| 主题 | 入口 | 实现点 |
|---|---|---|
| Parser 基类 | [`BaseOutputParser`](../libs/core/langchain_core/output_parsers/base.py) | `invoke/parse_result/parse` |
| 字符串 | [`StrOutputParser`](../libs/core/langchain_core/output_parsers/string.py) | `parse` |
| JSON | [`JsonOutputParser`](../libs/core/langchain_core/output_parsers/json.py) | partial JSON/累计流 |
| Pydantic | [`PydanticOutputParser`](../libs/core/langchain_core/output_parsers/pydantic.py) | schema validation |
| Tool call parser | [`openai_tools.py`](../libs/core/langchain_core/output_parsers/openai_tools.py) | `parse_tool_call/JsonOutputToolsParser` |
| Agent 输出策略 | [`agents/structured_output.py`](../libs/langchain_v1/langchain/agents/structured_output.py) | `ToolStrategy/ProviderStrategy/AutoStrategy` |

## Tool / Agent / Middleware

| 主题 | 公共入口 | 实现点 |
|---|---|---|
| Tool 基类 | [`BaseTool`](../libs/core/langchain_core/tools/base.py) | `invoke/run/_run/_parse_input` |
| 函数转 Tool | [`tool`](../libs/core/langchain_core/tools/convert.py) | `StructuredTool.from_function` |
| 函数 Tool | [`StructuredTool`](../libs/core/langchain_core/tools/structured.py) | `_run/_arun` |
| Retriever Tool | [`create_retriever_tool`](../libs/core/langchain_core/tools/retriever.py) | 文档格式化/返回 artifact |
| Agent 构造 | [`create_agent`](../libs/langchain_v1/langchain/agents/factory.py) | StateGraph nodes/edges/compile |
| model→tools | [`_make_model_to_tools_edge`](../libs/langchain_v1/langchain/agents/factory.py) | pending tool calls/退出条件 |
| tools→model | [`_make_tools_to_model_edge`](../libs/langchain_v1/langchain/agents/factory.py) | return_direct/结构化输出 |
| Agent 状态 | [`AgentState`](../libs/langchain_v1/langchain/agents/middleware/types.py) | messages/jump_to/structured_response |
| Middleware | [`AgentMiddleware`](../libs/langchain_v1/langchain/agents/middleware/types.py) | before/after/wrap hooks |
| ToolNode 边界 | [`langchain/tools/tool_node.py`](../libs/langchain_v1/langchain/tools/tool_node.py) | 主实现来自 `langgraph.prebuilt` |

## RAG / Retrieval

| 主题 | 抽象 | 具体/入口 |
|---|---|---|
| 文档 | [`Document`](../libs/core/langchain_core/documents/base.py) | page_content/metadata/id |
| Loader | [`BaseLoader`](../libs/core/langchain_core/document_loaders/base.py) | `load/lazy_load/alazy_load` |
| Splitter | [`TextSplitter`](../libs/text-splitters/langchain_text_splitters/base.py) | `split_text/create_documents/split_documents` |
| Embeddings | [`Embeddings`](../libs/core/langchain_core/embeddings/embeddings.py) | embed_documents/embed_query |
| VectorStore | [`VectorStore`](../libs/core/langchain_core/vectorstores/base.py) | add/search/delete/as_retriever |
| 内存向量库 | [`InMemoryVectorStore`](../libs/core/langchain_core/vectorstores/in_memory.py) | cosine/top-k/MMR |
| Retriever | [`BaseRetriever`](../libs/core/langchain_core/retrievers.py) | `invoke/_get_relevant_documents` |
| VS Retriever | [`VectorStoreRetriever`](../libs/core/langchain_core/vectorstores/base.py) | similarity/threshold/mmr |
| 增量索引 | [`indexing/api.py`](../libs/core/langchain_core/indexing/api.py) | `index/aindex` |
| 记录管理 | [`RecordManager`](../libs/core/langchain_core/indexing/base.py) | exists/update/delete_keys |

## Runtime / Observability / Persistence

| 主题 | 入口 | 实现点 |
|---|---|---|
| callbacks 接口 | [`callbacks/base.py`](../libs/core/langchain_core/callbacks/base.py) | `BaseCallbackHandler` + mixins |
| callback 管理 | [`callbacks/manager.py`](../libs/core/langchain_core/callbacks/manager.py) | `CallbackManager` + RunManagers |
| 控制台 trace | [`tracers/stdout.py`](../libs/core/langchain_core/tracers/stdout.py) | `ConsoleCallbackHandler` |
| LangSmith trace | [`tracers/langchain.py`](../libs/core/langchain_core/tracers/langchain.py) | `LangChainTracer` |
| event stream | [`tracers/event_stream.py`](../libs/core/langchain_core/tracers/event_stream.py) | tap output/过滤/事件构造 |
| cache | [`caches.py`](../libs/core/langchain_core/caches.py) | `BaseCache/InMemoryCache` |
| rate limit | [`rate_limiters.py`](../libs/core/langchain_core/rate_limiters.py) | `BaseRateLimiter/InMemoryRateLimiter` |
| 序列化 | [`Serializable`](../libs/core/langchain_core/load/serializable.py) | `to_json/lc_id/lc_secrets` |
| dump/load | [`load/dump.py`](../libs/core/langchain_core/load/dump.py) | [`load/load.py`](../libs/core/langchain_core/load/load.py) |
| KV store | [`stores.py`](../libs/core/langchain_core/stores.py) | BaseStore/InMemoryStore |

## 测试导航

| 学习主题 | 测试目录/文件 |
|---|---|
| Runnable | [`core/tests/unit_tests/runnables`](../libs/core/tests/unit_tests/runnables) |
| Prompt | [`core/tests/unit_tests/prompts`](../libs/core/tests/unit_tests/prompts) |
| Messages | [`core/tests/unit_tests/messages`](../libs/core/tests/unit_tests/messages) |
| Tool | [`core/tests/unit_tests/test_tools.py`](../libs/core/tests/unit_tests/test_tools.py) |
| Retriever/VectorStore | [`core/tests/unit_tests/vectorstores`](../libs/core/tests/unit_tests/vectorstores) |
| Agent | [`langchain_v1/tests/unit_tests/agents`](../libs/langchain_v1/tests/unit_tests/agents) |
| Agent Middleware | [`agents/middleware`](../libs/langchain_v1/tests/unit_tests/agents/middleware) |
| OpenAI Chat | [`partners/openai/tests/unit_tests/chat_models`](../libs/partners/openai/tests/unit_tests/chat_models) |

## 一个固定的追踪模板

面对任何陌生模块，按这六步搜索：

```text
1. 从 __init__.py 确认公共导出
2. 找 Base*/ABC/Protocol
3. 找 invoke/run 等公共入口
4. 找 _generate/_run/_get_* 等模板方法
5. 找一个最简单具体子类
6. 找对应 unit test，断点验证
```

常用命令：

```bash
rg -n "^class Base|@abstractmethod|def invoke|def _run|def _generate" path/to/module
rg -n "SymbolName" tests libs
```

