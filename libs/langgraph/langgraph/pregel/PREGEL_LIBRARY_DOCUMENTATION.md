# Pregel Library Documentation

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [File Categories](#file-categories)
4. [Core Infrastructure Files](#1-core-infrastructure-files)
5. [Configuration & Validation Files](#2-configuration--validation-files)
6. [I/O & Communication Files](#3-io--communication-files)
7. [Execution Engine Files](#4-execution-engine-files)
8. [Support & Utility Files](#5-support--utility-files)
9. [Visualization & Remote Files](#6-visualization--remote-files)
10. [File Interaction Diagram](#file-interaction-diagram)
11. [Execution Flow](#execution-flow)

---

## Overview

The **Pregel** library is the core execution engine of LangGraph, implementing a graph-based computational model inspired by Google's Pregel system for large-scale graph processing. It provides the foundation for building stateful, multi-actor AI agents with support for:

- **Graph-based execution**: Nodes represent computational units that process data
- **Channel-based communication**: Data flows between nodes through typed channels
- **Checkpointing**: State persistence for fault tolerance and time-travel debugging
- **Streaming**: Real-time output of intermediate results
- **Interrupts**: Human-in-the-loop interaction patterns
- **Retry policies**: Automatic retry with backoff for failed operations
- **Subgraph support**: Nested graph execution with state isolation

The library consists of **22 Python files** organized into logical categories based on their responsibilities.

---

## Architecture

```
                                    +------------------+
                                    |     User API     |
                                    | (invoke/stream)  |
                                    +--------+---------+
                                             |
                                             v
+---------------------------+       +------------------+
|       main.py             |<----->|   protocol.py    |
|    (Pregel Class)         |       | (PregelProtocol) |
+---------------------------+       +------------------+
             |
             v
+---------------------------+       +------------------+
|      _validate.py         |       |    _config.py    |
|  (Graph Validation)       |       | (Configuration)  |
+---------------------------+       +------------------+
             |
             v
+---------------------------+
|       _loop.py            |
|   (Execution Loop)        |
+---------------------------+
             |
     +-------+-------+
     |               |
     v               v
+----------+   +------------+
| _algo.py |   | _runner.py |
| (Core    |   | (Task      |
| Algorithm)|  | Execution) |
+----------+   +------------+
     |               |
     +-------+-------+
             |
             v
+---------------------------+
|     I/O Layer             |
| _io.py, _read.py, _write.py|
| _messages.py              |
+---------------------------+
             |
             v
+---------------------------+
|    Support Layer          |
| _checkpoint.py, _retry.py |
| _executor.py, _call.py    |
+---------------------------+
```

---

## File Categories

| Category | Files | Purpose |
|----------|-------|---------|
| **Core Infrastructure** | `__init__.py`, `main.py`, `types.py`, `protocol.py`, `debug.py` | Main classes and type definitions |
| **Configuration & Validation** | `_config.py`, `_validate.py` | Graph configuration and validation |
| **I/O & Communication** | `_io.py`, `_read.py`, `_write.py`, `_messages.py` | Data flow and channel operations |
| **Execution Engine** | `_algo.py`, `_loop.py`, `_runner.py`, `_executor.py`, `_call.py` | Task scheduling and execution |
| **Support & Utilities** | `_checkpoint.py`, `_retry.py`, `_utils.py`, `_log.py` | Checkpointing, retries, helpers |
| **Visualization & Remote** | `_draw.py`, `remote.py` | Graph visualization and remote execution |

---

## 1. Core Infrastructure Files

### 1.1 `__init__.py`

**Location**: `langgraph/pregel/__init__.py`

**Purpose**: Package entry point that exposes the public API of the Pregel module.

**Key Exports**:
```python
from langgraph.pregel.main import Pregel
from langgraph.pregel.protocol import PregelProtocol
from langgraph.pregel.types import All, StateSnapshot, StreamMode
```

**Responsibilities**:
- Defines the public interface for the Pregel package
- Re-exports key classes and types for external use
- Controls what is accessible via `from langgraph.pregel import *`

---

### 1.2 `main.py`

**Location**: `langgraph/pregel/main.py`

**Purpose**: Contains the `Pregel` class - the main orchestrator for graph execution.

**Key Class**: `Pregel`

**Inheritance**: Extends `Runnable` from LangChain Core and implements `PregelProtocol`

**Key Attributes**:
```python
class Pregel:
    nodes: dict[str, PregelNode]           # Graph nodes
    channels: dict[str, BaseChannel]        # Communication channels
    input_channels: str | Sequence[str]     # Input channel names
    output_channels: str | Sequence[str]    # Output channel names
    stream_channels: str | Sequence[str]    # Channels to stream
    interrupt_before_nodes: Sequence[str]   # Nodes to interrupt before
    interrupt_after_nodes: Sequence[str]    # Nodes to interrupt after
    checkpointer: Checkpointer | None       # State persistence
    store: BaseStore | None                 # Key-value storage
    retry_policy: Sequence[RetryPolicy]     # Retry configuration
```

**Key Methods**:

| Method | Description |
|--------|-------------|
| `invoke(input, config)` | Execute graph synchronously |
| `ainvoke(input, config)` | Execute graph asynchronously |
| `stream(input, config)` | Stream execution results |
| `astream(input, config)` | Async stream execution results |
| `get_state(config)` | Get current graph state |
| `update_state(config, values)` | Update graph state manually |
| `get_state_history(config)` | Get historical states |
| `get_graph(config)` | Get drawable graph representation |
| `get_subgraphs()` | Enumerate nested subgraphs |

**Interactions**:
- Uses `_validate.py` for graph structure validation
- Delegates to `_loop.py` for execution
- Uses `_checkpoint.py` for state persistence
- Calls `_draw.py` for graph visualization

**Code Flow for `invoke()`**:
```
invoke()
  -> _transform()
    -> _prepare()
      -> validate graph
      -> setup channels
    -> AsyncPregelLoop / PregelLoop
      -> execute nodes
      -> apply writes
      -> checkpoint state
    -> return output
```

---

### 1.3 `types.py`

**Location**: `langgraph/pregel/types.py`

**Purpose**: Defines core type definitions and data structures used throughout the Pregel library.

**Key Types**:

```python
# Execution types
StreamMode = Literal["values", "updates", "debug", "messages", "custom"]
All = Literal["*"]

# State representation
@dataclass
class StateSnapshot:
    values: dict[str, Any]           # Current channel values
    next: tuple[str, ...]            # Nodes scheduled for execution
    config: RunnableConfig           # Configuration
    metadata: CheckpointMetadata     # Checkpoint metadata
    created_at: str | None           # Creation timestamp
    parent_config: RunnableConfig    # Parent checkpoint config
    tasks: tuple[PregelTask, ...]    # Pending tasks
    interrupts: tuple[Interrupt, ...]# Active interrupts

# Task representation
@dataclass
class PregelTask:
    id: str                          # Unique task ID
    name: str                        # Node name
    path: tuple[str | int, ...]      # Path in graph
    error: Exception | None          # Error if failed
    interrupts: tuple[Interrupt, ...]# Task interrupts
    state: StateSnapshot | None      # Subgraph state
    result: Any | None               # Task result

# Executable task (internal)
@dataclass
class PregelExecutableTask:
    id: str
    name: str
    input: Any
    proc: Runnable
    writes: list[tuple[str, Any]]
    triggers: list[str]
    config: RunnableConfig
    retry_policy: Sequence[RetryPolicy] | None
    cache_policy: CachePolicy | None
```

**Constants**:
- `INPUT`, `OUTPUT`, `INTERRUPT`, `ERROR`, `TASKS` - Reserved channel names
- `NS_SEP`, `NS_END` - Namespace separators for subgraphs

---

### 1.4 `protocol.py`

**Location**: `langgraph/pregel/protocol.py`

**Purpose**: Defines the `PregelProtocol` - a Python Protocol (interface) that any graph implementation must satisfy.

**Key Protocol**: `PregelProtocol`

```python
class PregelProtocol(Protocol):
    """Protocol defining the interface for Pregel-like graph implementations."""

    @property
    def name(self) -> str: ...

    def get_graph(
        self,
        config: RunnableConfig | None = None,
        *,
        xray: int | bool = False,
    ) -> Graph: ...

    def get_state(
        self, config: RunnableConfig, *, subgraphs: bool = False
    ) -> StateSnapshot: ...

    async def aget_state(
        self, config: RunnableConfig, *, subgraphs: bool = False
    ) -> StateSnapshot: ...

    def get_state_history(
        self,
        config: RunnableConfig,
        *,
        filter: dict[str, Any] | None = None,
        before: RunnableConfig | None = None,
        limit: int | None = None,
    ) -> Iterator[StateSnapshot]: ...

    def update_state(
        self,
        config: RunnableConfig,
        values: dict[str, Any] | Any,
        as_node: str | None = None,
    ) -> RunnableConfig: ...

    def stream(
        self,
        input: dict[str, Any] | Any,
        config: RunnableConfig | None = None,
        *,
        stream_mode: StreamMode | list[StreamMode] | None = None,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
        subgraphs: bool = False,
    ) -> Iterator[dict[str, Any] | Any]: ...

    def invoke(
        self,
        input: dict[str, Any] | Any,
        config: RunnableConfig | None = None,
        *,
        interrupt_before: All | Sequence[str] | None = None,
        interrupt_after: All | Sequence[str] | None = None,
    ) -> dict[str, Any] | Any: ...
```

**Implementations**:
- `Pregel` (main.py) - Local graph execution
- `RemoteGraph` (remote.py) - Remote API-based execution

**Stream Protocol**:
```python
class StreamProtocol(Protocol):
    """Protocol for streaming data during execution."""
    modes: set[StreamMode]

    def __call__(self, data: tuple[tuple[str, ...], str, Any]) -> None: ...
```

---

### 1.5 `debug.py`

**Location**: `langgraph/pregel/debug.py`

**Purpose**: Provides debugging utilities for tracing graph execution.

**Key Classes**:

```python
class DebugTask:
    """Debug representation of a task execution."""
    id: str
    name: str
    path: tuple[str, ...]
    payload: dict[str, Any]

class DebugOutputBase:
    """Base class for debug output types."""
    step: int
    type: str
    timestamp: str
    payload: dict[str, Any]

class DebugOutputTask(DebugOutputBase):
    """Debug output for task execution events."""
    type = "task"

class DebugOutputTaskResult(DebugOutputBase):
    """Debug output for task completion events."""
    type = "task_result"

class DebugOutputCheckpoint(DebugOutputBase):
    """Debug output for checkpoint events."""
    type = "checkpoint"
```

**Key Functions**:

```python
def tasks_w_writes(
    tasks: Iterable[PregelExecutableTask],
    pending_writes: list[tuple[str, str, Any]] | None,
    task_states: dict[str, RunnableConfig | StateSnapshot],
    stream_channels: str | Sequence[str],
) -> tuple[PregelTask, ...]:
    """Convert executable tasks to debug-friendly PregelTask format."""

def map_debug_tasks(
    step: int,
    tasks: Iterable[PregelExecutableTask],
) -> Iterator[DebugOutputTask]:
    """Generate debug output for all tasks in a step."""

def map_debug_task_results(
    step: int,
    tasks: Iterable[PregelExecutableTask],
    stream_channels: str | Sequence[str],
) -> Iterator[DebugOutputTaskResult]:
    """Generate debug output for task results."""

def map_debug_checkpoint(
    step: int,
    config: RunnableConfig,
    channels: Mapping[str, BaseChannel],
    stream_channels: str | Sequence[str],
    metadata: CheckpointMetadata,
    checkpoint: Checkpoint,
    tasks: Iterable[PregelExecutableTask],
    pending_writes: list[tuple[str, str, Any]],
) -> DebugOutputCheckpoint:
    """Generate debug output for checkpoint events."""
```

**Usage**:
When `stream_mode="debug"` is used, these functions generate detailed execution traces including:
- Task scheduling and inputs
- Task outputs and writes
- Checkpoint data and channel values
- Timing information

---

## 2. Configuration & Validation Files

### 2.1 `_config.py`

**Location**: `langgraph/pregel/_config.py`

**Purpose**: Handles runtime configuration management for graph execution, including merging configurations, managing configurable parameters, and setting up execution context.

**Key Functions**:

```python
def prepare_next_config(
    config: RunnableConfig,
    checkpoint_ns: str,
    checkpoint_id: str | None,
    *,
    step: int = -1,
    stream_mode: set[StreamMode] = None,
    subgraph: bool = False,
) -> RunnableConfig:
    """Prepare configuration for the next execution step.

    Responsibilities:
    - Copies and patches the base configuration
    - Sets up checkpoint namespace for subgraph isolation
    - Configures stream modes for data output
    - Manages step counter for execution tracking
    """

def patch_configurable(
    config: RunnableConfig,
    patch: dict[str, Any],
) -> RunnableConfig:
    """Patch the configurable section of a config.

    Used to update specific configuration keys while
    preserving the rest of the configuration structure.
    """

def patch_checkpoint_map(
    config: RunnableConfig | None,
    metadata: CheckpointMetadata | None,
) -> RunnableConfig | None:
    """Patch checkpoint_map into config from metadata.

    Ensures checkpoint mapping is properly propagated
    through configuration for state reconstruction.
    """
```

**Configuration Keys Managed**:

| Key | Description |
|-----|-------------|
| `CONFIG_KEY_CHECKPOINT_NS` | Namespace for checkpoint isolation |
| `CONFIG_KEY_CHECKPOINT_ID` | Current checkpoint identifier |
| `CONFIG_KEY_CHECKPOINTER` | Checkpointer instance reference |
| `CONFIG_KEY_STREAM` | Stream callback for output |
| `CONFIG_KEY_SEND` | Send callback for messages |
| `CONFIG_KEY_READ` | Read callback for channel access |
| `CONFIG_KEY_TASK_ID` | Current task identifier |
| `CONFIG_KEY_RESUMING` | Flag indicating resume operation |
| `CONFIG_KEY_CALL` | Task call implementation |

**Interactions**:
- Used by `main.py` to prepare execution configuration
- Called by `_loop.py` to configure each execution step
- Consumed by `_runner.py` for task execution context

---

### 2.2 `_validate.py`

**Location**: `langgraph/pregel/_validate.py`

**Purpose**: Validates graph structure and configuration before execution, ensuring all nodes, channels, and dependencies are properly defined.

**Key Functions**:

```python
def validate_graph(
    nodes: dict[str, PregelNode],
    channels: dict[str, BaseChannel | ManagedValueSpec],
    input_channels: str | Sequence[str],
    output_channels: str | Sequence[str],
    stream_channels: str | Sequence[str] | None,
    interrupt_before_nodes: Sequence[str] | All,
    interrupt_after_nodes: Sequence[str] | All,
) -> None:
    """Validate the complete graph structure.

    Raises:
        ValueError: If validation fails

    Checks performed:
    - All input channels exist
    - All output channels exist
    - All stream channels exist
    - Interrupt nodes exist
    - Node triggers reference valid channels
    - No circular dependencies (optional)
    """

def validate_keys(
    keys: str | Sequence[str],
    channels: dict[str, BaseChannel | ManagedValueSpec],
    which: str,
) -> None:
    """Validate that specified keys exist in channels.

    Args:
        keys: Single key or list of keys to validate
        channels: Available channel definitions
        which: Description for error messages
    """

def validate_nodes_and_triggers(
    nodes: dict[str, PregelNode],
    channels: dict[str, BaseChannel | ManagedValueSpec],
) -> None:
    """Validate node definitions and their channel triggers.

    Ensures each node's triggers reference channels that:
    - Exist in the channel definitions
    - Are not managed values (which can't trigger)
    """
```

**Key Class**:

```python
class PregelNode:
    """Represents a validated node in the Pregel graph.

    Attributes:
        channels: Channels this node subscribes to
        triggers: Channel names that trigger this node
        mapper: Optional function to transform input
        writers: Output writers for this node
        bound: The actual runnable for this node
        metadata: Additional node metadata
        retry_policy: Retry configuration
        defer: Whether to defer execution
        subgraphs: List of subgraph Pregel instances
    """

    def validate(self) -> None:
        """Validate node configuration."""
```

**Validation Flow**:

```
Pregel.__init__()
    |
    v
validate_graph()
    |
    +-> validate_keys(input_channels)
    |
    +-> validate_keys(output_channels)
    |
    +-> validate_keys(stream_channels)
    |
    +-> validate_nodes_and_triggers()
    |
    +-> Check interrupt node existence
    |
    v
Graph ready for execution
```

**Error Handling**:

The validation functions raise descriptive `ValueError` exceptions:

```python
# Example validation errors:
ValueError("Missing input channel: 'messages'")
ValueError("Node 'agent' has trigger 'unknown' that doesn't exist")
ValueError("Interrupt before node 'review' not found in graph")
```

**Interactions**:
- Called by `main.py` during `Pregel.__init__()`
- Ensures graph correctness before any execution
- Prevents runtime errors from misconfiguration

---

## 3. I/O & Communication Files

### 3.1 `_io.py`

**Location**: `langgraph/pregel/_io.py`

**Purpose**: Handles input/output operations for the Pregel execution, including mapping inputs to channels, reading outputs from channels, and managing single/multiple value scenarios.

**Key Functions**:

```python
def map_input(
    input_channels: str | Sequence[str],
    chunk: Any,
) -> Iterator[tuple[str, Any]]:
    """Map input data to channel writes.

    Args:
        input_channels: Target channel(s) for input
        chunk: Input data to map

    Yields:
        (channel_name, value) tuples for each channel

    Behavior:
    - Single channel: wraps value as (channel, value)
    - Multiple channels: expects dict, yields per channel
    - Command input: extracts and yields updates
    """

def map_output_values(
    output_channels: str | Sequence[str],
    pending_writes: list[tuple[str, Any]],
    channels: Mapping[str, BaseChannel],
) -> Iterator[Any]:
    """Extract output values from channels after execution.

    Args:
        output_channels: Channel(s) to read from
        pending_writes: List of pending write operations
        channels: Available channel instances

    Yields:
        Output values from specified channels
    """

def map_output_updates(
    output_channels: str | Sequence[str],
    tasks: Iterable[PregelExecutableTask],
    cached_tasks: Iterable[PregelExecutableTask] | None = None,
) -> Iterator[dict[str, Any]]:
    """Map task writes to update dictionaries.

    Returns incremental updates from task execution,
    useful for streaming intermediate results.
    """

def read_channel(
    channels: Mapping[str, BaseChannel],
    chan: str,
    *,
    catch: bool = True,
    return_exception: bool = False,
) -> Any | None:
    """Read a single value from a channel.

    Args:
        channels: Available channel instances
        chan: Channel name to read
        catch: If True, catch EmptyChannelError
        return_exception: If True, return exception instead of raising

    Returns:
        Channel value or None if empty (when catch=True)
    """

def read_channels(
    channels: Mapping[str, BaseChannel],
    select: str | Sequence[str],
) -> dict[str, Any] | Any:
    """Read values from multiple channels.

    Args:
        channels: Available channel instances
        select: Channel name(s) to read

    Returns:
        - Single channel: direct value
        - Multiple channels: dict of {channel: value}
    """
```

**Input Handling**:

```
User Input                map_input()               Channel Writes
+---------+              +-----------+             +-------------+
| "hello" | --single---> | ("msg",   | --------->  | msg: "hello"|
+---------+              |  "hello") |             +-------------+
                         +-----------+

+-----------+            +-----------+             +-------------+
| {"a": 1,  | --multi--> | ("a", 1)  | --------->  | a: 1        |
|  "b": 2}  |            | ("b", 2)  |             | b: 2        |
+-----------+            +-----------+             +-------------+
```

**Interactions**:
- Called by `_loop.py` to process graph input
- Used by `_algo.py` to read channel values
- Provides data to `main.py` for output generation

---

### 3.2 `_read.py`

**Location**: `langgraph/pregel/_read.py`

**Purpose**: Defines the `PregelNode` class and channel reading mechanisms that allow nodes to access channel data during execution.

**Key Class**: `PregelNode`

```python
@dataclass
class PregelNode:
    """A node in the Pregel graph with its configuration.

    This class represents a single computational unit in the graph,
    including its input/output configuration and execution behavior.
    """

    # Channel configuration
    channels: Mapping[None, str] | Mapping[str, str]
    triggers: Sequence[str]

    # Execution configuration
    mapper: Callable[[Any], Any] | None = None
    writers: Sequence[Runnable] = field(default_factory=list)

    # The actual computation
    bound: Runnable = field(default=None)

    # Metadata and policies
    metadata: Mapping[str, Any] | None = None
    retry_policy: Sequence[RetryPolicy] | None = None
    defer: bool = False
    tags: Sequence[str] | None = None

    # Subgraph support
    subgraphs: list[PregelProtocol] = field(default_factory=list)

    def copy(self, update: dict[str, Any]) -> PregelNode:
        """Create a copy with updated attributes."""

    def get_writers(self) -> list[Runnable]:
        """Get all writer runnables for this node."""

def ChannelRead:
    """Helper for reading from channels within node execution.

    Provides a way for nodes to access channel data:
    - Direct channel access via read callback
    - Mapped input transformation
    - Multi-channel aggregation
    """
```

**Channel Mapping**:

```python
# Single channel (value passed directly)
channels = {None: "messages"}
# Result: node receives messages channel value directly

# Multiple channels (dict passed)
channels = {"msgs": "messages", "ctx": "context"}
# Result: node receives {"msgs": <messages>, "ctx": <context>}

# With mapper
mapper = lambda x: x[-1]  # Get last message
# Result: node receives last message from channel
```

**Trigger Mechanism**:

```python
# Node triggers when any trigger channel updates
triggers = ["messages", "user_input"]

# During execution:
# 1. Channel "messages" updated
# 2. System checks: "messages" in triggers? Yes
# 3. Node scheduled for execution
```

**Interactions**:
- Created by `StateGraph.compile()` for each node
- Used by `_algo.py` to prepare node inputs
- Writers connect to `_write.py` for output handling

---

### 3.3 `_write.py`

**Location**: `langgraph/pregel/_write.py`

**Purpose**: Handles channel write operations, including the `ChannelWrite` class that manages how node outputs are written to channels.

**Key Classes**:

```python
@dataclass
class ChannelWriteEntry:
    """Single channel write specification.

    Attributes:
        channel: Target channel name
        value: Value to write (can be SKIP_WRITE sentinel)
        skip_none: Whether to skip None values
        mapper: Optional transformation before writing
    """
    channel: str
    value: Any = PASSTHROUGH
    skip_none: bool = False
    mapper: Callable[[Any], Any] | None = None


class ChannelWrite(RunnableCallable):
    """Runnable that writes to channels.

    This class is the primary mechanism for nodes to output
    data to channels for consumption by other nodes.
    """

    writes: list[ChannelWriteEntry | Send]
    require_at_least_one_of: Sequence[str] | None

    def __init__(
        self,
        writes: Sequence[ChannelWriteEntry | Send],
        *,
        tags: Sequence[str] | None = None,
        require_at_least_one_of: Sequence[str] | None = None,
    ):
        """Initialize channel writer.

        Args:
            writes: List of write specifications
            tags: Optional tags for tracing
            require_at_least_one_of: Channels that must have at least one write
        """

    def invoke(
        self,
        input: Any,
        config: RunnableConfig,
    ) -> None:
        """Execute writes based on input and configuration.

        Process:
        1. Iterate through write specifications
        2. Apply any mappers to transform values
        3. Filter out SKIP_WRITE and None (if skip_none)
        4. Write values to channels via send callback
        """

    @staticmethod
    def register_writer(writer: ChannelWrite) -> Callable:
        """Register a writer for static analysis.

        Used by graph visualization to determine edges
        without executing the graph.
        """

    @staticmethod
    def get_static_writes(writer: ChannelWrite) -> list[tuple[str, Any, str | None]]:
        """Get statically declared writes for graph analysis."""
```

**Write Flow**:

```
Node Output              ChannelWrite              Channels
+-----------+           +-------------+           +---------+
| {"msg":   | --------> | Process     | --------> | msg:    |
|  "hello"} |           | entries:    |           | "hello" |
+-----------+           | - channel   |           +---------+
                        | - mapper    |
                        | - skip_none |
                        +-------------+
```

**Special Values**:

```python
PASSTHROUGH  # Pass node output directly to channel
SKIP_WRITE   # Skip this write operation
Send(node, value)  # Send value to specific node (dynamic routing)
```

**Send Class for Dynamic Routing**:

```python
class Send:
    """Represents a message to send to a specific node.

    Used for dynamic routing where the target node
    is determined at runtime.
    """
    node: str   # Target node name
    arg: Any    # Value to send

    def __hash__(self) -> int:
        """Make Send hashable for deduplication."""
```

**Interactions**:
- Attached to `PregelNode` as writers
- Invoked by `_runner.py` after node execution
- Writes processed by `_algo.py` to update channels

---

### 3.4 `_messages.py`

**Location**: `langgraph/pregel/_messages.py`

**Purpose**: Provides utilities for handling LangChain message streams during graph execution, enabling real-time streaming of AI-generated content.

**Key Functions**:

```python
def apply_messages_shim(
    messages: Sequence[AnyMessage] | tuple[AnyMessage, ...],
    *,
    tool_calls_key: str = "tool_calls",
) -> tuple[AnyMessage, ...]:
    """Apply message compatibility transformations.

    Handles differences between message formats across
    LangChain versions and providers.

    Args:
        messages: Input messages to transform
        tool_calls_key: Key used for tool call data

    Returns:
        Normalized message tuple
    """

def messages_to_stream_chunks(
    messages: Sequence[AnyMessage] | tuple[AnyMessage, ...],
) -> Iterator[tuple[AnyMessage, MessageChunkMetadata]]:
    """Convert complete messages to streaming chunks.

    Transforms full messages into chunk format suitable
    for streaming output, maintaining message metadata.

    Yields:
        (message_chunk, metadata) tuples
    """

def stream_messages(
    messages: Sequence[AnyMessage] | tuple[AnyMessage, ...],
    stream: StreamProtocol,
    ns: tuple[str, ...],
) -> None:
    """Stream messages through the stream protocol.

    Args:
        messages: Messages to stream
        stream: Stream callback
        ns: Namespace for the stream event

    Behavior:
    - Chunks messages appropriately
    - Applies metadata for client consumption
    - Handles AIMessage streaming specially
    """

@dataclass
class MessageChunkMetadata:
    """Metadata for message chunks in streaming.

    Attributes:
        run_id: Run identifier for tracing
        message_id: Unique message identifier
        is_final: Whether this is the final chunk
        tool_call_id: Optional tool call identifier
    """
    run_id: str | None = None
    message_id: str | None = None
    is_final: bool = False
    tool_call_id: str | None = None
```

**Streaming Flow**:

```
AI Response Generation          Message Streaming          Client
+-------------------+          +-----------------+        +--------+
| AIMessage(        | -------> | Chunk 1: "Hel"  | -----> | Display|
|   content="Hello" |          | Chunk 2: "lo"   |        | to     |
|   tool_calls=[..] |          | Chunk 3: (tool) |        | User   |
| )                 |          +-----------------+        +--------+
+-------------------+
```

**Message Types Handled**:

| Message Type | Streaming Behavior |
|--------------|-------------------|
| `AIMessage` | Chunked content + tool calls |
| `HumanMessage` | Passed through as-is |
| `SystemMessage` | Passed through as-is |
| `ToolMessage` | Passed through with metadata |
| `AIMessageChunk` | Already chunked, forwarded |

**Interactions**:
- Used by `_loop.py` for message streaming
- Integrates with LangChain message types
- Consumed by frontend clients for real-time display

---

## 4. Execution Engine Files

### 4.1 `_algo.py`

**Location**: `langgraph/pregel/_algo.py`

**Purpose**: Contains the core algorithms for the Pregel execution model, including task preparation, write application, and version management.

**Key Data Structures**:

```python
@dataclass
class PregelTaskWrites:
    """Container for task write operations.

    Attributes:
        path: Path in the execution tree
        name: Task/node name
        writes: List of (channel, value) pairs
        triggers: Channels that triggered this task
    """
    path: tuple[str | int, ...]
    name: str
    writes: list[tuple[str, Any]]
    triggers: list[str]
```

**Key Functions**:

```python
def prepare_next_tasks(
    checkpoint: Checkpoint,
    pending_writes: list[tuple[str, str, Any]],
    nodes: dict[str, PregelNode],
    channels: Mapping[str, BaseChannel],
    managed: ManagedValueMapping,
    config: RunnableConfig,
    step: int,
    stop: int,
    *,
    for_execution: bool,
    store: BaseStore | None,
    checkpointer: BaseCheckpointSaver | None,
    manager: ParentRunManager | None,
    trigger_to_nodes: Mapping[str, Sequence[str]] | None = None,
    updated_channels: set[str] | None = None,
) -> dict[str, PregelExecutableTask]:
    """Prepare tasks for the next execution step.

    Algorithm:
    1. Identify channels that were updated
    2. Find nodes triggered by those channels
    3. Prepare input for each triggered node
    4. Create PregelExecutableTask for each

    Args:
        checkpoint: Current checkpoint state
        pending_writes: Writes from previous step
        nodes: Available graph nodes
        channels: Channel instances
        managed: Managed value mappings
        config: Execution configuration
        step: Current step number
        stop: Maximum step number
        for_execution: Whether tasks are for execution vs introspection
        store: Optional key-value store
        checkpointer: Optional checkpointer
        manager: Optional run manager for tracing

    Returns:
        Dict mapping task IDs to executable tasks
    """

def apply_writes(
    checkpoint: Checkpoint,
    channels: Mapping[str, BaseChannel],
    tasks: Iterable[PregelTaskWrites],
    get_next_version: Callable[[int | None, BaseChannel], int] | None,
    trigger_to_nodes: Mapping[str, Sequence[str]],
) -> set[str]:
    """Apply task writes to channels.

    Algorithm:
    1. Group writes by channel
    2. Apply writes to each channel
    3. Update channel versions
    4. Track which channels were updated

    Args:
        checkpoint: Checkpoint to update
        channels: Channel instances
        tasks: Tasks with writes to apply
        get_next_version: Version increment function
        trigger_to_nodes: Mapping of triggers to node names

    Returns:
        Set of channel names that were updated
    """

def increment(
    current_version: int | None,
    channel: BaseChannel,
) -> int:
    """Default version increment function.

    Simple integer increment for channel versioning.
    """
    return (current_version or 0) + 1

def should_interrupt(
    checkpoint: Checkpoint,
    interrupt_nodes: Sequence[str] | All,
    tasks: Iterable[PregelExecutableTask],
) -> list[PregelExecutableTask]:
    """Determine which tasks should trigger an interrupt.

    Used for human-in-the-loop patterns where execution
    pauses at specified nodes.
    """
```

**Task Preparation Flow**:

```
Channel Updates          prepare_next_tasks()         Executable Tasks
+---------------+       +-------------------+        +----------------+
| messages: v2  | ----> | 1. Find triggers  | -----> | Task: agent    |
| context: v3   |       | 2. Check versions |        |   input: {...} |
+---------------+       | 3. Build inputs   |        |   config: {...}|
                        | 4. Create tasks   |        +----------------+
                        +-------------------+
```

**Version Tracking**:

The algorithm uses version tracking to ensure nodes only execute when their triggers have new data:

```python
# checkpoint.versions_seen tracks what each node has seen
versions_seen = {
    "agent": {"messages": 1, "context": 2},
    "tool": {"messages": 1}
}

# checkpoint.channel_versions tracks current versions
channel_versions = {
    "messages": 2,  # Updated!
    "context": 2
}

# Node "agent" triggers because messages version > seen version
# Node "tool" also triggers because messages version > seen version
```

**Interactions**:
- Called by `_loop.py` to prepare each step's tasks
- Uses `_io.py` for channel reading
- Provides tasks to `_runner.py` for execution

---

### 4.2 `_loop.py`

**Location**: `langgraph/pregel/_loop.py`

**Purpose**: Implements the main execution loop for Pregel graphs, coordinating task scheduling, execution, and state management.

**Key Classes**:

```python
class PregelLoop:
    """Synchronous execution loop for Pregel graphs.

    Manages the iterative execution of graph nodes,
    checkpointing, and output streaming.
    """

    def __init__(
        self,
        input: Any,
        *,
        config: RunnableConfig,
        checkpointer: BaseCheckpointSaver | None,
        nodes: dict[str, PregelNode],
        channels: dict[str, BaseChannel],
        managed: ManagedValueMapping,
        specs: dict[str, BaseChannel | ManagedValueSpec],
        output_channels: str | Sequence[str],
        stream_channels: str | Sequence[str],
        interrupt_before: Sequence[str] | All,
        interrupt_after: Sequence[str] | All,
        store: BaseStore | None,
        trigger_to_nodes: Mapping[str, Sequence[str]],
    ):
        """Initialize the execution loop."""

    def __iter__(self) -> Iterator[dict[str, Any]]:
        """Iterate through execution, yielding outputs."""

    def tick(self) -> bool:
        """Execute one step of the graph.

        Returns:
            True if more steps should execute, False if done

        Process:
        1. Check for pending interrupts
        2. Prepare next tasks
        3. Execute tasks
        4. Apply writes
        5. Create checkpoint
        6. Stream outputs
        """


class AsyncPregelLoop:
    """Asynchronous execution loop for Pregel graphs.

    Async variant of PregelLoop for use with asyncio.
    """

    async def __aiter__(self) -> AsyncIterator[dict[str, Any]]:
        """Async iterate through execution."""

    async def atick(self) -> bool:
        """Execute one async step of the graph."""
```

**Loop Lifecycle**:

```
                           PregelLoop
                               |
           +-------------------+-------------------+
           |                   |                   |
           v                   v                   v
    +------------+      +------------+      +------------+
    |  Step 0    |      |  Step 1    |      |  Step N    |
    | - Prepare  |      | - Prepare  |      | - Prepare  |
    | - Execute  | ---> | - Execute  | ---> | - Execute  |
    | - Write    |      | - Write    |      | - Write    |
    | - Checkpoint|     | - Checkpoint|     | - Checkpoint|
    +------------+      +------------+      +------------+
           |                   |                   |
           v                   v                   v
    +------------+      +------------+      +------------+
    | Stream     |      | Stream     |      | Stream     |
    | outputs    |      | outputs    |      | outputs    |
    +------------+      +------------+      +------------+
```

**Step Execution Details**:

```python
def tick(self) -> bool:
    # 1. Check for interrupts before nodes
    if self.step > 0 and should_interrupt(
        self.checkpoint,
        self.interrupt_before,
        self.tasks.values()
    ):
        return False

    # 2. Execute all current tasks
    with BackgroundExecutor(self.config) as submit:
        for task in self.tasks.values():
            submit(run_with_retry, task, self.retry_policy)

    # 3. Apply writes to channels
    updated = apply_writes(
        self.checkpoint,
        self.channels,
        self.tasks.values(),
        self.get_next_version,
        self.trigger_to_nodes,
    )

    # 4. Check for interrupts after nodes
    if should_interrupt(
        self.checkpoint,
        self.interrupt_after,
        self.tasks.values()
    ):
        return False

    # 5. Create checkpoint if checkpointer configured
    if self.checkpointer:
        self.checkpoint = create_checkpoint(...)
        self.checkpointer.put(...)

    # 6. Prepare next step's tasks
    self.tasks = prepare_next_tasks(...)

    # 7. Continue if there are more tasks
    return bool(self.tasks)
```

**Interactions**:
- Created and run by `main.py`
- Uses `_algo.py` for task preparation
- Uses `_runner.py` for task execution
- Uses `_checkpoint.py` for state persistence

---

### 4.3 `_runner.py`

**Location**: `langgraph/pregel/_runner.py`

**Purpose**: Handles the actual execution of individual tasks, including parallel execution management and result collection.

**Key Functions**:

```python
def run_task(
    task: PregelExecutableTask,
    *,
    retry_policy: Sequence[RetryPolicy] | None,
    config: RunnableConfig,
) -> None:
    """Execute a single task synchronously.

    Args:
        task: The task to execute
        retry_policy: Optional retry configuration
        config: Execution configuration

    Process:
    1. Prepare task configuration
    2. Invoke the task's runnable
    3. Collect outputs via writers
    4. Handle any errors or interrupts
    """

async def arun_task(
    task: PregelExecutableTask,
    *,
    retry_policy: Sequence[RetryPolicy] | None,
    config: RunnableConfig,
) -> None:
    """Execute a single task asynchronously."""

def execute_tasks(
    tasks: Iterable[PregelExecutableTask],
    *,
    config: RunnableConfig,
    retry_policy: Sequence[RetryPolicy] | None,
) -> list[PregelTaskWrites]:
    """Execute multiple tasks, potentially in parallel.

    Uses a thread pool executor for parallel execution
    of synchronous tasks.

    Returns:
        List of task writes from all executed tasks
    """

async def aexecute_tasks(
    tasks: Iterable[PregelExecutableTask],
    *,
    config: RunnableConfig,
    retry_policy: Sequence[RetryPolicy] | None,
) -> list[PregelTaskWrites]:
    """Execute multiple tasks asynchronously in parallel.

    Uses asyncio.gather for concurrent execution
    of async tasks.
    """
```

**Task Execution Flow**:

```
PregelExecutableTask               Execution                  Results
+-------------------+          +---------------+         +-------------+
| id: "abc123"      |          |               |         | writes:     |
| name: "agent"     | -------> | task.proc     | ------> |  [(msg, v)] |
| input: {...}      |          | .invoke(...)  |         | triggers:   |
| proc: Runnable    |          |               |         |  [msg]      |
| config: {...}     |          +---------------+         +-------------+
+-------------------+
```

**Parallel Execution Strategy**:

```python
# Sync tasks: Use ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=4) as executor:
    futures = [
        executor.submit(run_task, task, retry_policy=policy)
        for task in tasks
    ]
    results = [f.result() for f in futures]

# Async tasks: Use asyncio.gather
results = await asyncio.gather(*[
    arun_task(task, retry_policy=policy)
    for task in tasks
])
```

**Interactions**:
- Called by `_loop.py` during each tick
- Uses `_retry.py` for retry logic
- Uses `_executor.py` for background execution
- Task outputs processed by `_algo.py`

---

### 4.4 `_executor.py`

**Location**: `langgraph/pregel/_executor.py`

**Purpose**: Provides background execution capabilities for running tasks in separate threads or as async tasks, with proper lifecycle management.

**Key Classes**:

```python
class BackgroundExecutor(AbstractContextManager):
    """Context manager for running sync tasks in the background.

    Uses a thread pool executor to delegate tasks to separate threads.

    Lifecycle:
    - On enter: Initialize executor
    - During: Submit tasks for background execution
    - On exit: Wait for all tasks, re-raise any errors
    """

    def __init__(self, config: RunnableConfig) -> None:
        """Initialize with configuration.

        Args:
            config: Configuration with executor settings
        """
        self.stack = ExitStack()
        self.executor = self.stack.enter_context(
            get_executor_for_config(config)
        )
        self.tasks: dict[Future, tuple[bool, bool]] = {}

    def submit(
        self,
        fn: Callable[P, T],
        *args: P.args,
        __name__: str | None = None,
        __cancel_on_exit__: bool = False,
        __reraise_on_exit__: bool = True,
        __next_tick__: bool = False,
        **kwargs: P.kwargs,
    ) -> Future[T]:
        """Submit a task for background execution.

        Args:
            fn: Function to execute
            *args: Positional arguments
            __cancel_on_exit__: Cancel if not started on exit
            __reraise_on_exit__: Re-raise exceptions on exit
            __next_tick__: Yield control before executing

        Returns:
            Future representing the task
        """

    def __exit__(self, exc_type, exc_value, traceback) -> bool | None:
        """Clean up on exit.

        Process:
        1. Cancel tasks marked for cancellation
        2. Wait for all tasks to complete
        3. Shutdown executor
        4. Re-raise first exception if any
        """


class AsyncBackgroundExecutor(AbstractAsyncContextManager):
    """Async context manager for background async tasks.

    Uses asyncio for concurrent task execution.
    """

    def __init__(self, config: RunnableConfig) -> None:
        """Initialize with configuration."""
        self.tasks: dict[asyncio.Future, tuple[bool, bool]] = {}
        self.loop = asyncio.get_running_loop()
        if max_concurrency := config.get("max_concurrency"):
            self.semaphore = asyncio.Semaphore(max_concurrency)

    async def __aexit__(self, exc_type, exc_value, traceback) -> None:
        """Async cleanup on exit."""
```

**Submit Protocol**:

```python
class Submit(Protocol[P, T]):
    """Protocol for task submission.

    Defines the interface for submitting background tasks
    with lifecycle control flags.
    """

    def __call__(
        self,
        fn: Callable[P, T],
        *args: P.args,
        __name__: str | None = None,
        __cancel_on_exit__: bool = False,
        __reraise_on_exit__: bool = True,
        __next_tick__: bool = False,
        **kwargs: P.kwargs,
    ) -> Future[T]: ...
```

**Usage Pattern**:

```python
# Synchronous usage
with BackgroundExecutor(config) as submit:
    future1 = submit(task1.run, input1)
    future2 = submit(task2.run, input2, __cancel_on_exit__=True)
    # Tasks execute concurrently
# On exit: waits for completion, re-raises errors

# Asynchronous usage
async with AsyncBackgroundExecutor(config) as submit:
    future1 = submit(async_task1.run, input1)
    future2 = submit(async_task2.run, input2)
    # Tasks execute concurrently
# On exit: awaits completion, re-raises errors
```

**Helper Functions**:

```python
async def gated(
    semaphore: asyncio.Semaphore,
    coro: Coroutine[None, None, T],
) -> T:
    """Gate a coroutine with a semaphore.

    Used to limit concurrent async executions.
    """
    async with semaphore:
        return await coro

def next_tick(
    fn: Callable[P, T],
    *args: P.args,
    **kwargs: P.kwargs,
) -> T:
    """Yield control before executing.

    Allows other threads to run before this task starts.
    """
    time.sleep(0)
    return fn(*args, **kwargs)
```

**Interactions**:
- Used by `_loop.py` for parallel task execution
- Consumed by `_runner.py` for task submission
- Coordinates with Python's threading/asyncio modules

---

### 4.5 `_call.py`

**Location**: `langgraph/pregel/_call.py`

**Purpose**: Provides utilities for calling functions as tasks within the Pregel execution context, including creating runnables from user functions and handling async/sync interoperability.

**Key Functions**:

```python
def get_runnable_for_entrypoint(
    func: Callable[..., Any],
) -> Runnable:
    """Convert a function to a Runnable for graph entry.

    Args:
        func: User-provided function

    Returns:
        Runnable that can be used as graph input processor

    Handles:
    - Async functions: wraps directly
    - Sync functions: wraps with executor for async compat
    """

def get_runnable_for_task(
    func: Callable[..., Any],
) -> Runnable:
    """Convert a function to a Runnable for task execution.

    Args:
        func: User-provided function

    Returns:
        RunnableSeq with the function and ChannelWrite

    Creates a runnable sequence that:
    1. Executes the user function
    2. Writes the return value to RETURN channel
    """

def identifier(
    obj: Any,
    name: str | None = None,
) -> str | None:
    """Get the module and name identifier for an object.

    Used for caching and debugging purposes.

    Returns:
        String like "mymodule.my_function" or None
    """

def call(
    func: Callable[P, Awaitable[T]] | Callable[P, T],
    *args: Any,
    retry_policy: Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy | None = None,
    **kwargs: Any,
) -> SyncAsyncFuture[T]:
    """Call a function within the Pregel execution context.

    This is the primary way to invoke sub-tasks from within
    a node's execution.

    Args:
        func: Function to call
        *args: Positional arguments
        retry_policy: Optional retry configuration
        cache_policy: Optional caching configuration
        **kwargs: Keyword arguments

    Returns:
        Future that resolves to the function's result
    """
```

**SyncAsyncFuture**:

```python
class SyncAsyncFuture(Generic[T], Future[T]):
    """A Future that can be awaited in both sync and async contexts.

    Enables seamless interoperability between sync and async code
    within Pregel execution.
    """

    def __await__(self) -> Generator[T, None, T]:
        """Allow awaiting in async context."""
        yield cast(T, ...)
```

**Function Caching**:

```python
# Cache for converted runnables
CACHE: dict[tuple[Callable[..., Any], bool], Runnable] = {}

# Key is (function, is_task)
# - (func, False) -> entrypoint runnable
# - (func, True) -> task runnable with ChannelWrite
```

**Module Resolution**:

```python
def _lookup_module_and_qualname(
    obj: Any,
    name: str | None = None,
) -> tuple[ModuleType, str] | None:
    """Look up an object's module and qualified name.

    Used to determine if a function can be cached/pickled
    and for debugging information.

    Returns:
        (module, qualname) tuple or None if not resolvable
    """

def _whichmodule(obj: Any, name: str) -> str | None:
    """Find the module an object belongs to.

    More robust than pickle.whichmodule, handles edge cases
    like dynamically created modules.
    """
```

**Interactions**:
- Used by `main.py` for creating node runnables
- Provides `call()` for sub-task invocation
- Integrates with `_retry.py` for retry policies
- Connects with caching system for performance

---

## 5. Support & Utility Files

### 5.1 `_checkpoint.py`

**Location**: `langgraph/pregel/_checkpoint.py`

**Purpose**: Provides utilities for creating, managing, and restoring checkpoints that enable state persistence and time-travel debugging.

**Constants**:

```python
LATEST_VERSION = 4  # Current checkpoint format version
```

**Key Functions**:

```python
def empty_checkpoint() -> Checkpoint:
    """Create an empty checkpoint for initialization.

    Returns:
        Checkpoint with default values:
        - v: LATEST_VERSION
        - id: UUID6-based unique ID
        - ts: Current timestamp in ISO format
        - channel_values: Empty dict
        - channel_versions: Empty dict
        - versions_seen: Empty dict
    """

def create_checkpoint(
    checkpoint: Checkpoint,
    channels: Mapping[str, BaseChannel] | None,
    step: int,
    *,
    id: str | None = None,
    updated_channels: set[str] | None = None,
) -> Checkpoint:
    """Create a new checkpoint from current state.

    Args:
        checkpoint: Previous checkpoint to base on
        channels: Current channel instances
        step: Current execution step
        id: Optional specific checkpoint ID
        updated_channels: Channels that were updated

    Returns:
        New Checkpoint with:
        - Updated timestamp
        - New ID based on step
        - Current channel values
        - Updated version information
    """

def channels_from_checkpoint(
    specs: Mapping[str, BaseChannel | ManagedValueSpec],
    checkpoint: Checkpoint,
) -> tuple[Mapping[str, BaseChannel], ManagedValueMapping]:
    """Restore channels from a checkpoint.

    Args:
        specs: Channel/managed value specifications
        checkpoint: Checkpoint to restore from

    Returns:
        Tuple of (channels, managed_values)

    Process:
    1. Separate channel specs from managed value specs
    2. Restore each channel from checkpoint values
    3. Return both mappings
    """

def copy_checkpoint(checkpoint: Checkpoint) -> Checkpoint:
    """Create a deep copy of a checkpoint.

    Used when modifying checkpoint state to avoid
    affecting the original checkpoint.
    """
```

**Checkpoint Structure**:

```python
class Checkpoint(TypedDict):
    """Checkpoint data structure for state persistence.

    Attributes:
        v: Version number for format compatibility
        id: Unique identifier (UUID6-based)
        ts: ISO timestamp of creation
        channel_values: Serialized channel data
        channel_versions: Version numbers per channel
        versions_seen: Node -> channel versions mapping
        updated_channels: Channels modified in this step
    """
    v: int
    id: str
    ts: str
    channel_values: dict[str, Any]
    channel_versions: dict[str, int]
    versions_seen: dict[str, dict[str, int]]
    updated_channels: list[str] | None
```

**Checkpoint Lifecycle**:

```
                    Execution Flow
                         |
    +--------------------+--------------------+
    |                    |                    |
    v                    v                    v
+-----------+      +-----------+        +-----------+
| Checkpoint|      | Checkpoint|        | Checkpoint|
| Step 0    | ---> | Step 1    | ---->  | Step N    |
| id: abc   |      | id: def   |        | id: xyz   |
+-----------+      +-----------+        +-----------+
    |                    |                    |
    v                    v                    v
+-----------+      +-----------+        +-----------+
| channel_  |      | channel_  |        | channel_  |
| values    |      | values    |        | values    |
+-----------+      +-----------+        +-----------+
```

**Interactions**:
- Used by `_loop.py` for state persistence
- Consumed by `main.py` for state retrieval
- Works with `BaseCheckpointSaver` implementations

---

### 5.2 `_retry.py`

**Location**: `langgraph/pregel/_retry.py`

**Purpose**: Implements retry logic for task execution with configurable backoff strategies, allowing graceful handling of transient failures.

**Key Functions**:

```python
def run_with_retry(
    task: PregelExecutableTask,
    retry_policy: Sequence[RetryPolicy] | None,
    configurable: dict[str, Any] | None = None,
) -> None:
    """Run a task with retry support (synchronous).

    Args:
        task: Task to execute
        retry_policy: Sequence of retry policies to apply
        configurable: Additional configuration

    Behavior:
    1. Execute task
    2. On failure, find matching retry policy
    3. Calculate backoff with optional jitter
    4. Sleep and retry up to max_attempts
    5. Raise if all retries exhausted
    """

async def arun_with_retry(
    task: PregelExecutableTask,
    retry_policy: Sequence[RetryPolicy] | None,
    stream: bool = False,
    match_cached_writes: Callable[[], Awaitable[Sequence[PregelExecutableTask]]] | None = None,
    configurable: dict[str, Any] | None = None,
) -> None:
    """Run a task with retry support (asynchronous).

    Additional parameters:
        stream: Whether to use streaming execution
        match_cached_writes: Callback to check cache hits
    """

def _should_retry_on(
    retry_policy: RetryPolicy,
    exc: Exception,
) -> bool:
    """Check if exception matches retry policy.

    Handles three forms of retry_on:
    - Single exception class
    - Sequence of exception classes
    - Callable predicate function
    """
```

**RetryPolicy Structure**:

```python
@dataclass
class RetryPolicy:
    """Configuration for retry behavior.

    Attributes:
        max_attempts: Maximum number of attempts (including first)
        initial_interval: Base wait time in seconds
        max_interval: Maximum wait time in seconds
        backoff_factor: Multiplier for each retry
        jitter: Whether to add random jitter
        retry_on: Exception types or predicate to retry on
    """
    max_attempts: int = 3
    initial_interval: float = 0.5
    max_interval: float = 60.0
    backoff_factor: float = 2.0
    jitter: bool = True
    retry_on: type[Exception] | Sequence[type[Exception]] | Callable[[Exception], bool] = Exception
```

**Backoff Calculation**:

```python
# Exponential backoff with cap
interval = initial_interval * (backoff_factor ** (attempt - 1))
interval = min(max_interval, interval)

# Add jitter if configured
if jitter:
    interval += random.uniform(0, 1)
```

**Retry Flow**:

```
                Execute Task
                     |
                     v
              +------+------+
              | Success?    |
              +------+------+
                |         |
               Yes        No
                |         |
                v         v
            Return   +--------+
                     | Match  |
                     | Policy?|
                     +--------+
                       |    |
                      Yes   No
                       |    |
                       v    v
                   +------+ Raise
                   |Sleep |
                   |Retry |
                   +------+
                       |
                       v
                   Increment
                   Attempts
                       |
                       v
              +--------+--------+
              | < max_attempts? |
              +--------+--------+
                  |          |
                 Yes         No
                  |          |
                  v          v
             Execute      Raise
               Task
```

**Interactions**:
- Called by `_runner.py` for task execution
- Uses `RetryPolicy` from types module
- Logs retry attempts via `_log.py`

---

### 5.3 `_utils.py`

**Location**: `langgraph/pregel/_utils.py`

**Purpose**: Provides miscellaneous utility functions for version management, subgraph discovery, and AST-based code analysis.

**Key Functions**:

```python
def get_new_channel_versions(
    previous_versions: ChannelVersions,
    current_versions: ChannelVersions,
) -> ChannelVersions:
    """Get subset of current versions that are newer.

    Args:
        previous_versions: Versions from previous checkpoint
        current_versions: Current channel versions

    Returns:
        Dict of channels that have been updated

    Used to determine which channels changed
    for checkpoint metadata.
    """

def find_subgraph_pregel(
    candidate: Runnable,
) -> PregelProtocol | None:
    """Find a Pregel subgraph within a runnable.

    Recursively searches through:
    - RunnableSequence chains
    - RunnableLambda dependencies
    - RunnableCallable closures

    Returns:
        PregelProtocol instance or None

    Used for subgraph discovery during compilation.
    """

def get_function_nonlocals(
    func: Callable,
) -> list[Any]:
    """Get nonlocal variables accessed by a function.

    Uses AST analysis to find closure variables
    that a function references from outer scopes.

    Returns:
        List of nonlocal values
    """

def is_xxh3_128_hexdigest(
    value: str,
) -> bool:
    """Check if string is a valid XXH3-128 hex digest.

    Used for validating cache keys and checksums.

    Returns:
        True if 32-character hex string
    """
```

**AST Visitors**:

```python
class FunctionNonLocals(ast.NodeVisitor):
    """AST visitor to find function nonlocals.

    Visits:
    - FunctionDef
    - AsyncFunctionDef
    - Lambda

    Collects names that are loaded but not stored
    (i.e., come from outer scope).
    """
    nonlocals: set[str]

class NonLocals(ast.NodeVisitor):
    """AST visitor for name analysis.

    Tracks:
    - loads: Names that are read
    - stores: Names that are written

    Nonlocals = loads - stores
    """
    loads: set[str]
    stores: set[str]

    def visit_Name(self, node: ast.Name) -> Any:
        """Track name loads and stores."""

    def visit_Attribute(self, node: ast.Attribute) -> Any:
        """Track attribute access patterns."""
```

**Subgraph Discovery Flow**:

```
                Runnable
                    |
         +----------+----------+
         |          |          |
         v          v          v
    RunnableSeq  Lambda   RunnableCallable
         |          |          |
         v          v          v
    [steps]     [deps]      [func/afunc]
         |          |          |
         +--> check each for PregelProtocol
```

**Interactions**:
- Used by `main.py` for subgraph discovery
- Used by `_checkpoint.py` for version comparison
- Used by `_draw.py` for graph analysis

---

### 5.4 `_log.py`

**Location**: `langgraph/pregel/_log.py`

**Purpose**: Provides a centralized logger for the Pregel library.

**Content**:

```python
import logging

logger = logging.getLogger("langgraph")
```

**Usage**:

```python
from langgraph.pregel._log import logger

# Log retry attempts
logger.info(
    f"Retrying task {task.name} after {sleep_time:.2f}s "
    f"(attempt {attempts}) after {exc.__class__.__name__} {exc}",
    exc_info=exc,
)

# Log warnings
logger.warning("No events received from remote graph")

# Log debug information
logger.debug(f"Preparing {len(tasks)} tasks for step {step}")
```

**Log Levels Used**:

| Level | Usage |
|-------|-------|
| `DEBUG` | Detailed execution tracing |
| `INFO` | Retry attempts, state changes |
| `WARNING` | Unexpected but recoverable situations |
| `ERROR` | Errors that prevent normal operation |

**Interactions**:
- Used throughout the pregel package
- Integrates with Python's logging infrastructure
- Can be configured by application using LangGraph

---

## 6. Visualization & Remote Files

### 6.1 `_draw.py`

**Location**: `langgraph/pregel/_draw.py`

**Purpose**: Generates visual graph representations by simulating execution to discover edges and node relationships.

**Key Data Structures**:

```python
class Edge(NamedTuple):
    """Represents an edge in the graph visualization.

    Attributes:
        source: Source node name
        target: Target node name
        conditional: Whether edge is conditional
        data: Optional edge label
    """
    source: str
    target: str
    conditional: bool
    data: str | None

class TriggerEdge(NamedTuple):
    """Represents a trigger relationship.

    Attributes:
        source: Channel/node that triggers
        conditional: Whether conditional
        data: Optional label
    """
    source: str
    conditional: bool
    data: str | None
```

**Key Functions**:

```python
def draw_graph(
    config: RunnableConfig,
    *,
    nodes: dict[str, PregelNode],
    specs: dict[str, BaseChannel | ManagedValueSpec],
    input_channels: str | Sequence[str],
    interrupt_after_nodes: All | Sequence[str],
    interrupt_before_nodes: All | Sequence[str],
    trigger_to_nodes: Mapping[str, Sequence[str]],
    checkpointer: Checkpointer,
    subgraphs: dict[str, Graph],
    limit: int = 250,
) -> Graph:
    """Generate a drawable graph representation.

    Algorithm:
    1. Initialize empty checkpoint and channels
    2. Apply input writes to trigger first nodes
    3. Simulate execution loop:
       - Run task writers (with empty input)
       - Collect static writes for edge discovery
       - Apply writes and prepare next tasks
       - Record edges between nodes
    4. Add discovered edges to graph
    5. Replace subgraph nodes with expanded subgraphs

    Args:
        config: Execution configuration
        nodes: Graph node definitions
        specs: Channel specifications
        input_channels: Input channel names
        interrupt_*_nodes: Interrupt configuration
        trigger_to_nodes: Trigger mapping
        checkpointer: Checkpointer for versioning
        subgraphs: Nested subgraph representations
        limit: Maximum simulation steps

    Returns:
        Graph object suitable for visualization
    """

def add_edge(
    graph: Graph,
    source: str,
    target: str,
    *,
    data: Any | None = None,
    conditional: bool = False,
) -> None:
    """Add an edge to the graph if not exists.

    Avoids duplicate edges and auto-creates
    END node if needed.
    """
```

**Graph Discovery Process**:

```
             draw_graph()
                  |
                  v
    +---------------------------+
    | 1. Create empty state     |
    |    (channels, checkpoint) |
    +---------------------------+
                  |
                  v
    +---------------------------+
    | 2. Apply input writes     |
    |    (trigger initial nodes)|
    +---------------------------+
                  |
                  v
    +---------------------------+
    | 3. Simulation loop:       |
    |    - Execute writers      |<----+
    |    - Collect edges        |     |
    |    - Apply writes         |     |
    |    - Prepare next tasks   |-----+
    +---------------------------+
                  |
                  v
    +---------------------------+
    | 4. Build Graph object     |
    |    - Add nodes            |
    |    - Add discovered edges |
    |    - Expand subgraphs     |
    +---------------------------+
                  |
                  v
             Return Graph
```

**Edge Discovery**:

The function discovers edges by:
1. Running writers with static analysis mode
2. Collecting `ChannelWrite.get_static_writes()` results
3. Tracking which channels trigger which nodes
4. Building edge relationships from writes → channels → nodes

**Interactions**:
- Called by `main.py` in `get_graph()` method
- Uses `_algo.py` for task preparation simulation
- Uses `_checkpoint.py` for empty state initialization
- Returns `Graph` from LangChain Core

---

### 6.2 `remote.py`

**Location**: `langgraph/pregel/remote.py`

**Purpose**: Provides `RemoteGraph` class for interacting with LangGraph graphs deployed as remote APIs.

**Key Classes**:

```python
class RemoteException(Exception):
    """Exception raised when an error occurs in the remote graph."""
    pass

class RemoteGraph(PregelProtocol):
    """Client for calling remote LangGraph APIs.

    Implements PregelProtocol to behave like a local graph
    while actually making HTTP requests to a remote server.
    """

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
    ):
        """Initialize remote graph client.

        Args:
            assistant_id: Graph/assistant ID on the server
            url: Base URL of the remote API
            api_key: API key for authentication
            headers: Additional HTTP headers
            client: Pre-configured async client
            sync_client: Pre-configured sync client
            config: Default configuration
            name: Human-readable name
            distributed_tracing: Enable distributed tracing
        """
```

**Key Methods**:

```python
# Graph introspection
def get_graph(
    self,
    config: RunnableConfig | None = None,
    *,
    xray: int | bool = False,
) -> DrawableGraph:
    """Get graph representation from remote API.

    Calls: GET /assistants/{assistant_id}/graph
    """

# State management
def get_state(
    self,
    config: RunnableConfig,
    *,
    subgraphs: bool = False,
) -> StateSnapshot:
    """Get current state of a remote thread.

    Calls: GET /threads/{thread_id}/state
    or POST /threads/{thread_id}/state/checkpoint
    """

def update_state(
    self,
    config: RunnableConfig,
    values: dict[str, Any] | Any | None,
    as_node: str | None = None,
) -> RunnableConfig:
    """Update state of a remote thread.

    Calls: POST /threads/{thread_id}/state
    """

def get_state_history(
    self,
    config: RunnableConfig,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[StateSnapshot]:
    """Get state history of a remote thread.

    Calls: POST /threads/{thread_id}/history
    """

# Execution
def invoke(
    self,
    input: dict[str, Any] | Any,
    config: RunnableConfig | None = None,
    *,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
) -> dict[str, Any] | Any:
    """Execute graph and return final result."""

def stream(
    self,
    input: dict[str, Any] | Any,
    config: RunnableConfig | None = None,
    *,
    stream_mode: StreamMode | list[StreamMode] | None = None,
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    subgraphs: bool = False,
) -> Iterator[dict[str, Any] | Any]:
    """Stream execution results from remote graph.

    Calls: POST /threads/{thread_id}/runs/stream
    or POST /runs/stream
    """
```

**Remote Execution Flow**:

```
                Client (RemoteGraph)
                        |
                        v
            +------------------------+
            | Sanitize configuration |
            | - Remove non-serializable|
            | - Prepare thread_id    |
            +------------------------+
                        |
                        v
            +------------------------+
            |    HTTP Request        |
            | POST /runs/stream      |
            +------------------------+
                        |
                        v
                 [Remote Server]
                        |
                        v
            +------------------------+
            |  SSE Event Stream      |
            | - mode: updates/values |
            | - data: {...}          |
            +------------------------+
                        |
                        v
            +------------------------+
            | Process stream chunks  |
            | - Handle interrupts    |
            | - Raise exceptions     |
            | - Yield to caller      |
            +------------------------+
```

**Stream Mode Handling**:

```python
def _get_stream_modes(
    self,
    stream_mode: StreamMode | list[StreamMode] | None,
    config: RunnableConfig | None,
    default: StreamMode = "updates",
) -> tuple[list[StreamModeSDK], list[StreamModeSDK], bool, StreamProtocol | None]:
    """Prepare stream modes for remote request.

    Process:
    1. Coerce to list or use default
    2. Add modes from parent graph (if subgraph)
    3. Map 'messages' to 'messages-tuple'
    4. Ensure 'updates' is included (for interrupt detection)
    5. Remove unsupported modes ('events')

    Returns:
        (final_modes, requested_modes, single_mode, parent_stream)
    """
```

**Interrupt and Error Handling**:

```python
# In stream():
for chunk in sync_client.runs.stream(...):
    # Handle commands to parent
    if mode == "command" and chunk.data.get("graph") == Command.PARENT:
        raise ParentCommand(Command(**chunk.data))

    # Handle interrupts
    if chunk.event.startswith("updates"):
        if isinstance(chunk.data, dict) and INTERRUPT in chunk.data:
            if caller_ns:
                raise GraphInterrupt([Interrupt(**i) for i in chunk.data[INTERRUPT]])

    # Handle errors
    elif chunk.event.startswith("error"):
        raise RemoteException(chunk.data)
```

**Interactions**:
- Implements `PregelProtocol` from `protocol.py`
- Uses `langgraph_sdk` for HTTP communication
- Can be used as a subgraph in local Pregel graphs
- Integrates with LangSmith for distributed tracing

---

## File Interaction Diagram

The following diagram shows how all 22 files in the Pregel library interact:

```
                            ┌─────────────────┐
                            │   __init__.py   │
                            │  (Public API)   │
                            └────────┬────────┘
                                     │
                  ┌──────────────────┼──────────────────┐
                  │                  │                  │
                  ▼                  ▼                  ▼
         ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
         │  main.py    │    │ protocol.py │    │  types.py   │
         │  (Pregel)   │◄───│(PregelProto)│    │ (Types)     │
         └──────┬──────┘    └─────────────┘    └─────────────┘
                │                  ▲
                │                  │
    ┌───────────┼───────────┬──────┴──────┐
    │           │           │             │
    ▼           ▼           ▼             ▼
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌───────────┐
│_validate│ │ _config │ │ _draw.py│ │ remote.py │
│  .py    │ │  .py    │ │ (Visual)│ │(RemoteGr) │
└─────────┘ └─────────┘ └────┬────┘ └───────────┘
                             │
                             ▼
                      ┌─────────────┐
                      │  _loop.py   │
                      │ (Exec Loop) │
                      └──────┬──────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
    ┌─────────┐        ┌─────────┐        ┌─────────┐
    │ _algo.py│        │_runner. │        │_executor│
    │ (Core   │◄──────▶│  py     │◄──────▶│  .py    │
    │ Algo)   │        │ (Tasks) │        │ (Exec)  │
    └────┬────┘        └────┬────┘        └─────────┘
         │                  │
         │                  ▼
         │            ┌─────────┐
         │            │_retry.py│
         │            │ (Retry) │
         │            └─────────┘
         │
         ├──────────────────┬──────────────────┐
         │                  │                  │
         ▼                  ▼                  ▼
    ┌─────────┐        ┌─────────┐        ┌─────────┐
    │ _io.py  │        │_read.py │        │_write.py│
    │ (I/O)   │◄──────▶│ (Read)  │◄──────▶│ (Write) │
    └─────────┘        └─────────┘        └─────────┘
                                               │
                                               ▼
                                          ┌─────────┐
                                          │_messages│
                                          │  .py    │
                                          └─────────┘

    ┌─────────────────────────────────────────────────────┐
    │                  Support Layer                       │
    ├─────────────┬─────────────┬────────────┬────────────┤
    │ _checkpoint │   _call.py  │  _utils.py │  _log.py   │
    │    .py      │   (Calls)   │  (Utils)   │  (Logger)  │
    └─────────────┴─────────────┴────────────┴────────────┘

    ┌─────────────────────────────────────────────────────┐
    │                   debug.py                          │
    │             (Debugging Utilities)                   │
    └─────────────────────────────────────────────────────┘
```

### File Dependency Summary

| File | Imports From |
|------|--------------|
| `__init__.py` | main, protocol, types |
| `main.py` | All internal modules |
| `protocol.py` | types |
| `types.py` | (external only) |
| `debug.py` | types, _io |
| `_config.py` | (internal constants) |
| `_validate.py` | _read, types |
| `_io.py` | types |
| `_read.py` | _write, types |
| `_write.py` | types |
| `_messages.py` | protocol |
| `_algo.py` | _io, _read, types, _checkpoint |
| `_loop.py` | _algo, _runner, _checkpoint, _io |
| `_runner.py` | _retry, types |
| `_executor.py` | (external only) |
| `_call.py` | _write, types |
| `_checkpoint.py` | (external only) |
| `_retry.py` | types, _log |
| `_utils.py` | _read, protocol |
| `_log.py` | (external only) |
| `_draw.py` | _algo, _io, _read, _write, _checkpoint |
| `remote.py` | protocol, types |

---

## Execution Flow

### Complete Execution Lifecycle

```
User Call                                         Return Value
    │                                                  ▲
    ▼                                                  │
┌───────────────────────────────────────────────────────┐
│                    main.py: Pregel                    │
├───────────────────────────────────────────────────────┤
│  1. Validate inputs (via _validate.py)                │
│  2. Merge configuration (via _config.py)              │
│  3. Load checkpoint if exists (via checkpointer)      │
│  4. Initialize channels (_checkpoint.channels_from_)  │
└───────────────────────────────────────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────────────┐
│                _loop.py: PregelLoop                   │
├───────────────────────────────────────────────────────┤
│  Loop until no more tasks or interrupt:               │
│  ┌─────────────────────────────────────────────────┐  │
│  │ STEP N:                                         │  │
│  │  a. Check interrupt_before (via _algo.py)       │  │
│  │  b. Execute tasks (via _runner.py)              │  │
│  │  c. Apply writes (via _algo.py)                 │  │
│  │  d. Check interrupt_after (via _algo.py)        │  │
│  │  e. Create checkpoint (via _checkpoint.py)      │  │
│  │  f. Stream outputs (via _io.py)                 │  │
│  │  g. Prepare next tasks (via _algo.py)           │  │
│  └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────────────┐
│              _runner.py: Task Execution               │
├───────────────────────────────────────────────────────┤
│  For each task (potentially parallel):                │
│  ┌─────────────────────────────────────────────────┐  │
│  │  1. Prepare task config                         │  │
│  │  2. Read channels (_read.py)                    │  │
│  │  3. Execute node runnable                       │  │
│  │  4. Run writers (_write.py)                     │  │
│  │  5. Handle retries if needed (_retry.py)        │  │
│  └─────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────────────┐
│            _algo.py: Write Application                │
├───────────────────────────────────────────────────────┤
│  1. Collect writes from all tasks                     │
│  2. Group by channel                                  │
│  3. Apply to channels (update values)                 │
│  4. Update channel versions                           │
│  5. Return set of updated channels                    │
└───────────────────────────────────────────────────────┘
                         │
                         ▼
┌───────────────────────────────────────────────────────┐
│          _checkpoint.py: State Persistence            │
├───────────────────────────────────────────────────────┤
│  1. Serialize channel values                          │
│  2. Create new checkpoint with updated versions       │
│  3. Store via checkpointer implementation             │
│  4. Return new checkpoint ID                          │
└───────────────────────────────────────────────────────┘
```

### Stream Mode Data Flow

```
                    stream() called
                          │
                          ▼
            ┌─────────────────────────────┐
            │ Determine stream modes:     │
            │ - "values": full state      │
            │ - "updates": incremental    │
            │ - "debug": execution trace  │
            │ - "messages": AI messages   │
            │ - "custom": user-defined    │
            └─────────────────────────────┘
                          │
              ┌───────────┴───────────┐
              │                       │
              ▼                       ▼
    ┌─────────────────┐     ┌─────────────────┐
    │ "values" mode   │     │ "updates" mode  │
    │                 │     │                 │
    │ After each step:│     │ After each step:│
    │ yield full      │     │ yield node      │
    │ channel values  │     │ outputs only    │
    └─────────────────┘     └─────────────────┘
              │                       │
              └───────────┬───────────┘
                          │
                          ▼
            ┌─────────────────────────────┐
            │    Stream to callback       │
            │    or yield to iterator     │
            └─────────────────────────────┘
```

---

## Summary

The Pregel library is a sophisticated graph execution engine with these key characteristics:

1. **Modular Architecture**: 22 files organized into logical layers (core, I/O, execution, support)

2. **Graph-Based Model**: Nodes connected via channels with trigger-based execution

3. **Checkpointing**: Full state persistence for fault tolerance and debugging

4. **Flexible Execution**: Support for sync/async, streaming, and parallel execution

5. **Extensibility**: Protocol-based design allows local and remote implementations

6. **Robustness**: Built-in retry logic, interrupt handling, and error management

### Quick Reference

| Need | File |
|------|------|
| Create a graph | `main.py` (Pregel class) |
| Define node behavior | `_read.py` (PregelNode) |
| Write to channels | `_write.py` (ChannelWrite) |
| Understand execution | `_loop.py`, `_algo.py` |
| Add retry logic | `_retry.py` |
| Visualize graph | `_draw.py` |
| Remote execution | `remote.py` |
| Debug execution | `debug.py` |

