# Testing Guide for LangGraph

This guide provides comprehensive documentation for testing in the LangGraph project, covering infrastructure, patterns, best practices, and examples.

---

## Table of Contents

1. [Testing Infrastructure](#1-testing-infrastructure)
2. [Running Tests](#2-running-tests)
3. [Test Fixtures](#3-test-fixtures)
4. [Testing StateGraphs](#4-testing-stategraphs)
5. [Testing Nodes](#5-testing-nodes)
6. [Testing with Checkpointers](#6-testing-with-checkpointers)
7. [Async Testing](#7-async-testing)
8. [Mocking LLMs](#8-mocking-llms)
9. [Snapshot Testing](#9-snapshot-testing)
10. [Integration Tests](#10-integration-tests)
11. [Test Patterns](#11-test-patterns)
12. [Writing New Tests](#12-writing-new-tests)

---

## 1. Testing Infrastructure

### Pytest Setup

LangGraph uses pytest as its primary testing framework with several plugins configured in `pyproject.toml`:

```toml
[tool.pytest.ini_options]
addopts = "--full-trace --strict-markers --strict-config --durations=5 --snapshot-warn-unused"
```

### Key Testing Dependencies

From `pyproject.toml` dependency groups:

```toml
[dependency-groups]
test = [
    "pytest",               # Core testing framework
    "pytest-cov",          # Coverage reporting
    "pytest-dotenv",       # Environment variable support
    "pytest-mock",         # Mocking utilities
    "pytest-xdist[psutil]", # Parallel test execution
    "pytest-repeat",       # Test repetition
    "pytest-watcher",      # File watching for continuous testing
    "syrupy",             # Snapshot testing
    "httpx",              # HTTP client for API tests
]
```

### Pytest Plugins Used

1. **pytest-asyncio**: For async test support
   - All async tests are marked with `pytestmark = pytest.mark.anyio`

2. **pytest-mock**: For mocking and patching
   - Provides `MockerFixture` for creating mocks

3. **syrupy**: For snapshot testing
   - Used to compare complex outputs against saved snapshots

4. **pytest-xdist**: For parallel test execution
   - Run tests with `-n auto` for parallel execution

5. **pytest-parametrize**: Built-in parametrization
   - Used extensively for testing multiple configurations

### Test Configuration

The `anyio` backend is configured in `conftest.py`:

```python
@pytest.fixture
def anyio_backend():
    return "asyncio"
```

### Docker-Dependent Tests

Tests check for Docker availability:

```python
NO_DOCKER = os.getenv("NO_DOCKER", "false") == "true"
```

When Docker is unavailable, tests automatically skip Postgres and Redis implementations.

---

## 2. Running Tests

### Basic Test Execution

From the library directory (`libs/langgraph/`):

```bash
# Run all tests
make test

# Run specific test file
TEST=tests/test_pregel.py make test

# Run specific test function
TEST=tests/test_pregel.py::test_graph_validation make test

# Run with pytest options
TEST="tests/test_pregel.py -v -s" make test
```

### Advanced Test Commands

```bash
# Run tests in parallel
make test_parallel

# Watch mode (re-run tests on file changes)
make test_watch

# Run with custom workers and max failures
WORKERS=4 MAXFAIL=1 make test_watch

# Run without Docker (skips Postgres/Redis tests)
NO_DOCKER=true make test

# Run coverage report
make coverage
```

### Test Services Management

```bash
# Start Docker services (Postgres, Redis)
make start-services

# Stop Docker services
make stop-services

# Start dev server for integration tests
make start-dev-server

# Stop dev server
make stop-dev-server
```

### Running Specific Test Categories

```bash
# Integration tests only
make integration_tests

# Linting alongside tests
make lint && make test

# Format code before running tests
make format && make test
```

---

## 3. Test Fixtures

LangGraph provides comprehensive fixtures for testing different components.

### 3.1 Checkpointer Fixtures

#### Synchronous Checkpointers

Located in `/home/user/langgraph/libs/langgraph/tests/conftest.py`:

```python
@pytest.fixture(
    scope="function",
    params=[
        "memory",
        "memory_migrate_sends",
        "sqlite",
        "sqlite_aes",
        "postgres",
        "postgres_pipe",
        "postgres_pool",
    ],
)
def sync_checkpointer(request: pytest.FixtureRequest) -> Iterator[BaseCheckpointSaver]:
    checkpointer_name = request.param
    if checkpointer_name == "memory":
        with _checkpointer_memory() as checkpointer:
            yield checkpointer
    # ... other implementations
```

**Available sync checkpointers:**
- `memory`: In-memory checkpointer
- `memory_migrate_sends`: Tests pending sends migration
- `sqlite`: SQLite checkpointer (in-memory)
- `sqlite_aes`: SQLite with AES encryption
- `postgres`: PostgreSQL checkpointer (requires Docker)
- `postgres_pipe`: PostgreSQL with pipeline mode
- `postgres_pool`: PostgreSQL with connection pool

#### Async Checkpointers

```python
@pytest.fixture(
    scope="function",
    params=[
        "memory",
        "sqlite_aio",
        "postgres_aio",
        "postgres_aio_pipe",
        "postgres_aio_pool",
    ],
)
async def async_checkpointer(request: pytest.FixtureRequest) -> AsyncIterator[BaseCheckpointSaver]:
    # Implementation details...
```

#### Example Usage

```python
def test_with_checkpointer(sync_checkpointer: BaseCheckpointSaver):
    """This test runs once for each checkpointer implementation."""
    builder = StateGraph(State)
    builder.add_node("node", lambda x: x)
    builder.add_edge(START, "node")
    graph = builder.compile(checkpointer=sync_checkpointer)

    result = graph.invoke(
        {"value": "test"},
        {"configurable": {"thread_id": "1"}}
    )

    # Verify checkpoint was saved
    checkpoint = sync_checkpointer.get_tuple({"configurable": {"thread_id": "1"}})
    assert checkpoint is not None
```

### 3.2 Store Fixtures

#### Synchronous Stores

```python
@pytest.fixture(
    scope="function",
    params=["in_memory", "postgres", "postgres_pipe", "postgres_pool"]
)
def sync_store(request: pytest.FixtureRequest) -> Iterator[BaseStore]:
    store_name = request.param
    if store_name == "in_memory":
        with _store_memory() as store:
            yield store
    # ... other implementations
```

#### Async Stores

```python
@pytest.fixture(
    scope="function",
    params=["in_memory", "postgres_aio", "postgres_aio_pipe", "postgres_aio_pool"]
)
async def async_store(request: pytest.FixtureRequest) -> AsyncIterator[BaseStore]:
    # Implementation details...
```

#### Example Usage

```python
def test_store_operations(sync_store: BaseStore):
    """Test runs for each store implementation."""
    namespace = ("test", "documents")

    # Put operation
    sync_store.put(namespace, "key1", {"data": "value1"})

    # Get operation
    item = sync_store.get(namespace, "key1")
    assert item.value == {"data": "value1"}

    # Search operation
    results = sync_store.search(("test",))
    assert len(results) >= 1
```

### 3.3 Cache Fixtures

```python
@pytest.fixture(
    scope="function",
    params=["sqlite", "memory", "redis"]
)
def cache(request: pytest.FixtureRequest) -> Iterator[BaseCache]:
    if request.param == "sqlite":
        yield SqliteCache(path=":memory:")
    elif request.param == "memory":
        yield InMemoryCache()
    elif request.param == "redis":
        # Redis with worker-specific prefix for parallel test isolation
        worker_id = getattr(request.config, "workerinput", {}).get("workerid", "master")
        redis_client = redis.Redis(host="localhost", port=6379, db=0)
        cache = RedisCache(redis_client, prefix=f"test:cache:{worker_id}:")
        yield cache
        # Cleanup
        pattern = f"test:cache:{worker_id}:*"
        keys = redis_client.keys(pattern)
        if keys:
            redis_client.delete(*keys)
```

### 3.4 Utility Fixtures

#### Deterministic UUIDs

```python
@pytest.fixture()
def deterministic_uuids(mocker: MockerFixture) -> MockerFixture:
    """Generate deterministic UUIDs for reproducible tests."""
    side_effect = (
        UUID(f"00000000-0000-4000-8000-{i:012}", version=4)
        for i in range(10000)
    )
    return mocker.patch("uuid.uuid4", side_effect=side_effect)
```

**Usage:**
```python
def test_with_deterministic_ids(deterministic_uuids):
    # All uuid.uuid4() calls return predictable values
    id1 = uuid.uuid4()
    id2 = uuid.uuid4()
    assert str(id1) == "00000000-0000-4000-8000-000000000000"
    assert str(id2) == "00000000-0000-4000-8000-000000000001"
```

#### Durability Fixture

```python
@pytest.fixture(params=["sync", "async", "exit"])
def durability(request: pytest.FixtureRequest) -> Durability:
    return request.param
```

**Usage:**
```python
def test_with_durability(durability: Durability):
    """Test runs 3 times with different durability modes."""
    graph = builder.compile(checkpointer=checkpointer)
    result = graph.invoke(input, durability=durability)
```

---

## 4. Testing StateGraphs

### Basic StateGraph Testing

```python
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    messages: list[str]
    count: int

def test_basic_graph():
    """Test a simple StateGraph execution."""
    def node_a(state: State) -> State:
        return {"count": state["count"] + 1}

    def node_b(state: State) -> State:
        return {"messages": state["messages"] + ["processed"]}

    # Build graph
    builder = StateGraph(State)
    builder.add_node("a", node_a)
    builder.add_node("b", node_b)
    builder.add_edge(START, "a")
    builder.add_edge("a", "b")
    builder.add_edge("b", END)

    graph = builder.compile()

    # Test execution
    result = graph.invoke({"messages": [], "count": 0})

    assert result == {
        "messages": ["processed"],
        "count": 1
    }
```

### Testing Graph Validation

```python
def test_graph_validation():
    """Test that invalid graphs raise appropriate errors."""
    class State(TypedDict):
        value: str

    builder = StateGraph(State)
    builder.add_node("start", lambda x: x)
    builder.add_edge(START, "start")
    builder.add_edge("unknown_node", "start")  # Invalid edge
    builder.add_edge("start", END)

    with pytest.raises(ValueError, match="Found edge starting at unknown node"):
        builder.compile()
```

### Testing Conditional Edges

```python
def test_conditional_routing():
    """Test graph with conditional edges."""
    class State(TypedDict):
        value: int
        path: str

    def router(state: State) -> str:
        return "high" if state["value"] > 10 else "low"

    def high_node(state: State) -> State:
        return {"path": "high"}

    def low_node(state: State) -> State:
        return {"path": "low"}

    builder = StateGraph(State)
    builder.add_node("high", high_node)
    builder.add_node("low", low_node)
    builder.add_conditional_edges(START, router, {"high": "high", "low": "low"})
    builder.add_edge("high", END)
    builder.add_edge("low", END)

    graph = builder.compile()

    # Test high path
    result = graph.invoke({"value": 15, "path": ""})
    assert result["path"] == "high"

    # Test low path
    result = graph.invoke({"value": 5, "path": ""})
    assert result["path"] == "low"
```

### Testing with Command

```python
def test_graph_with_command():
    """Test Command for dynamic routing."""
    from langgraph.types import Command

    class State(TypedDict):
        foo: str
        bar: str

    def node_a(state: State):
        return Command(goto="b", update={"foo": "bar"})

    def node_b(state: State):
        return Command(goto=END, update={"bar": "baz"})

    builder = StateGraph(State)
    builder.add_node("a", node_a)
    builder.add_node("b", node_b)
    builder.add_edge(START, "a")

    graph = builder.compile()

    result = graph.invoke({"foo": "", "bar": ""})
    assert result == {"foo": "bar", "bar": "baz"}
```

### Testing Graph Output Schema

```python
def test_graph_output_schema():
    """Test graph with custom output schema."""
    class State(TypedDict):
        hello: str
        bye: str
        internal: str

    class Output(TypedDict):
        hello: str
        bye: str

    def node(state: State) -> State:
        return {"hello": "world", "internal": "hidden"}

    builder = StateGraph(State, output_schema=Output)
    builder.add_node("node", node)
    builder.add_edge(START, "node")
    builder.add_edge("node", END)

    graph = builder.compile()

    result = graph.invoke({"hello": "", "bye": "", "internal": ""})

    # Output schema excludes 'internal'
    assert "internal" not in result
    assert result["hello"] == "world"
```

---

## 5. Testing Nodes

### Unit Testing Individual Nodes

```python
def test_node_function():
    """Test a node function in isolation."""
    from typing_extensions import TypedDict

    class State(TypedDict):
        value: int
        result: int

    def multiply_by_two(state: State) -> State:
        return {"result": state["value"] * 2}

    # Test node directly
    state = {"value": 5, "result": 0}
    result = multiply_by_two(state)

    assert result == {"result": 10}
```

### Testing Nodes with Type Validation

```python
def test_node_type_validation():
    """Test node with specific input/output types."""
    from typing_extensions import TypedDict

    class InputState(TypedDict):
        input: str

    class OutputState(TypedDict):
        output: str
        processed: bool

    def processor(state: InputState) -> OutputState:
        return {
            "output": state["input"].upper(),
            "processed": True
        }

    result = processor({"input": "hello"})
    assert result["output"] == "HELLO"
    assert result["processed"] is True
```

### Testing Nodes with Dependencies

```python
def test_node_with_external_dependency():
    """Test node that depends on external service."""
    from unittest.mock import Mock

    class State(TypedDict):
        query: str
        response: str

    def api_node(state: State, api_client) -> State:
        response = api_client.fetch(state["query"])
        return {"response": response}

    # Mock the API client
    mock_client = Mock()
    mock_client.fetch.return_value = "mocked response"

    # Create partial function with mocked dependency
    from functools import partial
    test_node = partial(api_node, api_client=mock_client)

    result = test_node({"query": "test", "response": ""})
    assert result["response"] == "mocked response"
    mock_client.fetch.assert_called_once_with("test")
```

### Testing Node Error Handling

```python
def test_node_error_handling():
    """Test node error handling."""
    class State(TypedDict):
        value: int
        error: str | None

    def risky_node(state: State) -> State:
        try:
            if state["value"] < 0:
                raise ValueError("Negative value not allowed")
            return {"error": None}
        except ValueError as e:
            return {"error": str(e)}

    # Test success case
    result = risky_node({"value": 5, "error": None})
    assert result["error"] is None

    # Test error case
    result = risky_node({"value": -1, "error": None})
    assert result["error"] == "Negative value not allowed"
```

### Testing Nodes with Streaming

```python
def test_node_with_streaming():
    """Test node that uses stream writer."""
    from langgraph.config import get_stream_writer
    from langgraph.graph import StateGraph, MessagesState, START
    from langchain_core.messages import HumanMessage

    def streaming_node(state: MessagesState):
        writer = get_stream_writer()
        for i in range(3):
            writer({"custom_event": f"step_{i}"})
        return {"messages": []}

    builder = StateGraph(MessagesState)
    builder.add_node("stream", streaming_node)
    builder.add_edge(START, "stream")
    graph = builder.compile()

    events = list(graph.stream(
        {"messages": [HumanMessage("test")]},
        stream_mode="custom"
    ))

    assert len(events) == 3
    assert events[0] == {"custom_event": "step_0"}
    assert events[1] == {"custom_event": "step_1"}
    assert events[2] == {"custom_event": "step_2"}
```

---

## 6. Testing with Checkpointers

### Basic Checkpoint Testing

```python
def test_checkpoint_saves_state(sync_checkpointer: BaseCheckpointSaver):
    """Test that state is properly checkpointed."""
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
        value: str

    builder = StateGraph(State)
    builder.add_node("node", lambda x: x)
    builder.add_edge(START, "node")
    builder.add_edge("node", END)

    graph = builder.compile(checkpointer=sync_checkpointer)

    config = {"configurable": {"thread_id": "test-1"}}

    # Run graph
    result = graph.invoke({"value": "test"}, config)

    # Verify checkpoint exists
    checkpoint = sync_checkpointer.get_tuple(config)
    assert checkpoint is not None
    assert checkpoint.checkpoint["channel_values"]["value"] == "test"
```

### Testing Checkpoint Metadata

```python
async def test_checkpoint_metadata(async_checkpointer: BaseCheckpointSaver):
    """Test checkpoint metadata handling."""
    from langgraph.checkpoint.base import empty_checkpoint, create_checkpoint

    config = {
        "configurable": {
            "thread_id": "thread-1",
            "checkpoint_ns": "",
        },
        "metadata": {"run_id": "my_run_id"}
    }

    checkpoint = empty_checkpoint()
    metadata = {
        "source": "input",
        "step": 1,
        "writes": {},
    }

    await async_checkpointer.aput(config, checkpoint, metadata, {})

    # Retrieve and verify metadata
    retrieved = await async_checkpointer.aget_tuple(config)
    assert retrieved is not None
    assert retrieved.metadata["run_id"] == "my_run_id"
    assert retrieved.metadata["source"] == "input"
```

### Testing Checkpoint History

```python
def test_checkpoint_history(sync_checkpointer: BaseCheckpointSaver):
    """Test traversing checkpoint history."""
    class State(TypedDict):
        count: int

    def increment(state: State) -> State:
        return {"count": state["count"] + 1}

    builder = StateGraph(State)
    builder.add_node("inc", increment)
    builder.add_edge(START, "inc")
    builder.add_edge("inc", END)

    graph = builder.compile(checkpointer=sync_checkpointer)
    config = {"configurable": {"thread_id": "test-history"}}

    # Execute multiple times
    graph.invoke({"count": 0}, config)
    graph.invoke({"count": 1}, config)
    graph.invoke({"count": 2}, config)

    # Get all checkpoints
    checkpoints = list(sync_checkpointer.list(config))

    assert len(checkpoints) >= 3
```

### Testing Checkpoint Search/Filter

```python
async def test_checkpoint_search(async_checkpointer: BaseCheckpointSaver):
    """Test searching checkpoints by metadata."""
    from langgraph.checkpoint.base import empty_checkpoint

    configs = [
        {
            "configurable": {"thread_id": "thread-1", "checkpoint_ns": ""},
            "metadata": {"user": "alice"}
        },
        {
            "configurable": {"thread_id": "thread-2", "checkpoint_ns": ""},
            "metadata": {"user": "bob"}
        },
        {
            "configurable": {"thread_id": "thread-3", "checkpoint_ns": ""},
            "metadata": {"user": "alice"}
        }
    ]

    # Save checkpoints
    for config in configs:
        checkpoint = empty_checkpoint()
        metadata = {"source": "test", "step": 1, **config["metadata"]}
        await async_checkpointer.aput(config, checkpoint, metadata, {})

    # Search for alice's checkpoints
    alice_checkpoints = [
        c async for c in async_checkpointer.alist(None, filter={"user": "alice"})
    ]

    assert len(alice_checkpoints) == 2
```

### Testing Encrypted Checkpoints

```python
def test_encrypted_checkpointer():
    """Test checkpointer with encryption."""
    from langgraph.checkpoint.sqlite import SqliteSaver
    from langgraph.checkpoint.serde.encrypted import EncryptedSerializer

    with SqliteSaver.from_conn_string(":memory:") as checkpointer:
        # Add encryption
        checkpointer.serde = EncryptedSerializer.from_pycryptodome_aes(
            key=b"1234567890123456"
        )

        class State(TypedDict):
            secret: str

        builder = StateGraph(State)
        builder.add_node("node", lambda x: x)
        builder.add_edge(START, "node")
        builder.add_edge("node", END)

        graph = builder.compile(checkpointer=checkpointer)
        config = {"configurable": {"thread_id": "encrypted"}}

        # Save encrypted state
        graph.invoke({"secret": "sensitive_data"}, config)

        # Retrieve and verify
        checkpoint = checkpointer.get_tuple(config)
        assert checkpoint.checkpoint["channel_values"]["secret"] == "sensitive_data"
```

### Testing Checkpoint Immutability

The `MemorySaverAssertImmutable` class ensures checkpoints aren't modified after being saved:

```python
def test_checkpoint_immutability():
    """Test that checkpoints remain immutable."""
    from tests.memory_assert import MemorySaverAssertImmutable

    checkpointer = MemorySaverAssertImmutable()

    class State(TypedDict):
        data: dict

    builder = StateGraph(State)
    builder.add_node("node", lambda state: {"data": {"value": "modified"}})
    builder.add_edge(START, "node")
    builder.add_edge("node", END)

    graph = builder.compile(checkpointer=checkpointer)
    config = {"configurable": {"thread_id": "immutable"}}

    # This will raise AssertionError if checkpoint is mutated
    result = graph.invoke({"data": {"value": "original"}}, config)
```

---

## 7. Async Testing

### Basic Async Test

All async tests in LangGraph use the `anyio` marker:

```python
import pytest

pytestmark = pytest.mark.anyio

async def test_async_graph():
    """Basic async graph test."""
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
        value: str

    async def async_node(state: State) -> State:
        # Simulate async operation
        await asyncio.sleep(0.01)
        return {"value": state["value"].upper()}

    builder = StateGraph(State)
    builder.add_node("process", async_node)
    builder.add_edge(START, "process")
    builder.add_edge("process", END)

    graph = builder.compile()

    result = await graph.ainvoke({"value": "hello"})
    assert result["value"] == "HELLO"
```

### Testing Async Checkpointers

```python
async def test_async_checkpointer_operations(async_checkpointer: BaseCheckpointSaver):
    """Test async checkpointer save and retrieve."""
    from langgraph.checkpoint.base import empty_checkpoint, CheckpointMetadata

    config = {
        "configurable": {
            "thread_id": "async-test",
            "checkpoint_ns": "",
        }
    }

    checkpoint = empty_checkpoint()
    metadata: CheckpointMetadata = {
        "source": "input",
        "step": 1,
        "writes": {},
    }

    # Async save
    await async_checkpointer.aput(config, checkpoint, metadata, {})

    # Async retrieve
    retrieved = await async_checkpointer.aget_tuple(config)
    assert retrieved is not None
    assert retrieved.metadata["source"] == "input"
```

### Testing Async Stores

```python
async def test_async_store_operations(async_store: BaseStore):
    """Test async store put/get operations."""
    namespace = ("test", "async")

    # Async put
    await async_store.aput(namespace, "key1", {"data": "value1"})

    # Async get
    item = await async_store.aget(namespace, "key1")
    assert item is not None
    assert item.value == {"data": "value1"}

    # Async search
    results = await async_store.asearch(("test",))
    assert len(results) >= 1
```

### Testing Async Streaming

```python
async def test_async_streaming():
    """Test async graph streaming."""
    class State(TypedDict):
        messages: list[str]

    async def node_a(state: State) -> State:
        await asyncio.sleep(0.01)
        return {"messages": state["messages"] + ["a"]}

    async def node_b(state: State) -> State:
        await asyncio.sleep(0.01)
        return {"messages": state["messages"] + ["b"]}

    builder = StateGraph(State)
    builder.add_node("a", node_a)
    builder.add_node("b", node_b)
    builder.add_edge(START, "a")
    builder.add_edge("a", "b")
    builder.add_edge("b", END)

    graph = builder.compile()

    events = []
    async for event in graph.astream({"messages": []}):
        events.append(event)

    assert len(events) == 2
    assert "a" in events[0]
    assert "b" in events[1]
```

### Testing Concurrent Async Operations

```python
async def test_concurrent_graph_execution():
    """Test multiple concurrent graph executions."""
    import asyncio

    class State(TypedDict):
        value: int

    async def slow_node(state: State) -> State:
        await asyncio.sleep(0.1)
        return {"value": state["value"] * 2}

    builder = StateGraph(State)
    builder.add_node("process", slow_node)
    builder.add_edge(START, "process")
    builder.add_edge("process", END)

    graph = builder.compile()

    # Execute multiple graphs concurrently
    results = await asyncio.gather(
        graph.ainvoke({"value": 1}),
        graph.ainvoke({"value": 2}),
        graph.ainvoke({"value": 3}),
    )

    assert results[0]["value"] == 2
    assert results[1]["value"] == 4
    assert results[2]["value"] == 6
```

---

## 8. Mocking LLMs

### FakeToolCallingModel

LangGraph provides `FakeToolCallingModel` for testing agents without real LLM calls:

Located at `/home/user/langgraph/libs/prebuilt/tests/model.py`:

```python
from tests.model import FakeToolCallingModel
from langchain_core.messages import HumanMessage, AIMessage

def test_fake_model_basic():
    """Test basic fake model usage."""
    model = FakeToolCallingModel()

    messages = [HumanMessage("Hello"), HumanMessage("World")]
    result = model.invoke(messages)

    # Concatenates message contents with '-'
    assert result.content == "Hello-World"
```

### Mocking Tool Calls

```python
from langchain_core.messages import ToolCall

def test_fake_model_with_tools():
    """Test fake model that returns tool calls."""
    tool_calls = [
        [
            {"args": {"x": 1}, "id": "1", "name": "add"},
            {"args": {"x": 2}, "id": "2", "name": "multiply"}
        ],
        []  # Second call returns no tools (conversation ends)
    ]

    model = FakeToolCallingModel(tool_calls=tool_calls)

    # First invocation returns tool calls
    result1 = model.invoke([HumanMessage("Use tools")])
    assert len(result1.tool_calls) == 2
    assert result1.tool_calls[0]["name"] == "add"

    # Second invocation returns no tool calls
    result2 = model.invoke([HumanMessage("Done")])
    assert len(result2.tool_calls) == 0
```

### Testing React Agent with Mock Model

```python
from langgraph.prebuilt import create_react_agent
from langchain_core.tools import tool

def test_react_agent_with_mock():
    """Test react agent with mocked model."""
    @tool
    def get_weather(city: str) -> str:
        """Get weather for a city."""
        return f"Weather in {city}: Sunny, 72°F"

    # Configure model to call the tool
    tool_calls = [
        [{"args": {"city": "SF"}, "id": "1", "name": "get_weather"}],
        []  # End conversation after tool response
    ]

    model = FakeToolCallingModel(tool_calls=tool_calls)
    agent = create_react_agent(model, [get_weather])

    result = agent.invoke({
        "messages": [HumanMessage("What's the weather in SF?")]
    })

    # Verify tool was called
    tool_message = [m for m in result["messages"] if m.type == "tool"][0]
    assert "Sunny, 72°F" in tool_message.content
```

### Mocking Structured Output

```python
from pydantic import BaseModel, Field

def test_fake_model_structured_output():
    """Test fake model with structured output."""
    class WeatherResponse(BaseModel):
        temperature: float = Field(description="Temperature in Fahrenheit")
        condition: str = Field(description="Weather condition")

    expected_response = WeatherResponse(temperature=75.0, condition="Sunny")

    model = FakeToolCallingModel(
        tool_calls=[[]],
        structured_response=expected_response
    )

    agent = create_react_agent(
        model,
        [],
        response_format=WeatherResponse
    )

    result = agent.invoke({"messages": [HumanMessage("Get weather")]})

    assert result["structured_response"] == expected_response
    assert result["structured_response"].temperature == 75.0
```

### Using GenericFakeChatModel

For simpler cases without tools:

```python
from langchain_core.language_models import GenericFakeChatModel

def test_generic_fake_model():
    """Test with generic fake chat model."""
    model = GenericFakeChatModel(messages=iter([
        AIMessage(content="First response"),
        AIMessage(content="Second response"),
    ]))

    result1 = model.invoke([HumanMessage("Hello")])
    assert result1.content == "First response"

    result2 = model.invoke([HumanMessage("Hi again")])
    assert result2.content == "Second response"
```

---

## 9. Snapshot Testing

LangGraph uses `syrupy` for snapshot testing to verify complex outputs haven't changed.

### Basic Snapshot Test

```python
from syrupy import SnapshotAssertion

def test_graph_schema_snapshot(snapshot: SnapshotAssertion):
    """Test that graph schemas match snapshots."""
    import json

    class State(TypedDict):
        value: str

    builder = StateGraph(State)
    builder.add_node("node", lambda x: x)
    builder.add_edge(START, "node")
    builder.add_edge("node", END)

    graph = builder.compile()

    # Compare schemas against saved snapshots
    assert json.dumps(graph.get_input_jsonschema()) == snapshot
    assert json.dumps(graph.get_output_jsonschema()) == snapshot
    assert json.dumps(graph.get_graph().to_json(), indent=2) == snapshot
```

### Snapshot for Graph Visualization

```python
def test_graph_mermaid_snapshot(snapshot: SnapshotAssertion):
    """Test Mermaid diagram output."""
    builder = StateGraph(State)
    builder.add_node("start", lambda x: x)
    builder.add_node("process", lambda x: x)
    builder.add_node("end", lambda x: x)
    builder.add_edge(START, "start")
    builder.add_edge("start", "process")
    builder.add_edge("process", "end")
    builder.add_edge("end", END)

    graph = builder.compile()

    # Snapshot Mermaid diagram
    assert graph.get_graph().draw_mermaid(with_styles=False) == snapshot
```

### Managing Snapshots

```bash
# Update all snapshots (when intentional changes are made)
pytest --snapshot-update

# View snapshot warnings for unused snapshots
pytest  # Will show warnings due to --snapshot-warn-unused in config
```

### Snapshot Location

Snapshots are stored in `__snapshots__` directories next to test files:

```
tests/
  test_myfeature.py
  __snapshots__/
    test_myfeature.ambr
```

---

## 10. Integration Tests

### Testing with Real Services

Integration tests use Docker services (Postgres, Redis) via `make start-services`:

```python
def test_postgres_checkpointer_integration():
    """Integration test with real Postgres database."""
    # This test only runs when Docker is available
    if os.getenv("NO_DOCKER") == "true":
        pytest.skip("Docker not available")

    from langgraph.checkpoint.postgres import PostgresSaver
    from uuid import uuid4

    database = f"test_{uuid4().hex[:16]}"

    # Create database
    with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
        conn.execute(f"CREATE DATABASE {database}")

    try:
        # Test with real Postgres
        with PostgresSaver.from_conn_string(
            DEFAULT_POSTGRES_URI + database
        ) as checkpointer:
            checkpointer.setup()

            # Run actual graph operations
            graph = builder.compile(checkpointer=checkpointer)
            result = graph.invoke(
                {"value": "test"},
                {"configurable": {"thread_id": "integration-1"}}
            )

            # Verify persistence
            checkpoint = checkpointer.get_tuple(
                {"configurable": {"thread_id": "integration-1"}}
            )
            assert checkpoint is not None
    finally:
        # Cleanup
        with Connection.connect(DEFAULT_POSTGRES_URI, autocommit=True) as conn:
            conn.execute(f"DROP DATABASE {database}")
```

### Testing LangGraph Server

```python
def test_langgraph_server_integration():
    """Integration test with LangGraph dev server."""
    import httpx

    # Assumes dev server is running via make start-dev-server
    base_url = "http://localhost:8123"

    with httpx.Client(base_url=base_url) as client:
        # Test server health
        response = client.get("/health")
        assert response.status_code == 200

        # Test graph invocation
        response = client.post(
            "/runs/stream",
            json={
                "assistant_id": "agent",
                "input": {"messages": [{"role": "user", "content": "Hello"}]},
                "config": {"configurable": {"thread_id": "test-1"}}
            }
        )
        assert response.status_code == 200
```

### End-to-End Agent Testing

```python
@pytest.mark.integration
async def test_e2e_agent_workflow():
    """End-to-end test of complete agent workflow."""
    from langgraph.prebuilt import create_react_agent
    from langgraph.checkpoint.postgres import AsyncPostgresSaver
    from langgraph.store.postgres import AsyncPostgresStore

    # Setup real infrastructure
    async with AsyncPostgresSaver.from_conn_string(db_url) as checkpointer:
        await checkpointer.setup()

        async with AsyncPostgresStore.from_conn_string(db_url) as store:
            await store.setup()

            # Create agent with real dependencies
            agent = create_react_agent(
                model,
                tools,
                checkpointer=checkpointer,
                store=store
            )

            # Run multi-turn conversation
            config = {"configurable": {"thread_id": "e2e-1"}}

            # Turn 1
            result1 = await agent.ainvoke(
                {"messages": [HumanMessage("What's 2+2?")]},
                config
            )

            # Turn 2 - verify memory works
            result2 = await agent.ainvoke(
                {"messages": [HumanMessage("Double that")]},
                config
            )

            # Verify complete conversation
            assert len(result2["messages"]) > 2
```

---

## 11. Test Patterns

### 11.1 Parametrized Testing

Test the same logic with multiple inputs:

```python
@pytest.mark.parametrize("input,expected", [
    ({"value": 1}, {"result": 2}),
    ({"value": 5}, {"result": 10}),
    ({"value": 0}, {"result": 0}),
])
def test_node_with_params(input, expected):
    """Parametrized test for node logic."""
    def double_value(state):
        return {"result": state["value"] * 2}

    result = double_value(input)
    assert result == expected
```

### 11.2 Fixture Parametrization

Test with multiple fixture variants:

```python
@pytest.fixture(params=["v1", "v2"])
def agent_version(request):
    return request.param

def test_agent_versions(agent_version: str):
    """Test runs twice: once for v1, once for v2."""
    agent = create_react_agent(model, tools, version=agent_version)
    result = agent.invoke({"messages": [HumanMessage("test")]})
    assert len(result["messages"]) > 0
```

### 11.3 Test Helpers with AnyStr

Use `AnyStr` for flexible assertions on dynamic values:

```python
from tests.any_str import AnyStr

def test_with_dynamic_ids():
    """Test messages with unpredictable IDs."""
    from tests.messages import _AnyIdHumanMessage, _AnyIdAIMessage

    result = agent.invoke({"messages": [HumanMessage("test")]})

    assert result["messages"] == [
        _AnyIdHumanMessage(content="test"),
        _AnyIdAIMessage(content="response"),
    ]

    # IDs are compared flexibly using AnyStr
```

### 11.4 FloatBetween for Approximate Comparisons

```python
from tests.any_str import FloatBetween

def test_with_timing():
    """Test with approximate timing values."""
    import time

    start = time.time()
    # ... operation ...
    elapsed = time.time() - start

    # Assert elapsed is between 0.1 and 0.5 seconds
    assert elapsed == FloatBetween(0.1, 0.5)
```

### 11.5 UnsortedSequence for Order-Independent Comparison

```python
from tests.any_str import UnsortedSequence

def test_parallel_execution():
    """Test parallel node execution with order-independent assertion."""
    # Parallel nodes may complete in any order
    result = graph.invoke({"value": "test"})

    assert result["results"] == UnsortedSequence(
        "result_a",
        "result_b",
        "result_c"
    )
```

### 11.6 Interrupt Testing

```python
def test_node_interrupt(sync_checkpointer: BaseCheckpointSaver):
    """Test interrupt and resume flow."""
    from langgraph.types import interrupt, Command

    def node_with_interrupt(state):
        user_input = interrupt("Please provide input:")
        return {"result": user_input}

    builder = StateGraph(State)
    builder.add_node("ask", node_with_interrupt)
    builder.add_edge(START, "ask")
    builder.add_edge("ask", END)

    graph = builder.compile(checkpointer=sync_checkpointer)
    config = {"configurable": {"thread_id": "interrupt-1"}}

    # First invocation hits interrupt
    result1 = graph.invoke({"result": ""}, config)

    # Check interrupted state
    state = graph.get_state(config)
    assert state.next == ("ask",)
    assert len(state.tasks) == 1
    assert state.tasks[0].interrupts

    # Resume with value
    result2 = graph.invoke(Command(resume="user_response"), config)
    assert result2["result"] == "user_response"
```

### 11.7 Testing State Updates

```python
def test_multiple_state_updates():
    """Test node returning multiple state updates."""
    from langgraph.types import Command

    class State(TypedDict):
        messages: list
        count: int
        flag: bool

    def multi_update_node(state: State):
        return Command(
            update={
                "messages": state["messages"] + ["new"],
                "count": state["count"] + 1,
                "flag": True
            }
        )

    builder = StateGraph(State)
    builder.add_node("node", multi_update_node)
    builder.add_edge(START, "node")
    builder.add_edge("node", END)

    graph = builder.compile()

    result = graph.invoke({
        "messages": [],
        "count": 0,
        "flag": False
    })

    assert result == {
        "messages": ["new"],
        "count": 1,
        "flag": True
    }
```

### 11.8 Error Propagation Testing

```python
def test_error_propagation():
    """Test that errors propagate correctly through the graph."""
    class CustomError(Exception):
        pass

    def failing_node(state):
        raise CustomError("Node failed")

    builder = StateGraph(State)
    builder.add_node("fail", failing_node)
    builder.add_edge(START, "fail")

    graph = builder.compile()

    with pytest.raises(CustomError, match="Node failed"):
        graph.invoke({"value": "test"})
```

---

## 12. Writing New Tests

### Test File Organization

```
libs/langgraph/tests/
├── conftest.py                 # Shared fixtures
├── conftest_checkpointer.py    # Checkpointer fixtures
├── conftest_store.py           # Store fixtures
├── test_pregel.py              # Core graph tests
├── test_pregel_async.py        # Async graph tests
├── test_react_agent.py         # Agent tests
├── memory_assert.py            # Checkpoint immutability helpers
├── messages.py                 # Message test helpers
├── any_str.py                  # Flexible assertion helpers
└── model.py                    # Fake model implementations
```

### Test Naming Conventions

```python
# Good test names (descriptive and specific)
def test_graph_validation_raises_on_unknown_node():
    ...

def test_checkpoint_saves_state_with_metadata():
    ...

async def test_async_store_search_with_filters():
    ...

# Use parametrize for variants
@pytest.mark.parametrize("checkpointer_type", ["sqlite", "postgres"])
def test_checkpointer_persistence(checkpointer_type):
    ...
```

### Checklist for New Tests

**1. Import pytest markers for async tests:**
```python
import pytest

pytestmark = pytest.mark.anyio  # For async tests
```

**2. Use appropriate fixtures:**
```python
def test_with_fixtures(
    sync_checkpointer: BaseCheckpointSaver,
    sync_store: BaseStore,
    deterministic_uuids
):
    ...
```

**3. Clean up resources:**
```python
def test_with_cleanup():
    resource = create_resource()
    try:
        # Test logic
        assert resource.is_valid()
    finally:
        resource.cleanup()
```

**4. Use context managers for cleanup:**
```python
def test_with_context_manager():
    with create_temporary_resource() as resource:
        # Test logic - automatic cleanup on exit
        assert resource.is_valid()
```

**5. Add descriptive assertions:**
```python
# Bad: uninformative failure
assert result

# Good: clear failure message
assert result["status"] == "success", f"Expected success but got {result['status']}"
```

**6. Test both success and failure paths:**
```python
def test_node_success_and_failure():
    # Test success
    result = node({"value": 10})
    assert result["error"] is None

    # Test failure
    with pytest.raises(ValueError, match="Invalid value"):
        node({"value": -1})
```

**7. Use markers appropriately:**
```python
@pytest.mark.parametrize("version", ["v1", "v2"])
@pytest.mark.skipif(sys.version_info >= (3, 14), reason="Not supported in 3.14+")
async def test_feature(version: str):
    ...
```

### Example: Complete Test Module

```python
"""Tests for custom graph feature."""
import pytest
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.base import BaseCheckpointSaver

pytestmark = pytest.mark.anyio


class State(TypedDict):
    """Test state schema."""
    value: int
    result: str


@pytest.fixture
def sample_graph():
    """Fixture providing a configured graph."""
    def process(state: State) -> State:
        return {"result": f"processed_{state['value']}"}

    builder = StateGraph(State)
    builder.add_node("process", process)
    builder.add_edge(START, "process")
    builder.add_edge("process", END)

    return builder.compile()


def test_basic_execution(sample_graph):
    """Test basic graph execution."""
    result = sample_graph.invoke({"value": 42, "result": ""})
    assert result["result"] == "processed_42"


def test_with_checkpointer(sample_graph, sync_checkpointer: BaseCheckpointSaver):
    """Test graph with checkpointing."""
    graph = sample_graph.with_config(checkpointer=sync_checkpointer)
    config = {"configurable": {"thread_id": "test-1"}}

    result = graph.invoke({"value": 42, "result": ""}, config)

    # Verify checkpoint
    checkpoint = sync_checkpointer.get_tuple(config)
    assert checkpoint is not None
    assert checkpoint.checkpoint["channel_values"]["result"] == "processed_42"


@pytest.mark.parametrize("input_value,expected", [
    (1, "processed_1"),
    (100, "processed_100"),
    (-5, "processed_-5"),
])
def test_various_inputs(sample_graph, input_value, expected):
    """Test graph with various inputs."""
    result = sample_graph.invoke({"value": input_value, "result": ""})
    assert result["result"] == expected


async def test_async_execution():
    """Test async graph execution."""
    async def async_process(state: State) -> State:
        # Simulate async work
        import asyncio
        await asyncio.sleep(0.01)
        return {"result": f"async_{state['value']}"}

    builder = StateGraph(State)
    builder.add_node("process", async_process)
    builder.add_edge(START, "process")
    builder.add_edge("process", END)

    graph = builder.compile()

    result = await graph.ainvoke({"value": 42, "result": ""})
    assert result["result"] == "async_42"
```

### Running Your New Tests

```bash
# Run just your new test file
TEST=tests/test_myfeature.py make test

# Run specific test
TEST=tests/test_myfeature.py::test_basic_execution make test

# Run with verbose output
TEST="tests/test_myfeature.py -v" make test

# Run and show print statements
TEST="tests/test_myfeature.py -s" make test

# Run in watch mode during development
make test_watch TEST=tests/test_myfeature.py
```

### Best Practices Summary

1. **Use fixtures** - Leverage parametrized fixtures for testing multiple implementations
2. **Test async variants** - Always test both sync and async code paths
3. **Mock external dependencies** - Use `FakeToolCallingModel` and mocks for LLMs
4. **Clean up resources** - Use context managers and fixtures with cleanup
5. **Be specific** - Use descriptive test names and clear assertions
6. **Test edge cases** - Include error cases, empty inputs, and boundary conditions
7. **Use snapshots wisely** - For complex outputs like schemas, but update intentionally
8. **Parametrize similar tests** - Avoid code duplication with `@pytest.mark.parametrize`
9. **Isolate tests** - Each test should be independent and not rely on execution order
10. **Document intent** - Use docstrings to explain what the test verifies

---

## Additional Resources

- **Pytest Documentation**: https://docs.pytest.org/
- **Pytest-asyncio**: https://pytest-asyncio.readthedocs.io/
- **Syrupy (Snapshot Testing)**: https://github.com/tophat/syrupy
- **LangGraph Examples**: See `libs/langgraph/tests/` for comprehensive examples

---

This guide covers the essential testing patterns and infrastructure in LangGraph. For specific examples, refer to the test files in each library's `tests/` directory.
