# LangGraph Codebase Understanding Plan

This plan will guide you through understanding the LangGraph monorepo so you can confidently make changes.

---

## Overview

LangGraph is a framework for building stateful, multi-actor AI agents. The monorepo contains:
- **7 Python libraries** in `libs/`
- Core graph execution engine based on the Pregel algorithm
- Checkpoint persistence (SQLite, PostgreSQL)
- SDK clients and CLI tools

---

## Phase 1: Foundation (Day 1-2)

### 1.1 Understand the Repository Structure

**Goal:** Get familiar with the monorepo layout and build system.

| Task | Files to Read |
|------|---------------|
| Monorepo structure | `README.md`, `CLAUDE.md`, `Makefile` |
| Library organization | `libs/*/README.md` |
| Build configuration | `libs/*/pyproject.toml` |
| Dependency relationships | See diagram below |

**Dependency Map:**
```
checkpoint (base interfaces)
├── checkpoint-postgres
├── checkpoint-sqlite
├── prebuilt
└── langgraph

prebuilt → langgraph
sdk-py → langgraph, cli
```

### 1.2 Core Concepts & Types

**Goal:** Understand the fundamental types and constants.

| Task | Files to Read |
|------|---------------|
| Core constants (START, END, etc.) | `libs/langgraph/langgraph/constants.py` |
| Error types | `libs/langgraph/langgraph/errors.py` |
| Type definitions | `libs/langgraph/langgraph/types.py` |
| Configuration | `libs/langgraph/langgraph/config.py` |

**Key concepts to understand:**
- `START` / `END` - Special node markers
- `Command` / `Send` - Control flow primitives
- `StreamMode` - How data is streamed during execution
- `Interrupt` - Human-in-the-loop mechanism

---

## Phase 2: Graph Construction API (Day 3-4)

### 2.1 StateGraph Builder

**Goal:** Understand how users define graphs.

| Task | Files to Read |
|------|---------------|
| Main StateGraph class | `libs/langgraph/langgraph/graph/state.py` |
| Public API exports | `libs/langgraph/langgraph/graph/__init__.py` |
| Node specification | `libs/langgraph/langgraph/graph/_node.py` |
| Branching logic | `libs/langgraph/langgraph/graph/_branch.py` |
| MessagesState pattern | `libs/langgraph/langgraph/graph/message.py` |

**Key patterns:**
```python
from langgraph.graph import StateGraph, START, END

graph = StateGraph(State)
graph.add_node("node_name", node_function)
graph.add_edge(START, "node_name")
graph.add_conditional_edges("node_name", router_function)
compiled = graph.compile()
```

### 2.2 Functional API (Alternative)

**Goal:** Understand the decorator-based API.

| Task | Files to Read |
|------|---------------|
| @task and @entrypoint | `libs/langgraph/langgraph/func/__init__.py` |

**Key patterns:**
```python
from langgraph.func import task, entrypoint

@task
def my_task(input):
    return result

@entrypoint()
def my_workflow(input):
    result = my_task(input).result()
    return result
```

---

## Phase 3: Execution Engine (Day 5-7)

### 3.1 Pregel Algorithm Core

**Goal:** Understand how graphs execute.

| Task | Files to Read |
|------|---------------|
| Main Pregel executor | `libs/langgraph/langgraph/pregel/main.py` (largest file - 131KB) |
| Execution loop | `libs/langgraph/langgraph/pregel/_loop.py` |
| Algorithm implementation | `libs/langgraph/langgraph/pregel/_algo.py` |
| Task runner | `libs/langgraph/langgraph/pregel/_runner.py` |

**Key concepts:**
- **Supersteps** - Execution rounds where all active nodes run
- **Message passing** - Nodes communicate via channels
- **State management** - Shared state modified by nodes
- **Checkpointing** - State saved between supersteps

### 3.2 Channels (Data Flow)

**Goal:** Understand how data flows between nodes.

| Task | Files to Read |
|------|---------------|
| Base channel interface | `libs/langgraph/langgraph/channels/base.py` |
| LastValue channel | `libs/langgraph/langgraph/channels/last_value.py` |
| Topic channel | `libs/langgraph/langgraph/channels/topic.py` |
| Binary operator aggregation | `libs/langgraph/langgraph/channels/binop.py` |

**Channel types:**
- `LastValue` - Stores most recent value
- `Topic` - Pub/sub broadcast
- `BinaryOperatorAggregate` - Combines values with reducer function

---

## Phase 4: Checkpointing & Persistence (Day 8-9)

### 4.1 Checkpoint Interfaces

**Goal:** Understand state persistence.

| Task | Files to Read |
|------|---------------|
| Base interfaces | `libs/checkpoint/langgraph/checkpoint/base/__init__.py` |
| In-memory implementation | `libs/checkpoint/langgraph/checkpoint/memory/__init__.py` |
| Serialization | `libs/checkpoint/langgraph/checkpoint/serde/` |
| Store interfaces | `libs/checkpoint/langgraph/store/base.py` |
| Cache interfaces | `libs/checkpoint/langgraph/cache/base.py` |

### 4.2 Concrete Implementations

| Task | Files to Read |
|------|---------------|
| SQLite saver | `libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/__init__.py` |
| Async SQLite | `libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/aio.py` |
| PostgreSQL saver | `libs/checkpoint-postgres/langgraph/checkpoint/postgres/__init__.py` |
| Async PostgreSQL | `libs/checkpoint-postgres/langgraph/checkpoint/postgres/aio.py` |

---

## Phase 5: Prebuilt Components (Day 10)

### 5.1 High-Level Agent APIs

**Goal:** Understand ready-to-use components.

| Task | Files to Read |
|------|---------------|
| ReAct agent | `libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py` |
| Tool execution | `libs/prebuilt/langgraph/prebuilt/tool_node.py` |
| Tool validation | `libs/prebuilt/langgraph/prebuilt/tool_validator.py` |
| Human interrupts | `libs/prebuilt/langgraph/prebuilt/interrupt.py` |

**Key exports:**
```python
from langgraph.prebuilt import create_react_agent, ToolNode, tools_condition
```

---

## Phase 6: SDK & CLI (Day 11-12)

### 6.1 Python SDK

**Goal:** Understand the client library for remote graphs.

| Task | Files to Read |
|------|---------------|
| Main client | `libs/sdk-py/langgraph_sdk/client.py` (256KB) |
| Schema definitions | `libs/sdk-py/langgraph_sdk/schema.py` |
| Authentication | `libs/sdk-py/langgraph_sdk/auth/` |

### 6.2 CLI

**Goal:** Understand the command-line tools.

| Task | Files to Read |
|------|---------------|
| CLI commands | `libs/cli/langgraph_cli/cli.py` |
| Configuration | `libs/cli/langgraph_cli/config.py` |
| Schema definitions | `libs/cli/langgraph_cli/schemas.py` |
| Docker integration | `libs/cli/langgraph_cli/docker.py` |

---

## Phase 7: Testing Patterns (Day 13-14)

### 7.1 Test Infrastructure

**Goal:** Learn how to write and run tests.

| Task | Files to Read |
|------|---------------|
| Test fixtures | `libs/langgraph/tests/conftest.py` |
| Checkpointer fixtures | `libs/langgraph/tests/conftest_checkpointer.py` |
| Store fixtures | `libs/langgraph/tests/conftest_store.py` |

### 7.2 Test Examples

Study these test files to understand patterns:

| Test Type | File |
|-----------|------|
| Core execution | `libs/langgraph/tests/test_pregel.py` |
| Async execution | `libs/langgraph/tests/test_pregel_async.py` |
| Complex scenarios | `libs/langgraph/tests/test_large_cases.py` |
| Checkpoint migration | `libs/langgraph/tests/test_checkpoint_migration.py` |

### 7.3 Running Tests

```bash
# Run all tests for a library
cd libs/langgraph && make test

# Run specific test file
TEST=tests/test_pregel.py make test

# Run specific test
TEST="tests/test_pregel.py::test_function_name" make test
```

---

## Phase 8: Making Changes (Ongoing)

### 8.1 Development Workflow

```bash
# 1. Make your changes in the appropriate library

# 2. Format code
cd libs/<library> && make format

# 3. Run linter
make lint

# 4. Run tests
make test

# 5. For changes affecting multiple libraries, test dependents
```

### 8.2 Change Impact Reference

| If you change... | Also test... |
|------------------|--------------|
| `checkpoint` | `checkpoint-postgres`, `checkpoint-sqlite`, `prebuilt`, `langgraph` |
| `prebuilt` | `langgraph` |
| `sdk-py` | `langgraph`, `cli` |

---

## Quick Reference: Key Entry Points

```python
# Graph building
from langgraph.graph import StateGraph, START, END, MessagesState

# Execution
from langgraph.pregel import Pregel

# Checkpointing
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# Prebuilt agents
from langgraph.prebuilt import create_react_agent, ToolNode

# SDK client
from langgraph_sdk import get_client, get_sync_client

# Functional API
from langgraph.func import task, entrypoint
```

---

## Recommended Learning Order

1. **Start simple:** Build a basic graph using `StateGraph`
2. **Add persistence:** Use `InMemorySaver`, then `SqliteSaver`
3. **Study execution:** Read `pregel/main.py` to understand the engine
4. **Explore channels:** Understand data flow mechanisms
5. **Try prebuilt:** Use `create_react_agent` to see high-level patterns
6. **Dive into tests:** Study test patterns before writing your own changes

---

## Files by Complexity

### Simpler (Start Here)
- `langgraph/constants.py`
- `langgraph/errors.py`
- `langgraph/channels/last_value.py`
- `checkpoint/memory/__init__.py`

### Moderate
- `langgraph/graph/state.py`
- `langgraph/types.py`
- `prebuilt/tool_node.py`

### Complex (Study Later)
- `langgraph/pregel/main.py` (131KB - core engine)
- `langgraph/func/__init__.py` (decorator magic)
- `sdk-py/client.py` (256KB - full API client)

---

## Getting Help

- **Documentation:** `docs/` directory with MkDocs
- **Examples:** `examples/` with 23+ categories
- **Tests:** Study test files for usage patterns
