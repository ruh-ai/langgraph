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

