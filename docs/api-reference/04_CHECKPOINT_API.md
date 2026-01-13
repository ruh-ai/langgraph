# LangGraph Checkpoint API Reference

This document provides a comprehensive API reference for the LangGraph checkpoint module, which enables persistence and memory for stateful LangGraph agents.

## Table of Contents

- [Core Types](#core-types)
- [Base Classes](#base-classes)
- [Checkpoint Savers](#checkpoint-savers)
- [Serialization](#serialization)
- [Storage](#storage)
- [Caching](#caching)
- [Helper Functions](#helper-functions)

---

## Core Types

### Checkpoint

**Type:** `TypedDict`

State snapshot at a given point in time.

**Fields:**

| Name | Type | Description |
|------|------|-------------|
| v | int | The version of the checkpoint format. Currently `1`. |
| id | str | The ID of the checkpoint. This is both unique and monotonically increasing, so can be used for sorting checkpoints from first to last. |
| ts | str | The timestamp of the checkpoint in ISO 8601 format. |
| channel_values | dict[str, Any] | The values of the channels at the time of the checkpoint. Mapping from channel name to deserialized channel snapshot value. |
| channel_versions | ChannelVersions | The versions of the channels at the time of the checkpoint. The keys are channel names and the values are monotonically increasing version strings for each channel. |
| versions_seen | dict[str, ChannelVersions] | Map from node ID to map from channel name to version seen. This keeps track of the versions of the channels that each node has seen. Used to determine which nodes to execute next. |
| updated_channels | list[str] \| None | The channels that were updated in this checkpoint. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:59`

---

### CheckpointMetadata

**Type:** `TypedDict` (total=False)

Metadata associated with a checkpoint.

**Fields:**

| Name | Type | Description |
|------|------|-------------|
| source | Literal["input", "loop", "update", "fork"] | The source of the checkpoint. `"input"`: created from an input to invoke/stream/batch. `"loop"`: created from inside the pregel loop. `"update"`: created from a manual state update. `"fork"`: created as a copy of another checkpoint. |
| step | int | The step number of the checkpoint. `-1` for the first `"input"` checkpoint, `0` for the first `"loop"` checkpoint, etc. |
| parents | dict[str, str] | The IDs of the parent checkpoints. Mapping from checkpoint namespace to checkpoint ID. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:31`

---

### CheckpointTuple

**Type:** `NamedTuple`

A tuple containing a checkpoint and its associated data.

**Fields:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| config | RunnableConfig | (required) | Configuration for the checkpoint. |
| checkpoint | Checkpoint | (required) | The checkpoint data. |
| metadata | CheckpointMetadata | (required) | Metadata for the checkpoint. |
| parent_config | RunnableConfig \| None | None | Parent configuration if any. |
| pending_writes | list[PendingWrite] \| None | None | List of pending writes. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:106`

---

### ChannelVersions

**Type:** `dict[str, str | int | float]`

Dictionary mapping channel names to their version identifiers.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:56`

---

### PendingWrite

**Type:** `tuple[str, str, Any]`

Represents a pending write operation (task_id, channel, value).

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:27`

---

## Base Classes

### BaseCheckpointSaver

**Signature:**
```python
class BaseCheckpointSaver(Generic[V]):
    def __init__(
        self,
        *,
        serde: SerializerProtocol | None = None,
    ) -> None:
```

**Description:**

Base class for creating a graph checkpointer. Checkpointers allow LangGraph agents to persist their state within and across multiple interactions.

**Constructor Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| serde | SerializerProtocol \| None | No | None | Serializer for encoding/decoding checkpoints. Defaults to `JsonPlusSerializer()`. |

**Attributes:**

- `serde` (SerializerProtocol): Serializer for encoding/decoding checkpoints.

**Methods:**

#### config_specs

**Signature:** `@property def config_specs(self) -> list`

**Description:** Define the configuration options for the checkpoint saver.

**Returns:** List of configuration field specs.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:139`

---

#### get

**Signature:** `def get(self, config: RunnableConfig) -> Checkpoint | None`

**Description:** Fetch a checkpoint using the given configuration.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration specifying which checkpoint to retrieve. |

**Returns:** The requested checkpoint, or `None` if not found.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:148`

---

#### get_tuple

**Signature:** `def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None`

**Description:** Fetch a checkpoint tuple using the given configuration.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration specifying which checkpoint to retrieve. |

**Returns:** The requested checkpoint tuple, or `None` if not found.

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:160`

---

#### list

**Signature:**
```python
def list(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[CheckpointTuple]
```

**Description:** List checkpoints that match the given criteria.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig \| None | Base configuration for filtering checkpoints. |
| filter | dict[str, Any] \| None | Additional filtering criteria for metadata. |
| before | RunnableConfig \| None | List checkpoints created before this configuration. |
| limit | int \| None | Maximum number of checkpoints to return. |

**Returns:** Iterator of matching checkpoint tuples.

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:174`

---

#### put

**Signature:**
```python
def put(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig
```

**Description:** Store a checkpoint with its configuration and metadata.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration for the checkpoint. |
| checkpoint | Checkpoint | The checkpoint to store. |
| metadata | CheckpointMetadata | Additional metadata for the checkpoint. |
| new_versions | ChannelVersions | New channel versions as of this write. |

**Returns:** Updated configuration after storing the checkpoint.

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:198`

---

#### put_writes

**Signature:**
```python
def put_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None
```

**Description:** Store intermediate writes linked to a checkpoint.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration of the related checkpoint. |
| writes | Sequence[tuple[str, Any]] | List of writes to store, each as (channel, value) pair. |
| task_id | str | Identifier for the task creating the writes. |
| task_path | str | Path of the task creating the writes. Defaults to "". |

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:221`

---

#### delete_thread

**Signature:** `def delete_thread(self, thread_id: str) -> None`

**Description:** Delete all checkpoints and writes associated with a specific thread ID.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| thread_id | str | The thread ID whose checkpoints should be deleted. |

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:241`

---

#### aget

**Signature:** `async def aget(self, config: RunnableConfig) -> Checkpoint | None`

**Description:** Asynchronously fetch a checkpoint using the given configuration.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration specifying which checkpoint to retrieve. |

**Returns:** The requested checkpoint, or `None` if not found.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:252`

---

#### aget_tuple

**Signature:** `async def aget_tuple(self, config: RunnableConfig) -> CheckpointTuple | None`

**Description:** Asynchronously fetch a checkpoint tuple using the given configuration.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration specifying which checkpoint to retrieve. |

**Returns:** The requested checkpoint tuple, or `None` if not found.

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:264`

---

#### alist

**Signature:**
```python
async def alist(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> AsyncIterator[CheckpointTuple]
```

**Description:** Asynchronously list checkpoints that match the given criteria.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig \| None | Base configuration for filtering checkpoints. |
| filter | dict[str, Any] \| None | Additional filtering criteria for metadata. |
| before | RunnableConfig \| None | List checkpoints created before this configuration. |
| limit | int \| None | Maximum number of checkpoints to return. |

**Returns:** Async iterator of matching checkpoint tuples.

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:278`

---

#### aput

**Signature:**
```python
async def aput(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig
```

**Description:** Asynchronously store a checkpoint with its configuration and metadata.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration for the checkpoint. |
| checkpoint | Checkpoint | The checkpoint to store. |
| metadata | CheckpointMetadata | Additional metadata for the checkpoint. |
| new_versions | ChannelVersions | New channel versions as of this write. |

**Returns:** Updated configuration after storing the checkpoint.

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:303`

---

#### aput_writes

**Signature:**
```python
async def aput_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None
```

**Description:** Asynchronously store intermediate writes linked to a checkpoint.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration of the related checkpoint. |
| writes | Sequence[tuple[str, Any]] | List of writes to store, each as (channel, value) pair. |
| task_id | str | Identifier for the task creating the writes. |
| task_path | str | Path of the task creating the writes. Defaults to "". |

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:326`

---

#### adelete_thread

**Signature:** `async def adelete_thread(self, thread_id: str) -> None`

**Description:** Asynchronously delete all checkpoints and writes associated with a specific thread ID.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| thread_id | str | The thread ID whose checkpoints should be deleted. |

**Raises:** `NotImplementedError` - Implement this method in your custom checkpoint saver.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:346`

---

#### get_next_version

**Signature:** `def get_next_version(self, current: V | None, channel: None) -> V`

**Description:** Generate the next version ID for a channel. Default is to use integer versions, incrementing by `1`. If you override, you can use `str`/`int`/`float` versions, as long as they are monotonically increasing.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| current | V \| None | The current version identifier (`int`, `float`, or `str`). |
| channel | None | Deprecated argument, kept for backwards compatibility. |

**Returns:** The next version identifier, which must be monotonically increasing.

**Raises:** `NotImplementedError` - For string versions that are not implemented.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:357`

---

## Checkpoint Savers

### InMemorySaver

**Signature:**
```python
class InMemorySaver(
    BaseCheckpointSaver[str],
    AbstractContextManager,
    AbstractAsyncContextManager
):
    def __init__(
        self,
        *,
        serde: SerializerProtocol | None = None,
        factory: type[defaultdict] = defaultdict,
    ) -> None:
```

**Description:**

An in-memory checkpoint saver. This checkpoint saver stores checkpoints in memory using a `defaultdict`.

**Warning:** Only use `InMemorySaver` for debugging or testing purposes. For production use cases, use `PostgresSaver` / `AsyncPostgresSaver`.

**Constructor Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| serde | SerializerProtocol \| None | No | None | The serializer to use for serializing and deserializing checkpoints. |
| factory | type[defaultdict] | No | defaultdict | Factory for creating the storage dictionaries. |

**Example:**
```python
import asyncio

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph

builder = StateGraph(int)
builder.add_node("add_one", lambda x: x + 1)
builder.set_entry_point("add_one")
builder.set_finish_point("add_one")

memory = InMemorySaver()
graph = builder.compile(checkpointer=memory)
coro = graph.ainvoke(1, {"configurable": {"thread_id": "thread-1"}})
asyncio.run(coro)  # Output: 2
```

**Methods:**

All methods from `BaseCheckpointSaver` are implemented. Key methods include:

#### get_tuple

**Signature:** `def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None`

**Description:** Get a checkpoint tuple from the in-memory storage.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to use for retrieving the checkpoint. |

**Returns:** The retrieved checkpoint tuple, or None if no matching checkpoint was found.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/memory/__init__.py:135`

---

#### list

**Signature:**
```python
def list(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[CheckpointTuple]
```

**Description:** List checkpoints from the in-memory storage.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig \| None | Base configuration for filtering checkpoints. |
| filter | dict[str, Any] \| None | Additional filtering criteria for metadata. |
| before | RunnableConfig \| None | List checkpoints created before this configuration. |
| limit | int \| None | Maximum number of checkpoints to return. |

**Yields:** An iterator of matching checkpoint tuples.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/memory/__init__.py:217`

---

#### put

**Signature:**
```python
def put(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig
```

**Description:** Save a checkpoint to the in-memory storage.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to associate with the checkpoint. |
| checkpoint | Checkpoint | The checkpoint to save. |
| metadata | CheckpointMetadata | Additional metadata to save with the checkpoint. |
| new_versions | ChannelVersions | New versions as of this write. |

**Returns:** The updated config containing the saved checkpoint's timestamp.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/memory/__init__.py:326`

---

#### put_writes

**Signature:**
```python
def put_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None
```

**Description:** Save a list of writes to the in-memory storage.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to associate with the writes. |
| writes | Sequence[tuple[str, Any]] | The writes to save, each as (channel, value) pair. |
| task_id | str | Identifier for the task creating the writes. |
| task_path | str | Path of the task creating the writes. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/memory/__init__.py:372`

---

#### delete_thread

**Signature:** `def delete_thread(self, thread_id: str) -> None`

**Description:** Delete all checkpoints and writes associated with a thread ID.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| thread_id | str | The thread ID to delete. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/memory/__init__.py:410`

---

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/memory/__init__.py:31`

---

### MemorySaver

**Note:** `MemorySaver` is an alias for `InMemorySaver`, kept for backwards compatibility.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/memory/__init__.py:530`

---

### SqliteSaver

**Signature:**
```python
class SqliteSaver(BaseCheckpointSaver[str]):
    def __init__(
        self,
        conn: sqlite3.Connection,
        *,
        serde: SerializerProtocol | None = None,
    ) -> None:
```

**Description:**

A checkpoint saver that stores checkpoints in a SQLite database.

**Note:** This class is meant for lightweight, synchronous use cases (demos and small projects) and does not scale to multiple threads. For async support, consider using `AsyncSqliteSaver`.

**Constructor Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| conn | sqlite3.Connection | Yes | - | The SQLite database connection. |
| serde | SerializerProtocol \| None | No | None | The serializer to use for serializing and deserializing checkpoints. |

**Examples:**

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.graph import StateGraph

builder = StateGraph(int)
builder.add_node("add_one", lambda x: x + 1)
builder.set_entry_point("add_one")
builder.set_finish_point("add_one")
# Create a new SqliteSaver instance
# Note: check_same_thread=False is OK as the implementation uses a lock
# to ensure thread safety.
conn = sqlite3.connect("checkpoints.sqlite", check_same_thread=False)
memory = SqliteSaver(conn)
graph = builder.compile(checkpointer=memory)
config = {"configurable": {"thread_id": "1"}}
graph.get_state(config)
result = graph.invoke(3, config)
graph.get_state(config)
# StateSnapshot(values=4, next=(), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '0c62ca34-ac19-445d-bbb0-5b4984975b2a'}}, parent_config=None)
```

**Class Methods:**

#### from_conn_string

**Signature:**
```python
@classmethod
@contextmanager
def from_conn_string(cls, conn_string: str) -> Iterator[SqliteSaver]
```

**Description:** Create a new SqliteSaver instance from a connection string.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| conn_string | str | The SQLite connection string. |

**Yields:** A new SqliteSaver instance.

**Examples:**

In memory:
```python
with SqliteSaver.from_conn_string(":memory:") as memory:
    ...
```

To disk:
```python
with SqliteSaver.from_conn_string("checkpoints.sqlite") as memory:
    ...
```

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:90`

---

**Methods:**

#### setup

**Signature:** `def setup(self) -> None`

**Description:** Set up the checkpoint database. This method creates the necessary tables in the SQLite database if they don't already exist. It is called automatically when needed and should not be called directly by the user.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:122`

---

#### cursor

**Signature:**
```python
@contextmanager
def cursor(self, transaction: bool = True) -> Iterator[sqlite3.Cursor]
```

**Description:** Get a cursor for the SQLite database. This method is used internally by the SqliteSaver and should not be called directly by the user.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| transaction | bool | Whether to commit the transaction when the cursor is closed. Defaults to True. |

**Yields:** A cursor for the SQLite database.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:161`

---

#### get_tuple

**Signature:** `def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None`

**Description:** Get a checkpoint tuple from the database.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to use for retrieving the checkpoint. |

**Returns:** The retrieved checkpoint tuple, or None if no matching checkpoint was found.

**Examples:**

Basic:
```python
config = {"configurable": {"thread_id": "1"}}
checkpoint_tuple = memory.get_tuple(config)
print(checkpoint_tuple)
# CheckpointTuple(...)
```

With checkpoint ID:
```python
config = {
   "configurable": {
       "thread_id": "1",
       "checkpoint_ns": "",
       "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875",
   }
}
checkpoint_tuple = memory.get_tuple(config)
print(checkpoint_tuple)
# CheckpointTuple(...)
```

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:184`

---

#### list

**Signature:**
```python
def list(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[CheckpointTuple]
```

**Description:** List checkpoints from the database. The checkpoints are ordered by checkpoint ID in descending order (newest first).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig \| None | The config to use for listing the checkpoints. |
| filter | dict[str, Any] \| None | Additional filtering criteria for metadata. |
| before | RunnableConfig \| None | If provided, only checkpoints before the specified checkpoint ID are returned. |
| limit | int \| None | The maximum number of checkpoints to return. |

**Yields:** An iterator of checkpoint tuples.

**Examples:**
```python
from langgraph.checkpoint.sqlite import SqliteSaver
with SqliteSaver.from_conn_string(":memory:") as memory:
    # Run a graph, then list the checkpoints
    config = {"configurable": {"thread_id": "1"}}
    checkpoints = list(memory.list(config, limit=2))
print(checkpoints)
# [CheckpointTuple(...), CheckpointTuple(...)]

config = {"configurable": {"thread_id": "1"}}
before = {"configurable": {"checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875"}}
with SqliteSaver.from_conn_string(":memory:") as memory:
    # Run a graph, then list the checkpoints
    checkpoints = list(memory.list(config, before=before))
print(checkpoints)
# [CheckpointTuple(...), ...]
```

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:288`

---

#### put

**Signature:**
```python
def put(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig
```

**Description:** Save a checkpoint to the database.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to associate with the checkpoint. |
| checkpoint | Checkpoint | The checkpoint to save. |
| metadata | CheckpointMetadata | Additional metadata to save with the checkpoint. |
| new_versions | ChannelVersions | New channel versions as of this write. |

**Returns:** Updated configuration after storing the checkpoint.

**Examples:**

```python
from langgraph.checkpoint.sqlite import SqliteSaver
with SqliteSaver.from_conn_string(":memory:") as memory:
    config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
    checkpoint = {"ts": "2024-05-04T06:32:42.235444+00:00", "id": "1ef4f797-8335-6428-8001-8a1503f9b875", "channel_values": {"key": "value"}}
    saved_config = memory.put(config, checkpoint, {"source": "input", "step": 1, "writes": {"key": "value"}}, {})
print(saved_config)
# {'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef4f797-8335-6428-8001-8a1503f9b875'}}
```

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:380`

---

#### put_writes

**Signature:**
```python
def put_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None
```

**Description:** Store intermediate writes linked to a checkpoint.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration of the related checkpoint. |
| writes | Sequence[tuple[str, Any]] | List of writes to store, each as (channel, value) pair. |
| task_id | str | Identifier for the task creating the writes. |
| task_path | str | Path of the task creating the writes. |

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:438`

---

#### delete_thread

**Signature:** `def delete_thread(self, thread_id: str) -> None`

**Description:** Delete all checkpoints and writes associated with a thread ID.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| thread_id | str | The thread ID to delete. |

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:477`

---

#### get_next_version

**Signature:** `def get_next_version(self, current: str | None, channel: None) -> str`

**Description:** Generate the next version ID for a channel. This method creates a new version identifier for a channel based on its current version.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| current | str \| None | The current version identifier of the channel. |

**Returns:** The next version identifier, which is guaranteed to be monotonically increasing.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:537`

---

**Async Methods:**

The sync `SqliteSaver` class does not support async methods. If you call `aget_tuple`, `alist`, or `aput`, they will raise `NotImplementedError` with a message directing you to use `AsyncSqliteSaver` instead.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py:38`

---

### AsyncSqliteSaver

**Signature:**
```python
class AsyncSqliteSaver(BaseCheckpointSaver[str]):
    def __init__(
        self,
        conn: aiosqlite.Connection,
        *,
        serde: SerializerProtocol | None = None,
    ):
```

**Description:**

An asynchronous checkpoint saver that stores checkpoints in a SQLite database. This class provides an asynchronous interface for saving and retrieving checkpoints using a SQLite database.

**Warning:** While this class supports asynchronous checkpointing, it is not recommended for production workloads due to limitations in SQLite's write performance. For production use, consider PostgreSQL.

**Tip:** Remember to close the database connection after executing your code. The easiest way is to use the `async with` statement.

**Requirements:** Requires the `aiosqlite` package. Install with `pip install aiosqlite`.

**Constructor Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| conn | aiosqlite.Connection | Yes | - | The asynchronous SQLite database connection. |
| serde | SerializerProtocol \| None | No | None | The serializer used for encoding/decoding checkpoints. |

**Examples:**

Usage within StateGraph:
```python
import asyncio

from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver
from langgraph.graph import StateGraph

async def main():
    builder = StateGraph(int)
    builder.add_node("add_one", lambda x: x + 1)
    builder.set_entry_point("add_one")
    builder.set_finish_point("add_one")
    async with AsyncSqliteSaver.from_conn_string("checkpoints.db") as memory:
        graph = builder.compile(checkpointer=memory)
        coro = graph.ainvoke(1, {"configurable": {"thread_id": "thread-1"}})
        print(await asyncio.gather(coro))

asyncio.run(main())
# Output: [2]
```

Raw usage:
```python
import asyncio
import aiosqlite
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver

async def main():
    async with aiosqlite.connect("checkpoints.db") as conn:
        saver = AsyncSqliteSaver(conn)
        config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
        checkpoint = {"ts": "2023-05-03T10:00:00Z", "data": {"key": "value"}, "id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}
        saved_config = await saver.aput(config, checkpoint, {}, {})
        print(saved_config)
asyncio.run(main())
# {'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '0c62ca34-ac19-445d-bbb0-5b4984975b2a'}}
```

**Class Methods:**

#### from_conn_string

**Signature:**
```python
@classmethod
@asynccontextmanager
async def from_conn_string(
    cls, conn_string: str
) -> AsyncIterator[AsyncSqliteSaver]
```

**Description:** Create a new AsyncSqliteSaver instance from a connection string.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| conn_string | str | The SQLite connection string. |

**Yields:** A new AsyncSqliteSaver instance.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:123`

---

**Methods:**

#### setup

**Signature:** `async def setup(self) -> None`

**Description:** Set up the checkpoint database asynchronously. This method creates the necessary tables in the SQLite database if they don't already exist. It is called automatically when needed and should not be called directly by the user.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:274`

---

#### get_tuple

**Signature:** `def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None`

**Description:** Get a checkpoint tuple from the database. This is a synchronous wrapper that runs from a background thread.

**Note:** Synchronous calls to AsyncSqliteSaver are only allowed from a different thread. From the main thread, use the async interface (e.g., `await checkpointer.aget_tuple(...)`).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to use for retrieving the checkpoint. |

**Returns:** The retrieved checkpoint tuple, or None if no matching checkpoint was found.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:139`

---

#### aget_tuple

**Signature:** `async def aget_tuple(self, config: RunnableConfig) -> CheckpointTuple | None`

**Description:** Get a checkpoint tuple from the database asynchronously.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to use for retrieving the checkpoint. |

**Returns:** The retrieved checkpoint tuple, or None if no matching checkpoint was found.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:316`

---

#### list

**Signature:**
```python
def list(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[CheckpointTuple]
```

**Description:** List checkpoints from the database. The checkpoints are ordered by checkpoint ID in descending order (newest first).

**Note:** Synchronous calls to AsyncSqliteSaver are only allowed from a different thread. From the main thread, use the async interface (e.g., `checkpointer.alist(...)`).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig \| None | Base configuration for filtering checkpoints. |
| filter | dict[str, Any] \| None | Additional filtering criteria for metadata. |
| before | RunnableConfig \| None | If provided, only checkpoints before the specified checkpoint ID are returned. |
| limit | int \| None | Maximum number of checkpoints to return. |

**Yields:** An iterator of matching checkpoint tuples.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:169`

---

#### alist

**Signature:**
```python
async def alist(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> AsyncIterator[CheckpointTuple]
```

**Description:** List checkpoints from the database asynchronously. The checkpoints are ordered by checkpoint ID in descending order (newest first).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig \| None | Base configuration for filtering checkpoints. |
| filter | dict[str, Any] \| None | Additional filtering criteria for metadata. |
| before | RunnableConfig \| None | If provided, only checkpoints before the specified checkpoint ID are returned. |
| limit | int \| None | Maximum number of checkpoints to return. |

**Yields:** An asynchronous iterator of matching checkpoint tuples.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:400`

---

#### put

**Signature:**
```python
def put(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig
```

**Description:** Save a checkpoint to the database (synchronous wrapper).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to associate with the checkpoint. |
| checkpoint | Checkpoint | The checkpoint to save. |
| metadata | CheckpointMetadata | Additional metadata to save with the checkpoint. |
| new_versions | ChannelVersions | New channel versions as of this write. |

**Returns:** Updated configuration after storing the checkpoint.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:213`

---

#### aput

**Signature:**
```python
async def aput(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig
```

**Description:** Save a checkpoint to the database asynchronously.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to associate with the checkpoint. |
| checkpoint | Checkpoint | The checkpoint to save. |
| metadata | CheckpointMetadata | Additional metadata to save with the checkpoint. |
| new_versions | ChannelVersions | New channel versions as of this write. |

**Returns:** Updated configuration after storing the checkpoint.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:479`

---

#### put_writes

**Signature:**
```python
def put_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None
```

**Description:** Store intermediate writes linked to a checkpoint (synchronous wrapper).

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:238`

---

#### aput_writes

**Signature:**
```python
async def aput_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None
```

**Description:** Store intermediate writes linked to a checkpoint asynchronously.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration of the related checkpoint. |
| writes | Sequence[tuple[str, Any]] | List of writes to store, each as (channel, value) pair. |
| task_id | str | Identifier for the task creating the writes. |
| task_path | str | Path of the task creating the writes. |

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:531`

---

#### delete_thread

**Signature:** `def delete_thread(self, thread_id: str) -> None`

**Description:** Delete all checkpoints and writes associated with a thread ID (synchronous wrapper).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| thread_id | str | The thread ID to delete. |

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:249`

---

#### adelete_thread

**Signature:** `async def adelete_thread(self, thread_id: str) -> None`

**Description:** Delete all checkpoints and writes associated with a thread ID asynchronously.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| thread_id | str | The thread ID to delete. |

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:572`

---

#### get_next_version

**Signature:** `def get_next_version(self, current: str | None, channel: None) -> str`

**Description:** Generate the next version ID for a channel.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| current | str \| None | The current version identifier of the channel. |

**Returns:** The next version identifier, which is guaranteed to be monotonically increasing.

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:592`

---

**Source:** `/home/user/langgraph/libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py:30`

---

### PostgresSaver

**Signature:**
```python
class PostgresSaver(BasePostgresSaver):
    def __init__(
        self,
        conn: Conn,
        pipe: Pipeline | None = None,
        serde: SerializerProtocol | None = None,
    ) -> None:
```

**Description:**

Checkpointer that stores checkpoints in a Postgres database.

**Constructor Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| conn | Conn | Yes | - | The Postgres connection (Connection or ConnectionPool). |
| pipe | Pipeline \| None | No | None | Optional Pipeline for batching operations. |
| serde | SerializerProtocol \| None | No | None | The serializer to use for serializing and deserializing checkpoints. |

**Class Methods:**

#### from_conn_string

**Signature:**
```python
@classmethod
@contextmanager
def from_conn_string(
    cls, conn_string: str, *, pipeline: bool = False
) -> Iterator[PostgresSaver]
```

**Description:** Create a new PostgresSaver instance from a connection string.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| conn_string | str | The Postgres connection info string. |
| pipeline | bool | Whether to use Pipeline. |

**Returns:** A new PostgresSaver instance.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py:54`

---

**Methods:**

#### setup

**Signature:** `def setup(self) -> None`

**Description:** Set up the checkpoint database. This method creates the necessary tables in the Postgres database if they don't already exist and runs database migrations. It MUST be called directly by the user the first time checkpointer is used.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py:77`

---

#### get_tuple

**Signature:** `def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None`

**Description:** Get a checkpoint tuple from the database.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to use for retrieving the checkpoint. |

**Returns:** The retrieved checkpoint tuple, or None if no matching checkpoint was found.

**Examples:**

Basic:
```python
config = {"configurable": {"thread_id": "1"}}
checkpoint_tuple = memory.get_tuple(config)
print(checkpoint_tuple)
# CheckpointTuple(...)
```

With timestamp:
```python
config = {
   "configurable": {
       "thread_id": "1",
       "checkpoint_ns": "",
       "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875",
   }
}
checkpoint_tuple = memory.get_tuple(config)
print(checkpoint_tuple)
# CheckpointTuple(...)
```

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py:184`

---

#### list

**Signature:**
```python
def list(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[CheckpointTuple]
```

**Description:** List checkpoints from the database. The checkpoints are ordered by checkpoint ID in descending order (newest first).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig \| None | The config to use for listing the checkpoints. |
| filter | dict[str, Any] \| None | Additional filtering criteria for metadata. |
| before | RunnableConfig \| None | If provided, only checkpoints before the specified checkpoint ID are returned. |
| limit | int \| None | The maximum number of checkpoints to return. |

**Yields:** An iterator of checkpoint tuples.

**Examples:**
```python
from langgraph.checkpoint.postgres import PostgresSaver
DB_URI = "postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as memory:
    # Run a graph, then list the checkpoints
    config = {"configurable": {"thread_id": "1"}}
    checkpoints = list(memory.list(config, limit=2))
print(checkpoints)
# [CheckpointTuple(...), CheckpointTuple(...)]

config = {"configurable": {"thread_id": "1"}}
before = {"configurable": {"checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875"}}
with PostgresSaver.from_conn_string(DB_URI) as memory:
    # Run a graph, then list the checkpoints
    checkpoints = list(memory.list(config, before=before))
print(checkpoints)
# [CheckpointTuple(...), ...]
```

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py:104`

---

#### put

**Signature:**
```python
def put(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig
```

**Description:** Save a checkpoint to the database.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to associate with the checkpoint. |
| checkpoint | Checkpoint | The checkpoint to save. |
| metadata | CheckpointMetadata | Additional metadata to save with the checkpoint. |
| new_versions | ChannelVersions | New channel versions as of this write. |

**Returns:** Updated configuration after storing the checkpoint.

**Examples:**

```python
from langgraph.checkpoint.postgres import PostgresSaver
DB_URI = "postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as memory:
    config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
    checkpoint = {"ts": "2024-05-04T06:32:42.235444+00:00", "id": "1ef4f797-8335-6428-8001-8a1503f9b875", "channel_values": {"key": "value"}}
    saved_config = memory.put(config, checkpoint, {"source": "input", "step": 1, "writes": {"key": "value"}}, {})
print(saved_config)
# {'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef4f797-8335-6428-8001-8a1503f9b875'}}
```

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py:255`

---

#### put_writes

**Signature:**
```python
def put_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None
```

**Description:** Store intermediate writes linked to a checkpoint.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration of the related checkpoint. |
| writes | Sequence[tuple[str, Any]] | List of writes to store. |
| task_id | str | Identifier for the task creating the writes. |

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py:336`

---

#### delete_thread

**Signature:** `def delete_thread(self, thread_id: str) -> None`

**Description:** Delete all checkpoints and writes associated with a thread ID.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| thread_id | str | The thread ID to delete. |

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py:370`

---

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py:32`

---

### AsyncPostgresSaver

**Signature:**
```python
class AsyncPostgresSaver(BasePostgresSaver):
    def __init__(
        self,
        conn: Conn,
        pipe: AsyncPipeline | None = None,
        serde: SerializerProtocol | None = None,
    ) -> None:
```

**Description:**

Asynchronous checkpointer that stores checkpoints in a Postgres database.

**Constructor Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| conn | Conn | Yes | - | The Postgres connection (AsyncConnection or AsyncConnectionPool). |
| pipe | AsyncPipeline \| None | No | None | Optional AsyncPipeline for batching operations. |
| serde | SerializerProtocol \| None | No | None | The serializer to use for serializing and deserializing checkpoints. |

**Class Methods:**

#### from_conn_string

**Signature:**
```python
@classmethod
@asynccontextmanager
async def from_conn_string(
    cls,
    conn_string: str,
    *,
    pipeline: bool = False,
    serde: SerializerProtocol | None = None,
) -> AsyncIterator[AsyncPostgresSaver]
```

**Description:** Create a new AsyncPostgresSaver instance from a connection string.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| conn_string | str | The Postgres connection info string. |
| pipeline | bool | Whether to use AsyncPipeline. |
| serde | SerializerProtocol \| None | The serializer to use. |

**Returns:** A new AsyncPostgresSaver instance.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:55`

---

**Methods:**

#### setup

**Signature:** `async def setup(self) -> None`

**Description:** Set up the checkpoint database asynchronously. This method creates the necessary tables in the Postgres database if they don't already exist and runs database migrations. It MUST be called directly by the user the first time checkpointer is used.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:82`

---

#### get_tuple

**Signature:** `def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None`

**Description:** Get a checkpoint tuple from the database (synchronous wrapper).

**Note:** Synchronous calls to AsyncPostgresSaver are only allowed from a different thread. From the main thread, use the async interface (e.g., `await checkpointer.aget_tuple(...)`).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to use for retrieving the checkpoint. |

**Returns:** The retrieved checkpoint tuple, or None if no matching checkpoint was found.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:480`

---

#### aget_tuple

**Signature:** `async def aget_tuple(self, config: RunnableConfig) -> CheckpointTuple | None`

**Description:** Get a checkpoint tuple from the database asynchronously.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to use for retrieving the checkpoint. |

**Returns:** The retrieved checkpoint tuple, or None if no matching checkpoint was found.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:173`

---

#### list

**Signature:**
```python
def list(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[CheckpointTuple]
```

**Description:** List checkpoints from the database (synchronous wrapper). The checkpoints are ordered by checkpoint ID in descending order (newest first).

**Note:** Synchronous calls to AsyncPostgresSaver are only allowed from a different thread. From the main thread, use the async interface (e.g., `checkpointer.alist(...)`).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig \| None | Base configuration for filtering checkpoints. |
| filter | dict[str, Any] \| None | Additional filtering criteria for metadata. |
| before | RunnableConfig \| None | If provided, only checkpoints before the specified checkpoint ID are returned. |
| limit | int \| None | Maximum number of checkpoints to return. |

**Yields:** An iterator of matching checkpoint tuples.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:436`

---

#### alist

**Signature:**
```python
async def alist(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> AsyncIterator[CheckpointTuple]
```

**Description:** List checkpoints from the database asynchronously. The checkpoints are ordered by checkpoint ID in descending order (newest first).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig \| None | Base configuration for filtering checkpoints. |
| filter | dict[str, Any] \| None | Additional filtering criteria for metadata. |
| before | RunnableConfig \| None | If provided, only checkpoints before the specified checkpoint ID are returned. |
| limit | int \| None | Maximum number of checkpoints to return. |

**Yields:** An asynchronous iterator of matching checkpoint tuples.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:111`

---

#### put

**Signature:**
```python
def put(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig
```

**Description:** Save a checkpoint to the database (synchronous wrapper).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to associate with the checkpoint. |
| checkpoint | Checkpoint | The checkpoint to save. |
| metadata | CheckpointMetadata | Additional metadata to save with the checkpoint. |
| new_versions | ChannelVersions | New channel versions as of this write. |

**Returns:** Updated configuration after storing the checkpoint.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:510`

---

#### aput

**Signature:**
```python
async def aput(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig
```

**Description:** Save a checkpoint to the database asynchronously.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config to associate with the checkpoint. |
| checkpoint | Checkpoint | The checkpoint to save. |
| metadata | CheckpointMetadata | Additional metadata to save with the checkpoint. |
| new_versions | ChannelVersions | New channel versions as of this write. |

**Returns:** Updated configuration after storing the checkpoint.

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:224`

---

#### put_writes

**Signature:**
```python
def put_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None
```

**Description:** Store intermediate writes linked to a checkpoint (synchronous wrapper).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration of the related checkpoint. |
| writes | Sequence[tuple[str, Any]] | List of writes to store, each as (channel, value) pair. |
| task_id | str | Identifier for the task creating the writes. |
| task_path | str | Path of the task creating the writes. |

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:535`

---

#### aput_writes

**Signature:**
```python
async def aput_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None
```

**Description:** Store intermediate writes linked to a checkpoint asynchronously.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | Configuration of the related checkpoint. |
| writes | Sequence[tuple[str, Any]] | List of writes to store, each as (channel, value) pair. |
| task_id | str | Identifier for the task creating the writes. |

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:296`

---

#### delete_thread

**Signature:** `def delete_thread(self, thread_id: str) -> None`

**Description:** Delete all checkpoints and writes associated with a thread ID (synchronous wrapper).

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| thread_id | str | The thread ID to delete. |

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:556`

---

#### adelete_thread

**Signature:** `async def adelete_thread(self, thread_id: str) -> None`

**Description:** Delete all checkpoints and writes associated with a thread ID asynchronously.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| thread_id | str | The thread ID to delete. |

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:329`

---

**Source:** `/home/user/langgraph/libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py:32`

---

## Serialization

### SerializerProtocol

**Type:** `Protocol`

**Description:**

Protocol for serialization and deserialization of objects.

Valid implementations include the `pickle`, `json` and `orjson` modules.

**Methods:**

#### dumps_typed

**Signature:** `def dumps_typed(self, obj: Any) -> tuple[str, bytes]`

**Description:** Serialize an object to a tuple `(type, bytes)`.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| obj | Any | The object to serialize. |

**Returns:** A tuple of (type_name, serialized_bytes).

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/serde/base.py:15`

---

#### loads_typed

**Signature:** `def loads_typed(self, data: tuple[str, bytes]) -> Any`

**Description:** Deserialize an object from a tuple `(type, bytes)`.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| data | tuple[str, bytes] | The data to deserialize. |

**Returns:** The deserialized object.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/serde/base.py:15`

---

### JsonPlusSerializer

**Signature:**
```python
class JsonPlusSerializer(SerializerProtocol):
    def __init__(
        self,
        *,
        pickle_fallback: bool = False,
        allowed_json_modules: Sequence[tuple[str, ...]] | Literal[True] | None = None,
        __unpack_ext_hook__: Callable[[int, bytes], Any] | None = None,
    ) -> None:
```

**Description:**

Serializer that uses ormsgpack, with optional fallbacks.

**Warning:** Security note - This serializer is intended for use within the `BaseCheckpointSaver` class and called within the Pregel loop. It should not be used on untrusted python objects. If an attacker can write directly to your checkpoint database, they may be able to trigger code execution when data is deserialized.

**Constructor Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| pickle_fallback | bool | No | False | Whether to fall back to pickle for objects that can't be serialized with ormsgpack. |
| allowed_json_modules | Sequence[tuple[str, ...]] \| Literal[True] \| None | No | None | Allowed modules for deserialization. If True, all modules are allowed. If None, no modules are allowed. |
| __unpack_ext_hook__ | Callable[[int, bytes], Any] \| None | No | None | Custom hook for unpacking msgpack extensions. |

**Methods:**

#### dumps_typed

**Signature:** `def dumps_typed(self, obj: Any) -> tuple[str, bytes]`

**Description:** Serialize an object to a tuple `(type, bytes)`.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| obj | Any | The object to serialize. |

**Returns:** A tuple of (type_name, serialized_bytes).

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/serde/jsonplus.py:178`

---

#### loads_typed

**Signature:** `def loads_typed(self, data: tuple[str, bytes]) -> Any`

**Description:** Deserialize an object from a tuple `(type, bytes)`.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| data | tuple[str, bytes] | The data to deserialize. |

**Returns:** The deserialized object.

**Raises:** `NotImplementedError` - For unknown serialization types.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/serde/jsonplus.py:193`

---

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/serde/jsonplus.py:41`

---

## Storage

### BaseStore

**Signature:**
```python
class BaseStore(ABC):
    supports_ttl: bool = False
    ttl_config: TTLConfig | None = None
```

**Description:**

Abstract base class for persistent key-value stores. Stores enable persistence and memory that can be shared across threads, scoped to user IDs, assistant IDs, or other arbitrary namespaces. Some implementations may support semantic search capabilities through an optional `index` configuration.

**Note:** Semantic search capabilities vary by implementation and are typically disabled by default. TTL (time-to-live) support is also disabled by default. Subclasses must explicitly set `supports_ttl = True` to enable this feature.

**Attributes:**

- `supports_ttl` (bool): Whether the store supports TTL. Default: False.
- `ttl_config` (TTLConfig | None): TTL configuration if supported.

**Methods:**

#### batch

**Signature:** `def batch(self, ops: Iterable[Op]) -> list[Result]`

**Description:** Execute multiple operations synchronously in a single batch.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| ops | Iterable[Op] | An iterable of operations to execute. |

**Returns:** A list of results, where each result corresponds to an operation in the input. The order of results matches the order of input operations.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:724`

---

#### abatch

**Signature:** `async def abatch(self, ops: Iterable[Op]) -> list[Result]`

**Description:** Execute multiple operations asynchronously in a single batch.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| ops | Iterable[Op] | An iterable of operations to execute. |

**Returns:** A list of results, where each result corresponds to an operation in the input. The order of results matches the order of input operations.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:736`

---

#### get

**Signature:**
```python
def get(
    self,
    namespace: tuple[str, ...],
    key: str,
    *,
    refresh_ttl: bool | None = None,
) -> Item | None
```

**Description:** Retrieve a single item.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespace | tuple[str, ...] | Hierarchical path for the item. |
| key | str | Unique identifier within the namespace. |
| refresh_ttl | bool \| None | Whether to refresh TTLs for the returned item. If `None`, uses the store's default setting. |

**Returns:** The retrieved item or `None` if not found.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:748`

---

#### search

**Signature:**
```python
def search(
    self,
    namespace_prefix: tuple[str, ...],
    /,
    *,
    query: str | None = None,
    filter: dict[str, Any] | None = None,
    limit: int = 10,
    offset: int = 0,
    refresh_ttl: bool | None = None,
) -> list[SearchItem]
```

**Description:** Search for items within a namespace prefix.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespace_prefix | tuple[str, ...] | Hierarchical path prefix to search within. |
| query | str \| None | Optional query for natural language search. |
| filter | dict[str, Any] \| None | Key-value pairs to filter results. |
| limit | int | Maximum number of items to return. Default: 10. |
| offset | int | Number of items to skip before returning results. Default: 0. |
| refresh_ttl | bool \| None | Whether to refresh TTLs for the returned items. |

**Returns:** List of items matching the search criteria.

**Examples:**

Basic filtering:
```python
# Search for documents with specific metadata
results = store.search(
    ("docs",),
    filter={"type": "article", "status": "published"}
)
```

Natural language search (requires vector store implementation):
```python
# Initialize store with embedding configuration
store = InMemoryStore(
    index={
        "dims": 1536,  # embedding dimensions
        "embed": your_embedding_function,  # function to create embeddings
        "fields": ["text"]  # fields to embed. Defaults to ["$"]
    }
)

# Search for semantically similar documents
results = store.search(
    ("docs",),
    query="machine learning applications in healthcare",
    filter={"type": "research_paper"},
    limit=5
)
```

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:771`

---

#### put

**Signature:**
```python
def put(
    self,
    namespace: tuple[str, ...],
    key: str,
    value: dict[str, Any],
    index: Literal[False] | list[str] | None = None,
    *,
    ttl: float | None | NotProvided = NOT_PROVIDED,
) -> None
```

**Description:** Store or update an item in the store.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespace | tuple[str, ...] | Hierarchical path for the item. Example: `("documents", "user123")` |
| key | str | Unique identifier within the namespace. |
| value | dict[str, Any] | Dictionary containing the item's data. Must contain string keys and JSON-serializable values. |
| index | Literal[False] \| list[str] \| None | Controls how the item's fields are indexed for search. None (default): Use configured fields. False: Disable indexing. list[str]: List of field paths to index. |
| ttl | float \| None \| NotProvided | Time to live in minutes. If specified, the item will expire after this many minutes from when it was last accessed. |

**Note:** Indexing and TTL support depend on your store implementation.

**Examples:**

Store item:
```python
store.put(("docs",), "report", {"memory": "Will likes ai"})
```

Do not index item for semantic search:
```python
store.put(("docs",), "report", {"memory": "Will likes ai"}, index=False)
```

Index specific fields for search:
```python
store.put(("docs",), "report", {"memory": "Will likes ai"}, index=["memory"])
```

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:848`

---

#### delete

**Signature:** `def delete(self, namespace: tuple[str, ...], key: str) -> None`

**Description:** Delete an item.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespace | tuple[str, ...] | Hierarchical path for the item. |
| key | str | Unique identifier within the namespace. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:929`

---

#### list_namespaces

**Signature:**
```python
def list_namespaces(
    self,
    *,
    prefix: NamespacePath | None = None,
    suffix: NamespacePath | None = None,
    max_depth: int | None = None,
    limit: int = 100,
    offset: int = 0,
) -> list[tuple[str, ...]]
```

**Description:** List and filter namespaces in the store.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| prefix | NamespacePath \| None | Filter namespaces that start with this path. |
| suffix | NamespacePath \| None | Filter namespaces that end with this path. |
| max_depth | int \| None | Return namespaces up to this depth in the hierarchy. |
| limit | int | Maximum number of namespaces to return. Default: 100. |
| offset | int | Number of namespaces to skip for pagination. Default: 0. |

**Returns:** A list of namespace tuples that match the criteria.

**Examples:**
```python
# Example if you have the following namespaces:
# ("a", "b", "c")
# ("a", "b", "d", "e")
# ("a", "b", "d", "i")
# ("a", "b", "f")
# ("a", "c", "f")
store.list_namespaces(prefix=("a", "b"), max_depth=3)
# [("a", "b", "c"), ("a", "b", "d"), ("a", "b", "f")]
```

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:938`

---

#### aget

**Signature:**
```python
async def aget(
    self,
    namespace: tuple[str, ...],
    key: str,
    *,
    refresh_ttl: bool | None = None,
) -> Item | None
```

**Description:** Asynchronously retrieve a single item.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespace | tuple[str, ...] | Hierarchical path for the item. |
| key | str | Unique identifier within the namespace. |

**Returns:** The retrieved item or `None` if not found.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:993`

---

#### asearch

**Signature:**
```python
async def asearch(
    self,
    namespace_prefix: tuple[str, ...],
    /,
    *,
    query: str | None = None,
    filter: dict[str, Any] | None = None,
    limit: int = 10,
    offset: int = 0,
    refresh_ttl: bool | None = None,
) -> list[SearchItem]
```

**Description:** Asynchronously search for items within a namespace prefix.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespace_prefix | tuple[str, ...] | Hierarchical path prefix to search within. |
| query | str \| None | Optional query for natural language search. |
| filter | dict[str, Any] \| None | Key-value pairs to filter results. |
| limit | int | Maximum number of items to return. Default: 10. |
| offset | int | Number of items to skip before returning results. Default: 0. |
| refresh_ttl | bool \| None | Whether to refresh TTLs for the returned items. |

**Returns:** List of items matching the search criteria.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:1021`

---

#### aput

**Signature:**
```python
async def aput(
    self,
    namespace: tuple[str, ...],
    key: str,
    value: dict[str, Any],
    index: Literal[False] | list[str] | None = None,
    *,
    ttl: float | None | NotProvided = NOT_PROVIDED,
) -> None
```

**Description:** Asynchronously store or update an item in the store.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespace | tuple[str, ...] | Hierarchical path for the item. |
| key | str | Unique identifier within the namespace. |
| value | dict[str, Any] | Dictionary containing the item's data. |
| index | Literal[False] \| list[str] \| None | Controls how the item's fields are indexed for search. |
| ttl | float \| None \| NotProvided | Time to live in minutes. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:1101`

---

#### adelete

**Signature:** `async def adelete(self, namespace: tuple[str, ...], key: str) -> None`

**Description:** Asynchronously delete an item.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespace | tuple[str, ...] | Hierarchical path for the item. |
| key | str | Unique identifier within the namespace. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:1190`

---

#### alist_namespaces

**Signature:**
```python
async def alist_namespaces(
    self,
    *,
    prefix: NamespacePath | None = None,
    suffix: NamespacePath | None = None,
    max_depth: int | None = None,
    limit: int = 100,
    offset: int = 0,
) -> list[tuple[str, ...]]
```

**Description:** List and filter namespaces in the store asynchronously.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| prefix | NamespacePath \| None | Filter namespaces that start with this path. |
| suffix | NamespacePath \| None | Filter namespaces that end with this path. |
| max_depth | int \| None | Return namespaces up to this depth in the hierarchy. |
| limit | int | Maximum number of namespaces to return. Default: 100. |
| offset | int | Number of namespaces to skip for pagination. Default: 0. |

**Returns:** A list of namespace tuples that match the criteria.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:1199`

---

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:700`

---

### Item

**Signature:**
```python
class Item:
    def __init__(
        self,
        *,
        value: dict[str, Any],
        key: str,
        namespace: tuple[str, ...],
        created_at: datetime,
        updated_at: datetime,
    ):
```

**Description:**

Represents a stored item with metadata.

**Constructor Parameters:**

| Name | Type | Description |
|------|------|-------------|
| value | dict[str, Any] | The stored data as a dictionary. Keys are filterable. |
| key | str | Unique identifier within the namespace. |
| namespace | tuple[str, ...] | Hierarchical path defining the collection. |
| created_at | datetime | Timestamp of item creation. |
| updated_at | datetime | Timestamp of last update. |

**Methods:**

#### dict

**Signature:** `def dict(self) -> dict`

**Description:** Convert the Item to a dictionary representation.

**Returns:** Dictionary representation of the item.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:105`

---

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:51`

---

### Store Operation Types

#### GetOp

**Type:** `NamedTuple`

Operation to retrieve a specific item by its namespace and key.

**Fields:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| namespace | tuple[str, ...] | (required) | Hierarchical path that uniquely identifies the item's location. |
| key | str | (required) | Unique identifier for the item within its specific namespace. |
| refresh_ttl | bool | True | Whether to refresh TTLs for the returned item. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:157`

---

#### SearchOp

**Type:** `NamedTuple`

Operation to search for items within a specified namespace hierarchy.

**Fields:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| namespace_prefix | tuple[str, ...] | (required) | Hierarchical path prefix defining the search scope. |
| filter | dict[str, Any] \| None | None | Key-value pairs for filtering results based on exact matches or comparison operators. |
| limit | int | 10 | Maximum number of items to return in the search results. |
| offset | int | 0 | Number of matching items to skip for pagination. |
| query | str \| None | None | Natural language search query for semantic search capabilities. |
| refresh_ttl | bool | True | Whether to refresh TTLs for the returned items. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:203`

---

#### PutOp

**Type:** `NamedTuple`

Operation to store, update, or delete an item in the store.

**Fields:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| namespace | tuple[str, ...] | (required) | Hierarchical path that identifies the location of the item. |
| key | str | (required) | Unique identifier for the item within its namespace. |
| value | dict[str, Any] \| None | (required) | The data to store, or `None` to mark the item for deletion. |
| index | Literal[False] \| list[str] \| None | None | Controls how the item's fields are indexed for search operations. |
| ttl | float \| None | None | TTL (time-to-live) for the item in minutes. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:431`

---

#### ListNamespacesOp

**Type:** `NamedTuple`

Operation to list and filter namespaces in the store.

**Fields:**

| Name | Type | Default | Description |
|------|------|---------|-------------|
| match_conditions | tuple[MatchCondition, ...] \| None | None | Optional conditions for filtering namespaces. |
| max_depth | int \| None | None | Maximum depth of namespace hierarchy to return. |
| limit | int | 100 | Maximum number of namespaces to return. |
| offset | int | 0 | Number of namespaces to skip for pagination. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:368`

---

## Caching

### BaseCache

**Signature:**
```python
class BaseCache(ABC, Generic[ValueT]):
    def __init__(self, *, serde: SerializerProtocol | None = None) -> None:
```

**Description:**

Base class for a cache.

**Constructor Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| serde | SerializerProtocol \| None | No | None | The serializer to use. Defaults to `JsonPlusSerializer(pickle_fallback=True)`. |

**Attributes:**

- `serde` (SerializerProtocol): Serializer for encoding/decoding cached values.

**Type Definitions:**

- `ValueT`: TypeVar for the value type
- `Namespace`: `tuple[str, ...]` - Namespace path
- `FullKey`: `tuple[Namespace, str]` - Complete cache key (namespace, key)

**Methods:**

#### get

**Signature:** `def get(self, keys: Sequence[FullKey]) -> dict[FullKey, ValueT]`

**Description:** Get the cached values for the given keys.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| keys | Sequence[FullKey] | List of cache keys to retrieve. |

**Returns:** Dictionary mapping keys to their cached values.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/cache/base/__init__.py:24`

---

#### aget

**Signature:** `async def aget(self, keys: Sequence[FullKey]) -> dict[FullKey, ValueT]`

**Description:** Asynchronously get the cached values for the given keys.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| keys | Sequence[FullKey] | List of cache keys to retrieve. |

**Returns:** Dictionary mapping keys to their cached values.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/cache/base/__init__.py:28`

---

#### set

**Signature:** `def set(self, pairs: Mapping[FullKey, tuple[ValueT, int | None]]) -> None`

**Description:** Set the cached values for the given keys and TTLs.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| pairs | Mapping[FullKey, tuple[ValueT, int \| None]] | Dictionary mapping keys to (value, TTL) tuples. TTL is in seconds, None for no expiration. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/cache/base/__init__.py:32`

---

#### aset

**Signature:** `async def aset(self, pairs: Mapping[FullKey, tuple[ValueT, int | None]]) -> None`

**Description:** Asynchronously set the cached values for the given keys and TTLs.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| pairs | Mapping[FullKey, tuple[ValueT, int \| None]] | Dictionary mapping keys to (value, TTL) tuples. TTL is in seconds, None for no expiration. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/cache/base/__init__.py:36`

---

#### clear

**Signature:** `def clear(self, namespaces: Sequence[Namespace] | None = None) -> None`

**Description:** Delete the cached values for the given namespaces. If no namespaces are provided, clear all cached values.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespaces | Sequence[Namespace] \| None | List of namespaces to clear. None clears all. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/cache/base/__init__.py:40`

---

#### aclear

**Signature:** `async def aclear(self, namespaces: Sequence[Namespace] | None = None) -> None`

**Description:** Asynchronously delete the cached values for the given namespaces. If no namespaces are provided, clear all cached values.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| namespaces | Sequence[Namespace] \| None | List of namespaces to clear. None clears all. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/cache/base/__init__.py:45`

---

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/cache/base/__init__.py:15`

---

## Helper Functions

### copy_checkpoint

**Signature:** `def copy_checkpoint(checkpoint: Checkpoint) -> Checkpoint`

**Description:** Create a copy of a checkpoint.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| checkpoint | Checkpoint | The checkpoint to copy. |

**Returns:** A new checkpoint with copied values.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:93`

---

### empty_checkpoint

**Signature:** `def empty_checkpoint() -> Checkpoint`

**Description:** Create an empty checkpoint with default values.

**Returns:** A new empty checkpoint.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:448`

---

### create_checkpoint

**Signature:**
```python
def create_checkpoint(
    checkpoint: Checkpoint,
    channels: Mapping[str, ChannelProtocol] | None,
    step: int,
    *,
    id: str | None = None,
) -> Checkpoint
```

**Description:** Create a checkpoint for the given channels.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| checkpoint | Checkpoint | The base checkpoint. |
| channels | Mapping[str, ChannelProtocol] \| None | The channels to checkpoint. |
| step | int | The step number. |
| id | str \| None | Optional checkpoint ID. |

**Returns:** A new checkpoint.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:463`

---

### get_checkpoint_id

**Signature:** `def get_checkpoint_id(config: RunnableConfig) -> str | None`

**Description:** Get checkpoint ID from a config.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config. |

**Returns:** The checkpoint ID, or None if not found.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:386`

---

### get_checkpoint_metadata

**Signature:**
```python
def get_checkpoint_metadata(
    config: RunnableConfig, metadata: CheckpointMetadata
) -> CheckpointMetadata
```

**Description:** Get checkpoint metadata in a backwards-compatible manner.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| config | RunnableConfig | The config. |
| metadata | CheckpointMetadata | The metadata. |

**Returns:** The combined metadata.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:391`

---

### maybe_add_typed_methods

**Signature:**
```python
def maybe_add_typed_methods(
    serde: SerializerProtocol | UntypedSerializerProtocol,
) -> SerializerProtocol
```

**Description:** Wrap old serde implementations in a class with loads_typed and dumps_typed for backwards compatibility.

**Parameters:**

| Name | Type | Description |
|------|------|-------------|
| serde | SerializerProtocol \| UntypedSerializerProtocol | The serializer to wrap. |

**Returns:** A SerializerProtocol-compatible serializer.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/serde/base.py:40`

---

## Exceptions

### EmptyChannelError

**Description:** Raised when attempting to get the value of a channel that hasn't been updated for the first time yet.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:379`

---

### InvalidNamespaceError

**Description:** Raised when provided namespace is invalid.

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:541`

---

## Constants

### WRITES_IDX_MAP

**Type:** `dict`

Mapping from error type to error index. Regular writes just map to their index in the list of writes being saved. Special writes (e.g. errors) map to negative indices, to avoid those writes from conflicting with regular writes.

**Value:**
```python
{ERROR: -1, SCHEDULED: -2, INTERRUPT: -3, RESUME: -4}
```

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/checkpoint/base/__init__.py:429`

---

## Configuration Types

### IndexConfig

**Type:** `TypedDict` (total=False)

Configuration for indexing documents for semantic search in the store.

**Fields:**

| Name | Type | Description |
|------|------|-------------|
| dims | int | Number of dimensions in the embedding vectors. |
| embed | Embeddings \| EmbeddingsFunc \| AEmbeddingsFunc \| str | Function to generate embeddings from text. Can be a LangChain Embeddings instance, sync/async function, or provider string. |
| fields | list[str] \| None | Fields to extract text from for embedding generation. Uses JSON path syntax. |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:570`

---

### TTLConfig

**Type:** `TypedDict` (total=False)

Configuration for TTL (time-to-live) behavior in the store.

**Fields:**

| Name | Type | Description |
|------|------|-------------|
| refresh_on_read | bool | Default behavior for refreshing TTLs on read operations. Defaults to True. |
| default_ttl | float \| None | Default TTL in minutes for new items. Defaults to None (no expiration). |
| sweep_interval_minutes | int \| None | Interval in minutes between TTL sweep operations. Defaults to None (no sweeping). |

**Source:** `/home/user/langgraph/libs/checkpoint/langgraph/store/base/__init__.py:545`

---

This API reference provides comprehensive documentation for all public classes, methods, types, and functions in the LangGraph checkpoint module. For additional examples and usage patterns, refer to the individual class and method documentation above.
