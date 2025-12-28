# Channels API Reference

The Channels module provides the core abstraction for state management in LangGraph. Channels define how state is stored, updated, and checkpointed during graph execution.

## Overview

Channels are typed containers that manage state updates in a LangGraph graph. Each channel type implements different semantics for handling concurrent updates, persistence, and value availability.

All channels inherit from `BaseChannel` and implement a consistent interface for:
- Storing and retrieving values
- Handling concurrent updates
- Creating checkpoints for persistence
- Managing value availability

---

## Table of Contents

- [BaseChannel](#basechannel) - Abstract base class
- [LastValue](#lastvalue) - Single value storage
- [LastValueAfterFinish](#lastvalueafterfinish) - Delayed value availability
- [Topic](#topic) - PubSub message accumulation
- [BinaryOperatorAggregate](#binaryoperatoraggregate) - Aggregating updates with operators
- [EphemeralValue](#ephemeralvalue) - Temporary value storage
- [NamedBarrierValue](#namedbarriervalue) - Synchronization barrier
- [NamedBarrierValueAfterFinish](#namedbarriervalueafterfinish) - Delayed barrier
- [AnyValue](#anyvalue) - Permissive value storage
- [UntrackedValue](#untrackedvalue) - Non-persistent storage

---

## BaseChannel

**Signature:**
```python
class BaseChannel(Generic[Value, Update, Checkpoint], ABC):
    def __init__(self, typ: Any, key: str = "") -> None:
```

**Description:**
Abstract base class for all channels. Defines the interface that all channel implementations must follow. Channels are generic over three types:
- `Value`: The type of value returned by `get()`
- `Update`: The type of updates accepted by `update()`
- `Checkpoint`: The type used for serialization

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | Any | Yes | N/A | The type of the value stored in the channel |
| key | str | No | "" | The key identifier for this channel |

**Abstract Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> Any`
**Description:** The type of the value stored in the channel.
**Returns:** The value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> Any`
**Description:** The type of the update received by the channel.
**Returns:** The update type.

**Abstract Methods:**

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: Checkpoint | Any) -> Self`
**Description:** Return a new identical channel, optionally initialized from a checkpoint. If the checkpoint contains complex data structures, they should be copied.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | Checkpoint \| Any | The checkpoint data to restore from |

**Returns:** A new channel instance.

#### get
**Signature:** `def get(self) -> Value`
**Description:** Return the current value of the channel.
**Returns:** The current value.
**Raises:** `EmptyChannelError` if the channel is empty (never updated yet).

#### update
**Signature:** `def update(self, values: Sequence[Update]) -> bool`
**Description:** Update the channel's value with the given sequence of updates. The order of the updates in the sequence is arbitrary. This method is called by Pregel for all channels at the end of each step. If there are no updates, it is called with an empty sequence.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Update] | Sequence of updates to apply |

**Returns:** `True` if the channel was updated, `False` otherwise.
**Raises:** `InvalidUpdateError` if the sequence of updates is invalid.

**Concrete Methods:**

#### copy
**Signature:** `def copy(self) -> Self`
**Description:** Return a copy of the channel. By default, delegates to `checkpoint()` and `from_checkpoint()`. Subclasses can override this method with a more efficient implementation.
**Returns:** A copy of the channel.

#### checkpoint
**Signature:** `def checkpoint(self) -> Checkpoint | Any`
**Description:** Return a serializable representation of the channel's current state.
**Returns:** The checkpoint data.
**Raises:** `EmptyChannelError` if the channel is empty (never updated yet), or doesn't support checkpoints.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Return `True` if the channel is available (not empty), `False` otherwise. Subclasses should override this method to provide a more efficient implementation than calling `get()` and catching `EmptyChannelError`.
**Returns:** Boolean indicating availability.

#### consume
**Signature:** `def consume(self) -> bool`
**Description:** Notify the channel that a subscribed task ran. By default, no-op. A channel can use this method to modify its state, preventing the value from being consumed again.
**Returns:** `True` if the channel was updated, `False` otherwise.

#### finish
**Signature:** `def finish(self) -> bool`
**Description:** Notify the channel that the Pregel run is finishing. By default, no-op. A channel can use this method to modify its state, preventing finish.
**Returns:** `True` if the channel was updated, `False` otherwise.

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/base.py`

---

## LastValue

**Signature:**
```python
class LastValue(Generic[Value], BaseChannel[Value, Value, Value]):
    def __init__(self, typ: Any, key: str = "") -> None:
```

**Description:**
Stores the last value received, can receive at most one value per step. This is the most common channel type used for simple state values. If multiple values are written to this channel in a single step, an `InvalidUpdateError` is raised.

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | Any | Yes | N/A | The type of the value stored in the channel |
| key | str | No | "" | The key identifier for this channel |

**Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> type[Value]`
**Description:** The type of the value stored in the channel.
**Returns:** The value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> type[Value]`
**Description:** The type of the update received by the channel.
**Returns:** The update type (same as ValueType).

**Methods:**

#### __eq__
**Signature:** `def __eq__(self, value: object) -> bool`
**Description:** Check if another object is a LastValue channel.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| value | object | Object to compare with |

**Returns:** `True` if the value is a LastValue instance, `False` otherwise.

#### copy
**Signature:** `def copy(self) -> Self`
**Description:** Return a copy of the channel.
**Returns:** A new LastValue instance with the same value.

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: Value) -> Self`
**Description:** Return a new channel initialized from a checkpoint.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | Value | The checkpoint data to restore from |

**Returns:** A new LastValue instance.

#### update
**Signature:** `def update(self, values: Sequence[Value]) -> bool`
**Description:** Update the channel with a new value. Only accepts a single value per step.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Value] | Sequence of values (must contain exactly 0 or 1 element) |

**Returns:** `True` if the channel was updated, `False` if no values provided.
**Raises:** `InvalidUpdateError` if more than one value is provided (error code: `INVALID_CONCURRENT_GRAPH_UPDATE`).

#### get
**Signature:** `def get(self) -> Value`
**Description:** Return the current value of the channel.
**Returns:** The stored value.
**Raises:** `EmptyChannelError` if the channel is empty.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Check if the channel has a value.
**Returns:** `True` if a value is stored, `False` otherwise.

#### checkpoint
**Signature:** `def checkpoint(self) -> Value`
**Description:** Return the current value for checkpointing.
**Returns:** The stored value.

**Example:**
```python
from langgraph.channels import LastValue

# Create a channel for storing a string
name_channel = LastValue(str)

# Update the channel
name_channel.update(["Alice"])

# Get the value
assert name_channel.get() == "Alice"

# Check availability
assert name_channel.is_available() == True

# Attempting to update with multiple values raises an error
try:
    name_channel.update(["Bob", "Charlie"])
except InvalidUpdateError as e:
    print(f"Error: {e}")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/last_value.py:20-79`

---

## LastValueAfterFinish

**Signature:**
```python
class LastValueAfterFinish(Generic[Value], BaseChannel[Value, Value, tuple[Value, bool]]):
    def __init__(self, typ: Any, key: str = "") -> None:
```

**Description:**
Stores the last value received, but only makes it available after `finish()` is called. Once made available, calling `consume()` clears the value. This is useful for output channels that should only be read at the end of a graph execution.

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | Any | Yes | N/A | The type of the value stored in the channel |
| key | str | No | "" | The key identifier for this channel |

**Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> type[Value]`
**Description:** The type of the value stored in the channel.
**Returns:** The value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> type[Value]`
**Description:** The type of the update received by the channel.
**Returns:** The update type (same as ValueType).

**Methods:**

#### __eq__
**Signature:** `def __eq__(self, value: object) -> bool`
**Description:** Check if another object is a LastValueAfterFinish channel.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| value | object | Object to compare with |

**Returns:** `True` if the value is a LastValueAfterFinish instance, `False` otherwise.

#### checkpoint
**Signature:** `def checkpoint(self) -> tuple[Value | Any, bool] | Any`
**Description:** Return the current state for checkpointing, including the value and finished flag.
**Returns:** A tuple of (value, finished) or MISSING if empty.

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: tuple[Value | Any, bool] | Any) -> Self`
**Description:** Return a new channel initialized from a checkpoint.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | tuple[Value \| Any, bool] \| Any | The checkpoint data containing (value, finished) |

**Returns:** A new LastValueAfterFinish instance.

#### update
**Signature:** `def update(self, values: Sequence[Value | Any]) -> bool`
**Description:** Update the channel with a new value and reset the finished flag.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Value \| Any] | Sequence of values |

**Returns:** `True` if the channel was updated, `False` if no values provided.

#### consume
**Signature:** `def consume(self) -> bool`
**Description:** If the channel is finished, clear the value and reset the finished flag.
**Returns:** `True` if the channel was cleared, `False` otherwise.

#### finish
**Signature:** `def finish(self) -> bool`
**Description:** Mark the channel as finished, making the value available.
**Returns:** `True` if the channel was marked as finished, `False` if already finished or empty.

#### get
**Signature:** `def get(self) -> Value`
**Description:** Return the current value of the channel, only if finished.
**Returns:** The stored value.
**Raises:** `EmptyChannelError` if the channel is empty or not finished.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Check if the channel has a value and is finished.
**Returns:** `True` if value is stored and finished, `False` otherwise.

**Example:**
```python
from langgraph.channels import LastValueAfterFinish

# Create a channel for final output
output_channel = LastValueAfterFinish(str)

# Update the channel
output_channel.update(["Processing..."])

# Value is not available yet
assert output_channel.is_available() == False

# Mark as finished
output_channel.finish()

# Now the value is available
assert output_channel.get() == "Processing..."

# Consume the value
output_channel.consume()

# Value is no longer available
assert output_channel.is_available() == False
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/last_value.py:81-152`

---

## Topic

**Signature:**
```python
class Topic(Generic[Value], BaseChannel[Sequence[Value], Value | list[Value], list[Value]]):
    def __init__(self, typ: type[Value], accumulate: bool = False) -> None:
```

**Description:**
A configurable PubSub Topic channel. Accumulates values into a sequence. Can be configured to accumulate values across steps or clear after each step. Accepts individual values or lists of values, which are flattened into the sequence.

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | type[Value] | Yes | N/A | The type of individual values in the topic |
| accumulate | bool | No | False | Whether to accumulate values across steps. If `False`, the channel will be emptied after each step |

**Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> Any`
**Description:** The type of the value stored in the channel.
**Returns:** `Sequence[typ]` where typ is the value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> Any`
**Description:** The type of the update received by the channel.
**Returns:** `typ | list[typ]` - can accept individual values or lists.

**Methods:**

#### __eq__
**Signature:** `def __eq__(self, value: object) -> bool`
**Description:** Check if another object is a Topic channel with the same accumulate setting.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| value | object | Object to compare with |

**Returns:** `True` if the value is a Topic instance with matching accumulate flag, `False` otherwise.

#### copy
**Signature:** `def copy(self) -> Self`
**Description:** Return a copy of the channel including its current values.
**Returns:** A new Topic instance with copied values.

#### checkpoint
**Signature:** `def checkpoint(self) -> list[Value]`
**Description:** Return the current list of values for checkpointing.
**Returns:** The list of accumulated values.

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: list[Value]) -> Self`
**Description:** Return a new channel initialized from a checkpoint.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | list[Value] | The checkpoint data containing the list of values |

**Returns:** A new Topic instance.

#### update
**Signature:** `def update(self, values: Sequence[Value | list[Value]]) -> bool`
**Description:** Update the channel with new values. If `accumulate=False`, clears existing values first. Accepts both individual values and lists of values, which are flattened.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Value \| list[Value]] | Sequence of values or lists of values to add |

**Returns:** `True` if the channel was updated, `False` otherwise.

#### get
**Signature:** `def get(self) -> Sequence[Value]`
**Description:** Return the current list of values.
**Returns:** A list copy of the accumulated values.
**Raises:** `EmptyChannelError` if no values are stored.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Check if the channel has any values.
**Returns:** `True` if values are stored, `False` otherwise.

**Example:**
```python
from langgraph.channels import Topic

# Create a non-accumulating topic
messages = Topic(str, accumulate=False)

# Add some messages
messages.update(["Hello", "World"])
assert messages.get() == ["Hello", "World"]

# Update clears previous values (accumulate=False)
messages.update(["New message"])
assert messages.get() == ["New message"]

# Create an accumulating topic
all_messages = Topic(str, accumulate=True)

# Add messages
all_messages.update(["First"])
all_messages.update(["Second", "Third"])

# All messages are accumulated
assert all_messages.get() == ["First", "Second", "Third"]

# Can also pass lists
all_messages.update([["Fourth", "Fifth"]])
assert "Fourth" in all_messages.get()
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/topic.py:23-95`

---

## BinaryOperatorAggregate

**Signature:**
```python
class BinaryOperatorAggregate(Generic[Value], BaseChannel[Value, Value, Value]):
    def __init__(self, typ: type[Value], operator: Callable[[Value, Value], Value]) -> None:
```

**Description:**
Stores the result of applying a binary operator to the current value and each new value. This is useful for aggregating updates using operations like addition, concatenation, or custom merge logic. Supports the `Overwrite` special type to replace the current value instead of applying the operator.

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | type[Value] | Yes | N/A | The type of the value stored in the channel |
| operator | Callable[[Value, Value], Value] | Yes | N/A | Binary operator function to apply for aggregation |

**Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> type[Value]`
**Description:** The type of the value stored in the channel.
**Returns:** The value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> type[Value]`
**Description:** The type of the update received by the channel.
**Returns:** The update type (same as ValueType).

**Methods:**

#### __eq__
**Signature:** `def __eq__(self, value: object) -> bool`
**Description:** Check if another object is a BinaryOperatorAggregate with the same operator. Lambda functions are considered equal.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| value | object | Object to compare with |

**Returns:** `True` if the value is a BinaryOperatorAggregate with the same operator, `False` otherwise.

#### copy
**Signature:** `def copy(self) -> Self`
**Description:** Return a copy of the channel.
**Returns:** A new BinaryOperatorAggregate instance with the same value.

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: Value) -> Self`
**Description:** Return a new channel initialized from a checkpoint.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | Value | The checkpoint data to restore from |

**Returns:** A new BinaryOperatorAggregate instance.

#### update
**Signature:** `def update(self, values: Sequence[Value]) -> bool`
**Description:** Update the channel by applying the operator to each new value. If an `Overwrite` value is encountered, it replaces the current value. Only one `Overwrite` is allowed per super-step.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Value] | Sequence of values to aggregate |

**Returns:** `True` if the channel was updated, `False` if no values provided.
**Raises:** `InvalidUpdateError` if more than one `Overwrite` value is provided (error code: `INVALID_CONCURRENT_GRAPH_UPDATE`).

#### get
**Signature:** `def get(self) -> Value`
**Description:** Return the current aggregated value.
**Returns:** The stored value.
**Raises:** `EmptyChannelError` if the channel is empty.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Check if the channel has a value.
**Returns:** `True` if a value is stored, `False` otherwise.

#### checkpoint
**Signature:** `def checkpoint(self) -> Value`
**Description:** Return the current value for checkpointing.
**Returns:** The stored value.

**Example:**
```python
from langgraph.channels import BinaryOperatorAggregate
import operator

# Create a counter channel that sums integers
counter = BinaryOperatorAggregate(int, operator.add)

# Add some values
counter.update([5])
assert counter.get() == 5

counter.update([3, 7])
assert counter.get() == 15  # 5 + 3 + 7

# Create a list concatenation channel
items = BinaryOperatorAggregate(list, operator.add)
items.update([[1, 2]])
items.update([[3, 4], [5]])
assert items.get() == [1, 2, 3, 4, 5]

# Using Overwrite to replace the value
from langgraph.types import Overwrite

counter.update([Overwrite(100)])
assert counter.get() == 100
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/binop.py:41-135`

---

## EphemeralValue

**Signature:**
```python
class EphemeralValue(Generic[Value], BaseChannel[Value, Value, Value]):
    def __init__(self, typ: Any, guard: bool = True) -> None:
```

**Description:**
Stores the value received in the step immediately preceding, then clears it after. This is useful for temporary values that should only be available for one step. The `guard` parameter controls whether multiple values per step are allowed.

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | Any | Yes | N/A | The type of the value stored in the channel |
| guard | bool | No | True | If `True`, only one value per step is allowed. If `False`, stores any one of multiple values |

**Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> type[Value]`
**Description:** The type of the value stored in the channel.
**Returns:** The value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> type[Value]`
**Description:** The type of the update received by the channel.
**Returns:** The update type (same as ValueType).

**Methods:**

#### __eq__
**Signature:** `def __eq__(self, value: object) -> bool`
**Description:** Check if another object is an EphemeralValue with the same guard setting.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| value | object | Object to compare with |

**Returns:** `True` if the value is an EphemeralValue with matching guard flag, `False` otherwise.

#### copy
**Signature:** `def copy(self) -> Self`
**Description:** Return a copy of the channel.
**Returns:** A new EphemeralValue instance with the same value.

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: Value) -> Self`
**Description:** Return a new channel initialized from a checkpoint.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | Value | The checkpoint data to restore from |

**Returns:** A new EphemeralValue instance.

#### update
**Signature:** `def update(self, values: Sequence[Value]) -> bool`
**Description:** Update the channel with a new value. If no values are provided, clears the current value. If `guard=True`, only one value is allowed per step.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Value] | Sequence of values |

**Returns:** `True` if the channel was updated, `False` otherwise.
**Raises:** `InvalidUpdateError` if `guard=True` and more than one value is provided.

#### get
**Signature:** `def get(self) -> Value`
**Description:** Return the current value of the channel.
**Returns:** The stored value.
**Raises:** `EmptyChannelError` if the channel is empty.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Check if the channel has a value.
**Returns:** `True` if a value is stored, `False` otherwise.

#### checkpoint
**Signature:** `def checkpoint(self) -> Value`
**Description:** Return the current value for checkpointing.
**Returns:** The stored value.

**Example:**
```python
from langgraph.channels import EphemeralValue

# Create an ephemeral channel for temporary messages
temp_message = EphemeralValue(str)

# Set a value
temp_message.update(["Temporary data"])
assert temp_message.get() == "Temporary data"

# Update with no values clears it
temp_message.update([])
assert temp_message.is_available() == False

# With guard=False, multiple values are allowed (last one wins)
lenient_channel = EphemeralValue(str, guard=False)
lenient_channel.update(["First", "Second", "Third"])
assert lenient_channel.get() == "Third"
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/ephemeral_value.py:15-80`

---

## NamedBarrierValue

**Signature:**
```python
class NamedBarrierValue(Generic[Value], BaseChannel[Value, Value, set[Value]]):
    def __init__(self, typ: type[Value], names: set[Value]) -> None:
```

**Description:**
A channel that waits until all named values are received before making the value available. This is useful for synchronization, where execution should wait for multiple named signals before proceeding. The channel becomes available when all expected names have been received, and `consume()` resets it.

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | type[Value] | Yes | N/A | The type of the values (typically str for named signals) |
| names | set[Value] | Yes | N/A | The set of all expected named values |

**Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> type[Value]`
**Description:** The type of the value stored in the channel.
**Returns:** The value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> type[Value]`
**Description:** The type of the update received by the channel.
**Returns:** The update type (same as ValueType).

**Methods:**

#### __eq__
**Signature:** `def __eq__(self, value: object) -> bool`
**Description:** Check if another object is a NamedBarrierValue with the same names.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| value | object | Object to compare with |

**Returns:** `True` if the value is a NamedBarrierValue with matching names, `False` otherwise.

#### copy
**Signature:** `def copy(self) -> Self`
**Description:** Return a copy of the channel including seen names.
**Returns:** A new NamedBarrierValue instance.

#### checkpoint
**Signature:** `def checkpoint(self) -> set[Value]`
**Description:** Return the set of seen names for checkpointing.
**Returns:** The set of names that have been received.

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: set[Value]) -> Self`
**Description:** Return a new channel initialized from a checkpoint.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | set[Value] | The checkpoint data containing seen names |

**Returns:** A new NamedBarrierValue instance.

#### update
**Signature:** `def update(self, values: Sequence[Value]) -> bool`
**Description:** Update the channel by adding received names to the seen set.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Value] | Sequence of named values to mark as received |

**Returns:** `True` if new names were added, `False` otherwise.
**Raises:** `InvalidUpdateError` if a value is not in the expected names set.

#### get
**Signature:** `def get(self) -> Value`
**Description:** Return None when all names have been received.
**Returns:** `None`
**Raises:** `EmptyChannelError` if not all names have been received yet.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Check if all expected names have been received.
**Returns:** `True` if all names received, `False` otherwise.

#### consume
**Signature:** `def consume(self) -> bool`
**Description:** Reset the barrier by clearing all seen names.
**Returns:** `True` if all names were received and cleared, `False` otherwise.

**Example:**
```python
from langgraph.channels import NamedBarrierValue

# Create a barrier waiting for three named signals
barrier = NamedBarrierValue(str, {"task_a", "task_b", "task_c"})

# Initially not available
assert barrier.is_available() == False

# Receive some signals
barrier.update(["task_a"])
assert barrier.is_available() == False

barrier.update(["task_b", "task_c"])
# Now all signals received
assert barrier.is_available() == True

# Get returns None (barrier is just for synchronization)
assert barrier.get() is None

# Consume resets the barrier
barrier.consume()
assert barrier.is_available() == False
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/named_barrier_value.py:13-82`

---

## NamedBarrierValueAfterFinish

**Signature:**
```python
class NamedBarrierValueAfterFinish(Generic[Value], BaseChannel[Value, Value, set[Value]]):
    def __init__(self, typ: type[Value], names: set[Value]) -> None:
```

**Description:**
A channel that waits until all named values are received before making the value ready to be made available. It is only made available after `finish()` is called. This combines barrier synchronization with delayed availability, useful for final synchronization before graph completion.

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | type[Value] | Yes | N/A | The type of the values (typically str for named signals) |
| names | set[Value] | Yes | N/A | The set of all expected named values |

**Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> type[Value]`
**Description:** The type of the value stored in the channel.
**Returns:** The value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> type[Value]`
**Description:** The type of the update received by the channel.
**Returns:** The update type (same as ValueType).

**Methods:**

#### __eq__
**Signature:** `def __eq__(self, value: object) -> bool`
**Description:** Check if another object is a NamedBarrierValueAfterFinish with the same names.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| value | object | Object to compare with |

**Returns:** `True` if the value is a NamedBarrierValueAfterFinish with matching names, `False` otherwise.

#### copy
**Signature:** `def copy(self) -> Self`
**Description:** Return a copy of the channel including seen names and finished state.
**Returns:** A new NamedBarrierValueAfterFinish instance.

#### checkpoint
**Signature:** `def checkpoint(self) -> tuple[set[Value], bool]`
**Description:** Return the seen names and finished flag for checkpointing.
**Returns:** A tuple of (seen_names, finished).

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: tuple[set[Value], bool]) -> Self`
**Description:** Return a new channel initialized from a checkpoint.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | tuple[set[Value], bool] | The checkpoint data containing (seen_names, finished) |

**Returns:** A new NamedBarrierValueAfterFinish instance.

#### update
**Signature:** `def update(self, values: Sequence[Value]) -> bool`
**Description:** Update the channel by adding received names to the seen set.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Value] | Sequence of named values to mark as received |

**Returns:** `True` if new names were added, `False` otherwise.
**Raises:** `InvalidUpdateError` if a value is not in the expected names set.

#### get
**Signature:** `def get(self) -> Value`
**Description:** Return None when finished and all names have been received.
**Returns:** `None`
**Raises:** `EmptyChannelError` if not finished or not all names have been received.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Check if finished and all expected names have been received.
**Returns:** `True` if finished and all names received, `False` otherwise.

#### consume
**Signature:** `def consume(self) -> bool`
**Description:** Reset the barrier by clearing all seen names and finished flag.
**Returns:** `True` if the barrier was available and got reset, `False` otherwise.

#### finish
**Signature:** `def finish(self) -> bool`
**Description:** Mark the channel as finished if all names have been received.
**Returns:** `True` if the channel was marked as finished, `False` if already finished or not all names received.

**Example:**
```python
from langgraph.channels import NamedBarrierValueAfterFinish

# Create a barrier for final synchronization
final_barrier = NamedBarrierValueAfterFinish(str, {"cleanup", "save", "notify"})

# Receive signals
final_barrier.update(["cleanup", "save"])
# Not available yet (missing "notify")
assert final_barrier.is_available() == False

final_barrier.update(["notify"])
# All received but not finished yet
assert final_barrier.is_available() == False

# Mark as finished
final_barrier.finish()
assert final_barrier.is_available() == True

# Consume resets everything
final_barrier.consume()
assert final_barrier.is_available() == False
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/named_barrier_value.py:84-168`

---

## AnyValue

**Signature:**
```python
class AnyValue(Generic[Value], BaseChannel[Value, Value, Value]):
    def __init__(self, typ: Any, key: str = "") -> None:
```

**Description:**
Stores the last value received, assumes that if multiple values are received, they are all equal. This is a more permissive version of `LastValue` that doesn't raise an error when multiple values are written in a single step - it simply takes the last one, assuming all values are equivalent.

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | Any | Yes | N/A | The type of the value stored in the channel |
| key | str | No | "" | The key identifier for this channel |

**Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> type[Value]`
**Description:** The type of the value stored in the channel.
**Returns:** The value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> type[Value]`
**Description:** The type of the update received by the channel.
**Returns:** The update type (same as ValueType).

**Methods:**

#### __eq__
**Signature:** `def __eq__(self, value: object) -> bool`
**Description:** Check if another object is an AnyValue channel.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| value | object | Object to compare with |

**Returns:** `True` if the value is an AnyValue instance, `False` otherwise.

#### copy
**Signature:** `def copy(self) -> Self`
**Description:** Return a copy of the channel.
**Returns:** A new AnyValue instance with the same value.

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: Value) -> Self`
**Description:** Return a new channel initialized from a checkpoint.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | Value | The checkpoint data to restore from |

**Returns:** A new AnyValue instance.

#### update
**Signature:** `def update(self, values: Sequence[Value]) -> bool`
**Description:** Update the channel with new values. If empty sequence is provided, clears the value. Otherwise, stores the last value from the sequence.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Value] | Sequence of values |

**Returns:** `True` if the channel was updated, `False` if no change occurred.

#### get
**Signature:** `def get(self) -> Value`
**Description:** Return the current value of the channel.
**Returns:** The stored value.
**Raises:** `EmptyChannelError` if the channel is empty.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Check if the channel has a value.
**Returns:** `True` if a value is stored, `False` otherwise.

#### checkpoint
**Signature:** `def checkpoint(self) -> Value`
**Description:** Return the current value for checkpointing.
**Returns:** The stored value.

**Example:**
```python
from langgraph.channels import AnyValue

# Create a channel that accepts multiple equal values
config = AnyValue(dict)

# Update with multiple values (assumes they're equal, takes last one)
config.update([
    {"setting": "value1"},
    {"setting": "value1"},
    {"setting": "value1"}
])
assert config.get() == {"setting": "value1"}

# Unlike LastValue, this doesn't raise an error
# It assumes all concurrent updates are equivalent

# Clear with empty sequence
config.update([])
assert config.is_available() == False
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/any_value.py:15-73`

---

## UntrackedValue

**Signature:**
```python
class UntrackedValue(Generic[Value], BaseChannel[Value, Value, Value]):
    def __init__(self, typ: type[Value], guard: bool = True) -> None:
```

**Description:**
Stores the last value received, but is never checkpointed. This is useful for values that should not be persisted, such as database connections, file handles, or other runtime-specific resources. The `guard` parameter controls whether multiple values per step are allowed.

**Constructor Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| typ | type[Value] | Yes | N/A | The type of the value stored in the channel |
| guard | bool | No | True | If `True`, only one value per step is allowed. If `False`, stores any one of multiple values |

**Properties:**

#### ValueType
**Signature:** `@property def ValueType(self) -> type[Value]`
**Description:** The type of the value stored in the channel.
**Returns:** The value type.

#### UpdateType
**Signature:** `@property def UpdateType(self) -> type[Value]`
**Description:** The type of the update received by the channel.
**Returns:** The update type (same as ValueType).

**Methods:**

#### __eq__
**Signature:** `def __eq__(self, value: object) -> bool`
**Description:** Check if another object is an UntrackedValue with the same guard setting.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| value | object | Object to compare with |

**Returns:** `True` if the value is an UntrackedValue with matching guard flag, `False` otherwise.

#### copy
**Signature:** `def copy(self) -> Self`
**Description:** Return a copy of the channel.
**Returns:** A new UntrackedValue instance with the same value.

#### checkpoint
**Signature:** `def checkpoint(self) -> Value | Any`
**Description:** Always returns MISSING since this channel is never checkpointed.
**Returns:** MISSING (sentinel value).

#### from_checkpoint
**Signature:** `def from_checkpoint(self, checkpoint: Value) -> Self`
**Description:** Return a new empty channel, ignoring the checkpoint.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| checkpoint | Value | Ignored - untracked channels don't restore from checkpoints |

**Returns:** A new empty UntrackedValue instance.

#### update
**Signature:** `def update(self, values: Sequence[Value]) -> bool`
**Description:** Update the channel with a new value. If `guard=True`, only one value is allowed per step.
**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| values | Sequence[Value] | Sequence of values |

**Returns:** `True` if the channel was updated, `False` if no values provided.
**Raises:** `InvalidUpdateError` if `guard=True` and more than one value is provided.

#### get
**Signature:** `def get(self) -> Value`
**Description:** Return the current value of the channel.
**Returns:** The stored value.
**Raises:** `EmptyChannelError` if the channel is empty.

#### is_available
**Signature:** `def is_available(self) -> bool`
**Description:** Check if the channel has a value.
**Returns:** `True` if a value is stored, `False` otherwise.

**Example:**
```python
from langgraph.channels import UntrackedValue

# Create a channel for a database connection (not persisted)
db_connection = UntrackedValue(object)

# Store a connection object
class DatabaseConnection:
    def __init__(self, url):
        self.url = url

conn = DatabaseConnection("postgresql://localhost/mydb")
db_connection.update([conn])

# Value is available in memory
assert db_connection.get() == conn

# But checkpoint returns MISSING (not persisted)
checkpoint = db_connection.checkpoint()
# checkpoint is MISSING

# Restoring from checkpoint creates empty channel
restored = db_connection.from_checkpoint({"some": "data"})
assert restored.is_available() == False

# With guard=False, multiple values allowed
cache = UntrackedValue(dict, guard=False)
cache.update([{"a": 1}, {"b": 2}, {"c": 3}])
assert cache.get() == {"c": 3}  # Last value wins
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/untracked_value.py:15-74`

---

## Channel Selection Guide

Choosing the right channel type depends on your requirements:

| Channel Type | Use When | Key Features |
|--------------|----------|--------------|
| **LastValue** | You need simple, single-value storage | Most common; strict single-value per step |
| **LastValueAfterFinish** | Value should only be available at end | Delayed availability; auto-clears on consume |
| **Topic** | You need to collect multiple messages | PubSub pattern; optional accumulation |
| **BinaryOperatorAggregate** | You need to aggregate updates | Custom merge logic; supports overwrite |
| **EphemeralValue** | Value should only exist for one step | Temporary storage; auto-clears |
| **NamedBarrierValue** | You need synchronization on named signals | Barrier pattern; resets on consume |
| **NamedBarrierValueAfterFinish** | You need final synchronization | Barrier + delayed availability |
| **AnyValue** | Multiple concurrent updates are expected to be equal | Permissive; no error on multiple values |
| **UntrackedValue** | Value should not be persisted | Never checkpointed; runtime-only |

## Common Patterns

### Aggregating Numeric Values
```python
from langgraph.channels import BinaryOperatorAggregate
import operator

# Sum integers
total = BinaryOperatorAggregate(int, operator.add)

# Multiply floats
product = BinaryOperatorAggregate(float, operator.mul)
```

### Message Accumulation
```python
from langgraph.channels import Topic

# Collect all messages in a step
messages = Topic(str, accumulate=False)

# Accumulate all messages across steps
all_messages = Topic(str, accumulate=True)
```

### Synchronization Barriers
```python
from langgraph.channels import NamedBarrierValue

# Wait for all workers to complete
workers_done = NamedBarrierValue(str, {"worker_1", "worker_2", "worker_3"})
```

### Runtime-Only State
```python
from langgraph.channels import UntrackedValue

# Store connections, file handles, etc.
db_conn = UntrackedValue(DatabaseConnection)
file_handle = UntrackedValue(FileHandle)
```

## Error Handling

All channels can raise the following exceptions:

- **EmptyChannelError**: Raised by `get()` when the channel has no value
- **InvalidUpdateError**: Raised by `update()` when updates are invalid (e.g., multiple values to LastValue, unknown names to NamedBarrierValue, multiple Overwrites to BinaryOperatorAggregate)

Example:
```python
from langgraph.channels import LastValue
from langgraph.errors import EmptyChannelError, InvalidUpdateError

channel = LastValue(str)

try:
    value = channel.get()
except EmptyChannelError:
    print("Channel is empty")

try:
    channel.update(["value1", "value2"])
except InvalidUpdateError as e:
    print(f"Invalid update: {e}")
```

## Type Safety

All channels are generic and support type hints:

```python
from langgraph.channels import LastValue, Topic
from typing import TypedDict

class Message(TypedDict):
    role: str
    content: str

# Type-safe channels
current_message: LastValue[Message] = LastValue(Message)
message_history: Topic[Message] = Topic(Message, accumulate=True)
```

---

**Module Source:** `/home/user/langgraph/libs/langgraph/langgraph/channels/`
