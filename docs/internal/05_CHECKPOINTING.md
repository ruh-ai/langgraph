# LangGraph Checkpointing System

## Table of Contents

1. [Purpose of Checkpointing](#1-purpose-of-checkpointing)
2. [Checkpoint Data Structure](#2-checkpoint-data-structure)
3. [BaseCheckpointSaver Interface](#3-basecheckpointsaver-interface)
4. [MemorySaver](#4-memorysaver)
5. [SqliteSaver](#5-sqlitesaver)
6. [PostgresSaver](#6-postgressaver)
7. [Serialization (Serde)](#7-serialization-serde)
8. [BaseStore Interface](#8-basestore-interface)
9. [BaseCache Interface](#9-basecache-interface)
10. [Thread & Checkpoint IDs](#10-thread--checkpoint-ids)
11. [Human-in-the-Loop](#11-human-in-the-loop)
12. [Migration](#12-migration)

---

## 1. Purpose of Checkpointing

The LangGraph checkpointing system provides persistent state management for stateful, multi-actor agents. It solves several critical problems:

### Key Problems Solved

1. **State Persistence**: Maintains agent state across multiple interactions and sessions
2. **Conversation History**: Enables agents to remember previous conversations and context
3. **Crash Recovery**: Allows agents to resume from the last known state after failures
4. **Time Travel**: Supports replaying agent execution from any previous checkpoint
5. **Human-in-the-Loop**: Enables pausing execution for human review and intervention
6. **Debugging**: Provides visibility into agent state at each execution step
7. **Branching**: Supports creating alternative execution paths from any checkpoint

### Core Concepts

- **Thread**: A conversation or execution context, identified by `thread_id`
- **Checkpoint**: A snapshot of agent state at a specific point in time
- **Channel**: A state container (e.g., messages, context, custom state)
- **Pending Writes**: Intermediate writes that haven't been committed to a checkpoint yet

---

## 2. Checkpoint Data Structure

A checkpoint is a TypedDict containing the complete state of an agent at a specific point in time.

### Checkpoint Schema

```python
class Checkpoint(TypedDict):
    """State snapshot at a given point in time."""

    v: int
    """The version of the checkpoint format. Currently `1`."""

    id: str
    """The ID of the checkpoint.

    This is both unique and monotonically increasing, so can be used for sorting
    checkpoints from first to last."""

    ts: str
    """The timestamp of the checkpoint in ISO 8601 format."""

    channel_values: dict[str, Any]
    """The values of the channels at the time of the checkpoint.

    Mapping from channel name to deserialized channel snapshot value.
    """

    channel_versions: ChannelVersions
    """The versions of the channels at the time of the checkpoint.

    The keys are channel names and the values are monotonically increasing
    version strings for each channel.
    """

    versions_seen: dict[str, ChannelVersions]
    """Map from node ID to map from channel name to version seen.

    This keeps track of the versions of the channels that each node has seen.
    Used to determine which nodes to execute next.
    """

    pending_sends: list[tuple[str, Any]]
    """Messages pending to be sent to other nodes."""

    updated_channels: list[str] | None
    """The channels that were updated in this checkpoint."""
```

### CheckpointMetadata Schema

```python
class CheckpointMetadata(TypedDict, total=False):
    """Metadata associated with a checkpoint."""

    source: Literal["input", "loop", "update", "fork"]
    """The source of the checkpoint.

    - "input": The checkpoint was created from an input to invoke/stream/batch.
    - "loop": The checkpoint was created from inside the pregel loop.
    - "update": The checkpoint was created from a manual state update.
    - "fork": The checkpoint was created as a copy of another checkpoint.
    """

    step: int
    """The step number of the checkpoint.

    -1 for the first "input" checkpoint.
    0 for the first "loop" checkpoint.
    ... for the nth checkpoint afterwards.
    """

    parents: dict[str, str]
    """The IDs of the parent checkpoints.

    Mapping from checkpoint namespace to checkpoint ID.
    """
```

### CheckpointTuple

Checkpoints are returned as `CheckpointTuple` objects:

```python
class CheckpointTuple(NamedTuple):
    """A tuple containing a checkpoint and its associated data."""

    config: RunnableConfig
    """Configuration containing thread_id, checkpoint_id, and checkpoint_ns."""

    checkpoint: Checkpoint
    """The checkpoint data."""

    metadata: CheckpointMetadata
    """Metadata about the checkpoint."""

    parent_config: RunnableConfig | None
    """Configuration for the parent checkpoint, if any."""

    pending_writes: list[PendingWrite] | None
    """Writes that are pending for this checkpoint."""
```

---

## 3. BaseCheckpointSaver Interface

`BaseCheckpointSaver` is the abstract base class that all checkpoint savers must implement. It defines the contract for persisting and retrieving checkpoints.

### Core Methods

#### get_tuple

```python
def get_tuple(self, config: RunnableConfig) -> CheckpointTuple | None:
    """Fetch a checkpoint tuple using the given configuration.

    Args:
        config: Configuration specifying which checkpoint to retrieve.
            Must contain configurable.thread_id.
            May contain configurable.checkpoint_id to retrieve a specific checkpoint.

    Returns:
        The requested checkpoint tuple, or None if not found.
    """
```

#### list

```python
def list(
    self,
    config: RunnableConfig | None,
    *,
    filter: dict[str, Any] | None = None,
    before: RunnableConfig | None = None,
    limit: int | None = None,
) -> Iterator[CheckpointTuple]:
    """List checkpoints that match the given criteria.

    Args:
        config: Base configuration for filtering checkpoints.
        filter: Additional filtering criteria for metadata.
        before: List checkpoints created before this configuration.
        limit: Maximum number of checkpoints to return.

    Returns:
        Iterator of matching checkpoint tuples, ordered by checkpoint ID descending.
    """
```

#### put

```python
def put(
    self,
    config: RunnableConfig,
    checkpoint: Checkpoint,
    metadata: CheckpointMetadata,
    new_versions: ChannelVersions,
) -> RunnableConfig:
    """Store a checkpoint with its configuration and metadata.

    Args:
        config: Configuration for the checkpoint.
        checkpoint: The checkpoint to store.
        metadata: Additional metadata for the checkpoint.
        new_versions: New channel versions as of this write.

    Returns:
        RunnableConfig: Updated configuration after storing the checkpoint.
    """
```

#### put_writes

```python
def put_writes(
    self,
    config: RunnableConfig,
    writes: Sequence[tuple[str, Any]],
    task_id: str,
    task_path: str = "",
) -> None:
    """Store intermediate writes linked to a checkpoint.

    Args:
        config: Configuration of the related checkpoint.
        writes: List of writes to store, each as (channel, value) pair.
        task_id: Identifier for the task creating the writes.
        task_path: Path of the task creating the writes.
    """
```

#### delete_thread

```python
def delete_thread(self, thread_id: str) -> None:
    """Delete all checkpoints and writes associated with a specific thread ID.

    Args:
        thread_id: The thread ID whose checkpoints should be deleted.
    """
```

### Async Methods

All synchronous methods have async equivalents:

- `aget_tuple(config)` - Async version of get_tuple
- `alist(config, *, filter, before, limit)` - Async version of list
- `aput(config, checkpoint, metadata, new_versions)` - Async version of put
- `aput_writes(config, writes, task_id, task_path)` - Async version of put_writes
- `adelete_thread(thread_id)` - Async version of delete_thread

### Version Management

```python
def get_next_version(self, current: V | None, channel: None) -> V:
    """Generate the next version ID for a channel.

    Default is to use integer versions, incrementing by 1.

    Args:
        current: The current version identifier (int, float, or str).
        channel: Deprecated argument, kept for backwards compatibility.

    Returns:
        V: The next version identifier, which must be increasing.
    """
```

### Serializer

All checkpoint savers use a `SerializerProtocol` for serialization:

```python
class BaseCheckpointSaver(Generic[V]):
    serde: SerializerProtocol = JsonPlusSerializer()

    def __init__(self, *, serde: SerializerProtocol | None = None) -> None:
        self.serde = maybe_add_typed_methods(serde or self.serde)
```

---

## 4. MemorySaver

`InMemorySaver` (aliased as `MemorySaver` for backwards compatibility) is an in-memory checkpoint saver for development and testing.

### Implementation Details

```python
class InMemorySaver(BaseCheckpointSaver[str]):
    """An in-memory checkpoint saver.

    This checkpoint saver stores checkpoints in memory using a defaultdict.

    Note:
        Only use InMemorySaver for debugging or testing purposes.
        For production use cases we recommend PostgresSaver or AsyncPostgresSaver.
    """

    # thread ID -> checkpoint NS -> checkpoint ID -> checkpoint mapping
    storage: defaultdict[
        str,
        dict[str, dict[str, tuple[tuple[str, bytes], tuple[str, bytes], str | None]]],
    ]

    # (thread ID, checkpoint NS, checkpoint ID) -> (task ID, write idx)
    writes: defaultdict[
        tuple[str, str, str],
        dict[tuple[str, int], tuple[str, str, tuple[str, bytes], str]],
    ]

    # Blobs for channel values
    blobs: dict[
        tuple[str, str, str, str | int | float],  # thread id, checkpoint ns, channel, version
        tuple[str, bytes],
    ]
```

### Storage Strategy

1. **Checkpoints**: Stored in a nested dict structure keyed by thread_id, checkpoint_ns, and checkpoint_id
2. **Blobs**: Channel values are stored separately to optimize memory usage
3. **Writes**: Pending writes are stored separately and associated with checkpoints
4. **Versions**: Uses string versions with format `{version:032}.{random_hash:016}` for ordering

### Usage Example

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph

# Create a simple graph
builder = StateGraph(int)
builder.add_node("add_one", lambda x: x + 1)
builder.set_entry_point("add_one")
builder.set_finish_point("add_one")

# Initialize with memory saver
memory = InMemorySaver()
graph = builder.compile(checkpointer=memory)

# Run with thread ID
config = {"configurable": {"thread_id": "thread-1"}}
result = graph.invoke(1, config)  # Output: 2

# Retrieve state
state = graph.get_state(config)
print(state.values)  # 2
```

### Context Manager Support

```python
# InMemorySaver can be used with alternative storage backends
from langgraph.checkpoint.memory import InMemorySaver, PersistentDict

# Use persistent dict for disk-backed storage
with InMemorySaver(factory=PersistentDict) as memory:
    graph = builder.compile(checkpointer=memory)
    # ... use graph
```

### Limitations

- **Not thread-safe across processes**: Multiple processes cannot share the same InMemorySaver
- **Memory constraints**: All data is kept in RAM
- **No persistence**: Data is lost when the process ends (unless using PersistentDict)
- **Not for production**: Use PostgresSaver or AsyncPostgresSaver for production workloads

---

## 5. SqliteSaver

`SqliteSaver` is a checkpoint saver that stores checkpoints in a SQLite database. It's suitable for lightweight, single-threaded applications.

### Implementation Details

```python
class SqliteSaver(BaseCheckpointSaver[str]):
    """A checkpoint saver that stores checkpoints in a SQLite database.

    Note:
        This class is meant for lightweight, synchronous use cases
        (demos and small projects) and does not scale to multiple threads.
        For async support, use AsyncSqliteSaver.
    """

    conn: sqlite3.Connection
    is_setup: bool
    lock: threading.Lock
    jsonplus_serde: JsonPlusSerializer
```

### Database Schema

The SqliteSaver creates two tables:

#### checkpoints table

```sql
CREATE TABLE IF NOT EXISTS checkpoints (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    parent_checkpoint_id TEXT,
    type TEXT,
    checkpoint BLOB,
    metadata BLOB,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id)
);
```

#### writes table

```sql
CREATE TABLE IF NOT EXISTS writes (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    idx INTEGER NOT NULL,
    channel TEXT NOT NULL,
    type TEXT,
    value BLOB,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id, task_id, idx)
);
```

### Configuration

#### Basic Usage

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.graph import StateGraph

# Create connection
# Note: check_same_thread=False is OK as the implementation uses a lock
conn = sqlite3.connect("checkpoints.sqlite", check_same_thread=False)

# Initialize saver
memory = SqliteSaver(conn)

# Use with graph
builder = StateGraph(int)
builder.add_node("add_one", lambda x: x + 1)
builder.set_entry_point("add_one")
builder.set_finish_point("add_one")

graph = builder.compile(checkpointer=memory)
config = {"configurable": {"thread_id": "1"}}
result = graph.invoke(3, config)  # Output: 4
```

#### Using from_conn_string

```python
from langgraph.checkpoint.sqlite import SqliteSaver

# In-memory database
with SqliteSaver.from_conn_string(":memory:") as memory:
    graph = builder.compile(checkpointer=memory)
    # ... use graph

# Persistent database
with SqliteSaver.from_conn_string("checkpoints.sqlite") as memory:
    graph = builder.compile(checkpointer=memory)
    # ... use graph
```

### Thread Safety

- Uses a `threading.Lock` to ensure thread-safe access
- Safe to use with `check_same_thread=False` in SQLite connection
- Transactions are automatically committed after each operation
- WAL (Write-Ahead Logging) mode is enabled for better concurrency

### Versioning

SqliteSaver uses string-based versions with format: `{version:032}.{random_hash:016}`

```python
def get_next_version(self, current: str | None, channel: None) -> str:
    if current is None:
        current_v = 0
    elif isinstance(current, int):
        current_v = current
    else:
        current_v = int(current.split(".")[0])
    next_v = current_v + 1
    next_h = random.random()
    return f"{next_v:032}.{next_h:016}"
```

### Async Support

SqliteSaver does not support async operations. Use `AsyncSqliteSaver` instead:

```python
from langgraph.checkpoint.sqlite.aio import AsyncSqliteSaver

# Requires: pip install aiosqlite
```

### Best Practices

1. **Setup**: Call `setup()` is automatic on first use, creates tables if needed
2. **Connection Management**: Use context managers to ensure proper cleanup
3. **Single Process**: Best for single-process applications
4. **File Path**: Use absolute paths for database files
5. **Backups**: Regular database backups are recommended

---

## 6. PostgresSaver

`PostgresSaver` is a production-grade checkpoint saver that stores checkpoints in PostgreSQL with support for connection pooling and pipelines.

### Implementation Details

```python
class PostgresSaver(BasePostgresSaver):
    """Checkpointer that stores checkpoints in a Postgres database."""

    lock: threading.Lock
    conn: Connection | ConnectionPool
    pipe: Pipeline | None
    supports_pipeline: bool
```

### Database Schema

PostgresSaver uses three main tables:

#### checkpoints table

```sql
CREATE TABLE IF NOT EXISTS checkpoints (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    parent_checkpoint_id TEXT,
    checkpoint JSONB NOT NULL,
    metadata JSONB NOT NULL,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id)
);
```

#### checkpoint_writes table

```sql
CREATE TABLE IF NOT EXISTS checkpoint_writes (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL DEFAULT '',
    checkpoint_id TEXT NOT NULL,
    task_id TEXT NOT NULL,
    idx INTEGER NOT NULL,
    channel TEXT NOT NULL,
    type TEXT,
    value BYTEA,
    task_path TEXT,
    PRIMARY KEY (thread_id, checkpoint_ns, checkpoint_id, task_id, idx)
);
```

#### checkpoint_blobs table

```sql
CREATE TABLE IF NOT EXISTS checkpoint_blobs (
    thread_id TEXT NOT NULL,
    checkpoint_ns TEXT NOT NULL,
    channel TEXT NOT NULL,
    version TEXT NOT NULL,
    type TEXT NOT NULL,
    blob BYTEA,
    PRIMARY KEY (thread_id, checkpoint_ns, channel, version)
);
```

#### checkpoint_migrations table

```sql
CREATE TABLE IF NOT EXISTS checkpoint_migrations (
    v INTEGER PRIMARY KEY
);
```

### Configuration

#### Basic Usage with Connection String

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgres://user:password@localhost:5432/dbname?sslmode=disable"

with PostgresSaver.from_conn_string(DB_URI) as memory:
    memory.setup()  # MUST call setup() first time
    graph = builder.compile(checkpointer=memory)

    config = {"configurable": {"thread_id": "1"}}
    result = graph.invoke(3, config)
```

#### Connection Pooling

```python
from psycopg_pool import ConnectionPool
from langgraph.checkpoint.postgres import PostgresSaver

# Create connection pool
pool = ConnectionPool(
    conninfo=DB_URI,
    max_size=20,
    kwargs={
        "autocommit": True,
        "prepare_threshold": 0,
        "row_factory": dict_row
    }
)

# Use pool with saver
memory = PostgresSaver(pool)
memory.setup()
```

#### Pipeline Mode

PostgresSaver supports pipeline mode for batching operations:

```python
from langgraph.checkpoint.postgres import PostgresSaver

# Enable pipeline mode
with PostgresSaver.from_conn_string(DB_URI, pipeline=True) as memory:
    memory.setup()
    graph = builder.compile(checkpointer=memory)
    # Operations are batched automatically
```

### Optimization: Blob Storage

PostgresSaver optimizes storage by separating primitive values from complex objects:

- **Inline Storage**: Primitive values (str, int, float, bool, None) are stored in the checkpoint JSONB
- **Blob Storage**: Complex objects are stored in the checkpoint_blobs table

```python
def put(self, config, checkpoint, metadata, new_versions):
    # Separate primitive values from complex objects
    blob_values = {}
    for k, v in checkpoint["channel_values"].items():
        if v is None or isinstance(v, (str, int, float, bool)):
            pass  # Keep in checkpoint
        else:
            blob_values[k] = copy["channel_values"].pop(k)  # Move to blobs

    # Store blobs separately
    if blob_values:
        cur.executemany(UPSERT_CHECKPOINT_BLOBS_SQL, ...)

    # Store checkpoint with inline values
    cur.execute(UPSERT_CHECKPOINTS_SQL, ...)
```

### Migration System

PostgresSaver includes a migration system to handle schema updates:

```python
MIGRATIONS = [
    # Migration 0: Create initial tables
    "CREATE TABLE IF NOT EXISTS checkpoint_migrations ...",

    # Migration 1: Add new column
    "ALTER TABLE checkpoints ADD COLUMN ...",

    # Migration N: Latest schema changes
]

def setup(self):
    # Get current version
    results = cur.execute(
        "SELECT v FROM checkpoint_migrations ORDER BY v DESC LIMIT 1"
    )
    version = results.fetchone()["v"] if results.fetchone() else -1

    # Apply pending migrations
    for v, migration in enumerate(MIGRATIONS[version + 1:], start=version + 1):
        cur.execute(migration)
        cur.execute("INSERT INTO checkpoint_migrations (v) VALUES (%s)", (v,))
```

### Async Support

For async operations, use `AsyncPostgresSaver`:

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

async with AsyncPostgresSaver.from_conn_string(DB_URI) as memory:
    await memory.setup()
    graph = builder.compile(checkpointer=memory)

    config = {"configurable": {"thread_id": "1"}}
    result = await graph.ainvoke(3, config)
```

### Best Practices

1. **Call setup()**: Always call `setup()` before first use to create tables and run migrations
2. **Connection Pooling**: Use connection pools for production applications
3. **Index Strategy**: Add indexes on frequently queried columns (thread_id, checkpoint_id)
4. **Cleanup**: Implement regular cleanup of old checkpoints
5. **Monitoring**: Monitor connection pool usage and query performance
6. **Backups**: Regular database backups and point-in-time recovery

### Production Configuration Example

```python
from psycopg_pool import ConnectionPool
from langgraph.checkpoint.postgres import PostgresSaver
from psycopg.rows import dict_row

# Production-grade connection pool
pool = ConnectionPool(
    conninfo="postgres://user:password@postgres-host:5432/langgraph",
    min_size=2,
    max_size=20,
    max_waiting=10,
    max_lifetime=3600,
    max_idle=600,
    kwargs={
        "autocommit": True,
        "prepare_threshold": 0,
        "row_factory": dict_row,
    }
)

# Initialize saver
checkpointer = PostgresSaver(pool)
checkpointer.setup()

# Use with graph
graph = builder.compile(checkpointer=checkpointer)
```

---

## 7. Serialization (Serde)

The serialization system handles converting Python objects to bytes and back for storage in checkpoints.

### SerializerProtocol

The base protocol that all serializers must implement:

```python
@runtime_checkable
class SerializerProtocol(Protocol):
    """Protocol for serialization and deserialization of objects.

    - dumps_typed: Serialize an object to a tuple (type, bytes).
    - loads_typed: Deserialize an object from a tuple (type, bytes).
    """

    def dumps_typed(self, obj: Any) -> tuple[str, bytes]:
        """Serialize an object to (type_name, serialized_bytes)."""

    def loads_typed(self, data: tuple[str, bytes]) -> Any:
        """Deserialize an object from (type_name, serialized_bytes)."""
```

### JsonPlusSerializer

`JsonPlusSerializer` is the default serializer, using `ormsgpack` (MessagePack) with extensive Python type support.

```python
class JsonPlusSerializer(SerializerProtocol):
    """Serializer that uses ormsgpack, with optional fallbacks.

    Warning:
        This serializer is intended for use within the BaseCheckpointSaver
        class and called within the Pregel loop. It should not be used on
        untrusted python objects.
    """

    def __init__(
        self,
        *,
        pickle_fallback: bool = False,
        allowed_json_modules: Sequence[tuple[str, ...]] | Literal[True] | None = None,
    ) -> None:
        self.pickle_fallback = pickle_fallback
        self._allowed_modules = allowed_json_modules
```

### Supported Types

JsonPlusSerializer supports extensive Python types out of the box:

#### Built-in Types
- `None`, `bytes`, `bytearray`
- Basic types: `int`, `float`, `str`, `bool`
- Collections: `list`, `dict`, `tuple`, `set`, `frozenset`, `deque`

#### Date/Time Types
- `datetime`, `date`, `time`, `timedelta`
- `timezone`, `ZoneInfo`

#### Numeric Types
- `decimal.Decimal`
- `UUID`

#### Network Types
- `IPv4Address`, `IPv4Interface`, `IPv4Network`
- `IPv6Address`, `IPv6Interface`, `IPv6Network`

#### Python Types
- `re.Pattern`
- `pathlib.Path`
- `Enum` subclasses
- `NamedTuple`
- `dataclass` instances
- Pydantic models (v1 and v2)

#### Scientific Computing
- NumPy arrays (if numpy is installed)

#### LangGraph Types
- `SendProtocol` (for sending messages between nodes)
- `Item` (from store)

### Custom Type Serialization

JsonPlusSerializer handles custom types through MessagePack extension codes:

```python
EXT_CONSTRUCTOR_SINGLE_ARG = 0    # ClassName(arg)
EXT_CONSTRUCTOR_POS_ARGS = 1      # ClassName(*args)
EXT_CONSTRUCTOR_KW_ARGS = 2       # ClassName(**kwargs)
EXT_METHOD_SINGLE_ARG = 3         # ClassName.method(arg)
EXT_PYDANTIC_V1 = 4               # Pydantic v1 models
EXT_PYDANTIC_V2 = 5               # Pydantic v2 models
EXT_NUMPY_ARRAY = 6               # NumPy arrays
```

### Serialization Examples

#### Primitive Types

```python
serde = JsonPlusSerializer()

# None
serde.dumps_typed(None)  # ("null", b"")

# Bytes
serde.dumps_typed(b"hello")  # ("bytes", b"hello")

# MessagePack for everything else
serde.dumps_typed({"key": "value"})  # ("msgpack", <bytes>)
```

#### Pydantic Models

```python
from pydantic import BaseModel

class User(BaseModel):
    name: str
    age: int

user = User(name="Alice", age=30)
type_name, data = serde.dumps_typed(user)
# type_name: "msgpack"
# data contains: (module, class_name, model_dump(), method)

# Deserialize
restored = serde.loads_typed((type_name, data))
# User(name='Alice', age=30)
```

#### Dataclasses

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

point = Point(1.0, 2.0)
type_name, data = serde.dumps_typed(point)

# Deserialize
restored = serde.loads_typed((type_name, data))
# Point(x=1.0, y=2.0)
```

#### NumPy Arrays

```python
import numpy as np

arr = np.array([[1, 2], [3, 4]])
type_name, data = serde.dumps_typed(arr)

# Deserialize
restored = serde.loads_typed((type_name, data))
# array([[1, 2], [3, 4]])
```

### Security: Module Allowlists

For security, JsonPlusSerializer supports allowlisting modules for deserialization:

```python
# Default: no allowlist (safe for LangGraph internal use)
serde = JsonPlusSerializer()

# Allowlist specific modules
serde = JsonPlusSerializer(
    allowed_json_modules=[
        ("myapp", "models", "User"),
        ("myapp", "utils", "Config"),
    ]
)

# Allow all modules (DANGEROUS - only for trusted data)
serde = JsonPlusSerializer(allowed_json_modules=True)
```

### Pickle Fallback

For types that cannot be serialized with MessagePack, pickle can be used as a fallback:

```python
serde = JsonPlusSerializer(pickle_fallback=True)

# Custom class not in supported types
class CustomClass:
    def __init__(self, value):
        self.value = value

obj = CustomClass(42)
type_name, data = serde.dumps_typed(obj)
# type_name: "pickle"
# Falls back to pickle.dumps(obj)
```

### Backwards Compatibility

The serializer supports older serialization formats:

```python
class SerializerCompat(SerializerProtocol):
    """Wrapper for old serializers without typed methods."""

    def __init__(self, serde: UntypedSerializerProtocol):
        self.serde = serde

    def dumps_typed(self, obj: Any) -> tuple[str, bytes]:
        return type(obj).__name__, self.serde.dumps(obj)

    def loads_typed(self, data: tuple[str, bytes]) -> Any:
        return self.serde.loads(data[1])
```

---

## 8. BaseStore Interface

`BaseStore` is an abstract base class for persistent key-value stores that work alongside checkpoints for long-term memory.

### Purpose

While checkpoints store graph execution state, stores provide:
- **Long-term memory**: Data that persists across threads and conversations
- **User-scoped data**: Information specific to users, assistants, or other entities
- **Semantic search**: Optional vector search capabilities
- **Hierarchical organization**: Namespace-based data organization

### Core Concepts

```python
class BaseStore(ABC):
    """Abstract base class for persistent key-value stores.

    Stores enable persistence and memory that can be shared across threads,
    scoped to user IDs, assistant IDs, or other arbitrary namespaces.
    """

    supports_ttl: bool = False
    ttl_config: TTLConfig | None = None
```

### Data Model

#### Item

```python
class Item:
    """Represents a stored item with metadata."""

    value: dict[str, Any]          # The stored data
    key: str                        # Unique identifier within namespace
    namespace: tuple[str, ...]      # Hierarchical path
    created_at: datetime            # Creation timestamp
    updated_at: datetime            # Last update timestamp
```

#### Namespace Hierarchy

Namespaces are tuples of strings representing a hierarchical path:

```python
# User-specific documents
namespace = ("users", "user123", "documents")

# Assistant memories
namespace = ("assistants", "assistant-v2", "memories")

# Cache with versioning
namespace = ("cache", "embeddings", "v1")
```

### Operations

#### Get Operation

```python
class GetOp(NamedTuple):
    namespace: tuple[str, ...]
    key: str
    refresh_ttl: bool = True

# Usage
item = store.get(
    namespace=("users", "alice"),
    key="profile"
)
```

#### Put Operation

```python
class PutOp(NamedTuple):
    namespace: tuple[str, ...]
    key: str
    value: dict[str, Any] | None  # None = delete
    index: Literal[False] | list[str] | None = None
    ttl: float | None = None

# Usage
store.put(
    namespace=("users", "alice"),
    key="profile",
    value={"name": "Alice", "age": 30},
    index=["name"]  # Fields to index for search
)
```

#### Search Operation

```python
class SearchOp(NamedTuple):
    namespace_prefix: tuple[str, ...]
    filter: dict[str, Any] | None = None
    limit: int = 10
    offset: int = 0
    query: str | None = None  # Natural language query
    refresh_ttl: bool = True

# Usage - exact match search
results = store.search(
    namespace_prefix=("users",),
    filter={"status": "active"},
    limit=10
)

# Usage - semantic search (requires index config)
results = store.search(
    namespace_prefix=("documents",),
    query="machine learning papers about transformers",
    filter={"year": {"$gte": 2020}},
    limit=5
)
```

#### List Namespaces Operation

```python
class ListNamespacesOp(NamedTuple):
    match_conditions: tuple[MatchCondition, ...] | None = None
    max_depth: int | None = None
    limit: int = 100
    offset: int = 0

# Usage
namespaces = store.list_namespaces(
    prefix=("users",),
    max_depth=3,
    limit=100
)
```

### Indexing Configuration

For semantic search capabilities:

```python
class IndexConfig(TypedDict, total=False):
    dims: int  # Embedding dimensions
    embed: Embeddings | EmbeddingsFunc | AEmbeddingsFunc | str
    fields: list[str] | None  # JSON paths to index

# Example configuration
from langgraph.store.memory import InMemoryStore

store = InMemoryStore(
    index={
        "dims": 1536,
        "embed": "openai:text-embedding-3-small",
        "fields": ["$"]  # Index entire document
    }
)

# Store with indexing
store.put(
    namespace=("docs",),
    key="doc1",
    value={"text": "Machine learning is a subset of AI"},
    index=["text"]  # Index specific fields
)

# Semantic search
results = store.search(
    namespace_prefix=("docs",),
    query="artificial intelligence subfields"
)
```

### TTL (Time-To-Live) Support

```python
class TTLConfig(TypedDict, total=False):
    refresh_on_read: bool  # Refresh TTL on read operations
    default_ttl: float | None  # Default TTL in minutes
    sweep_interval_minutes: int | None  # Cleanup interval

# Example with TTL
store.put(
    namespace=("cache",),
    key="result",
    value={"data": "..."},
    ttl=60.0  # Expire after 60 minutes
)
```

### Batch Operations

All operations can be batched:

```python
from langgraph.store.base import GetOp, PutOp, SearchOp

ops = [
    GetOp(namespace=("users",), key="alice"),
    PutOp(namespace=("users",), key="bob", value={"name": "Bob"}),
    SearchOp(namespace_prefix=("users",), filter={"status": "active"}),
]

results = store.batch(ops)  # Sync
results = await store.abatch(ops)  # Async
```

### Usage with LangGraph

```python
from langgraph.store.memory import InMemoryStore
from langgraph.graph import StateGraph

# Create store
store = InMemoryStore()

# Create graph with store
def my_node(state, *, store):
    # Access user data
    user_data = store.get(
        namespace=("users", state["user_id"]),
        key="preferences"
    )

    # Update user data
    store.put(
        namespace=("users", state["user_id"]),
        key="last_interaction",
        value={"timestamp": datetime.now().isoformat()}
    )

    return state

builder = StateGraph(dict)
builder.add_node("process", my_node)
graph = builder.compile(store=store)
```

---

## 9. BaseCache Interface

`BaseCache` provides a caching abstraction for storing computed values with TTL support.

### Purpose

Caches are used to:
- **Speed up repeated computations**: Store expensive computation results
- **Reduce API calls**: Cache LLM responses and API results
- **Share results**: Share cached values across threads and processes
- **Automatic expiration**: TTL-based cleanup of stale data

### Interface

```python
class BaseCache(ABC, Generic[ValueT]):
    """Base class for a cache."""

    serde: SerializerProtocol = JsonPlusSerializer(pickle_fallback=True)

    def __init__(self, *, serde: SerializerProtocol | None = None):
        """Initialize the cache with a serializer."""
        self.serde = serde or self.serde

    @abstractmethod
    def get(self, keys: Sequence[FullKey]) -> dict[FullKey, ValueT]:
        """Get the cached values for the given keys."""

    @abstractmethod
    async def aget(self, keys: Sequence[FullKey]) -> dict[FullKey, ValueT]:
        """Asynchronously get the cached values for the given keys."""

    @abstractmethod
    def set(self, pairs: Mapping[FullKey, tuple[ValueT, int | None]]) -> None:
        """Set the cached values for the given keys and TTLs."""

    @abstractmethod
    async def aset(self, pairs: Mapping[FullKey, tuple[ValueT, int | None]]) -> None:
        """Asynchronously set the cached values for the given keys and TTLs."""

    @abstractmethod
    def clear(self, namespaces: Sequence[Namespace] | None = None) -> None:
        """Delete the cached values for the given namespaces."""

    @abstractmethod
    async def aclear(self, namespaces: Sequence[Namespace] | None = None) -> None:
        """Asynchronously delete the cached values for the given namespaces."""
```

### Key Structure

```python
Namespace = tuple[str, ...]
FullKey = tuple[Namespace, str]

# Examples
key1 = (("user", "alice"), "embeddings")
key2 = (("llm", "gpt4"), "response_123")
```

### Usage Example

```python
from langgraph.cache.memory import InMemoryCache

# Create cache
cache = InMemoryCache()

# Set values with TTL
cache.set({
    (("embeddings",), "doc1"): ([0.1, 0.2, 0.3], 3600),  # 1 hour TTL
    (("responses",), "q1"): ("The answer is...", None),  # No expiration
})

# Get values
results = cache.get([
    (("embeddings",), "doc1"),
    (("responses",), "q1"),
])

# Clear namespace
cache.clear([("embeddings",)])  # Clear all embeddings

# Clear all
cache.clear()  # Clear entire cache
```

### Namespace Organization

Organize cache keys by purpose:

```python
# LLM responses
namespace = ("llm", "gpt-4", "responses")

# Embeddings
namespace = ("embeddings", "text-embedding-3-small")

# API results
namespace = ("api", "weather", "forecasts")

# User-specific caches
namespace = ("users", "user123", "recommendations")
```

### Implementations

#### InMemoryCache

```python
from langgraph.cache.memory import InMemoryCache

cache = InMemoryCache()
# Simple in-memory dict-based cache
# Lost when process ends
```

#### RedisCache

```python
from langgraph.cache.redis import RedisCache

cache = RedisCache(
    host="localhost",
    port=6379,
    db=0,
)
# Persistent, shared across processes
# Automatic TTL support
```

---

## 10. Thread & Checkpoint IDs

Understanding how threads and checkpoints are identified is crucial for proper state management.

### Thread ID

A `thread_id` represents a conversation or execution context:

```python
config = {
    "configurable": {
        "thread_id": "conversation-123"
    }
}
```

**Thread ID Characteristics:**
- String identifier
- User-defined (you control the naming)
- Groups related checkpoints together
- Can represent: user sessions, conversations, workflows, etc.

**Naming Strategies:**

```python
# User-based threads
thread_id = f"user-{user_id}"

# Conversation-based
thread_id = f"conv-{conversation_id}"

# Date-based
thread_id = f"session-{user_id}-{date}"

# UUID for uniqueness
import uuid
thread_id = str(uuid.uuid4())
```

### Checkpoint ID

A `checkpoint_id` uniquely identifies a specific checkpoint within a thread:

```python
config = {
    "configurable": {
        "thread_id": "conversation-123",
        "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875"
    }
}
```

**Checkpoint ID Characteristics:**
- Generated automatically using UUID v6
- Monotonically increasing (can sort chronologically)
- Unique within a thread
- Format: UUIDv6 string

**Generation:**

```python
from langgraph.checkpoint.base.id import uuid6

checkpoint_id = str(uuid6(clock_seq=step))
```

### Checkpoint Namespace

A `checkpoint_ns` provides additional isolation within a thread:

```python
config = {
    "configurable": {
        "thread_id": "conversation-123",
        "checkpoint_ns": "subgraph-1",
        "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875"
    }
}
```

**Use Cases:**
- **Subgraphs**: Separate checkpoints for nested graphs
- **Parallel execution**: Isolate parallel branches
- **Multi-agent**: Separate checkpoints per agent

Default namespace is empty string `""`.

### Config Structure

Complete configuration structure:

```python
from langchain_core.runnables import RunnableConfig

config: RunnableConfig = {
    "configurable": {
        # Required
        "thread_id": str,

        # Optional
        "checkpoint_id": str,           # Specific checkpoint to retrieve
        "checkpoint_ns": str,            # Namespace (default: "")
        "checkpoint_map": dict,          # Advanced: map of namespaces
    },

    # Optional metadata (not used for checkpoint lookup)
    "metadata": dict,

    # Optional tags
    "tags": list[str],
}
```

### Retrieving Checkpoints

#### Get Latest Checkpoint

```python
config = {"configurable": {"thread_id": "conv-123"}}
checkpoint_tuple = checkpointer.get_tuple(config)
```

#### Get Specific Checkpoint

```python
config = {
    "configurable": {
        "thread_id": "conv-123",
        "checkpoint_id": "1ef4f797-8335-6428-8001-8a1503f9b875"
    }
}
checkpoint_tuple = checkpointer.get_tuple(config)
```

#### List All Checkpoints for Thread

```python
config = {"configurable": {"thread_id": "conv-123"}}
for checkpoint_tuple in checkpointer.list(config):
    print(f"Step {checkpoint_tuple.metadata['step']}: {checkpoint_tuple.config}")
```

### Time Travel

Access historical states:

```python
# Get current state
current_state = graph.get_state({"configurable": {"thread_id": "123"}})

# Go back to parent checkpoint
parent_config = current_state.parent_config
if parent_config:
    previous_state = graph.get_state(parent_config)

# List all states in order
states = list(graph.get_state_history({"configurable": {"thread_id": "123"}}))
for state in states:
    print(f"Step {state.metadata['step']}: {state.values}")
```

### Branching from Checkpoints

Create alternative execution paths:

```python
# Get checkpoint to branch from
config = {
    "configurable": {
        "thread_id": "original",
        "checkpoint_id": "checkpoint-to-branch-from"
    }
}

# Update state to create new branch
new_config = graph.update_state(
    config,
    {"messages": [HumanMessage(content="Alternative path")]},
    as_node="user_input"
)

# This creates a new checkpoint with source="update"
# Continue execution from this new branch
result = graph.invoke(None, new_config)
```

---

## 11. Human-in-the-Loop

Checkpointing enables powerful human-in-the-loop patterns by allowing execution to pause and resume.

### Interrupts

Interrupt execution before or after specific nodes:

```python
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node("fetch_data", fetch_data)
builder.add_node("process_data", process_data)
builder.add_node("store_results", store_results)

# Interrupt before processing for review
graph = builder.compile(
    checkpointer=checkpointer,
    interrupt_before=["process_data"]
)
```

### Interrupt Flow

1. **Execution pauses**: Graph stops before the interrupt node
2. **State saved**: Current state saved to checkpoint
3. **Review**: Human can inspect state
4. **Resume or Update**: Continue execution or modify state

### Example: Review Before Processing

```python
from langgraph.graph import StateGraph, END
from langchain_core.messages import HumanMessage

# Define graph with interrupt
builder = StateGraph(MessagesState)
builder.add_node("fetch", fetch_data_node)
builder.add_node("analyze", analyze_node)
builder.add_node("respond", respond_node)

builder.set_entry_point("fetch")
builder.add_edge("fetch", "analyze")
builder.add_edge("analyze", "respond")
builder.add_edge("respond", END)

# Interrupt before analysis
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=["analyze"]
)

# Run until interrupt
config = {"configurable": {"thread_id": "review-123"}}
result = graph.invoke(
    {"messages": [HumanMessage(content="Fetch user data")]},
    config
)

# Execution pauses before "analyze"
# Inspect current state
state = graph.get_state(config)
print("Data fetched:", state.values)
print("Next node:", state.next)  # ["analyze"]

# Option 1: Resume execution
graph.invoke(None, config)  # Continues from checkpoint

# Option 2: Modify state before resuming
graph.update_state(
    config,
    {"messages": [HumanMessage(content="Additional context")]},
)
graph.invoke(None, config)  # Continues with modified state
```

### Approval Workflows

```python
class ApprovalState(TypedDict):
    request: str
    approved: bool
    result: str

def request_node(state):
    return {"request": "Process sensitive data"}

def check_approval(state):
    # This will interrupt - human must approve
    return state

def process_node(state):
    if not state.get("approved"):
        return {"result": "Not approved"}
    return {"result": "Processing complete"}

builder = StateGraph(ApprovalState)
builder.add_node("request", request_node)
builder.add_node("check_approval", check_approval)
builder.add_node("process", process_node)

builder.set_entry_point("request")
builder.add_edge("request", "check_approval")
builder.add_edge("check_approval", "process")
builder.add_edge("process", END)

# Interrupt for approval
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=["process"]
)

# Submit request
config = {"configurable": {"thread_id": "approval-1"}}
graph.invoke({}, config)

# Paused at "check_approval" -> "process"
state = graph.get_state(config)

# Human approves
graph.update_state(config, {"approved": True})

# Continue processing
result = graph.invoke(None, config)
print(result["result"])  # "Processing complete"
```

### Dynamic Interrupts

Conditionally interrupt based on state:

```python
def should_review(state) -> bool:
    """Determine if human review is needed."""
    return state.get("confidence", 1.0) < 0.8

def process_with_confidence(state):
    result = analyze(state["data"])
    return {
        "result": result,
        "confidence": calculate_confidence(result)
    }

builder = StateGraph(State)
builder.add_node("analyze", process_with_confidence)
builder.add_node("review", human_review_node)
builder.add_node("finalize", finalize_node)

builder.set_entry_point("analyze")
builder.add_conditional_edges(
    "analyze",
    should_review,
    {
        True: "review",      # Low confidence -> review
        False: "finalize"    # High confidence -> finalize
    }
)

graph = builder.compile(
    checkpointer=memory,
    interrupt_before=["review"]  # Only interrupts if routed to review
)
```

### Editing State During Interrupts

```python
# Start execution
config = {"configurable": {"thread_id": "edit-1"}}
graph.invoke({"query": "What is AI?"}, config)

# Paused for review
state = graph.get_state(config)
print("Current state:", state.values)

# Edit state before resuming
graph.update_state(
    config,
    {
        "query": "What is AI? Focus on machine learning.",
        "context": "Previous conversation about neural networks"
    },
    as_node="user_input"  # Act as if this came from user_input node
)

# Resume with edited state
result = graph.invoke(None, config)
```

### Interrupt After

Interrupt after a node completes:

```python
graph = builder.compile(
    checkpointer=memory,
    interrupt_after=["fetch_data"]
)

# Execute through fetch_data, then pause
config = {"configurable": {"thread_id": "after-1"}}
graph.invoke({"query": "..."}, config)

# fetch_data completed, execution paused
state = graph.get_state(config)
print("Fetched data:", state.values["data"])

# Continue to next node
graph.invoke(None, config)
```

### Multi-Step Approval

```python
graph = builder.compile(
    checkpointer=memory,
    interrupt_before=["send_email", "charge_card", "delete_data"]
)

# Each sensitive operation requires approval
config = {"configurable": {"thread_id": "multi-approval"}}

# Step 1: Runs until send_email
graph.invoke({"action": "purchase"}, config)
state = graph.get_state(config)
print(f"About to: {state.next}")  # ["send_email"]

# Approve email
graph.invoke(None, config)

# Step 2: Runs until charge_card
state = graph.get_state(config)
print(f"About to: {state.next}")  # ["charge_card"]

# Approve charge
graph.invoke(None, config)

# Continues until next interrupt or END
```

---

## 12. Migration

The migration system handles schema evolution and data format changes across checkpoint versions.

### Migration Framework

PostgresSaver includes a built-in migration system:

```python
class BasePostgresSaver:
    MIGRATIONS: list[str] = [
        # Migration 0: Initial schema
        """
        CREATE TABLE IF NOT EXISTS checkpoint_migrations (
            v INTEGER PRIMARY KEY
        );
        CREATE TABLE IF NOT EXISTS checkpoints (...);
        CREATE TABLE IF NOT EXISTS checkpoint_writes (...);
        """,

        # Migration 1: Add blobs table
        """
        CREATE TABLE IF NOT EXISTS checkpoint_blobs (
            thread_id TEXT NOT NULL,
            checkpoint_ns TEXT NOT NULL,
            channel TEXT NOT NULL,
            version TEXT NOT NULL,
            type TEXT NOT NULL,
            blob BYTEA,
            PRIMARY KEY (thread_id, checkpoint_ns, channel, version)
        );
        """,

        # Migration 2: Add indexes
        """
        CREATE INDEX IF NOT EXISTS idx_checkpoints_thread_id
        ON checkpoints(thread_id, checkpoint_ns, checkpoint_id DESC);
        """,

        # Future migrations...
    ]
```

### Setup and Migration Execution

```python
def setup(self) -> None:
    """Set up the checkpoint database asynchronously.

    This method creates the necessary tables in the Postgres database if they don't
    already exist and runs database migrations. It MUST be called directly by the user
    the first time checkpointer is used.
    """
    with self._cursor() as cur:
        # Create migrations table
        cur.execute(self.MIGRATIONS[0])

        # Get current version
        results = cur.execute(
            "SELECT v FROM checkpoint_migrations ORDER BY v DESC LIMIT 1"
        )
        row = results.fetchone()
        version = row["v"] if row is not None else -1

        # Apply pending migrations
        for v, migration in enumerate(
            self.MIGRATIONS[version + 1:],
            start=version + 1
        ):
            cur.execute(migration)
            cur.execute(
                "INSERT INTO checkpoint_migrations (v) VALUES (%s)",
                (v,)
            )
```

### Migration Strategy

#### 1. Additive Changes (Safe)

Add new columns/tables without breaking existing code:

```python
# Migration N: Add new column
"""
ALTER TABLE checkpoints
ADD COLUMN IF NOT EXISTS new_field JSONB DEFAULT '{}';
"""

# Old code continues to work
# New code can use new field
```

#### 2. Data Transformations

Transform existing data to new format:

```python
# Migration N: Transform data
"""
-- Add new column
ALTER TABLE checkpoints ADD COLUMN metadata_v2 JSONB;

-- Migrate data
UPDATE checkpoints
SET metadata_v2 = metadata::jsonb
WHERE metadata_v2 IS NULL;

-- Drop old column (only after all code updated)
-- ALTER TABLE checkpoints DROP COLUMN metadata;
"""
```

#### 3. Checkpoint Version Evolution

Handle checkpoint format changes:

```python
def _migrate_checkpoint(
    self,
    checkpoint: dict[str, Any]
) -> dict[str, Any]:
    """Migrate checkpoint to current version."""
    v = checkpoint.get("v", 1)

    if v < 2:
        # Migrate v1 -> v2
        checkpoint = self._migrate_v1_to_v2(checkpoint)

    if v < 3:
        # Migrate v2 -> v3
        checkpoint = self._migrate_v2_to_v3(checkpoint)

    return checkpoint

def _migrate_v1_to_v2(self, checkpoint: dict) -> dict:
    """Example: Add pending_sends field."""
    if "pending_sends" not in checkpoint:
        checkpoint["pending_sends"] = []
    checkpoint["v"] = 2
    return checkpoint
```

### PostgresSaver Migration Example

Real migration from PostgresSaver handling pending_sends:

```python
def _migrate_pending_sends(
    self,
    sends: list[dict],
    checkpoint: dict,
    channel_values: list[tuple],
) -> None:
    """Migrate pending sends from v3 to v4 format.

    In v3, pending sends were stored separately.
    In v4, they're stored in the checkpoint.
    """
    if sends and checkpoint["v"] < 4:
        # Add pending sends to checkpoint
        if "pending_sends" not in checkpoint:
            checkpoint["pending_sends"] = []

        for send in sends:
            checkpoint["pending_sends"].append(
                (send["channel"], send["value"])
            )

        checkpoint["v"] = 4
```

### Custom Migration for SqliteSaver

SqliteSaver doesn't have built-in migrations, but you can implement them:

```python
def migrate_sqlite_checkpoint(conn: sqlite3.Connection):
    """Custom migration for SQLite."""
    cur = conn.cursor()

    # Check if migration needed
    cur.execute(
        "SELECT name FROM sqlite_master "
        "WHERE type='table' AND name='checkpoint_blobs'"
    )
    if not cur.fetchone():
        # Add blobs table
        cur.execute("""
            CREATE TABLE checkpoint_blobs (
                thread_id TEXT NOT NULL,
                checkpoint_ns TEXT NOT NULL,
                channel TEXT NOT NULL,
                version TEXT NOT NULL,
                type TEXT,
                blob BLOB,
                PRIMARY KEY (thread_id, checkpoint_ns, channel, version)
            )
        """)
        conn.commit()
```

### Migration Best Practices

1. **Always Forward**: Migrations should only move forward, never backward
2. **Idempotent**: Migrations should be safe to run multiple times
3. **Test**: Test migrations on copy of production data
4. **Backup**: Always backup database before running migrations
5. **Gradual Rollout**: Deploy migrations before code that requires them
6. **Version Tracking**: Track migration version in database
7. **Backwards Compatibility**: Maintain compatibility during transition period

### Migration Workflow

```python
# 1. Add migration to MIGRATIONS list
MIGRATIONS = [
    # ... existing migrations ...

    # New migration
    """
    CREATE TABLE IF NOT EXISTS new_feature (
        id SERIAL PRIMARY KEY,
        data JSONB
    );
    """
]

# 2. Deploy database migration
checkpointer = PostgresSaver(conn)
checkpointer.setup()  # Runs new migrations

# 3. Deploy code that uses new feature
# Code can now use new_feature table

# 4. Optional: Clean up deprecated features in future migration
```

### Handling Breaking Changes

For unavoidable breaking changes:

```python
# Migration N: Mark for deprecation
"""
-- Add new table
CREATE TABLE checkpoints_v2 (...);

-- Migrate data
INSERT INTO checkpoints_v2
SELECT * FROM checkpoints;

-- Keep old table for compatibility
-- Will drop in migration N+5
"""

# Migration N+5: Remove deprecated
"""
DROP TABLE IF EXISTS checkpoints;
ALTER TABLE checkpoints_v2 RENAME TO checkpoints;
"""
```

### Version Checking

Check checkpoint version compatibility:

```python
CURRENT_CHECKPOINT_VERSION = 4

def validate_checkpoint(checkpoint: dict) -> None:
    """Validate checkpoint version."""
    v = checkpoint.get("v", 1)

    if v > CURRENT_CHECKPOINT_VERSION:
        raise ValueError(
            f"Checkpoint version {v} is newer than supported "
            f"version {CURRENT_CHECKPOINT_VERSION}. "
            f"Please upgrade LangGraph."
        )

    if v < MINIMUM_SUPPORTED_VERSION:
        raise ValueError(
            f"Checkpoint version {v} is too old. "
            f"Please migrate using an intermediate version."
        )
```

---

## Summary

The LangGraph checkpointing system provides a comprehensive solution for:

- **State Persistence**: Save and restore agent state across sessions
- **Multiple Backends**: In-memory, SQLite, and PostgreSQL implementations
- **Serialization**: Robust serialization of Python objects with JsonPlusSerializer
- **Human-in-the-Loop**: Pause execution for human review and intervention
- **Time Travel**: Access and branch from historical states
- **Long-term Memory**: Store and search data with BaseStore
- **Caching**: Speed up computations with BaseCache
- **Migrations**: Evolve schema and data formats over time

This architecture enables building production-grade, stateful AI agents with full control over execution flow and state management.
