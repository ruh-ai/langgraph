# LangGraph Channels: Data Flow Architecture

## Table of Contents

1. [What are Channels](#what-are-channels)
2. [BaseChannel Interface](#basechannel-interface)
3. [Channel Types](#channel-types)
   - [LastValue](#lastvalue-channel)
   - [AnyValue](#anyvalue-channel)
   - [Topic](#topic-channel)
   - [BinaryOperatorAggregate](#binaryoperatoraggregate-channel)
   - [EphemeralValue](#ephemeralvalue-channel)
   - [UntrackedValue](#untrackedvalue-channel)
   - [NamedBarrierValue](#namedbarriervalue-channel)
4. [How Channels Connect to State](#how-channels-connect-to-state)
5. [Custom Reducers](#custom-reducers)
6. [Advanced Channel Behaviors](#advanced-channel-behaviors)

---

## What are Channels

Channels are the fundamental data containers in LangGraph that manage how state values are stored, updated, and propagated through the graph during execution. They define the semantics of state updates, including:

- **How multiple updates are aggregated** (e.g., last write wins, accumulation, reduction)
- **When values become available** to downstream nodes
- **Whether values are checkpointed** for persistence
- **How values are cleared** between steps

Every field in a LangGraph state schema is backed by a channel. When you define a state field, LangGraph automatically selects or creates the appropriate channel type based on type annotations and metadata.

### Key Responsibilities

1. **Value Storage**: Hold the current value(s) for a state field
2. **Update Semantics**: Define how new values are combined with existing ones
3. **Availability Control**: Determine when values can be read by nodes
4. **Checkpointing**: Manage serialization for persistence and time-travel
5. **Synchronization**: Coordinate concurrent updates from multiple nodes

---

## BaseChannel Interface

All channels inherit from `BaseChannel[Value, Update, Checkpoint]`, a generic abstract base class with three type parameters:

- **Value**: The type returned by `get()`
- **Update**: The type accepted in `update()`
- **Checkpoint**: The serialized representation

### Core Properties

```python
@property
@abstractmethod
def ValueType(self) -> Any:
    """The type of the value stored in the channel."""

@property
@abstractmethod
def UpdateType(self) -> Any:
    """The type of the update received by the channel."""
```

### Lifecycle Methods

#### Serialization/Deserialization

```python
def checkpoint(self) -> Checkpoint | Any:
    """Return a serializable representation of the channel's current state.

    Raises EmptyChannelError if the channel is empty or doesn't support checkpoints.
    """

@abstractmethod
def from_checkpoint(self, checkpoint: Checkpoint | Any) -> Self:
    """Return a new channel initialized from a checkpoint.

    Complex data structures in the checkpoint should be copied.
    """

def copy(self) -> Self:
    """Return a copy of the channel.

    Default implementation uses checkpoint() and from_checkpoint().
    Subclasses can override for efficiency.
    """
```

#### Read Operations

```python
@abstractmethod
def get(self) -> Value:
    """Return the current value of the channel.

    Raises EmptyChannelError if the channel is empty (never updated).
    """

def is_available(self) -> bool:
    """Return True if the channel has a value, False otherwise.

    More efficient than catching EmptyChannelError from get().
    """
```

#### Write Operations

```python
@abstractmethod
def update(self, values: Sequence[Update]) -> bool:
    """Update the channel with a sequence of updates.

    The order of updates is arbitrary. Called by Pregel at the end of each step.
    If there are no updates, called with an empty sequence.

    Raises InvalidUpdateError if the updates are invalid.
    Returns True if the channel was modified, False otherwise.
    """

def consume(self) -> bool:
    """Notify the channel that a subscribed task ran.

    Default: no-op. Channels can use this to modify state
    (e.g., clear ephemeral values).

    Returns True if the channel was modified, False otherwise.
    """

def finish(self) -> bool:
    """Notify the channel that the Pregel run is finishing.

    Default: no-op. Channels can use this to transition state
    (e.g., make values available after finish).

    Returns True if the channel was modified, False otherwise.
    """
```

---

## Channel Types

### LastValue Channel

**Purpose**: Stores the last value received. Can accept at most one value per step.

**When to Use**:
- Default channel for most state fields
- When you want "last write wins" semantics
- When concurrent updates to the same field are an error

**Behavior**:
- Accepts exactly one update per step
- Raises `InvalidUpdateError` if multiple nodes write to it in the same step
- Value is checkpointed and persisted
- Empty until first update

**Code Example**:

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph

# Implicit LastValue channel
class State(TypedDict):
    message: str  # Automatically uses LastValue channel
    count: int    # Automatically uses LastValue channel

graph = StateGraph(State)

def node_a(state: State) -> dict:
    return {"message": "Hello from A"}

def node_b(state: State) -> dict:
    return {"count": 42}

graph.add_node("a", node_a)
graph.add_node("b", node_b)
graph.add_edge("a", "b")
graph.set_entry_point("a")
graph.set_finish_point("b")

app = graph.compile()
result = app.invoke({"message": "", "count": 0})
# result: {"message": "Hello from A", "count": 42}
```

**Error Case**:

```python
# This will raise InvalidUpdateError
graph.add_edge("__start__", "a")
graph.add_edge("__start__", "b")
# Both nodes run concurrently and try to update different fields - OK

def node_c(state: State) -> dict:
    return {"message": "From C"}

def node_d(state: State) -> dict:
    return {"message": "From D"}  # ERROR: concurrent write to same field

# If both run in parallel, LastValue will reject the concurrent updates
```

---

### AnyValue Channel

**Purpose**: Like `LastValue`, but assumes all concurrent updates are equal.

**When to Use**:
- Multiple nodes may write the same value to a field
- You want to avoid errors when concurrent identical updates occur
- Typically used for read-only shared state

**Behavior**:
- Accepts multiple updates per step (uses the last one)
- Assumes all updates are equal (doesn't verify)
- Value is checkpointed
- Can be cleared by passing empty updates

**Code Example**:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.channels import AnyValue
from langgraph.graph import StateGraph

class State(TypedDict):
    # Multiple nodes can set the same config value
    config: Annotated[dict, AnyValue]
    result: str

def node_a(state: State) -> dict:
    return {"config": {"version": "1.0"}}

def node_b(state: State) -> dict:
    # Same value - OK with AnyValue
    return {"config": {"version": "1.0"}}

graph = StateGraph(State)
graph.add_node("a", node_a)
graph.add_node("b", node_b)
# Both can run in parallel and set config
```

---

### Topic Channel

**Purpose**: Implements a pub/sub pattern. Collects multiple values into a sequence.

**When to Use**:
- Fan-out pattern where multiple nodes produce values
- Collecting messages, events, or results from parallel nodes
- Need to process all values, not just the last one

**Parameters**:
- `accumulate` (bool): Whether to accumulate values across steps (default: False)

**Behavior**:
- Collects all updates into a list
- If `accumulate=False`, clears values at the start of each step
- If `accumulate=True`, keeps accumulating across steps
- Can receive lists or individual values (flattens them)
- Value type is `Sequence[T]`

**Code Example**:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.channels import Topic
from langgraph.graph import StateGraph

class State(TypedDict):
    # Collect results from multiple analyzers
    analyses: Annotated[list[str], Topic(str)]

def analyzer_1(state: State) -> dict:
    return {"analyses": "Analysis from source 1"}

def analyzer_2(state: State) -> dict:
    return {"analyses": "Analysis from source 2"}

def analyzer_3(state: State) -> dict:
    # Can also return a list
    return {"analyses": ["Finding A", "Finding B"]}

def summarize(state: State) -> dict:
    # All analyses are collected in the list
    all_analyses = state["analyses"]
    # Returns: ["Analysis from source 1", "Analysis from source 2",
    #           "Finding A", "Finding B"]
    return {"summary": f"Combined {len(all_analyses)} analyses"}

graph = StateGraph(State)
graph.add_node("a1", analyzer_1)
graph.add_node("a2", analyzer_2)
graph.add_node("a3", analyzer_3)
graph.add_node("summarize", summarize)

# All analyzers run in parallel
graph.add_edge("__start__", "a1")
graph.add_edge("__start__", "a2")
graph.add_edge("__start__", "a3")

# Summarize runs after all complete
graph.add_edge("a1", "summarize")
graph.add_edge("a2", "summarize")
graph.add_edge("a3", "summarize")
```

**Accumulating Topic**:

```python
class LogState(TypedDict):
    # Accumulate logs across all steps
    logs: Annotated[list[str], Topic(str, accumulate=True)]

# Logs persist and accumulate across the entire graph execution
```

---

### BinaryOperatorAggregate Channel

**Purpose**: Aggregates updates using a binary operator (reducer function).

**When to Use**:
- Need custom aggregation logic (sum, merge, concatenate, etc.)
- Multiple nodes contribute to the same aggregate value
- Standard reducers like `operator.add` for lists/numbers

**Behavior**:
- First update sets the initial value (or uses type's default constructor)
- Subsequent updates are combined using the operator: `value = operator(value, update)`
- Supports `Overwrite` to replace the value instead of reducing
- Can only receive one `Overwrite` per super-step

**Code Example**:

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.graph import StateGraph

class State(TypedDict):
    # List concatenation
    messages: Annotated[list[str], operator.add]

    # Integer sum
    total_cost: Annotated[int, operator.add]

    # Custom reducer
    metadata: Annotated[dict, lambda a, b: {**a, **b}]

def step_1(state: State) -> dict:
    return {
        "messages": ["Hello"],
        "total_cost": 10,
        "metadata": {"source": "step_1"}
    }

def step_2(state: State) -> dict:
    return {
        "messages": ["World"],
        "total_cost": 20,
        "metadata": {"timestamp": "2024-01-01"}
    }

def step_3(state: State) -> dict:
    return {
        "messages": ["!"],
        "total_cost": 5,
    }

graph = StateGraph(State)
graph.add_node("s1", step_1)
graph.add_node("s2", step_2)
graph.add_node("s3", step_3)

# Run sequentially
graph.add_edge("__start__", "s1")
graph.add_edge("s1", "s2")
graph.add_edge("s2", "s3")
graph.set_finish_point("s3")

app = graph.compile()
result = app.invoke({"messages": [], "total_cost": 0, "metadata": {}})

# Result:
# {
#     "messages": ["Hello", "World", "!"],
#     "total_cost": 35,
#     "metadata": {"source": "step_1", "timestamp": "2024-01-01"}
# }
```

**Using Overwrite**:

```python
from langgraph.types import Overwrite

def reset_node(state: State) -> dict:
    # Reset total_cost instead of adding to it
    return {"total_cost": Overwrite(0)}

# Or using dict syntax:
def reset_node_alt(state: State) -> dict:
    return {"total_cost": {"__overwrite__": 0}}
```

**Common Patterns**:

```python
from typing import Annotated
from collections.abc import Sequence

# List concatenation
Annotated[list[str], operator.add]

# Set union
Annotated[set[int], operator.or_]

# Dictionary merge (shallow)
Annotated[dict, lambda a, b: {**a, **b}]

# Dictionary deep merge
def deep_merge(a: dict, b: dict) -> dict:
    result = a.copy()
    for key, value in b.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = deep_merge(result[key], value)
        else:
            result[key] = value
    return result

Annotated[dict, deep_merge]

# Custom aggregation
def collect_unique(existing: list, new: str) -> list:
    if new not in existing:
        return existing + [new]
    return existing

Annotated[list[str], collect_unique]
```

---

### EphemeralValue Channel

**Purpose**: Stores a value for one step, then clears it automatically.

**When to Use**:
- Passing temporary data between nodes in the same step
- Interrupt signals or one-time commands
- Data that shouldn't persist in checkpoints

**Parameters**:
- `guard` (bool): If True, only accept one value per step (default: True)

**Behavior**:
- Value is available for one step only
- Automatically cleared in the next `update()` call
- Value IS checkpointed (but typically empty in checkpoints)
- With `guard=True`, raises error on concurrent updates

**Code Example**:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.channels import EphemeralValue
from langgraph.graph import StateGraph

class State(TypedDict):
    persistent_data: str
    # Temporary command, cleared after each step
    command: Annotated[str, EphemeralValue]

def control_node(state: State) -> dict:
    # Issue a command
    return {"command": "process"}

def worker_node(state: State) -> dict:
    command = state.get("command")
    if command == "process":
        return {"persistent_data": "Processed!"}
    return {}

def observer_node(state: State) -> dict:
    # Command is already cleared by this step
    command = state.get("command")  # Will be None/missing
    return {}

graph = StateGraph(State)
graph.add_node("control", control_node)
graph.add_node("worker", worker_node)
graph.add_node("observer", observer_node)

graph.add_edge("__start__", "control")
graph.add_edge("control", "worker")  # command available here
graph.add_edge("worker", "observer")  # command cleared here
```

**Guard Parameter**:

```python
class State(TypedDict):
    # Multiple nodes can set signal, takes last one
    signal: Annotated[str, EphemeralValue(guard=False)]

# With guard=False, no error if multiple nodes write
```

---

### UntrackedValue Channel

**Purpose**: Stores a value that is NEVER checkpointed.

**When to Use**:
- Runtime-only data (connections, file handles, etc.)
- Large objects that shouldn't be serialized
- Cached computed values
- Sensitive data that shouldn't be persisted

**Parameters**:
- `guard` (bool): If True, only accept one value per step (default: True)

**Behavior**:
- Works like `LastValue` but never serialized
- `checkpoint()` always returns `MISSING`
- Value is lost when loading from checkpoint
- Must be recomputed or reinitialized after loading

**Code Example**:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.channels import UntrackedValue
from langgraph.graph import StateGraph

class State(TypedDict):
    # Checkpointed data
    user_id: str

    # Never checkpointed
    db_connection: Annotated[object, UntrackedValue]
    cache: Annotated[dict, UntrackedValue]

def setup_node(state: State) -> dict:
    # Create connection (won't be saved)
    conn = create_database_connection()
    return {"db_connection": conn}

def work_node(state: State) -> dict:
    conn = state["db_connection"]
    # Use connection...
    return {"user_id": "processed"}

# When loading from checkpoint:
# - user_id is restored
# - db_connection is lost, must be recreated
```

---

### NamedBarrierValue Channel

**Purpose**: Synchronization primitive that waits for all named signals before becoming available.

**When to Use**:
- Coordinating multiple concurrent branches
- Waiting for all prerequisite tasks to complete
- Implementing join points in parallel workflows

**Parameters**:
- `names` (set): Set of expected signal names

**Behavior**:
- Starts empty with no signals received
- Each update must be one of the expected names
- Becomes available only when ALL names received
- `get()` returns `None` when complete
- `consume()` clears all signals after use

**Code Example**:

```python
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.channels import NamedBarrierValue
from langgraph.graph import StateGraph

class State(TypedDict):
    # Wait for all three processors to complete
    ready: Annotated[None, NamedBarrierValue(str, {"proc_a", "proc_b", "proc_c"})]
    result: str

def processor_a(state: State) -> dict:
    # Do work...
    return {"ready": "proc_a"}  # Signal completion

def processor_b(state: State) -> dict:
    # Do work...
    return {"ready": "proc_b"}  # Signal completion

def processor_c(state: State) -> dict:
    # Do work...
    return {"ready": "proc_c"}  # Signal completion

def finalizer(state: State) -> dict:
    # Only runs when all processors have signaled
    # state["ready"] is available and equals None
    return {"result": "All processors complete"}

graph = StateGraph(State)
graph.add_node("a", processor_a)
graph.add_node("b", processor_b)
graph.add_node("c", processor_c)
graph.add_node("final", finalizer)

# All processors run in parallel
graph.add_edge("__start__", "a")
graph.add_edge("__start__", "b")
graph.add_edge("__start__", "c")

# Finalizer waits for barrier
graph.add_edge("a", "final")
graph.add_edge("b", "final")
graph.add_edge("c", "final")
```

**Error Handling**:

```python
def bad_node(state: State) -> dict:
    # ERROR: "invalid" not in expected names
    return {"ready": "invalid"}  # Raises InvalidUpdateError
```

---

### Special Variants: AfterFinish Channels

Two channels have `AfterFinish` variants that only make values available after the graph's `finish()` is called:

#### LastValueAfterFinish

- Accumulates updates like `LastValue`
- Only becomes available after `finish()`
- Cleared by `consume()` after being read

#### NamedBarrierValueAfterFinish

- Waits for all named signals
- Only becomes available after `finish()` AND all signals received
- Cleared by `consume()` after being read

**Use Case**: Output channels that should only be read after the graph completes.

```python
from langgraph.channels import LastValueAfterFinish

class State(TypedDict):
    # Available during execution
    intermediate: str

    # Only available after finish()
    final_output: Annotated[str, LastValueAfterFinish]
```

---

## How Channels Connect to State

When you define a state schema, LangGraph automatically creates channels for each field based on type annotations.

### Channel Selection Logic

```python
def _get_channel(name: str, annotation: Any) -> BaseChannel:
    """
    1. Check for explicit channel annotation
    2. Check for reducer function (creates BinaryOperatorAggregate)
    3. Fall back to LastValue
    """
```

### Explicit Channel Annotation

```python
from typing import Annotated
from langgraph.channels import EphemeralValue, Topic

class State(TypedDict):
    # Explicitly use EphemeralValue channel
    temp: Annotated[str, EphemeralValue]

    # Explicitly use Topic channel
    events: Annotated[list[str], Topic(str)]

    # Can also pass channel instance
    custom: Annotated[int, EphemeralValue(int, guard=False)]
```

### Reducer Function Annotation

```python
import operator
from typing import Annotated

class State(TypedDict):
    # Creates BinaryOperatorAggregate with operator.add
    items: Annotated[list[str], operator.add]

    # Custom reducer
    merged: Annotated[dict, lambda a, b: {**a, **b}]
```

### Default Channel (LastValue)

```python
class State(TypedDict):
    # No annotation = LastValue channel
    name: str
    age: int
    data: dict
```

### Channel Access

Channels are stored in the graph's `channels` dictionary:

```python
from langgraph.graph import StateGraph

class State(TypedDict):
    messages: Annotated[list, operator.add]

graph = StateGraph(State)

# Access the channel
channel = graph.channels["messages"]
print(type(channel))  # <class 'BinaryOperatorAggregate'>
print(channel.ValueType)  # <class 'list'>
```

### State Schema Resolution

```python
from typing import Annotated
from typing_extensions import TypedDict
import operator

class State(TypedDict):
    # Field 1: LastValue (default)
    user_id: str

    # Field 2: BinaryOperatorAggregate (reducer)
    messages: Annotated[list[str], operator.add]

    # Field 3: Topic (explicit channel)
    events: Annotated[list[dict], Topic(dict, accumulate=True)]

    # Field 4: EphemeralValue (explicit channel)
    command: Annotated[str, EphemeralValue]

# LangGraph creates:
# channels = {
#     "user_id": LastValue(str),
#     "messages": BinaryOperatorAggregate(list, operator.add),
#     "events": Topic(dict, accumulate=True),
#     "command": EphemeralValue(str),
# }
```

---

## Custom Reducers

Reducers are binary functions that combine a current value with a new update. They power the `BinaryOperatorAggregate` channel.

### Reducer Signature

```python
def reducer(current: T, update: U) -> T:
    """
    Args:
        current: The existing value in the channel
        update: The new value being added

    Returns:
        The combined result
    """
```

### Built-in Operators

```python
import operator

# Addition (works for numbers, lists, strings)
operator.add
# Examples:
#   add(1, 2) -> 3
#   add([1], [2]) -> [1, 2]
#   add("hello", " world") -> "hello world"

# OR (bitwise/set union)
operator.or_
# Examples:
#   or_({1, 2}, {2, 3}) -> {1, 2, 3}
#   or_(0b1010, 0b1100) -> 0b1110

# Multiplication
operator.mul
# Examples:
#   mul(2, 3) -> 6
```

### Custom Reducer Examples

#### List with Unique Items

```python
def append_unique(current: list, update: str) -> list:
    """Add item only if not already in list."""
    if update not in current:
        return current + [update]
    return current

class State(TypedDict):
    tags: Annotated[list[str], append_unique]
```

#### Limited-Size List

```python
def append_max_10(current: list, update: Any) -> list:
    """Keep only last 10 items."""
    result = current + [update]
    return result[-10:]

class State(TypedDict):
    recent_events: Annotated[list[dict], append_max_10]
```

#### Dictionary Deep Merge

```python
def deep_merge(current: dict, update: dict) -> dict:
    """Recursively merge dictionaries."""
    result = current.copy()
    for key, value in update.items():
        if key in result and isinstance(result[key], dict) and isinstance(value, dict):
            result[key] = deep_merge(result[key], value)
        else:
            result[key] = value
    return result

class State(TypedDict):
    config: Annotated[dict, deep_merge]

# Usage:
# Step 1: {"config": {"db": {"host": "localhost"}}}
# Step 2: {"config": {"db": {"port": 5432}, "cache": True}}
# Result: {"config": {"db": {"host": "localhost", "port": 5432}, "cache": True}}
```

#### Counter Aggregation

```python
from collections import Counter

def merge_counters(current: Counter, update: Counter) -> Counter:
    """Combine two counters."""
    return current + update

class State(TypedDict):
    word_counts: Annotated[Counter, merge_counters]
```

#### Custom Class Aggregation

```python
from dataclasses import dataclass

@dataclass
class Stats:
    count: int = 0
    total: float = 0.0

    @property
    def average(self) -> float:
        return self.total / self.count if self.count > 0 else 0.0

def aggregate_stats(current: Stats, update: float) -> Stats:
    """Accumulate statistics."""
    return Stats(
        count=current.count + 1,
        total=current.total + update
    )

class State(TypedDict):
    statistics: Annotated[Stats, aggregate_stats]

# Usage:
# Input: {"statistics": 10.5}
# Input: {"statistics": 20.3}
# Result: Stats(count=2, total=30.8, average=15.4)
```

#### Conditional Reducer

```python
def max_value(current: int, update: int) -> int:
    """Keep the maximum value."""
    return max(current, update)

def min_value(current: int, update: int) -> int:
    """Keep the minimum value."""
    return min(current, update)

class State(TypedDict):
    max_score: Annotated[int, max_value]
    min_score: Annotated[int, min_value]
```

### Reducer Best Practices

1. **Immutability**: Don't modify `current` in place; return a new value
   ```python
   # Bad
   def bad_reducer(current: list, update: str) -> list:
       current.append(update)  # Mutates current!
       return current

   # Good
   def good_reducer(current: list, update: str) -> list:
       return current + [update]  # Returns new list
   ```

2. **Type Consistency**: Return type should match current type
   ```python
   def reducer(current: list[str], update: str) -> list[str]:
       return current + [update]  # Returns list[str]
   ```

3. **Handle Empty State**: First call uses type's default or first update
   ```python
   # BinaryOperatorAggregate tries typ() first
   # For list: current starts as []
   # For dict: current starts as {}
   # For int: current starts as 0
   ```

4. **Use Overwrite When Needed**: Allow replacing instead of reducing
   ```python
   from langgraph.types import Overwrite

   # In a node:
   return {"messages": Overwrite([])}  # Replace, don't append
   ```

---

## Advanced Channel Behaviors

### Concurrent Updates

Different channels handle concurrent updates differently:

| Channel | Concurrent Updates |
|---------|-------------------|
| LastValue | ERROR - raises InvalidUpdateError |
| AnyValue | OK - uses last value (assumes equal) |
| Topic | OK - collects all values |
| BinaryOperatorAggregate | OK - applies reducer to all |
| EphemeralValue (guard=True) | ERROR - raises InvalidUpdateError |
| EphemeralValue (guard=False) | OK - uses last value |
| UntrackedValue (guard=True) | ERROR - raises InvalidUpdateError |
| UntrackedValue (guard=False) | OK - uses last value |
| NamedBarrierValue | OK - collects named signals |

### Update Order

The order of updates in `update(values)` is **arbitrary**. Channels must handle updates in any order:

```python
# BinaryOperatorAggregate applies operator left-to-right on values list
# For commutative operators (add, or), order doesn't matter
# For non-commutative operators, behavior may vary

def non_commutative(a: str, b: str) -> str:
    return a + " -> " + b

# Updates ["A", "B", "C"] could produce:
# "A -> B -> C" or "A -> C -> B" or other permutations
# Don't rely on update order!
```

### Checkpointing Behavior

| Channel | Checkpointed? | Restored Value |
|---------|---------------|----------------|
| LastValue | Yes | Last value |
| AnyValue | Yes | Last value |
| Topic | Yes | All accumulated values |
| BinaryOperatorAggregate | Yes | Aggregated result |
| EphemeralValue | Yes | Current value (often MISSING) |
| UntrackedValue | **No** | Always MISSING |
| NamedBarrierValue | Yes | Set of received signals |

### Value Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                     Channel Lifecycle                        │
└─────────────────────────────────────────────────────────────┘

1. INITIALIZATION
   ├─ from_checkpoint(checkpoint) → New channel with restored state
   └─ __init__() → New empty channel

2. STEP EXECUTION
   ├─ is_available() → Check if value present
   ├─ get() → Read current value
   └─ (Node execution...)

3. STEP COMPLETION
   └─ update(values) → Apply all updates from step

4. POST-STEP HOOKS
   ├─ consume() → Clear ephemeral values, reset barriers
   └─ finish() → Transition AfterFinish channels

5. PERSISTENCE
   └─ checkpoint() → Serialize for storage

6. COPY/FORK
   └─ copy() → Duplicate for branches
```

### Empty Channels

Channels can be empty (no value available):

```python
from langgraph.errors import EmptyChannelError

# All channels start empty
channel = LastValue(str)

try:
    value = channel.get()
except EmptyChannelError:
    print("Channel is empty")

# Check without exception
if channel.is_available():
    value = channel.get()
```

### Channel Equality

Channels implement `__eq__` for comparison:

```python
from langgraph.channels import LastValue, Topic

# Same type and configuration
assert LastValue(str) == LastValue(str)
assert Topic(int, accumulate=True) == Topic(int, accumulate=True)

# Different configurations
assert Topic(int, accumulate=True) != Topic(int, accumulate=False)
```

---

## Summary

Channels are the low-level data flow primitives in LangGraph:

- **LastValue**: Default, last-write-wins, rejects concurrent updates
- **AnyValue**: Like LastValue but accepts concurrent equal updates
- **Topic**: Pub/sub, collects all updates into a sequence
- **BinaryOperatorAggregate**: Reduces updates with a binary operator
- **EphemeralValue**: Temporary, clears after each step
- **UntrackedValue**: Never checkpointed, for runtime-only data
- **NamedBarrierValue**: Synchronization, waits for all named signals

Most users interact with channels indirectly through state annotations:

```python
import operator
from typing import Annotated
from typing_extensions import TypedDict
from langgraph.channels import Topic, EphemeralValue

class State(TypedDict):
    # LastValue (default)
    user_id: str

    # BinaryOperatorAggregate (reducer)
    messages: Annotated[list, operator.add]

    # Topic (explicit)
    events: Annotated[list, Topic(dict)]

    # EphemeralValue (explicit)
    command: Annotated[str, EphemeralValue]
```

Understanding channels helps you:
- Choose the right channel for your data flow needs
- Write effective custom reducers
- Debug concurrent update errors
- Optimize checkpointing and performance
- Build advanced synchronization patterns
