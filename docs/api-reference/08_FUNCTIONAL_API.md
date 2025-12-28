# LangGraph Functional API Reference

This document provides comprehensive API documentation for the LangGraph Functional API module, which enables building stateful workflows using Python decorators and a functional programming style.

## Table of Contents

1. [Core Decorators](#core-decorators)
   - [entrypoint](#entrypoint)
   - [task](#task)
2. [entrypoint Class Members](#entrypoint-class-members)
   - [entrypoint.final](#entrypointfinal)
3. [Task Functions](#task-functions)
   - [_TaskFunction](#_taskfunction)
   - [_TaskFunction.clear_cache](#_taskfunctionclear_cache)
   - [_TaskFunction.aclear_cache](#_taskfunctionaclear_cache)
4. [Future Objects](#future-objects)
   - [SyncAsyncFuture](#syncasyncfuture)
   - [SyncAsyncFuture.result](#syncasyncfutureresult)
5. [Supporting Types](#supporting-types)
   - [Runtime](#runtime)
   - [StreamWriter](#streamwriter)
   - [RetryPolicy](#retrypolicy)
   - [CachePolicy](#cachepolicy)
6. [Utility Functions](#utility-functions)
   - [interrupt](#interrupt)
   - [get_runtime](#get_runtime)

---

## Core Decorators

### entrypoint

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:228`

Define a LangGraph workflow using the `entrypoint` decorator. This decorator converts a regular Python function into a compiled LangGraph `Pregel` graph that can be invoked, streamed, and checkpointed.

**Signature:**
```python
class entrypoint(Generic[ContextT]):
    def __init__(
        self,
        checkpointer: BaseCheckpointSaver | None = None,
        store: BaseStore | None = None,
        cache: BaseCache | None = None,
        context_schema: type[ContextT] | None = None,
        cache_policy: CachePolicy | None = None,
        retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
    ) -> None:
```

**Description:**

The `entrypoint` decorator creates a workflow that can be executed with `.invoke()`, `.stream()`, `.ainvoke()`, and `.astream()`. The decorated function must accept a single parameter as input. To pass multiple values, use a dictionary.

The function can optionally request injectable parameters that will be provided automatically at runtime:
- `config`: RunnableConfig object with runtime configuration
- `previous`: Previous return value for the thread (requires checkpointer)
- `runtime`: Runtime object containing context, store, and writer

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| checkpointer | BaseCheckpointSaver \| None | No | None | A checkpointer to persist workflow state across runs. Enables access to `previous` parameter and interrupts. |
| store | BaseStore \| None | No | None | A generalized key-value store with optional semantic search capabilities. |
| cache | BaseCache \| None | No | None | A cache for storing task results. |
| context_schema | type[ContextT] \| None | No | None | Schema for the context object passed to the workflow. Defines structure for run-scoped context data like `user_id`. |
| cache_policy | CachePolicy \| None | No | None | Policy for caching workflow results. |
| retry_policy | RetryPolicy \| Sequence[RetryPolicy] \| None | No | None | Policy or list of policies for retrying the workflow on failure. |

**Returns:**

| Type | Description |
|------|-------------|
| Pregel | A compiled graph that can be invoked with `.invoke()`, `.stream()`, `.ainvoke()`, `.astream()` |

**Usage:**
```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def my_workflow(input_data: str) -> str:
    return f"Processed: {input_data}"
```

**Example: Basic entrypoint**
```python
from langgraph.func import entrypoint, task

@task
def add_one(n: int) -> int:
    return n + 1

@entrypoint()
def process_numbers(numbers: list[int]) -> list[int]:
    futures = [add_one(n) for n in numbers]
    results = [f.result() for f in futures]
    return results

# Call the entrypoint
result = process_numbers.invoke([1, 2, 3])
# Returns [2, 3, 4]
```

**Example: Using previous state**
```python
from typing import Optional
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

@entrypoint(checkpointer=InMemorySaver())
def counter(increment: int, previous: Optional[int] = None) -> int:
    current = previous or 0
    return current + increment

config = {"configurable": {"thread_id": "thread-1"}}
counter.invoke(1, config)  # Returns 1
counter.invoke(2, config)  # Returns 3
counter.invoke(5, config)  # Returns 8
```

**Example: Using context schema**
```python
from dataclasses import dataclass
from langgraph.func import entrypoint
from langgraph.runtime import Runtime

@dataclass
class Context:
    user_id: str
    api_key: str

@entrypoint(context_schema=Context)
def personalized_greeting(name: str, runtime: Runtime[Context]) -> str:
    user_id = runtime.context.user_id
    return f"Hello {name}! Your user ID is {user_id}"

result = personalized_greeting.invoke(
    "Alice",
    context=Context(user_id="user123", api_key="secret")
)
# Returns "Hello Alice! Your user ID is user123"
```

**Example: Interrupts for human-in-the-loop**
```python
from langgraph.func import entrypoint, task
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@task
def compose_essay(topic: str) -> str:
    return f"An essay about {topic}"

@entrypoint(checkpointer=InMemorySaver())
def review_workflow(topic: str) -> dict:
    essay_future = compose_essay(topic)
    essay = essay_future.result()

    # Interrupt for human review
    human_review = interrupt({
        "question": "Please provide a review",
        "essay": essay
    })

    return {
        "essay": essay,
        "review": human_review,
    }

config = {"configurable": {"thread_id": "some_thread"}}

# First execution - will interrupt
for result in review_workflow.stream("cats", config):
    print(result)

# Resume with human review
for result in review_workflow.stream(Command(resume="Great essay!"), config):
    print(result)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:228-564`

---

### task

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:115`

Define a LangGraph task using the `task` decorator. Tasks are parallelizable units of work that can be called from within an entrypoint or StateGraph.

**Signature:**
```python
def task(
    __func_or_none__: Callable[P, Awaitable[T]] | Callable[P, T] | None = None,
    *,
    name: str | None = None,
    retry_policy: RetryPolicy | Sequence[RetryPolicy] | None = None,
    cache_policy: CachePolicy[Callable[P, str | bytes]] | None = None,
) -> Callable[[Callable[P, Awaitable[T]] | Callable[P, T]], _TaskFunction[P, T]] | _TaskFunction[P, T]
```

**Description:**

The `task` decorator wraps a function to create a parallelizable task. When called, it returns a future object instead of executing immediately, enabling parallel execution patterns.

Key characteristics:
- Supports both sync and async functions (async requires Python 3.11+)
- Returns a `SyncAsyncFuture` when called, not the actual result
- Call `.result()` on the future to get the actual value
- Can only be called from within an entrypoint or StateGraph
- Inputs and outputs must be serializable when checkpointer is enabled

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| name | str \| None | No | None | Optional name for the task. If not provided, uses the function name. |
| retry_policy | RetryPolicy \| Sequence[RetryPolicy] \| None | No | None | Retry policy or list of policies to apply on task failure. |
| cache_policy | CachePolicy[Callable[P, str \| bytes]] \| None | No | None | Cache policy to use for caching task results. |

**Returns:**

| Type | Description |
|------|-------------|
| _TaskFunction[P, T] | A wrapped task function that returns futures when called |

**Usage:**
```python
@task
def my_task(a: int, b: int) -> int:
    return a + b

@task(name="custom_name", retry_policy=RetryPolicy(max_attempts=5))
def my_task_with_options(x: str) -> str:
    return x.upper()
```

**Example: Sync task**
```python
from langgraph.func import entrypoint, task

@task
def add_one(n: int) -> int:
    return n + 1

@entrypoint()
def process_list(numbers: list[int]) -> list[int]:
    # Execute tasks in parallel
    futures = [add_one(n) for n in numbers]
    # Collect results
    results = [f.result() for f in futures]
    return results

result = process_list.invoke([1, 2, 3])
# Returns [2, 3, 4]
```

**Example: Async task (Python 3.11+)**
```python
import asyncio
from langgraph.func import entrypoint, task

@task
async def fetch_data(url: str) -> dict:
    # Simulate async operation
    await asyncio.sleep(0.1)
    return {"url": url, "data": "..."}

@entrypoint()
async def fetch_all(urls: list[str]) -> list[dict]:
    futures = [fetch_data(url) for url in urls]
    # For async tasks, can use asyncio.gather
    results = await asyncio.gather(*futures)
    return results

# Call the async entrypoint
result = await fetch_all.ainvoke([
    "https://api1.com",
    "https://api2.com"
])
```

**Example: Task with retry policy**
```python
from langgraph.func import task, entrypoint
from langgraph.types import RetryPolicy

@task(retry_policy=RetryPolicy(max_attempts=3, initial_interval=1.0))
def unreliable_task(value: int) -> int:
    # This task will retry up to 3 times on failure
    import random
    if random.random() < 0.5:
        raise ValueError("Random failure")
    return value * 2

@entrypoint()
def run_unreliable(value: int) -> int:
    future = unreliable_task(value)
    return future.result()
```

**Example: Task with cache policy**
```python
from langgraph.func import task, entrypoint
from langgraph.types import CachePolicy
from langgraph.cache.memory import InMemoryCache

def custom_cache_key(url: str) -> str:
    return f"fetch:{url}"

@task(cache_policy=CachePolicy(key_func=custom_cache_key, ttl=3600))
def fetch_expensive_data(url: str) -> dict:
    # This result will be cached for 1 hour
    return {"url": url, "data": "expensive"}

@entrypoint(cache=InMemoryCache())
def get_data(url: str) -> dict:
    future = fetch_expensive_data(url)
    return future.result()

# First call executes the task
result1 = get_data.invoke("https://api.com")
# Second call uses cached result
result2 = get_data.invoke("https://api.com")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:115-218`

---

## entrypoint Class Members

### entrypoint.final

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:431`

A dataclass that can be returned from an entrypoint to specify different values for the return and the saved checkpoint state.

**Signature:**
```python
@dataclass
class final(Generic[R, S]):
    value: R
    save: S
```

**Description:**

The `entrypoint.final` class allows you to decouple the value returned to the caller from the value saved to the checkpoint. This is useful when you want to return one value to the user but save different state for the next invocation.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| value | R | Value to return to the caller. Always returned even if `None`. |
| save | S | Value to save to the checkpoint. Will be accessible via `previous` parameter in the next invocation. Always saved even if `None`. |

**Usage:**
```python
@entrypoint(checkpointer=InMemorySaver())
def my_workflow(
    number: int,
    *,
    previous: Any = None,
) -> entrypoint.final[int, int]:
    previous = previous or 0
    return entrypoint.final(value=previous, save=2 * number)
```

**Example: Separating return and save values**
```python
from typing import Any
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint

@entrypoint(checkpointer=InMemorySaver())
def accumulator(
    number: int,
    *,
    previous: Any = None,
) -> entrypoint.final[int, int]:
    """Returns the previous accumulated value, saves the new accumulated value."""
    previous = previous or 0
    # Return the old value to caller, save new accumulated value
    return entrypoint.final(value=previous, save=previous + number)

config = {"configurable": {"thread_id": "thread-1"}}

result1 = accumulator.invoke(3, config)  # Returns 0, saves 3
result2 = accumulator.invoke(5, config)  # Returns 3, saves 8
result3 = accumulator.invoke(2, config)  # Returns 8, saves 10
```

**Example: Type annotations with entrypoint.final**
```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def process(
    data: dict,
    previous: list[str] | None = None
) -> entrypoint.final[dict, list[str]]:
    """Returns processed data, saves history of operations."""
    history = previous or []
    new_history = history + [f"processed {data.get('id')}"]

    processed = {
        "result": data.get("value", 0) * 2,
        "history_length": len(new_history)
    }

    return entrypoint.final(value=processed, save=new_history)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:431-469`

---

## Task Functions

### _TaskFunction

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:46`

Internal wrapper class created by the `@task` decorator. Users don't instantiate this directly but receive instances when using `@task`.

**Signature:**
```python
class _TaskFunction(Generic[P, T]):
    def __init__(
        self,
        func: Callable[P, Awaitable[T]] | Callable[P, T],
        *,
        retry_policy: Sequence[RetryPolicy],
        cache_policy: CachePolicy[Callable[P, str | bytes]] | None = None,
        name: str | None = None,
    ) -> None:
```

**Description:**

This class wraps task functions and provides additional functionality like caching and metadata. When called, it returns a `SyncAsyncFuture` instead of executing the function directly.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| func | Callable[P, Awaitable[T]] \| Callable[P, T] | The original function being wrapped |
| retry_policy | Sequence[RetryPolicy] | Retry policies to apply on failure |
| cache_policy | CachePolicy[Callable[P, str \| bytes]] \| None | Cache policy for the task |

**Methods:**

- `__call__(*args, **kwargs) -> SyncAsyncFuture[T]`: Execute the task and return a future
- `clear_cache(cache: BaseCache) -> None`: Clear the cache for this task (sync)
- `aclear_cache(cache: BaseCache) -> None`: Clear the cache for this task (async)

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:46-91`

---

### _TaskFunction.clear_cache

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:80`

Clear the cache for this task (synchronous version).

**Signature:**
```python
def clear_cache(self, cache: BaseCache) -> None:
```

**Description:**

Clears all cached results for this task from the provided cache. Only works if the task has a cache_policy configured.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| cache | BaseCache | Yes | - | The cache instance to clear |

**Returns:**

| Type | Description |
|------|-------------|
| None | No return value |

**Usage:**
```python
from langgraph.cache.memory import InMemoryCache
from langgraph.func import task
from langgraph.types import CachePolicy

cache = InMemoryCache()

@task(cache_policy=CachePolicy(ttl=3600))
def expensive_task(x: int) -> int:
    return x * 2

# Clear the cache for this specific task
expensive_task.clear_cache(cache)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:80-83`

---

### _TaskFunction.aclear_cache

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:85`

Clear the cache for this task (asynchronous version).

**Signature:**
```python
async def aclear_cache(self, cache: BaseCache) -> None:
```

**Description:**

Asynchronously clears all cached results for this task from the provided cache. Only works if the task has a cache_policy configured.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| cache | BaseCache | Yes | - | The cache instance to clear |

**Returns:**

| Type | Description |
|------|-------------|
| None | No return value |

**Usage:**
```python
from langgraph.cache.memory import InMemoryCache
from langgraph.func import task
from langgraph.types import CachePolicy

cache = InMemoryCache()

@task(cache_policy=CachePolicy(ttl=3600))
async def expensive_task(x: int) -> int:
    return x * 2

# Async clear the cache for this specific task
await expensive_task.aclear_cache(cache)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/func/__init__.py:85-90`

---

## Future Objects

### SyncAsyncFuture

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_call.py:248`

A future object returned by task functions that supports both synchronous and asynchronous access patterns.

**Signature:**
```python
class SyncAsyncFuture(Generic[T], concurrent.futures.Future[T]):
    def __await__(self) -> Generator[T, None, T]:
```

**Description:**

This class extends `concurrent.futures.Future` to work in both sync and async contexts. You can either call `.result()` to get the value synchronously, or `await` the future directly in async code.

**Methods:**

- `result()`: Get the result synchronously (blocks if not ready)
- `__await__()`: Make the future awaitable in async contexts

**Usage:**
```python
# Synchronous usage
future = my_task(42)
result = future.result()

# Asynchronous usage
future = my_task(42)
result = await future
```

**Example: Sync access**
```python
from langgraph.func import entrypoint, task

@task
def compute(x: int) -> int:
    return x * 2

@entrypoint()
def process(value: int) -> int:
    future = compute(value)
    # Get result synchronously
    result = future.result()
    return result + 1

result = process.invoke(5)  # Returns 11
```

**Example: Async access**
```python
from langgraph.func import entrypoint, task

@task
async def compute(x: int) -> int:
    return x * 2

@entrypoint()
async def process(value: int) -> int:
    future = compute(value)
    # Await the future directly
    result = await future
    return result + 1

result = await process.ainvoke(5)  # Returns 11
```

**Example: Parallel execution**
```python
from langgraph.func import entrypoint, task

@task
def process_item(item: str) -> str:
    return item.upper()

@entrypoint()
def process_batch(items: list[str]) -> list[str]:
    # Create futures for parallel execution
    futures = [process_item(item) for item in items]
    # Collect results
    results = [f.result() for f in futures]
    return results

result = process_batch.invoke(["a", "b", "c"])
# Returns ["A", "B", "C"]
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/pregel/_call.py:248-250`

---

### SyncAsyncFuture.result

**Signature:**
```python
def result(timeout: float | None = None) -> T:
```

**Description:**

Get the result of the future. This method blocks until the result is available. Inherited from `concurrent.futures.Future`.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| timeout | float \| None | No | None | Maximum time in seconds to wait for the result. If None, waits indefinitely. |

**Returns:**

| Type | Description |
|------|-------------|
| T | The result of the task execution |

**Raises:**

| Exception | When |
|-----------|------|
| TimeoutError | If timeout is specified and result is not available within that time |
| Exception | Any exception raised by the task during execution |

**Usage:**
```python
future = my_task(42)
result = future.result()  # Wait indefinitely
result = future.result(timeout=5.0)  # Wait max 5 seconds
```

---

## Supporting Types

### Runtime

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/runtime.py:28`

Convenience class that bundles run-scoped context and other runtime utilities. Available as an injectable parameter in entrypoints and nodes.

**Signature:**
```python
@dataclass
class Runtime(Generic[ContextT]):
    context: ContextT = field(default=None)
    store: BaseStore | None = field(default=None)
    stream_writer: StreamWriter = field(default=_no_op_stream_writer)
    previous: Any = field(default=None)
```

**Description:**

The `Runtime` object provides access to runtime resources like context, store, and stream writer. It's automatically injected when requested as a parameter named `runtime` in your entrypoint or node functions.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| context | ContextT | Static context for the graph run, like `user_id`, `db_conn`, etc. |
| store | BaseStore \| None | Store for the graph run, enabling persistence and memory |
| stream_writer | StreamWriter | Function that writes to the custom stream (for `stream_mode="custom"`) |
| previous | Any | Previous return value for the given thread (functional API with checkpointer only) |

**Methods:**

| Method | Description |
|--------|-------------|
| merge(other: Runtime[ContextT]) -> Runtime[ContextT] | Merge two runtimes, preferring values from `other` |
| override(**overrides) -> Runtime[ContextT] | Create a new runtime with specified overrides |

**Usage:**
```python
from langgraph.func import entrypoint
from langgraph.runtime import Runtime

@dataclass
class Context:
    user_id: str

@entrypoint(context_schema=Context)
def my_workflow(input: str, runtime: Runtime[Context]) -> str:
    user_id = runtime.context.user_id
    return f"Processing {input} for {user_id}"
```

**Example: Using context**
```python
from dataclasses import dataclass
from langgraph.func import entrypoint
from langgraph.runtime import Runtime

@dataclass
class Context:
    user_id: str
    api_key: str

@entrypoint(context_schema=Context)
def authenticated_request(
    endpoint: str,
    runtime: Runtime[Context]
) -> dict:
    api_key = runtime.context.api_key
    user_id = runtime.context.user_id
    return {
        "endpoint": endpoint,
        "user": user_id,
        "authenticated": bool(api_key)
    }

result = authenticated_request.invoke(
    "/api/data",
    context=Context(user_id="user123", api_key="secret")
)
```

**Example: Using store**
```python
from typing import TypedDict
from dataclasses import dataclass
from langgraph.func import entrypoint
from langgraph.runtime import Runtime
from langgraph.store.memory import InMemoryStore

@dataclass
class Context:
    user_id: str

class State(TypedDict):
    message: str

store = InMemoryStore()

@entrypoint(context_schema=Context, store=store)
def personalized_greeting(
    name: str,
    runtime: Runtime[Context]
) -> str:
    user_id = runtime.context.user_id

    # Access store
    if runtime.store:
        memory = runtime.store.get(("users",), user_id)
        if memory:
            return f"Welcome back, {memory.value['name']}!"

    return f"Hello, {name}!"
```

**Example: Using stream_writer**
```python
from langgraph.func import entrypoint
from langgraph.runtime import Runtime

@entrypoint()
def streaming_process(data: list[int], runtime: Runtime) -> int:
    total = 0
    for item in data:
        total += item
        # Write to custom stream
        runtime.stream_writer({"progress": total})
    return total

# Stream with mode "custom" to see the custom events
for event in streaming_process.stream([1, 2, 3], stream_mode="custom"):
    print(event)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/runtime.py:28-122`

---

### StreamWriter

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:107`

A callable that writes custom data to the output stream when using `stream_mode="custom"`.

**Signature:**
```python
StreamWriter = Callable[[Any], None]
```

**Description:**

StreamWriter is a function injected into nodes and entrypoints when requested as a keyword argument. It's a no-op unless `stream_mode="custom"` is used. Allows you to emit custom events during execution.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| value | Any | Yes | - | The custom data to write to the stream |

**Returns:**

| Type | Description |
|------|-------------|
| None | No return value |

**Usage:**
```python
def my_node(state: State, writer: StreamWriter) -> State:
    writer({"event": "processing", "step": 1})
    # ... do work ...
    writer({"event": "complete", "step": 1})
    return state
```

**Example: Custom streaming in functional API**
```python
from langgraph.func import entrypoint, task
from langgraph.runtime import Runtime

@task
def process_item(item: str, runtime: Runtime) -> str:
    runtime.stream_writer({"processing": item})
    result = item.upper()
    runtime.stream_writer({"processed": result})
    return result

@entrypoint()
def process_all(items: list[str], runtime: Runtime) -> list[str]:
    runtime.stream_writer({"started": len(items)})
    futures = [process_item(item, runtime=runtime) for item in items]
    results = [f.result() for f in futures]
    runtime.stream_writer({"completed": len(results)})
    return results

# Stream with mode "custom"
for event in process_all.stream(["a", "b"], stream_mode="custom"):
    print(event)
```

**Example: Progress tracking**
```python
from langgraph.func import entrypoint
from langgraph.runtime import Runtime

@entrypoint()
def long_process(total: int, runtime: Runtime) -> int:
    result = 0
    for i in range(total):
        result += i
        # Emit progress updates
        runtime.stream_writer({
            "progress": i + 1,
            "total": total,
            "percentage": ((i + 1) / total) * 100
        })
    return result

for event in long_process.stream(100, stream_mode="custom"):
    if "progress" in event:
        print(f"Progress: {event['percentage']:.1f}%")
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:107-110`

---

### RetryPolicy

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:115`

Configuration for retrying tasks and entrypoints on failure.

**Signature:**
```python
class RetryPolicy(NamedTuple):
    initial_interval: float = 0.5
    backoff_factor: float = 2.0
    max_interval: float = 128.0
    max_attempts: int = 3
    jitter: bool = True
    retry_on: (
        type[Exception] | Sequence[type[Exception]] | Callable[[Exception], bool]
    ) = default_retry_on
```

**Description:**

RetryPolicy defines how tasks and entrypoints should retry on failure, with exponential backoff, jitter, and configurable exception handling.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| initial_interval | float | No | 0.5 | Amount of time (seconds) before the first retry occurs |
| backoff_factor | float | No | 2.0 | Multiplier by which the interval increases after each retry |
| max_interval | float | No | 128.0 | Maximum time (seconds) between retries |
| max_attempts | int | No | 3 | Maximum number of attempts including the first |
| jitter | bool | No | True | Whether to add random jitter to retry intervals |
| retry_on | type[Exception] \| Sequence[type[Exception]] \| Callable[[Exception], bool] | No | default_retry_on | Exception types to retry on, or a callable that returns True for retryable exceptions |

**Returns:**

| Type | Description |
|------|-------------|
| RetryPolicy | A configured retry policy |

**Usage:**
```python
from langgraph.types import RetryPolicy

# Basic retry policy
policy = RetryPolicy(max_attempts=5)

# Custom retry policy
policy = RetryPolicy(
    initial_interval=1.0,
    backoff_factor=3.0,
    max_interval=60.0,
    max_attempts=10,
    jitter=False
)

# Retry only on specific exceptions
policy = RetryPolicy(
    max_attempts=5,
    retry_on=(ValueError, TypeError)
)

# Retry based on custom logic
def should_retry(exc: Exception) -> bool:
    return isinstance(exc, ValueError) and "temporary" in str(exc)

policy = RetryPolicy(retry_on=should_retry)
```

**Example: Task with retry policy**
```python
from langgraph.func import task, entrypoint
from langgraph.types import RetryPolicy

@task(retry_policy=RetryPolicy(max_attempts=5, initial_interval=1.0))
def flaky_api_call(url: str) -> dict:
    # This will retry up to 5 times with exponential backoff
    import requests
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

@entrypoint()
def fetch_data(url: str) -> dict:
    future = flaky_api_call(url)
    return future.result()
```

**Example: Multiple retry policies**
```python
from langgraph.types import RetryPolicy
from langgraph.func import entrypoint

# Apply different retry strategies
policies = [
    RetryPolicy(max_attempts=3, initial_interval=0.1),  # Fast retries
    RetryPolicy(max_attempts=2, initial_interval=5.0),  # Slower retries
]

@entrypoint(retry_policy=policies)
def resilient_workflow(data: str) -> str:
    # Will first do 3 fast retries, then 2 slower retries
    return data.upper()
```

**Example: Conditional retry**
```python
from langgraph.types import RetryPolicy
from langgraph.func import task

def retry_on_rate_limit(exc: Exception) -> bool:
    """Only retry on rate limit errors."""
    return isinstance(exc, Exception) and "rate limit" in str(exc).lower()

@task(retry_policy=RetryPolicy(
    max_attempts=10,
    initial_interval=60.0,
    retry_on=retry_on_rate_limit
))
def api_with_rate_limit(endpoint: str) -> dict:
    # Only retries on rate limit errors, not other exceptions
    ...
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:115-134`

---

### CachePolicy

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:140`

Configuration for caching task results.

**Signature:**
```python
@dataclass
class CachePolicy(Generic[KeyFuncT]):
    key_func: KeyFuncT = default_cache_key
    ttl: int | None = None
```

**Description:**

CachePolicy defines how task results should be cached, including the cache key generation function and time-to-live.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| key_func | KeyFuncT | No | default_cache_key | Function to generate a cache key from the task's input. Defaults to hashing the input with pickle. |
| ttl | int \| None | No | None | Time to live for the cache entry in seconds. If None, the entry never expires. |

**Returns:**

| Type | Description |
|------|-------------|
| CachePolicy | A configured cache policy |

**Usage:**
```python
from langgraph.types import CachePolicy

# Default cache policy (pickle-based key, no expiration)
policy = CachePolicy()

# Cache with TTL
policy = CachePolicy(ttl=3600)  # Expire after 1 hour

# Custom cache key function
def my_key_func(arg1: str, arg2: int) -> str:
    return f"cache:{arg1}:{arg2}"

policy = CachePolicy(key_func=my_key_func, ttl=1800)
```

**Example: Basic caching**
```python
from langgraph.func import task, entrypoint
from langgraph.types import CachePolicy
from langgraph.cache.memory import InMemoryCache

@task(cache_policy=CachePolicy(ttl=3600))
def expensive_computation(n: int) -> int:
    # This result will be cached for 1 hour
    import time
    time.sleep(2)  # Simulate expensive work
    return n * n

@entrypoint(cache=InMemoryCache())
def compute(n: int) -> int:
    future = expensive_computation(n)
    return future.result()

# First call takes 2 seconds
result1 = compute.invoke(10)

# Second call returns instantly from cache
result2 = compute.invoke(10)
```

**Example: Custom cache key**
```python
from langgraph.func import task, entrypoint
from langgraph.types import CachePolicy
from langgraph.cache.memory import InMemoryCache

def url_cache_key(url: str, params: dict) -> str:
    """Generate cache key from URL and params."""
    import json
    param_str = json.dumps(params, sort_keys=True)
    return f"url:{url}:{param_str}"

@task(cache_policy=CachePolicy(key_func=url_cache_key, ttl=300))
def fetch_url(url: str, params: dict) -> dict:
    # Cached by URL and params for 5 minutes
    import requests
    response = requests.get(url, params=params)
    return response.json()

@entrypoint(cache=InMemoryCache())
def get_data(url: str, params: dict) -> dict:
    future = fetch_url(url, params)
    return future.result()
```

**Example: Clearing cache**
```python
from langgraph.cache.memory import InMemoryCache
from langgraph.func import task
from langgraph.types import CachePolicy

cache = InMemoryCache()

@task(cache_policy=CachePolicy(ttl=3600))
def cached_task(x: int) -> int:
    return x * 2

# Clear cache for this specific task
cached_task.clear_cache(cache)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:140-149`

---

## Utility Functions

### interrupt

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:416`

Interrupt the graph execution with a resumable exception from within a node or entrypoint, enabling human-in-the-loop workflows.

**Signature:**
```python
def interrupt(value: Any) -> Any:
```

**Description:**

The `interrupt` function pauses graph execution and surfaces a value to the client. The client can then resume execution by providing a resume value via the `Command` primitive. The node re-executes from the start when resumed.

Key behaviors:
- First invocation raises a `GraphInterrupt` exception and halts execution
- The provided `value` is sent to the client with the exception
- Requires a checkpointer to be enabled
- Node re-executes from the beginning when resumed
- Multiple interrupts in the same node are matched to resume values by order

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| value | Any | Yes | - | The value to surface to the client when interrupted. Can be any serializable data. |

**Returns:**

| Type | Description |
|------|-------------|
| Any | On subsequent invocations (after resume), returns the value provided by the client |

**Raises:**

| Exception | When |
|-----------|------|
| GraphInterrupt | On the first invocation, halts execution and surfaces the value to the client |

**Usage:**
```python
from langgraph.types import interrupt

# In a node or entrypoint
human_input = interrupt({"question": "What should I do next?"})
```

**Example: Basic interrupt**
```python
from langgraph.func import entrypoint
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def approval_workflow(request: str) -> dict:
    # Process request
    processed = f"Processed: {request}"

    # Ask for approval
    approval = interrupt({
        "question": "Do you approve?",
        "request": processed
    })

    return {
        "request": processed,
        "approved": approval
    }

config = {"configurable": {"thread_id": "thread-1"}}

# First call - will interrupt
for event in approval_workflow.stream("Create user", config):
    print(event)
    # Output: {'__interrupt__': (Interrupt(value={'question': 'Do you approve?', ...}),)}

# Resume with approval
for event in approval_workflow.stream(Command(resume=True), config):
    print(event)
    # Output: final result with approved=True
```

**Example: Multiple interrupts**
```python
from langgraph.func import entrypoint
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@entrypoint(checkpointer=InMemorySaver())
def multi_approval_workflow(document: str) -> dict:
    # First approval
    tech_approval = interrupt({
        "stage": "technical",
        "question": "Technical review approved?"
    })

    # Second approval
    business_approval = interrupt({
        "stage": "business",
        "question": "Business review approved?"
    })

    return {
        "document": document,
        "tech_approved": tech_approval,
        "business_approved": business_approval
    }

config = {"configurable": {"thread_id": "thread-1"}}

# First interrupt
for event in multi_approval_workflow.stream("Contract", config):
    print(event)

# Resume first interrupt
for event in multi_approval_workflow.stream(Command(resume=True), config):
    print(event)  # Will hit second interrupt

# Resume second interrupt
for event in multi_approval_workflow.stream(Command(resume=True), config):
    print(event)  # Final result
```

**Example: Interrupt with rich data**
```python
from langgraph.func import entrypoint, task
from langgraph.types import interrupt, Command
from langgraph.checkpoint.memory import InMemorySaver

@task
def generate_report(data: dict) -> str:
    return f"Report for {data['topic']}: ..."

@entrypoint(checkpointer=InMemorySaver())
def report_workflow(topic: str) -> dict:
    # Generate report
    report = generate_report({"topic": topic}).result()

    # Request human review with rich context
    review = interrupt({
        "type": "review_request",
        "report": report,
        "instructions": "Please review and provide feedback",
        "metadata": {
            "topic": topic,
            "length": len(report),
            "timestamp": "2024-01-01"
        }
    })

    return {
        "report": report,
        "review": review,
        "status": "complete"
    }

config = {"configurable": {"thread_id": "thread-1"}}

# Interrupt with rich data
for event in report_workflow.stream("AI Safety", config):
    if "__interrupt__" in event:
        interrupt_data = event["__interrupt__"][0].value
        print(f"Review needed for: {interrupt_data['metadata']['topic']}")

# Resume with review
review_feedback = {"rating": 5, "comments": "Excellent work"}
for event in report_workflow.stream(Command(resume=review_feedback), config):
    print(event)
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/types.py:416-539`

---

### get_runtime

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/runtime.py:132`

Get the Runtime object for the current graph execution.

**Signature:**
```python
def get_runtime(context_schema: type[ContextT] | None = None) -> Runtime[ContextT]:
```

**Description:**

Retrieves the Runtime object for the current graph run. This function can be called from within nodes or tasks to access runtime context, store, and other utilities without requiring the `runtime` parameter.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| context_schema | type[ContextT] \| None | No | None | Optional schema for type hinting the return type of the runtime |

**Returns:**

| Type | Description |
|------|-------------|
| Runtime[ContextT] | The runtime object for the current graph run |

**Usage:**
```python
from langgraph.runtime import get_runtime

def my_node(state: State) -> State:
    runtime = get_runtime()
    user_id = runtime.context.user_id
    return state
```

**Example: Accessing runtime without injection**
```python
from dataclasses import dataclass
from langgraph.func import entrypoint, task
from langgraph.runtime import get_runtime, Runtime

@dataclass
class Context:
    user_id: str
    session_id: str

@task
def process_data(data: str) -> str:
    # Get runtime without parameter injection
    runtime = get_runtime(Context)
    user_id = runtime.context.user_id

    return f"{data} processed by {user_id}"

@entrypoint(context_schema=Context)
def workflow(data: str) -> str:
    future = process_data(data)
    return future.result()

result = workflow.invoke(
    "test",
    context=Context(user_id="user123", session_id="session456")
)
```

**Example: Using get_runtime in helper functions**
```python
from langgraph.runtime import get_runtime
from langgraph.func import entrypoint

def log_action(action: str) -> None:
    """Helper function that accesses runtime context."""
    runtime = get_runtime()
    if runtime.context:
        print(f"[{runtime.context.user_id}] {action}")

@entrypoint(context_schema=Context)
def workflow_with_logging(data: str) -> str:
    log_action("Starting workflow")
    result = data.upper()
    log_action("Workflow complete")
    return result
```

**Source:** `/home/user/langgraph/libs/langgraph/langgraph/runtime.py:132-146`

---

## Notes and Best Practices

### Function Signature Requirements

**Entrypoint functions:**
- Must accept at least one parameter (the input)
- Can optionally request `config`, `previous`, or `runtime` as injectable parameters
- Return type can be any type, or `entrypoint.final[R, S]` to separate return and save values

**Task functions:**
- Can have any signature
- Will receive actual arguments when called (not futures)
- Should return serializable values when checkpointer is enabled

### Parallelization Patterns

Tasks enable easy parallelization:

```python
@task
def process(item: str) -> str:
    return item.upper()

@entrypoint()
def parallel_process(items: list[str]) -> list[str]:
    # All tasks execute in parallel
    futures = [process(item) for item in items]
    # Wait for all results
    return [f.result() for f in futures]
```

### Checkpointing and State

When a checkpointer is enabled:
- Task results are automatically cached across interrupts
- Access previous return value via `previous` parameter
- Use `entrypoint.final` to decouple return and save values
- Interrupts require a checkpointer to function

### Error Handling

Use retry policies for automatic error recovery:

```python
@task(retry_policy=RetryPolicy(
    max_attempts=5,
    initial_interval=1.0,
    retry_on=(ConnectionError, TimeoutError)
))
def resilient_task(url: str) -> dict:
    # Automatically retries on connection errors
    ...
```

### Caching

Cache expensive computations:

```python
@task(cache_policy=CachePolicy(ttl=3600))
def expensive_task(input: str) -> str:
    # Result cached for 1 hour
    ...
```

Clear cache when needed:

```python
expensive_task.clear_cache(cache)
await expensive_task.aclear_cache(cache)
```

### Streaming

Use custom streaming to emit progress updates:

```python
@entrypoint()
def streaming_workflow(data: list, runtime: Runtime) -> int:
    for i, item in enumerate(data):
        runtime.stream_writer({"progress": i, "total": len(data)})
        # Process item...
    return len(data)

# Stream with mode "custom"
for event in streaming_workflow.stream(data, stream_mode="custom"):
    print(event)
```

### Version Information

- Functional API available since: v0.2.0
- `Runtime` class added in: v0.6.0
- `context_schema` parameter added in: v0.6.0 (replaces deprecated `config_schema`)
- Async task support requires Python 3.11+
