# Pregel (Execution Engine) API Reference

This document provides a comprehensive API reference for the LangGraph Pregel execution engine module. The Pregel class is the core execution engine that powers LangGraph, managing the runtime behavior for stateful, multi-actor agent applications.

## Table of Contents

- [Core Classes](#core-classes)
  - [Pregel](#pregel-class)
  - [NodeBuilder](#nodebuilder-class)
  - [PregelNode](#pregelnode-class)
  - [RemoteGraph](#remotegraph-class)
- [Channel Operations](#channel-operations)
  - [ChannelRead](#channelread-class)
  - [ChannelWrite](#channelwrite-class)
- [Protocols](#protocols)
  - [PregelProtocol](#pregelprotocol-class)
  - [StreamProtocol](#streamprotocol-class)
- [Helper Types](#helper-types)

---

## Core Classes

### Pregel Class

The `Pregel` class is the core execution engine that manages runtime behavior for LangGraph applications. It combines actors (nodes) and channels into a single application following the Pregel/Bulk Synchronous Parallel model.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:323`

#### __init__

**Signature:**
```python
def __init__(
    self,
    *,
    nodes: dict[str, PregelNode | NodeBuilder],
    channels: dict[str, BaseChannel | ManagedValueSpec] | None,
    auto_validate: bool = True,
    stream_mode: StreamMode = "values",
    stream_eager: bool = False,
    output_channels: str | Sequence[str],
    stream_channels: str | Sequence[str] | None = None,
    interrupt_after_nodes: All | Sequence[str] = (),
    interrupt_before_nodes: All | Sequence[str] = (),
    input_channels: str | Sequence[str],
    step_timeout: float | None = None,
    debug: bool | None = None,
    checkpointer: Checkpointer = None,
    store: BaseStore | None = None,
    cache: BaseCache | None = None,
    retry_policy: RetryPolicy | Sequence[RetryPolicy] = (),
    cache_policy: CachePolicy | None = None,
    context_schema: type[ContextT] | None = None,
    config: RunnableConfig | None = None,
    trigger_to_nodes: Mapping[str, Sequence[str]] | None = None,
    name: str = "LangGraph",
    **deprecated_kwargs: Unpack[DeprecatedKwargs],
) -> None
```

**Description:**
Initialize a Pregel execution engine instance with the specified configuration.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| nodes | dict[str, PregelNode \| NodeBuilder] | Yes | - | Dictionary mapping node names to PregelNode or NodeBuilder instances |
| channels | dict[str, BaseChannel \| ManagedValueSpec] \| None | Yes | - | Dictionary of channels for inter-node communication |
| auto_validate | bool | No | True | Whether to automatically validate the graph structure |
| stream_mode | StreamMode | No | "values" | Default streaming mode for graph execution |
| stream_eager | bool | No | False | Whether to force emitting stream events eagerly |
| output_channels | str \| Sequence[str] | Yes | - | Channel(s) to use for graph output |
| stream_channels | str \| Sequence[str] \| None | No | None | Channels to stream (defaults to all non-reserved channels) |
| interrupt_after_nodes | All \| Sequence[str] | No | () | Nodes to interrupt execution after |
| interrupt_before_nodes | All \| Sequence[str] | No | () | Nodes to interrupt execution before |
| input_channels | str \| Sequence[str] | Yes | - | Channel(s) to use for graph input |
| step_timeout | float \| None | No | None | Maximum time to wait for a step to complete, in seconds |
| debug | bool \| None | No | None | Whether to print debug information during execution |
| checkpointer | Checkpointer | No | None | Checkpointer used to save and load graph state |
| store | BaseStore \| None | No | None | Memory store to use for SharedValues |
| cache | BaseCache \| None | No | None | Cache to use for storing node results |
| retry_policy | RetryPolicy \| Sequence[RetryPolicy] | No | () | Retry policies to use when running tasks |
| cache_policy | CachePolicy \| None | No | None | Cache policy to use for all nodes |
| context_schema | type[ContextT] \| None | No | None | Schema for the context object passed to the workflow |
| config | RunnableConfig \| None | No | None | Default configuration for the graph |
| trigger_to_nodes | Mapping[str, Sequence[str]] \| None | No | None | Mapping of triggers to nodes |
| name | str | No | "LangGraph" | Name of the graph |

**Returns:**
| Type | Description |
|------|-------------|
| None | Initializes the Pregel instance |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If the TASKS channel is reserved or if validation fails |

**Example:**
```python
from langgraph.channels import LastValue, EphemeralValue
from langgraph.pregel import Pregel, NodeBuilder

node1 = (
    NodeBuilder()
    .subscribe_only("a")
    .do(lambda x: x + x)
    .write_to("b")
    .build()
)

app = Pregel(
    nodes={"node1": node1},
    channels={
        "a": EphemeralValue(str),
        "b": LastValue(str),
    },
    input_channels=["a"],
    output_channels=["b"],
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:630`

---

#### invoke

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
) -> dict[str, Any] | Any
```

**Description:**
Run the graph with a single input and config, waiting for completion.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | The input data for the graph |
| config | RunnableConfig \| None | No | None | The configuration for the graph run |
| context | ContextT \| None | No | None | The static context to use for the run |
| stream_mode | StreamMode | No | "values" | The stream mode for the graph run |
| print_mode | StreamMode \| Sequence[StreamMode] | No | () | Print output to console for debugging |
| output_keys | str \| Sequence[str] \| None | No | None | The output keys to retrieve from the graph run |
| interrupt_before | All \| Sequence[str] \| None | No | None | The nodes to interrupt the graph run before |
| interrupt_after | All \| Sequence[str] \| None | No | None | The nodes to interrupt the graph run after |
| durability | Durability \| None | No | None | The durability mode ("sync", "async", or "exit") |

**Returns:**
| Type | Description |
|------|-------------|
| dict[str, Any] \| Any | The output of the graph run |

**Raises:**
| Exception | When |
|-----------|------|
| GraphRecursionError | If recursion limit is reached |

**Example:**
```python
result = app.invoke({"a": "hello"})
# Returns: {"b": "hellohello"}
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:3024`

---

#### ainvoke

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
) -> dict[str, Any] | Any
```

**Description:**
Asynchronously run the graph with a single input and config.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | The input data for the graph |
| config | RunnableConfig \| None | No | None | The configuration for the graph run |
| context | ContextT \| None | No | None | The static context to use for the run |
| stream_mode | StreamMode | No | "values" | The stream mode for the graph run |
| print_mode | StreamMode \| Sequence[StreamMode] | No | () | Print output to console for debugging |
| output_keys | str \| Sequence[str] \| None | No | None | The output keys to retrieve from the graph run |
| interrupt_before | All \| Sequence[str] \| None | No | None | The nodes to interrupt the graph run before |
| interrupt_after | All \| Sequence[str] \| None | No | None | The nodes to interrupt the graph run after |
| durability | Durability \| None | No | None | The durability mode ("sync", "async", or "exit") |

**Returns:**
| Type | Description |
|------|-------------|
| dict[str, Any] \| Any | The output of the graph run |

**Raises:**
| Exception | When |
|-----------|------|
| GraphRecursionError | If recursion limit is reached |

**Example:**
```python
result = await app.ainvoke({"a": "hello"})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:3114`

---

#### stream

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
    **kwargs: Unpack[DeprecatedKwargs],
) -> Iterator[dict[str, Any] | Any]
```

**Description:**
Stream graph steps for a single input. Yields output at each step according to the stream_mode.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | The input to the graph |
| config | RunnableConfig \| None | No | None | The configuration to use for the run |
| context | ContextT \| None | No | None | The static context to use for the run |
| stream_mode | StreamMode \| Sequence[StreamMode] \| None | No | None | The mode(s) to stream output. Options: "values", "updates", "custom", "messages", "checkpoints", "tasks", "debug" |
| print_mode | StreamMode \| Sequence[StreamMode] | No | () | Same as stream_mode but only prints to console |
| output_keys | str \| Sequence[str] \| None | No | None | The keys to stream (defaults to all non-context channels) |
| interrupt_before | All \| Sequence[str] \| None | No | None | Nodes to interrupt before |
| interrupt_after | All \| Sequence[str] \| None | No | None | Nodes to interrupt after |
| durability | Durability \| None | No | None | The durability mode ("sync", "async", or "exit") |
| subgraphs | bool | No | False | Whether to stream events from inside subgraphs |
| debug | bool \| None | No | None | Whether to print debug information |

**Returns:**
| Type | Description |
|------|-------------|
| Iterator[dict[str, Any] \| Any] | Iterator yielding graph output at each step |

**Raises:**
| Exception | When |
|-----------|------|
| GraphRecursionError | If recursion limit is reached |

**Example:**
```python
for chunk in app.stream({"a": "hello"}):
    print(chunk)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:2407`

---

#### astream

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
    **kwargs: Unpack[DeprecatedKwargs],
) -> AsyncIterator[dict[str, Any] | Any]
```

**Description:**
Asynchronously stream graph steps for a single input.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | The input to the graph |
| config | RunnableConfig \| None | No | None | The configuration to use for the run |
| context | ContextT \| None | No | None | The static context to use for the run |
| stream_mode | StreamMode \| Sequence[StreamMode] \| None | No | None | The mode(s) to stream output |
| print_mode | StreamMode \| Sequence[StreamMode] | No | () | Same as stream_mode but only prints to console |
| output_keys | str \| Sequence[str] \| None | No | None | The keys to stream |
| interrupt_before | All \| Sequence[str] \| None | No | None | Nodes to interrupt before |
| interrupt_after | All \| Sequence[str] \| None | No | None | Nodes to interrupt after |
| durability | Durability \| None | No | None | The durability mode |
| subgraphs | bool | No | False | Whether to stream events from inside subgraphs |
| debug | bool \| None | No | None | Whether to print debug information |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[dict[str, Any] \| Any] | Async iterator yielding graph output |

**Raises:**
| Exception | When |
|-----------|------|
| GraphRecursionError | If recursion limit is reached |

**Example:**
```python
async for chunk in app.astream({"a": "hello"}):
    print(chunk)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:2681`

---

#### get_state

**Signature:**
```python
def get_state(
    self,
    config: RunnableConfig,
    *,
    subgraphs: bool = False
) -> StateSnapshot
```

**Description:**
Get the current state of the graph. Requires a checkpointer to be configured.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| subgraphs | bool | No | False | Whether to include subgraph states |

**Returns:**
| Type | Description |
|------|-------------|
| StateSnapshot | The current state snapshot of the graph |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set |

**Example:**
```python
state = app.get_state({"configurable": {"thread_id": "1"}})
print(state.values)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1235`

---

#### aget_state

**Signature:**
```python
async def aget_state(
    self,
    config: RunnableConfig,
    *,
    subgraphs: bool = False
) -> StateSnapshot
```

**Description:**
Asynchronously get the current state of the graph.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| subgraphs | bool | No | False | Whether to include subgraph states |

**Returns:**
| Type | Description |
|------|-------------|
| StateSnapshot | The current state snapshot of the graph |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set |

**Example:**
```python
state = await app.aget_state({"configurable": {"thread_id": "1"}})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1277`

---

#### get_state_history

**Signature:**
```python
def get_state_history(
    self,
    config: RunnableConfig,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[StateSnapshot]
```

**Description:**
Get the history of the state of the graph.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| filter | dict[str, Any] \| None | No | None | Metadata to filter on |
| before | RunnableConfig \| None | No | None | A RunnableConfig that includes checkpoint metadata |
| limit | int \| None | No | None | Max number of states to return |

**Returns:**
| Type | Description |
|------|-------------|
| Iterator[StateSnapshot] | Iterator of state snapshots in history |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set |

**Example:**
```python
for state in app.get_state_history({"configurable": {"thread_id": "1"}}, limit=5):
    print(state.values)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1319`

---

#### aget_state_history

**Signature:**
```python
async def aget_state_history(
    self,
    config: RunnableConfig,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> AsyncIterator[StateSnapshot]
```

**Description:**
Asynchronously get the history of the state of the graph.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| filter | dict[str, Any] \| None | No | None | Metadata to filter on |
| before | RunnableConfig \| None | No | None | A RunnableConfig that includes checkpoint metadata |
| limit | int \| None | No | None | Max number of states to return |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[StateSnapshot] | Async iterator of state snapshots |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set |

**Example:**
```python
async for state in app.aget_state_history({"configurable": {"thread_id": "1"}}):
    print(state.values)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1370`

---

#### update_state

**Signature:**
```python
def update_state(
    self,
    config: RunnableConfig,
    values: dict[str, Any] | Any | None,
    as_node: str | None = None,
    task_id: str | None = None,
) -> RunnableConfig
```

**Description:**
Update the state of the graph with the given values, as if they came from node `as_node`. If `as_node` is not provided, it will be set to the last node that updated the state, if not ambiguous.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The config to apply the updates to |
| values | dict[str, Any] \| Any \| None | Yes | - | Values to update to the state |
| as_node | str \| None | No | None | Update the state as if this node had just executed |
| task_id | str \| None | No | None | Optional task ID for the update |

**Returns:**
| Type | Description |
|------|-------------|
| RunnableConfig | RunnableConfig for the updated thread |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set |
| InvalidUpdateError | If an invalid update is provided |

**Example:**
```python
config = app.update_state(
    {"configurable": {"thread_id": "1"}},
    {"key": "new_value"},
    as_node="node1"
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:2309`

---

#### aupdate_state

**Signature:**
```python
async def aupdate_state(
    self,
    config: RunnableConfig,
    values: dict[str, Any] | Any,
    as_node: str | None = None,
    task_id: str | None = None,
) -> RunnableConfig
```

**Description:**
Asynchronously update the state of the graph with the given values.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The config to apply the updates to |
| values | dict[str, Any] \| Any | Yes | - | Values to update to the state |
| as_node | str \| None | No | None | Update the state as if this node had just executed |
| task_id | str \| None | No | None | Optional task ID for the update |

**Returns:**
| Type | Description |
|------|-------------|
| RunnableConfig | RunnableConfig for the updated thread |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set |
| InvalidUpdateError | If an invalid update is provided |

**Example:**
```python
config = await app.aupdate_state(
    {"configurable": {"thread_id": "1"}},
    {"key": "new_value"}
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:2322`

---

#### bulk_update_state

**Signature:**
```python
def bulk_update_state(
    self,
    config: RunnableConfig,
    supersteps: Sequence[Sequence[StateUpdate]],
) -> RunnableConfig
```

**Description:**
Apply updates to the graph state in bulk. Requires a checkpointer to be set.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The config to apply the updates to |
| supersteps | Sequence[Sequence[StateUpdate]] | Yes | - | A list of supersteps, each including a list of updates to apply sequentially. Each update is a tuple of the form (values, as_node, task_id) where task_id is optional |

**Returns:**
| Type | Description |
|------|-------------|
| RunnableConfig | The updated config |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set or no updates are provided |
| InvalidUpdateError | If an invalid update is provided |

**Example:**
```python
config = app.bulk_update_state(
    {"configurable": {"thread_id": "1"}},
    [
        [StateUpdate({"key1": "value1"}, "node1")],
        [StateUpdate({"key2": "value2"}, "node2")]
    ]
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1425`

---

#### abulk_update_state

**Signature:**
```python
async def abulk_update_state(
    self,
    config: RunnableConfig,
    supersteps: Sequence[Sequence[StateUpdate]],
) -> RunnableConfig
```

**Description:**
Asynchronously apply updates to the graph state in bulk.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The config to apply the updates to |
| supersteps | Sequence[Sequence[StateUpdate]] | Yes | - | A list of supersteps, each including a list of updates |

**Returns:**
| Type | Description |
|------|-------------|
| RunnableConfig | The updated config |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no checkpointer is set or no updates are provided |
| InvalidUpdateError | If an invalid update is provided |

**Example:**
```python
config = await app.abulk_update_state(
    {"configurable": {"thread_id": "1"}},
    [[StateUpdate({"key": "value"}, "node1")]]
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:1869`

---

#### get_graph

**Signature:**
```python
def get_graph(
    self,
    config: RunnableConfig | None = None,
    *,
    xray: int | bool = False
) -> Graph
```

**Description:**
Return a drawable representation of the computation graph.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | The configuration for the graph |
| xray | int \| bool | No | False | Include graph representation of subgraphs. If an integer, only subgraphs with depth <= value will be included |

**Returns:**
| Type | Description |
|------|-------------|
| Graph | A drawable graph representation |

**Example:**
```python
graph = app.get_graph()
print(graph.to_ascii())
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:703`

---

#### aget_graph

**Signature:**
```python
async def aget_graph(
    self,
    config: RunnableConfig | None = None,
    *,
    xray: int | bool = False
) -> Graph
```

**Description:**
Asynchronously return a drawable representation of the computation graph.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | The configuration for the graph |
| xray | int \| bool | No | False | Include graph representation of subgraphs |

**Returns:**
| Type | Description |
|------|-------------|
| Graph | A drawable graph representation |

**Example:**
```python
graph = await app.aget_graph()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:731`

---

#### get_subgraphs

**Signature:**
```python
def get_subgraphs(
    self,
    *,
    namespace: str | None = None,
    recurse: bool = False
) -> Iterator[tuple[str, PregelProtocol]]
```

**Description:**
Get the subgraphs of the graph.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| namespace | str \| None | No | None | The namespace to filter the subgraphs by |
| recurse | bool | No | False | Whether to recurse into the subgraphs. If False, only immediate subgraphs will be returned |

**Returns:**
| Type | Description |
|------|-------------|
| Iterator[tuple[str, PregelProtocol]] | An iterator of (namespace, subgraph) pairs |

**Example:**
```python
for name, subgraph in app.get_subgraphs():
    print(f"Subgraph: {name}")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:932`

---

#### aget_subgraphs

**Signature:**
```python
async def aget_subgraphs(
    self,
    *,
    namespace: str | None = None,
    recurse: bool = False
) -> AsyncIterator[tuple[str, PregelProtocol]]
```

**Description:**
Asynchronously get the subgraphs of the graph.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| namespace | str \| None | No | None | The namespace to filter the subgraphs by |
| recurse | bool | No | False | Whether to recurse into the subgraphs |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[tuple[str, PregelProtocol]] | An async iterator of (namespace, subgraph) pairs |

**Example:**
```python
async for name, subgraph in app.aget_subgraphs():
    print(f"Subgraph: {name}")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:971`

---

#### with_config

**Signature:**
```python
def with_config(
    self,
    config: RunnableConfig | None = None,
    **kwargs: Any
) -> Self
```

**Description:**
Create a copy of the Pregel object with an updated config.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | Config to merge with current config |
| **kwargs | Any | No | - | Additional config values to merge |

**Returns:**
| Type | Description |
|------|-------------|
| Self | A new Pregel instance with merged config |

**Example:**
```python
configured_app = app.with_config({"recursion_limit": 100})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:785`

---

#### validate

**Signature:**
```python
def validate(self) -> Self
```

**Description:**
Validate the graph structure to ensure it is properly configured.

**Returns:**
| Type | Description |
|------|-------------|
| Self | Returns self for chaining |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If the graph structure is invalid |

**Example:**
```python
app.validate()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:791`

---

#### clear_cache

**Signature:**
```python
def clear_cache(self, nodes: Sequence[str] | None = None) -> None
```

**Description:**
Clear the cache for the given nodes.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| nodes | Sequence[str] \| None | No | None | Node names to clear cache for (defaults to all nodes) |

**Returns:**
| Type | Description |
|------|-------------|
| None | - |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no cache is set for this graph |

**Example:**
```python
app.clear_cache(["node1", "node2"])
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:3204`

---

#### aclear_cache

**Signature:**
```python
async def aclear_cache(self, nodes: Sequence[str] | None = None) -> None
```

**Description:**
Asynchronously clear the cache for the given nodes.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| nodes | Sequence[str] \| None | No | None | Node names to clear cache for |

**Returns:**
| Type | Description |
|------|-------------|
| None | - |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If no cache is set for this graph |

**Example:**
```python
await app.aclear_cache()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:3223`

---

#### get_input_schema

**Signature:**
```python
def get_input_schema(self, config: RunnableConfig | None = None) -> type[BaseModel]
```

**Description:**
Get the input schema for the graph as a Pydantic model.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | The configuration for schema generation |

**Returns:**
| Type | Description |
|------|-------------|
| type[BaseModel] | A Pydantic model representing the input schema |

**Example:**
```python
schema = app.get_input_schema()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:870`

---

#### get_output_schema

**Signature:**
```python
def get_output_schema(self, config: RunnableConfig | None = None) -> type[BaseModel]
```

**Description:**
Get the output schema for the graph as a Pydantic model.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | The configuration for schema generation |

**Returns:**
| Type | Description |
|------|-------------|
| type[BaseModel] | A Pydantic model representing the output schema |

**Example:**
```python
schema = app.get_output_schema()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:897`

---

#### get_context_jsonschema

**Signature:**
```python
def get_context_jsonschema(self) -> dict[str, Any] | None
```

**Description:**
Get the JSON schema for the context object.

**Returns:**
| Type | Description |
|------|-------------|
| dict[str, Any] \| None | JSON schema for the context, or None if no context schema is defined |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If the context schema type is invalid |

**Example:**
```python
schema = app.get_context_jsonschema()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:850`

---

### NodeBuilder Class

The `NodeBuilder` class provides a fluent interface for building PregelNode instances. It allows you to configure node behavior step by step.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:159`

#### __init__

**Signature:**
```python
def __init__(self) -> None
```

**Description:**
Initialize a new NodeBuilder instance with default empty configuration.

**Returns:**
| Type | Description |
|------|-------------|
| None | Initializes the NodeBuilder |

**Example:**
```python
builder = NodeBuilder()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:180`

---

#### subscribe_only

**Signature:**
```python
def subscribe_only(self, channel: str) -> Self
```

**Description:**
Subscribe to a single channel. The node will receive the channel's value directly (not as a dict).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| channel | str | Yes | - | Channel name to subscribe to |

**Returns:**
| Type | Description |
|------|-------------|
| Self | Self for method chaining |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If other channels are already subscribed to |

**Example:**
```python
node = NodeBuilder().subscribe_only("input")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:192`

---

#### subscribe_to

**Signature:**
```python
def subscribe_to(
    self,
    *channels: str,
    read: bool = True,
) -> Self
```

**Description:**
Add channels to subscribe to. Node will be invoked when any of these channels are updated, with a dict of the channel values as input.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| *channels | str | Yes | - | Channel name(s) to subscribe to |
| read | bool | No | True | If True, the channels will be included in the input to the node. Otherwise, they will trigger the node without being sent in input |

**Returns:**
| Type | Description |
|------|-------------|
| Self | Self for method chaining |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If subscribed to a single channel |

**Example:**
```python
node = NodeBuilder().subscribe_to("channel1", "channel2")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:208`

---

#### read_from

**Signature:**
```python
def read_from(self, *channels: str) -> Self
```

**Description:**
Adds the specified channels to read from, without subscribing to them.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| *channels | str | Yes | - | Channel names to read from |

**Returns:**
| Type | Description |
|------|-------------|
| Self | Self for method chaining |

**Example:**
```python
node = NodeBuilder().subscribe_to("trigger").read_from("data1", "data2")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:243`

---

#### do

**Signature:**
```python
def do(self, node: RunnableLike) -> Self
```

**Description:**
Adds the specified node (runnable) to execute. Multiple calls will chain the runnables in sequence.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| node | RunnableLike | Yes | - | The runnable to execute |

**Returns:**
| Type | Description |
|------|-------------|
| Self | Self for method chaining |

**Example:**
```python
node = NodeBuilder().subscribe_only("input").do(lambda x: x.upper())
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:254`

---

#### write_to

**Signature:**
```python
def write_to(
    self,
    *channels: str | ChannelWriteEntry,
    **kwargs: _WriteValue,
) -> Self
```

**Description:**
Add channel writes. Writes will be executed after the node completes.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| *channels | str \| ChannelWriteEntry | No | - | Channel names to write to, or ChannelWriteEntry objects |
| **kwargs | _WriteValue | No | - | Channel name and value mappings. Value can be a callable (mapper) or a constant |

**Returns:**
| Type | Description |
|------|-------------|
| Self | Self for method chaining |

**Example:**
```python
node = (
    NodeBuilder()
    .subscribe_only("input")
    .do(lambda x: x.upper())
    .write_to("output")
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:267`

---

#### meta

**Signature:**
```python
def meta(self, *tags: str, **metadata: Any) -> Self
```

**Description:**
Add tags or metadata to the node for tracing and monitoring.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| *tags | str | No | - | Tags to add to the node |
| **metadata | Any | No | - | Metadata key-value pairs |

**Returns:**
| Type | Description |
|------|-------------|
| Self | Self for method chaining |

**Example:**
```python
node = NodeBuilder().subscribe_only("input").meta("important", version="1.0")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:293`

---

#### add_retry_policies

**Signature:**
```python
def add_retry_policies(self, *policies: RetryPolicy) -> Self
```

**Description:**
Adds retry policies to the node for error handling.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| *policies | RetryPolicy | Yes | - | Retry policies to add |

**Returns:**
| Type | Description |
|------|-------------|
| Self | Self for method chaining |

**Example:**
```python
from langgraph.pregel import RetryPolicy
node = NodeBuilder().subscribe_only("input").add_retry_policies(
    RetryPolicy(max_attempts=3)
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:299`

---

#### add_cache_policy

**Signature:**
```python
def add_cache_policy(self, policy: CachePolicy) -> Self
```

**Description:**
Adds cache policy to the node.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| policy | CachePolicy | Yes | - | Cache policy to add |

**Returns:**
| Type | Description |
|------|-------------|
| Self | Self for method chaining |

**Example:**
```python
from langgraph.types import CachePolicy
node = NodeBuilder().subscribe_only("input").add_cache_policy(
    CachePolicy()
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:304`

---

#### build

**Signature:**
```python
def build(self) -> PregelNode
```

**Description:**
Builds and returns the configured PregelNode.

**Returns:**
| Type | Description |
|------|-------------|
| PregelNode | The constructed PregelNode instance |

**Example:**
```python
node = (
    NodeBuilder()
    .subscribe_only("input")
    .do(lambda x: x.upper())
    .write_to("output")
    .build()
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/main.py:309`

---

### PregelNode Class

A node in a Pregel graph. This acts as a container for the components necessary to make a PregelExecutableTask for a node.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_read.py:95`

#### __init__

**Signature:**
```python
def __init__(
    self,
    *,
    channels: str | list[str],
    triggers: Sequence[str],
    mapper: Callable[[Any], Any] | None = None,
    writers: list[Runnable] | None = None,
    tags: list[str] | None = None,
    metadata: Mapping[str, Any] | None = None,
    bound: Runnable[Any, Any] | None = None,
    retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy | None = None,
    subgraphs: Sequence[PregelProtocol] | None = None,
) -> None
```

**Description:**
Initialize a PregelNode with the specified configuration.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| channels | str \| list[str] | Yes | - | The channels that will be passed as input to bound. If a str, the node will be invoked with its value if it isn't empty. If a list, the node will be invoked with a dict of those channels' values |
| triggers | Sequence[str] | Yes | - | If any of these channels is written to, this node will be triggered in the next step |
| mapper | Callable[[Any], Any] \| None | No | None | A function to transform the input before passing it to bound |
| writers | list[Runnable] \| None | No | None | A list of writers that will be executed after bound |
| tags | list[str] \| None | No | None | Tags to attach to the node for tracing |
| metadata | Mapping[str, Any] \| None | No | None | Metadata to attach to the node for tracing |
| bound | Runnable[Any, Any] \| None | No | None | The main logic of the node |
| retry_policy | RetryPolicy \| Sequence[RetryPolicy] \| None | No | None | The retry policies to use when invoking the node |
| cache_policy | CachePolicy \| None | No | None | The cache policy to use when invoking the node |
| subgraphs | Sequence[PregelProtocol] \| None | No | None | Subgraphs used by the node |

**Returns:**
| Type | Description |
|------|-------------|
| None | Initializes the PregelNode |

**Example:**
```python
from langgraph.pregel._read import PregelNode
node = PregelNode(
    channels=["input"],
    triggers=["input"],
    bound=lambda x: x["input"].upper()
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_read.py:135`

---

#### invoke

**Signature:**
```python
def invoke(
    self,
    input: Any,
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> Any
```

**Description:**
Invoke the node's bound runnable with the given input.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | Any | Yes | - | Input to the node |
| config | RunnableConfig \| None | No | None | Configuration for the run |
| **kwargs | Any \| None | No | - | Additional keyword arguments |

**Returns:**
| Type | Description |
|------|-------------|
| Any | The output of the bound runnable |

**Example:**
```python
result = node.invoke({"input": "hello"})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_read.py:226`

---

#### ainvoke

**Signature:**
```python
async def ainvoke(
    self,
    input: Any,
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> Any
```

**Description:**
Asynchronously invoke the node's bound runnable.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | Any | Yes | - | Input to the node |
| config | RunnableConfig \| None | No | None | Configuration for the run |
| **kwargs | Any \| None | No | - | Additional keyword arguments |

**Returns:**
| Type | Description |
|------|-------------|
| Any | The output of the bound runnable |

**Example:**
```python
result = await node.ainvoke({"input": "hello"})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_read.py:239`

---

#### stream

**Signature:**
```python
def stream(
    self,
    input: Any,
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> Iterator[Any]
```

**Description:**
Stream the output of the node's bound runnable.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | Any | Yes | - | Input to the node |
| config | RunnableConfig \| None | No | None | Configuration for the run |
| **kwargs | Any \| None | No | - | Additional keyword arguments |

**Returns:**
| Type | Description |
|------|-------------|
| Iterator[Any] | Iterator of output chunks |

**Example:**
```python
for chunk in node.stream({"input": "hello"}):
    print(chunk)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_read.py:252`

---

#### astream

**Signature:**
```python
async def astream(
    self,
    input: Any,
    config: RunnableConfig | None = None,
    **kwargs: Any | None,
) -> AsyncIterator[Any]
```

**Description:**
Asynchronously stream the output of the node's bound runnable.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | Any | Yes | - | Input to the node |
| config | RunnableConfig \| None | No | None | Configuration for the run |
| **kwargs | Any \| None | No | - | Additional keyword arguments |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[Any] | Async iterator of output chunks |

**Example:**
```python
async for chunk in node.astream({"input": "hello"}):
    print(chunk)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_read.py:265`

---

### RemoteGraph Class

The `RemoteGraph` class is a client implementation for calling remote APIs that implement the LangGraph Server API specification. It behaves the same way as a Graph and can be used directly as a node in another Graph.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:108`

#### __init__

**Signature:**
```python
def __init__(
    self,
    assistant_id: str,
    /,
    *,
    url: str | None = None,
    api_key: str | None = None,
    headers: dict[str, str] | None = None,
    client: LangGraphClient | None = None,
    sync_client: SyncLangGraphClient | None = None,
    config: RunnableConfig | None = None,
    name: str | None = None,
    distributed_tracing: bool = False,
) -> None
```

**Description:**
Initialize a RemoteGraph client. Specify `url`, `api_key`, and/or `headers` to create default sync and async clients. If `client` or `sync_client` are provided, they will be used instead of the default clients.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str | Yes | - | The assistant ID or graph name of the remote graph to use |
| url | str \| None | No | None | The URL of the remote API |
| api_key | str \| None | No | None | The API key to use for authentication. If not provided, it will be read from the environment |
| headers | dict[str, str] \| None | No | None | Additional headers to include in the requests |
| client | LangGraphClient \| None | No | None | A LangGraphClient instance to use instead of creating a default client |
| sync_client | SyncLangGraphClient \| None | No | None | A SyncLangGraphClient instance to use instead of creating a default client |
| config | RunnableConfig \| None | No | None | An optional RunnableConfig instance with additional configuration |
| name | str \| None | No | None | Human-readable name to attach to the RemoteGraph instance. Defaults to the assistant ID |
| distributed_tracing | bool | No | False | Whether to enable sending LangSmith distributed tracing headers |

**Returns:**
| Type | Description |
|------|-------------|
| None | Initializes the RemoteGraph |

**Example:**
```python
from langgraph.pregel.remote import RemoteGraph

remote = RemoteGraph(
    "my-assistant",
    url="https://api.example.com",
    api_key="my-key"
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:122`

---

#### invoke

**Signature:**
```python
def invoke(
    self,
    input: dict[str, Any] | Any,
    config: RunnableConfig | None = None,
    *,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
    **kwargs: Any,
) -> dict[str, Any] | Any
```

**Description:**
Create a run, wait until it finishes and return the final state.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | dict[str, Any] \| Any | Yes | - | Input to the graph |
| config | RunnableConfig \| None | No | None | A RunnableConfig for graph invocation |
| interrupt_before | All \| Sequence[str] \| None | No | None | Interrupt the graph before these nodes |
| interrupt_after | All \| Sequence[str] \| None | No | None | Interrupt the graph after these nodes |
| headers | dict[str, str] \| None | No | None | Additional headers to pass to the request |
| params | QueryParamTypes \| None | No | None | Additional query parameters to pass to the request |
| **kwargs | Any | No | - | Additional params to pass to RemoteGraph.stream |

**Returns:**
| Type | Description |
|------|-------------|
| dict[str, Any] \| Any | The output of the graph |

**Example:**
```python
result = remote.invoke({"input": "hello"})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:921`

---

#### ainvoke

**Signature:**
```python
async def ainvoke(
    self,
    input: dict[str, Any] | Any,
    config: RunnableConfig | None = None,
    *,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
    **kwargs: Any,
) -> dict[str, Any] | Any
```

**Description:**
Asynchronously create a run, wait until it finishes and return the final state.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | dict[str, Any] \| Any | Yes | - | Input to the graph |
| config | RunnableConfig \| None | No | None | A RunnableConfig for graph invocation |
| interrupt_before | All \| Sequence[str] \| None | No | None | Interrupt the graph before these nodes |
| interrupt_after | All \| Sequence[str] \| None | No | None | Interrupt the graph after these nodes |
| headers | dict[str, str] \| None | No | None | Additional headers to pass to the request |
| params | QueryParamTypes \| None | No | None | Additional query parameters |
| **kwargs | Any | No | - | Additional params to pass to RemoteGraph.astream |

**Returns:**
| Type | Description |
|------|-------------|
| dict[str, Any] \| Any | The output of the graph |

**Example:**
```python
result = await remote.ainvoke({"input": "hello"})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:962`

---

#### stream

**Signature:**
```python
def stream(
    self,
    input: dict[str, Any] | Any,
    config: RunnableConfig | None = None,
    *,
    stream_mode: StreamMode | list[StreamMode] | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    subgraphs: bool = False,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
    **kwargs: Any,
) -> Iterator[dict[str, Any] | Any]
```

**Description:**
Create a run and stream the results. This method calls `POST /threads/{thread_id}/runs/stream` if a `thread_id` is specified in the `configurable` field of the config or `POST /runs/stream` otherwise.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | dict[str, Any] \| Any | Yes | - | Input to the graph |
| config | RunnableConfig \| None | No | None | A RunnableConfig for graph invocation |
| stream_mode | StreamMode \| list[StreamMode] \| None | No | None | Stream mode(s) to use |
| interrupt_before | All \| Sequence[str] \| None | No | None | Interrupt the graph before these nodes |
| interrupt_after | All \| Sequence[str] \| None | No | None | Interrupt the graph after these nodes |
| subgraphs | bool | No | False | Stream from subgraphs |
| headers | dict[str, str] \| None | No | None | Additional headers to pass to the request |
| params | QueryParamTypes \| None | No | None | Additional query parameters |
| **kwargs | Any | No | - | Additional params to pass to client.runs.stream |

**Returns:**
| Type | Description |
|------|-------------|
| Iterator[dict[str, Any] \| Any] | Iterator yielding the output of the graph |

**Raises:**
| Exception | When |
|-----------|------|
| RemoteException | When an error occurs in the remote graph |
| GraphInterrupt | When the graph is interrupted |
| ParentCommand | When a parent command is issued |

**Example:**
```python
for chunk in remote.stream({"input": "hello"}):
    print(chunk)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:685`

---

#### astream

**Signature:**
```python
async def astream(
    self,
    input: dict[str, Any] | Any,
    config: RunnableConfig | None = None,
    *,
    stream_mode: StreamMode | list[StreamMode] | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    subgraphs: bool = False,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
    **kwargs: Any,
) -> AsyncIterator[dict[str, Any] | Any]
```

**Description:**
Asynchronously create a run and stream the results.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | dict[str, Any] \| Any | Yes | - | Input to the graph |
| config | RunnableConfig \| None | No | None | A RunnableConfig for graph invocation |
| stream_mode | StreamMode \| list[StreamMode] \| None | No | None | Stream mode(s) to use |
| interrupt_before | All \| Sequence[str] \| None | No | None | Interrupt the graph before these nodes |
| interrupt_after | All \| Sequence[str] \| None | No | None | Interrupt the graph after these nodes |
| subgraphs | bool | No | False | Stream from subgraphs |
| headers | dict[str, str] \| None | No | None | Additional headers to pass to the request |
| params | QueryParamTypes \| None | No | None | Additional query parameters |
| **kwargs | Any | No | - | Additional params to pass to client.runs.stream |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[dict[str, Any] \| Any] | Async iterator yielding the output of the graph |

**Raises:**
| Exception | When |
|-----------|------|
| RemoteException | When an error occurs in the remote graph |
| GraphInterrupt | When the graph is interrupted |
| ParentCommand | When a parent command is issued |

**Example:**
```python
async for chunk in remote.astream({"input": "hello"}):
    print(chunk)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:795`

---

#### get_state

**Signature:**
```python
def get_state(
    self,
    config: RunnableConfig,
    *,
    subgraphs: bool = False,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> StateSnapshot
```

**Description:**
Get the state of a thread. This method calls `POST /threads/{thread_id}/state/checkpoint` if a checkpoint is specified in the config or `GET /threads/{thread_id}/state` if no checkpoint is specified.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| subgraphs | bool | No | False | Include subgraphs in the state |
| headers | dict[str, str] \| None | No | None | Optional custom headers to include with the request |
| params | QueryParamTypes \| None | No | None | Optional query parameters to include with the request |

**Returns:**
| Type | Description |
|------|-------------|
| StateSnapshot | The latest state of the thread |

**Example:**
```python
state = remote.get_state({"configurable": {"thread_id": "1"}})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:398`

---

#### aget_state

**Signature:**
```python
async def aget_state(
    self,
    config: RunnableConfig,
    *,
    subgraphs: bool = False,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> StateSnapshot
```

**Description:**
Asynchronously get the state of a thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| subgraphs | bool | No | False | Include subgraphs in the state |
| headers | dict[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| StateSnapshot | The latest state of the thread |

**Example:**
```python
state = await remote.aget_state({"configurable": {"thread_id": "1"}})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:434`

---

#### get_state_history

**Signature:**
```python
def get_state_history(
    self,
    config: RunnableConfig,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Iterator[StateSnapshot]
```

**Description:**
Get the state history of a thread. This method calls `POST /threads/{thread_id}/history`.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| filter | dict[str, Any] \| None | No | None | Metadata to filter on |
| before | RunnableConfig \| None | No | None | A RunnableConfig that includes checkpoint metadata |
| limit | int \| None | No | None | Max number of states to return |
| headers | dict[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Iterator[StateSnapshot] | States of the thread |

**Example:**
```python
for state in remote.get_state_history({"configurable": {"thread_id": "1"}}):
    print(state.values)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:470`

---

#### aget_state_history

**Signature:**
```python
async def aget_state_history(
    self,
    config: RunnableConfig,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> AsyncIterator[StateSnapshot]
```

**Description:**
Asynchronously get the state history of a thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| filter | dict[str, Any] \| None | No | None | Metadata to filter on |
| before | RunnableConfig \| None | No | None | A RunnableConfig that includes checkpoint metadata |
| limit | int \| None | No | None | Max number of states to return |
| headers | dict[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[StateSnapshot] | Async iterator of states of the thread |

**Example:**
```python
async for state in remote.aget_state_history({"configurable": {"thread_id": "1"}}):
    print(state.values)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:509`

---

#### update_state

**Signature:**
```python
def update_state(
    self,
    config: RunnableConfig,
    values: dict[str, Any] | Any | None,
    as_node: str | None = None,
    *,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> RunnableConfig
```

**Description:**
Update the state of a thread. This method calls `POST /threads/{thread_id}/state`.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| values | dict[str, Any] \| Any \| None | Yes | - | Values to update to the state |
| as_node | str \| None | No | None | Update the state as if this node had just executed |
| headers | dict[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| RunnableConfig | RunnableConfig for the updated thread |

**Example:**
```python
config = remote.update_state(
    {"configurable": {"thread_id": "1"}},
    {"key": "value"},
    as_node="node1"
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:564`

---

#### aupdate_state

**Signature:**
```python
async def aupdate_state(
    self,
    config: RunnableConfig,
    values: dict[str, Any] | Any | None,
    as_node: str | None = None,
    *,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> RunnableConfig
```

**Description:**
Asynchronously update the state of a thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | A RunnableConfig that includes thread_id in the configurable field |
| values | dict[str, Any] \| Any \| None | Yes | - | Values to update to the state |
| as_node | str \| None | No | None | Update the state as if this node had just executed |
| headers | dict[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| RunnableConfig | RunnableConfig for the updated thread |

**Example:**
```python
config = await remote.aupdate_state(
    {"configurable": {"thread_id": "1"}},
    {"key": "value"}
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:599`

---

#### get_graph

**Signature:**
```python
def get_graph(
    self,
    config: RunnableConfig | None = None,
    *,
    xray: int | bool = False,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> DrawableGraph
```

**Description:**
Get graph by graph name. This method calls `GET /assistants/{assistant_id}/graph`.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | This parameter is not used |
| xray | int \| bool | No | False | Include graph representation of subgraphs. If an integer value is provided, only subgraphs with a depth less than or equal to the value will be included |
| headers | dict[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| DrawableGraph | The graph information for the assistant |

**Example:**
```python
graph = remote.get_graph()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:218`

---

#### aget_graph

**Signature:**
```python
async def aget_graph(
    self,
    config: RunnableConfig | None = None,
    *,
    xray: int | bool = False,
    headers: dict[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> DrawableGraph
```

**Description:**
Asynchronously get graph by graph name.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | This parameter is not used |
| xray | int \| bool | No | False | Include graph representation of subgraphs |
| headers | dict[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| DrawableGraph | The graph information for the assistant |

**Example:**
```python
graph = await remote.aget_graph()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:251`

---

#### with_config

**Signature:**
```python
def with_config(self, config: RunnableConfig | None = None, **kwargs: Any) -> Self
```

**Description:**
Create a copy of the RemoteGraph with an updated config.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | Config to merge with current config |
| **kwargs | Any | No | - | Additional config values |

**Returns:**
| Type | Description |
|------|-------------|
| Self | A new RemoteGraph instance with merged config |

**Example:**
```python
configured = remote.with_config({"recursion_limit": 50})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:189`

---

## Channel Operations

### ChannelRead Class

Implements the logic for reading state from CONFIG_KEY_READ. Usable both as a runnable as well as a static method to call imperatively.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_read.py:23`

#### __init__

**Signature:**
```python
def __init__(
    self,
    channel: str | list[str],
    *,
    fresh: bool = False,
    mapper: Callable[[Any], Any] | None = None,
    tags: list[str] | None = None,
) -> None
```

**Description:**
Initialize a ChannelRead instance for reading from channels.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| channel | str \| list[str] | Yes | - | Channel name(s) to read from |
| fresh | bool | No | False | Whether to read fresh values (bypass cache) |
| mapper | Callable[[Any], Any] \| None | No | None | Optional function to transform the read value |
| tags | list[str] \| None | No | None | Tags for tracing |

**Returns:**
| Type | Description |
|------|-------------|
| None | Initializes the ChannelRead |

**Example:**
```python
from langgraph.pregel._read import ChannelRead
reader = ChannelRead("my_channel")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_read.py:33`

---

#### do_read

**Signature:**
```python
@staticmethod
def do_read(
    config: RunnableConfig,
    *,
    select: str | list[str],
    fresh: bool = False,
    mapper: Callable[[Any], Any] | None = None,
) -> Any
```

**Description:**
Static method to read channel values from the execution context.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The runnable config containing the read function |
| select | str \| list[str] | Yes | - | Channel name(s) to read |
| fresh | bool | No | False | Whether to read fresh values |
| mapper | Callable[[Any], Any] \| None | No | None | Optional function to transform the value |

**Returns:**
| Type | Description |
|------|-------------|
| Any | The channel value(s) |

**Raises:**
| Exception | When |
|-----------|------|
| RuntimeError | If not configured with a read function |

**Example:**
```python
value = ChannelRead.do_read(config, select="my_channel")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_read.py:71`

---

### ChannelWrite Class

Implements the logic for sending writes to CONFIG_KEY_SEND. Can be used as a runnable or as a static method to call imperatively.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:46`

#### __init__

**Signature:**
```python
def __init__(
    self,
    writes: Sequence[ChannelWriteEntry | ChannelWriteTupleEntry | Send],
    *,
    tags: Sequence[str] | None = None,
) -> None
```

**Description:**
Initialize a ChannelWrite instance with the specified write entries.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| writes | Sequence[ChannelWriteEntry \| ChannelWriteTupleEntry \| Send] | Yes | - | Sequence of write entries or Send objects to write |
| tags | Sequence[str] \| None | No | None | Tags for tracing |

**Returns:**
| Type | Description |
|------|-------------|
| None | Initializes the ChannelWrite |

**Example:**
```python
from langgraph.pregel._write import ChannelWrite, ChannelWriteEntry
writer = ChannelWrite([ChannelWriteEntry("output")])
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:53`

---

#### do_write

**Signature:**
```python
@staticmethod
def do_write(
    config: RunnableConfig,
    writes: Sequence[ChannelWriteEntry | ChannelWriteTupleEntry | Send],
    allow_passthrough: bool = True,
) -> None
```

**Description:**
Static method to write values to channels.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The runnable config containing the write function |
| writes | Sequence[ChannelWriteEntry \| ChannelWriteTupleEntry \| Send] | Yes | - | The writes to execute |
| allow_passthrough | bool | No | True | Whether to allow PASSTHROUGH values |

**Returns:**
| Type | Description |
|------|-------------|
| None | - |

**Raises:**
| Exception | When |
|-----------|------|
| InvalidUpdateError | If writing to reserved channel TASKS or if PASSTHROUGH is not allowed |

**Example:**
```python
from langgraph.pregel._write import ChannelWrite, ChannelWriteEntry
ChannelWrite.do_write(
    config,
    [ChannelWriteEntry("output", value="hello")]
)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:105`

---

#### is_writer

**Signature:**
```python
@staticmethod
def is_writer(runnable: Runnable) -> bool
```

**Description:**
Check if a runnable is a writer. Used by PregelNode to distinguish between writers and other runnables.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| runnable | Runnable | Yes | - | The runnable to check |

**Returns:**
| Type | Description |
|------|-------------|
| bool | True if the runnable is a writer |

**Example:**
```python
is_writer = ChannelWrite.is_writer(some_runnable)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:128`

---

#### get_static_writes

**Signature:**
```python
@staticmethod
def get_static_writes(
    runnable: Runnable,
) -> Sequence[tuple[str, Any, str | None]] | None
```

**Description:**
Get conditional writes a writer declares for static analysis.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| runnable | Runnable | Yes | - | The runnable to analyze |

**Returns:**
| Type | Description |
|------|-------------|
| Sequence[tuple[str, Any, str \| None]] \| None | List of static write declarations or None |

**Example:**
```python
writes = ChannelWrite.get_static_writes(writer)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:136`

---

#### register_writer

**Signature:**
```python
@staticmethod
def register_writer(
    runnable: R,
    static: Sequence[tuple[ChannelWriteEntry | Send, str | None]] | None = None,
) -> R
```

**Description:**
Mark a runnable as a writer so that it can be detected by is_writer. Instances of ChannelWrite are automatically marked as writers.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| runnable | R | Yes | - | The runnable to mark as a writer |
| static | Sequence[tuple[ChannelWriteEntry \| Send, str \| None]] \| None | No | None | Optional list of declared writes for static analysis |

**Returns:**
| Type | Description |
|------|-------------|
| R | The same runnable, now marked as a writer |

**Example:**
```python
writer = ChannelWrite.register_writer(my_runnable)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:158`

---

## Protocols

### PregelProtocol Class

Abstract protocol defining the interface that all Pregel-compatible graph implementations must follow.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:17`

#### with_config

**Signature:**
```python
@abstractmethod
def with_config(
    self,
    config: RunnableConfig | None = None,
    **kwargs: Any
) -> Self
```

**Description:**
Create a copy with an updated config (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | Config to merge |
| **kwargs | Any | No | - | Additional config values |

**Returns:**
| Type | Description |
|------|-------------|
| Self | A new instance with merged config |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:18`

---

#### get_graph

**Signature:**
```python
@abstractmethod
def get_graph(
    self,
    config: RunnableConfig | None = None,
    *,
    xray: int | bool = False,
) -> DrawableGraph
```

**Description:**
Get a drawable representation of the graph (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | The configuration |
| xray | int \| bool | No | False | Include subgraphs |

**Returns:**
| Type | Description |
|------|-------------|
| DrawableGraph | A drawable graph representation |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:23`

---

#### aget_graph

**Signature:**
```python
@abstractmethod
async def aget_graph(
    self,
    config: RunnableConfig | None = None,
    *,
    xray: int | bool = False,
) -> DrawableGraph
```

**Description:**
Asynchronously get a drawable representation of the graph (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig \| None | No | None | The configuration |
| xray | int \| bool | No | False | Include subgraphs |

**Returns:**
| Type | Description |
|------|-------------|
| DrawableGraph | A drawable graph representation |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:31`

---

#### get_state

**Signature:**
```python
@abstractmethod
def get_state(
    self,
    config: RunnableConfig,
    *,
    subgraphs: bool = False
) -> StateSnapshot
```

**Description:**
Get the current state (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The configuration |
| subgraphs | bool | No | False | Include subgraphs |

**Returns:**
| Type | Description |
|------|-------------|
| StateSnapshot | The current state snapshot |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:39`

---

#### aget_state

**Signature:**
```python
@abstractmethod
async def aget_state(
    self,
    config: RunnableConfig,
    *,
    subgraphs: bool = False
) -> StateSnapshot
```

**Description:**
Asynchronously get the current state (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The configuration |
| subgraphs | bool | No | False | Include subgraphs |

**Returns:**
| Type | Description |
|------|-------------|
| StateSnapshot | The current state snapshot |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:44`

---

#### get_state_history

**Signature:**
```python
@abstractmethod
def get_state_history(
    self,
    config: RunnableConfig,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[StateSnapshot]
```

**Description:**
Get the state history (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The configuration |
| filter | dict[str, Any] \| None | No | None | Metadata to filter on |
| before | RunnableConfig \| None | No | None | Get states before this config |
| limit | int \| None | No | None | Maximum number of states to return |

**Returns:**
| Type | Description |
|------|-------------|
| Iterator[StateSnapshot] | Iterator of state snapshots |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:49`

---

#### aget_state_history

**Signature:**
```python
@abstractmethod
def aget_state_history(
    self,
    config: RunnableConfig,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> AsyncIterator[StateSnapshot]
```

**Description:**
Asynchronously get the state history (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The configuration |
| filter | dict[str, Any] \| None | No | None | Metadata to filter on |
| before | RunnableConfig \| None | No | None | Get states before this config |
| limit | int \| None | No | None | Maximum number of states to return |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[StateSnapshot] | Async iterator of state snapshots |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:59`

---

#### update_state

**Signature:**
```python
@abstractmethod
def update_state(
    self,
    config: RunnableConfig,
    values: dict[str, Any] | Any | None,
    as_node: str | None = None,
) -> RunnableConfig
```

**Description:**
Update the graph state (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The configuration |
| values | dict[str, Any] \| Any \| None | Yes | - | Values to update |
| as_node | str \| None | No | None | Update as if this node executed |

**Returns:**
| Type | Description |
|------|-------------|
| RunnableConfig | Updated configuration |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:83`

---

#### aupdate_state

**Signature:**
```python
@abstractmethod
async def aupdate_state(
    self,
    config: RunnableConfig,
    values: dict[str, Any] | Any | None,
    as_node: str | None = None,
) -> RunnableConfig
```

**Description:**
Asynchronously update the graph state (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| config | RunnableConfig | Yes | - | The configuration |
| values | dict[str, Any] \| Any \| None | Yes | - | Values to update |
| as_node | str \| None | No | None | Update as if this node executed |

**Returns:**
| Type | Description |
|------|-------------|
| RunnableConfig | Updated configuration |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:91`

---

#### stream

**Signature:**
```python
@abstractmethod
def stream(
    self,
    input: InputT | Command | None,
    config: RunnableConfig | None = None,
    *,
    context: ContextT | None = None,
    stream_mode: StreamMode | list[StreamMode] | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    subgraphs: bool = False,
) -> Iterator[dict[str, Any] | Any]
```

**Description:**
Stream graph execution (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | Input to the graph |
| config | RunnableConfig \| None | No | None | The configuration |
| context | ContextT \| None | No | None | Static context |
| stream_mode | StreamMode \| list[StreamMode] \| None | No | None | Stream mode(s) |
| interrupt_before | All \| Sequence[str] \| None | No | None | Nodes to interrupt before |
| interrupt_after | All \| Sequence[str] \| None | No | None | Nodes to interrupt after |
| subgraphs | bool | No | False | Stream from subgraphs |

**Returns:**
| Type | Description |
|------|-------------|
| Iterator[dict[str, Any] \| Any] | Iterator of output |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:99`

---

#### astream

**Signature:**
```python
@abstractmethod
def astream(
    self,
    input: InputT | Command | None,
    config: RunnableConfig | None = None,
    *,
    context: ContextT | None = None,
    stream_mode: StreamMode | list[StreamMode] | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    subgraphs: bool = False,
) -> AsyncIterator[dict[str, Any] | Any]
```

**Description:**
Asynchronously stream graph execution (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | Input to the graph |
| config | RunnableConfig \| None | No | None | The configuration |
| context | ContextT \| None | No | None | Static context |
| stream_mode | StreamMode \| list[StreamMode] \| None | No | None | Stream mode(s) |
| interrupt_before | All \| Sequence[str] \| None | No | None | Nodes to interrupt before |
| interrupt_after | All \| Sequence[str] \| None | No | None | Nodes to interrupt after |
| subgraphs | bool | No | False | Stream from subgraphs |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[dict[str, Any] \| Any] | Async iterator of output |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:112`

---

#### invoke

**Signature:**
```python
@abstractmethod
def invoke(
    self,
    input: InputT | Command | None,
    config: RunnableConfig | None = None,
    *,
    context: ContextT | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
) -> dict[str, Any] | Any
```

**Description:**
Invoke the graph synchronously (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | Input to the graph |
| config | RunnableConfig \| None | No | None | The configuration |
| context | ContextT \| None | No | None | Static context |
| interrupt_before | All \| Sequence[str] \| None | No | None | Nodes to interrupt before |
| interrupt_after | All \| Sequence[str] \| None | No | None | Nodes to interrupt after |

**Returns:**
| Type | Description |
|------|-------------|
| dict[str, Any] \| Any | The output |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:125`

---

#### ainvoke

**Signature:**
```python
@abstractmethod
async def ainvoke(
    self,
    input: InputT | Command | None,
    config: RunnableConfig | None = None,
    *,
    context: ContextT | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
) -> dict[str, Any] | Any
```

**Description:**
Asynchronously invoke the graph (abstract method).

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | InputT \| Command \| None | Yes | - | Input to the graph |
| config | RunnableConfig \| None | No | None | The configuration |
| context | ContextT \| None | No | None | Static context |
| interrupt_before | All \| Sequence[str] \| None | No | None | Nodes to interrupt before |
| interrupt_after | All \| Sequence[str] \| None | No | None | Nodes to interrupt after |

**Returns:**
| Type | Description |
|------|-------------|
| dict[str, Any] \| Any | The output |

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:136`

---

### StreamProtocol Class

Protocol for streaming data during graph execution.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:151`

#### __init__

**Signature:**
```python
def __init__(
    self,
    __call__: Callable[[StreamChunk], None],
    modes: set[StreamMode],
) -> None
```

**Description:**
Initialize a StreamProtocol instance.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| __call__ | Callable[[StreamChunk], None] | Yes | - | The callback function to handle stream chunks |
| modes | set[StreamMode] | Yes | - | The set of stream modes to use |

**Returns:**
| Type | Description |
|------|-------------|
| None | Initializes the StreamProtocol |

**Example:**
```python
from langgraph.pregel.protocol import StreamProtocol

def handler(chunk):
    print(chunk)

stream = StreamProtocol(handler, {"values", "updates"})
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/protocol.py:158`

---

## Helper Types

### ChannelWriteEntry

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:26`

A NamedTuple representing a single channel write operation.

**Fields:**
| Name | Type | Description |
|------|------|-------------|
| channel | str | Channel name to write to |
| value | Any | Value to write, or PASSTHROUGH to use the input (default: PASSTHROUGH) |
| skip_none | bool | Whether to skip writing if the value is None (default: False) |
| mapper | Callable \| None | Function to transform the value before writing (default: None) |

**Example:**
```python
from langgraph.pregel._write import ChannelWriteEntry
entry = ChannelWriteEntry(channel="output", value="hello", skip_none=True)
```

---

### ChannelWriteTupleEntry

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:37`

A NamedTuple for writing multiple tuples to channels.

**Fields:**
| Name | Type | Description |
|------|------|-------------|
| mapper | Callable[[Any], Sequence[tuple[str, Any]] \| None] | Function to extract tuples from value |
| value | Any | Value to write, or PASSTHROUGH to use the input (default: PASSTHROUGH) |
| static | Sequence[tuple[str, Any, str \| None]] \| None | Optional, declared writes for static analysis (default: None) |

**Example:**
```python
from langgraph.pregel._write import ChannelWriteTupleEntry

def mapper(x):
    return [("out1", x), ("out2", x * 2)]

entry = ChannelWriteTupleEntry(mapper=mapper)
```

---

### RemoteException

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/remote.py:102`

Exception raised when an error occurs in the remote graph.

**Example:**
```python
from langgraph.pregel.remote import RemoteException

try:
    result = remote.invoke({"input": "test"})
except RemoteException as e:
    print(f"Remote error: {e}")
```

---

## Constants

### PASSTHROUGH

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:23`

Sentinel value indicating that the input should be passed through without modification.

**Example:**
```python
from langgraph.pregel._write import PASSTHROUGH, ChannelWriteEntry

# This will pass the node's output to the channel
entry = ChannelWriteEntry("output", PASSTHROUGH)
```

---

### SKIP_WRITE

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_write.py:22`

Sentinel value that can be returned from a mapper to skip writing to a channel.

**Example:**
```python
from langgraph.pregel._write import SKIP_WRITE, ChannelWriteEntry

def conditional_mapper(value):
    if value is None:
        return SKIP_WRITE
    return value.upper()

entry = ChannelWriteEntry("output", mapper=conditional_mapper)
```

---

## Notes

### Stream Modes

The following stream modes are available for the `stream` and `astream` methods:

- **"values"**: Emit all values in the state after each step, including interrupts
- **"updates"**: Emit only the node or task names and updates returned by the nodes or tasks after each step
- **"custom"**: Emit custom data from inside nodes or tasks using `StreamWriter`
- **"messages"**: Emit LLM messages token-by-token together with metadata for any LLM invocations
- **"checkpoints"**: Emit an event when a checkpoint is created
- **"tasks"**: Emit events when tasks start and finish, including their results and errors
- **"debug"**: Emit debug events with as much information as possible for each step

### Durability Modes

The following durability modes are available for graph execution:

- **"sync"**: Changes are persisted synchronously before the next step starts
- **"async"**: Changes are persisted asynchronously while the next step executes (default)
- **"exit"**: Changes are persisted only when the graph exits

### Pregel Algorithm

The Pregel execution model follows the Bulk Synchronous Parallel paradigm with three phases per step:

1. **Plan**: Determine which actors (nodes) to execute in this step
2. **Execution**: Execute all selected actors in parallel until completion, failure, or timeout
3. **Update**: Update the channels with values written by the actors

This repeats until no actors are selected or the maximum number of steps is reached.

---

## See Also

- [StateGraph API Reference](01_GRAPH_API.md)
- [LangGraph Channels](../channels/index.md)
- [LangGraph Types](../types/index.md)
- [LangGraph Checkpointers](../checkpointers/index.md)
