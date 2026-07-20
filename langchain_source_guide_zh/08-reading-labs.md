# 08. 源码阅读实验

所有实验都以“观察对象类型、命中断点、解释调用栈”为目标。先在 `libs/core` 完成环境同步；Agent 实验在 `libs/langchain_v1` 执行。

## 实验 1：看见 RunnableSequence

```python
from langchain_core.runnables import RunnableLambda

add = RunnableLambda(lambda x: x + 1)
double = RunnableLambda(lambda x: x * 2)
chain = add | double

print(type(chain))
print([type(step).__name__ for step in chain.steps])
print(chain.invoke(3))
print(chain.get_graph().draw_mermaid())
```

断点：[`Runnable.__or__`](../libs/core/langchain_core/runnables/base.py)、[`RunnableSequence.invoke`](../libs/core/langchain_core/runnables/base.py)。

应该能回答：

1. `|` 返回了什么类型？
2. `input_` 在每次循环后是什么值？
3. 为什么两个 lambda 都有 child run？

## 实验 2：跟踪 Prompt → Model → Parser

使用 [01 文档](01-local-run-and-debug.md) 的 Fake Model 脚本，额外打印中间结果：

```python
prompt_value = prompt.invoke({"topic": "Runnable"})
print(type(prompt_value), prompt_value)
print(prompt_value.to_messages())

message = model.invoke(prompt_value)
print(type(message), message)

text = StrOutputParser().invoke(message)
print(type(text), text)
```

断点顺序：

1. [`BasePromptTemplate._validate_input`](../libs/core/langchain_core/prompts/base.py)
2. [`BaseChatModel._convert_input`](../libs/core/langchain_core/language_models/chat_models.py)
3. [`BaseChatModel.generate`](../libs/core/langchain_core/language_models/chat_models.py)
4. [`FakeListChatModel._call`](../libs/core/langchain_core/language_models/fake_chat_models.py)
5. [`BaseOutputParser.invoke`](../libs/core/langchain_core/output_parsers/base.py)

故意把输入改为 `{}`，观察缺少 `topic` 时在哪一层抛错。

## 实验 3：同步、批量、异步、流式

```python
import asyncio

from langchain_core.language_models.fake_chat_models import FakeListChatModel

model = FakeListChatModel(responses=["abc", "xyz"])

print(model.invoke("one"))
print(model.batch(["two", "three"]))

async def main() -> None:
    print(await model.ainvoke("four"))
    async for chunk in model.astream("five"):
        print(repr(chunk.content))

asyncio.run(main())
```

对照：[`Runnable.ainvoke/batch`](../libs/core/langchain_core/runnables/base.py)、[`FakeListChatModel.batch/_stream/_astream`](../libs/core/langchain_core/language_models/fake_chat_models.py)。

应该能回答：Fake 模型为什么覆盖 `batch`？流式为什么逐字符产生 `AIMessageChunk`？

## 实验 4：Tool schema 与执行

```python
from langchain_core.tools import tool

@tool
def divide(a: float, b: float) -> float:
    """Divide a by b."""
    return a / b

print(type(divide))
print(divide.name)
print(divide.args_schema.model_json_schema())
print(divide.invoke({"a": 10, "b": 2}))
```

依次尝试：缺少 `b`、把 `a` 写成无法转换的字符串、让 `b=0`。

断点：

- [`tool`](../libs/core/langchain_core/tools/convert.py)
- [`create_schema_from_function`](../libs/core/langchain_core/tools/base.py)
- [`BaseTool._parse_input`](../libs/core/langchain_core/tools/base.py)
- [`BaseTool.run`](../libs/core/langchain_core/tools/base.py)
- [`StructuredTool._run`](../libs/core/langchain_core/tools/structured.py)

区分 Pydantic validation error 与函数内部 `ZeroDivisionError` 的处理路径。

## 实验 5：无网络 VectorStore 与 Retriever

```python
from langchain_core.documents import Document
from langchain_core.embeddings import DeterministicFakeEmbedding
from langchain_core.vectorstores import InMemoryVectorStore

embedding = DeterministicFakeEmbedding(size=8)
store = InMemoryVectorStore(embedding=embedding)
store.add_documents(
    [
        Document(page_content="Python uses indentation.", metadata={"id": 1}),
        Document(page_content="LangChain components implement Runnable.", metadata={"id": 2}),
        Document(page_content="Vector stores support similarity search.", metadata={"id": 3}),
    ]
)

retriever = store.as_retriever(search_kwargs={"k": 2})
for doc in retriever.invoke("Runnable protocol"):
    print(doc.metadata, doc.page_content)
```

Fake embedding 只适合观察流程，不代表语义质量。断点：

1. [`InMemoryVectorStore.add_documents`](../libs/core/langchain_core/vectorstores/in_memory.py)
2. [`BaseRetriever.invoke`](../libs/core/langchain_core/retrievers.py)
3. [`VectorStoreRetriever._get_relevant_documents`](../libs/core/langchain_core/vectorstores/base.py)
4. [`InMemoryVectorStore.similarity_search_with_score`](../libs/core/langchain_core/vectorstores/in_memory.py)
5. [`_similarity_search_with_score_by_vector`](../libs/core/langchain_core/vectorstores/in_memory.py)

## 实验 6：打印 Agent 构图结果（无网络）

这个实验借用仓库测试专用模型，只用于源码学习，不要在业务代码依赖 `tests`：

```python
from langchain.agents import create_agent
from langchain_core.tools import tool
from tests.unit_tests.agents.model import FakeToolCallingModel

@tool
def echo(text: str) -> str:
    """Echo text."""
    return text

model = FakeToolCallingModel(
    tool_calls=[
        [{"name": "echo", "args": {"text": "hello"}, "id": "call-1"}],
        [],
    ]
)
agent = create_agent(model=model, tools=[echo])

print(agent.get_graph().draw_mermaid())
result = agent.invoke({"messages": [{"role": "user", "content": "echo hello"}]})
for message in result["messages"]:
    print(type(message).__name__, message)
```

在 `libs/langchain_v1` 下运行，测试模型源码是 [`tests/unit_tests/agents/model.py`](../libs/langchain_v1/tests/unit_tests/agents/model.py)。

断点：

- [`create_agent`](../libs/langchain_v1/langchain/agents/factory.py)
- `create_agent` 内的 model node/执行函数
- [`_make_model_to_tools_edge`](../libs/langchain_v1/langchain/agents/factory.py)
- [`BaseTool.run`](../libs/core/langchain_core/tools/base.py)
- [`_make_tools_to_model_edge`](../libs/langchain_v1/langchain/agents/factory.py)

最终消息类型应呈现 Human → AI(tool call) → Tool → AI 的基本结构。

## 实验 7：观察 Callback 树

```python
from langchain_core.tracers import ConsoleCallbackHandler

config = {
    "run_name": "callback-lab",
    "tags": ["source-study"],
    "metadata": {"lab": 7},
    "callbacks": [ConsoleCallbackHandler()],
}
print(chain.invoke({"topic": "callbacks"}, config=config))
```

断点：[`ensure_config`](../libs/core/langchain_core/runnables/config.py)、[`patch_config`](../libs/core/langchain_core/runnables/config.py)、[`CallbackManager.on_chain_start`](../libs/core/langchain_core/callbacks/manager.py)。

画出实际看到的 `parent_run_id → run_id` 树，并把每个 child 对应到 Prompt/Model/Parser。

## 实验 8：验证缓存分支

```python
from langchain_core.caches import InMemoryCache
from langchain_core.globals import set_llm_cache
from langchain_core.language_models.fake_chat_models import FakeListChatModel

set_llm_cache(InMemoryCache())
model = FakeListChatModel(responses=["first", "second"])

print(model.invoke("same input").content)
print(model.invoke("same input").content)
print(model.invoke("different input").content)
```

在 [`BaseChatModel._generate_with_cache`](../libs/core/langchain_core/language_models/chat_models.py) 观察 `prompt`、`llm_string` 和 `cache_val`。解释为什么第二次相同输入没有消费下一条 fake response。

## 实验完成标准

不要以“代码跑通”为完成，而要能用自己的话回答：

- 公共 `invoke` 和受保护模板方法分别负责什么？
- `|` 何时创建 Sequence，dict 何时创建 Parallel？
- config 为什么能自动传到子调用？
- Model、Tool、Retriever 的回调生命周期有哪些共同结构？
- Agent 为什么必须用条件图，而非 Sequence？
- RAG 为什么必须分索引阶段和查询阶段？

