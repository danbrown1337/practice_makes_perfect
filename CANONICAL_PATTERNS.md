# Canonical AICE Agent Patterns

> Reference document derived from `aice-studio-cookbooks`. Use as a sanity
> check for current state and all future changes. If a PR diverges from these
> patterns without documented justification, push back.

---

## 1. Authentication & LLM Construction

**Pattern:** Single cached credential → token provider → injected into every LLM/embeddings client.

```python
from azure.identity import DefaultAzureCredential, get_bearer_token_provider

credential = DefaultAzureCredential()
token_provider = get_bearer_token_provider(
    credential, "https://cognitiveservices.azure.com/.default"
)

llm = AzureChatOpenAI(
    azure_deployment="gpt-4o",
    azure_ad_token_provider=token_provider,
    api_version="2025-04-01-preview",
    temperature=0,
)
```

**Rules:**
- Never use API keys. Always AAD tokens.
- `DefaultAzureCredential` (no pinned auth method — chain selects env/CLI/managed-identity).
- Token provider constructed once, reused everywhere.
- Temperature defaults to `0` for deterministic tasks; caller overrides for creative ones.

---

## 2. Typed State Schema

**Pattern:** A Pydantic `BaseModel` with `Annotated[list, add_messages]` for automatic message accumulation.

```python
from typing import Annotated
from langgraph.graph.message import add_messages
from pydantic import BaseModel

class State(BaseModel):
    messages: Annotated[list, add_messages]
```

**Rules:**
- State is a `BaseModel`, not a dict.
- `add_messages` ensures nodes *append* rather than overwrite conversation history.
- Domain-specific fields go on the same class (e.g. `doc_hashes`, `mode`).
- State is the single data contract flowing through the graph.

---

## 3. Context Schema (Runtime Parameters)

**Pattern:** A separate `Context(BaseModel)` carrying config that can be overridden per-request without touching the state.

```python
from typing import Optional
import os

class Context(BaseModel):
    model: Optional[str] = "gpt-4o"
    temperature: float = 0
    api_version: Optional[str] = os.environ.get("OPENAI_API_VERSION", "2025-04-01-preview")
```

**Rules:**
- Context holds *deployment-time knobs*, not stateful data.
- Passed via `graph.ainvoke(payload, context={...})`.
- Access inside nodes via `get_runtime().context`.
- Defaults match your env; overrides are for benchmarking / A-B testing.

---

## 4. Graph Nodes

**Pattern:** Each node is a function `(state, config) → partial state update`. Always provide sync + async variants.

```python
from langchain_core.messages import SystemMessage
from langchain_core.runnables import RunnableConfig
from langgraph.utils.runnable import RunnableCallable

def chatbot(state: State, config: RunnableConfig):
    llm = create_llm(config)
    messages = [SystemMessage("You are a helpful Chatbot."), *state.messages]
    response = llm.invoke(messages)
    return {"messages": [response]}

async def achatbot(state: State, config: RunnableConfig):
    llm = create_llm(config)
    messages = [SystemMessage("You are a helpful Chatbot."), *state.messages]
    response = await llm.ainvoke(messages)
    return {"messages": [response]}

# Register with dual sync/async support:
graph.add_node("chatbot", RunnableCallable(chatbot, achatbot), input_schema=State)
```

**Rules:**
- Nodes return a *partial* state dict — only the keys they modify.
- System prompt prepended inside the node, not globally.
- LLM constructed fresh per call (picks up runtime context).
- `RunnableCallable(sync, async)` enables both `invoke` and `ainvoke`.

---

## 5. Tool Definition & Registration

**Pattern:** Functions decorated with `@tool`, bound to the LLM, executed by a `ToolNode`.

```python
from langchain.tools import tool
from langgraph.prebuilt import ToolNode

@tool
def search_documents(query: str) -> str:
    """Search the document store for relevant chunks.
    
    Use this when the user asks a question about a specific document.
    """
    chunks = embeddings.search(query, doc_hashes, parser_mode, top_k=8)
    return "\n\n".join(c.text for c in chunks)

# Bind to LLM so it knows the tool exists:
llm_with_tools = llm.bind_tools([search_documents])

# Register the executor node:
graph.add_node("tools_node", ToolNode([search_documents]))
```

**Rules:**
- Docstring is critical — the LLM reads it to decide when to use the tool.
- Args must be typed and descriptive.
- `bind_tools()` tells the LLM the tool schema; `ToolNode` executes the call.
- Tools are pure functions — no side effects on state except via return value.

---

## 6. Conditional Routing

**Pattern:** A router function that inspects state and returns the next node name.

```python
from langgraph.graph import END

def router(state: State) -> str:
    """Route to tools if the LLM made tool calls, else end."""
    if state.messages[-1].tool_calls:
        return "tools_node"
    return END
```

```python
graph.add_conditional_edges("chatbot", router)
graph.add_edge("tools_node", "chatbot")  # after tool execution, loop back
```

**Rules:**
- Router returns a node name (string) or `END`.
- Tool loop: `chatbot → tools_node → chatbot → ... → END`.
- No arbitrary Python control flow — routing is declarative via the graph.

---

## 7. Graph Assembly

**Pattern:** One `load_graph()` function that wires everything and returns a compiled graph.

```python
from langgraph.graph import END, StateGraph
from langgraph.graph.state import CompiledStateGraph

def load_graph() -> CompiledStateGraph:
    graph = StateGraph(state_schema=State, context_schema=Context)
    
    graph.add_node("chatbot", RunnableCallable(chatbot, achatbot), input_schema=State)
    graph.add_node("tools_node", ToolNode([search_documents]))
    
    graph.set_entry_point("chatbot")
    graph.add_edge("tools_node", "chatbot")
    graph.add_conditional_edges("chatbot", router)
    
    return graph.compile()
```

**Rules:**
- `state_schema` and `context_schema` explicitly typed.
- Entry point is always the LLM node (agent decides what to do).
- `.compile()` validates the graph at build time.
- Cache the compiled graph at module level; `invalidate_graph()` to rebuild.

---

## 8. RAG (Retrieval Augmented Generation)

**Pattern:** FAISS vectorstore + Azure embeddings + retriever exposed as a tool.

```python
from langchain.vectorstores import FAISS
from langchain_openai import AzureOpenAIEmbeddings
from langchain_text_splitters import RecursiveCharacterTextSplitter

# Build:
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
doc_splits = text_splitter.split_documents(documents)
vectorstore = FAISS.from_documents(doc_splits, embedding=AzureOpenAIEmbeddings(...))

# Expose as tool:
retriever_tool = vectorstore.as_retriever().as_tool(
    name="knowledge_base",
    description="Search the knowledge base for relevant information.",
)
```

**Rules:**
- Chunk size 1000, overlap 200 is the default starting point.
- FAISS for local/transient indexes (no external vector DB needed).
- Retriever wrapped as tool = agent decides when to search.
- `save_local()` / `load_local()` for persistence across runs.

---

## 9. MLflow Model Logging & Tracing

**Pattern:** Autolog for tracing; `ResponsesAgent` wrapper for deployment.

```python
import mlflow

# Tracing (add at app startup — gives you visibility for free):
mlflow.set_tracking_uri("databricks")
mlflow.set_experiment("my-experiment")
mlflow.langchain.autolog()

# Model logging (for deployment to AICE Space):
mlflow.pyfunc.log_model(
    name="my-agent",
    python_model="agent.py",
    input_example={"input": [{"role": "user", "content": "Hello"}], "metadata": params},
    streamable=True,
)
```

**Rules:**
- `mlflow.langchain.autolog()` captures every LLM call, tool call, and token.
- Model-from-code: agent logic lives in a standalone `.py` file.
- `ResponsesAgent` provides `predict` and `predict_stream` interface.
- Input example must include `metadata` for runtime context params.

---

## 10. Streaming

**Pattern:** Stream via `graph.stream(..., stream_mode=["updates"])`, yield events.

```python
from mlflow.pyfunc import ResponsesAgent
from mlflow.types.responses import (
    ResponsesAgentStreamEvent,
    output_to_responses_items_stream,
    to_chat_completions_input,
)

class LangGraphResponsesAgent(ResponsesAgent):
    def predict_stream(self, request):
        cc_msgs = to_chat_completions_input([i.model_dump() for i in request.input])
        for _, events in self.agent.stream(
            {"messages": cc_msgs},
            context=request.metadata,
            stream_mode=["updates"],
        ):
            for node_data in events.values():
                yield from output_to_responses_items_stream(node_data["messages"])
```

**Rules:**
- `stream_mode=["updates"]` yields per-node outputs as they complete.
- Each yield is a `ResponsesAgentStreamEvent`.
- Caller receives tokens incrementally — no waiting for full response.

---

## 11. MCP Tool Integration

**Pattern:** Connect to external tool servers dynamically over SSE.

```python
from langchain_mcp_adapters.client import MultiServerMCPClient
from asyncache import cached
from cachetools import TTLCache

mcp_client = MultiServerMCPClient({
    "calculator": {"url": "http://localhost:8008/sse/", "transport": "sse"}
})

@cached(TTLCache(ttl=300, maxsize=1))
async def get_tools():
    return await mcp_client.get_tools()
```

**Rules:**
- Tools are fetched dynamically (server can add/remove tools at runtime).
- Cache with TTL to avoid hitting the server every request.
- Bind dynamically-fetched tools to LLM just like local `@tool` functions.

---

## 12. Prompty Format

**Pattern:** YAML front-matter + Jinja-style template body.

```yaml
---
name: Basic Prompt
description: A prompt that uses context to ground an incoming question
model:
  api: chat
  configuration:
    type: azure_openai
    azure_deployment: gpt-4o-mini
  parameters:
    max_tokens: 3000
inputs:
  question:
    type: string
    description: The user question.
---

system:
You are an AI assistant who helps people.

user:
{{question}}
```

**Rules:**
- Metadata (model, parameters, inputs) is machine-readable.
- Template uses `{{variable}}` placeholders.
- Version-controlled alongside code.

---

## 13. Benchmarking & Evaluation

**Pattern:** Eval dataset + scorer + `AICEBenchmarker`.

```python
from aice.benchmarker import AICEBenchmarker
from mlflow.genai.scorers import Correctness

eval_dataset = [
    {"inputs": {"question": "What is X?"}, "expectations": {"expected_response": "Y"}},
]

def wrapper_fn(model, question, **kwargs) -> str:
    result = model.invoke({"messages": [{"role": "user", "content": question}]}, context=kwargs)
    return result["messages"][-1].content

controller = AICEBenchmarker(
    experiment_name="my-eval",
    n_trials=3,
    eval_dataset=eval_dataset,
    scorers=[Correctness()],
    predict_fn=predict_fn,
)
evaluations = controller.run_experiment()
```

**Rules:**
- Dataset has `inputs` (what you ask) and `expectations` (ground truth).
- Wrapper function adapts your agent's output to a plain string for scoring.
- `n_trials` > 1 for statistical confidence.
- Results logged to MLflow for comparison across agent versions.

---

## 14. Dependency & Tooling Standards

| Tool | Standard |
|------|----------|
| Python | ≥ 3.11 |
| Package manager | `uv` with `pyproject.toml` + `uv.lock` |
| Linter | Ruff (`F, W, E, I, UP, PL, T201`), line-length 100 |
| Formatter | Ruff format (or Black) |
| Type checking | Pyright / mypy optional but encouraged |
| Tests | pytest |

---

## Quick Compliance Checklist

Use this before merging any PR:

- [ ] LLM uses `AzureChatOpenAI` + `get_bearer_token_provider` (no API keys)
- [ ] State is a typed `BaseModel` with `add_messages`
- [ ] Nodes are pure functions `(state, config) → partial update`
- [ ] Tools use `@tool` decorator with typed args + docstring
- [ ] Graph has conditional routing (not hard-coded if/else)
- [ ] `mlflow.langchain.autolog()` is enabled
- [ ] Prompt changes are version-controlled (not inline strings)
- [ ] New features have eval dataset entries
- [ ] No new `pip install` without `pyproject.toml` update
- [ ] Ruff passes clean
