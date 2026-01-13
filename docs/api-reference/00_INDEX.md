# LangGraph API Reference Index

Complete function and class documentation for the LangGraph codebase.

---

## Quick Navigation

| # | Document | Lines | Coverage |
|---|----------|-------|----------|
| 01 | [Graph API](01_GRAPH_API.md) | 1,743 | StateGraph, CompiledGraph, nodes, edges |
| 02 | [Pregel API](02_PREGEL_API.md) | 3,152 | Pregel, RemoteGraph, execution engine |
| 03 | [Channels API](03_CHANNELS_API.md) | 1,261 | All channel types and reducers |
| 04 | [Checkpoint API](04_CHECKPOINT_API.md) | 2,926 | Savers, stores, caches, serialization |
| 05 | [Prebuilt API](05_PREBUILT_API.md) | 1,389 | create_react_agent, ToolNode, validation |
| 06 | [SDK API](06_SDK_API.md) | 3,183 | Python client for LangGraph API |
| 07 | [CLI API](07_CLI_API.md) | 1,926 | Commands, config, Docker integration |
| 08 | [Functional API](08_FUNCTIONAL_API.md) | 1,527 | @task, @entrypoint decorators |
| 09 | [Types & Utilities](09_TYPES_UTILITIES_API.md) | 1,878 | Core types, errors, constants |

**Total: 18,985 lines of API documentation**

---

## By Category

### Graph Building

| Function/Class | Module | Description |
|----------------|--------|-------------|
| `StateGraph` | [Graph API](01_GRAPH_API.md#stategraph) | Main graph builder class |
| `StateGraph.add_node()` | [Graph API](01_GRAPH_API.md#add_node) | Add a node to the graph |
| `StateGraph.add_edge()` | [Graph API](01_GRAPH_API.md#add_edge) | Add an edge between nodes |
| `StateGraph.add_conditional_edges()` | [Graph API](01_GRAPH_API.md#add_conditional_edges) | Add conditional routing |
| `StateGraph.compile()` | [Graph API](01_GRAPH_API.md#compile) | Compile graph for execution |
| `CompiledStateGraph` | [Graph API](01_GRAPH_API.md#compiledstategraph) | Compiled executable graph |
| `MessagesState` | [Graph API](01_GRAPH_API.md#messagesstate) | Pre-built state for chat |
| `add_messages` | [Graph API](01_GRAPH_API.md#add_messages) | Message list reducer |

### Execution

| Function/Class | Module | Description |
|----------------|--------|-------------|
| `Pregel` | [Pregel API](02_PREGEL_API.md#pregel) | Core execution engine |
| `Pregel.invoke()` | [Pregel API](02_PREGEL_API.md#invoke) | Execute graph synchronously |
| `Pregel.stream()` | [Pregel API](02_PREGEL_API.md#stream) | Stream graph execution |
| `Pregel.get_state()` | [Pregel API](02_PREGEL_API.md#get_state) | Get current state |
| `Pregel.update_state()` | [Pregel API](02_PREGEL_API.md#update_state) | Modify state externally |
| `RemoteGraph` | [Pregel API](02_PREGEL_API.md#remotegraph) | Execute remote graphs |
| `ChannelRead` | [Pregel API](02_PREGEL_API.md#channelread) | Read from channels |
| `ChannelWrite` | [Pregel API](02_PREGEL_API.md#channelwrite) | Write to channels |

### Channels

| Class | Module | Description |
|-------|--------|-------------|
| `BaseChannel` | [Channels API](03_CHANNELS_API.md#basechannel) | Abstract base class |
| `LastValue` | [Channels API](03_CHANNELS_API.md#lastvalue) | Store last value |
| `Topic` | [Channels API](03_CHANNELS_API.md#topic) | Pub/sub accumulation |
| `BinaryOperatorAggregate` | [Channels API](03_CHANNELS_API.md#binaryoperatoraggregate) | Custom reducers |
| `EphemeralValue` | [Channels API](03_CHANNELS_API.md#ephemeralvalue) | Temporary storage |
| `NamedBarrierValue` | [Channels API](03_CHANNELS_API.md#namedbarriervalue) | Synchronization |
| `AnyValue` | [Channels API](03_CHANNELS_API.md#anyvalue) | Permissive storage |
| `UntrackedValue` | [Channels API](03_CHANNELS_API.md#untrackedvalue) | Non-checkpointed |

### Checkpointing

| Class/Function | Module | Description |
|----------------|--------|-------------|
| `BaseCheckpointSaver` | [Checkpoint API](04_CHECKPOINT_API.md#basecheckpointsaver) | Abstract saver interface |
| `MemorySaver` | [Checkpoint API](04_CHECKPOINT_API.md#memorysaver) | In-memory storage |
| `SqliteSaver` | [Checkpoint API](04_CHECKPOINT_API.md#sqlitesaver) | SQLite persistence |
| `PostgresSaver` | [Checkpoint API](04_CHECKPOINT_API.md#postgressaver) | PostgreSQL persistence |
| `AsyncSqliteSaver` | [Checkpoint API](04_CHECKPOINT_API.md#asyncsqlitesaver) | Async SQLite |
| `AsyncPostgresSaver` | [Checkpoint API](04_CHECKPOINT_API.md#asyncpostgressaver) | Async PostgreSQL |
| `JsonPlusSerializer` | [Checkpoint API](04_CHECKPOINT_API.md#jsonplusserializer) | Serialization |
| `BaseStore` | [Checkpoint API](04_CHECKPOINT_API.md#basestore) | Key-value store |
| `BaseCache` | [Checkpoint API](04_CHECKPOINT_API.md#basecache) | Caching interface |

### Prebuilt Components

| Function/Class | Module | Description |
|----------------|--------|-------------|
| `create_react_agent()` | [Prebuilt API](05_PREBUILT_API.md#create_react_agent) | Create ReAct agent |
| `ToolNode` | [Prebuilt API](05_PREBUILT_API.md#toolnode) | Execute tool calls |
| `ValidationNode` | [Prebuilt API](05_PREBUILT_API.md#validationnode) | Validate tool calls |
| `tools_condition` | [Prebuilt API](05_PREBUILT_API.md#tools_condition) | Route on tool calls |
| `InjectedState` | [Prebuilt API](05_PREBUILT_API.md#injectedstate) | State injection |
| `InjectedStore` | [Prebuilt API](05_PREBUILT_API.md#injectedstore) | Store injection |

### SDK Client

| Function/Class | Module | Description |
|----------------|--------|-------------|
| `get_client()` | [SDK API](06_SDK_API.md#get_client) | Get async client |
| `get_sync_client()` | [SDK API](06_SDK_API.md#get_sync_client) | Get sync client |
| `LangGraphClient` | [SDK API](06_SDK_API.md#langgraphclient) | Main client class |
| `AssistantsClient` | [SDK API](06_SDK_API.md#assistantsclient) | Manage assistants |
| `ThreadsClient` | [SDK API](06_SDK_API.md#threadsclient) | Manage threads |
| `RunsClient` | [SDK API](06_SDK_API.md#runsclient) | Manage runs |
| `StoreClient` | [SDK API](06_SDK_API.md#storeclient) | Key-value operations |

### CLI Commands

| Command | Module | Description |
|---------|--------|-------------|
| `langgraph new` | [CLI API](07_CLI_API.md#langgraph-new) | Create new project |
| `langgraph dev` | [CLI API](07_CLI_API.md#langgraph-dev) | Development server |
| `langgraph build` | [CLI API](07_CLI_API.md#langgraph-build) | Build Docker image |
| `langgraph up` | [CLI API](07_CLI_API.md#langgraph-up) | Run with Docker |
| `langgraph dockerfile` | [CLI API](07_CLI_API.md#langgraph-dockerfile) | Generate Dockerfile |

### Functional API

| Decorator/Function | Module | Description |
|--------------------|--------|-------------|
| `@entrypoint` | [Functional API](08_FUNCTIONAL_API.md#entrypoint) | Create workflow |
| `@task` | [Functional API](08_FUNCTIONAL_API.md#task) | Create parallel task |
| `entrypoint.final` | [Functional API](08_FUNCTIONAL_API.md#entrypointfinal) | Separate return/save |
| `interrupt()` | [Functional API](08_FUNCTIONAL_API.md#interrupt) | Human-in-the-loop |

### Types & Constants

| Type/Constant | Module | Description |
|---------------|--------|-------------|
| `Command` | [Types](09_TYPES_UTILITIES_API.md#command) | Control execution |
| `Send` | [Types](09_TYPES_UTILITIES_API.md#send) | Send to node |
| `Interrupt` | [Types](09_TYPES_UTILITIES_API.md#interrupt) | Interrupt info |
| `StateSnapshot` | [Types](09_TYPES_UTILITIES_API.md#statesnapshot) | State snapshot |
| `RetryPolicy` | [Types](09_TYPES_UTILITIES_API.md#retrypolicy) | Retry configuration |
| `CachePolicy` | [Types](09_TYPES_UTILITIES_API.md#cachepolicy) | Cache configuration |
| `START` | [Types](09_TYPES_UTILITIES_API.md#start) | Graph start constant |
| `END` | [Types](09_TYPES_UTILITIES_API.md#end) | Graph end constant |
| `interrupt()` | [Types](09_TYPES_UTILITIES_API.md#interrupt-function) | Create interrupt |

### Errors

| Exception | Module | Description |
|-----------|--------|-------------|
| `GraphRecursionError` | [Types](09_TYPES_UTILITIES_API.md#graphrecursionerror) | Recursion limit |
| `InvalidUpdateError` | [Types](09_TYPES_UTILITIES_API.md#invalidupdateerror) | Invalid state update |
| `GraphInterrupt` | [Types](09_TYPES_UTILITIES_API.md#graphinterrupt) | Interrupt exception |
| `NodeInterrupt` | [Types](09_TYPES_UTILITIES_API.md#nodeinterrupt) | Node interrupt (deprecated) |

### Configuration

| Function | Module | Description |
|----------|--------|-------------|
| `get_config()` | [Types](09_TYPES_UTILITIES_API.md#get_config) | Get runtime config |
| `get_store()` | [Types](09_TYPES_UTILITIES_API.md#get_store) | Get store instance |
| `get_stream_writer()` | [Types](09_TYPES_UTILITIES_API.md#get_stream_writer) | Get stream writer |

---

## Common Patterns

### Creating a Simple Graph

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

class State(TypedDict):
    messages: list[str]

graph = StateGraph(State)
graph.add_node("process", lambda state: {"messages": state["messages"] + ["processed"]})
graph.add_edge(START, "process")
graph.add_edge("process", END)

app = graph.compile()
result = app.invoke({"messages": ["hello"]})
```

### Using Checkpointing

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver

# In-memory (development)
memory = MemorySaver()
app = graph.compile(checkpointer=memory)

# SQLite (production)
with SqliteSaver.from_conn_string(":memory:") as saver:
    app = graph.compile(checkpointer=saver)
```

### Creating a ReAct Agent

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model=my_llm,
    tools=[my_tool],
    checkpointer=memory
)

result = agent.invoke({"messages": [("user", "Hello")]})
```

### Using the Functional API

```python
from langgraph.func import task, entrypoint

@task
def process(data: str) -> str:
    return data.upper()

@entrypoint(checkpointer=memory)
def workflow(input: str) -> str:
    result = process(input).result()
    return result
```

### Using the SDK Client

```python
from langgraph_sdk import get_client

async with get_client(url="http://localhost:8123") as client:
    thread = await client.threads.create()
    result = await client.runs.wait(
        thread["thread_id"],
        "my-assistant",
        input={"messages": [{"role": "user", "content": "Hello"}]}
    )
```

---

## Source File Locations

| Module | Primary Source Files |
|--------|---------------------|
| Graph API | `libs/langgraph/langgraph/graph/state.py` |
| Pregel API | `libs/langgraph/langgraph/pregel/main.py` |
| Channels API | `libs/langgraph/langgraph/channels/*.py` |
| Checkpoint API | `libs/checkpoint/langgraph/checkpoint/base/__init__.py` |
| Prebuilt API | `libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py` |
| SDK API | `libs/sdk-py/langgraph_sdk/client.py` |
| CLI API | `libs/cli/langgraph_cli/cli.py` |
| Functional API | `libs/langgraph/langgraph/func/__init__.py` |
| Types & Utilities | `libs/langgraph/langgraph/types.py` |

---

## See Also

- [Internal Documentation](../internal/00_INDEX.md) - Conceptual guides and explanations
- [Examples](../../examples/) - Working code examples
- [Official Docs](../docs/) - User-facing documentation
