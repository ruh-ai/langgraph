# LangGraph Graph Building API

This document provides a comprehensive reference for building graphs using the LangGraph StateGraph API.

## Table of Contents

1. [StateGraph Class](#stategraph-class)
2. [State Definition](#state-definition)
3. [Adding Nodes](#adding-nodes)
4. [Adding Edges](#adding-edges)
5. [Branching and Conditional Routing](#branching-and-conditional-routing)
6. [MessagesState Pattern](#messagesstate-pattern)
7. [Compilation](#compilation)
8. [Complete Examples](#complete-examples)

---

## StateGraph Class

### Overview

`StateGraph` is the primary builder class for creating stateful, multi-actor graphs in LangGraph. It represents a graph where nodes communicate by reading from and writing to a shared state.

**Key Characteristics:**
- Each node receives the current state as input
- Each node returns a partial state update (not the full state)
- State keys can have reducer functions to aggregate updates from multiple nodes
- StateGraph is a **builder** - you must call `.compile()` to create an executable graph

### Constructor

```python
StateGraph(
    state_schema: type[StateT],
    context_schema: type[ContextT] | None = None,
    *,
    input_schema: type[InputT] | None = None,
    output_schema: type[OutputT] | None = None,
)
```

**Parameters:**

- **state_schema**: The schema defining the graph's state structure. Can be:
  - `TypedDict` class
  - Pydantic `BaseModel` class
  - Dataclass
  - Annotated type with reducer function

- **context_schema** *(optional)*: Schema for runtime context data
  - Exposes immutable context to nodes (e.g., user_id, db_conn, API keys)
  - Access via `runtime.context` in nodes
  - Replaces deprecated `config_schema`

- **input_schema** *(optional)*: Schema for graph input
  - If not provided, defaults to `state_schema`
  - Allows accepting different input shape than internal state

- **output_schema** *(optional)*: Schema for graph output
  - If not provided, defaults to `state_schema`
  - Allows returning different output shape than internal state

**Example:**

```python
from typing_extensions import TypedDict, Annotated
from langgraph.graph import StateGraph

def reducer(a: list, b: int | None) -> list:
    if b is not None:
        return a + [b]
    return a

class State(TypedDict):
    x: Annotated[list, reducer]

class Context(TypedDict):
    user_id: str
    api_key: str

graph = StateGraph(state_schema=State, context_schema=Context)
```

### Core Methods

The StateGraph class provides methods for building your graph structure:

| Method | Purpose |
|--------|---------|
| `add_node()` | Add a node to the graph |
| `add_edge()` | Add a direct edge between nodes |
| `add_conditional_edges()` | Add conditional routing based on state |
| `add_sequence()` | Add a sequence of nodes executed in order |
| `set_entry_point()` | Set the starting node |
| `set_conditional_entry_point()` | Set conditional starting node |
| `set_finish_point()` | Mark a node as an exit point |
| `compile()` | Build the executable graph |

---

## State Definition

The state schema defines the structure of data that flows through your graph. There are three main approaches to defining state.

### TypedDict (Recommended)

Most flexible approach using Python's `TypedDict`:

```python
from typing_extensions import TypedDict, Annotated
import operator

class State(TypedDict):
    messages: Annotated[list, operator.add]  # List with append reducer
    user_input: str                           # Last value only
    count: Annotated[int, operator.add]      # Summing reducer
    metadata: dict                            # Last value only
```

**Key Features:**
- Use `Annotated[Type, reducer_func]` to specify custom reducers
- Without annotation, uses "last value wins" strategy
- Supports `Required` and `NotRequired` fields

### Pydantic BaseModel

Using Pydantic for validation and type safety:

```python
from pydantic import BaseModel
from typing import Annotated
import operator

class State(BaseModel):
    messages: Annotated[list[str], operator.add]
    temperature: float = 0.7
    max_tokens: int = 1000

    class Config:
        arbitrary_types_allowed = True
```

**Advantages:**
- Built-in validation
- Default values
- Type coercion
- Can use Pydantic validators

### Dataclass

Standard Python dataclasses:

```python
from dataclasses import dataclass, field
from typing import Annotated
import operator

@dataclass
class State:
    messages: Annotated[list[str], operator.add] = field(default_factory=list)
    count: int = 0
    user_id: str = ""
```

### Reducer Functions

Reducers determine how to merge multiple updates to the same state key:

**Built-in Reducers:**

```python
import operator

# Addition/Concatenation
Annotated[list, operator.add]      # Append to list
Annotated[int, operator.add]       # Sum numbers
Annotated[str, operator.add]       # Concatenate strings

# Logical operations
Annotated[dict, operator.or_]      # Merge dictionaries (right precedence)
```

**Custom Reducers:**

```python
def merge_unique(left: list, right: list) -> list:
    """Merge lists keeping only unique items."""
    return list(set(left + right))

def max_value(left: int, right: int) -> int:
    """Keep maximum value."""
    return max(left, right)

class State(TypedDict):
    unique_items: Annotated[list, merge_unique]
    max_score: Annotated[int, max_value]
```

**Reducer Signature:**
- Must accept exactly 2 parameters: `(left_value, right_value)`
- Must return a value of the same type
- `left` is the current/existing value
- `right` is the new/incoming value

### Special State Patterns

**Root-only State:**

```python
# When state is a simple type (not dict-like)
graph = StateGraph(Annotated[list[str], operator.add])
```

**Separate Input/Output Schemas:**

```python
class InputState(TypedDict):
    question: str

class InternalState(InputState):
    context: list[str]
    reasoning: str

class OutputState(TypedDict):
    answer: str

graph = StateGraph(
    state_schema=InternalState,
    input_schema=InputState,
    output_schema=OutputState
)
```

---

## Adding Nodes

Nodes are the computational units in your graph. Each node is a function or Runnable that processes state.

### add_node() Method

```python
def add_node(
    node: str | StateNode,
    action: StateNode | None = None,
    *,
    defer: bool = False,
    metadata: dict[str, Any] | None = None,
    input_schema: type[NodeInputT] | None = None,
    retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy | None = None,
    destinations: dict[str, str] | tuple[str, ...] | None = None,
) -> Self
```

**Parameters:**

- **node**: Node name (str) or the node function/Runnable
  - If Runnable/function provided, name is inferred from it

- **action** *(optional)*: The node function when `node` is a string name

- **defer** *(optional)*: Whether to defer node execution until run is ending
  - Useful for cleanup or finalization nodes

- **metadata** *(optional)*: Arbitrary metadata attached to the node

- **input_schema** *(optional)*: Custom input schema for this node
  - If not provided, uses graph's `state_schema`
  - Can be inferred from function type hints

- **retry_policy** *(optional)*: Retry configuration for the node
  - Single policy or sequence of policies

- **cache_policy** *(optional)*: Caching configuration for the node

- **destinations** *(optional)*: For rendering only - indicates possible routing targets

**Returns:** Self (for method chaining)

### Node Function Signatures

Nodes can have various signatures depending on what they need:

**Basic Node (state only):**

```python
def my_node(state: State) -> dict:
    """Process state and return updates."""
    return {"count": state["count"] + 1}
```

**Node with Config:**

```python
from langchain_core.runnables import RunnableConfig

def my_node(state: State, config: RunnableConfig) -> dict:
    """Access configuration during execution."""
    thread_id = config["configurable"]["thread_id"]
    return {"metadata": {"thread": thread_id}}
```

**Node with Runtime Context:**

```python
from langgraph.runtime import Runtime

def my_node(state: State, runtime: Runtime[Context]) -> dict:
    """Access immutable runtime context."""
    user_id = runtime.context["user_id"]
    return {"user_data": fetch_user(user_id)}
```

**Node with Stream Writer:**

```python
from langgraph.types import StreamWriter

def my_node(state: State, *, writer: StreamWriter) -> dict:
    """Stream custom data during execution."""
    writer({"progress": 50})
    # ... do work ...
    writer({"progress": 100})
    return {"result": "done"}
```

**Node with Store:**

```python
from langgraph.store.base import BaseStore

def my_node(state: State, *, store: BaseStore) -> dict:
    """Access persistent store."""
    data = store.get(("user", state["user_id"]))
    return {"cached_data": data}
```

**Combined Parameters:**

```python
def my_node(
    state: State,
    *,
    config: RunnableConfig,
    writer: StreamWriter,
    store: BaseStore,
    runtime: Runtime[Context]
) -> dict:
    """Node with all available parameters."""
    # ... implementation ...
    return {"status": "complete"}
```

### Node Return Values

Nodes can return different types of values:

**1. Dictionary (Partial State Update):**

```python
def node(state: State) -> dict:
    return {"key1": value1, "key2": value2}
```

**2. State Schema Instance:**

```python
def node(state: State) -> State:
    return State(key1=value1, key2=value2)
```

**3. Command Object (Advanced Routing):**

```python
from langgraph.types import Command

def node(state: State) -> Command[str]:
    if state["condition"]:
        return Command(
            update={"status": "processed"},
            goto="next_node"
        )
    return Command(goto="error_handler")
```

**4. List/Tuple of Updates and Commands:**

```python
def node(state: State) -> list:
    return [
        {"key": "value"},
        Command(goto="next_node")
    ]
```

**5. None (No Update):**

```python
def node(state: State) -> None:
    # Side effects only
    log_state(state)
    return None
```

### Adding Nodes - Examples

**Example 1: Simple node with inferred name**

```python
def process_input(state: State) -> dict:
    return {"processed": True}

builder.add_node(process_input)  # Name: "process_input"
```

**Example 2: Custom node name**

```python
builder.add_node("processor", process_input)
```

**Example 3: Node with custom input schema**

```python
class NodeInput(TypedDict):
    query: str

def search_node(state: NodeInput) -> dict:
    results = search(state["query"])
    return {"results": results}

builder.add_node(search_node, input_schema=NodeInput)
```

**Example 4: Node with retry policy**

```python
from langgraph.types import RetryPolicy

retry = RetryPolicy(
    initial_interval=1.0,
    max_attempts=3,
    backoff_factor=2.0
)

builder.add_node("api_call", make_api_call, retry_policy=retry)
```

**Example 5: Node with caching**

```python
from langgraph.types import CachePolicy

cache = CachePolicy(ttl=3600)  # 1 hour cache

builder.add_node("expensive_computation", compute, cache_policy=cache)
```

**Example 6: Runnable as node**

```python
from langchain_core.runnables import RunnableLambda

runnable = RunnableLambda(lambda x: {"result": x["value"] * 2})
builder.add_node("doubler", runnable)
```

**Example 7: Lambda node**

```python
builder.add_node(
    "increment",
    lambda state: {"count": state["count"] + 1}
)
```

---

## Adding Edges

Edges define the flow of execution through your graph.

### add_edge() - Direct Edges

Creates a deterministic connection from one node to another.

```python
def add_edge(start_key: str | list[str], end_key: str) -> Self
```

**Parameters:**
- **start_key**: Source node name (or list of nodes for fan-in)
- **end_key**: Destination node name

**Single Edge:**

```python
builder.add_edge("node_a", "node_b")
# node_a → node_b
```

**Fan-in (Multiple sources, single destination):**

```python
builder.add_edge(["node_a", "node_b"], "node_c")
# Waits for BOTH node_a AND node_b to complete before executing node_c
```

**Using START and END:**

```python
from langgraph.graph import START, END

builder.add_edge(START, "first_node")    # Entry point
builder.add_edge("last_node", END)       # Exit point
```

### add_conditional_edges() - Branching

Creates dynamic routing based on state evaluation.

```python
def add_conditional_edges(
    source: str,
    path: Callable[..., Hashable | Sequence[Hashable]],
    path_map: dict[Hashable, str] | list[str] | None = None,
) -> Self
```

**Parameters:**

- **source**: The node where the condition is evaluated
- **path**: Function that determines next node(s)
  - Receives state as input
  - Returns node name(s) or path key(s)
- **path_map** *(optional)*: Maps return values to node names
  - If omitted, path function must return actual node names

**Example 1: Direct node names**

```python
def route(state: State) -> str:
    if state["score"] > 0.8:
        return "high_confidence"
    elif state["score"] > 0.5:
        return "medium_confidence"
    else:
        return "low_confidence"

builder.add_conditional_edges("classifier", route)
```

**Example 2: Using path_map**

```python
def route(state: State) -> str:
    if state["score"] > 0.8:
        return "high"
    elif state["score"] > 0.5:
        return "medium"
    else:
        return "low"

builder.add_conditional_edges(
    "classifier",
    route,
    path_map={
        "high": "process_immediately",
        "medium": "queue_for_review",
        "low": "reject"
    }
)
```

**Example 3: Multiple destinations (fan-out)**

```python
def route(state: State) -> list[str]:
    destinations = []
    if state["needs_search"]:
        destinations.append("search")
    if state["needs_validation"]:
        destinations.append("validate")
    if not destinations:
        destinations.append(END)
    return destinations

builder.add_conditional_edges("preprocessor", route)
```

**Example 4: Using Literal type hints**

```python
from typing import Literal

def route(state: State) -> Literal["success", "failure", END]:
    if state["error"]:
        return "failure"
    elif state["complete"]:
        return END
    return "success"

# No path_map needed - inferred from type hint
builder.add_conditional_edges("processor", route)
```

**Example 5: Using Send for dynamic fan-out**

```python
from langgraph.types import Send

def route(state: State) -> list[Send]:
    # Send different inputs to same node
    return [
        Send("process_item", {"item": item})
        for item in state["items"]
    ]

builder.add_conditional_edges("splitter", route)
```

### add_sequence() - Linear Chains

Convenience method to add multiple nodes in sequence.

```python
def add_sequence(
    nodes: Sequence[StateNode | tuple[str, StateNode]]
) -> Self
```

**Example:**

```python
builder.add_sequence([
    ("fetch", fetch_data),
    ("process", process_data),
    ("validate", validate_results),
    ("save", save_to_db)
])

# Equivalent to:
# builder.add_node("fetch", fetch_data)
# builder.add_node("process", process_data)
# builder.add_node("validate", validate_results)
# builder.add_node("save", save_to_db)
# builder.add_edge("fetch", "process")
# builder.add_edge("process", "validate")
# builder.add_edge("validate", "save")
```

### Helper Methods

**set_entry_point()**

```python
def set_entry_point(key: str) -> Self
```

Equivalent to `add_edge(START, key)`

```python
builder.set_entry_point("first_node")
```

**set_conditional_entry_point()**

```python
def set_conditional_entry_point(
    path: Callable,
    path_map: dict | list | None = None
) -> Self
```

Equivalent to `add_conditional_edges(START, path, path_map)`

```python
def route_start(state: State) -> str:
    return "new_user" if state["is_new"] else "returning_user"

builder.set_conditional_entry_point(route_start)
```

**set_finish_point()**

```python
def set_finish_point(key: str) -> Self
```

Equivalent to `add_edge(key, END)`

```python
builder.set_finish_point("final_node")
```

---

## Branching and Conditional Routing

This section covers the internal mechanics of how conditional routing works.

### BranchSpec Internal Structure

When you call `add_conditional_edges()`, LangGraph creates a `BranchSpec` internally:

```python
class BranchSpec(NamedTuple):
    path: Runnable[Any, Hashable | list[Hashable]]
    ends: dict[Hashable, str] | None
    input_schema: type[Any] | None
```

**Components:**
- **path**: The routing function (converted to Runnable)
- **ends**: Mapping of return values to node names
- **input_schema**: Schema for the state passed to routing function

### Routing Function Execution

**Step 1: State Preparation**

The routing function receives state based on its schema:

```python
# If input_schema specified or inferred from type hints:
def route(state: CustomSchema) -> str:
    # Receives only keys from CustomSchema
    pass

# Otherwise, receives full graph state:
def route(state: State) -> str:
    # Receives all state keys
    pass
```

**Step 2: Path Resolution**

```python
# Path function is invoked
result = path.invoke(state, config)

# Result is normalized to list
if not isinstance(result, (list, tuple)):
    result = [result]

# Each result is mapped to node name
if ends_mapping:
    destinations = [ends_mapping[r] for r in result]
else:
    destinations = result
```

**Step 3: Validation**

```python
# Invalid destinations raise errors:
if any(dest is None or dest == START for dest in destinations):
    raise ValueError("Branch did not return a valid destination")

if any(isinstance(p, Send) and p.node == END for p in destinations):
    raise InvalidUpdateError("Cannot send a packet to the END node")
```

**Step 4: Write to Channels**

```python
# Creates write entries for each destination
entries = [
    ChannelWriteEntry(f"branch:to:{dest}", None)
    for dest in destinations
    if dest != END
]

# Executes writes to trigger next nodes
ChannelWrite.do_write(config, entries)
```

### Dynamic Routing with Send

The `Send` primitive enables map-reduce patterns:

```python
from langgraph.types import Send

def map_step(state: State) -> list[Send]:
    """Fan out to process multiple items."""
    return [
        Send("process", {"item": item, "index": i})
        for i, item in enumerate(state["items"])
    ]

def process(state: dict) -> dict:
    """Process individual item."""
    result = expensive_operation(state["item"])
    return {"results": [result]}

class State(TypedDict):
    items: list
    results: Annotated[list, operator.add]

builder = StateGraph(State)
builder.add_node("process", process)
builder.add_conditional_edges(START, map_step)
builder.add_edge("process", END)
```

**How Send Works:**

1. Each `Send` creates a separate task
2. Tasks execute in parallel (if possible)
3. Each task receives its own state (the `arg` parameter)
4. Results are merged back using reducers

### Command-Based Routing

The `Command` object provides more control over routing:

```python
from langgraph.types import Command

def node(state: State) -> Command:
    return Command(
        update={"status": "processed"},  # State updates
        goto="next_node"                  # Where to go next
    )

# Multiple destinations:
def node(state: State) -> Command:
    return Command(
        update={"checkpoint": "saved"},
        goto=["validator", "logger"]  # Fan-out to multiple nodes
    )

# Conditional with Send:
def node(state: State) -> Command:
    return Command(
        update={"started": True},
        goto=[Send("worker", {"task": t}) for t in state["tasks"]]
    )

# Parent graph routing:
def node(state: State) -> Command:
    return Command(
        graph=Command.PARENT,  # Route in parent graph
        goto="parent_node"
    )
```

---

## MessagesState Pattern

LangGraph provides special support for conversation/message-based applications.

### MessagesState

Pre-built state schema for message-based workflows:

```python
from langgraph.graph import MessagesState

class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

**Usage:**

```python
from langgraph.graph import MessagesState, StateGraph

builder = StateGraph(MessagesState)
```

### add_messages Reducer

Special reducer for merging message lists:

```python
from langgraph.graph import add_messages
```

**Features:**

1. **Append-only by default**: New messages are added to the list
2. **Message deduplication**: Messages with same ID are merged
3. **Message deletion**: Use `RemoveMessage` to delete by ID
4. **Format conversion**: Optional OpenAI format conversion

**Basic Usage:**

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]
```

**Examples:**

**Appending messages:**

```python
from langchain_core.messages import HumanMessage, AIMessage

msgs1 = [HumanMessage(content="Hello", id="1")]
msgs2 = [AIMessage(content="Hi there!", id="2")]

result = add_messages(msgs1, msgs2)
# [HumanMessage(content='Hello', id='1'),
#  AIMessage(content='Hi there!', id='2')]
```

**Updating messages by ID:**

```python
msgs1 = [HumanMessage(content="Hello", id="1")]
msgs2 = [HumanMessage(content="Hello again", id="1")]

result = add_messages(msgs1, msgs2)
# [HumanMessage(content='Hello again', id='1')]
# Original message replaced
```

**Removing messages:**

```python
from langchain_core.messages import RemoveMessage

msgs1 = [
    HumanMessage(content="Hello", id="1"),
    AIMessage(content="Hi", id="2")
]
msgs2 = [RemoveMessage(id="1")]

result = add_messages(msgs1, msgs2)
# [AIMessage(content='Hi', id='2')]
```

**Remove all messages:**

```python
from langgraph.graph.message import REMOVE_ALL_MESSAGES

msgs2 = [
    RemoveMessage(id=REMOVE_ALL_MESSAGES),
    HumanMessage(content="Fresh start", id="3")
]

result = add_messages(msgs1, msgs2)
# [HumanMessage(content='Fresh start', id='3')]
```

**OpenAI format conversion:**

```python
from typing import Annotated

class State(TypedDict):
    messages: Annotated[list, add_messages(format="langchain-openai")]

# Messages are automatically converted to OpenAI-compatible format
```

### Message-Based Graph Example

```python
from langchain_core.messages import HumanMessage, AIMessage
from langgraph.graph import MessagesState, StateGraph, START, END

def chatbot(state: MessagesState) -> dict:
    """Simple chatbot node."""
    user_message = state["messages"][-1].content
    response = f"You said: {user_message}"
    return {"messages": [AIMessage(content=response)]}

builder = StateGraph(MessagesState)
builder.add_node("chatbot", chatbot)
builder.add_edge(START, "chatbot")
builder.add_edge("chatbot", END)

graph = builder.compile()

# Usage
result = graph.invoke({
    "messages": [HumanMessage(content="Hello!")]
})

# Result:
# {
#   "messages": [
#       HumanMessage(content="Hello!", id="..."),
#       AIMessage(content="You said: Hello!", id="...")
#   ]
# }
```

### Advanced Message Patterns

**Multi-turn conversation:**

```python
def conversational_node(state: MessagesState) -> dict:
    # Access full conversation history
    history = state["messages"]

    # Get last user message
    last_user_msg = next(
        (m for m in reversed(history) if isinstance(m, HumanMessage)),
        None
    )

    # Generate response
    response = llm.invoke(history)

    return {"messages": [response]}
```

**Tool calling pattern:**

```python
from langchain_core.messages import ToolMessage

def agent(state: MessagesState) -> dict:
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}

def tool_executor(state: MessagesState) -> dict:
    last_message = state["messages"][-1]

    tool_calls = last_message.tool_calls
    responses = []

    for tool_call in tool_calls:
        result = execute_tool(tool_call)
        responses.append(
            ToolMessage(
                content=result,
                tool_call_id=tool_call["id"]
            )
        )

    return {"messages": responses}

def should_continue(state: MessagesState) -> str:
    last_message = state["messages"][-1]
    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return END

builder = StateGraph(MessagesState)
builder.add_node("agent", agent)
builder.add_node("tools", tool_executor)
builder.set_entry_point("agent")
builder.add_conditional_edges("agent", should_continue, ["tools", END])
builder.add_edge("tools", "agent")
```

---

## Compilation

The `compile()` method transforms a StateGraph builder into an executable `CompiledStateGraph`.

### compile() Method

```python
def compile(
    checkpointer: Checkpointer = None,
    *,
    cache: BaseCache | None = None,
    store: BaseStore | None = None,
    interrupt_before: All | list[str] | None = None,
    interrupt_after: All | list[str] | None = None,
    debug: bool = False,
    name: str | None = None,
) -> CompiledStateGraph
```

**Parameters:**

- **checkpointer** *(optional)*: Checkpoint saver for persistence
  - `None`: May inherit from parent graph
  - `False`: Disable checkpointing explicitly
  - `BaseCheckpointSaver` instance: Enable with specific implementation

- **cache** *(optional)*: Cache implementation for node results

- **store** *(optional)*: Persistent key-value store

- **interrupt_before** *(optional)*: Nodes to interrupt before execution
  - `"*"` or `All`: Interrupt before every node
  - `["node1", "node2"]`: Interrupt before specific nodes

- **interrupt_after** *(optional)*: Nodes to interrupt after execution
  - `"*"` or `All`: Interrupt after every node
  - `["node1", "node2"]`: Interrupt after specific nodes

- **debug** *(optional)*: Enable debug mode

- **name** *(optional)*: Name for the compiled graph

**Returns:** `CompiledStateGraph` - executable graph implementing Runnable interface

### Checkpointing

Enables persistence, time-travel, and human-in-the-loop patterns:

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# Usage with thread ID
config = {"configurable": {"thread_id": "conversation-1"}}
result = graph.invoke({"input": "Hello"}, config)

# Continue same thread
result = graph.invoke({"input": "Continue"}, config)
```

**Available Checkpointers:**

- `InMemorySaver`: Memory-only (testing)
- `SqliteSaver`: SQLite database
- `PostgresSaver`: PostgreSQL database
- `AsyncPostgresSaver`: Async PostgreSQL

### Interrupts

Human-in-the-loop workflows:

```python
# Interrupt before node for approval
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["human_review"]
)

# First run - stops before human_review
graph.invoke({"data": "..."}, config)

# Resume after approval
graph.invoke(None, config)  # Continues from interrupt
```

**Interrupt all nodes:**

```python
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before="*"
)
```

### Cache Configuration

Cache expensive computations:

```python
from langgraph.cache.base import BaseCache

cache = MyCache()
graph = builder.compile(cache=cache)
```

Nodes with `cache_policy` will use this cache.

### Store Configuration

Persistent key-value storage:

```python
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
graph = builder.compile(store=store)

# Access in nodes:
def node(state: State, *, store: BaseStore) -> dict:
    value = store.get(("namespace", "key"))
    store.put(("namespace", "key"), {"data": "value"})
    return {}
```

### CompiledStateGraph

The result of `compile()` is a `CompiledStateGraph` which:

- Implements LangChain's `Runnable` interface
- Provides `invoke()`, `ainvoke()`, `stream()`, `astream()`, `batch()`, `abatch()`
- Supports all standard Runnable operations

**Key Methods:**

```python
# Synchronous invocation
result = graph.invoke(input_state, config)

# Asynchronous invocation
result = await graph.ainvoke(input_state, config)

# Streaming
for chunk in graph.stream(input_state, config):
    print(chunk)

# Async streaming
async for chunk in graph.astream(input_state, config):
    print(chunk)

# Get state snapshot
snapshot = graph.get_state(config)

# Update state
graph.update_state(config, {"key": "value"})

# Get state history
for state in graph.get_state_history(config):
    print(state)
```

**Stream Modes:**

```python
# Stream state values
for chunk in graph.stream(input, config, stream_mode="values"):
    print(chunk)

# Stream updates only
for chunk in graph.stream(input, config, stream_mode="updates"):
    print(chunk)

# Stream messages (for LLM tokens)
for chunk in graph.stream(input, config, stream_mode="messages"):
    print(chunk)

# Multiple modes
for chunk in graph.stream(input, config, stream_mode=["updates", "messages"]):
    print(chunk)
```

### Validation

The `compile()` method validates your graph:

- **Entry point exists**: At least one edge from START
- **Valid nodes**: All referenced nodes exist
- **Valid edges**: All edges connect valid nodes
- **No cycles** (for certain graph types)
- **Interrupt nodes exist**: If specified

Validation errors are raised with descriptive messages.

### Example Compilation Patterns

**Basic compilation:**

```python
graph = builder.compile()
result = graph.invoke({"input": "data"})
```

**With checkpointing:**

```python
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver.from_conn_string("checkpoints.db")
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "thread-1"}}
graph.invoke({"input": "data"}, config)
```

**Human-in-the-loop:**

```python
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["approval_needed"]
)

# First run
state = graph.invoke({"request": "..."}, config)
# Stops at approval_needed

# After human approval
graph.invoke(None, config)  # Continues
```

**Debug mode:**

```python
graph = builder.compile(debug=True)
# Provides detailed execution logs
```

---

## Complete Examples

### Example 1: Simple Linear Chain

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    input: str
    output: str

def step1(state: State) -> dict:
    return {"output": f"Processed: {state['input']}"}

def step2(state: State) -> dict:
    return {"output": state["output"].upper()}

builder = StateGraph(State)
builder.add_node("step1", step1)
builder.add_node("step2", step2)
builder.add_edge(START, "step1")
builder.add_edge("step1", "step2")
builder.add_edge("step2", END)

graph = builder.compile()
result = graph.invoke({"input": "hello"})
# {"input": "hello", "output": "PROCESSED: HELLO"}
```

### Example 2: Conditional Branching

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    number: int
    result: str

def check_number(state: State) -> str:
    if state["number"] > 0:
        return "positive"
    elif state["number"] < 0:
        return "negative"
    return "zero"

def handle_positive(state: State) -> dict:
    return {"result": "Number is positive"}

def handle_negative(state: State) -> dict:
    return {"result": "Number is negative"}

def handle_zero(state: State) -> dict:
    return {"result": "Number is zero"}

builder = StateGraph(State)
builder.add_node("positive", handle_positive)
builder.add_node("negative", handle_negative)
builder.add_node("zero", handle_zero)

builder.add_conditional_edges(
    START,
    check_number,
    {
        "positive": "positive",
        "negative": "negative",
        "zero": "zero"
    }
)

builder.add_edge("positive", END)
builder.add_edge("negative", END)
builder.add_edge("zero", END)

graph = builder.compile()
result = graph.invoke({"number": 5})
# {"number": 5, "result": "Number is positive"}
```

### Example 3: Map-Reduce with Send

```python
from typing import Annotated
from typing_extensions import TypedDict
import operator
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send

class State(TypedDict):
    items: list[int]
    results: Annotated[list[int], operator.add]

class ItemState(TypedDict):
    item: int

def map_items(state: State) -> list[Send]:
    """Fan out to process each item."""
    return [
        Send("process_item", {"item": item})
        for item in state["items"]
    ]

def process_item(state: ItemState) -> dict:
    """Process individual item."""
    result = state["item"] * 2
    return {"results": [result]}

builder = StateGraph(State)
builder.add_node("process_item", process_item)
builder.add_conditional_edges(START, map_items)
builder.add_edge("process_item", END)

graph = builder.compile()
result = graph.invoke({"items": [1, 2, 3, 4, 5]})
# {
#     "items": [1, 2, 3, 4, 5],
#     "results": [2, 4, 6, 8, 10]
# }
```

### Example 4: Agent with Tools

```python
from typing import Annotated
from langchain_core.messages import HumanMessage, AIMessage, ToolMessage
from langgraph.graph import MessagesState, StateGraph, START, END

def agent(state: MessagesState) -> dict:
    """Agent decides whether to use tools."""
    messages = state["messages"]
    # Simulate LLM with tool calling
    last_msg = messages[-1]

    if "search" in last_msg.content.lower():
        # Simulate tool call
        return {
            "messages": [
                AIMessage(
                    content="",
                    tool_calls=[{
                        "id": "call_1",
                        "name": "search",
                        "args": {"query": "weather"}
                    }]
                )
            ]
        }

    return {"messages": [AIMessage(content="I don't need tools for that.")]}

def execute_tools(state: MessagesState) -> dict:
    """Execute tool calls."""
    last_message = state["messages"][-1]
    responses = []

    for tool_call in last_message.tool_calls:
        # Simulate tool execution
        result = f"Search results for: {tool_call['args']['query']}"
        responses.append(
            ToolMessage(
                content=result,
                tool_call_id=tool_call["id"]
            )
        )

    return {"messages": responses}

def route(state: MessagesState) -> str:
    """Route based on last message."""
    last_message = state["messages"][-1]

    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        return "tools"
    return END

builder = StateGraph(MessagesState)
builder.add_node("agent", agent)
builder.add_node("tools", execute_tools)
builder.set_entry_point("agent")
builder.add_conditional_edges("agent", route, ["tools", END])
builder.add_edge("tools", "agent")

graph = builder.compile()

result = graph.invoke({
    "messages": [HumanMessage(content="Search for weather")]
})
```

### Example 5: Cyclic Graph with Iteration Limit

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    count: int
    max_iterations: int

def increment(state: State) -> dict:
    return {"count": state["count"] + 1}

def should_continue(state: State) -> str:
    if state["count"] >= state["max_iterations"]:
        return END
    return "increment"

builder = StateGraph(State)
builder.add_node("increment", increment)
builder.set_entry_point("increment")
builder.add_conditional_edges("increment", should_continue, ["increment", END])

graph = builder.compile()

result = graph.invoke({"count": 0, "max_iterations": 5})
# {"count": 5, "max_iterations": 5}
```

### Example 6: Human-in-the-Loop

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt

class State(TypedDict):
    data: str
    approved: bool

def process(state: State) -> dict:
    # Ask for human approval
    approval = interrupt("Do you approve this data?")
    return {"approved": approval}

def finalize(state: State) -> dict:
    if state["approved"]:
        return {"data": f"APPROVED: {state['data']}"}
    return {"data": f"REJECTED: {state['data']}"}

builder = StateGraph(State)
builder.add_node("process", process)
builder.add_node("finalize", finalize)
builder.add_edge(START, "process")
builder.add_edge("process", "finalize")
builder.add_edge("finalize", END)

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "thread-1"}}

# First run - will interrupt
result = graph.invoke({"data": "sensitive info"}, config)
# Returns interrupt information

# Resume with approval
from langgraph.types import Command
result = graph.invoke(Command(resume=True), config)
# {"data": "APPROVED: sensitive info", "approved": True}
```

### Example 7: Multiple State Schemas

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class InputState(TypedDict):
    question: str

class InternalState(InputState):
    context: list[str]
    reasoning: str

class OutputState(TypedDict):
    answer: str
    confidence: float

def gather_context(state: InputState) -> dict:
    return {
        "context": ["fact1", "fact2"],
        "reasoning": "Based on the question..."
    }

def generate_answer(state: InternalState) -> dict:
    return {
        "answer": f"Answer to: {state['question']}",
        "confidence": 0.95
    }

builder = StateGraph(
    state_schema=InternalState,
    input_schema=InputState,
    output_schema=OutputState
)

builder.add_node("gather_context", gather_context)
builder.add_node("generate_answer", generate_answer)
builder.add_edge(START, "gather_context")
builder.add_edge("gather_context", "generate_answer")
builder.add_edge("generate_answer", END)

graph = builder.compile()

# Input uses InputState schema
result = graph.invoke({"question": "What is LangGraph?"})

# Output uses OutputState schema
# {"answer": "Answer to: What is LangGraph?", "confidence": 0.95}
# Note: context and reasoning are not in output
```

### Example 8: Using Runtime Context

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.runtime import Runtime

class State(TypedDict):
    query: str
    result: str

class Context(TypedDict):
    api_key: str
    user_id: str

def fetch_data(state: State, runtime: Runtime[Context]) -> dict:
    api_key = runtime.context["api_key"]
    user_id = runtime.context["user_id"]

    # Use context in processing
    result = f"Data for {user_id}: {state['query']}"
    return {"result": result}

builder = StateGraph(state_schema=State, context_schema=Context)
builder.add_node("fetch", fetch_data)
builder.add_edge(START, "fetch")
builder.add_edge("fetch", END)

graph = builder.compile()

result = graph.invoke(
    {"query": "search term"},
    context={"api_key": "sk-...", "user_id": "user-123"}
)
# {
#     "query": "search term",
#     "result": "Data for user-123: search term"
# }
```

---

## Best Practices

### State Design

1. **Use TypedDict for clarity**: Explicit field types help catch errors
2. **Choose appropriate reducers**: Match reducer to your merge semantics
3. **Minimize state size**: Only include necessary data
4. **Use separate I/O schemas**: When internal state differs from API

### Node Design

1. **Pure functions**: Nodes should be stateless
2. **Return partial updates**: Only return changed fields
3. **Type hints**: Enable automatic schema inference
4. **Error handling**: Use try/except and return error state

### Graph Structure

1. **Clear entry/exit**: Always define START and END edges
2. **Validate branches**: Ensure all paths are handled
3. **Avoid deep nesting**: Flatten complex logic into separate nodes
4. **Use subgraphs**: For reusable components

### Performance

1. **Enable checkpointing selectively**: Only when needed
2. **Use caching**: For expensive, deterministic nodes
3. **Parallel execution**: Use Send for map-reduce patterns
4. **Batch operations**: Process multiple items together

### Debugging

1. **Use debug mode**: `compile(debug=True)`
2. **Stream updates**: Monitor execution in real-time
3. **Check state history**: Review past states
4. **Add logging nodes**: Create nodes just for logging

---

## Common Patterns

### Loop Until Condition

```python
def should_continue(state: State) -> str:
    return "process" if state["continue"] else END

builder.add_conditional_edges("process", should_continue, ["process", END])
```

### Error Handling

```python
def node_with_error_handling(state: State) -> dict:
    try:
        result = risky_operation(state)
        return {"result": result, "error": None}
    except Exception as e:
        return {"result": None, "error": str(e)}

def route_on_error(state: State) -> str:
    return "error_handler" if state["error"] else "success"

builder.add_conditional_edges("process", route_on_error)
```

### Parallel Processing

```python
def fan_out(state: State) -> list[str]:
    return ["worker_1", "worker_2", "worker_3"]

def fan_in(state: State) -> dict:
    # All workers must complete before this runs
    return {"final": "combined result"}

builder.add_conditional_edges("split", fan_out)
builder.add_edge(["worker_1", "worker_2", "worker_3"], "combine")
```

### Retry Logic

```python
from langgraph.types import RetryPolicy

retry = RetryPolicy(
    initial_interval=1.0,
    max_attempts=3,
    backoff_factor=2.0,
    retry_on=Exception  # or specific exception types
)

builder.add_node("api_call", make_api_call, retry_policy=retry)
```

---

## Summary

The LangGraph Graph Building API provides:

- **StateGraph**: Builder for stateful, multi-actor graphs
- **Flexible state schemas**: TypedDict, Pydantic, dataclass
- **Reducer functions**: Custom state update logic
- **Multiple edge types**: Direct, conditional, fan-in, fan-out
- **Message patterns**: Built-in support for conversations
- **Compilation**: Transform builder to executable Runnable
- **Advanced features**: Checkpointing, interrupts, caching, streaming

The API is designed to be:
- **Type-safe**: Leverage Python type hints
- **Composable**: Build complex workflows from simple parts
- **Observable**: Stream execution and inspect state
- **Persistent**: Save and resume execution
- **Flexible**: Support many workflow patterns

For more examples and patterns, see the LangGraph documentation and example repository.
