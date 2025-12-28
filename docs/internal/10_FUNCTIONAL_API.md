# LangGraph Functional API

## Overview

### What is the Functional API?

The **Functional API** is a Python-native way to build LangGraph workflows using decorators (`@entrypoint` and `@task`) instead of the graph-building approach with `StateGraph`. It provides a more imperative, function-oriented programming model that feels natural for Python developers.

**Key Characteristics:**
- **Imperative Programming Model**: Write workflows as regular Python functions with normal control flow
- **Automatic Parallelization**: Tasks return futures that enable concurrent execution
- **Type-Safe**: Full type inference and IDE autocomplete support
- **Checkpointing Support**: Built-in state persistence for long-running workflows
- **Seamless Integration**: Can be combined with StateGraph for hybrid workflows

### Quick Example

```python
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver


@task
def process_item(item: str) -> str:
    return item.upper()


@entrypoint(checkpointer=InMemorySaver())
def workflow(items: list[str]) -> list[str]:
    # Tasks run in parallel automatically
    futures = [process_item(item) for item in items]
    # Collect results
    return [f.result() for f in futures]


# Execute the workflow
result = workflow.invoke(["hello", "world"])
# Returns: ["HELLO", "WORLD"]
```

---

## When to Use Functional API vs StateGraph

### Use the Functional API When:

1. **Linear or Simple Workflows**: Your workflow follows a straightforward execution path
2. **Familiar with Imperative Programming**: You prefer writing code with standard control flow (if/else, loops, etc.)
3. **Rapid Prototyping**: You want to quickly build a workflow without defining a state schema
4. **Map-Reduce Patterns**: You need to process items in parallel and aggregate results
5. **Type Safety is Critical**: You want full IDE support and type checking

### Use StateGraph When:

1. **Complex State Management**: You need fine-grained control over state updates and reduction logic
2. **Dynamic Routing**: Your workflow requires conditional edges and complex routing logic
3. **Visualization**: You want to visualize and debug the workflow graph structure
4. **Explicit State Schema**: You need to enforce a specific state structure across all nodes
5. **Multiple Entry Points**: Your workflow has multiple possible starting points

### Comparison Table

| Feature | Functional API | StateGraph |
|---------|---------------|------------|
| **Programming Model** | Imperative (functions) | Declarative (graph) |
| **State Management** | Implicit (return values) | Explicit (state dict) |
| **Parallelization** | Automatic (via futures) | Manual (via Send) |
| **Type Safety** | Full type inference | Schema-based |
| **Learning Curve** | Lower (familiar to Python devs) | Higher (graph concepts) |
| **Visualization** | Limited | Built-in graph visualization |
| **Complex Routing** | Manual (if/else) | Conditional edges |
| **IDE Support** | Excellent autocomplete | Good, with schema |

---

## @entrypoint Decorator

The `@entrypoint` decorator converts a Python function into a LangGraph workflow (Pregel graph). It serves as the main entry point for your workflow execution.

### Basic Syntax

```python
from langgraph.func import entrypoint

@entrypoint()
def my_workflow(input_data: str) -> str:
    return input_data.upper()
```

### Parameters

#### `checkpointer: BaseCheckpointSaver | None = None`

Specifies a checkpointer to persist workflow state across runs. When enabled, the workflow can:
- Resume from interrupts
- Access previous return values via the `previous` parameter
- Survive failures and resume from the last checkpoint

**Example:**
```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

@entrypoint(checkpointer=InMemorySaver())
def stateful_workflow(msg: str, *, previous: str | None = None) -> str:
    previous = previous or ""
    return previous + " " + msg

config = {"configurable": {"thread_id": "thread_1"}}
stateful_workflow.invoke("Hello", config)  # " Hello"
stateful_workflow.invoke("World", config)  # " Hello World"
```

#### `store: BaseStore | None = None`

A generalized key-value store for persisting data across workflow runs. Some implementations support semantic search through an optional `index` configuration.

**Example:**
```python
from langgraph.func import entrypoint
from langgraph.store.memory import InMemoryStore

@entrypoint(store=InMemoryStore())
def workflow_with_store(input_data: str, *, runtime) -> str:
    # Access store via runtime
    runtime.store.put(("namespace",), "key", {"data": input_data})
    return input_data
```

#### `cache: BaseCache | None = None`

A cache instance for caching task results. Must be used in conjunction with `cache_policy` on tasks.

**Example:**
```python
from langgraph.func import entrypoint, task
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

@task(cache_policy=CachePolicy())
def expensive_operation(x: int) -> int:
    return x * 2

@entrypoint(cache=InMemoryCache())
def cached_workflow(x: int) -> int:
    return expensive_operation(x).result()
```

#### `context_schema: type[ContextT] | None = None`

Specifies the schema for the context object passed to the workflow. Useful for type-safe access to run-scoped context.

**Example:**
```python
from typing_extensions import TypedDict
from langgraph.func import entrypoint

class Context(TypedDict):
    user_id: str
    api_key: str

@entrypoint(context_schema=Context)
def workflow(input_data: str, *, runtime) -> str:
    # Context is type-safe
    context = runtime.context
    return f"User: {context['user_id']}"
```

#### `cache_policy: CachePolicy | None = None`

A cache policy for caching the entire workflow's results.

#### `retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None`

Retry policy (or list of policies) to use for the workflow in case of failure.

**Example:**
```python
from langgraph.func import entrypoint
from langgraph.types import RetryPolicy

@entrypoint(retry_policy=RetryPolicy(max_attempts=3, initial_interval=1.0))
def resilient_workflow(input_data: str) -> str:
    # Workflow will retry up to 3 times on failure
    return input_data
```

### Function Signature Requirements

The decorated function must accept **a single positional parameter** as input. To pass multiple values, use a dictionary or tuple:

```python
# ❌ WRONG - multiple positional parameters
@entrypoint()
def workflow(a: int, b: int) -> int:
    return a + b

# ✅ CORRECT - single parameter (tuple)
@entrypoint()
def workflow(inputs: tuple[int, int]) -> int:
    a, b = inputs
    return a + b

# ✅ CORRECT - single parameter (dict)
@entrypoint()
def workflow(inputs: dict) -> int:
    return inputs["a"] + inputs["b"]
```

### Injectable Parameters

The entrypoint function can request additional parameters that are automatically injected at runtime:

#### `config: RunnableConfig`

The configuration object containing runtime settings, including thread_id, callbacks, etc.

```python
from langgraph.func import entrypoint
from langchain_core.runnables import RunnableConfig

@entrypoint()
def workflow(input_data: str, *, config: RunnableConfig) -> str:
    thread_id = config["configurable"]["thread_id"]
    return f"Thread: {thread_id}, Input: {input_data}"
```

#### `previous: Any`

The previous return value from the last invocation on the same thread (requires a checkpointer).

```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def counter(increment: int, *, previous: int | None = None) -> int:
    previous = previous or 0
    return previous + increment

config = {"configurable": {"thread_id": "1"}}
counter.invoke(5, config)   # 5
counter.invoke(3, config)   # 8
counter.invoke(2, config)   # 10
```

#### `runtime: Runtime`

A Runtime object containing information about the current run, including context, store, and writer.

```python
from langgraph.func import entrypoint

@entrypoint()
def workflow(input_data: str, *, runtime) -> str:
    # Access runtime components
    context = runtime.context
    store = runtime.store
    return input_data
```

### Return Value Handling

The entrypoint can return any value. When a checkpointer is enabled, this return value becomes available in the next invocation via the `previous` parameter.

#### Using `entrypoint.final` for Decoupled Return and Save Values

The `entrypoint.final` primitive allows you to return one value to the caller while saving a different value to the checkpoint:

```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from typing import Any

@entrypoint(checkpointer=InMemorySaver())
def workflow(
    number: int,
    *,
    previous: Any = None,
) -> entrypoint.final[int, int]:
    previous = previous or 0
    # Return the current previous value, but save 2 * number for next time
    return entrypoint.final(value=previous, save=2 * number)

config = {"configurable": {"thread_id": "1"}}
workflow.invoke(3, config)  # Returns: 0 (previous was None)
workflow.invoke(5, config)  # Returns: 6 (previous was 3 * 2)
workflow.invoke(7, config)  # Returns: 10 (previous was 5 * 2)
```

**Type Annotation:**
- `entrypoint.final[R, S]` where `R` is the return type and `S` is the save type
- Both types must be specified for proper type checking

### Invocation Methods

The decorated entrypoint becomes a Pregel graph with standard invocation methods:

#### `invoke(input, config=None)`

Synchronously execute the workflow and return the final result.

```python
result = workflow.invoke("input data", {"configurable": {"thread_id": "1"}})
```

#### `stream(input, config=None, stream_mode="updates")`

Stream intermediate updates during workflow execution.

```python
for update in workflow.stream("input data", config):
    print(update)
```

#### `ainvoke(input, config=None)` / `astream(input, config=None)`

Async variants for async workflows.

```python
result = await async_workflow.ainvoke("input data", config)

async for update in async_workflow.astream("input data", config):
    print(update)
```

### Error Handling

Errors in entrypoints propagate normally unless a `retry_policy` is specified:

```python
from langgraph.func import entrypoint
from langgraph.types import RetryPolicy

@entrypoint(retry_policy=RetryPolicy(max_attempts=3))
def may_fail(input_data: str) -> str:
    if some_condition():
        raise ValueError("Failed!")
    return input_data
```

### Complete Entrypoint Example

```python
from typing import Any
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, RetryPolicy

@task
def process_data(data: str) -> str:
    return data.upper()

@entrypoint(
    checkpointer=InMemorySaver(),
    retry_policy=RetryPolicy(max_attempts=2)
)
def review_workflow(
    data: str,
    *,
    previous: Any = None
) -> entrypoint.final[str, list[str]]:
    # Track history
    history = previous or []

    # Process data
    processed = process_data(data).result()

    # Request human review
    review = interrupt({"question": "Approve?", "data": processed})

    # Return result and update history
    result = f"{processed} (Reviewed: {review})"
    return entrypoint.final(value=result, save=history + [processed])
```

---

## @task Decorator

The `@task` decorator defines a unit of work that can be executed within an entrypoint or StateGraph. Tasks return futures, enabling automatic parallelization.

### Basic Syntax

```python
from langgraph.func import task

@task
def my_task(input_data: str) -> str:
    return input_data.upper()
```

### Parameters

#### `name: str | None = None`

Optional name for the task. If not provided, the function name is used. Useful for debugging and visualization.

**Example:**
```python
@task(name="data_processor")
def process(data: str) -> str:
    return data

# Function is still called 'process' but task name is 'data_processor'
```

#### `retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None`

Retry policy (or list of policies) for handling task failures.

**RetryPolicy Parameters:**
- `initial_interval: float = 0.5` - Time before first retry (seconds)
- `backoff_factor: float = 2.0` - Multiplier for retry interval after each attempt
- `max_interval: float = 128.0` - Maximum time between retries (seconds)
- `max_attempts: int = 3` - Maximum number of attempts (including first)
- `jitter: bool = True` - Add randomness to retry intervals
- `retry_on: Exception | Sequence[Exception] | Callable` - Which exceptions trigger retries

**Example:**
```python
from langgraph.func import task
from langgraph.types import RetryPolicy

@task(retry_policy=RetryPolicy(
    max_attempts=5,
    initial_interval=1.0,
    backoff_factor=2.0,
    jitter=True
))
def unreliable_task(data: str) -> str:
    # Will retry up to 5 times with exponential backoff
    result = make_api_call(data)  # May fail
    return result
```

**Multiple Retry Policies:**
```python
@task(retry_policy=[
    RetryPolicy(max_attempts=3, retry_on=ConnectionError),
    RetryPolicy(max_attempts=5, retry_on=TimeoutError, initial_interval=2.0)
])
def task_with_multiple_policies(data: str) -> str:
    return data
```

#### `cache_policy: CachePolicy[Callable[P, str | bytes]] | None = None`

Cache policy for caching task results. Requires a cache to be configured on the entrypoint.

**CachePolicy Parameters:**
- `key_func: Callable` - Function to generate cache key from inputs (defaults to hashing with pickle)
- `ttl: int | None = None` - Time to live in seconds (None = never expires)

**Example:**
```python
from langgraph.func import task, entrypoint
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

# Custom cache key function
def custom_key(x: int) -> str:
    return f"expensive_{x}"

@task(cache_policy=CachePolicy(key_func=custom_key, ttl=300))
def expensive_task(x: int) -> int:
    # This result is cached for 5 minutes
    return x * x

@entrypoint(cache=InMemoryCache())
def workflow(x: int) -> int:
    return expensive_task(x).result()

# First call executes the task
workflow.invoke(5)  # Computes 25

# Second call within 5 minutes uses cache
workflow.invoke(5)  # Returns cached 25 without computation
```

**Clearing Task Cache:**
```python
from langgraph.cache.memory import InMemoryCache

cache = InMemoryCache()

@task(cache_policy=CachePolicy())
def my_task(x: int) -> int:
    return x * 2

# Clear cache for this specific task
my_task.clear_cache(cache)

# Async version
await my_task.aclear_cache(cache)
```

### How Tasks are Scheduled

Tasks are **not executed immediately** when called. Instead, they return a future object that represents the eventual result:

1. **Task Call**: Returns a `SyncAsyncFuture` immediately
2. **Scheduling**: Task is scheduled for execution by the Pregel engine
3. **Execution**: Task runs when its dependencies are ready
4. **Result Available**: Future completes and result can be retrieved

**Execution Flow:**
```python
@task
def task_a(x: int) -> int:
    print("Executing task_a")
    return x + 1

@task
def task_b(x: int) -> int:
    print("Executing task_b")
    return x * 2

@entrypoint()
def workflow(x: int) -> int:
    # These calls don't execute the tasks yet
    future_a = task_a(x)
    future_b = task_b(x)

    # Tasks execute in parallel here
    # Both are scheduled and run concurrently
    result_a = future_a.result()
    result_b = future_b.result()

    return result_a + result_b

# Output shows parallel execution:
# Executing task_a
# Executing task_b
```

### Task Execution Context

Tasks execute within the same context as their parent entrypoint:
- They have access to the same checkpointer
- They can access the runtime via injected parameters
- They can use `interrupt()` for human-in-the-loop workflows

**Example:**
```python
from langgraph.func import task, entrypoint
from langgraph.types import interrupt

@task
def review_task(data: str) -> str:
    # interrupt() can be called within a task
    approval = interrupt({"question": "Approve this?", "data": data})
    return f"{data} (approved: {approval})"

@entrypoint(checkpointer=InMemorySaver())
def workflow(data: str) -> str:
    return review_task(data).result()
```

### Sync vs Async Tasks

Tasks can be either synchronous or asynchronous:

**Sync Task:**
```python
@task
def sync_task(x: int) -> int:
    return x * 2
```

**Async Task (requires Python 3.11+):**
```python
import asyncio

@task
async def async_task(x: int) -> int:
    await asyncio.sleep(1)
    return x * 2
```

**Mixing in Workflows:**
```python
import asyncio

@task
def sync_task(x: int) -> int:
    return x + 1

@task
async def async_task(x: int) -> int:
    await asyncio.sleep(0.1)
    return x * 2

@entrypoint()
async def async_workflow(x: int) -> int:
    # Sync task called from async workflow
    future_sync = sync_task(x)

    # Async task
    future_async = async_task(x)

    # Gather results
    results = await asyncio.gather(future_sync, future_async)
    return sum(results)
```

### Task Nesting

Tasks can call other tasks, creating a hierarchy:

```python
@task
def sub_task(x: int) -> int:
    return x * 2

@task
def parent_task(x: int) -> int:
    # Call another task from within a task
    result = sub_task(x).result()
    return result + 1

@entrypoint()
def workflow(x: int) -> int:
    return parent_task(x).result()

workflow.invoke(5)  # Returns: 11 (5 * 2 + 1)
```

### Calling Graphs from Tasks

Tasks can invoke StateGraph or other entrypoints:

```python
from langgraph.graph import StateGraph, START
from langgraph.func import task, entrypoint

# Define a StateGraph
class State(TypedDict):
    value: int

def increment(state: State) -> dict:
    return {"value": state["value"] + 1}

graph = StateGraph(State)
graph.add_node("increment", increment)
graph.add_edge(START, "increment")
compiled_graph = graph.compile()

# Call the graph from a task
@task
def call_graph(x: int) -> int:
    result = compiled_graph.invoke({"value": x})
    return result["value"]

@entrypoint()
def workflow(x: int) -> int:
    return call_graph(x).result()

workflow.invoke(5)  # Returns: 6
```

---

## Task Futures

When you call a task, it returns a **future** object (`SyncAsyncFuture[T]`) that represents the eventual result of the task execution.

### Understanding Futures

A future is a placeholder for a value that will be available later:

```python
@task
def add(a: int, b: int) -> int:
    return a + b

@entrypoint()
def workflow(x: int) -> int:
    # This doesn't execute add() yet - it returns a future
    future = add(x, 10)

    # Type of future: SyncAsyncFuture[int]
    print(type(future))  # <class 'SyncAsyncFuture'>

    # Calling result() executes the task and waits for completion
    result = future.result()
    return result
```

### The `.result()` Method

The `.result()` method blocks until the task completes and returns the task's return value:

**Sync Usage:**
```python
@task
def compute(x: int) -> int:
    return x * 2

@entrypoint()
def workflow(x: int) -> int:
    future = compute(x)
    # Blocks until compute() completes
    result = future.result()
    return result
```

**Async Usage:**
```python
import asyncio

@task
async def compute(x: int) -> int:
    await asyncio.sleep(1)
    return x * 2

@entrypoint()
async def workflow(x: int) -> int:
    future = compute(x)
    # Awaits until compute() completes
    result = await future
    return result
```

### Lazy Evaluation

Futures enable **lazy evaluation** - tasks only execute when `.result()` is called:

```python
@task
def expensive_task(x: int) -> int:
    print(f"Computing {x}")
    return x * 2

@entrypoint()
def workflow(condition: bool) -> int:
    # Task is not executed here
    future = expensive_task(10)

    if condition:
        # Only executed if condition is True
        return future.result()
    else:
        # Task never executes if condition is False
        return 0

workflow.invoke(False)  # Prints nothing - task never runs
workflow.invoke(True)   # Prints "Computing 10"
```

### Future Properties

**Type Safety:**
Futures preserve type information for IDE autocomplete:

```python
@task
def get_string() -> str:
    return "hello"

@entrypoint()
def workflow() -> str:
    future = get_string()
    # IDE knows future.result() returns str
    result: str = future.result()
    return result.upper()  # Autocomplete works!
```

**Awaitable:**
Futures can be awaited in async contexts:

```python
@task
async def async_task() -> str:
    return "result"

@entrypoint()
async def workflow() -> str:
    future = async_task()
    # These are equivalent:
    result1 = await future
    result2 = await future.result()
    return result1
```

### Error Handling with Futures

Errors are raised when `.result()` is called:

```python
@task
def failing_task() -> str:
    raise ValueError("Task failed!")

@entrypoint()
def workflow() -> str:
    future = failing_task()

    try:
        # Error is raised here, not at task call
        result = future.result()
    except ValueError as e:
        return f"Caught: {e}"

    return result

workflow.invoke()  # Returns: "Caught: Task failed!"
```

### Multiple Result Calls

Calling `.result()` multiple times on the same future returns the cached result:

```python
@task
def compute() -> int:
    print("Computing...")
    return 42

@entrypoint()
def workflow() -> int:
    future = compute()

    result1 = future.result()  # Prints "Computing..."
    result2 = future.result()  # No print - cached result
    result3 = future.result()  # No print - cached result

    return result1 + result2 + result3  # 126
```

---

## Parallel Execution

One of the key advantages of the Functional API is **automatic parallelization** through futures. Tasks that don't depend on each other execute concurrently.

### Basic Parallel Execution

```python
import time
from langgraph.func import task, entrypoint

@task
def slow_task(name: str, duration: float) -> str:
    time.sleep(duration)
    return f"{name} completed"

@entrypoint()
def parallel_workflow() -> list[str]:
    # All three tasks start execution in parallel
    future1 = slow_task("Task 1", 1.0)
    future2 = slow_task("Task 2", 1.0)
    future3 = slow_task("Task 3", 1.0)

    # Collect results (total time ~1 second, not 3)
    results = [
        future1.result(),
        future2.result(),
        future3.result(),
    ]
    return results

start = time.time()
parallel_workflow.invoke()
print(f"Took {time.time() - start:.1f}s")  # ~1.0s, not 3.0s
```

### Map-Reduce Pattern

Process multiple items in parallel and aggregate results:

```python
from langgraph.func import task, entrypoint

@task
def process_item(item: str) -> int:
    # Simulate processing
    return len(item)

@entrypoint()
def map_reduce_workflow(items: list[str]) -> int:
    # Map: Process all items in parallel
    futures = [process_item(item) for item in items]

    # Reduce: Aggregate results
    results = [f.result() for f in futures]
    return sum(results)

result = map_reduce_workflow.invoke(["hello", "world", "foo"])
# Returns: 12 (5 + 5 + 3)
```

### Async Parallel Execution

Use `asyncio.gather()` for async tasks:

```python
import asyncio
from langgraph.func import task, entrypoint

@task
async def async_task(x: int) -> int:
    await asyncio.sleep(0.1)
    return x * 2

@entrypoint()
async def async_parallel_workflow(numbers: list[int]) -> list[int]:
    # Start all tasks
    futures = [async_task(n) for n in numbers]

    # Wait for all to complete in parallel
    results = await asyncio.gather(*futures)
    return results

result = await async_parallel_workflow.ainvoke([1, 2, 3, 4, 5])
# Returns: [2, 4, 6, 8, 10]
```

### Conditional Parallelism

Control which tasks run based on conditions:

```python
@task
def task_a(x: int) -> int:
    return x + 1

@task
def task_b(x: int) -> int:
    return x * 2

@task
def task_c(x: int) -> int:
    return x ** 2

@entrypoint()
def conditional_parallel(x: int, run_all: bool) -> int:
    # Always run task_a
    future_a = task_a(x)
    result_a = future_a.result()

    if run_all:
        # Run b and c in parallel only if run_all is True
        future_b = task_b(result_a)
        future_c = task_c(result_a)
        return future_b.result() + future_c.result()
    else:
        return result_a

conditional_parallel.invoke(5, run_all=False)  # Returns: 6
conditional_parallel.invoke(5, run_all=True)   # Returns: 48 (12 + 36)
```

### Dependent Tasks

Tasks that depend on previous results execute sequentially:

```python
@task
def step1(x: int) -> int:
    return x + 10

@task
def step2(x: int) -> int:
    return x * 2

@task
def step3(x: int) -> int:
    return x - 5

@entrypoint()
def sequential_workflow(x: int) -> int:
    # These must execute in order due to dependencies
    result1 = step1(x).result()      # x + 10
    result2 = step2(result1).result() # (x + 10) * 2
    result3 = step3(result2).result() # ((x + 10) * 2) - 5
    return result3

sequential_workflow.invoke(5)  # Returns: 25
```

### Mixed Parallel and Sequential

Combine parallel and sequential execution:

```python
@task
def fetch_user(user_id: int) -> dict:
    return {"id": user_id, "name": f"User {user_id}"}

@task
def fetch_posts(user_id: int) -> list[dict]:
    return [{"id": 1, "title": "Post 1"}]

@task
def fetch_comments(user_id: int) -> list[dict]:
    return [{"id": 1, "text": "Comment 1"}]

@task
def aggregate(user: dict, posts: list, comments: list) -> dict:
    return {
        "user": user,
        "posts": posts,
        "comments": comments,
        "total_content": len(posts) + len(comments)
    }

@entrypoint()
def user_dashboard(user_id: int) -> dict:
    # Step 1: Fetch user info (sequential - needed first)
    user = fetch_user(user_id).result()

    # Step 2: Fetch posts and comments in parallel
    posts_future = fetch_posts(user_id)
    comments_future = fetch_comments(user_id)

    posts = posts_future.result()
    comments = comments_future.result()

    # Step 3: Aggregate (sequential - needs all data)
    return aggregate(user, posts, comments).result()
```

### Parallel Execution with Limits

Control concurrency by batching:

```python
from langgraph.func import task, entrypoint

@task
def process_batch(items: list[str]) -> list[str]:
    return [item.upper() for item in items]

@entrypoint()
def limited_parallel_workflow(items: list[str], batch_size: int = 10) -> list[str]:
    # Process in batches to limit parallelism
    results = []

    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        # Process this batch
        result = process_batch(batch).result()
        results.extend(result)

    return results

# Process 100 items in batches of 10
items = [f"item_{i}" for i in range(100)]
result = limited_parallel_workflow.invoke(items, batch_size=10)
```

### Race Condition Handling

First completed task wins:

```python
import asyncio
from langgraph.func import task, entrypoint

@task
async def source_a() -> str:
    await asyncio.sleep(1.0)
    return "Result from A"

@task
async def source_b() -> str:
    await asyncio.sleep(0.5)
    return "Result from B"

@task
async def source_c() -> str:
    await asyncio.sleep(2.0)
    return "Result from C"

@entrypoint()
async def race_workflow() -> str:
    # Start all sources
    futures = [source_a(), source_b(), source_c()]

    # Return first completed result
    done, pending = await asyncio.wait(
        futures,
        return_when=asyncio.FIRST_COMPLETED
    )

    # Cancel pending tasks
    for task_future in pending:
        task_future.cancel()

    # Get result from first completed
    result = await done.pop()
    return result

# Returns: "Result from B" (fastest)
```

---

## State Management

The Functional API provides implicit state management through return values and the `previous` parameter when checkpointing is enabled.

### Return Values as State

In the Functional API, the return value of an entrypoint becomes the state:

```python
from langgraph.func import entrypoint

@entrypoint()
def simple_workflow(input_data: str) -> str:
    return input_data.upper()

result = simple_workflow.invoke("hello")
# result = "HELLO"
```

### The `previous` Parameter

When a checkpointer is configured, you can access the previous return value using the `previous` injectable parameter:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

@entrypoint(checkpointer=InMemorySaver())
def stateful_workflow(
    message: str,
    *,
    previous: list[str] | None = None
) -> list[str]:
    # Initialize history if first run
    history = previous or []
    # Append new message
    history.append(message)
    return history

config = {"configurable": {"thread_id": "conversation_1"}}

stateful_workflow.invoke("Hello", config)
# Returns: ["Hello"]

stateful_workflow.invoke("How are you?", config)
# Returns: ["Hello", "How are you?"]

stateful_workflow.invoke("Goodbye", config)
# Returns: ["Hello", "How are you?", "Goodbye"]
```

**Key Points:**
- `previous` is only available when a checkpointer is configured
- `previous` is scoped to the thread_id in the config
- `previous` should have a default value (usually `None`)
- The type of `previous` matches the return type of the entrypoint

### Thread Isolation

Different threads maintain separate state:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

@entrypoint(checkpointer=InMemorySaver())
def counter(increment: int, *, previous: int | None = None) -> int:
    return (previous or 0) + increment

thread1 = {"configurable": {"thread_id": "thread_1"}}
thread2 = {"configurable": {"thread_id": "thread_2"}}

counter.invoke(5, thread1)   # Returns: 5
counter.invoke(3, thread1)   # Returns: 8
counter.invoke(10, thread2)  # Returns: 10 (different thread)
counter.invoke(2, thread2)   # Returns: 12
counter.invoke(1, thread1)   # Returns: 9 (back to thread 1)
```

### Decoupling Return and Save with `entrypoint.final`

Use `entrypoint.final` to return one value while saving a different value for the next invocation:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint
from typing import Any

@entrypoint(checkpointer=InMemorySaver())
def workflow(
    number: int,
    *,
    previous: Any = None,
) -> entrypoint.final[int, int]:
    previous_value = previous or 0
    new_value = number * 2

    # Return the OLD value to the caller
    # Save the NEW value for next invocation
    return entrypoint.final(value=previous_value, save=new_value)

config = {"configurable": {"thread_id": "1"}}

result1 = workflow.invoke(3, config)  # Returns: 0,  Saves: 6
result2 = workflow.invoke(5, config)  # Returns: 6,  Saves: 10
result3 = workflow.invoke(7, config)  # Returns: 10, Saves: 14
```

**Use Cases for `entrypoint.final`:**
- Accumulating state internally while returning processed results
- Implementing state machines where internal state differs from output
- Caching computed values between runs

### Complex State Types

The state can be any serializable Python type:

**Dictionary State:**
```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint
from typing import TypedDict

class ConversationState(TypedDict):
    messages: list[str]
    user_name: str
    turn_count: int

@entrypoint(checkpointer=InMemorySaver())
def chat_workflow(
    user_message: str,
    *,
    previous: ConversationState | None = None
) -> ConversationState:
    if previous is None:
        # Initialize state on first message
        state = {
            "messages": [],
            "user_name": "Unknown",
            "turn_count": 0
        }
    else:
        state = previous

    # Update state
    state["messages"].append(user_message)
    state["turn_count"] += 1

    return state
```

**Custom Class State:**
```python
from dataclasses import dataclass
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

@dataclass
class GameState:
    score: int
    level: int
    inventory: list[str]

@entrypoint(checkpointer=InMemorySaver())
def game_workflow(
    action: str,
    *,
    previous: GameState | None = None
) -> GameState:
    state = previous or GameState(score=0, level=1, inventory=[])

    if action == "score":
        state.score += 10
    elif action == "level_up":
        state.level += 1
    elif action.startswith("add_"):
        item = action[4:]
        state.inventory.append(item)

    return state
```

### State Updates in Tasks

Tasks don't directly modify state - they return values that the entrypoint uses:

```python
from langgraph.func import task, entrypoint
from langgraph.checkpoint.memory import InMemorySaver

@task
def process_message(message: str) -> dict:
    return {
        "processed": message.upper(),
        "length": len(message),
        "timestamp": "2024-01-01"
    }

@entrypoint(checkpointer=InMemorySaver())
def workflow(
    message: str,
    *,
    previous: list[dict] | None = None
) -> list[dict]:
    history = previous or []

    # Task processes the message
    processed = process_message(message).result()

    # Entrypoint updates state
    history.append(processed)
    return history
```

### Resetting State

To reset state, use a different thread_id or clear the checkpoint:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def workflow(x: int, *, previous: int | None = None) -> int:
    return (previous or 0) + x

config = {"configurable": {"thread_id": "1"}}

workflow.invoke(5, config)  # 5
workflow.invoke(3, config)  # 8

# Reset by deleting the thread
checkpointer.delete_thread("1")

workflow.invoke(2, config)  # 2 (reset)
```

---

## Checkpointing

Checkpointing enables the Functional API to persist state, survive failures, and support human-in-the-loop workflows.

### Enabling Checkpointing

Pass a checkpointer instance to the `@entrypoint` decorator:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

@entrypoint(checkpointer=InMemorySaver())
def workflow(data: str, *, previous: str | None = None) -> str:
    previous = previous or ""
    return previous + data

# Thread ID is required when using a checkpointer
config = {"configurable": {"thread_id": "thread_1"}}
workflow.invoke("Hello ", config)
workflow.invoke("World", config)  # Returns: "Hello World"
```

### Checkpointer Types

**InMemorySaver:**
Stores checkpoints in memory (lost on restart):

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()
```

**SqliteSaver:**
Persists checkpoints to a SQLite database:

```python
from langgraph.checkpoint.sqlite import SqliteSaver

checkpointer = SqliteSaver.from_conn_string("checkpoints.db")
```

**PostgresSaver:**
Persists checkpoints to PostgreSQL:

```python
from langgraph.checkpoint.postgres import PostgresSaver

checkpointer = PostgresSaver.from_conn_string(
    "postgresql://user:pass@localhost/db"
)
```

### How Checkpointing Works

1. **Before Execution**: LangGraph loads the checkpoint for the given thread_id
2. **During Execution**: Task results are written to the checkpoint incrementally
3. **After Execution**: Final state is saved to the checkpoint
4. **On Resume**: Cached task results are loaded, avoiding re-execution

**Example:**
```python
from langgraph.func import task, entrypoint
from langgraph.checkpoint.memory import InMemorySaver

@task
def expensive_computation(x: int) -> int:
    print(f"Computing {x}...")
    return x * 2

@entrypoint(checkpointer=InMemorySaver())
def workflow(x: int, *, previous: int | None = None) -> int:
    result = expensive_computation(x).result()
    return (previous or 0) + result

config = {"configurable": {"thread_id": "1"}}

# First run - executes the task
workflow.invoke(5, config)
# Prints: "Computing 5..."
# Returns: 10

# Second run - task result is cached
workflow.invoke(3, config)
# Prints: "Computing 3..."
# Returns: 16 (10 + 6)
```

### Checkpoint State History

Access previous checkpoints using `get_state_history()`:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def workflow(msg: str, *, previous: list | None = None) -> list:
    history = previous or []
    history.append(msg)
    return history

config = {"configurable": {"thread_id": "1"}}

workflow.invoke("First", config)
workflow.invoke("Second", config)
workflow.invoke("Third", config)

# Get state history
history = list(workflow.get_state_history(config))

for state in history:
    print(f"Values: {state.values}")
    print(f"Created: {state.created_at}")
    print("---")
```

### Updating State Manually

Manually update checkpoint state without executing the workflow:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

@entrypoint(checkpointer=InMemorySaver())
def workflow(x: int, *, previous: int | None = None) -> int:
    return (previous or 0) + x

config = {"configurable": {"thread_id": "1"}}

# Normal execution
workflow.invoke(5, config)  # 5

# Manually update the state
workflow.update_state(config, 100)

# Next execution uses the updated state
workflow.invoke(3, config)  # 103
```

### Failure Recovery

Checkpointing enables workflows to resume after failures:

```python
from langgraph.func import task, entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import RetryPolicy

call_count = 0

@task(retry_policy=RetryPolicy(max_attempts=3))
def flaky_task(x: int) -> int:
    global call_count
    call_count += 1

    if call_count < 2:
        raise ConnectionError("Temporary failure")

    return x * 2

@entrypoint(checkpointer=InMemorySaver())
def workflow(x: int) -> int:
    return flaky_task(x).result()

config = {"configurable": {"thread_id": "1"}}

# First attempt fails, second succeeds (thanks to retry policy)
result = workflow.invoke(5, config)  # Returns: 10
```

### Time-Travel Debugging

Use checkpoints to replay workflow execution:

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def workflow(x: int, *, previous: int | None = None) -> int:
    return (previous or 0) + x

config = {"configurable": {"thread_id": "1"}}

workflow.invoke(5, config)   # State: 5
workflow.invoke(10, config)  # State: 15
workflow.invoke(3, config)   # State: 18

# Get all states
history = list(workflow.get_state_history(config))

# Go back to second state
second_state = history[1]
print(f"Value at step 2: {second_state.values}")

# Continue from that point (using its config)
result = workflow.invoke(7, second_state.config)
# Starts from state 15, adds 7 → returns 22
```

### Checkpoint Configuration

Thread IDs are required for checkpointing:

```python
# ❌ WRONG - Missing thread_id
config = {}
workflow.invoke(data, config)  # Error!

# ✅ CORRECT - Thread ID provided
config = {"configurable": {"thread_id": "user_123"}}
workflow.invoke(data, config)
```

**Multiple Threads:**
```python
config_user1 = {"configurable": {"thread_id": "user_1"}}
config_user2 = {"configurable": {"thread_id": "user_2"}}

workflow.invoke(data1, config_user1)  # Separate state
workflow.invoke(data2, config_user2)  # Separate state
```

---

## Interrupts

The `interrupt()` function enables human-in-the-loop workflows by pausing execution and requesting input.

### Basic Interrupt

```python
from langgraph.func import entrypoint
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def workflow(data: str) -> str:
    # Process data
    processed = data.upper()

    # Request human input
    approval = interrupt({"question": "Approve?", "data": processed})

    # Continue with human input
    if approval == "yes":
        return f"Approved: {processed}"
    else:
        return f"Rejected: {processed}"

config = {"configurable": {"thread_id": "1"}}

# First invocation - stops at interrupt
result = workflow.invoke("hello", config)
# Returns: {"__interrupt__": [Interrupt(value={"question": "Approve?", ...})]}

# Resume with input
result = workflow.invoke(Command(resume="yes"), config)
# Returns: "Approved: HELLO"
```

### How Interrupts Work

1. **First Call**: `interrupt()` raises a `GraphInterrupt` exception, halting execution
2. **State Saved**: Current state and interrupt value are saved to checkpoint
3. **Client Notified**: Interrupt value is returned to the caller
4. **Resume**: Client calls workflow with `Command(resume=value)`
5. **Replay**: Workflow re-executes from the start of the node
6. **Value Returned**: `interrupt()` returns the resume value instead of raising

### Interrupts in Tasks

Tasks can use interrupts for approval workflows:

```python
from langgraph.func import task, entrypoint
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@task
def process_and_review(data: str) -> str:
    # Process the data
    processed = data.upper()

    # Request approval within the task
    approved = interrupt({"question": "Approve this?", "data": processed})

    if approved:
        return f"✓ {processed}"
    else:
        return f"✗ {processed}"

@entrypoint(checkpointer=InMemorySaver())
def workflow(data: str) -> str:
    return process_and_review(data).result()

config = {"configurable": {"thread_id": "1"}}

# Stops at interrupt
workflow.invoke("hello", config)

# Resume with approval
result = workflow.invoke(Command(resume=True), config)
# Returns: "✓ HELLO"
```

### Multiple Interrupts

A workflow can have multiple sequential interrupts:

```python
from langgraph.func import entrypoint
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def multi_interrupt_workflow(data: str) -> dict:
    # First interrupt
    name = interrupt({"question": "What's your name?"})

    # Second interrupt
    age = interrupt({"question": "What's your age?"})

    # Third interrupt
    city = interrupt({"question": "What's your city?"})

    return {
        "name": name,
        "age": age,
        "city": city,
        "original": data
    }

config = {"configurable": {"thread_id": "1"}}

# First run
workflow.invoke("start", config)
# Returns interrupt with "What's your name?"

# Resume with name
workflow.invoke(Command(resume="Alice"), config)
# Returns interrupt with "What's your age?"

# Resume with age
workflow.invoke(Command(resume="30"), config)
# Returns interrupt with "What's your city?"

# Resume with city
result = workflow.invoke(Command(resume="NYC"), config)
# Returns: {"name": "Alice", "age": "30", "city": "NYC", "original": "start"}
```

### Multiple Interrupts with IDs

When there are multiple pending interrupts, use interrupt IDs to resume specific ones:

```python
from langgraph.func import task, entrypoint
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@task
def review_text(text: str, label: str) -> str:
    approval = interrupt({"question": f"Approve {label}?", "text": text})
    return f"{text} ({approval})"

@entrypoint(checkpointer=InMemorySaver())
def parallel_interrupts(text1: str, text2: str) -> dict:
    # Two tasks that both interrupt
    future1 = review_text(text1, "text1")
    future2 = review_text(text2, "text2")

    return {
        "result1": future1.result(),
        "result2": future2.result()
    }

config = {"configurable": {"thread_id": "1"}}

# First run - both tasks interrupt
result = parallel_interrupts.invoke("Hello", "World", config)

# Get interrupt IDs
state = parallel_interrupts.get_state(config)
interrupts = state.interrupts

# Resume with a mapping of interrupt ID to value
resume_map = {
    interrupts[0].id: "approved",
    interrupts[1].id: "rejected"
}

result = parallel_interrupts.invoke(Command(resume=resume_map), config)
# Returns: {"result1": "Hello (approved)", "result2": "World (rejected)"}
```

### Interrupt Context

The interrupt value can be any serializable Python object:

**Simple Value:**
```python
answer = interrupt("What's your name?")
```

**Dictionary:**
```python
response = interrupt({
    "question": "Review this essay",
    "essay": essay_text,
    "word_count": len(essay_text.split())
})
```

**List:**
```python
choice = interrupt({
    "question": "Choose an option",
    "options": ["Option A", "Option B", "Option C"]
})
```

### Conditional Interrupts

Interrupt only when certain conditions are met:

```python
from langgraph.func import entrypoint
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def conditional_interrupt_workflow(amount: float, auto_approve_limit: float) -> str:
    if amount > auto_approve_limit:
        # Only interrupt for amounts above the limit
        approved = interrupt({
            "question": f"Approve transaction of ${amount}?",
            "amount": amount
        })

        if not approved:
            return f"Transaction of ${amount} rejected"

    # Auto-approve or approved by human
    return f"Transaction of ${amount} approved"

config = {"configurable": {"thread_id": "1"}}

# Small amount - auto-approved
result = conditional_interrupt_workflow.invoke(50.0, 100.0, config)
# Returns: "Transaction of $50.0 approved"

# Large amount - requires approval
conditional_interrupt_workflow.invoke(150.0, 100.0, config)
# Returns interrupt

result = conditional_interrupt_workflow.invoke(Command(resume=True), config)
# Returns: "Transaction of $150.0 approved"
```

### Error Handling with Interrupts

Handle cases where resume value is invalid:

```python
from langgraph.func import entrypoint
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def validated_interrupt_workflow() -> str:
    while True:
        age = interrupt({"question": "What's your age?"})

        # Validate the input
        try:
            age_int = int(age)
            if 0 < age_int < 150:
                return f"Your age is {age_int}"
            else:
                # Invalid - interrupt again with error message
                continue
        except (ValueError, TypeError):
            # Invalid format - interrupt again
            continue

config = {"configurable": {"thread_id": "1"}}

workflow.invoke("start", config)
workflow.invoke(Command(resume="invalid"), config)  # Interrupts again
workflow.invoke(Command(resume="-5"), config)        # Interrupts again
result = workflow.invoke(Command(resume="30"), config)  # Success
# Returns: "Your age is 30"
```

---

## Combining with StateGraph

The Functional API can be seamlessly combined with StateGraph for hybrid workflows.

### Calling StateGraph from Functional API

Use tasks to invoke StateGraph workflows:

```python
from langgraph.graph import StateGraph, START
from langgraph.func import task, entrypoint
from typing_extensions import TypedDict

# Define a StateGraph
class GraphState(TypedDict):
    count: int

def increment(state: GraphState) -> dict:
    return {"count": state["count"] + 1}

def double(state: GraphState) -> dict:
    return {"count": state["count"] * 2}

builder = StateGraph(GraphState)
builder.add_node("increment", increment)
builder.add_node("double", double)
builder.add_edge(START, "increment")
builder.add_edge("increment", "double")
graph = builder.compile()

# Call from Functional API
@task
def run_state_graph(x: int) -> int:
    result = graph.invoke({"count": x})
    return result["count"]

@entrypoint()
def workflow(x: int) -> int:
    # Run the StateGraph through a task
    return run_state_graph(x).result()

result = workflow.invoke(5)
# Returns: 12 ((5 + 1) * 2)
```

### Calling Functional API from StateGraph

Use functional entrypoints as nodes in a StateGraph:

```python
from langgraph.graph import StateGraph, START
from langgraph.func import entrypoint, task
from typing_extensions import TypedDict

# Define functional workflow
@task
def process_text(text: str) -> str:
    return text.upper()

@entrypoint()
def text_processor(text: str) -> str:
    return process_text(text).result()

# Define StateGraph that uses it
class State(TypedDict):
    text: str
    processed: str

def call_functional_api(state: State) -> dict:
    result = text_processor.invoke(state["text"])
    return {"processed": result}

builder = StateGraph(State)
builder.add_node("process", call_functional_api)
builder.add_edge(START, "process")
graph = builder.compile()

result = graph.invoke({"text": "hello world"})
# Returns: {"text": "hello world", "processed": "HELLO WORLD"}
```

### Hybrid Architecture Pattern

Build complex workflows with both paradigms:

```python
from langgraph.graph import StateGraph, START, END
from langgraph.func import task, entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from typing_extensions import TypedDict
import operator
from typing import Annotated

# Functional API for parallel data processing
@task
def analyze_chunk(chunk: str) -> dict:
    return {
        "length": len(chunk),
        "words": len(chunk.split()),
        "has_numbers": any(c.isdigit() for c in chunk)
    }

@entrypoint()
def analyze_chunks(chunks: list[str]) -> list[dict]:
    futures = [analyze_chunk(chunk) for chunk in chunks]
    return [f.result() for f in futures]

# StateGraph for orchestration
class WorkflowState(TypedDict):
    input_text: str
    chunks: list[str]
    analysis: list[dict]
    summary: dict

def split_text(state: WorkflowState) -> dict:
    # Split into chunks
    text = state["input_text"]
    chunk_size = 100
    chunks = [text[i:i+chunk_size] for i in range(0, len(text), chunk_size)]
    return {"chunks": chunks}

def run_analysis(state: WorkflowState) -> dict:
    # Use functional API for parallel analysis
    analysis = analyze_chunks.invoke(state["chunks"])
    return {"analysis": analysis}

def create_summary(state: WorkflowState) -> dict:
    # Aggregate results
    total_length = sum(a["length"] for a in state["analysis"])
    total_words = sum(a["words"] for a in state["analysis"])
    has_numbers = any(a["has_numbers"] for a in state["analysis"])

    return {
        "summary": {
            "total_length": total_length,
            "total_words": total_words,
            "has_numbers": has_numbers,
            "chunk_count": len(state["chunks"])
        }
    }

# Build the graph
builder = StateGraph(WorkflowState)
builder.add_node("split", split_text)
builder.add_node("analyze", run_analysis)
builder.add_node("summarize", create_summary)
builder.add_edge(START, "split")
builder.add_edge("split", "analyze")
builder.add_edge("analyze", "summarize")
builder.add_edge("summarize", END)

hybrid_workflow = builder.compile()

# Execute
result = hybrid_workflow.invoke({
    "input_text": "Your long text here..." * 100
})
print(result["summary"])
```

### Nested Workflows

Create modular, reusable components:

```python
from langgraph.func import entrypoint, task
from langgraph.graph import StateGraph, START
from typing_extensions import TypedDict

# Low-level functional workflow
@task
def validate_email(email: str) -> bool:
    return "@" in email and "." in email

@entrypoint()
def email_validator(email: str) -> dict:
    is_valid = validate_email(email).result()
    return {"email": email, "valid": is_valid}

# Mid-level functional workflow
@task
def check_user(user_data: dict) -> dict:
    email_result = email_validator.invoke(user_data["email"])
    return {
        **user_data,
        "email_valid": email_result["valid"]
    }

@entrypoint()
def user_validator(users: list[dict]) -> list[dict]:
    futures = [check_user(user) for user in users]
    return [f.result() for f in futures]

# Top-level StateGraph
class AppState(TypedDict):
    users: list[dict]
    valid_users: list[dict]
    invalid_users: list[dict]

def validate_users(state: AppState) -> dict:
    results = user_validator.invoke(state["users"])
    valid = [u for u in results if u["email_valid"]]
    invalid = [u for u in results if not u["email_valid"]]
    return {"valid_users": valid, "invalid_users": invalid}

builder = StateGraph(AppState)
builder.add_node("validate", validate_users)
builder.add_edge(START, "validate")
app = builder.compile()

# Execute nested workflow
result = app.invoke({
    "users": [
        {"name": "Alice", "email": "alice@example.com"},
        {"name": "Bob", "email": "invalid-email"},
        {"name": "Charlie", "email": "charlie@test.org"}
    ]
})

print(f"Valid users: {len(result['valid_users'])}")
print(f"Invalid users: {len(result['invalid_users'])}")
```

---

## Complete Examples

### Example 1: Document Processing Pipeline

```python
from langgraph.func import task, entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command, RetryPolicy, CachePolicy
import time

@task(retry_policy=RetryPolicy(max_attempts=3))
def fetch_document(doc_id: str) -> dict:
    """Simulate fetching a document from external API."""
    time.sleep(0.1)
    return {
        "id": doc_id,
        "content": f"Content of document {doc_id}",
        "metadata": {"author": "Alice", "date": "2024-01-01"}
    }

@task(cache_policy=CachePolicy(ttl=300))
def extract_entities(content: str) -> list[str]:
    """Extract named entities from content."""
    time.sleep(0.2)
    # Simplified entity extraction
    words = content.split()
    return [w for w in words if w[0].isupper()]

@task
def analyze_sentiment(content: str) -> str:
    """Analyze sentiment of content."""
    time.sleep(0.15)
    # Simplified sentiment analysis
    positive_words = ["good", "great", "excellent"]
    return "positive" if any(w in content.lower() for w in positive_words) else "neutral"

@task
def generate_summary(content: str, max_length: int = 50) -> str:
    """Generate a summary of the content."""
    if len(content) <= max_length:
        return content
    return content[:max_length] + "..."

@entrypoint(checkpointer=InMemorySaver())
def document_pipeline(
    doc_ids: list[str],
    *,
    previous: list[dict] | None = None
) -> entrypoint.final[list[dict], dict]:
    """Process multiple documents in parallel."""

    # Track processed documents
    processed_history = previous or {}

    # Fetch all documents in parallel
    doc_futures = [fetch_document(doc_id) for doc_id in doc_ids]
    documents = [f.result() for f in doc_futures]

    # Process each document in parallel
    results = []
    for doc in documents:
        content = doc["content"]

        # Run analysis tasks in parallel
        entities_future = extract_entities(content)
        sentiment_future = analyze_sentiment(content)
        summary_future = generate_summary(content)

        # Collect results
        result = {
            "id": doc["id"],
            "entities": entities_future.result(),
            "sentiment": sentiment_future.result(),
            "summary": summary_future.result(),
            "metadata": doc["metadata"]
        }
        results.append(result)

    # Request human review for documents with negative sentiment
    needs_review = [r for r in results if r["sentiment"] != "positive"]
    if needs_review:
        review = interrupt({
            "question": "Review documents with non-positive sentiment",
            "documents": needs_review
        })
        # Update based on review
        for doc in results:
            if doc["id"] in review:
                doc["reviewed"] = True
                doc["review_notes"] = review[doc["id"]]

    # Update history
    for result in results:
        processed_history[result["id"]] = {
            "processed_at": "2024-01-01",
            "sentiment": result["sentiment"]
        }

    return entrypoint.final(value=results, save=processed_history)


# Usage
config = {"configurable": {"thread_id": "pipeline_1"}}

# Process documents
result = document_pipeline.invoke(["doc_1", "doc_2", "doc_3"], config)

# If there are interrupts, handle them
if "__interrupt__" in result:
    print("Documents need review!")
    # Resume with review
    review_data = {
        "doc_2": "Looks good despite neutral sentiment"
    }
    result = document_pipeline.invoke(Command(resume=review_data), config)

print(f"Processed {len(result)} documents")
```

### Example 2: Multi-Stage Data Pipeline with Error Handling

```python
from langgraph.func import task, entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import RetryPolicy
from typing import TypedDict
import random

class DataRecord(TypedDict):
    id: str
    value: float
    status: str

@task(retry_policy=RetryPolicy(
    max_attempts=3,
    initial_interval=0.5,
    backoff_factor=2.0
))
def fetch_data(source: str) -> list[DataRecord]:
    """Fetch data from external source with retry on failure."""
    # Simulate occasional failures
    if random.random() < 0.2:
        raise ConnectionError(f"Failed to connect to {source}")

    return [
        {"id": f"{source}_1", "value": 10.5, "status": "raw"},
        {"id": f"{source}_2", "value": 20.3, "status": "raw"},
    ]

@task
def validate_record(record: DataRecord) -> DataRecord:
    """Validate a single data record."""
    if record["value"] < 0:
        record["status"] = "invalid"
    else:
        record["status"] = "valid"
    return record

@task
def transform_record(record: DataRecord) -> DataRecord:
    """Transform valid records."""
    if record["status"] == "valid":
        record["value"] = record["value"] * 1.1  # Apply 10% markup
        record["status"] = "transformed"
    return record

@task
def load_to_destination(records: list[DataRecord], destination: str) -> dict:
    """Load records to destination."""
    return {
        "destination": destination,
        "loaded_count": len(records),
        "status": "success"
    }

@entrypoint(checkpointer=InMemorySaver())
def etl_pipeline(
    sources: list[str],
    destination: str,
    *,
    previous: dict | None = None
) -> dict:
    """ETL pipeline with error handling and checkpointing."""

    # Track metrics
    metrics = previous or {
        "total_processed": 0,
        "total_errors": 0,
        "runs": 0
    }

    # Extract: Fetch from all sources in parallel
    fetch_futures = [fetch_data(source) for source in sources]
    all_records = []
    for future in fetch_futures:
        try:
            records = future.result()
            all_records.extend(records)
        except Exception as e:
            print(f"Failed to fetch data: {e}")
            metrics["total_errors"] += 1

    if not all_records:
        return {"error": "No data fetched", "metrics": metrics}

    # Transform: Validate and transform in parallel
    validate_futures = [validate_record(record) for record in all_records]
    validated_records = [f.result() for f in validate_futures]

    transform_futures = [transform_record(record) for record in validated_records]
    transformed_records = [f.result() for f in transform_futures]

    # Filter to only transformed records
    final_records = [r for r in transformed_records if r["status"] == "transformed"]

    # Load: Write to destination
    load_result = load_to_destination(final_records, destination).result()

    # Update metrics
    metrics["total_processed"] += len(final_records)
    metrics["runs"] += 1

    return {
        "load_result": load_result,
        "metrics": metrics,
        "processed_records": len(final_records),
        "invalid_records": len([r for r in validated_records if r["status"] == "invalid"])
    }


# Usage
config = {"configurable": {"thread_id": "etl_run_1"}}

result = etl_pipeline.invoke(
    sources=["source_a", "source_b", "source_c"],
    destination="warehouse",
    config=config
)

print(f"Processed: {result['processed_records']} records")
print(f"Total runs: {result['metrics']['runs']}")
```

### Example 3: Agent Workflow with Tools

```python
from langgraph.func import task, entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt, Command
from typing import Any
import json

@task
def search_web(query: str) -> list[dict]:
    """Simulate web search."""
    return [
        {"title": f"Result for {query}", "url": "http://example.com", "snippet": "..."},
    ]

@task
def read_document(url: str) -> str:
    """Simulate reading a document."""
    return f"Content from {url}"

@task
def calculate(expression: str) -> float:
    """Evaluate a mathematical expression."""
    return eval(expression)

@task
def generate_response(context: dict, query: str) -> str:
    """Generate a response based on context."""
    return f"Based on {len(context)} sources, here's the answer to '{query}': ..."

@entrypoint(checkpointer=InMemorySaver())
def agent_workflow(
    user_query: str,
    *,
    previous: list[dict] | None = None
) -> entrypoint.final[str, list[dict]]:
    """Agent that uses tools to answer questions."""

    # Track conversation history
    history = previous or []

    # Step 1: Determine which tool to use
    needs_search = "search" in user_query.lower() or "find" in user_query.lower()
    needs_calculation = any(op in user_query for op in ["+", "-", "*", "/", "calculate"])

    context = {}

    # Step 2: Execute tools in parallel if needed
    futures = []

    if needs_search:
        search_future = search_web(user_query)
        futures.append(("search", search_future))

    if needs_calculation:
        # Extract expression (simplified)
        calc_future = calculate("2 + 2")
        futures.append(("calc", calc_future))

    # Collect tool results
    for tool_name, future in futures:
        context[tool_name] = future.result()

    # Step 3: Generate response
    response = generate_response(context, user_query).result()

    # Step 4: Request human feedback
    feedback = interrupt({
        "question": "Was this response helpful?",
        "response": response,
        "query": user_query
    })

    # Step 5: Update history
    history.append({
        "query": user_query,
        "response": response,
        "feedback": feedback,
        "tools_used": list(context.keys())
    })

    return entrypoint.final(value=response, save=history)


# Usage
config = {"configurable": {"thread_id": "agent_session_1"}}

# Ask question
result = agent_workflow.invoke("Search for information about Python", config)

# Provide feedback
result = agent_workflow.invoke(Command(resume="yes"), config)
print(result)
```

### Example 4: Async Batch Processing

```python
import asyncio
from langgraph.func import task, entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from typing import TypedDict

class ImageMetadata(TypedDict):
    id: str
    size: tuple[int, int]
    format: str

@task
async def download_image(url: str) -> bytes:
    """Simulate downloading an image."""
    await asyncio.sleep(0.1)
    return b"fake_image_data"

@task
async def resize_image(image_data: bytes, size: tuple[int, int]) -> bytes:
    """Simulate resizing an image."""
    await asyncio.sleep(0.05)
    return b"resized_image_data"

@task
async def extract_metadata(image_data: bytes) -> ImageMetadata:
    """Extract metadata from image."""
    await asyncio.sleep(0.03)
    return {
        "id": "img_123",
        "size": (800, 600),
        "format": "JPEG"
    }

@task
async def upload_image(image_data: bytes, destination: str) -> str:
    """Upload processed image."""
    await asyncio.sleep(0.1)
    return f"{destination}/image_123.jpg"

@entrypoint(checkpointer=InMemorySaver())
async def image_processing_pipeline(
    image_urls: list[str],
    target_size: tuple[int, int],
    destination: str
) -> list[dict]:
    """Process multiple images in parallel."""

    results = []

    for url in image_urls:
        # Download
        image_data_future = download_image(url)

        # Process in parallel after download
        image_data = await image_data_future

        resize_future = resize_image(image_data, target_size)
        metadata_future = extract_metadata(image_data)

        # Wait for both
        resized_data, metadata = await asyncio.gather(
            resize_future,
            metadata_future
        )

        # Upload
        upload_url = await upload_image(resized_data, destination)

        results.append({
            "original_url": url,
            "processed_url": upload_url,
            "metadata": metadata
        })

    return results


# Usage
async def main():
    config = {"configurable": {"thread_id": "batch_1"}}

    urls = [
        "http://example.com/img1.jpg",
        "http://example.com/img2.jpg",
        "http://example.com/img3.jpg",
    ]

    results = await image_processing_pipeline.ainvoke(
        image_urls=urls,
        target_size=(800, 600),
        destination="s3://my-bucket",
        config=config
    )

    print(f"Processed {len(results)} images")

# Run
asyncio.run(main())
```

---

## API Reference Summary

### Decorators

#### `@entrypoint(checkpointer=None, store=None, cache=None, context_schema=None, cache_policy=None, retry_policy=None)`

Converts a function into a LangGraph workflow.

**Returns:** `Pregel` graph instance

#### `@task(name=None, retry_policy=None, cache_policy=None)`

Defines a task that returns a future for parallel execution.

**Returns:** `_TaskFunction` callable

### Return Types

#### `SyncAsyncFuture[T]`

Future object returned by task calls.

**Methods:**
- `.result()` - Block until task completes and return result

#### `entrypoint.final[R, S]`

Primitive for decoupling return value from saved value.

**Fields:**
- `value: R` - Value to return to caller
- `save: S` - Value to save in checkpoint

### Functions

#### `interrupt(value: Any) -> Any`

Pause execution and request human input.

**Parameters:**
- `value` - Data to send to client

**Returns:** Resume value provided by client

### Types

#### `RetryPolicy`

Configuration for retrying tasks/workflows on failure.

**Fields:**
- `initial_interval: float = 0.5`
- `backoff_factor: float = 2.0`
- `max_interval: float = 128.0`
- `max_attempts: int = 3`
- `jitter: bool = True`
- `retry_on: Exception | Sequence[Exception] | Callable`

#### `CachePolicy`

Configuration for caching task results.

**Fields:**
- `key_func: Callable` - Generate cache key from inputs
- `ttl: int | None = None` - Time to live in seconds

#### `Command`

Primitive for resuming workflows and controlling execution.

**Fields:**
- `graph: str | None = None` - Target graph
- `update: Any | None = None` - State update
- `resume: dict[str, Any] | Any | None = None` - Resume values
- `goto: Send | Sequence[Send | N] | N = ()` - Navigation

---

## Best Practices

### 1. Use Type Hints

Always annotate function parameters and return types for better IDE support:

```python
@task
def process_data(data: str, limit: int = 10) -> dict:
    return {"processed": data, "limit": limit}

@entrypoint()
def workflow(input: str) -> dict:
    return process_data(input).result()
```

### 2. Handle Errors Gracefully

Use retry policies and error handling:

```python
@task(retry_policy=RetryPolicy(max_attempts=3))
def unreliable_task(data: str) -> str:
    try:
        return external_api_call(data)
    except Exception as e:
        logging.error(f"Task failed: {e}")
        raise
```

### 3. Leverage Parallel Execution

Group independent tasks for concurrent execution:

```python
@entrypoint()
def optimized_workflow(items: list[str]) -> dict:
    # Bad: Sequential execution
    # results = [process_item(item).result() for item in items]

    # Good: Parallel execution
    futures = [process_item(item) for item in items]
    results = [f.result() for f in futures]

    return {"results": results}
```

### 4. Use Checkpointers for Long-Running Workflows

Enable checkpointing for workflows that may be interrupted:

```python
@entrypoint(checkpointer=SqliteSaver.from_conn_string("workflow.db"))
def long_running_workflow(data: str) -> str:
    # Workflow state is persisted
    return data
```

### 5. Cache Expensive Operations

Use cache policies for expensive or repetitive computations:

```python
@task(cache_policy=CachePolicy(ttl=3600))
def expensive_computation(x: int) -> int:
    # This result is cached for 1 hour
    return complex_calculation(x)
```

### 6. Design for Observability

Structure workflows to make debugging easier:

```python
@task(name="fetch_user_data")
def fetch_user(user_id: str) -> dict:
    logging.info(f"Fetching user {user_id}")
    return {"id": user_id}

@entrypoint()
def observable_workflow(user_id: str) -> dict:
    user = fetch_user(user_id).result()
    logging.info(f"Processing user: {user}")
    return user
```

---

## Migration Guide

### From StateGraph to Functional API

**StateGraph Version:**
```python
from langgraph.graph import StateGraph, START
from typing_extensions import TypedDict

class State(TypedDict):
    items: list[str]
    results: list[str]

def process_items(state: State) -> dict:
    results = [item.upper() for item in state["items"]]
    return {"results": results}

builder = StateGraph(State)
builder.add_node("process", process_items)
builder.add_edge(START, "process")
graph = builder.compile()

result = graph.invoke({"items": ["a", "b", "c"]})
```

**Functional API Version:**
```python
from langgraph.func import task, entrypoint

@task
def process_item(item: str) -> str:
    return item.upper()

@entrypoint()
def workflow(items: list[str]) -> list[str]:
    futures = [process_item(item) for item in items]
    return [f.result() for f in futures]

result = workflow.invoke(["a", "b", "c"])
```

### From Functional API to StateGraph

**Functional API Version:**
```python
from langgraph.func import task, entrypoint

@task
def step_a(x: int) -> int:
    return x + 1

@task
def step_b(x: int) -> int:
    return x * 2

@entrypoint()
def workflow(x: int) -> int:
    result_a = step_a(x).result()
    result_b = step_b(result_a).result()
    return result_b
```

**StateGraph Version:**
```python
from langgraph.graph import StateGraph, START
from typing_extensions import TypedDict

class State(TypedDict):
    value: int

def step_a(state: State) -> dict:
    return {"value": state["value"] + 1}

def step_b(state: State) -> dict:
    return {"value": state["value"] * 2}

builder = StateGraph(State)
builder.add_node("step_a", step_a)
builder.add_node("step_b", step_b)
builder.add_edge(START, "step_a")
builder.add_edge("step_a", "step_b")
graph = builder.compile()

result = graph.invoke({"value": 5})
```

---

## Troubleshooting

### Common Issues

**Issue: Tasks not executing in parallel**

```python
# ❌ Wrong - sequential execution
@entrypoint()
def workflow(items: list[str]) -> list[str]:
    results = []
    for item in items:
        result = process_item(item).result()  # Blocks immediately
        results.append(result)
    return results

# ✅ Correct - parallel execution
@entrypoint()
def workflow(items: list[str]) -> list[str]:
    futures = [process_item(item) for item in items]
    return [f.result() for f in futures]
```

**Issue: Missing checkpointer for stateful workflows**

```python
# ❌ Wrong - previous is always None
@entrypoint()
def workflow(x: int, *, previous: int | None = None) -> int:
    return (previous or 0) + x

# ✅ Correct - checkpointer enabled
@entrypoint(checkpointer=InMemorySaver())
def workflow(x: int, *, previous: int | None = None) -> int:
    return (previous or 0) + x
```

**Issue: Interrupt not working**

```python
# ❌ Wrong - no checkpointer
@entrypoint()
def workflow(data: str) -> str:
    approval = interrupt("Approve?")  # Error!
    return data

# ✅ Correct - checkpointer required for interrupts
@entrypoint(checkpointer=InMemorySaver())
def workflow(data: str) -> str:
    approval = interrupt("Approve?")
    return data
```

**Issue: Type errors with entrypoint.final**

```python
# ❌ Wrong - missing type parameters
@entrypoint()
def workflow(x: int) -> entrypoint.final:
    return entrypoint.final(value=x, save=x*2)

# ✅ Correct - type parameters specified
@entrypoint()
def workflow(x: int) -> entrypoint.final[int, int]:
    return entrypoint.final(value=x, save=x*2)
```

---

## Performance Considerations

### Task Granularity

**Too Fine-Grained:**
```python
# Creates overhead with many tiny tasks
@task
def add(a: int, b: int) -> int:
    return a + b

@entrypoint()
def workflow(numbers: list[int]) -> int:
    # Creates hundreds of tasks
    total = 0
    for n in numbers:
        total = add(total, n).result()
    return total
```

**Optimal Granularity:**
```python
# Better - batch operations
@task
def sum_batch(numbers: list[int]) -> int:
    return sum(numbers)

@entrypoint()
def workflow(numbers: list[int], batch_size: int = 100) -> int:
    batches = [numbers[i:i+batch_size] for i in range(0, len(numbers), batch_size)]
    futures = [sum_batch(batch) for batch in batches]
    return sum(f.result() for f in futures)
```

### Caching Strategy

Use caching for expensive, deterministic operations:

```python
@task(cache_policy=CachePolicy(ttl=3600))
def expensive_lookup(key: str) -> dict:
    # Results cached for 1 hour
    return database_query(key)
```

### Checkpoint Frequency

Balance durability with performance:

```python
# For critical workflows
@entrypoint(checkpointer=PostgresSaver.from_conn_string(...))
def critical_workflow(data: str) -> str:
    # Every task is checkpointed
    return data

# For high-throughput workflows
@entrypoint()  # No checkpointer = faster
def fast_workflow(data: str) -> str:
    return data
```

---

This documentation provides a comprehensive guide to the LangGraph Functional API, covering all major concepts, patterns, and best practices for building workflows with decorators.
