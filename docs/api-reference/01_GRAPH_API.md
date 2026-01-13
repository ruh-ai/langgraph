# LangGraph Graph Building API Reference

This document provides comprehensive API documentation for the LangGraph Graph Building module, covering all classes, methods, and functions for building stateful multi-agent workflows.

## Table of Contents

1. [Core Classes](#core-classes)
   - [StateGraph](#stategraph)
   - [CompiledStateGraph](#compiledstategraph)
2. [State Management](#state-management)
   - [MessagesState](#messagesstate)
   - [add_messages](#add_messages)
   - [MessageGraph (Deprecated)](#messagegraph-deprecated)
3. [Node Types and Protocols](#node-types-and-protocols)
   - [StateNode](#statenode)
   - [StateNodeSpec](#statenodespec)
4. [Branching and Routing](#branching-and-routing)
   - [BranchSpec](#branchspec)
   - [Send](#send)
   - [Command](#command)
5. [Constants](#constants)
   - [START](#start)
   - [END](#end)
6. [Types and Helpers](#types-and-helpers)
   - [StateSnapshot](#statesnapshot)

---

## Core Classes

### StateGraph

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:112`

A graph whose nodes communicate by reading and writing to a shared state. The signature of each node is `State -> Partial<State>`.

Each state key can optionally be annotated with a reducer function that will be used to aggregate the values of that key received from multiple nodes. The signature of a reducer function is `(Value, Value) -> Value`.

**Signature:**
```python
class StateGraph(Generic[StateT, ContextT, InputT, OutputT]):
    def __init__(
        self,
        state_schema: type[StateT],
        context_schema: type[ContextT] | None = None,
        *,
        input_schema: type[InputT] | None = None,
        output_schema: type[OutputT] | None = None,
    ) -> None:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| state_schema | type[StateT] | Yes | - | The schema class that defines the state. Can be a TypedDict, Pydantic model, or dataclass. |
| context_schema | type[ContextT] \| None | No | None | The schema class that defines the runtime context. Use this to expose immutable context data to your nodes, like `user_id`, `db_conn`, etc. |
| input_schema | type[InputT] \| None | No | None | The schema class that defines the input to the graph. If not provided, defaults to `state_schema`. |
| output_schema | type[OutputT] \| None | No | None | The schema class that defines the output from the graph. If not provided, defaults to `state_schema`. |

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| nodes | dict[str, StateNodeSpec[Any, ContextT]] | Dictionary mapping node names to their specifications |
| edges | set[tuple[str, str]] | Set of directed edges in the graph |
| branches | defaultdict[str, dict[str, BranchSpec]] | Conditional branches from each node |
| channels | dict[str, BaseChannel] | State channels for communication |
| managed | dict[str, ManagedValueSpec] | Managed value specifications |
| compiled | bool | Whether the graph has been compiled |
| state_schema | type[StateT] | The state schema |
| context_schema | type[ContextT] \| None | The context schema |
| input_schema | type[InputT] | The input schema |
| output_schema | type[OutputT] | The output schema |

**Raises:**

| Exception | When |
|-----------|------|
| ValueError | When invalid schemas are provided or when managed channels are detected in input/output schemas |

**Example:**
```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph


def reducer(a: list, b: int | None) -> list:
    if b is not None:
        return a + [b]
    return a


class State(TypedDict):
    x: Annotated[list, reducer]


class Context(TypedDict):
    r: float


graph = StateGraph(state_schema=State, context_schema=Context)


def node(state: State, runtime: Runtime[Context]) -> dict:
    r = runtime.context.get("r", 1.0)
    x = state["x"][-1]
    next_value = x * r * (1 - x)
    return {"x": next_value}


graph.add_node("A", node)
graph.set_entry_point("A")
graph.set_finish_point("A")
compiled = graph.compile()

step1 = compiled.invoke({"x": 0.5}, context={"r": 3.0})
# {'x': [0.5, 0.75]}
```

---

### StateGraph.add_node

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:359`

Add a new node to the StateGraph.

**Signature:**
```python
def add_node(
    self,
    node: str | StateNode[NodeInputT, ContextT],
    action: StateNode[NodeInputT, ContextT] | None = None,
    *,
    defer: bool = False,
    metadata: dict[str, Any] | None = None,
    input_schema: type[NodeInputT] | None = None,
    retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy | None = None,
    destinations: dict[str, str] | tuple[str, ...] | None = None,
) -> Self:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| node | str \| StateNode[NodeInputT, ContextT] | Yes | - | The function or runnable this node will run. If a string is provided, it will be used as the node name, and action will be used as the function or runnable. |
| action | StateNode[NodeInputT, ContextT] \| None | No | None | The action associated with the node. Will be used as the node function or runnable if `node` is a string (node name). |
| defer | bool | No | False | Whether to defer the execution of the node until the run is about to end. |
| metadata | dict[str, Any] \| None | No | None | The metadata associated with the node. |
| input_schema | type[NodeInputT] \| None | No | None | The input schema for the node. Defaults to the graph's state schema. |
| retry_policy | RetryPolicy \| Sequence[RetryPolicy] \| None | No | None | The retry policy for the node. If a sequence is provided, the first matching policy will be applied. |
| cache_policy | CachePolicy \| None | No | None | The cache policy for the node. |
| destinations | dict[str, str] \| tuple[str, ...] \| None | No | None | Destinations that indicate where a node can route to. Useful for edgeless graphs with nodes that return Command objects. If a dict is provided, the keys will be used as the target node names and the values will be used as the labels for the edges. If a tuple is provided, the values will be used as the target node names. This is only used for graph rendering. |

**Returns:**

| Type | Description |
|------|-------------|
| Self | The instance of the StateGraph, allowing for method chaining. |

**Raises:**

| Exception | When |
|-----------|------|
| ValueError | If the node name already exists, is reserved (END or START), or contains reserved characters |
| RuntimeError | If action is None when required |

**Example:**
```python
from typing_extensions import TypedDict
from langchain_core.runnables import RunnableConfig
from langgraph.graph import START, StateGraph


class State(TypedDict):
    x: int


def my_node(state: State, config: RunnableConfig) -> State:
    return {"x": state["x"] + 1}


builder = StateGraph(State)
builder.add_node(my_node)  # node name will be 'my_node'
builder.add_edge(START, "my_node")
graph = builder.compile()
graph.invoke({"x": 1})
# {'x': 2}
```

**Example: Customize the name:**
```python
builder = StateGraph(State)
builder.add_node("my_fair_node", my_node)
builder.add_edge(START, "my_fair_node")
graph = builder.compile()
graph.invoke({"x": 1})
# {'x': 2}
```

---

### StateGraph.add_edge

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:574`

Add a directed edge from the start node (or list of start nodes) to the end node.

When a single start node is provided, the graph will wait for that node to complete before executing the end node. When multiple start nodes are provided, the graph will wait for ALL of the start nodes to complete before executing the end node.

**Signature:**
```python
def add_edge(
    self,
    start_key: str | list[str],
    end_key: str
) -> Self:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| start_key | str \| list[str] | Yes | - | The key(s) of the start node(s) of the edge. |
| end_key | str | Yes | - | The key of the end node of the edge. |

**Returns:**

| Type | Description |
|------|-------------|
| Self | The instance of the StateGraph, allowing for method chaining. |

**Raises:**

| Exception | When |
|-----------|------|
| ValueError | If the start key is 'END' or if START is the end key, or if the nodes don't exist |

**Example:**
```python
from langgraph.graph import StateGraph, START, END

builder = StateGraph(State)
builder.add_node("node_a", node_a_fn)
builder.add_node("node_b", node_b_fn)
builder.add_edge(START, "node_a")
builder.add_edge("node_a", "node_b")
builder.add_edge("node_b", END)
```

**Example: Wait for multiple nodes:**
```python
# node_c will only execute after both node_a AND node_b complete
builder.add_edge(["node_a", "node_b"], "node_c")
```

---

### StateGraph.add_conditional_edges

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:628`

Add a conditional edge from the starting node to any number of destination nodes.

**Signature:**
```python
def add_conditional_edges(
    self,
    source: str,
    path: Callable[..., Hashable | Sequence[Hashable]]
        | Callable[..., Awaitable[Hashable | Sequence[Hashable]]]
        | Runnable[Any, Hashable | Sequence[Hashable]],
    path_map: dict[Hashable, str] | list[str] | None = None,
) -> Self:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| source | str | Yes | - | The starting node. This conditional edge will run when exiting this node. |
| path | Callable \| Runnable | Yes | - | The callable that determines the next node or nodes. If not specifying `path_map` it should return one or more nodes. If it returns 'END', the graph will stop execution. |
| path_map | dict[Hashable, str] \| list[str] \| None | No | None | Optional mapping of paths to node names. If omitted the paths returned by `path` should be node names. |

**Returns:**

| Type | Description |
|------|-------------|
| Self | The instance of the graph, allowing for method chaining. |

**Raises:**

| Exception | When |
|-----------|------|
| ValueError | If a branch with the same name already exists for the source node |

**Warning:**
Without type hints on the `path` function's return value (e.g., `-> Literal["foo", "__end__"]:`) or a path_map, the graph visualization assumes the edge could transition to any node in the graph.

**Example:**
```python
from typing import Literal

def route_fn(state: State) -> Literal["node_a", "node_b"]:
    if state["x"] > 10:
        return "node_a"
    else:
        return "node_b"

builder.add_conditional_edges("start_node", route_fn)
```

**Example: With path_map:**
```python
def route_fn(state: State) -> str:
    if state["x"] > 10:
        return "high"
    else:
        return "low"

builder.add_conditional_edges(
    "start_node",
    route_fn,
    {"high": "node_a", "low": "node_b"}
)
```

---

### StateGraph.add_sequence

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:678`

Add a sequence of nodes that will be executed in the provided order.

**Signature:**
```python
def add_sequence(
    self,
    nodes: Sequence[
        StateNode[NodeInputT, ContextT]
        | tuple[str, StateNode[NodeInputT, ContextT]]
    ],
) -> Self:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| nodes | Sequence | Yes | - | A sequence of StateNode (callables that accept a state arg) or (name, StateNode) tuples. If no names are provided, the name will be inferred from the node object. Each node will be executed in the order provided. |

**Returns:**

| Type | Description |
|------|-------------|
| Self | The instance of the StateGraph, allowing for method chaining. |

**Raises:**

| Exception | When |
|-----------|------|
| ValueError | If the sequence is empty or contains duplicate node names |

**Example:**
```python
def step1(state: State) -> State:
    return {"x": state["x"] + 1}

def step2(state: State) -> State:
    return {"x": state["x"] * 2}

def step3(state: State) -> State:
    return {"x": state["x"] - 5}

builder = StateGraph(State)
builder.add_sequence([step1, step2, step3])
# Equivalent to:
# builder.add_node("step1", step1)
# builder.add_node("step2", step2)
# builder.add_node("step3", step3)
# builder.add_edge("step1", "step2")
# builder.add_edge("step2", "step3")
```

---

### StateGraph.set_entry_point

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:725`

Specifies the first node to be called in the graph. Equivalent to calling `add_edge(START, key)`.

**Signature:**
```python
def set_entry_point(self, key: str) -> Self:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| key | str | Yes | - | The key of the node to set as the entry point. |

**Returns:**

| Type | Description |
|------|-------------|
| Self | The instance of the graph, allowing for method chaining. |

**Example:**
```python
builder = StateGraph(State)
builder.add_node("first_node", first_node_fn)
builder.set_entry_point("first_node")
```

---

### StateGraph.set_conditional_entry_point

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:738`

Sets a conditional entry point in the graph.

**Signature:**
```python
def set_conditional_entry_point(
    self,
    path: Callable[..., Hashable | Sequence[Hashable]]
        | Callable[..., Awaitable[Hashable | Sequence[Hashable]]]
        | Runnable[Any, Hashable | Sequence[Hashable]],
    path_map: dict[Hashable, str] | list[str] | None = None,
) -> Self:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| path | Callable \| Runnable | Yes | - | The callable that determines the next node or nodes. If not specifying `path_map` it should return one or more nodes. If it returns END, the graph will stop execution. |
| path_map | dict[Hashable, str] \| list[str] \| None | No | None | Optional mapping of paths to node names. If omitted the paths returned by `path` should be node names. |

**Returns:**

| Type | Description |
|------|-------------|
| Self | The instance of the graph, allowing for method chaining. |

**Example:**
```python
def entry_router(state: State) -> str:
    if state["mode"] == "debug":
        return "debug_node"
    else:
        return "normal_node"

builder.set_conditional_entry_point(entry_router)
```

---

### StateGraph.set_finish_point

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:762`

Marks a node as a finish point of the graph. If the graph reaches this node, it will cease execution.

**Signature:**
```python
def set_finish_point(self, key: str) -> Self:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| key | str | Yes | - | The key of the node to set as the finish point. |

**Returns:**

| Type | Description |
|------|-------------|
| Self | The instance of the graph, allowing for method chaining. |

**Example:**
```python
builder = StateGraph(State)
builder.add_node("final_node", final_node_fn)
builder.set_finish_point("final_node")
# Equivalent to: builder.add_edge("final_node", END)
```

---

### StateGraph.compile

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:824`

Compiles the StateGraph into a CompiledStateGraph object. The compiled graph implements the Runnable interface and can be invoked, streamed, batched, and run asynchronously.

**Signature:**
```python
def compile(
    self,
    checkpointer: Checkpointer = None,
    *,
    cache: BaseCache | None = None,
    store: BaseStore | None = None,
    interrupt_before: All | list[str] | None = None,
    interrupt_after: All | list[str] | None = None,
    debug: bool = False,
    name: str | None = None,
) -> CompiledStateGraph[StateT, ContextT, InputT, OutputT]:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| checkpointer | Checkpointer | No | None | A checkpoint saver object or flag. If provided, this Checkpointer serves as a fully versioned "short-term memory" for the graph, allowing it to be paused, resumed, and replayed from any point. If None, it may inherit the parent graph's checkpointer when used as a subgraph. If False, it will not use or inherit any checkpointer. |
| cache | BaseCache \| None | No | None | Cache to use for storing node results. |
| store | BaseStore \| None | No | None | Memory store to use for SharedValues. |
| interrupt_before | All \| list[str] \| None | No | None | An optional list of node names to interrupt before. Use "*" for all nodes. |
| interrupt_after | All \| list[str] \| None | No | None | An optional list of node names to interrupt after. Use "*" for all nodes. |
| debug | bool | No | False | A flag indicating whether to enable debug mode. |
| name | str \| None | No | None | The name to use for the compiled graph. |

**Returns:**

| Type | Description |
|------|-------------|
| CompiledStateGraph | The compiled StateGraph. |

**Example:**
```python
from langgraph.checkpoint.memory import InMemorySaver

builder = StateGraph(State)
builder.add_node("node1", node1_fn)
builder.set_entry_point("node1")
builder.set_finish_point("node1")

# Compile without checkpointing
graph = builder.compile()

# Compile with checkpointing
checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# Compile with interrupts
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["node1"],
    interrupt_after=["node2"]
)
```

---

## CompiledStateGraph

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:932`

The compiled version of a StateGraph. This class extends Pregel and provides the execution runtime for LangGraph workflows. It implements the Runnable interface from LangChain.

**Signature:**
```python
class CompiledStateGraph(
    Pregel[StateT, ContextT, InputT, OutputT],
    Generic[StateT, ContextT, InputT, OutputT],
):
```

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| builder | StateGraph[StateT, ContextT, InputT, OutputT] | Reference to the original StateGraph builder |
| schema_to_mapper | dict[type[Any], Callable[[Any], Any] \| None] | Mapping of schemas to their mapper functions |

---

### CompiledStateGraph.invoke

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:3024`

Run the graph with a single input and config.

**Signature:**
```python
def invoke(
    self,
    input: InputT | Command | None,
    config: RunnableConfig | None = None,
    *,
    context: ContextT | None = None,
    stream_mode: StreamMode = "values",
    print_mode: StreamMode | Sequence[StreamMode] = (),
    output_keys: str | Sequence[str] | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    durability: Durability | None = None,
    **kwargs: Any,
) -> dict[str, Any] | Any:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | The input data for the graph. It can be a dictionary or any other type. |
| config | RunnableConfig \| None | No | None | The configuration for the graph run. Should include thread_id for checkpointing. |
| context | ContextT \| None | No | None | The static context to use for the run. Added in version 0.6.0. |
| stream_mode | StreamMode | No | "values" | The stream mode for the graph run. |
| print_mode | StreamMode \| Sequence[StreamMode] | No | () | Accepts the same values as stream_mode, but only prints the output to the console, for debugging purposes. Does not affect the output of the graph in any way. |
| output_keys | str \| Sequence[str] \| None | No | None | The output keys to retrieve from the graph run. |
| interrupt_before | All \| Sequence[str] \| None | No | None | The nodes to interrupt the graph run before. |
| interrupt_after | All \| Sequence[str] \| None | No | None | The nodes to interrupt the graph run after. |
| durability | Durability \| None | No | None | The durability mode for the graph execution. Options: "sync" (changes persisted synchronously before next step), "async" (changes persisted asynchronously while next step executes), "exit" (changes persisted only when graph exits). |

**Returns:**

| Type | Description |
|------|-------------|
| dict[str, Any] \| Any | The output of the graph run. If stream_mode is "values", it returns the latest output. If stream_mode is not "values", it returns a list of output chunks. |

**Example:**
```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)

# Simple invocation
result = graph.invoke({"x": 1})

# With config
result = graph.invoke(
    {"x": 1},
    config={"configurable": {"thread_id": "thread-1"}}
)

# With context
result = graph.invoke(
    {"x": 1},
    context={"user_id": "123"},
    config={"configurable": {"thread_id": "thread-1"}}
)
```

---

### CompiledStateGraph.stream

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:2407`

Stream graph steps for a single input.

**Signature:**
```python
def stream(
    self,
    input: InputT | Command | None,
    config: RunnableConfig | None = None,
    *,
    context: ContextT | None = None,
    stream_mode: StreamMode | Sequence[StreamMode] | None = None,
    print_mode: StreamMode | Sequence[StreamMode] = (),
    output_keys: str | Sequence[str] | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    durability: Durability | None = None,
    subgraphs: bool = False,
    debug: bool | None = None,
) -> Iterator[dict[str, Any] | Any]:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | The input to the graph. |
| config | RunnableConfig \| None | No | None | The configuration to use for the run. |
| context | ContextT \| None | No | None | The static context to use for the run. Added in version 0.6.0. |
| stream_mode | StreamMode \| Sequence[StreamMode] \| None | No | None | The mode to stream output, defaults to self.stream_mode. Options: "values" (emit all values in state after each step), "updates" (emit only node/task names and updates), "custom" (emit custom data using StreamWriter), "messages" (emit LLM messages token-by-token), "checkpoints" (emit checkpoint events), "tasks" (emit task start/finish events), "debug" (emit debug events). Can pass a list to stream multiple modes; outputs will be tuples of (mode, data). |
| print_mode | StreamMode \| Sequence[StreamMode] | No | () | Same values as stream_mode, but only prints to console for debugging. |
| output_keys | str \| Sequence[str] \| None | No | None | The keys to stream, defaults to all non-context channels. |
| interrupt_before | All \| Sequence[str] \| None | No | None | Nodes to interrupt before, defaults to all nodes in the graph. |
| interrupt_after | All \| Sequence[str] \| None | No | None | Nodes to interrupt after, defaults to all nodes in the graph. |
| durability | Durability \| None | No | None | The durability mode for the graph execution. Options: "sync", "async" (default), "exit". |
| subgraphs | bool | No | False | Whether to stream events from inside subgraphs. If True, events emitted as tuples (namespace, data) or (namespace, mode, data). |
| debug | bool \| None | No | None | Whether to enable debug mode. |

**Yields:**

| Type | Description |
|------|-------------|
| dict[str, Any] \| Any | The output of each step in the graph. The output shape depends on the stream_mode. |

**Example:**
```python
# Stream values mode
for chunk in graph.stream({"x": 1}, config={"configurable": {"thread_id": "1"}}):
    print(chunk)

# Stream updates mode
for chunk in graph.stream(
    {"x": 1},
    stream_mode="updates",
    config={"configurable": {"thread_id": "1"}}
):
    print(chunk)

# Stream multiple modes
for chunk in graph.stream(
    {"x": 1},
    stream_mode=["values", "updates"],
    config={"configurable": {"thread_id": "1"}}
):
    mode, data = chunk
    print(f"{mode}: {data}")
```

---

### CompiledStateGraph.get_state

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1235`

Get the current state of the graph.

**Signature:**
```python
def get_state(
    self,
    config: RunnableConfig,
    *,
    subgraphs: bool = False
) -> StateSnapshot:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The configuration specifying which thread/checkpoint to retrieve. Must include thread_id. |
| subgraphs | bool | No | False | Whether to include subgraph states in the snapshot. |

**Returns:**

| Type | Description |
|------|-------------|
| StateSnapshot | A snapshot of the current state including values, next nodes, config, metadata, created_at, parent_config, tasks, and interrupts. |

**Raises:**

| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set or if the subgraph is not found. |

**Example:**
```python
# Get current state
state = graph.get_state({"configurable": {"thread_id": "thread-1"}})
print(state.values)
print(state.next)

# Get state with subgraphs
state = graph.get_state(
    {"configurable": {"thread_id": "thread-1"}},
    subgraphs=True
)
```

---

### CompiledStateGraph.get_state_history

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1319`

Get the history of the state of the graph.

**Signature:**
```python
def get_state_history(
    self,
    config: RunnableConfig,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[StateSnapshot]:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The configuration specifying which thread to retrieve history for. |
| filter | dict[str, Any] \| None | No | None | Filter criteria for the history. |
| before | RunnableConfig \| None | No | None | Only return states before this checkpoint. |
| limit | int \| None | No | None | Maximum number of states to return. |

**Yields:**

| Type | Description |
|------|-------------|
| StateSnapshot | State snapshots in reverse chronological order (newest first). |

**Raises:**

| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set or if the subgraph is not found. |

**Example:**
```python
# Get last 10 states
for state in graph.get_state_history(
    {"configurable": {"thread_id": "thread-1"}},
    limit=10
):
    print(f"Step {state.metadata['step']}: {state.values}")
```

---

### CompiledStateGraph.update_state

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:2309`

Update the state of the graph with the given values, as if they came from node `as_node`. If `as_node` is not provided, it will be set to the last node that updated the state, if not ambiguous.

**Signature:**
```python
def update_state(
    self,
    config: RunnableConfig,
    values: dict[str, Any] | Any | None,
    as_node: str | None = None,
    task_id: str | None = None,
) -> RunnableConfig:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The configuration specifying which thread to update. |
| values | dict[str, Any] \| Any \| None | Yes | - | The values to update the state with. Can be a partial state update. |
| as_node | str \| None | No | None | The node to attribute the update to. If None, will be inferred from the last node that updated state. |
| task_id | str \| None | No | None | Optional task ID to associate with this update. |

**Returns:**

| Type | Description |
|------|-------------|
| RunnableConfig | The updated config pointing to the new checkpoint. |

**Raises:**

| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set |
| InvalidUpdateError | If the update is invalid (e.g., ambiguous as_node, node doesn't exist) |

**Example:**
```python
# Update state as if it came from a specific node
config = {"configurable": {"thread_id": "thread-1"}}
new_config = graph.update_state(
    config,
    {"x": 100},
    as_node="node1"
)

# Update state (as_node will be inferred)
new_config = graph.update_state(
    config,
    {"x": 200}
)
```

---

### CompiledStateGraph.get_graph

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:704`

Return a drawable representation of the computation graph.

**Signature:**
```python
def get_graph(
    self,
    config: RunnableConfig | None = None,
    *,
    xray: int | bool = False
) -> Graph:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | Optional configuration. |
| xray | int \| bool | No | False | If True or a positive integer, include subgraphs in the visualization. Integer value controls recursion depth. |

**Returns:**

| Type | Description |
|------|-------------|
| Graph | A drawable Graph object that can be visualized. |

**Example:**
```python
# Get graph structure
graph_structure = graph.get_graph()

# Visualize as ASCII
print(graph_structure.draw_ascii())

# Get graph with subgraphs
graph_structure = graph.get_graph(xray=True)
```

---

### CompiledStateGraph.ainvoke

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:3114`

Asynchronously run the graph with a single input and config. Same parameters and behavior as `invoke()` but runs asynchronously.

**Signature:**
```python
async def ainvoke(
    self,
    input: InputT | Command | None,
    config: RunnableConfig | None = None,
    *,
    context: ContextT | None = None,
    stream_mode: StreamMode = "values",
    print_mode: StreamMode | Sequence[StreamMode] = (),
    output_keys: str | Sequence[str] | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    durability: Durability | None = None,
    **kwargs: Any,
) -> dict[str, Any] | Any:
```

**Example:**
```python
result = await graph.ainvoke(
    {"x": 1},
    config={"configurable": {"thread_id": "thread-1"}}
)
```

---

### CompiledStateGraph.astream

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py` (async version)

Asynchronously stream graph steps for a single input. Same parameters and behavior as `stream()` but runs asynchronously and returns an async iterator.

**Signature:**
```python
async def astream(
    self,
    input: InputT | Command | None,
    config: RunnableConfig | None = None,
    *,
    context: ContextT | None = None,
    stream_mode: StreamMode | Sequence[StreamMode] | None = None,
    print_mode: StreamMode | Sequence[StreamMode] = (),
    output_keys: str | Sequence[str] | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    durability: Durability | None = None,
    subgraphs: bool = False,
    debug: bool | None = None,
) -> AsyncIterator[dict[str, Any] | Any]:
```

**Example:**
```python
async for chunk in graph.astream(
    {"x": 1},
    config={"configurable": {"thread_id": "thread-1"}}
):
    print(chunk)
```

---

### CompiledStateGraph.aget_state

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1277`

Asynchronously get the current state of the graph. Same parameters and behavior as `get_state()` but runs asynchronously.

**Signature:**
```python
async def aget_state(
    self,
    config: RunnableConfig,
    *,
    subgraphs: bool = False
) -> StateSnapshot:
```

**Example:**
```python
state = await graph.aget_state(
    {"configurable": {"thread_id": "thread-1"}}
)
```

---

### CompiledStateGraph.aget_state_history

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1370`

Asynchronously get the history of the state of the graph. Same parameters and behavior as `get_state_history()` but runs asynchronously.

**Signature:**
```python
async def aget_state_history(
    self,
    config: RunnableConfig,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> AsyncIterator[StateSnapshot]:
```

**Example:**
```python
async for state in graph.aget_state_history(
    {"configurable": {"thread_id": "thread-1"}},
    limit=10
):
    print(state.values)
```

---

### CompiledStateGraph.aupdate_state

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:2322`

Asynchronously update the state of the graph. Same parameters and behavior as `update_state()` but runs asynchronously.

**Signature:**
```python
async def aupdate_state(
    self,
    config: RunnableConfig,
    values: dict[str, Any] | Any,
    as_node: str | None = None,
    task_id: str | None = None,
) -> RunnableConfig:
```

**Example:**
```python
new_config = await graph.aupdate_state(
    {"configurable": {"thread_id": "thread-1"}},
    {"x": 100},
    as_node="node1"
)
```

---

## State Management

### MessagesState

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/message.py:307`

A pre-built state schema for graphs that work with messages. This is a TypedDict with a single key `messages` that uses the `add_messages` reducer.

**Signature:**
```python
class MessagesState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

**Example:**
```python
from langgraph.graph import StateGraph
from langgraph.graph.message import MessagesState

builder = StateGraph(MessagesState)

def chatbot(state: MessagesState):
    return {"messages": [("assistant", "Hello!")]}

builder.add_node("chatbot", chatbot)
builder.set_entry_point("chatbot")
builder.set_finish_point("chatbot")
graph = builder.compile()

result = graph.invoke({"messages": [("user", "Hi there")]})
# {'messages': [HumanMessage(content='Hi there'), AIMessage(content='Hello!')]}
```

---

### add_messages

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/message.py:61`

Merges two lists of messages, updating existing messages by ID. By default, this ensures the state is "append-only", unless the new message has the same ID as an existing message.

**Signature:**
```python
def add_messages(
    left: Messages,
    right: Messages,
    *,
    format: Literal["langchain-openai"] | None = None,
) -> Messages:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| left | Messages | Yes | - | The base list of Messages. |
| right | Messages | Yes | - | The list of Messages (or single Message) to merge into the base list. |
| format | Literal["langchain-openai"] \| None | No | None | The format to return messages in. If None then Messages will be returned as is. If "langchain-openai" then Messages will be returned as BaseMessage objects with their contents formatted to match OpenAI message format. Requires langchain-core>=0.3.11. |

**Returns:**

| Type | Description |
|------|-------------|
| Messages | A new list of messages with the messages from `right` merged into `left`. If a message in `right` has the same ID as a message in `left`, the message from `right` will replace the message from `left`. |

**Raises:**

| Exception | When |
|-----------|------|
| ValueError | When attempting to delete a message with an ID that doesn't exist |

**Example: Basic usage:**
```python
from langchain_core.messages import AIMessage, HumanMessage

msgs1 = [HumanMessage(content="Hello", id="1")]
msgs2 = [AIMessage(content="Hi there!", id="2")]
add_messages(msgs1, msgs2)
# [HumanMessage(content='Hello', id='1'), AIMessage(content='Hi there!', id='2')]
```

**Example: Overwrite existing message:**
```python
msgs1 = [HumanMessage(content="Hello", id="1")]
msgs2 = [HumanMessage(content="Hello again", id="1")]
add_messages(msgs1, msgs2)
# [HumanMessage(content='Hello again', id='1')]
```

**Example: Use in a StateGraph:**
```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph


class State(TypedDict):
    messages: Annotated[list, add_messages]


builder = StateGraph(State)
builder.add_node("chatbot", lambda state: {"messages": [("assistant", "Hello")]})
builder.set_entry_point("chatbot")
builder.set_finish_point("chatbot")
graph = builder.compile()
graph.invoke({})
# {'messages': [AIMessage(content='Hello', id=...)]}
```

**Example: Use OpenAI message format:**
```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages(format="langchain-openai")]


def chatbot_node(state: State) -> list:
    return {
        "messages": [
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": "Here's an image:",
                        "cache_control": {"type": "ephemeral"},
                    },
                    {
                        "type": "image",
                        "source": {
                            "type": "base64",
                            "media_type": "image/jpeg",
                            "data": "1234",
                        },
                    },
                ],
            },
        ]
    }


builder = StateGraph(State)
builder.add_node("chatbot", chatbot_node)
builder.set_entry_point("chatbot")
builder.set_finish_point("chatbot")
graph = builder.compile()
graph.invoke({"messages": []})
# {
#     'messages': [
#         HumanMessage(
#             content=[
#                 {"type": "text", "text": "Here's an image:"},
#                 {
#                     "type": "image_url",
#                     "image_url": {"url": "data:image/jpeg;base64,1234"},
#                 },
#             ],
#         ),
#     ]
# }
```

---

### MessageGraph (Deprecated)

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/message.py:251`

**Deprecated:** MessageGraph is deprecated in langgraph 1.0.0, to be removed in 2.0.0. Please use StateGraph with a `messages` key instead.

A StateGraph where every node receives a list of messages as input and returns one or more messages as output. MessageGraph is a subclass of StateGraph whose entire state is a single, append-only list of messages.

**Signature:**
```python
@deprecated(
    "MessageGraph is deprecated in langgraph 1.0.0, to be removed in 2.0.0. "
    "Please use StateGraph with a `messages` key instead.",
    category=None,
)
class MessageGraph(StateGraph):
    def __init__(self) -> None:
```

**Example (use MessagesState instead):**
```python
# Old way (deprecated):
from langgraph.graph.message import MessageGraph
builder = MessageGraph()

# New way (recommended):
from langgraph.graph import StateGraph
from langgraph.graph.message import MessagesState

builder = StateGraph(MessagesState)
```

---

## Node Types and Protocols

### StateNode

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/_node.py:70`

A type alias representing the various node signatures supported by StateGraph. A StateNode is any callable that accepts state and optionally additional parameters.

**Signature:**
```python
StateNode: TypeAlias = (
    _Node[NodeInputT]
    | _NodeWithConfig[NodeInputT]
    | _NodeWithWriter[NodeInputT]
    | _NodeWithStore[NodeInputT]
    | _NodeWithWriterStore[NodeInputT]
    | _NodeWithConfigWriter[NodeInputT]
    | _NodeWithConfigStore[NodeInputT]
    | _NodeWithConfigWriterStore[NodeInputT]
    | _NodeWithRuntime[NodeInputT, ContextT]
    | Runnable[NodeInputT, Any]
)
```

**Supported Signatures:**

1. **Basic Node:**
```python
def node(state: State) -> Partial[State]:
    return {"key": "value"}
```

2. **Node with Config:**
```python
def node(state: State, config: RunnableConfig) -> Partial[State]:
    return {"key": "value"}
```

3. **Node with Runtime:**
```python
def node(state: State, *, runtime: Runtime[Context]) -> Partial[State]:
    context_value = runtime.context.get("key")
    return {"key": "value"}
```

4. **Node with Writer:**
```python
def node(state: State, *, writer: StreamWriter) -> Partial[State]:
    writer("custom event")
    return {"key": "value"}
```

5. **Node with Store:**
```python
def node(state: State, *, store: BaseStore) -> Partial[State]:
    data = store.get("namespace", "key")
    return {"key": "value"}
```

6. **Node with Multiple Parameters:**
```python
def node(state: State, config: RunnableConfig, *, writer: StreamWriter, store: BaseStore) -> Partial[State]:
    return {"key": "value"}
```

7. **Runnable:**
```python
from langchain_core.runnables import RunnableLambda

node = RunnableLambda(lambda state: {"key": "value"})
```

---

### StateNodeSpec

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/_node.py:84`

A dataclass that holds the specification for a node in the StateGraph.

**Signature:**
```python
@dataclass(slots=True)
class StateNodeSpec(Generic[NodeInputT, ContextT]):
    runnable: StateNode[NodeInputT, ContextT]
    metadata: dict[str, Any] | None
    input_schema: type[NodeInputT]
    retry_policy: RetryPolicy | Sequence[RetryPolicy] | None
    cache_policy: CachePolicy | None
    ends: tuple[str, ...] | dict[str, str] | None = EMPTY_SEQ
    defer: bool = False
```

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| runnable | StateNode[NodeInputT, ContextT] | The actual node function or runnable |
| metadata | dict[str, Any] \| None | Metadata associated with the node |
| input_schema | type[NodeInputT] | The input schema for the node |
| retry_policy | RetryPolicy \| Sequence[RetryPolicy] \| None | Retry policies for the node |
| cache_policy | CachePolicy \| None | Cache policy for the node |
| ends | tuple[str, ...] \| dict[str, str] \| None | Possible end destinations for the node |
| defer | bool | Whether to defer execution until the run is about to end |

---

## Branching and Routing

### BranchSpec

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/_branch.py:83`

A named tuple that specifies a conditional branch in the graph.

**Signature:**
```python
class BranchSpec(NamedTuple):
    path: Runnable[Any, Hashable | list[Hashable]]
    ends: dict[Hashable, str] | None
    input_schema: type[Any] | None = None
```

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| path | Runnable[Any, Hashable \| list[Hashable]] | The runnable that determines which branch to take |
| ends | dict[Hashable, str] \| None | Mapping from branch return values to node names |
| input_schema | type[Any] \| None | Optional input schema for the branch condition |

**Methods:**

#### BranchSpec.from_path

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/_branch.py:88`

Create a BranchSpec from a path function and optional path map.

**Signature:**
```python
@classmethod
def from_path(
    cls,
    path: Runnable[Any, Hashable | list[Hashable]],
    path_map: dict[Hashable, str] | list[str] | None,
    infer_schema: bool = False,
) -> BranchSpec:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| path | Runnable | Yes | - | The routing function |
| path_map | dict \| list \| None | Yes | - | Mapping from route values to node names |
| infer_schema | bool | No | False | Whether to infer the input schema from type hints |

**Returns:**

| Type | Description |
|------|-------------|
| BranchSpec | A new BranchSpec instance |

---

### Send

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:285`

A message or packet to send to a specific node in the graph. The Send class is used within a StateGraph's conditional edges to dynamically invoke a node with a custom state at the next step.

Importantly, the sent state can differ from the core graph's state, allowing for flexible and dynamic workflow management (e.g., map-reduce workflows).

**Signature:**
```python
class Send:
    def __init__(self, /, node: str, arg: Any) -> None:
```

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| node | str | The name of the target node to send the message to |
| arg | Any | The state or message to send to the target node |

**Example:**
```python
from typing import Annotated
from langgraph.types import Send
from langgraph.graph import END, START, StateGraph
import operator


class OverallState(TypedDict):
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]


def continue_to_jokes(state: OverallState):
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

---

### Command

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:364`

One or more commands to update the graph's state and send messages to nodes. Commands provide a powerful way to control graph execution from within nodes.

**Signature:**
```python
@dataclass
class Command(Generic[N], ToolOutputMixin):
    graph: str | None = None
    update: Any | None = None
    resume: dict[str, Any] | Any | None = None
    goto: Send | Sequence[Send | N] | N = ()
```

**Attributes:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| graph | str \| None | None | Graph to send the command to. Supported values: None (current graph), Command.PARENT (closest parent graph) |
| update | Any \| None | None | Update to apply to the graph's state |
| resume | dict[str, Any] \| Any \| None | None | Value to resume execution with. To be used together with `interrupt()`. Can be a mapping of interrupt ids to resume values, or a single value to resume the next interrupt |
| goto | Send \| Sequence[Send \| N] \| N | () | Can be: name of the node to navigate to next (any node that belongs to the specified graph), sequence of node names to navigate to next, Send object (to execute a node with the input provided), or sequence of Send objects |

**Constants:**

| Name | Type | Value | Description |
|------|------|-------|-------------|
| PARENT | Literal["__parent__"] | "__parent__" | Special value to target the parent graph |

**Example:**
```python
from langgraph.types import Command
from langgraph.graph import StateGraph


class State(TypedDict):
    value: int


def node_a(state: State):
    # Update state and navigate to node_b
    return Command(
        update={"value": state["value"] + 1},
        goto="node_b"
    )


def node_b(state: State):
    # Just update state
    return Command(update={"value": state["value"] * 2})


builder = StateGraph(State)
builder.add_node("node_a", node_a)
builder.add_node("node_b", node_b)
builder.set_entry_point("node_a")
builder.set_finish_point("node_b")
graph = builder.compile()

result = graph.invoke({"value": 5})
# {'value': 12}  # (5 + 1) * 2
```

---

## Constants

### START

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/constants.py:30`

The first (virtual) node in graph-style Pregel. Use this constant to create edges from the graph's entry point.

**Signature:**
```python
START: str = "__start__"
```

**Example:**
```python
from langgraph.graph import START, StateGraph

builder = StateGraph(State)
builder.add_node("first_node", first_node_fn)
builder.add_edge(START, "first_node")
```

---

### END

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/constants.py:28`

The last (virtual) node in graph-style Pregel. Use this constant to mark nodes as terminal points in the graph.

**Signature:**
```python
END: str = "__end__"
```

**Example:**
```python
from langgraph.graph import END, StateGraph

builder = StateGraph(State)
builder.add_node("final_node", final_node_fn)
builder.add_edge("final_node", END)
```

---

## Types and Helpers

### StateSnapshot

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:264`

Snapshot of the state of the graph at the beginning of a step. This is returned by `get_state()` and `get_state_history()`.

**Signature:**
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

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| values | dict[str, Any] \| Any | Current values of channels (the state) |
| next | tuple[str, ...] | The name of the node to execute in each task for this step |
| config | RunnableConfig | Config used to fetch this snapshot |
| metadata | CheckpointMetadata \| None | Metadata associated with this snapshot |
| created_at | str \| None | Timestamp of snapshot creation |
| parent_config | RunnableConfig \| None | Config used to fetch the parent snapshot, if any |
| tasks | tuple[PregelTask, ...] | Tasks to execute in this step. If already attempted, may contain an error |
| interrupts | tuple[Interrupt, ...] | Interrupts that occurred in this step that are pending resolution |

**Example:**
```python
# Get current state snapshot
snapshot = graph.get_state({"configurable": {"thread_id": "thread-1"}})

print(f"Current state: {snapshot.values}")
print(f"Next nodes: {snapshot.next}")
print(f"Created at: {snapshot.created_at}")
print(f"Metadata: {snapshot.metadata}")

# Check if there are pending tasks
if snapshot.tasks:
    print(f"Pending tasks: {[t.name for t in snapshot.tasks]}")

# Check for interrupts
if snapshot.interrupts:
    print(f"Interrupts: {snapshot.interrupts}")
```

---

## Additional Methods

### CompiledStateGraph.get_input_jsonschema

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:950`

Get the JSON schema for the graph's input.

**Signature:**
```python
def get_input_jsonschema(
    self,
    config: RunnableConfig | None = None
) -> dict[str, Any]:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | Optional configuration |

**Returns:**

| Type | Description |
|------|-------------|
| dict[str, Any] | JSON schema for the input |

---

### CompiledStateGraph.get_output_jsonschema

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/graph/state.py:960`

Get the JSON schema for the graph's output.

**Signature:**
```python
def get_output_jsonschema(
    self,
    config: RunnableConfig | None = None
) -> dict[str, Any]:
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | Optional configuration |

**Returns:**

| Type | Description |
|------|-------------|
| dict[str, Any] | JSON schema for the output |

---

## Summary

This API reference covers all the core components of the LangGraph Graph Building module:

- **StateGraph**: The main builder class for creating stateful multi-agent workflows
- **CompiledStateGraph**: The executable runtime that provides invoke(), stream(), get_state(), and update_state() methods
- **MessagesState & add_messages**: Pre-built utilities for message-based workflows
- **Node types**: Various signatures supported for graph nodes
- **Branching utilities**: Send and Command for dynamic routing
- **Constants**: START and END for graph entry and exit points
- **State management**: StateSnapshot for inspecting graph state and history

For more examples and tutorials, see the [LangGraph documentation](https://langchain-ai.github.io/langgraph/).
