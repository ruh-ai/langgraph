# LangGraph Types and Utilities API Reference

This document provides a comprehensive reference for all public types, utilities, constants, and error classes in LangGraph.

## Table of Contents

- [Core Types](#core-types)
  - [Command](#command)
  - [Send](#send)
  - [Interrupt](#interrupt)
  - [Overwrite](#overwrite)
- [Type Aliases](#type-aliases)
  - [Durability](#durability)
  - [All](#all)
  - [Checkpointer](#checkpointer)
  - [StreamMode](#streammode)
  - [StreamWriter](#streamwriter)
- [Policy Types](#policy-types)
  - [RetryPolicy](#retrypolicy)
  - [CachePolicy](#cachepolicy)
  - [CacheKey](#cachekey)
- [State Management Types](#state-management-types)
  - [StateSnapshot](#statesnapshot)
  - [StateUpdate](#stateupdate)
- [Task Types](#task-types)
  - [PregelTask](#pregeltask)
  - [PregelExecutableTask](#pregelexecutabletask)
- [Constants](#constants)
  - [START](#start)
  - [END](#end)
  - [TAG_NOSTREAM](#tag_nostream)
  - [TAG_HIDDEN](#tag_hidden)
- [Error Classes](#error-classes)
  - [ErrorCode](#errorcode)
  - [GraphRecursionError](#graphrecursionerror)
  - [InvalidUpdateError](#invalidupdateerror)
  - [GraphBubbleUp](#graphbubbleup)
  - [GraphInterrupt](#graphinterrupt)
  - [NodeInterrupt](#nodeinterrupt)
  - [ParentCommand](#parentcommand)
  - [EmptyInputError](#emptyinputerror)
  - [TaskNotFound](#tasknotfound)
  - [EmptyChannelError](#emptychannelerror)
- [Configuration Functions](#configuration-functions)
  - [get_config](#get_config)
  - [get_store](#get_store)
  - [get_stream_writer](#get_stream_writer)
- [Core Functions](#core-functions)
  - [interrupt](#interrupt-1)
  - [ensure_valid_checkpointer](#ensure_valid_checkpointer)
- [Utility Classes](#utility-classes)
  - [RunnableCallable](#runnablecallable)
- [Utility Functions](#utility-functions)
  - [ensure_config](#ensure_config)
  - [patch_configurable](#patch_configurable)
- [Managed Values](#managed-values)
  - [ManagedValue](#managedvalue)
  - [ManagedValueSpec](#managedvaluespec)
  - [ManagedValueMapping](#managedvaluemapping)
  - [IsLastStep](#islaststep)
  - [RemainingSteps](#remainingsteps)

---

## Core Types

### Command

**Definition:**
```python
@dataclass(kw_only=True, slots=True, frozen=True)
class Command(Generic[N], ToolOutputMixin):
    graph: str | None = None
    update: Any | None = None
    resume: dict[str, Any] | Any | None = None
    goto: Send | Sequence[Send | N] | N = ()
```

**Description:**
One or more commands to update the graph's state and send messages to nodes. The `Command` type is a powerful primitive for controlling graph execution, allowing you to update state, resume from interrupts, and direct execution flow.

**Fields:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| graph | `str \| None` | No | Graph to send the command to. `None` means the current graph, `Command.PARENT` targets the closest parent graph. Default: `None` |
| update | `Any \| None` | No | Update to apply to the graph's state. Can be a dict, a list of tuples, or any object with annotated fields. Default: `None` |
| resume | `dict[str, Any] \| Any \| None` | No | Value to resume execution with. Used together with `interrupt()`. Can be a mapping of interrupt IDs to resume values, or a single value to resume the next interrupt. Default: `None` |
| goto | `Send \| Sequence[Send \| N] \| N` | No | Can be: a node name, sequence of node names, `Send` object, or sequence of `Send` objects to navigate to next. Default: `()` |

**Class Attributes:**

| Name | Type | Description |
|------|------|-------------|
| PARENT | `Literal["__parent__"]` | Special value to target the parent graph |

**Example:**
```python
from langgraph.types import Command
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class State(TypedDict):
    value: int

def node(state: State):
    # Update state and navigate to a specific node
    return Command(
        update={"value": state["value"] + 1},
        goto="next_node"
    )

# Resume from an interrupt
def resume_node(state: State):
    return Command(
        resume={"user_input": "approved"}
    )

# Navigate to parent graph
def child_node(state: State):
    return Command(
        graph=Command.PARENT,
        update={"value": 42}
    )
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:363-414`

---

### Send

**Definition:**
```python
class Send:
    __slots__ = ("node", "arg")

    node: str
    arg: Any

    def __init__(self, /, node: str, arg: Any) -> None:
        ...
```

**Description:**
A message or packet to send to a specific node in the graph. The `Send` class is used within a `StateGraph`'s conditional edges to dynamically invoke a node with a custom state at the next step. The sent state can differ from the core graph's state, enabling flexible workflows like map-reduce patterns.

**Fields:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| node | `str` | Yes | The name of the target node to send the message to |
| arg | `Any` | Yes | The state or message to send to the target node |

**Methods:**

- `__init__(/, node: str, arg: Any) -> None`: Initialize a new Send instance
- `__hash__() -> int`: Returns hash of (node, arg) tuple
- `__repr__() -> str`: Returns string representation
- `__eq__(value: object) -> bool`: Checks equality with another Send object

**Example:**
```python
from typing import Annotated
from langgraph.types import Send
from langgraph.graph import END, START, StateGraph
from typing_extensions import TypedDict
import operator

class OverallState(TypedDict):
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]

def continue_to_jokes(state: OverallState):
    # Send to the same node multiple times with different args
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

builder = StateGraph(OverallState)
builder.add_node("generate_joke", lambda state: {"jokes": [f"Joke about {state['subject']}"]})
builder.add_conditional_edges(START, continue_to_jokes)
builder.add_edge("generate_joke", END)
graph = builder.compile()

# Invoking with two subjects results in a generated joke for each
result = graph.invoke({"subjects": ["cats", "dogs"]})
# {'subjects': ['cats', 'dogs'], 'jokes': ['Joke about cats', 'Joke about dogs']}
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:285-358`

---

### Interrupt

**Definition:**
```python
@final
@dataclass(init=False, slots=True)
class Interrupt:
    value: Any
    id: str

    def __init__(
        self,
        value: Any,
        id: str = _DEFAULT_INTERRUPT_ID,
        **deprecated_kwargs: Unpack[DeprecatedKwargs],
    ) -> None:
        ...
```

**Description:**
Information about an interrupt that occurred in a node. Contains the value associated with the interrupt and a unique identifier that can be used to resume execution.

**Fields:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| value | `Any` | Yes | The value associated with the interrupt |
| id | `str` | Yes | The ID of the interrupt. Can be used to resume the interrupt directly. Default: auto-generated |

**Methods:**

- `from_ns(cls, value: Any, ns: str) -> Interrupt`: Create an Interrupt from a namespace string
- `interrupt_id` (property, deprecated): Use `id` instead

**Version History:**
- Added in version 0.2.24
- Changed in version 0.4.0: `interrupt_id` was introduced as a property
- Changed in version 0.6.0: `ns`, `when`, `resumable`, `interrupt_id` attributes removed

**Example:**
```python
from langgraph.types import Interrupt

# Create an interrupt with a specific ID
interrupt = Interrupt(value="Please approve this action", id="approval-1")

# Access interrupt information
print(interrupt.value)  # "Please approve this action"
print(interrupt.id)     # "approval-1"

# Create from namespace
interrupt2 = Interrupt.from_ns(value="data", ns="namespace")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:155-211`

---

### Overwrite

**Definition:**
```python
@dataclass(slots=True)
class Overwrite:
    value: Any
```

**Description:**
Bypass a reducer and write the wrapped value directly to a `BinaryOperatorAggregate` channel. This allows you to replace the entire value instead of using the channel's reducer (e.g., `operator.add`). Receiving multiple `Overwrite` values for the same channel in a single super-step will raise an `InvalidUpdateError`.

**Fields:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| value | `Any` | Yes | The value to write directly to the channel, bypassing any reducer |

**Example:**
```python
from typing import Annotated
import operator
from langgraph.graph import StateGraph
from langgraph.types import Overwrite
from typing_extensions import TypedDict

class State(TypedDict):
    messages: Annotated[list, operator.add]

def node_a(state: State):
    # Normal update: uses the reducer (operator.add)
    return {"messages": ["a"]}

def node_b(state: State):
    # Overwrite: bypasses the reducer and replaces the entire value
    return {"messages": Overwrite(value=["b"])}

builder = StateGraph(State)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.set_entry_point("node_a")
builder.add_edge("node_a", "node_b")
graph = builder.compile()

# Without Overwrite in node_b, messages would be ["START", "a", "b"]
# With Overwrite, messages is just ["b"]
result = graph.invoke({"messages": ["START"]})
assert result == {"messages": ["b"]}
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:542-584`

---

## Type Aliases

### Durability

**Definition:**
```python
Durability = Literal["sync", "async", "exit"]
```

**Description:**
Durability mode for the graph execution. Controls when changes are persisted to the checkpointer.

**Values:**

| Value | Description |
|-------|-------------|
| `"sync"` | Changes are persisted synchronously before the next step starts |
| `"async"` | Changes are persisted asynchronously while the next step executes |
| `"exit"` | Changes are persisted only when the graph exits |

**Example:**
```python
from langgraph.graph import StateGraph

# Specify durability mode when compiling
graph = builder.compile(
    checkpointer=checkpointer,
    durability="async"  # Persist asynchronously
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:62-66`

---

### All

**Definition:**
```python
All = Literal["*"]
```

**Description:**
Special value to indicate that graph should interrupt on all nodes.

**Example:**
```python
from langgraph.graph import StateGraph

# Interrupt on all nodes
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before="*"  # or interrupt_after="*"
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:68-69`

---

### Checkpointer

**Definition:**
```python
Checkpointer = None | bool | BaseCheckpointSaver
```

**Description:**
Type of the checkpointer to use for a subgraph.

**Values:**

| Value | Description |
|-------|-------------|
| `True` | Enables persistent checkpointing for this subgraph |
| `False` | Disables checkpointing, even if the parent graph has a checkpointer |
| `None` | Inherits checkpointer from the parent graph |
| `BaseCheckpointSaver` | A specific checkpointer instance to use |

**Example:**
```python
from langgraph.graph import StateGraph
from langgraph.checkpoint.memory import InMemorySaver

# Use a specific checkpointer
graph = builder.compile(checkpointer=InMemorySaver())

# Enable checkpointing (inherits from parent)
subgraph = subgraph_builder.compile(checkpointer=True)

# Disable checkpointing
no_checkpoint_subgraph = builder.compile(checkpointer=False)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:71-76`

---

### StreamMode

**Definition:**
```python
StreamMode = Literal[
    "values", "updates", "checkpoints", "tasks", "debug", "messages", "custom"
]
```

**Description:**
How the stream method should emit outputs.

**Values:**

| Mode | Description |
|------|-------------|
| `"values"` | Emit all values in the state after each step, including interrupts. When used with functional API, values are emitted once at the end of the workflow |
| `"updates"` | Emit only the node or task names and updates returned by the nodes or tasks after each step. If multiple updates are made in the same step, those updates are emitted separately |
| `"custom"` | Emit custom data from inside nodes or tasks using `StreamWriter` |
| `"messages"` | Emit LLM messages token-by-token together with metadata for any LLM invocations inside nodes or tasks |
| `"checkpoints"` | Emit an event when a checkpoint is created, in the same format as returned by `get_state()` |
| `"tasks"` | Emit events when tasks start and finish, including their results and errors |
| `"debug"` | Emit both `"checkpoints"` and `"tasks"` events for debugging purposes |

**Example:**
```python
from langgraph.graph import StateGraph

graph = builder.compile()

# Stream values
for chunk in graph.stream(inputs, stream_mode="values"):
    print(chunk)

# Stream updates only
for chunk in graph.stream(inputs, stream_mode="updates"):
    print(chunk)

# Stream multiple modes
for chunk in graph.stream(inputs, stream_mode=["values", "updates"]):
    print(chunk)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:91-105`

---

### StreamWriter

**Definition:**
```python
StreamWriter = Callable[[Any], None]
```

**Description:**
`Callable` that accepts a single argument and writes it to the output stream. Always injected into nodes if requested as a keyword argument, but it's a no-op when not using `stream_mode="custom"`.

**Example:**
```python
from langgraph.types import StreamWriter
from langgraph.graph import StateGraph
from typing_extensions import TypedDict

class State(TypedDict):
    value: int

def my_node(state: State, writer: StreamWriter):
    # Write custom data to the stream
    writer({"progress": 50})
    # ... do work ...
    writer({"progress": 100})
    return {"value": state["value"] + 1}

builder = StateGraph(State)
builder.add_node("my_node", my_node)
graph = builder.compile()

# Stream custom data
for chunk in graph.stream({"value": 0}, stream_mode="custom"):
    print(chunk)  # {"progress": 50}, {"progress": 100}
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:107-110`

---

## Policy Types

### RetryPolicy

**Definition:**
```python
class RetryPolicy(NamedTuple):
    initial_interval: float = 0.5
    backoff_factor: float = 2.0
    max_interval: float = 128.0
    max_attempts: int = 3
    jitter: bool = True
    retry_on: (
        type[Exception] | Sequence[type[Exception]] | Callable[[Exception], bool]
    ) = default_retry_on
```

**Description:**
Configuration for retrying nodes. Defines how nodes should be retried on failure, including exponential backoff and jitter.

**Fields:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| initial_interval | `float` | No | `0.5` | Amount of time that must elapse before the first retry occurs, in seconds |
| backoff_factor | `float` | No | `2.0` | Multiplier by which the interval increases after each retry |
| max_interval | `float` | No | `128.0` | Maximum amount of time that may elapse between retries, in seconds |
| max_attempts | `int` | No | `3` | Maximum number of attempts to make before giving up, including the first |
| jitter | `bool` | No | `True` | Whether to add random jitter to the interval between retries |
| retry_on | `type[Exception] \| Sequence[type[Exception]] \| Callable[[Exception], bool]` | No | `default_retry_on` | List of exception classes that should trigger a retry, or a callable that returns `True` for exceptions that should trigger a retry |

**Version History:**
- Added in version 0.2.24

**Example:**
```python
from langgraph.types import RetryPolicy
from langgraph.graph import StateGraph

# Create a custom retry policy
retry_policy = RetryPolicy(
    initial_interval=1.0,
    backoff_factor=2.0,
    max_interval=60.0,
    max_attempts=5,
    jitter=True,
    retry_on=ValueError  # Only retry on ValueError
)

builder = StateGraph(State)
builder.add_node("flaky_node", node_func, retry=retry_policy)
graph = builder.compile()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:115-135`

---

### CachePolicy

**Definition:**
```python
@dataclass(kw_only=True, slots=True, frozen=True)
class CachePolicy(Generic[KeyFuncT]):
    key_func: KeyFuncT = default_cache_key
    ttl: int | None = None
```

**Description:**
Configuration for caching nodes. Defines how node results should be cached to avoid redundant computation.

**Fields:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| key_func | `KeyFuncT` | No | `default_cache_key` | Function to generate a cache key from the node's input. Defaults to hashing the input with pickle |
| ttl | `int \| None` | No | `None` | Time to live for the cache entry in seconds. If `None`, the entry never expires |

**Example:**
```python
from langgraph.types import CachePolicy
from langgraph.graph import StateGraph

# Create a custom cache policy
def my_cache_key(input_data):
    # Custom logic to generate cache key
    return f"key_{input_data['id']}"

cache_policy = CachePolicy(
    key_func=my_cache_key,
    ttl=3600  # Cache for 1 hour
)

builder = StateGraph(State)
builder.add_node("cached_node", node_func, cache=cache_policy)
graph = builder.compile()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:140-150`

---

### CacheKey

**Definition:**
```python
class CacheKey(NamedTuple):
    ns: tuple[str, ...]
    key: str
    ttl: int | None
```

**Description:**
Cache key for a task. Used internally to represent cache entries.

**Fields:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| ns | `tuple[str, ...]` | Yes | Namespace for the cache entry |
| key | `str` | Yes | Key for the cache entry |
| ttl | `int \| None` | Yes | Time to live for the cache entry in seconds |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:237-246`

---

## State Management Types

### StateSnapshot

**Definition:**
```python
class StateSnapshot(NamedTuple):
    values: dict[str, Any] | Any
    next: tuple[str, ...]
    config: RunnableConfig
    metadata: CheckpointMetadata | None
    created_at: str | None
    parent_config: RunnableConfig | None
    tasks: tuple[PregelTask, ...]
    interrupts: tuple[Interrupt, ...]
```

**Description:**
Snapshot of the state of the graph at the beginning of a step. Contains all information about the graph's current execution state.

**Fields:**

| Name | Type | Required | Description |
|------|------|----------|-------------|
| values | `dict[str, Any] \| Any` | Yes | Current values of channels |
| next | `tuple[str, ...]` | Yes | The name of the node to execute in each task for this step |
| config | `RunnableConfig` | Yes | Config used to fetch this snapshot |
| metadata | `CheckpointMetadata \| None` | Yes | Metadata associated with this snapshot |
| created_at | `str \| None` | Yes | Timestamp of snapshot creation |
| parent_config | `RunnableConfig \| None` | Yes | Config used to fetch the parent snapshot, if any |
| tasks | `tuple[PregelTask, ...]` | Yes | Tasks to execute in this step. If already attempted, may contain an error |
| interrupts | `tuple[Interrupt, ...]` | Yes | Interrupts that occurred in this step that are pending resolution |

**Example:**
```python
from langgraph.graph import StateGraph

graph = builder.compile(checkpointer=checkpointer)

# Get current state snapshot
config = {"configurable": {"thread_id": "1"}}
snapshot = graph.get_state(config)

print(snapshot.values)       # Current state values
print(snapshot.next)         # Next nodes to execute
print(snapshot.tasks)        # Pending tasks
print(snapshot.interrupts)   # Active interrupts
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:264-283`

---

### StateUpdate

**Definition:**
```python
class StateUpdate(NamedTuple):
    values: dict[str, Any] | None
    as_node: str | None = None
    task_id: str | None = None
```

**Description:**
Represents an update to the graph state. Used when updating state externally.

**Fields:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| values | `dict[str, Any] \| None` | Yes | - | The state values to update |
| as_node | `str \| None` | No | `None` | The node name to attribute this update to |
| task_id | `str \| None` | No | `None` | The task ID associated with this update |

**Example:**
```python
from langgraph.types import StateUpdate
from langgraph.graph import StateGraph

graph = builder.compile(checkpointer=checkpointer)
config = {"configurable": {"thread_id": "1"}}

# Update state as if "node_a" produced this update
update = StateUpdate(
    values={"value": 42},
    as_node="node_a"
)

graph.update_state(config, update)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:213-217`

---

## Task Types

### PregelTask

**Definition:**
```python
class PregelTask(NamedTuple):
    id: str
    name: str
    path: tuple[str | int | tuple, ...]
    error: Exception | None = None
    interrupts: tuple[Interrupt, ...] = ()
    state: None | RunnableConfig | StateSnapshot = None
    result: Any | None = None
```

**Description:**
A Pregel task. Represents a single unit of work to be executed in the graph.

**Fields:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| id | `str` | Yes | - | Unique identifier for the task |
| name | `str` | Yes | - | Name of the node being executed |
| path | `tuple[str \| int \| tuple, ...]` | Yes | - | Path to this task in the execution tree |
| error | `Exception \| None` | No | `None` | Error that occurred during task execution, if any |
| interrupts | `tuple[Interrupt, ...]` | No | `()` | Interrupts that occurred during task execution |
| state | `None \| RunnableConfig \| StateSnapshot` | No | `None` | State snapshot for subgraph tasks |
| result | `Any \| None` | No | `None` | Result of the task execution |

**Example:**
```python
# Tasks are typically accessed from StateSnapshot
snapshot = graph.get_state(config)

for task in snapshot.tasks:
    print(f"Task ID: {task.id}")
    print(f"Node: {task.name}")
    if task.error:
        print(f"Error: {task.error}")
    if task.interrupts:
        print(f"Interrupts: {task.interrupts}")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:219-229`

---

### PregelExecutableTask

**Definition:**
```python
@dataclass(weakref_slot=True, slots=True, frozen=True)  # Python 3.11+
# or @dataclass(frozen=True) for Python < 3.11
class PregelExecutableTask:
    name: str
    input: Any
    proc: Runnable
    writes: deque[tuple[str, Any]]
    config: RunnableConfig
    triggers: Sequence[str]
    retry_policy: Sequence[RetryPolicy]
    cache_key: CacheKey | None
    id: str
    path: tuple[str | int | tuple, ...]
    writers: Sequence[Runnable] = ()
    subgraphs: Sequence[PregelProtocol] = ()
```

**Description:**
Internal representation of an executable task. Contains all information needed to execute a task, including retry policies, cache keys, and writers.

**Fields:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| name | `str` | Yes | - | Name of the node being executed |
| input | `Any` | Yes | - | Input to the node |
| proc | `Runnable` | Yes | - | The runnable to execute |
| writes | `deque[tuple[str, Any]]` | Yes | - | Queue of writes to apply |
| config | `RunnableConfig` | Yes | - | Configuration for execution |
| triggers | `Sequence[str]` | Yes | - | List of nodes that triggered this task |
| retry_policy | `Sequence[RetryPolicy]` | Yes | - | Retry policies to apply |
| cache_key | `CacheKey \| None` | Yes | - | Cache key for this task, if caching is enabled |
| id | `str` | Yes | - | Unique identifier for the task |
| path | `tuple[str \| int \| tuple, ...]` | Yes | - | Path to this task in the execution tree |
| writers | `Sequence[Runnable]` | No | `()` | Writer runnables to execute after the main proc |
| subgraphs | `Sequence[PregelProtocol]` | No | `()` | Subgraphs to execute |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:248-262`

---

## Constants

### START

**Definition:**
```python
START = sys.intern("__start__")
```

**Description:**
The first (maybe virtual) node in graph-style Pregel. Used to define the entry point of a graph.

**Example:**
```python
from langgraph.constants import START
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node("first_node", first_node_func)
builder.add_edge(START, "first_node")
graph = builder.compile()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/constants.py:30-31`

---

### END

**Definition:**
```python
END = sys.intern("__end__")
```

**Description:**
The last (maybe virtual) node in graph-style Pregel. Used to define the exit point of a graph.

**Example:**
```python
from langgraph.constants import END
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node("last_node", last_node_func)
builder.add_edge("last_node", END)
graph = builder.compile()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/constants.py:28-29`

---

### TAG_NOSTREAM

**Definition:**
```python
TAG_NOSTREAM = sys.intern("nostream")
```

**Description:**
Tag to disable streaming for a chat model. When added to a node's tags, prevents the node from streaming output.

**Example:**
```python
from langgraph.constants import TAG_NOSTREAM
from langgraph.graph import StateGraph

def my_node(state):
    # This node will not stream
    return {"value": "result"}

builder = StateGraph(State)
builder.add_node("my_node", my_node, tags=[TAG_NOSTREAM])
graph = builder.compile()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/constants.py:24-25`

---

### TAG_HIDDEN

**Definition:**
```python
TAG_HIDDEN = sys.intern("langsmith:hidden")
```

**Description:**
Tag to hide a node/edge from certain tracing/streaming environments. When added to a node's tags, the node will be hidden from LangSmith traces and similar observability tools.

**Example:**
```python
from langgraph.constants import TAG_HIDDEN
from langgraph.graph import StateGraph

def internal_node(state):
    # This node will be hidden from traces
    return {"value": "internal"}

builder = StateGraph(State)
builder.add_node("internal_node", internal_node, tags=[TAG_HIDDEN])
graph = builder.compile()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/constants.py:26-27`

---

## Error Classes

### ErrorCode

**Definition:**
```python
class ErrorCode(Enum):
    GRAPH_RECURSION_LIMIT = "GRAPH_RECURSION_LIMIT"
    INVALID_CONCURRENT_GRAPH_UPDATE = "INVALID_CONCURRENT_GRAPH_UPDATE"
    INVALID_GRAPH_NODE_RETURN_VALUE = "INVALID_GRAPH_NODE_RETURN_VALUE"
    MULTIPLE_SUBGRAPHS = "MULTIPLE_SUBGRAPHS"
    INVALID_CHAT_HISTORY = "INVALID_CHAT_HISTORY"
```

**Description:**
Enumeration of error codes used in LangGraph exceptions. Each error code corresponds to a specific troubleshooting guide in the documentation.

**Values:**

| Code | Description |
|------|-------------|
| `GRAPH_RECURSION_LIMIT` | Graph has exhausted the maximum number of steps |
| `INVALID_CONCURRENT_GRAPH_UPDATE` | Invalid concurrent update to graph state |
| `INVALID_GRAPH_NODE_RETURN_VALUE` | Node returned an invalid value |
| `MULTIPLE_SUBGRAPHS` | Multiple subgraphs detected in invalid configuration |
| `INVALID_CHAT_HISTORY` | Invalid chat history format |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:29-35`

---

### GraphRecursionError

**Definition:**
```python
class GraphRecursionError(RecursionError):
    pass
```

**Description:**
Raised when the graph has exhausted the maximum number of steps. This prevents infinite loops. To increase the maximum number of steps, run your graph with a config specifying a higher `recursion_limit`.

**Example:**
```python
from langgraph.graph import StateGraph

graph = builder.compile()

try:
    graph.invoke(
        {"messages": [("user", "Hello, world!")]},
        # The config is the second positional argument
        {"recursion_limit": 1000},
    )
except GraphRecursionError:
    print("Graph exceeded recursion limit")
```

**Troubleshooting:**
- [GRAPH_RECURSION_LIMIT](https://docs.langchain.com/oss/python/langgraph/errors/GRAPH_RECURSION_LIMIT)

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:45-65`

---

### InvalidUpdateError

**Definition:**
```python
class InvalidUpdateError(Exception):
    pass
```

**Description:**
Raised when attempting to update a channel with an invalid set of updates. This can occur when:
- Multiple nodes try to write to the same channel without a reducer
- Multiple `Overwrite` values are provided for the same channel in a single super-step
- A node returns an invalid update format

**Troubleshooting:**
- [INVALID_CONCURRENT_GRAPH_UPDATE](https://docs.langchain.com/oss/python/langgraph/errors/INVALID_CONCURRENT_GRAPH_UPDATE)
- [INVALID_GRAPH_NODE_RETURN_VALUE](https://docs.langchain.com/oss/python/langgraph/errors/INVALID_GRAPH_NODE_RETURN_VALUE)

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:68-77`

---

### GraphBubbleUp

**Definition:**
```python
class GraphBubbleUp(Exception):
    pass
```

**Description:**
Internal exception used to bubble up control flow from subgraphs to parent graphs. Not typically raised directly or surfaced to users.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:80-81`

---

### GraphInterrupt

**Definition:**
```python
class GraphInterrupt(GraphBubbleUp):
    def __init__(self, interrupts: Sequence[Interrupt] = ()) -> None:
        super().__init__(interrupts)
```

**Description:**
Raised when a subgraph is interrupted. This exception is suppressed by the root graph and never raised directly or surfaced to the user. Contains a sequence of `Interrupt` objects representing all active interrupts.

**Parameters:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| interrupts | `Sequence[Interrupt]` | `()` | Sequence of interrupt objects |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:84-90`

---

### NodeInterrupt

**Definition:**
```python
@deprecated(
    "NodeInterrupt is deprecated. Please use `interrupt` instead.",
    category=None,
)
class NodeInterrupt(GraphInterrupt):
    def __init__(self, value: Any, id: str | None = None) -> None:
        ...
```

**Description:**
**DEPRECATED:** Use [`interrupt()`](#interrupt-1) instead. Previously used by nodes to interrupt execution.

**Parameters:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| value | `Any` | - | The value associated with the interrupt |
| id | `str \| None` | `None` | Optional interrupt ID |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:92-109`

---

### ParentCommand

**Definition:**
```python
class ParentCommand(GraphBubbleUp):
    args: tuple[Command]

    def __init__(self, command: Command) -> None:
        super().__init__(command)
```

**Description:**
Internal exception used to send commands to parent graphs. Raised when a `Command` with `graph=Command.PARENT` is returned from a node.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| command | `Command` | The command to send to the parent graph |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:111-116`

---

### EmptyInputError

**Definition:**
```python
class EmptyInputError(Exception):
    pass
```

**Description:**
Raised when graph receives an empty input. This typically occurs when invoking a graph with `None` or an empty dictionary when input is required.

**Example:**
```python
from langgraph.errors import EmptyInputError

try:
    graph.invoke(None)
except EmptyInputError:
    print("Graph requires non-empty input")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:118-121`

---

### TaskNotFound

**Definition:**
```python
class TaskNotFound(Exception):
    pass
```

**Description:**
Raised when the executor is unable to find a task. This is primarily used in distributed execution mode when a task cannot be located.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:124-127`

---

### EmptyChannelError

**Definition:**
```python
# Re-exported from langgraph.checkpoint.base
from langgraph.checkpoint.base import EmptyChannelError
```

**Description:**
Raised when attempting to read from an empty channel. This can occur when a node tries to access a state channel that hasn't been initialized yet.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/errors.py:9`

---

## Configuration Functions

### get_config

**Signature:**
```python
def get_config() -> RunnableConfig:
```

**Description:**
Get the current `RunnableConfig` from within a node or task at runtime. Uses context variables to retrieve the configuration that was passed when invoking the graph.

**Returns:**
- `RunnableConfig`: The current runnable configuration

**Raises:**
- `RuntimeError`: If called outside of a runnable context
- `RuntimeError`: If called in an async context with Python < 3.11

**Limitations:**
- **Python < 3.11 with async**: If you are using Python < 3.11 and are running LangGraph asynchronously, `get_config()` won't work since it relies on `contextvar` propagation (only available in Python >= 3.11).

**Example:**
```python
from langgraph.config import get_config
from langgraph.graph import StateGraph
from typing_extensions import TypedDict

class State(TypedDict):
    value: int

def my_node(state: State):
    config = get_config()
    print(f"Thread ID: {config['configurable']['thread_id']}")
    print(f"Recursion limit: {config.get('recursion_limit', 25)}")
    return {"value": state["value"] + 1}

builder = StateGraph(State)
builder.add_node("my_node", my_node)
graph = builder.compile()

graph.invoke(
    {"value": 0},
    {"configurable": {"thread_id": "1"}, "recursion_limit": 100}
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/config.py:17-29`

---

### get_store

**Signature:**
```python
def get_store() -> BaseStore:
```

**Description:**
Access LangGraph store from inside a graph node or entrypoint task at runtime. Can be called from inside any `StateGraph` node or functional API `task`, as long as the `StateGraph` or the `entrypoint` was initialized with a store.

**Returns:**
- `BaseStore`: The store instance configured for the graph

**Raises:**
- `RuntimeError`: If called outside of a runnable context
- `RuntimeError`: If called in an async context with Python < 3.11
- `AttributeError`: If no store was configured

**Limitations:**
- **Python < 3.11 with async**: If you are using Python < 3.11 and are running LangGraph asynchronously, `get_store()` won't work since it relies on `contextvar` propagation.

**Example (StateGraph):**
```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START
from langgraph.store.memory import InMemoryStore
from langgraph.config import get_store

store = InMemoryStore()
store.put(("values",), "foo", {"bar": 2})

class State(TypedDict):
    foo: int

def my_node(state: State):
    my_store = get_store()
    stored_value = my_store.get(("values",), "foo").value["bar"]
    return {"foo": stored_value + 1}

graph = (
    StateGraph(State)
    .add_node(my_node)
    .add_edge(START, "my_node")
    .compile(store=store)
)

result = graph.invoke({"foo": 1})
# {"foo": 3}
```

**Example (Functional API):**
```python
from langgraph.func import entrypoint, task
from langgraph.store.memory import InMemoryStore
from langgraph.config import get_store

store = InMemoryStore()
store.put(("values",), "foo", {"bar": 2})

@task
def my_task(value: int):
    my_store = get_store()
    stored_value = my_store.get(("values",), "foo").value["bar"]
    return stored_value + 1

@entrypoint(store=store)
def workflow(value: int):
    return my_task(value).result()

result = workflow.invoke(1)
# 3
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/config.py:32-123`

---

### get_stream_writer

**Signature:**
```python
def get_stream_writer() -> StreamWriter:
```

**Description:**
Access LangGraph `StreamWriter` from inside a graph node or entrypoint task at runtime. Can be called from inside any `StateGraph` node or functional API `task`. The stream writer allows you to emit custom data to the output stream when using `stream_mode="custom"`.

**Returns:**
- `StreamWriter`: A callable that writes data to the custom stream

**Raises:**
- `RuntimeError`: If called outside of a runnable context
- `RuntimeError`: If called in an async context with Python < 3.11

**Limitations:**
- **Python < 3.11 with async**: If you are using Python < 3.11 and are running LangGraph asynchronously, `get_stream_writer()` won't work since it relies on `contextvar` propagation.

**Example (StateGraph):**
```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START
from langgraph.config import get_stream_writer

class State(TypedDict):
    foo: int

def my_node(state: State):
    my_stream_writer = get_stream_writer()
    my_stream_writer({"custom_data": "Hello!"})
    return {"foo": state["foo"] + 1}

graph = (
    StateGraph(State)
    .add_node(my_node)
    .add_edge(START, "my_node")
    .compile()
)

for chunk in graph.stream({"foo": 1}, stream_mode="custom"):
    print(chunk)
# {"custom_data": "Hello!"}
```

**Example (Functional API):**
```python
from langgraph.func import entrypoint, task
from langgraph.config import get_stream_writer

@task
def my_task(value: int):
    my_stream_writer = get_stream_writer()
    my_stream_writer({"custom_data": "Hello!"})
    return value + 1

@entrypoint()
def workflow(value: int):
    return my_task(value).result()

for chunk in workflow.stream(1, stream_mode="custom"):
    print(chunk)
# {"custom_data": "Hello!"}
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/config.py:126-197`

---

## Core Functions

### interrupt

**Signature:**
```python
def interrupt(value: Any) -> Any:
```

**Description:**
Interrupt the graph with a resumable exception from within a node. The `interrupt` function enables human-in-the-loop workflows by pausing graph execution and surfacing a value to the client. This value can communicate context or request input required to resume execution.

In a given node, the first invocation of this function raises a `GraphInterrupt` exception, halting execution. The provided `value` is included with the exception and sent to the client executing the graph.

A client resuming the graph must use the `Command` primitive to specify a value for the interrupt and continue execution. The graph resumes from the start of the node, **re-executing** all logic.

If a node contains multiple `interrupt` calls, LangGraph matches resume values to interrupts based on their order in the node. This list of resume values is scoped to the specific task executing the node and is not shared across tasks.

**Requirements:**
- A checkpointer must be enabled, as interrupts rely on persisting the graph state

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| value | `Any` | The value to surface to the client when the graph is interrupted |

**Returns:**
- `Any`: On subsequent invocations within the same node (same task to be precise), returns the value provided when resuming from the interrupt

**Raises:**
- `GraphInterrupt`: On the first invocation within the node, halts execution and surfaces the provided value to the client

**Example:**
```python
import uuid
from typing import Optional
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.constants import START
from langgraph.graph import StateGraph
from langgraph.types import interrupt, Command

class State(TypedDict):
    """The graph state."""
    foo: str
    human_value: Optional[str]
    """Human value will be updated using an interrupt."""

def node(state: State):
    answer = interrupt(
        # This value will be sent to the client
        # as part of the interrupt information.
        "what is your age?"
    )
    print(f"> Received an input from the interrupt: {answer}")
    return {"human_value": answer}

builder = StateGraph(State)
builder.add_node("node", node)
builder.add_edge(START, "node")

# A checkpointer must be enabled for interrupts to work!
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

config = {
    "configurable": {
        "thread_id": uuid.uuid4(),
    }
}

# First invocation - will interrupt
for chunk in graph.stream({"foo": "abc"}, config):
    print(chunk)
# > {'__interrupt__': (Interrupt(value='what is your age?', id='45fda8478b2ef754419799e10992af06'),)}

# Resume with a value
command = Command(resume="some input from a human!!!")

for chunk in graph.stream(command, config):
    print(chunk)
# > Received an input from the interrupt: some input from a human!!!
# > {'node': {'human_value': 'some input from a human!!!'}}
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:416-539`

---

### ensure_valid_checkpointer

**Signature:**
```python
def ensure_valid_checkpointer(checkpointer: Checkpointer) -> Checkpointer:
```

**Description:**
Validates that a checkpointer value is valid. Ensures the checkpointer is either `None`, `True`, `False`, or an instance of `BaseCheckpointSaver`.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| checkpointer | `Checkpointer` | The checkpointer to validate |

**Returns:**
- `Checkpointer`: The same checkpointer if valid

**Raises:**
- `TypeError`: If the checkpointer is not a valid type

**Example:**
```python
from langgraph.types import ensure_valid_checkpointer
from langgraph.checkpoint.memory import InMemorySaver

# Valid checkpointers
ensure_valid_checkpointer(None)           # OK
ensure_valid_checkpointer(True)           # OK
ensure_valid_checkpointer(False)          # OK
ensure_valid_checkpointer(InMemorySaver())  # OK

# Invalid checkpointer
try:
    ensure_valid_checkpointer("invalid")
except TypeError as e:
    print(e)
# Invalid checkpointer provided. Expected an instance of
# `BaseCheckpointSaver`, `True`, `False`, or `None`.
# Received str. Pass a proper saver (e.g., InMemorySaver, AsyncPostgresSaver).
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:78-88`

---

## Utility Classes

### RunnableCallable

**Definition:**
```python
class RunnableCallable(Runnable):
    def __init__(
        self,
        func: Callable[..., Any | Runnable] | None,
        afunc: Callable[..., Awaitable[Any | Runnable]] | None = None,
        *,
        name: str | None = None,
        tags: Sequence[str] | None = None,
        trace: bool = True,
        recurse: bool = True,
        explode_args: bool = False,
        **kwargs: Any,
    ) -> None:
        ...
```

**Description:**
A much simpler version of `RunnableLambda` that requires sync and async functions. This is an internal utility class used by LangGraph to wrap functions as `Runnable` objects with support for dependency injection of `config`, `writer`, `store`, and other runtime parameters.

**Parameters:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| func | `Callable[..., Any \| Runnable] \| None` | - | Synchronous function to wrap. At least one of `func` or `afunc` must be provided |
| afunc | `Callable[..., Awaitable[Any \| Runnable]] \| None` | `None` | Asynchronous function to wrap |
| name | `str \| None` | `None` | Name for the runnable. If not provided, inferred from function name |
| tags | `Sequence[str] \| None` | `None` | Tags to attach to the runnable |
| trace | `bool` | `True` | Whether to enable tracing for this runnable |
| recurse | `bool` | `True` | Whether to recursively invoke if the result is a Runnable |
| explode_args | `bool` | `False` | Whether to explode args tuple into positional arguments |
| **kwargs | `Any` | - | Additional keyword arguments to pass to the function |

**Supported Injected Parameters:**
The function can request these parameters via keyword arguments with proper type annotations:
- `config: RunnableConfig` - The current configuration
- `writer: StreamWriter` - Stream writer for custom output
- `store: BaseStore` - The configured store
- `previous: Any` - Previous value in a sequence

**Methods:**
- `invoke(input, config=None, **kwargs)`: Synchronously invoke the wrapped function
- `ainvoke(input, config=None, **kwargs)`: Asynchronously invoke the wrapped function

**Example:**
```python
from langgraph._internal._runnable import RunnableCallable
from langgraph.types import StreamWriter
from langchain_core.runnables import RunnableConfig

# Function with injected parameters
def my_func(input_value: int, config: RunnableConfig, writer: StreamWriter):
    writer({"progress": "starting"})
    result = input_value * 2
    writer({"progress": "done"})
    return result

# Wrap as RunnableCallable
runnable = RunnableCallable(my_func, name="my_func")

# The config and writer will be injected automatically
result = runnable.invoke(5, config={...})
```

**Note:** This is an internal utility class. For public use, prefer using standard `Runnable` types or the functional API.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/_internal/_runnable.py:254-477`

---

## Utility Functions

### ensure_config

**Signature:**
```python
def ensure_config(config: RunnableConfig | None = None) -> RunnableConfig:
```

**Description:**
Internal utility function to ensure a valid `RunnableConfig` exists. If no config is provided, returns a minimal default configuration. This is used internally by LangGraph to ensure nodes always have a valid configuration.

**Parameters:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| config | `RunnableConfig \| None` | `None` | Optional configuration to validate/normalize |

**Returns:**
- `RunnableConfig`: A valid runnable configuration

**Note:** This is an internal utility function re-exported for backward compatibility. It may be removed in future versions.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/utils/config.py:3`

---

### patch_configurable

**Signature:**
```python
def patch_configurable(
    config: RunnableConfig | None,
    patch: dict[str, Any]
) -> RunnableConfig:
```

**Description:**
Internal utility function to patch the `configurable` section of a `RunnableConfig`. Merges the provided patch dictionary into the config's configurable section.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | `RunnableConfig \| None` | The configuration to patch |
| patch | `dict[str, Any]` | Dictionary of values to merge into the configurable section |

**Returns:**
- `RunnableConfig`: A new configuration with the patched configurable section

**Example:**
```python
from langgraph._internal._config import patch_configurable

config = {
    "configurable": {"thread_id": "1"}
}

new_config = patch_configurable(config, {"user_id": "user123"})
# {
#     "configurable": {
#         "thread_id": "1",
#         "user_id": "user123"
#     }
# }
```

**Note:** This is an internal utility function re-exported for backward compatibility. It may be removed in future versions.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/_internal/_config.py:47-56`

---

## Managed Values

### ManagedValue

**Definition:**
```python
class ManagedValue(ABC, Generic[V]):
    @staticmethod
    @abstractmethod
    def get(scratchpad: PregelScratchpad) -> V:
        ...
```

**Description:**
Abstract base class for managed values. Managed values are special state values that are computed dynamically at runtime based on the current execution context. They are injected into nodes via type annotations and provide access to runtime information like whether this is the last step or how many steps remain.

**Type Parameters:**

| Name | Description |
|------|-------------|
| V | The type of value this managed value provides |

**Methods:**

| Name | Signature | Description |
|------|-----------|-------------|
| get | `get(scratchpad: PregelScratchpad) -> V` | Static abstract method that computes the value from the execution scratchpad |

**Example:**
```python
from langgraph.managed.base import ManagedValue
from langgraph._internal._scratchpad import PregelScratchpad

class MyManagedValue(ManagedValue[str]):
    @staticmethod
    def get(scratchpad: PregelScratchpad) -> str:
        return f"Step {scratchpad.step} of {scratchpad.stop}"

# Use in a type annotation
from typing import Annotated

MyValue = Annotated[str, MyManagedValue]

def my_node(state: State, my_value: MyValue):
    print(f"Current execution: {my_value}")
    return state
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/managed/base.py:18-22`

---

### ManagedValueSpec

**Definition:**
```python
ManagedValueSpec = type[ManagedValue]
```

**Description:**
Type alias for a managed value class (not an instance). Represents the type of a `ManagedValue` subclass.

**Example:**
```python
from langgraph.managed.base import ManagedValueSpec, ManagedValue

class MyManagedValue(ManagedValue[int]):
    @staticmethod
    def get(scratchpad):
        return 42

# MyManagedValue is a ManagedValueSpec
spec: ManagedValueSpec = MyManagedValue
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/managed/base.py:24`

---

### ManagedValueMapping

**Definition:**
```python
ManagedValueMapping = dict[str, ManagedValueSpec]
```

**Description:**
Type alias for a dictionary mapping parameter names to managed value specifications. Used internally to track which parameters should be injected with managed values.

**Example:**
```python
from langgraph.managed.base import ManagedValueMapping
from langgraph.managed.is_last_step import IsLastStepManager

# Map parameter names to managed value types
mapping: ManagedValueMapping = {
    "is_last": IsLastStepManager,
}
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/managed/base.py:31`

---

### IsLastStep

**Definition:**
```python
class IsLastStepManager(ManagedValue[bool]):
    @staticmethod
    def get(scratchpad: PregelScratchpad) -> bool:
        return scratchpad.step == scratchpad.stop - 1

IsLastStep = Annotated[bool, IsLastStepManager]
```

**Description:**
Managed value that indicates whether the current step is the last step in the graph execution. Computed dynamically based on the current step number and recursion limit.

**Type:**
- `Annotated[bool, IsLastStepManager]`

**Usage:**
Use as a type annotation on a node parameter to get automatic injection of the last-step flag.

**Example:**
```python
from langgraph.managed import IsLastStep
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class State(TypedDict):
    value: int

def my_node(state: State, is_last: IsLastStep):
    if is_last:
        print("This is the last step!")
        return {"value": state["value"]}
    else:
        print("More steps to come...")
        return {"value": state["value"] + 1}

builder = StateGraph(State)
builder.add_node("my_node", my_node)
builder.add_edge(START, "my_node")
builder.add_edge("my_node", END)
graph = builder.compile()

# Run with recursion_limit=5
graph.invoke({"value": 0}, {"recursion_limit": 5})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/managed/is_last_step.py:9-15`

---

### RemainingSteps

**Definition:**
```python
class RemainingStepsManager(ManagedValue[int]):
    @staticmethod
    def get(scratchpad: PregelScratchpad) -> int:
        return scratchpad.stop - scratchpad.step

RemainingSteps = Annotated[int, RemainingStepsManager]
```

**Description:**
Managed value that indicates how many steps remain in the graph execution. Computed dynamically as the difference between the recursion limit and the current step number.

**Type:**
- `Annotated[int, RemainingStepsManager]`

**Usage:**
Use as a type annotation on a node parameter to get automatic injection of the remaining steps count.

**Example:**
```python
from langgraph.managed import RemainingSteps
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

class State(TypedDict):
    value: int

def my_node(state: State, remaining: RemainingSteps):
    print(f"Steps remaining: {remaining}")

    if remaining <= 2:
        print("Almost done!")

    return {"value": state["value"] + 1}

builder = StateGraph(State)
builder.add_node("my_node", my_node)
builder.add_edge(START, "my_node")
builder.add_edge("my_node", END)
graph = builder.compile()

# Run with recursion_limit=10
graph.invoke({"value": 0}, {"recursion_limit": 10})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/managed/is_last_step.py:18-24`

---

## Additional Notes

### Context Variable Support

Several functions (`get_config`, `get_store`, `get_stream_writer`) rely on Python's `contextvar` mechanism to provide runtime access to configuration and services. **Important limitation**: In Python < 3.11, these functions do not work in asynchronous contexts because `contextvar` propagation in async tasks was added in Python 3.11.

### Dependency Injection

LangGraph supports automatic dependency injection for node functions. When a node function includes properly typed keyword parameters, LangGraph will automatically inject:

- `config: RunnableConfig` - The current configuration
- `writer: StreamWriter` - Stream writer for custom output (no-op if not using `stream_mode="custom"`)
- `store: BaseStore` - The configured store (if one was provided)
- Managed values like `IsLastStep` and `RemainingSteps`

Example:
```python
from langgraph.types import StreamWriter
from langgraph.store.base import BaseStore
from langchain_core.runnables import RunnableConfig
from langgraph.managed import IsLastStep

def my_node(
    state: State,
    config: RunnableConfig,
    writer: StreamWriter,
    store: BaseStore,
    is_last: IsLastStep
):
    # All parameters are automatically injected
    writer({"status": "processing"})
    data = store.get(("namespace",), "key")
    print(f"Thread: {config['configurable']['thread_id']}")
    print(f"Last step: {is_last}")
    return {"value": state["value"] + 1}
```

### Version History

Many types and functions have version history annotations indicating when they were added or changed. Always check the version history in the docstrings for compatibility information.

---

This concludes the comprehensive API reference for LangGraph Types and Utilities. For more information, see the [LangGraph documentation](https://langchain-ai.github.io/langgraph/).
