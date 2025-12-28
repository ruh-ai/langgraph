# LangGraph Internal Documentation Index

This documentation suite provides comprehensive coverage of the LangGraph codebase to help developers understand and contribute to the project.

---

## Quick Navigation

| # | Document | Lines | Description |
|---|----------|-------|-------------|
| 01 | [Core Architecture](01_CORE_ARCHITECTURE.md) | 1,413 | Types, constants, errors, configuration |
| 02 | [Graph Building API](02_GRAPH_BUILDING_API.md) | 1,775 | StateGraph, nodes, edges, compilation |
| 03 | [Execution Engine](03_EXECUTION_ENGINE.md) | 1,489 | Pregel algorithm, execution loop, streaming |
| 04 | [Channels & Data Flow](04_CHANNELS_DATA_FLOW.md) | 1,128 | Channel types, reducers, state management |
| 05 | [Checkpointing](05_CHECKPOINTING.md) | 2,086 | Persistence, savers, serialization |
| 06 | [Prebuilt Components](06_PREBUILT_COMPONENTS.md) | 1,907 | ReAct agent, ToolNode, validation |
| 07 | [Python SDK](07_SDK_PYTHON.md) | 1,632 | Client library for LangGraph API |
| 08 | [CLI](08_CLI.md) | 2,999 | Command-line tools, Docker, deployment |
| 09 | [Testing Guide](09_TESTING_GUIDE.md) | 1,728 | Test patterns, fixtures, best practices |
| 10 | [Functional API](10_FUNCTIONAL_API.md) | 2,900 | @task, @entrypoint decorators |

**Total: 19,057 lines of documentation**

---

## Recommended Reading Order

### For New Contributors

1. **[Core Architecture](01_CORE_ARCHITECTURE.md)** - Start here to understand fundamental types and concepts
2. **[Graph Building API](02_GRAPH_BUILDING_API.md)** - Learn how users define graphs
3. **[Channels & Data Flow](04_CHANNELS_DATA_FLOW.md)** - Understand data management
4. **[Execution Engine](03_EXECUTION_ENGINE.md)** - Deep dive into how graphs execute
5. **[Testing Guide](09_TESTING_GUIDE.md)** - Learn to write and run tests

### For Feature Development

1. **[Prebuilt Components](06_PREBUILT_COMPONENTS.md)** - High-level agent APIs
2. **[Functional API](10_FUNCTIONAL_API.md)** - Alternative decorator-based API
3. **[Checkpointing](05_CHECKPOINTING.md)** - State persistence system

### For Deployment & Integration

1. **[CLI](08_CLI.md)** - Development and deployment tools
2. **[Python SDK](07_SDK_PYTHON.md)** - Client library for remote graphs

---

## Library Map

```
┌─────────────────────────────────────────────────────────────────────┐
│                         LangGraph Monorepo                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    CORE FRAMEWORK                            │   │
│  │  ┌─────────────┐  ┌──────────────┐  ┌─────────────────┐    │   │
│  │  │  langgraph  │  │   prebuilt   │  │   checkpoint    │    │   │
│  │  │  (libs/     │  │  (libs/      │  │  (libs/         │    │   │
│  │  │  langgraph) │  │  prebuilt)   │  │  checkpoint)    │    │   │
│  │  │             │  │              │  │                 │    │   │
│  │  │ • Graph API │  │ • ReAct      │  │ • Base Saver    │    │   │
│  │  │ • Pregel    │  │ • ToolNode   │  │ • MemorySaver   │    │   │
│  │  │ • Channels  │  │ • Validation │  │ • Store/Cache   │    │   │
│  │  │ • Func API  │  │              │  │                 │    │   │
│  │  └─────────────┘  └──────────────┘  └─────────────────┘    │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                  PERSISTENCE BACKENDS                        │   │
│  │  ┌─────────────────────┐  ┌─────────────────────────────┐   │   │
│  │  │  checkpoint-sqlite  │  │     checkpoint-postgres     │   │   │
│  │  │  (libs/checkpoint-  │  │  (libs/checkpoint-postgres) │   │   │
│  │  │  sqlite)            │  │                             │   │   │
│  │  │                     │  │  • PostgresSaver            │   │   │
│  │  │  • SqliteSaver      │  │  • AsyncPostgresSaver       │   │   │
│  │  │  • AsyncSqliteSaver │  │  • Connection pooling       │   │   │
│  │  └─────────────────────┘  └─────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    CLIENT & TOOLS                            │   │
│  │  ┌──────────────────────┐  ┌────────────────────────────┐   │   │
│  │  │       sdk-py         │  │           cli              │   │   │
│  │  │  (libs/sdk-py)       │  │  (libs/cli)                │   │   │
│  │  │                      │  │                            │   │   │
│  │  │  • LangGraphClient   │  │  • langgraph new           │   │   │
│  │  │  • Assistants API    │  │  • langgraph dev           │   │   │
│  │  │  • Threads/Runs      │  │  • langgraph build         │   │   │
│  │  │  • Streaming         │  │  • langgraph up            │   │   │
│  │  └──────────────────────┘  └────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Dependency Flow

```
                    ┌──────────────┐
                    │  checkpoint  │ (base interfaces)
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
┌──────────────────┐ ┌──────────┐ ┌─────────────────┐
│checkpoint-sqlite │ │ prebuilt │ │checkpoint-postgres│
└──────────────────┘ └────┬─────┘ └─────────────────┘
                          │
                          ▼
                    ┌───────────┐
                    │ langgraph │ (core framework)
                    └─────┬─────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
        ┌──────────┐            ┌──────────┐
        │  sdk-py  │            │   cli    │
        └──────────┘            └──────────┘
```

---

## Key Concepts Quick Reference

### Graph Building
```python
from langgraph.graph import StateGraph, START, END

graph = StateGraph(MyState)
graph.add_node("node_name", my_function)
graph.add_edge(START, "node_name")
graph.add_edge("node_name", END)
compiled = graph.compile(checkpointer=checkpointer)
```

### Functional API
```python
from langgraph.func import task, entrypoint

@task
def process(data):
    return result

@entrypoint(checkpointer=checkpointer)
def workflow(input):
    return process(input).result()
```

### Checkpointing
```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver
from langgraph.checkpoint.postgres import PostgresSaver

# Use any saver with compile()
graph.compile(checkpointer=SqliteSaver.from_conn_string(":memory:"))
```

### Prebuilt Agent
```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(model, tools, checkpointer=checkpointer)
result = agent.invoke({"messages": [("user", "Hello")]})
```

### SDK Client
```python
from langgraph_sdk import get_client

client = get_client(url="http://localhost:8123")
thread = await client.threads.create()
result = await client.runs.wait(thread["thread_id"], assistant_id, input={...})
```

---

## Development Commands

```bash
# Navigate to a library
cd libs/langgraph

# Format code
make format

# Run linter
make lint

# Run all tests
make test

# Run specific test file
TEST=tests/test_pregel.py make test

# Run specific test
TEST="tests/test_pregel.py::test_function_name" make test
```

---

## File Locations Reference

| Component | Path |
|-----------|------|
| StateGraph | `libs/langgraph/langgraph/graph/state.py` |
| Pregel Engine | `libs/langgraph/langgraph/pregel/main.py` |
| Channels | `libs/langgraph/langgraph/channels/` |
| Functional API | `libs/langgraph/langgraph/func/__init__.py` |
| Types | `libs/langgraph/langgraph/types.py` |
| Checkpoint Base | `libs/checkpoint/langgraph/checkpoint/base/` |
| SQLite Saver | `libs/checkpoint-sqlite/langgraph/checkpoint/sqlite/` |
| Postgres Saver | `libs/checkpoint-postgres/langgraph/checkpoint/postgres/` |
| Prebuilt Agent | `libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py` |
| ToolNode | `libs/prebuilt/langgraph/prebuilt/tool_node.py` |
| SDK Client | `libs/sdk-py/langgraph_sdk/client.py` |
| CLI | `libs/cli/langgraph_cli/cli.py` |
| Test Fixtures | `libs/langgraph/tests/conftest.py` |

---

## Getting Help

- **Official Docs**: `docs/` directory (MkDocs)
- **Examples**: `examples/` directory (23+ categories)
- **Tests**: Study test files for usage patterns
- **GitHub Issues**: Report bugs and request features
