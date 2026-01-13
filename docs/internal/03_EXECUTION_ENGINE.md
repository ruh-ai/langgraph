# LangGraph Execution Engine (Pregel)

## Table of Contents
1. [Overview](#overview)
2. [Pregel Model](#pregel-model)
3. [Pregel Class](#pregel-class)
4. [Execution Loop](#execution-loop)
5. [Supersteps](#supersteps)
6. [Task Scheduling](#task-scheduling)
7. [State Management](#state-management)
8. [Streaming](#streaming)
9. [Remote Execution](#remote-execution)
10. [Interrupts & Resumption](#interrupts--resumption)

---

## Overview

The LangGraph execution engine is based on the **Pregel** model (also known as Bulk Synchronous Parallel or BSP model). Pregel is a vertex-centric computation framework originally designed by Google for large-scale graph processing. LangGraph adapts this model for orchestrating stateful, multi-actor workflows.

**Core Concepts:**
- **Actors (Nodes)**: Computational units (`PregelNode`) that read from and write to channels
- **Channels**: Communication primitives for passing data between actors
- **Supersteps**: Discrete execution phases where actors run in parallel
- **Checkpointing**: Persistent state snapshots for durability and time-travel

**Key Files:**
- `/libs/langgraph/langgraph/pregel/main.py` - Main Pregel class and graph orchestration
- `/libs/langgraph/langgraph/pregel/_loop.py` - Execution loop (SyncPregelLoop, AsyncPregelLoop)
- `/libs/langgraph/langgraph/pregel/_algo.py` - Task preparation and write application algorithms
- `/libs/langgraph/langgraph/pregel/_runner.py` - Concurrent task execution (PregelRunner)
- `/libs/langgraph/langgraph/pregel/remote.py` - Remote graph execution (RemoteGraph)

---

## Pregel Model

The Pregel model organizes computation into discrete **supersteps**. Each superstep consists of three phases:

```
┌─────────────────────────────────────────────────────────────┐
│                         SUPERSTEP N                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. PLAN                                                    │
│     ┌─────────────────────────────────────────────────┐    │
│     │ • Determine which nodes to execute              │    │
│     │ • Based on updated channels from previous step  │    │
│     │ • Create PregelExecutableTask for each node     │    │
│     └─────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  2. EXECUTE                                                 │
│     ┌─────────────────────────────────────────────────┐    │
│     │ • Run selected nodes in parallel                │    │
│     │ • Each node reads from channels (immutable)     │    │
│     │ • Each node writes to local buffer              │    │
│     │ • Wait for all to complete or one to fail       │    │
│     └─────────────────────────────────────────────────┘    │
│                           ↓                                 │
│  3. UPDATE                                                  │
│     ┌─────────────────────────────────────────────────┐    │
│     │ • Apply all writes to channels atomically       │    │
│     │ • Update channel versions                       │    │
│     │ • Save checkpoint (if checkpointer enabled)     │    │
│     │ • Check for interrupts                          │    │
│     └─────────────────────────────────────────────────┘    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           ↓
                  Next Superstep or Done
```

**Key Properties:**
1. **Isolation**: Channel values are immutable during execution phase
2. **Parallelism**: All selected nodes run concurrently within a superstep
3. **Atomicity**: All writes are applied together at the end of the superstep
4. **Determinism**: Same input and channel state produces same output

**Comparison to Traditional Models:**

| Aspect | Pregel (LangGraph) | Traditional DAG | Actor Model |
|--------|-------------------|-----------------|-------------|
| Execution | Supersteps (BSP) | Topological order | Asynchronous messaging |
| Parallelism | Within superstep | Task-level | Unbounded |
| State | Channels (shared) | Task outputs | Actor-local |
| Coordination | Barrier at step end | Dependencies | Message passing |
| Determinism | High | High | Low |

---

## Pregel Class

The `Pregel` class is the main executor for LangGraph applications. It manages the runtime behavior, coordinating nodes, channels, and execution flow.

### Class Definition

```python
class Pregel(PregelProtocol[StateT, ContextT, InputT, OutputT]):
    """Pregel manages the runtime behavior for LangGraph applications."""

    nodes: dict[str, PregelNode]              # Node definitions
    channels: dict[str, BaseChannel | ManagedValueSpec]  # Communication channels
    input_channels: str | Sequence[str]       # Input channel names
    output_channels: str | Sequence[str]      # Output channel names
    stream_channels: str | Sequence[str] | None  # Channels to stream

    interrupt_after_nodes: All | Sequence[str]   # Interrupt after these nodes
    interrupt_before_nodes: All | Sequence[str]  # Interrupt before these nodes

    checkpointer: Checkpointer = None         # Checkpoint saver
    store: BaseStore | None = None            # Memory store
    cache: BaseCache | None = None            # Result cache

    retry_policy: Sequence[RetryPolicy] = ()  # Retry configuration
    cache_policy: CachePolicy | None = None   # Cache configuration

    step_timeout: float | None = None         # Max time per step
    debug: bool                               # Enable debug output
```

### Key Methods

**Execution Methods:**
- `invoke(input, config)` - Execute graph synchronously, return final output
- `stream(input, config, stream_mode)` - Execute graph, yield intermediate outputs
- `batch(inputs, config)` - Execute graph for multiple inputs in parallel
- `ainvoke()`, `astream()`, `abatch()` - Async versions

**State Methods:**
- `get_state(config)` - Get current graph state at a checkpoint
- `update_state(config, values, as_node)` - Manually update graph state
- `get_state_history(config)` - Iterate through checkpoint history

**Introspection:**
- `get_graph(config)` - Get drawable graph representation
- `get_subgraphs()` - Enumerate nested subgraphs

### Initialization Flow

```
Pregel.__init__()
    ↓
Build nodes (NodeBuilder → PregelNode)
    ↓
Validate graph structure
    ↓
Create TASKS channel (Topic[Send])
    ↓
Compute trigger_to_nodes mapping
```

---

## Execution Loop

The execution loop is implemented in `_loop.py` with two variants:
- **SyncPregelLoop**: Synchronous execution with threading
- **AsyncPregelLoop**: Asynchronous execution with asyncio

### Loop Lifecycle

```
┌─────────────────────────────────────────────────────────────┐
│                    ENTER CONTEXT MANAGER                    │
│  __enter__() / __aenter__()                                 │
├─────────────────────────────────────────────────────────────┤
│  • Load checkpoint from checkpointer                        │
│  • Restore channels from checkpoint                         │
│  • Initialize managed values                                │
│  • Apply input or resume from checkpoint                    │
│  • Create checkpoint (if input applied)                     │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                      TICK LOOP                              │
│  while loop.tick():                                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  loop.tick()                                                │
│    ┌─────────────────────────────────────────────────┐     │
│    │ • Check iteration limit                         │     │
│    │ • Prepare next tasks (prepare_next_tasks)       │     │
│    │ • Emit checkpoint event (if checkpointer)       │     │
│    │ • Return False if no tasks, True otherwise      │     │
│    │ • Check interrupt_before                        │     │
│    │ • Emit tasks event                              │     │
│    └─────────────────────────────────────────────────┘     │
│                           ↓                                 │
│  runner.tick(tasks)                                         │
│    ┌─────────────────────────────────────────────────┐     │
│    │ • Execute tasks concurrently                    │     │
│    │ • Yield control to emit stream output           │     │
│    │ • Commit writes via put_writes()                │     │
│    └─────────────────────────────────────────────────┘     │
│                           ↓                                 │
│  loop.after_tick()                                          │
│    ┌─────────────────────────────────────────────────┐     │
│    │ • Apply writes to channels                      │     │
│    │ • Emit values event (if channels updated)       │     │
│    │ • Save checkpoint                               │     │
│    │ • Check interrupt_after                         │     │
│    └─────────────────────────────────────────────────┘     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│                     EXIT CONTEXT MANAGER                    │
│  __exit__() / __aexit__()                                   │
├─────────────────────────────────────────────────────────────┤
│  • Save final checkpoint (if durability="exit")             │
│  • Suppress GraphInterrupt exception (if not nested)        │
│  • Set loop.output from final channel values               │
└─────────────────────────────────────────────────────────────┘
```

### invoke() Implementation

```python
def invoke(self, input, config):
    """Execute graph and return final output."""
    # Setup: merge config, initialize stream
    config = ensure_config(self.config, config)
    stream_modes, output_keys, ... = self._defaults(config, ...)

    # Create execution loop context
    with SyncPregelLoop(
        input=input,
        stream=StreamProtocol(...),
        config=config,
        checkpointer=checkpointer,
        nodes=self.nodes,
        specs=self.channels,
        ...
    ) as loop:
        # Create task runner
        runner = PregelRunner(
            submit=weakref.WeakMethod(loop.submit),
            put_writes=weakref.WeakMethod(loop.put_writes)
        )

        # Execute supersteps
        while loop.tick():
            # Execute tasks for this superstep
            for _ in runner.tick(loop.tasks.values(), ...):
                pass  # Could yield stream events here
            # Finalize superstep
            loop.after_tick()

        # Check termination status
        if loop.status == "out_of_steps":
            raise GraphRecursionError(...)

        # Return final output
        return loop.output
```

### stream() Implementation

The `stream()` method is similar but yields intermediate outputs:

```python
def stream(self, input, config, stream_mode="values"):
    """Stream graph execution, yielding outputs."""
    stream = SyncQueue()  # Thread-safe queue for outputs

    with SyncPregelLoop(..., stream=StreamProtocol(stream.put, stream_modes)) as loop:
        runner = PregelRunner(...)

        while loop.tick():
            for _ in runner.tick(loop.tasks.values(), ...):
                # Yield stream outputs as they become available
                yield from _output(stream_mode, stream.get, ...)
            loop.after_tick()

        # Yield final outputs
        yield from _output(stream_mode, stream.get, ...)
```

### Key Loop State

```python
class PregelLoop:
    # Configuration
    config: RunnableConfig
    step: int               # Current step number
    stop: int               # Maximum steps allowed
    is_nested: bool         # Is this a subgraph?

    # Execution state
    status: Literal["input", "pending", "done",
                    "interrupt_before", "interrupt_after", "out_of_steps"]
    tasks: dict[str, PregelExecutableTask]  # Tasks for current superstep
    output: None | dict[str, Any] | Any     # Final output

    # Checkpoint state
    checkpoint: Checkpoint
    checkpoint_id_saved: str
    checkpoint_config: RunnableConfig
    checkpoint_metadata: CheckpointMetadata
    checkpoint_pending_writes: list[PendingWrite]

    # Channel state
    channels: Mapping[str, BaseChannel]
    managed: ManagedValueMapping
    updated_channels: set[str] | None
```

---

## Supersteps

A **superstep** is one complete iteration of the execution loop. Understanding supersteps is key to understanding how LangGraph executes graphs.

### Superstep Anatomy

```
SUPERSTEP N (step=N)
├── Input: Checkpoint from step N-1
├── Process:
│   ├── 1. prepare_next_tasks()
│   │      └── Determine which nodes should run
│   ├── 2. Execute tasks in parallel
│   │      └── Each task produces writes
│   └── 3. apply_writes()
│          └── Update channels with all writes
└── Output: Checkpoint for step N
```

### Task Types

There are two types of tasks in a superstep:

1. **PULL Tasks** - Regular nodes triggered by channel updates
   ```
   Path: (PULL, node_name)
   Triggered when: Channel subscribed by node was updated
   Input: Values from subscribed channels
   ```

2. **PUSH Tasks** - Dynamic tasks sent via `Send()`
   ```
   Path: (PUSH, idx) or (PUSH, parent_path, write_idx, parent_id, Call)
   Triggered when: Parent task called Send() or call()
   Input: Argument from Send packet or Call
   ```

### Task Preparation (`prepare_next_tasks`)

Located in `_algo.py`, this function determines which tasks to execute:

```python
def prepare_next_tasks(
    checkpoint: Checkpoint,
    pending_writes: list[PendingWrite],
    processes: Mapping[str, PregelNode],
    channels: Mapping[str, BaseChannel],
    managed: ManagedValueMapping,
    config: RunnableConfig,
    step: int,
    stop: int,
    *,
    for_execution: bool,
    trigger_to_nodes: Mapping[str, Sequence[str]] | None = None,
    updated_channels: set[str] | None = None,
    ...
) -> dict[str, PregelTask | PregelExecutableTask]:
    """Prepare the set of tasks that will make up the next superstep."""

    tasks = []

    # 1. Consume PUSH tasks from TASKS channel
    tasks_channel = channels.get(TASKS)
    if tasks_channel and tasks_channel.is_available():
        for idx, packet in enumerate(tasks_channel.get()):
            if task := prepare_single_task((PUSH, idx), ...):
                tasks.append(task)

    # 2. Determine candidate PULL nodes
    if updated_channels and trigger_to_nodes:
        # Optimization: Only check nodes that subscribe to updated channels
        triggered_nodes = set()
        for channel in updated_channels:
            if node_ids := trigger_to_nodes.get(channel):
                triggered_nodes.update(node_ids)
        candidate_nodes = sorted(triggered_nodes)
    else:
        # No optimization info: check all nodes
        candidate_nodes = processes.keys()

    # 3. Check each candidate node
    for name in candidate_nodes:
        if task := prepare_single_task((PULL, name), ...):
            tasks.append(task)

    return {t.id: t for t in tasks}
```

### Task ID Generation

Task IDs are deterministically generated:

```python
# For v2+ checkpoints (default)
task_id = xxh3_128_hexdigest(
    checkpoint_id_bytes +
    checkpoint_ns.encode() +
    str(step).encode() +
    name.encode() +
    task_type.encode() +
    *trigger_channels.encode()
)

# Formatted as UUID: "12345678-1234-1234-1234-123456789abc"
```

This ensures:
- Same checkpoint + step + node → same task ID
- Enables task result caching
- Supports checkpoint replay

### Write Application (`apply_writes`)

After all tasks complete, their writes are applied atomically:

```python
def apply_writes(
    checkpoint: Checkpoint,
    channels: Mapping[str, BaseChannel],
    tasks: Iterable[WritesProtocol],
    get_next_version: GetNextVersion | None,
    trigger_to_nodes: Mapping[str, Sequence[str]],
) -> set[str]:
    """Apply writes from completed tasks to checkpoint and channels."""

    # 1. Sort tasks by path for determinism
    tasks = sorted(tasks, key=lambda t: task_path_str(t.path[:3]))

    # 2. Update versions_seen for each task
    for task in tasks:
        checkpoint["versions_seen"].setdefault(task.name, {}).update({
            chan: checkpoint["channel_versions"][chan]
            for chan in task.triggers
            if chan in checkpoint["channel_versions"]
        })

    # 3. Get next version number
    next_version = get_next_version(
        max(checkpoint["channel_versions"].values()),
        None
    ) if get_next_version else None

    # 4. Consume channels that were read
    for chan in {c for task in tasks for c in task.triggers}:
        if channels[chan].consume() and next_version:
            checkpoint["channel_versions"][chan] = next_version

    # 5. Group writes by channel
    pending_writes_by_channel = defaultdict(list)
    for task in tasks:
        for chan, val in task.writes:
            if chan in channels:
                pending_writes_by_channel[chan].append(val)

    # 6. Apply writes to channels
    updated_channels = set()
    for chan, vals in pending_writes_by_channel.items():
        if channels[chan].update(vals) and next_version:
            checkpoint["channel_versions"][chan] = next_version
            if channels[chan].is_available():
                updated_channels.add(chan)

    # 7. Notify unchanged channels of new step
    for chan in channels:
        if chan not in updated_channels and channels[chan].is_available():
            if channels[chan].update([]) and next_version:
                checkpoint["channel_versions"][chan] = next_version
                updated_channels.add(chan)

    return updated_channels
```

---

## Task Scheduling

Task scheduling is handled by the `PregelRunner` class in `_runner.py`. It executes tasks concurrently and manages their lifecycle.

### PregelRunner Architecture

```
PregelRunner
├── submit: Submit        # Function to submit tasks to executor
├── put_writes: Callable  # Function to save task writes
└── commit: Callable      # Called when task completes

FuturesDict (tracking structure)
├── event: Event          # Signaled when all tasks done
├── counter: int          # Number of pending tasks
├── done: set[Future]     # Completed task futures
└── callback: Callable    # Called when task completes (commit)
```

### Task Execution Flow

```python
def tick(self, tasks, *, reraise=True, timeout=None, get_waiter=None, schedule_task):
    """Execute tasks concurrently and yield control for streaming."""

    tasks = tuple(tasks)
    futures = FuturesDict(
        callback=weakref.WeakMethod(self.commit),
        event=threading.Event()
    )

    # Give control to caller for setup
    yield

    # Fast path: single task, no timeout, no waiter
    if len(tasks) == 1 and timeout is None and get_waiter is None:
        t = tasks[0]
        try:
            run_with_retry(t, retry_policy, ...)
            self.commit(t, None)
        except Exception as exc:
            self.commit(t, exc)
            if reraise:
                raise
        return

    # Schedule all tasks
    for t in tasks:
        fut = self.submit()(run_with_retry, t, retry_policy, ...)
        futures[fut] = t

    # Wait for tasks to complete
    while len(futures) > 0:
        done, inflight = concurrent.futures.wait(
            futures,
            return_when=FIRST_COMPLETED,
            timeout=timeout
        )

        # Process completed tasks
        for fut in done:
            task = futures.pop(fut)
            # commit() callback already called via FuturesDict

        # Check if we should stop (error or interrupt)
        if _should_stop_others(done):
            break

        # Yield control for streaming
        yield

    # Wait for commit callbacks to finish
    futures.event.wait(timeout=...)

    # Yield control once more
    yield

    # Raise any errors
    _panic_or_proceed(futures.done, panic=reraise)
```

### Commit Callback

When a task completes (success or failure), the commit callback is invoked:

```python
def commit(self, task: PregelExecutableTask, exception: BaseException | None):
    """Process task completion."""

    if isinstance(exception, asyncio.CancelledError):
        # Save cancellation error
        task.writes.append((ERROR, exception))
        self.put_writes()(task.id, task.writes)

    elif exception:
        if isinstance(exception, GraphInterrupt):
            # Save interrupt to checkpoint
            if exception.args[0]:
                writes = [(INTERRUPT, exception.args[0])]
                self.put_writes()(task.id, writes)
        elif isinstance(exception, GraphBubbleUp):
            # Will be re-raised in _panic_or_proceed
            pass
        else:
            # Save error to checkpoint
            task.writes.append((ERROR, exception))
            self.put_writes()(task.id, task.writes)

    else:
        # Success
        if not task.writes:
            # Add marker indicating no writes
            task.writes.append((NO_WRITES, None))
        # Save task writes to checkpoint
        self.put_writes()(task.id, task.writes)
```

### Retry Policy

Tasks can be configured with retry policies:

```python
def run_with_retry(task, retry_policy, ...):
    """Execute task with retry logic."""

    policies = task.retry_policy or retry_policy or []

    for attempt in range(max_attempts):
        try:
            # Execute the task
            result = task.proc.invoke(task.input, task.config)
            # Save result to writes
            task.writes.append((RETURN, result))
            return

        except Exception as exc:
            # Check if we should retry
            for policy in policies:
                if policy.should_retry(exc, attempt):
                    # Wait with backoff
                    time.sleep(policy.backoff(attempt))
                    # Clear previous writes
                    task.writes.clear()
                    # Try again
                    break
            else:
                # No retry policy matched, re-raise
                raise
```

### Cache Integration

The runner integrates with the cache system:

```python
# Before execution: check cache
for task in self.tasks.values():
    if task.cache_key and not task.writes:
        # Check if result is cached
        if cached_writes := cache.get(task.cache_key):
            task.writes.extend(cached_writes)
            # Skip execution

# After execution: save to cache
if task.cache_key and task.writes:
    cache.set(task.cache_key, task.writes, ttl=...)
```

---

## State Management

State in LangGraph is managed through **channels** and **checkpoints**.

### Channels

Channels are the primary state abstraction. Each channel has:
- **ValueType**: The type of value stored
- **UpdateType**: The type of updates accepted
- **update(values)**: Apply updates to the channel
- **get()**: Read current value
- **consume()**: Mark channel as consumed
- **is_available()**: Check if value is available

Common channel types:
```python
# LastValue: Stores most recent value
LastValue(int)  # Replaces value on each update

# Topic: Accumulates or deduplicates values
Topic(str, accumulate=True)   # Collects all values
Topic(str, accumulate=False)  # Only keeps unique values

# BinaryOperatorAggregate: Applies binary operator
BinaryOperatorAggregate(int, operator.add)  # Sums all updates

# EphemeralValue: Clears value after read
EphemeralValue(str)  # Value only available once

# Context: Manages lifecycle of external resources
Context(httpx.Client)  # Sets up and tears down client
```

### Channel Lifecycle

```
Channel Version Timeline:

Step 0 (Input):
  messages: v1 ─┐

Step 1:
  messages: v1 ─┤─── agent writes ───> v2 ─┐

Step 2:
  messages: v2 ─┤─── tool1 writes ───> v3 ─┐
                └─── tool2 writes ───> v3 ─┘

Step 3:
  messages: v3 ─┤─── agent writes ───> v4
```

Version tracking enables:
- Determining which nodes should run (compare version to last seen)
- Time-travel (restore to previous version)
- Optimistic locking (detect concurrent modifications)

### Checkpoints

A checkpoint is a complete snapshot of graph state:

```python
class Checkpoint(TypedDict):
    v: int                              # Checkpoint format version
    id: str                             # Unique checkpoint ID (UUID)
    ts: str                             # ISO timestamp
    channel_values: dict[str, Any]      # Current channel values
    channel_versions: dict[str, V]      # Current channel versions
    versions_seen: dict[str, dict[str, V]]  # Versions seen by each node
    pending_sends: list[Send]           # DEPRECATED (v4+: use TASKS channel)
```

**versions_seen** tracking:
```python
# Example:
checkpoint["versions_seen"] = {
    "agent": {
        "messages": v2,  # Last saw messages at v2
        "context": v1,   # Last saw context at v1
    },
    "tool": {
        "messages": v3,  # Last saw messages at v3
    }
}

# A node runs if any subscribed channel has:
#   channel_versions[chan] > versions_seen[node][chan]
```

### Checkpoint Lifecycle

```
┌───────────────────────────────────────────────────────────┐
│                    Input Checkpoint                       │
│  Step -1: Initial state (or loaded from checkpointer)     │
└───────────────────────────────────────────────────────────┘
                           ↓
              Apply input to channels
                           ↓
┌───────────────────────────────────────────────────────────┐
│                   Checkpoint 0 (Input)                    │
│  Metadata: {"source": "input", "step": 0}                 │
└───────────────────────────────────────────────────────────┘
                           ↓
              Execute superstep 1
                           ↓
┌───────────────────────────────────────────────────────────┐
│                   Checkpoint 1 (Loop)                     │
│  Metadata: {"source": "loop", "step": 1}                  │
└───────────────────────────────────────────────────────────┘
                           ↓
              Execute superstep 2
                           ↓
┌───────────────────────────────────────────────────────────┐
│                   Checkpoint 2 (Loop)                     │
│  Metadata: {"source": "loop", "step": 2}                  │
└───────────────────────────────────────────────────────────┘
                          ...
```

### Pending Writes

Writes are buffered as **pending writes** until applied:

```python
PendingWrite = tuple[str, str, Any]
# (task_id, channel, value)

# Example:
checkpoint_pending_writes = [
    ("task-123", "messages", AIMessage(...)),
    ("task-123", "context", {...}),
    ("task-456", "messages", HumanMessage(...)),
]
```

Pending writes are:
1. Collected during task execution
2. Saved incrementally to checkpointer (if durability != "exit")
3. Applied atomically at end of superstep
4. Cleared after application

### Durability Modes

LangGraph supports three durability modes:

```python
durability: Durability = "async"  # "sync" | "async" | "exit"
```

**"async" (default)**: Checkpoints saved asynchronously
```
Task executes ──> Writes collected ──┐
                                     ├──> checkpointer.put_writes() (background)
Next task starts ─────────────────────┘
```

**"sync"**: Checkpoints saved synchronously
```
Task executes ──> Writes collected ──┐
                                     ├──> checkpointer.put_writes() (blocking)
                                     └──> Wait for completion
Next task starts ─────────────────────┘
```

**"exit"**: Checkpoints saved only on exit
```
All tasks execute ──> All writes collected ──> On exit ──> checkpointer.put()
```

---

## Streaming

LangGraph supports multiple streaming modes to emit different types of events during execution.

### Stream Modes

```python
stream_mode: StreamMode | list[StreamMode]
# "values" | "updates" | "custom" | "messages" | "checkpoints" | "tasks" | "debug"
```

**"values"**: Emit complete state after each superstep
```python
for chunk in graph.stream(input, stream_mode="values"):
    print(chunk)
    # {"messages": [...], "context": {...}}
```

**"updates"**: Emit individual node outputs
```python
for chunk in graph.stream(input, stream_mode="updates"):
    print(chunk)
    # {"agent": AIMessage(...)}
    # {"tool": ToolMessage(...)}
```

**"custom"**: Emit custom data via StreamWriter
```python
def my_node(state):
    writer = get_config()["configurable"]["stream_writer"]
    writer({"custom_data": "..."})
    return state

for chunk in graph.stream(input, stream_mode="custom"):
    print(chunk)
    # {"custom_data": "..."}
```

**"messages"**: Stream LLM tokens in real-time
```python
for chunk in graph.stream(input, stream_mode="messages"):
    print(chunk)
    # (AIMessageChunk(content="Hello"), {"langgraph_node": "agent", ...})
    # (AIMessageChunk(content=" world"), {"langgraph_node": "agent", ...})
```

**"checkpoints"**: Emit checkpoint after each superstep
```python
for chunk in graph.stream(input, stream_mode="checkpoints"):
    print(chunk)
    # StateSnapshot(values={...}, next=("tool",), config={...})
```

**"tasks"**: Emit task start/completion events
```python
for chunk in graph.stream(input, stream_mode="tasks"):
    print(chunk)
    # PregelTask(id="...", name="agent", path=("PULL", "agent"))
```

**"debug"**: Emit all available debug information
```python
for chunk in graph.stream(input, stream_mode="debug"):
    print(chunk)
    # {"type": "checkpoint", "timestamp": "...", "payload": {...}}
    # {"type": "task", "timestamp": "...", "payload": {...}}
    # {"type": "task_result", "timestamp": "...", "payload": {...}}
```

### Multi-Mode Streaming

Multiple modes can be combined:

```python
for chunk in graph.stream(input, stream_mode=["values", "updates"]):
    mode, data = chunk
    if mode == "values":
        print(f"State: {data}")
    elif mode == "updates":
        print(f"Update: {data}")
```

### Subgraph Streaming

When `subgraphs=True`, events include namespace:

```python
for chunk in graph.stream(input, stream_mode="updates", subgraphs=True):
    namespace, data = chunk
    # namespace: tuple[str, ...] = ("parent_node:task-123", "child_node:task-456")
    # data: dict = {"grandchild_node": ...}
    print(f"{NS_SEP.join(namespace)}: {data}")
```

### Stream Implementation

The streaming infrastructure uses a thread-safe queue:

```python
# In stream() method:
stream = SyncQueue()  # or AsyncQueue for astream()

with SyncPregelLoop(..., stream=StreamProtocol(stream.put, stream_modes)) as loop:
    runner = PregelRunner(...)

    while loop.tick():
        # Tasks execute and emit to stream via stream.put()
        for _ in runner.tick(loop.tasks.values(), ...):
            # Yield stream outputs as they arrive
            yield from _output(stream_mode, stream.get, ...)
        loop.after_tick()

    # Yield any final outputs
    yield from _output(stream_mode, stream.get, ...)
```

Events are emitted through `_emit()` method:

```python
def _emit(self, mode: StreamMode, values: Callable[..., Iterator], *args, **kwargs):
    """Emit events to stream if mode is active."""
    if self.stream is None:
        return
    if mode not in self.stream.modes:
        return

    for v in values(*args, **kwargs):
        # Emit as (namespace, mode, value)
        self.stream((self.checkpoint_ns, mode, v))
```

### Stream Eager Mode

When `stream_eager=True` or using "messages"/"custom" modes, a waiter future is used:

```python
def get_waiter() -> Future[None]:
    """Return a future that resolves when stream has items."""
    nonlocal waiter
    if waiter is None or waiter.done():
        waiter = loop.submit(stream.wait)
    return waiter

# In runner.tick():
for _ in runner.tick(tasks, get_waiter=get_waiter, ...):
    # Emit stream outputs immediately as tasks produce them
    yield from _output(...)
```

This enables:
- Token-by-token streaming from LLMs
- Real-time custom events
- Subgraph event propagation

---

## Remote Execution

The `RemoteGraph` class enables executing graphs hosted on remote LangGraph servers (e.g., LangSmith deployments).

### RemoteGraph Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        Local Client                          │
│                                                              │
│  RemoteGraph(assistant_id="my-graph")                        │
│    ├── client: LangGraphClient (async)                       │
│    ├── sync_client: SyncLangGraphClient                      │
│    └── config: RunnableConfig                                │
│                                                              │
└───────────────────────┬──────────────────────────────────────┘
                        │
                        │ HTTP/HTTPS (REST API)
                        │
┌───────────────────────▼──────────────────────────────────────┐
│                     LangGraph Server                         │
│                                                              │
│  Graph Runtime                                               │
│    ├── Assistants (compiled graphs)                          │
│    ├── Threads (execution threads)                           │
│    └── Checkpointer (persistent state)                       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### RemoteGraph Usage

```python
from langgraph.pregel.remote import RemoteGraph

# Initialize remote graph
remote_graph = RemoteGraph(
    "my-assistant-id",
    url="https://my-deployment.langgraph.app",
    api_key="lsv2_...",
)

# Use like a local graph
result = remote_graph.invoke(
    {"messages": [HumanMessage(content="Hello")]},
    config={"configurable": {"thread_id": "thread-1"}}
)

# Stream from remote graph
for chunk in remote_graph.stream(input, stream_mode="values"):
    print(chunk)
```

### RemoteGraph as Subgraph

RemoteGraph implements `PregelProtocol`, so it can be used as a node:

```python
from langgraph.graph import StateGraph

# Local parent graph
parent_graph = StateGraph(...)

# Add remote graph as a node
parent_graph.add_node("remote_agent", remote_graph)

# Edges work normally
parent_graph.add_edge("other_node", "remote_agent")
```

### Stream Mode Mapping

RemoteGraph translates stream modes to the server API:

```python
def _get_stream_modes(self, stream_mode, config):
    """Map local stream modes to server stream modes."""

    # "messages" → "messages-tuple" (server format)
    if "messages" in stream_modes:
        stream_modes.remove("messages")
        stream_modes.append("messages-tuple")

    # Always add "updates" to detect interrupts
    if "updates" not in stream_modes:
        stream_modes.append("updates")

    return stream_modes
```

### Interrupt Handling

RemoteGraph handles interrupts from the remote server:

```python
for chunk in sync_client.runs.stream(...):
    # Parse namespace
    if NS_SEP in chunk.event:
        mode, ns_ = chunk.event.split(NS_SEP, 1)
        ns = tuple(ns_.split(NS_SEP))

    # Detect and raise interrupts
    if chunk.event.startswith("updates"):
        if isinstance(chunk.data, dict) and INTERRUPT in chunk.data:
            raise GraphInterrupt([
                Interrupt(**i) for i in chunk.data[INTERRUPT]
            ])

    # Detect and raise errors
    elif chunk.event.startswith("error"):
        raise RemoteException(chunk.data)
```

### Config Sanitization

Configs are sanitized before sending to server:

```python
def _sanitize_config(self, config: RunnableConfig) -> RunnableConfig:
    """Remove non-serializable fields."""
    sanitized = {}

    # Keep only serializable metadata
    if "metadata" in config:
        sanitized["metadata"] = {
            k: _sanitize_config_value(v)
            for k, v in config["metadata"].items()
            if isinstance(k, str)
        }

    # Remove internal configurable keys
    if "configurable" in config:
        sanitized["configurable"] = {
            k: v
            for k, v in config["configurable"].items()
            if k not in _CONF_DROPLIST  # Excludes internal keys
        }

    return sanitized
```

### State Snapshot Conversion

Remote state snapshots are converted to local format:

```python
def _create_state_snapshot(self, state: ThreadState) -> StateSnapshot:
    """Convert server state to local StateSnapshot."""

    tasks = []
    for task in state["tasks"]:
        tasks.append(PregelTask(
            id=task["id"],
            name=task["name"],
            path=tuple(),  # Path not available from server
            error=Exception(task["error"]) if task["error"] else None,
            interrupts=tuple(Interrupt(**i) for i in task["interrupts"]),
            state=...,  # Recursively convert nested states
        ))

    return StateSnapshot(
        values=state["values"],
        next=tuple(state["next"]),
        config={"configurable": state["checkpoint"]},
        metadata=CheckpointMetadata(**state["metadata"]),
        tasks=tuple(tasks),
        interrupts=tuple([i for task in tasks for i in task.interrupts]),
    )
```

### Distributed Tracing

RemoteGraph supports distributed tracing via LangSmith:

```python
remote_graph = RemoteGraph(
    "my-graph",
    url="...",
    distributed_tracing=True  # Enable tracing
)

# Tracing headers are automatically injected
def _merge_tracing_headers(headers):
    if rt := ls.get_current_run_tree():
        tracing_headers = rt.to_headers()
        if headers:
            headers.update(tracing_headers)
        else:
            headers = tracing_headers
    return headers
```

---

## Interrupts & Resumption

LangGraph supports interrupting execution and resuming from where it left off. This enables human-in-the-loop workflows.

### Interrupt Types

**1. Configured Interrupts** - Defined when creating the graph
```python
graph = StateGraph(...)
graph.compile(
    checkpointer=...,
    interrupt_before=["human_approval"],  # Interrupt before node
    interrupt_after=["agent"]              # Interrupt after node
)

# Or override at runtime
graph.stream(input, interrupt_before=["tool"])
```

**2. Programmatic Interrupts** - Raised within a node
```python
from langgraph.types import interrupt

def my_node(state):
    # Ask user for approval
    approval = interrupt({
        "question": "Approve this action?",
        "action": state["pending_action"]
    })

    if approval:
        return {"status": "approved"}
    else:
        return {"status": "rejected"}
```

### Interrupt Detection

Interrupts are detected in `_loop.py`:

```python
# Before execution
if self.interrupt_before and should_interrupt(
    self.checkpoint, self.interrupt_before, self.tasks.values()
):
    self.status = "interrupt_before"
    raise GraphInterrupt()

# After execution (in after_tick)
if self.interrupt_after and should_interrupt(
    self.checkpoint, self.interrupt_after, self.tasks.values()
):
    self.status = "interrupt_after"
    raise GraphInterrupt()
```

The `should_interrupt()` function checks:
```python
def should_interrupt(
    checkpoint: Checkpoint,
    interrupt_nodes: All | Sequence[str],
    tasks: Iterable[PregelExecutableTask],
) -> list[PregelExecutableTask]:
    """Check if the graph should be interrupted."""

    # Get version when last interrupted
    seen = checkpoint["versions_seen"].get(INTERRUPT, {})

    # Check if any channel was updated since last interrupt
    any_updates = any(
        version > seen.get(chan, null_version)
        for chan, version in checkpoint["channel_versions"].items()
    )

    # Return tasks that should trigger interrupt
    if any_updates:
        return [
            task for task in tasks
            if interrupt_nodes == "*" or task.name in interrupt_nodes
        ]
    else:
        return []
```

### Interrupt Storage

When an interrupt occurs:

```python
# In PregelRunner.commit():
if isinstance(exception, GraphInterrupt):
    # Save interrupt to checkpoint
    if exception.args[0]:
        writes = [(INTERRUPT, exception.args[0])]
        self.put_writes()(task.id, writes)

# This creates a pending write:
checkpoint_pending_writes.append((
    task_id,
    INTERRUPT,
    [Interrupt(value={"question": "..."}, resumable=True, ...)]
))
```

The interrupt is saved with the checkpoint, including:
- `value`: The interrupt payload
- `resumable`: Whether this interrupt can be resumed
- `ns`: The namespace where the interrupt occurred
- `id`: Unique interrupt ID

### Resumption Methods

**Method 1: Resume with `None` input**
```python
# First run - hits interrupt
for chunk in graph.stream({"input": "..."}):
    pass  # Interrupted before "human_approval" node

# Resume by calling again with None
for chunk in graph.stream(None):
    pass  # Continues from "human_approval" node
```

**Method 2: Resume with `Command`**
```python
from langgraph.types import Command

# Resume and provide value for interrupt
result = graph.invoke(Command(resume={"approval": True}))

# Resume specific interrupt by ID
result = graph.invoke(Command(resume={
    "interrupt-id-123": {"approval": True}
}))
```

**Method 3: Update state and resume**
```python
# Get current state
state = graph.get_state(config)

# Update state as if "human_approval" node ran
graph.update_state(
    config,
    {"approved": True},
    as_node="human_approval"
)

# Resume execution
result = graph.invoke(None, config)
```

### Resume Logic

The resume logic is in `_loop.py`:

```python
def _first(self, *, input_keys, updated_channels):
    """Handle first tick: apply input or resume."""

    # Check if resuming
    is_resuming = bool(self.checkpoint["channel_versions"]) and bool(
        configurable.get(CONFIG_KEY_RESUMING, self.input is None)
    )

    # Handle Command input with resume
    if isinstance(self.input, Command):
        if (resume := self.input.resume) is not None:
            if isinstance(resume, dict) and all(is_xxh3_128_hexdigest(k) for k in resume):
                # Resume by interrupt ID map
                self.config[CONF][CONFIG_KEY_RESUME_MAP] = resume
            else:
                # Resume single interrupt
                if len(self._pending_interrupts()) > 1:
                    raise RuntimeError("Must specify interrupt ID when multiple interrupts pending")

            # Group resume values by task ID
            writes = defaultdict(list)
            for tid, c, v in map_command(cmd=self.input):
                if not (c == RESUME and resume_is_map):
                    writes[tid].append((c, v))

            # Save resume writes
            for tid, ws in writes.items():
                self.put_writes(tid, ws)

    # Proceed past previous checkpoint
    if is_resuming:
        # Mark all channels as seen at current version
        self.checkpoint["versions_seen"].setdefault(INTERRUPT, {})
        for k in self.channels:
            if k in self.checkpoint["channel_versions"]:
                version = self.checkpoint["channel_versions"][k]
                self.checkpoint["versions_seen"][INTERRUPT][k] = version

        # Emit current values
        self._emit("values", map_output_values, self.output_keys, True, self.channels)

    # ... (apply input if not resuming)
```

### Multiple Interrupts

When multiple interrupts are pending, you must specify which to resume:

```python
# Get pending interrupts
state = graph.get_state(config)
interrupts = state.interrupts  # List of Interrupt objects

# Resume specific interrupt
graph.invoke(Command(resume={
    interrupts[0].id: {"approval": True},  # Resume first interrupt
    interrupts[1].id: {"approval": False}  # Resume second interrupt
}))
```

### Resumption with Scratchpad

The scratchpad manages resume values for tasks:

```python
def _scratchpad(
    parent_scratchpad,
    pending_writes,
    task_id,
    namespace_hash,
    resume_map,
    step,
    stop,
) -> PregelScratchpad:
    """Create scratchpad with resume values."""

    # Find global resume value
    null_resume_write = None
    for w in pending_writes:
        if w[0] == NULL_TASK_ID and w[1] == RESUME:
            null_resume_write = w
            break

    # Find task-specific resume values
    task_resume_write = []
    for w in pending_writes:
        if w[0] == task_id and w[1] == RESUME:
            task_resume_write.append(w[2])

    # Find namespace-specific resume value
    if resume_map and namespace_hash in resume_map:
        task_resume_write.append(resume_map[namespace_hash])

    return PregelScratchpad(
        step=step,
        stop=stop,
        resume=task_resume_write,
        get_null_resume=lambda: null_resume_write[2] if null_resume_write else None,
        ...
    )
```

### Interrupt Suppression

At the top level, `GraphInterrupt` is suppressed:

```python
def _suppress_interrupt(self, exc_type, exc_value, traceback):
    """Suppress interrupt at top level."""

    # Persist current checkpoint
    if self.durability == "exit":
        self._put_checkpoint(self.checkpoint_metadata)
        self._put_pending_writes()

    # Suppress only at top level (not nested graphs)
    suppress = isinstance(exc_value, GraphInterrupt) and not self.is_nested

    if suppress:
        # Save final output
        self.output = read_channels(self.channels, self.output_keys)
        return True  # Suppress exception
    elif exc_type is None:
        # Normal exit
        self.output = read_channels(self.channels, self.output_keys)
```

This ensures:
- Interrupts bubble up from nested graphs
- Interrupts are caught and saved at the top level
- Final state is available even when interrupted

---

## Summary

The LangGraph execution engine implements a sophisticated orchestration system based on the Pregel/BSP model:

1. **Pregel Model**: Execution proceeds in discrete supersteps with plan-execute-update phases
2. **Pregel Class**: Main graph orchestrator with invoke/stream/batch methods
3. **Execution Loop**: Managed by PregelLoop, coordinates supersteps and streaming
4. **Supersteps**: Atomic units of execution with parallel task execution
5. **Task Scheduling**: PregelRunner executes tasks concurrently with retry and caching
6. **State Management**: Channels and checkpoints provide durable, versioned state
7. **Streaming**: Multiple modes for different event types, with subgraph support
8. **Remote Execution**: RemoteGraph enables distributed graph execution
9. **Interrupts**: Flexible human-in-the-loop with multiple resumption strategies

The engine provides:
- **Determinism**: Same input and state produces same output
- **Parallelism**: Tasks execute concurrently within supersteps
- **Durability**: State persisted via checkpoints with configurable modes
- **Observability**: Rich streaming modes for monitoring execution
- **Resilience**: Retry policies and error handling
- **Flexibility**: Interrupt/resume for human-in-the-loop workflows

This architecture enables LangGraph to orchestrate complex, stateful workflows with strong guarantees around consistency, observability, and fault tolerance.
