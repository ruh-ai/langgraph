# LangGraph Core Architecture

## Overview

### What is LangGraph?

LangGraph is a **low-level orchestration framework** for building, managing, and deploying long-running, stateful agents and workflows. Unlike high-level abstractions that hide implementation details, LangGraph provides the foundational infrastructure needed for any stateful, multi-actor workflow while giving developers full control over prompts and architecture.

**Key Characteristics:**
- **Stateful by Design**: Built for workflows that need to persist state across multiple steps
- **Durable Execution**: Agents can survive failures and resume from exactly where they left off
- **Human-in-the-Loop**: Native support for inspecting and modifying agent state during execution
- **Production-Ready**: Designed to handle the unique challenges of long-running, stateful workflows at scale

**Trusted by**: Klarna, Replit, Elastic, and many other companies shaping the future of agents.

### Quick Example

```python
from langgraph.graph import START, StateGraph
from typing_extensions import TypedDict


class State(TypedDict):
    text: str


def node_a(state: State) -> dict:
    return {"text": state["text"] + "a"}


def node_b(state: State) -> dict:
    return {"text": state["text"] + "b"}


graph = StateGraph(State)
graph.add_node("node_a", node_a)
graph.add_node("node_b", node_b)
graph.add_edge(START, "node_a")
graph.add_edge("node_a", "node_b")

print(graph.compile().invoke({"text": ""}))
# {'text': 'ab'}
```

---

## Core Philosophy: The Pregel Computation Model

LangGraph is inspired by **Google's Pregel** (a system for large-scale graph processing) and **Apache Beam**. The Pregel model provides a natural fit for agent workflows where:

1. **Nodes** represent computational units (functions, agents, tools)
2. **Edges** define the flow of information and control
3. **State** flows through the graph and can be updated by nodes
4. **Iterations** allow for cyclic workflows (unlike traditional DAGs)

### Why Pregel for Agents?

Traditional workflow orchestrators use **Directed Acyclic Graphs (DAGs)**, which cannot handle cycles. Agent workflows often need:
- **Iterative reasoning**: An agent might need to revise its thinking multiple times
- **Dynamic routing**: Conditional logic that can loop back to earlier steps
- **Human intervention**: The ability to pause, inspect, and resume execution
- **State persistence**: Maintaining context across potentially long-running workflows

The Pregel model, adapted for agents, provides all of these capabilities while maintaining a clear, graph-based mental model.

### Graph-Style Pregel Concepts

In LangGraph's implementation:
- Every graph has virtual **START** and **END** nodes that bookend execution
- Nodes can be executed in parallel when they don't depend on each other
- State updates from nodes are merged according to defined reduction logic
- Checkpointing enables time-travel debugging and resumable execution

---

## Key Constants

### Special Node Identifiers

#### `START`
```python
START = sys.intern("__start__")
```
**Purpose**: The first (possibly virtual) node in every graph-style Pregel execution.

**Usage**:
- Used to define the entry point(s) of your graph
- Connect START to one or more initial nodes
- The graph begins execution from nodes connected to START

**Example**:
```python
graph.add_edge(START, "initial_node")
# Or with conditional routing
graph.add_conditional_edges(START, routing_function)
```

#### `END`
```python
END = sys.intern("__end__")
```
**Purpose**: The last (possibly virtual) node in every graph-style Pregel execution.

**Usage**:
- Signals that a branch of execution has completed
- Multiple nodes can connect to END
- When all active branches reach END, the graph execution terminates

**Example**:
```python
graph.add_edge("final_node", END)
# Or from conditional logic
def should_continue(state):
    if state["done"]:
        return END
    return "continue_processing"
```

### Streaming and Visibility Tags

#### `TAG_NOSTREAM`
```python
TAG_NOSTREAM = sys.intern("nostream")
```
**Purpose**: Tag to disable streaming for a chat model or specific operations.

**Usage**: Apply to runnable configs to prevent token-by-token streaming.

#### `TAG_HIDDEN`
```python
TAG_HIDDEN = sys.intern("langsmith:hidden")
```
**Purpose**: Tag to hide a node or edge from certain tracing/streaming environments.

**Usage**: Useful for internal implementation details that shouldn't appear in user-facing traces or LangSmith visualizations.

### Deprecated Constants (Retained for Backwards Compatibility)

The following constants are retained but should not be used directly:
- `CONF`: Internal configuration constant
- `TASKS`: Internal task tracking constant
- `CONFIG_KEY_CHECKPOINTER`: Internal checkpointer configuration key

These will be removed in version 2.0.

---

## Type System

LangGraph's type system provides the building blocks for defining agent behavior, controlling execution flow, and managing state.

### Execution Control Types

#### `Command`

The `Command` type is the primary mechanism for controlling graph execution from within nodes.

```python
@dataclass(kw_only=True, slots=True, frozen=True)
class Command(Generic[N], ToolOutputMixin):
    graph: str | None = None
    update: Any | None = None
    resume: dict[str, Any] | Any | None = None
    goto: Send | Sequence[Send | N] | N = ()
```

**Purpose**: Allows nodes to return commands that:
1. Update the graph's state
2. Resume from interrupts
3. Control which node(s) execute next
4. Communicate with parent graphs

**Attributes**:
- `graph`: Target graph for the command
  - `None`: Current graph (default)
  - `Command.PARENT`: Closest parent graph (for subgraph communication)
- `update`: State update to apply (dict, tuple of tuples, or state object)
- `resume`: Value(s) to resume interrupted execution
  - Can be a mapping of interrupt IDs to resume values
  - Or a single value for the next interrupt
- `goto`: Navigation control
  - Node name (string) to execute next
  - Sequence of node names
  - `Send` object for dynamic routing with custom state
  - Sequence of `Send` objects for parallel execution

**Example - Basic State Update**:
```python
def my_node(state: State):
    # Update state and move to next node
    return Command(
        update={"result": "processed"},
        goto="next_node"
    )
```

**Example - Parallel Execution**:
```python
def router_node(state: State):
    # Send to multiple nodes in parallel with different inputs
    return Command(
        goto=[
            Send("process_a", {"data": state["items"][0]}),
            Send("process_b", {"data": state["items"][1]}),
        ]
    )
```

**Example - Resuming from Interrupt**:
```python
# After interruption, resume with user input
graph.stream(
    Command(resume="user provided value"),
    config
)
```

**Example - Parent Graph Communication**:
```python
def subgraph_node(state: State):
    # Send command to parent graph
    return Command(
        graph=Command.PARENT,
        update={"parent_state_key": "value"},
        goto="parent_node"
    )
```

#### `Send`

```python
class Send:
    __slots__ = ("node", "arg")

    node: str
    arg: Any
```

**Purpose**: A message or packet to send to a specific node with custom state.

**Key Use Case - Map-Reduce Pattern**:
The `Send` class enables dynamic parallel execution where the same node is invoked multiple times with different states, then results are aggregated.

**Attributes**:
- `node`: Name of the target node
- `arg`: State or message to send (can differ from the main graph state)

**Example - Map-Reduce Workflow**:
```python
from typing import Annotated
from langgraph.types import Send
from langgraph.graph import StateGraph, START, END
import operator


class OverallState(TypedDict):
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]


def continue_to_jokes(state: OverallState):
    # Map: Create a Send for each subject
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]


def generate_joke(state):
    # Process individual item
    return {"jokes": [f"Joke about {state['subject']}"]}


builder = StateGraph(OverallState)
builder.add_node("generate_joke", generate_joke)
builder.add_conditional_edges(START, continue_to_jokes)
builder.add_edge("generate_joke", END)
graph = builder.compile()

# Invoking with two subjects results in parallel joke generation
result = graph.invoke({"subjects": ["cats", "dogs"]})
# {'subjects': ['cats', 'dogs'], 'jokes': ['Joke about cats', 'Joke about dogs']}
```

#### `Interrupt`

```python
@final
@dataclass(init=False, slots=True)
class Interrupt:
    value: Any
    id: str
```

**Purpose**: Information about an interrupt that occurred in a node.

**Attributes**:
- `value`: The value associated with the interrupt (context, question, or data for the client)
- `id`: Unique identifier for the interrupt (used to resume specific interrupts)

**Creation**:
```python
# Typically created by the interrupt() function, not directly
interrupt = Interrupt(value="Please provide input", id="unique-id")

# Or created from a namespace
interrupt = Interrupt.from_ns(value="data", ns="node/path/id")
```

**Note**: In version 0.6.0+, use the `interrupt()` function instead of creating `Interrupt` objects directly.

#### `interrupt()` Function

```python
def interrupt(value: Any) -> Any:
    """Interrupt the graph with a resumable exception from within a node."""
```

**Purpose**: Pause graph execution and request input from the client (human-in-the-loop).

**How It Works**:
1. **First Call**: Raises `GraphInterrupt` exception, pausing execution
2. **Value Surfaced**: The provided value is sent to the client
3. **Resume**: Client provides resume value via `Command(resume=...)`
4. **Subsequent Calls**: Returns the resume value, allowing node to continue

**Key Characteristics**:
- **Must have checkpointer**: Interrupts require state persistence
- **Re-executes node**: Node runs from the beginning when resumed
- **Multiple interrupts**: Multiple `interrupt()` calls in a node are matched to resume values by order

**Example**:
```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command


class State(TypedDict):
    foo: str
    human_value: Optional[str]


def node(state: State):
    # First execution: raises GraphInterrupt
    # Subsequent execution: returns resume value
    answer = interrupt("What is your age?")
    print(f"> Received: {answer}")
    return {"human_value": answer}


builder = StateGraph(State)
builder.add_node("node", node)
builder.add_edge(START, "node")

checkpointer = InMemorySaver()  # Required!
graph = builder.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "abc-123"}}

# First stream: hits interrupt
for chunk in graph.stream({"foo": "abc"}, config):
    print(chunk)
# Output: {'__interrupt__': (Interrupt(value='What is your age?', id='...'),)}

# Resume with user input
for chunk in graph.stream(Command(resume="25"), config):
    print(chunk)
# Output: > Received: 25
#         {'node': {'human_value': '25'}}
```

### State Management Types

#### `StateSnapshot`

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

**Purpose**: Immutable snapshot of graph state at a specific point in time.

**Attributes**:
- `values`: Current values of all state channels
- `next`: Names of nodes to execute in the next step
- `config`: Configuration used to fetch this snapshot
- `metadata`: User-defined metadata associated with this checkpoint
- `created_at`: ISO timestamp of snapshot creation
- `parent_config`: Config for the parent snapshot (for time-travel)
- `tasks`: Tasks scheduled for execution (may include errors if previously attempted)
- `interrupts`: Pending interrupts awaiting resolution

**Usage**:
```python
# Get current state snapshot
snapshot = graph.get_state(config)

print(f"Current values: {snapshot.values}")
print(f"Next nodes: {snapshot.next}")
print(f"Pending interrupts: {snapshot.interrupts}")

# Time-travel to previous state
if snapshot.parent_config:
    previous = graph.get_state(snapshot.parent_config)
```

#### `StateUpdate`

```python
class StateUpdate(NamedTuple):
    values: dict[str, Any] | None
    as_node: str | None = None
    task_id: str | None = None
```

**Purpose**: Represents a state update operation, optionally as if from a specific node or task.

**Attributes**:
- `values`: The state updates to apply
- `as_node`: Apply update as if it came from this node
- `task_id`: Associate update with a specific task

**Usage**:
```python
# External state update (e.g., from API)
graph.update_state(
    config,
    StateUpdate(
        values={"user_feedback": "looks good"},
        as_node="human_reviewer"
    )
)
```

#### `Overwrite`

```python
@dataclass(slots=True)
class Overwrite:
    value: Any
```

**Purpose**: Bypass a reducer and write a value directly to a channel (typically for `Annotated` fields with reducers).

**Key Points**:
- Normally, annotated fields with reducers (like `operator.add`) combine new values with existing ones
- `Overwrite` replaces the entire value, ignoring the reducer
- Receiving multiple `Overwrite` values for the same channel in one step raises `InvalidUpdateError`

**Example**:
```python
from typing import Annotated
import operator
from langgraph.types import Overwrite


class State(TypedDict):
    messages: Annotated[list, operator.add]


def node_a(state: State):
    # Normal: adds to existing messages
    return {"messages": ["a"]}


def node_b(state: State):
    # Overwrite: replaces entire messages list
    return {"messages": Overwrite(value=["b"])}


graph = StateGraph(State)
graph.add_node("node_a", node_a)
graph.add_node("node_b", node_b)
graph.add_edge(START, "node_a")
graph.add_edge("node_a", "node_b")

result = graph.compile().invoke({"messages": ["START"]})
# Without Overwrite: {"messages": ["START", "a", "b"]}
# With Overwrite: {"messages": ["b"]}
```

### Task Types

#### `PregelTask`

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

**Purpose**: Represents a single executable task in the Pregel computation model.

**Attributes**:
- `id`: Unique identifier for the task
- `name`: Name of the node being executed
- `path`: Hierarchical path in the execution tree (for subgraphs)
- `error`: Exception if the task failed
- `interrupts`: Any interrupts raised during task execution
- `state`: Task's local state (for subgraphs or `Send` operations)
- `result`: Result of task execution (if completed)

**Usage**: Primarily used internally and in state snapshots to track task execution.

#### `PregelExecutableTask`

```python
@dataclass(weakref_slot=True, slots=True, frozen=True)  # Python 3.11+
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

**Purpose**: Internal representation of a task ready for execution, with all necessary context.

**Note**: This is an internal type used by the LangGraph execution engine.

### Streaming Types

#### `StreamMode`

```python
StreamMode = Literal[
    "values", "updates", "checkpoints", "tasks", "debug", "messages", "custom"
]
```

**Purpose**: Controls how the `stream()` method emits outputs during graph execution.

**Modes**:

1. **`"values"`**: Emit complete state after each step
   - Includes state after interrupts
   - Functional API: Emitted once at end
   ```python
   for chunk in graph.stream(input, stream_mode="values"):
       print(chunk)
   # {'text': 'a'}, {'text': 'ab'}, ...
   ```

2. **`"updates"`**: Emit only node updates
   - Shows what each node changed
   - Multiple updates in same step emitted separately
   ```python
   for chunk in graph.stream(input, stream_mode="updates"):
       print(chunk)
   # {'node_a': {'text': 'a'}}, {'node_b': {'text': 'b'}}, ...
   ```

3. **`"custom"`**: Emit custom data from nodes
   - Requires using `StreamWriter` in nodes
   - Useful for progress updates, partial results
   ```python
   def node(state, writer: StreamWriter):
       writer({"progress": 50})
       return state

   for chunk in graph.stream(input, stream_mode="custom"):
       print(chunk)
   # {"progress": 50}, ...
   ```

4. **`"messages"`**: Emit LLM token-by-token streaming
   - Captures streaming from any LLM calls inside nodes
   - Includes metadata about the invocation
   ```python
   for chunk in graph.stream(input, stream_mode="messages"):
       print(chunk)
   # (AIMessageChunk, metadata), ...
   ```

5. **`"checkpoints"`**: Emit checkpoint events
   - Same format as `get_state()` returns
   - Useful for understanding state evolution
   ```python
   for chunk in graph.stream(input, stream_mode="checkpoints"):
       print(chunk.values)
   ```

6. **`"tasks"`**: Emit task lifecycle events
   - When tasks start and finish
   - Includes results and errors
   ```python
   for chunk in graph.stream(input, stream_mode="tasks"):
       print(chunk)
   # PregelTask(...), ...
   ```

7. **`"debug"`**: Emit both checkpoints and tasks
   - Combination of `"checkpoints"` and `"tasks"`
   - Maximum visibility for debugging

**Multiple Modes**: You can stream with multiple modes simultaneously:
```python
for chunk in graph.stream(input, stream_mode=["values", "updates"]):
    print(chunk)
```

#### `StreamWriter`

```python
StreamWriter = Callable[[Any], None]
```

**Purpose**: Callable injected into nodes for writing custom streaming data.

**Characteristics**:
- Automatically injected if node requests it as keyword argument
- No-op when not using `stream_mode="custom"`
- Accepts any serializable data

**Example**:
```python
def my_node(state: State, writer: StreamWriter):
    writer({"status": "Starting processing..."})

    # Do some work
    for i in range(10):
        process_item(i)
        writer({"progress": i + 1, "total": 10})

    writer({"status": "Complete!"})
    return {"result": "done"}


# Stream custom updates
for chunk in graph.stream(input, stream_mode="custom"):
    print(chunk)
# {"status": "Starting processing..."}
# {"progress": 1, "total": 10}
# {"progress": 2, "total": 10}
# ...
# {"status": "Complete!"}
```

### Configuration and Policy Types

#### `Checkpointer`

```python
Checkpointer = None | bool | BaseCheckpointSaver
```

**Purpose**: Controls checkpointing behavior for subgraphs.

**Values**:
- `None`: Inherit checkpointer from parent graph (default)
- `True`: Enable persistent checkpointing for this subgraph
- `False`: Disable checkpointing, even if parent has one
- `BaseCheckpointSaver` instance: Use specific checkpointer

**Example**:
```python
from langgraph.checkpoint.memory import InMemorySaver

# Main graph with checkpointer
main_graph = StateGraph(MainState).compile(
    checkpointer=InMemorySaver()
)

# Subgraph inherits checkpointer
subgraph_inherit = StateGraph(SubState).compile(checkpointer=None)

# Subgraph has own checkpointer
subgraph_own = StateGraph(SubState).compile(checkpointer=InMemorySaver())

# Subgraph without checkpointing
subgraph_none = StateGraph(SubState).compile(checkpointer=False)
```

#### `Durability`

```python
Durability = Literal["sync", "async", "exit"]
```

**Purpose**: Controls when state changes are persisted to the checkpointer.

**Modes**:
- **`"sync"`**: Changes persisted synchronously before next step starts
  - Maximum durability
  - Slight performance overhead
  - Guarantees consistency at each step

- **`"async"`**: Changes persisted asynchronously while next step executes
  - Better performance
  - Still durable across failures
  - Eventual consistency within same execution

- **`"exit"`**: Changes persisted only when graph exits
  - Best performance
  - No durability during execution
  - Only final state guaranteed

**Example**:
```python
# Maximum safety - persist at each step
graph = builder.compile(
    checkpointer=checkpointer,
    durability="sync"
)

# Balanced - async persistence
graph = builder.compile(
    checkpointer=checkpointer,
    durability="async"  # Default
)

# Performance - only persist at end
graph = builder.compile(
    checkpointer=checkpointer,
    durability="exit"
)
```

#### `RetryPolicy`

```python
class RetryPolicy(NamedTuple):
    initial_interval: float = 0.5
    backoff_factor: float = 2.0
    max_interval: float = 128.0
    max_attempts: int = 3
    jitter: bool = True
    retry_on: type[Exception] | Sequence[type[Exception]] | Callable[[Exception], bool] = default_retry_on
```

**Purpose**: Configuration for automatic retry of failed nodes.

**Attributes**:
- `initial_interval`: Time before first retry (seconds)
- `backoff_factor`: Multiplier for interval after each retry (exponential backoff)
- `max_interval`: Maximum time between retries (seconds)
- `max_attempts`: Total attempts including first execution
- `jitter`: Add randomness to intervals (prevents thundering herd)
- `retry_on`: Which exceptions trigger retry
  - Exception type or list of types
  - Callable that returns True for retryable exceptions

**Example**:
```python
from langgraph.types import RetryPolicy


def should_retry(exc: Exception) -> bool:
    # Retry on network errors, not on validation errors
    return isinstance(exc, (NetworkError, TimeoutError))


retry_policy = RetryPolicy(
    initial_interval=1.0,      # Wait 1s before first retry
    backoff_factor=2.0,        # Double wait time each retry
    max_interval=60.0,         # Max 60s between retries
    max_attempts=5,            # Try up to 5 times total
    jitter=True,               # Add randomness
    retry_on=should_retry      # Custom retry logic
)

graph = builder.compile(
    checkpointer=checkpointer,
    retry_policy=retry_policy
)
```

**Retry Sequence Example**:
```
Attempt 1: Fails immediately
Wait: 1.0s (± jitter)
Attempt 2: Fails
Wait: 2.0s (± jitter)
Attempt 3: Fails
Wait: 4.0s (± jitter)
Attempt 4: Fails
Wait: 8.0s (± jitter)
Attempt 5: Final attempt
```

#### `CachePolicy`

```python
@dataclass(kw_only=True, slots=True, frozen=True)
class CachePolicy(Generic[KeyFuncT]):
    key_func: KeyFuncT = default_cache_key
    ttl: int | None = None
```

**Purpose**: Configuration for caching node results.

**Attributes**:
- `key_func`: Function to generate cache key from node input
  - Default: Hash input with pickle
  - Custom: Define your own cache key logic
- `ttl`: Time-to-live for cache entries (seconds)
  - `None`: Never expires
  - Integer: Expire after N seconds

**Example**:
```python
from langgraph.types import CachePolicy


def my_cache_key(input_data):
    # Custom cache key based on specific fields
    return f"{input_data['user_id']}:{input_data['query']}"


cache_policy = CachePolicy(
    key_func=my_cache_key,
    ttl=3600  # Cache for 1 hour
)

graph = builder.compile(cache_policy=cache_policy)
```

#### `CacheKey`

```python
class CacheKey(NamedTuple):
    ns: tuple[str, ...]
    key: str
    ttl: int | None
```

**Purpose**: Internal representation of a cache key for a task.

**Attributes**:
- `ns`: Namespace tuple (identifies cache scope)
- `key`: The actual cache key string
- `ttl`: Time-to-live in seconds

### Special Type Literals

#### `All`

```python
All = Literal["*"]
```

**Purpose**: Special value to indicate that graph should interrupt on all nodes.

**Example**:
```python
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before="*"  # Interrupt before every node
)

# Or
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_after="*"  # Interrupt after every node
)
```

---

## Error Types

LangGraph defines several exception types to handle different error conditions during graph execution. Understanding these errors helps with debugging and building robust agent systems.

### `GraphRecursionError`

```python
class GraphRecursionError(RecursionError):
    """Raised when the graph has exhausted the maximum number of steps."""
```

**When Raised**:
- Graph has executed more steps than the configured `recursion_limit`
- Prevents infinite loops in cyclic graphs

**How to Fix**:
1. **Increase limit** if your workflow legitimately needs more steps:
   ```python
   graph.invoke(
       {"messages": [("user", "Hello")]},
       {"recursion_limit": 1000}  # Default is usually 25
   )
   ```

2. **Check for infinite loops** in your graph logic:
   - Conditional edges that never terminate
   - Missing base case in recursive patterns
   - Incorrect routing logic

**Example Error Scenario**:
```python
def should_continue(state):
    # Bug: Always returns True, creating infinite loop
    return True


graph.add_conditional_edges(
    "agent",
    should_continue,
    {True: "agent", False: END}
)

# Will raise GraphRecursionError after 25 steps
graph.invoke({"input": "test"})
```

**Documentation**: [GRAPH_RECURSION_LIMIT troubleshooting guide](https://docs.langchain.com/oss/python/langgraph/GRAPH_RECURSION_LIMIT)

### `InvalidUpdateError`

```python
class InvalidUpdateError(Exception):
    """Raised when attempting to update a channel with an invalid set of updates."""
```

**When Raised**:
1. **Concurrent conflicting updates**: Multiple nodes try to update the same channel differently in one step
2. **Invalid return value**: Node returns value that doesn't match state schema
3. **Multiple Overwrites**: Multiple `Overwrite` values for same channel in one step

**Common Scenarios**:

**Scenario 1 - Conflicting Parallel Updates**:
```python
def node_a(state):
    return {"counter": 1}

def node_b(state):
    return {"counter": 2}

# If both run in parallel and counter doesn't have a reducer
# InvalidUpdateError: Conflicting updates for channel 'counter'
```

**Fix**: Use a reducer or ensure nodes don't conflict:
```python
from typing import Annotated
import operator

class State(TypedDict):
    counter: Annotated[int, operator.add]  # Reducer combines updates

# Now both updates are added: 1 + 2 = 3
```

**Scenario 2 - Invalid Return Value**:
```python
class State(TypedDict):
    count: int

def bad_node(state):
    return {"count": "not a number"}  # Wrong type!

# InvalidUpdateError: Invalid value for channel 'count'
```

**Troubleshooting Guides**:
- [INVALID_CONCURRENT_GRAPH_UPDATE](https://docs.langchain.com/oss/python/langgraph/INVALID_CONCURRENT_GRAPH_UPDATE)
- [INVALID_GRAPH_NODE_RETURN_VALUE](https://docs.langchain.com/oss/python/langgraph/INVALID_GRAPH_NODE_RETURN_VALUE)

### `GraphBubbleUp`

```python
class GraphBubbleUp(Exception):
    pass
```

**Purpose**: Internal base class for exceptions that should propagate up through subgraphs.

**Usage**: Not raised directly by user code. Base class for `GraphInterrupt` and `ParentCommand`.

### `GraphInterrupt`

```python
class GraphInterrupt(GraphBubbleUp):
    """Raised when a subgraph is interrupted, suppressed by the root graph.
    Never raised directly, or surfaced to the user."""

    def __init__(self, interrupts: Sequence[Interrupt] = ()) -> None:
        super().__init__(interrupts)
```

**Purpose**: Internal exception raised by `interrupt()` function.

**Key Points**:
- **Never raised directly by users**: Use `interrupt()` function instead
- **Suppressed at root graph**: Root graph catches and handles it
- **Not surfaced to user**: Converted to interrupt state in checkpoints
- **Contains interrupts**: Carries `Interrupt` objects with values and IDs

**How It Works**:
```python
# Inside a node
answer = interrupt("What is your name?")  # Raises GraphInterrupt internally

# LangGraph catches it, creates checkpoint, surfaces to client
# User sees interrupt in state snapshot, not exception
```

### `NodeInterrupt` (Deprecated)

```python
@deprecated("NodeInterrupt is deprecated. Use `interrupt()` instead.")
class NodeInterrupt(GraphInterrupt):
    """Raised by a node to interrupt execution."""
```

**Status**: Deprecated since version 0.10

**Migration**:
```python
# Old way (deprecated)
raise NodeInterrupt("Need user input")

# New way
from langgraph.types import interrupt

value = interrupt("Need user input")
```

### `ParentCommand`

```python
class ParentCommand(GraphBubbleUp):
    args: tuple[Command]

    def __init__(self, command: Command) -> None:
        super().__init__(command)
```

**Purpose**: Exception raised when a subgraph issues a `Command` targeting its parent graph.

**When Raised**:
- Subgraph node returns `Command(graph=Command.PARENT, ...)`
- Allows subgraphs to communicate with and control parent execution

**Example**:
```python
def subgraph_node(state):
    # This raises ParentCommand internally
    return Command(
        graph=Command.PARENT,
        update={"parent_field": "value"},
        goto="parent_node_name"
    )

# Parent graph catches ParentCommand and applies the command
```

### `EmptyInputError`

```python
class EmptyInputError(Exception):
    """Raised when graph receives an empty input."""
```

**When Raised**:
- Graph invoked with `None` or empty input when input is required
- Functional API receives empty input

**How to Fix**:
```python
# Bad
graph.invoke(None)  # EmptyInputError

# Good
graph.invoke({"messages": []})
```

### `TaskNotFound`

```python
class TaskNotFound(Exception):
    """Raised when the executor is unable to find a task (for distributed mode)."""
```

**Purpose**: Distributed execution error when a worker cannot locate a task.

**Context**: Used in advanced distributed execution scenarios where tasks are executed across multiple workers/processes.

### `EmptyChannelError`

```python
# Re-exported from langgraph.checkpoint.base
class EmptyChannelError(Exception):
    """Raised when attempting to read from an empty channel."""
```

**When Raised**:
- Attempting to access a channel that hasn't been initialized
- Reading state field before any node has written to it

**Common Scenario**:
```python
class State(TypedDict):
    optional_field: str  # Not initialized

def node(state):
    # If optional_field was never set, this might fail
    value = state["optional_field"]
```

**How to Fix**:
```python
# Option 1: Provide default in state
class State(TypedDict):
    optional_field: str = ""  # Default value

# Option 2: Check before accessing
def node(state):
    value = state.get("optional_field", "default")
```

### `ErrorCode` Enum

```python
class ErrorCode(Enum):
    GRAPH_RECURSION_LIMIT = "GRAPH_RECURSION_LIMIT"
    INVALID_CONCURRENT_GRAPH_UPDATE = "INVALID_CONCURRENT_GRAPH_UPDATE"
    INVALID_GRAPH_NODE_RETURN_VALUE = "INVALID_GRAPH_NODE_RETURN_VALUE"
    MULTIPLE_SUBGRAPHS = "MULTIPLE_SUBGRAPHS"
    INVALID_CHAT_HISTORY = "INVALID_CHAT_HISTORY"
```

**Purpose**: Enumeration of error codes for linking to troubleshooting documentation.

**Usage**: Internal helper for creating informative error messages with documentation links.

---

## Configuration System

LangGraph provides a configuration system that allows nodes to access runtime context, stores, and streaming capabilities.

### `get_config()`

```python
def get_config() -> RunnableConfig:
    """Get the current RunnableConfig from execution context."""
```

**Purpose**: Access the current execution configuration from within a node or task.

**Returns**: `RunnableConfig` with execution metadata and settings.

**Important Constraints**:
- **Must be called from runnable context**: Only works inside nodes/tasks during execution
- **Python 3.11+ for async**: Async usage requires Python 3.11 or later (uses `contextvars`)

**Example**:
```python
from langgraph.config import get_config


def my_node(state):
    config = get_config()

    # Access configuration
    thread_id = config["configurable"].get("thread_id")
    user_id = config["configurable"].get("user_id")

    print(f"Executing for thread {thread_id}, user {user_id}")
    return state
```

**Error Handling**:
```python
try:
    config = get_config()
except RuntimeError:
    # Called outside of runnable context
    print("Not inside a graph execution")
```

### `get_store()`

```python
def get_store() -> BaseStore:
    """Access LangGraph store from inside a graph node or entrypoint task."""
```

**Purpose**: Access the configured store for reading/writing persistent data across executions.

**Requirements**:
- Graph must be compiled with a store: `.compile(store=store)`
- Or functional API entrypoint must specify store: `@entrypoint(store=store)`

**Constraints**:
- **Python 3.11+ for async**: Async usage requires Python 3.11+ (uses `contextvars`)

**Example with StateGraph**:
```python
from langgraph.graph import StateGraph, START
from langgraph.store.memory import InMemoryStore
from langgraph.config import get_store


store = InMemoryStore()
store.put(("values",), "foo", {"bar": 2})


class State(TypedDict):
    foo: int


def my_node(state: State):
    my_store = get_store()

    # Read from store
    stored_value = my_store.get(("values",), "foo").value["bar"]

    # Write to store
    my_store.put(("results",), "result", {"value": stored_value + 1})

    return {"foo": stored_value + 1}


graph = (
    StateGraph(State)
    .add_node(my_node)
    .add_edge(START, "my_node")
    .compile(store=store)  # Store must be configured!
)

result = graph.invoke({"foo": 1})
# {"foo": 3}
```

**Example with Functional API**:
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


@entrypoint(store=store)  # Store must be configured!
def workflow(value: int):
    return my_task(value).result()


result = workflow.invoke(1)
# 3
```

**Store Operations**:
```python
def my_node(state):
    store = get_store()

    # Get item
    item = store.get(namespace=("users",), key="user-123")

    # Put item
    store.put(
        namespace=("users",),
        key="user-123",
        value={"name": "Alice", "count": 5}
    )

    # Search items
    results = store.search(
        namespace=("users",),
        filter={"count": {"$gte": 5}}
    )

    # Delete item
    store.delete(namespace=("users",), key="user-123")

    return state
```

### `get_stream_writer()`

```python
def get_stream_writer() -> StreamWriter:
    """Access StreamWriter from inside a graph node or entrypoint task."""
```

**Purpose**: Get the stream writer for emitting custom streaming data.

**Requirements**:
- Use with `stream_mode="custom"` to see output
- Works in any node or task

**Constraints**:
- **Python 3.11+ for async**: Async usage requires Python 3.11+ (uses `contextvars`)

**Example with StateGraph**:
```python
from langgraph.graph import StateGraph, START
from langgraph.config import get_stream_writer


class State(TypedDict):
    foo: int


def my_node(state: State):
    writer = get_stream_writer()

    # Emit custom streaming data
    writer({"status": "processing", "progress": 0})

    # Do work
    for i in range(5):
        process_step(i)
        writer({"status": "processing", "progress": (i + 1) * 20})

    writer({"status": "complete", "progress": 100})

    return {"foo": state["foo"] + 1}


graph = (
    StateGraph(State)
    .add_node(my_node)
    .add_edge(START, "my_node")
    .compile()
)

# Stream with custom mode
for chunk in graph.stream({"foo": 1}, stream_mode="custom"):
    print(chunk)

# Output:
# {"status": "processing", "progress": 0}
# {"status": "processing", "progress": 20}
# {"status": "processing", "progress": 40}
# {"status": "processing", "progress": 60}
# {"status": "processing", "progress": 80}
# {"status": "processing", "progress": 100}
# {"status": "complete", "progress": 100}
```

**Example with Functional API**:
```python
from langgraph.func import entrypoint, task
from langgraph.config import get_stream_writer


@task
def my_task(value: int):
    writer = get_stream_writer()
    writer({"custom_data": "Hello from task!"})
    return value + 1


@entrypoint()
def workflow(value: int):
    return my_task(value).result()


for chunk in workflow.stream(1, stream_mode="custom"):
    print(chunk)

# Output:
# {"custom_data": "Hello from task!"}
```

**Alternative - Injected Parameter**:

Instead of calling `get_stream_writer()`, you can also request it as a keyword argument:

```python
def my_node(state: State, writer: StreamWriter):
    # writer is automatically injected
    writer({"custom": "data"})
    return state
```

Both approaches work identically; use whichever fits your coding style.

---

## Summary

This document covered the foundational architecture of LangGraph:

1. **Overview**: LangGraph is a low-level framework for stateful, long-running agent workflows
2. **Core Philosophy**: Based on Google's Pregel model, adapted for agent workflows with cycles, state, and durability
3. **Key Constants**: `START`, `END` for graph boundaries; `TAG_NOSTREAM`, `TAG_HIDDEN` for control
4. **Type System**: Rich types for execution control (`Command`, `Send`, `interrupt`), state management (`StateSnapshot`, `Overwrite`), streaming (`StreamMode`, `StreamWriter`), and configuration (`RetryPolicy`, `CachePolicy`)
5. **Error Types**: Comprehensive error handling with `GraphRecursionError`, `InvalidUpdateError`, and others
6. **Configuration**: Runtime access to config, stores, and streaming via `get_config()`, `get_store()`, `get_stream_writer()`

Understanding these core concepts provides the foundation for building sophisticated, production-ready agent systems with LangGraph.
