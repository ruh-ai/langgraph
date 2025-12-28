# LangGraph Python SDK API Reference

## Overview

The LangGraph Python SDK provides programmatic access to LangGraph deployments. It supports both asynchronous and synchronous operations, allowing you to manage assistants, threads, runs, cron jobs, and persistent storage.

The SDK consists of:
- **Client initialization functions**: `get_client()` and `get_sync_client()`
- **Main client classes**: `LangGraphClient` (async) and `SyncLangGraphClient` (sync)
- **Sub-clients** for different resources: Assistants, Threads, Runs, Crons, Store
- **Authentication system**: `Auth` class for custom authentication/authorization
- **Schema types**: Data models for API interactions
- **Exception classes**: Typed errors for better error handling

---

## Client Initialization

### get_client

**Signature:**
```python
def get_client(
    *,
    url: str | None = None,
    api_key: str | None = NOT_PROVIDED,
    headers: Mapping[str, str] | None = None,
    timeout: TimeoutTypes | None = None,
) -> LangGraphClient:
```

**Description:**
Create and configure an asynchronous LangGraphClient. The client provides programmatic access to LangGraph deployments, supporting both remote servers and local in-process connections.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| url | str \| None | No | None | Base URL of the LangGraph API. If None, attempts in-process connection via ASGI transport |
| api_key | str \| None | No | NOT_PROVIDED | API key for authentication. Can be a string (use exact key), None (skip env loading), or NOT_PROVIDED (auto-load from LANGGRAPH_API_KEY, LANGSMITH_API_KEY, or LANGCHAIN_API_KEY) |
| headers | Mapping[str, str] \| None | No | None | Additional HTTP headers to include in requests |
| timeout | TimeoutTypes \| None | No | None | HTTP timeout configuration. May be httpx.Timeout instance, float (seconds), or tuple (connect, read, write, pool) |

**Returns:**
| Type | Description |
|------|-------------|
| LangGraphClient | Asynchronous client exposing sub-clients for assistants, threads, runs, crons, and store |

**Example:**
```python
from langgraph_sdk import get_client

# Connect to remote server
client = get_client(url="http://localhost:8123")
assistants = await client.assistants.get(assistant_id="some_uuid")

# In-process connection (when running inside LangGraph server)
client = get_client(url=None)
result = await client.runs.wait(
    thread_id=None,
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hello"}]},
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:177-289`

---

### get_sync_client

**Signature:**
```python
def get_sync_client(
    *,
    url: str | None = None,
    api_key: str | None = NOT_PROVIDED,
    headers: Mapping[str, str] | None = None,
    timeout: TimeoutTypes | None = None,
) -> SyncLangGraphClient:
```

**Description:**
Create and configure a synchronous LangGraphClient for blocking I/O operations.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| url | str \| None | No | None | Base URL of the LangGraph API. Defaults to "http://localhost:8123" if None |
| api_key | str \| None | No | NOT_PROVIDED | API key for authentication. Same behavior as get_client() |
| headers | Mapping[str, str] \| None | No | None | Additional HTTP headers to include in requests |
| timeout | TimeoutTypes \| None | No | None | HTTP timeout configuration |

**Returns:**
| Type | Description |
|------|-------------|
| SyncLangGraphClient | Synchronous client exposing sub-clients for assistants, threads, runs, crons, and store |

**Example:**
```python
from langgraph_sdk import get_sync_client

client = get_sync_client(url="http://localhost:8123")
assistant = client.assistants.get(assistant_id="some_uuid")
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3568-3635`

---

## Main Client Classes

### LangGraphClient

**Description:**
Top-level asynchronous client for LangGraph API. Provides access to all resource sub-clients.

**Attributes:**
- `assistants` (AssistantsClient): Manages versioned configuration for graphs
- `threads` (ThreadsClient): Handles multi-turn interactions and conversational threads
- `runs` (RunsClient): Controls individual invocations of graphs
- `crons` (CronClient): Manages scheduled operations
- `store` (StoreClient): Interfaces with persistent shared data storage

**Methods:**

#### \_\_aenter\_\_
```python
async def __aenter__(self) -> LangGraphClient:
```
Enter the async context manager.

#### \_\_aexit\_\_
```python
async def __aexit__(
    self,
    exc_type: type[BaseException] | None,
    exc_val: BaseException | None,
    exc_tb: TracebackType | None,
) -> None:
```
Exit the async context manager.

#### aclose
```python
async def aclose(self) -> None:
```
Close the underlying HTTP client.

**Example:**
```python
async with get_client(url="http://localhost:8123") as client:
    assistant = await client.assistants.get("asst_123")
    # client automatically closed after context
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:291-328`

---

### SyncLangGraphClient

**Description:**
Top-level synchronous client for LangGraph API. Provides blocking access to all resource sub-clients.

**Attributes:**
- `assistants` (SyncAssistantsClient): Manages versioned configuration for graphs
- `threads` (SyncThreadsClient): Handles multi-turn interactions and conversational threads
- `runs` (SyncRunsClient): Controls individual invocations of graphs
- `crons` (SyncCronClient): Manages scheduled operations
- `store` (SyncStoreClient): Interfaces with persistent shared data storage

**Methods:**

#### \_\_enter\_\_
```python
def __enter__(self) -> SyncLangGraphClient:
```
Enter the sync context manager.

#### \_\_exit\_\_
```python
def __exit__(
    self,
    exc_type: type[BaseException] | None,
    exc_val: BaseException | None,
    exc_tb: TracebackType | None,
) -> None:
```
Exit the sync context manager.

#### close
```python
def close(self) -> None:
```
Close the underlying HTTP client.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3637-3676`

---

## AssistantsClient

Client for managing assistants in LangGraph. Assistants are versioned configurations of your graph.

### get

**Signature:**
```python
async def get(
    self,
    assistant_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Assistant:
```

**Description:**
Get an assistant by ID.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str | Yes | - | The ID of the assistant to get |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Assistant | Assistant object with all configuration details |

**Example:**
```python
assistant = await client.assistants.get(assistant_id="my_assistant_id")
print(assistant)
# {
#     'assistant_id': 'my_assistant_id',
#     'graph_id': 'agent',
#     'created_at': '2024-06-25T17:10:33.109781+00:00',
#     'updated_at': '2024-06-25T17:10:33.109781+00:00',
#     'config': {},
#     'metadata': {'created_by': 'system'},
#     'version': 1,
#     'name': 'my_assistant'
# }
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:631-674`

---

### create

**Signature:**
```python
async def create(
    self,
    graph_id: str | None,
    config: Config | None = None,
    *,
    context: Context | None = None,
    metadata: Json = None,
    assistant_id: str | None = None,
    if_exists: OnConflictBehavior | None = None,
    name: str | None = None,
    headers: Mapping[str, str] | None = None,
    description: str | None = None,
    params: QueryParamTypes | None = None,
) -> Assistant:
```

**Description:**
Create a new assistant. Useful when graph is configurable and you want to create different assistants based on different configurations.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| graph_id | str \| None | Yes | - | The ID of the graph the assistant should use (from langgraph.json) |
| config | Config \| None | No | None | Configuration to use for the graph |
| context | Context \| None | No | None | Static context to add to the assistant (added in v0.6.0) |
| metadata | Json | No | None | Metadata to add to assistant |
| assistant_id | str \| None | No | None | Assistant ID to use (defaults to random UUID) |
| if_exists | OnConflictBehavior \| None | No | None | How to handle duplicates: 'raise' or 'do_nothing' |
| name | str \| None | No | None | The name of the assistant (defaults to 'Untitled') |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| description | str \| None | No | None | Optional description (requires langgraph-api >= 0.0.45) |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Assistant | The created assistant |

**Example:**
```python
assistant = await client.assistants.create(
    graph_id="agent",
    context={"model_name": "openai"},
    metadata={"number": 1},
    assistant_id="my-assistant-id",
    if_exists="do_nothing",
    name="my_name"
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:893-962`

---

### update

**Signature:**
```python
async def update(
    self,
    assistant_id: str,
    *,
    graph_id: str | None = None,
    config: Config | None = None,
    context: Context | None = None,
    metadata: Json = None,
    name: str | None = None,
    headers: Mapping[str, str] | None = None,
    description: str | None = None,
    params: QueryParamTypes | None = None,
) -> Assistant:
```

**Description:**
Update an assistant. Use this to point to a different graph, update configuration, or change metadata.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str | Yes | - | Assistant to update |
| graph_id | str \| None | No | None | The ID of the graph (None keeps current graph) |
| config | Config \| None | No | None | Configuration to use for the graph |
| context | Context \| None | No | None | Static context to add to the assistant |
| metadata | Json | No | None | Metadata to merge with existing metadata |
| name | str \| None | No | None | The new name for the assistant |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| description | str \| None | No | None | Optional description |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Assistant | The updated assistant |

**Example:**
```python
assistant = await client.assistants.update(
    assistant_id='e280dad7-8618-443f-87f1-8e41841c180f',
    graph_id="other-graph",
    context={"model_name": "anthropic"},
    metadata={"number": 2}
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:964-1029`

---

### delete

**Signature:**
```python
async def delete(
    self,
    assistant_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> None:
```

**Description:**
Delete an assistant.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str | Yes | - | The assistant ID to delete |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| None | No return value |

**Example:**
```python
await client.assistants.delete(assistant_id="my_assistant_id")
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1031-1060`

---

### search

**Signature:**
```python
async def search(
    self,
    *,
    metadata: Json = None,
    graph_id: str | None = None,
    name: str | None = None,
    limit: int = 10,
    offset: int = 0,
    sort_by: AssistantSortBy | None = None,
    sort_order: SortOrder | None = None,
    select: list[AssistantSelectField] | None = None,
    response_format: Literal["array", "object"] = "array",
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> AssistantsSearchResponse | list[Assistant]:
```

**Description:**
Search for assistants with filtering, sorting, and pagination.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| metadata | Json | No | None | Metadata to filter by (exact match for each KV pair) |
| graph_id | str \| None | No | None | The ID of the graph to filter by |
| name | str \| None | No | None | Name substring to filter by (case insensitive) |
| limit | int | No | 10 | Maximum number of results to return |
| offset | int | No | 0 | Number of results to skip |
| sort_by | AssistantSortBy \| None | No | None | Field to sort by: "assistant_id", "graph_id", "name", "created_at", "updated_at" |
| sort_order | SortOrder \| None | No | None | Sort order: "asc" or "desc" |
| select | list[AssistantSelectField] \| None | No | None | Specific assistant fields to include in response |
| response_format | Literal["array", "object"] | No | "array" | Return format: "array" for list, "object" for paginated response with metadata |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| list[Assistant] \| AssistantsSearchResponse | List of assistants (array format) or paginated response with next cursor (object format) |

**Example:**
```python
response = await client.assistants.search(
    metadata={"name": "my_name"},
    graph_id="my_graph_id",
    limit=5,
    offset=5,
    response_format="object"
)
next_cursor = response["next"]
assistants = response["assistants"]
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1096-1189`

---

### count

**Signature:**
```python
async def count(
    self,
    *,
    metadata: Json = None,
    graph_id: str | None = None,
    name: str | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> int:
```

**Description:**
Count assistants matching filters.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| metadata | Json | No | None | Metadata to filter by (exact match) |
| graph_id | str \| None | No | None | Optional graph id to filter by |
| name | str \| None | No | None | Name substring to filter by (case insensitive) |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| int | Number of assistants matching criteria |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1191-1222`

---

### get_graph

**Signature:**
```python
async def get_graph(
    self,
    assistant_id: str,
    *,
    xray: int | bool = False,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> dict[str, list[dict[str, Any]]]:
```

**Description:**
Get the graph structure of an assistant.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str | Yes | - | The ID of the assistant to get the graph of |
| xray | int \| bool | No | False | Include subgraph representations. If int, only subgraphs with depth <= value are included |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| dict[str, list[dict[str, Any]]] | Graph information with nodes and edges in JSON format |

**Example:**
```python
graph_info = await client.assistants.get_graph(assistant_id="my_assistant_id")
# {
#     'nodes': [
#         {'id': '__start__', 'type': 'schema', 'data': '__start__'},
#         {'id': '__end__', 'type': 'schema', 'data': '__end__'},
#         {'id': 'agent', 'type': 'runnable', 'data': {...}}
#     ],
#     'edges': [
#         {'source': '__start__', 'target': 'agent'},
#         {'source': 'agent', 'target': '__end__'}
#     ]
# }
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:676-732`

---

### get_schemas

**Signature:**
```python
async def get_schemas(
    self,
    assistant_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> GraphSchema:
```

**Description:**
Get the schemas (input, output, state, config, context) of an assistant.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str | Yes | - | The ID of the assistant to get schemas for |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| GraphSchema | The graph schema for the assistant including input_schema, output_schema, state_schema, config_schema, context_schema |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:734-853`

---

### get_subgraphs

**Signature:**
```python
async def get_subgraphs(
    self,
    assistant_id: str,
    namespace: str | None = None,
    recurse: bool = False,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Subgraphs:
```

**Description:**
Get the subgraph schemas of an assistant.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str | Yes | - | The ID of the assistant |
| namespace | str \| None | No | None | Optional namespace to filter by |
| recurse | bool | No | False | Whether to recursively get subgraphs |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Subgraphs | Dictionary of subgraph schemas |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:855-891`

---

### get_versions

**Signature:**
```python
async def get_versions(
    self,
    assistant_id: str,
    metadata: Json = None,
    limit: int = 10,
    offset: int = 0,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> list[AssistantVersion]:
```

**Description:**
List all versions of an assistant.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str | Yes | - | The assistant ID to get versions for |
| metadata | Json | No | None | Metadata to filter versions by (exact match) |
| limit | int | No | 10 | Maximum number of versions to return |
| offset | int | No | 0 | Number of versions to skip |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| list[AssistantVersion] | List of assistant versions |

**Example:**
```python
versions = await client.assistants.get_versions(assistant_id="my_assistant_id")
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1224-1268`

---

### set_latest

**Signature:**
```python
async def set_latest(
    self,
    assistant_id: str,
    version: int,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Assistant:
```

**Description:**
Change the version of an assistant.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str | Yes | - | The assistant ID |
| version | int | Yes | - | The version to change to |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Assistant | Updated assistant object |

**Example:**
```python
new_version_assistant = await client.assistants.set_latest(
    assistant_id="my_assistant_id",
    version=3
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1270-1308`

---

## ThreadsClient

Client for managing threads in LangGraph. A thread maintains the state of a graph across multiple interactions/invocations (runs).

### get

**Signature:**
```python
async def get(
    self,
    thread_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Thread:
```

**Description:**
Get a thread by ID.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The ID of the thread to get |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Thread | Thread object with state and metadata |

**Example:**
```python
thread = await client.threads.get(thread_id="my_thread_id")
# {
#     'thread_id': 'my_thread_id',
#     'created_at': '2024-07-18T18:35:15.540834+00:00',
#     'updated_at': '2024-07-18T18:35:15.540834+00:00',
#     'metadata': {'graph_id': 'agent'}
# }
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1329-1371`

---

### create

**Signature:**
```python
async def create(
    self,
    *,
    metadata: Json = None,
    thread_id: str | None = None,
    if_exists: OnConflictBehavior | None = None,
    supersteps: Sequence[dict[str, Sequence[dict[str, Any]]]] | None = None,
    graph_id: str | None = None,
    ttl: int | Mapping[str, Any] | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Thread:
```

**Description:**
Create a new thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| metadata | Json | No | None | Metadata to add to thread |
| thread_id | str \| None | No | None | ID of thread (random UUID if None) |
| if_exists | OnConflictBehavior \| None | No | None | How to handle duplicates: 'raise' or 'do_nothing' |
| supersteps | Sequence[dict] \| None | No | None | Apply supersteps when creating (for copying threads) |
| graph_id | str \| None | No | None | Optional graph ID to associate with thread |
| ttl | int \| Mapping[str, Any] \| None | No | None | Time-to-live in minutes or mapping with 'ttl' and 'strategy' keys |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Thread | The created thread |

**Example:**
```python
thread = await client.threads.create(
    metadata={"number": 1},
    thread_id="my-thread-id",
    if_exists="raise"
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1373-1448`

---

### update

**Signature:**
```python
async def update(
    self,
    thread_id: str,
    *,
    metadata: Mapping[str, Any],
    ttl: int | Mapping[str, Any] | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Thread:
```

**Description:**
Update a thread's metadata.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | ID of thread to update |
| metadata | Mapping[str, Any] | Yes | - | Metadata to merge with existing thread metadata |
| ttl | int \| Mapping[str, Any] \| None | No | None | Time-to-live in minutes or mapping with 'ttl' and 'strategy' keys |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Thread | The updated thread |

**Example:**
```python
thread = await client.threads.update(
    thread_id="my-thread-id",
    metadata={"number": 1},
    ttl=43_200,
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1450-1495`

---

### delete

**Signature:**
```python
async def delete(
    self,
    thread_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> None:
```

**Description:**
Delete a thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The ID of the thread to delete |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| None | No return value |

**Example:**
```python
await client.threads.delete(thread_id="my_thread_id")
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1497-1524`

---

### search

**Signature:**
```python
async def search(
    self,
    *,
    metadata: Json = None,
    values: Json = None,
    ids: Sequence[str] | None = None,
    status: ThreadStatus | None = None,
    limit: int = 10,
    offset: int = 0,
    sort_by: ThreadSortBy | None = None,
    sort_order: SortOrder | None = None,
    select: list[ThreadSelectField] | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> list[Thread]:
```

**Description:**
Search for threads with filtering and pagination.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| metadata | Json | No | None | Thread metadata to filter on |
| values | Json | No | None | State values to filter on |
| ids | Sequence[str] \| None | No | None | List of thread IDs to filter by |
| status | ThreadStatus \| None | No | None | Thread status: 'idle', 'busy', 'interrupted', or 'error' |
| limit | int | No | 10 | Maximum number of threads to return |
| offset | int | No | 0 | Offset in threads table |
| sort_by | ThreadSortBy \| None | No | None | Field to sort by: "thread_id", "status", "created_at", "updated_at" |
| sort_order | SortOrder \| None | No | None | Sort order: "asc" or "desc" |
| select | list[ThreadSelectField] \| None | No | None | Specific thread fields to include |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| list[Thread] | List of threads matching search parameters |

**Example:**
```python
threads = await client.threads.search(
    metadata={"number": 1},
    status="interrupted",
    limit=15,
    offset=5
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1526-1595`

---

### count

**Signature:**
```python
async def count(
    self,
    *,
    metadata: Json = None,
    values: Json = None,
    status: ThreadStatus | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> int:
```

**Description:**
Count threads matching filters.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| metadata | Json | No | None | Thread metadata to filter on |
| values | Json | No | None | State values to filter on |
| status | ThreadStatus \| None | No | None | Thread status to filter on |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| int | Number of threads matching criteria |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1597-1627`

---

### copy

**Signature:**
```python
async def copy(
    self,
    thread_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> None:
```

**Description:**
Copy a thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The ID of the thread to copy |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| None | No return value |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1629-1658`

---

### get_state

**Signature:**
```python
async def get_state(
    self,
    thread_id: str,
    checkpoint: Checkpoint | None = None,
    checkpoint_id: str | None = None,  # deprecated
    *,
    subgraphs: bool = False,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> ThreadState:
```

**Description:**
Get the state of a thread at a specific checkpoint.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The ID of the thread |
| checkpoint | Checkpoint \| None | No | None | The checkpoint to get state for |
| checkpoint_id | str \| None | No | None | (Deprecated) The checkpoint ID |
| subgraphs | bool | No | False | Include subgraphs states |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| ThreadState | The thread state with values, next steps, checkpoint info, and metadata |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1660-1794`

---

### update_state

**Signature:**
```python
async def update_state(
    self,
    thread_id: str,
    values: dict[str, Any] | Sequence[dict] | None,
    *,
    as_node: str | None = None,
    checkpoint: Checkpoint | None = None,
    checkpoint_id: str | None = None,  # deprecated
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> ThreadUpdateStateResponse:
```

**Description:**
Update the state of a thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The ID of the thread to update |
| values | dict[str, Any] \| Sequence[dict] \| None | Yes | - | The values to update the state with |
| as_node | str \| None | No | None | Update state as if this node had just executed |
| checkpoint | Checkpoint \| None | No | None | The checkpoint to update state of |
| checkpoint_id | str \| None | No | None | (Deprecated) The checkpoint ID |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| ThreadUpdateStateResponse | Response with checkpoint of latest state |

**Example:**
```python
response = await client.threads.update_state(
    thread_id="my_thread_id",
    values={"messages": [{"role": "user", "content": "hello!"}]},
    as_node="my_node",
)
# {'checkpoint': {'thread_id': '...', 'checkpoint_id': '...', ...}}
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1796-1857`

---

### get_history

**Signature:**
```python
async def get_history(
    self,
    thread_id: str,
    *,
    limit: int = 10,
    before: str | Checkpoint | None = None,
    metadata: Mapping[str, Any] | None = None,
    checkpoint: Checkpoint | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> list[ThreadState]:
```

**Description:**
Get the state history of a thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The ID of the thread |
| limit | int | No | 10 | Maximum number of states to return |
| before | str \| Checkpoint \| None | No | None | Return states before this checkpoint |
| metadata | Mapping[str, Any] \| None | No | None | Filter states by metadata key-value pairs |
| checkpoint | Checkpoint \| None | No | None | Return states for this subgraph (defaults to root) |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| list[ThreadState] | The state history of the thread |

**Example:**
```python
history = await client.threads.get_history(
    thread_id="my_thread_id",
    limit=5,
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1859-1909`

---

### join_stream

**Signature:**
```python
def join_stream(
    self,
    thread_id: str,
    *,
    last_event_id: str | None = None,
    stream_mode: ThreadStreamMode | Sequence[ThreadStreamMode] = "run_modes",
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> AsyncIterator[StreamPart]:
```

**Description:**
Get a stream of events for a thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The ID of the thread |
| last_event_id | str \| None | No | None | The ID of the last event |
| stream_mode | ThreadStreamMode \| Sequence[ThreadStreamMode] | No | "run_modes" | Stream mode: "run_modes", "lifecycle", or "state_update" |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[StreamPart] | An async iterator of stream parts |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:1911-1956`

---

## RunsClient

Client for managing runs in LangGraph. A run is a single assistant invocation with optional input, config, context, and metadata.

### stream

**Signature:**
```python
def stream(
    self,
    thread_id: str | None,
    assistant_id: str,
    *,
    input: Input | None = None,
    command: Command | None = None,
    stream_mode: StreamMode | Sequence[StreamMode] = "values",
    stream_subgraphs: bool = False,
    stream_resumable: bool = False,
    metadata: Mapping[str, Any] | None = None,
    config: Config | None = None,
    context: Context | None = None,
    checkpoint: Checkpoint | None = None,
    checkpoint_id: str | None = None,
    checkpoint_during: bool | None = None,  # deprecated
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    feedback_keys: Sequence[str] | None = None,
    on_disconnect: DisconnectMode | None = None,
    on_completion: OnCompletionBehavior | None = None,
    webhook: str | None = None,
    multitask_strategy: MultitaskStrategy | None = None,
    if_not_exists: IfNotExists | None = None,
    after_seconds: int | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
    on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    durability: Durability | None = None,
) -> AsyncIterator[StreamPart]:
```

**Description:**
Create a run and stream the results in real-time.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str \| None | Yes | - | Thread ID for stateful run, or None for stateless |
| assistant_id | str | Yes | - | The assistant ID or graph name |
| input | Input \| None | No | None | The input to the graph |
| command | Command \| None | No | None | A command to execute (cannot combine with input) |
| stream_mode | StreamMode \| Sequence[StreamMode] | No | "values" | Stream mode(s): "values", "messages", "updates", "events", "checkpoints", "tasks", "debug", "custom", "messages-tuple" |
| stream_subgraphs | bool | No | False | Whether to stream output from subgraphs |
| stream_resumable | bool | No | False | Whether stream can be resumed after disconnection |
| metadata | Mapping[str, Any] \| None | No | None | Metadata to assign to run |
| config | Config \| None | No | None | Configuration for assistant |
| context | Context \| None | No | None | Static context to add (v0.6.0+) |
| checkpoint | Checkpoint \| None | No | None | Checkpoint to resume from |
| checkpoint_id | str \| None | No | None | (Deprecated) Checkpoint ID |
| checkpoint_during | bool \| None | No | None | (Deprecated) Use durability instead |
| interrupt_before | All \| Sequence[str] \| None | No | None | Nodes to interrupt before execution |
| interrupt_after | All \| Sequence[str] \| None | No | None | Nodes to interrupt after execution |
| feedback_keys | Sequence[str] \| None | No | None | Feedback keys to assign |
| on_disconnect | DisconnectMode \| None | No | None | Disconnect behavior: "cancel" or "continue" |
| on_completion | OnCompletionBehavior \| None | No | None | Stateless run cleanup: "delete" or "keep" |
| webhook | str \| None | No | None | Webhook URL to call after completion |
| multitask_strategy | MultitaskStrategy \| None | No | None | Strategy: "reject", "interrupt", "rollback", "enqueue" |
| if_not_exists | IfNotExists \| None | No | None | Missing thread handling: "reject" or "create" |
| after_seconds | int \| None | No | None | Seconds to wait before starting (scheduling) |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |
| on_run_created | Callable \| None | No | None | Callback when run is created |
| durability | Durability \| None | No | None | Durability mode: "sync", "async", or "exit" |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[StreamPart] | Asynchronous iterator of stream results |

**Example:**
```python
async for chunk in client.runs.stream(
    thread_id=None,
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "how are you?"}]},
    stream_mode=["values", "debug"],
    metadata={"name": "my_run"},
    context={"model_name": "anthropic"},
):
    print(chunk)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2033-2189`

---

### create

**Signature:**
```python
async def create(
    self,
    thread_id: str | None,
    assistant_id: str,
    *,
    input: Input | None = None,
    command: Command | None = None,
    stream_mode: StreamMode | Sequence[StreamMode] = "values",
    stream_subgraphs: bool = False,
    stream_resumable: bool = False,
    metadata: Mapping[str, Any] | None = None,
    config: Config | None = None,
    context: Context | None = None,
    checkpoint: Checkpoint | None = None,
    checkpoint_id: str | None = None,
    checkpoint_during: bool | None = None,  # deprecated
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    webhook: str | None = None,
    multitask_strategy: MultitaskStrategy | None = None,
    if_not_exists: IfNotExists | None = None,
    on_completion: OnCompletionBehavior | None = None,
    after_seconds: int | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
    on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    durability: Durability | None = None,
) -> Run:
```

**Description:**
Create a background run (non-streaming).

**Parameters:**
Similar to `stream()` method, but returns a Run object instead of streaming.

**Returns:**
| Type | Description |
|------|-------------|
| Run | The created background run object |

**Example:**
```python
background_run = await client.runs.create(
    thread_id="my_thread_id",
    assistant_id="my_assistant_id",
    input={"messages": [{"role": "user", "content": "hello!"}]},
    metadata={"name": "my_run"},
    context={"model_name": "openai"},
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2245-2427`

---

### create_batch

**Signature:**
```python
async def create_batch(
    self,
    payloads: list[RunCreate],
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> list[Run]:
```

**Description:**
Create a batch of stateless background runs.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| payloads | list[RunCreate] | Yes | - | List of run creation payloads |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| list[Run] | List of created runs |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2429-2444`

---

### wait

**Signature:**
```python
async def wait(
    self,
    thread_id: str | None,
    assistant_id: str,
    *,
    input: Input | None = None,
    command: Command | None = None,
    metadata: Mapping[str, Any] | None = None,
    config: Config | None = None,
    context: Context | None = None,
    checkpoint: Checkpoint | None = None,
    checkpoint_id: str | None = None,
    checkpoint_during: bool | None = None,  # deprecated
    interrupt_before: All | Sequence[str] | None = None,
    interrupt_after: All | Sequence[str] | None = None,
    webhook: str | None = None,
    on_disconnect: DisconnectMode | None = None,
    on_completion: OnCompletionBehavior | None = None,
    multitask_strategy: MultitaskStrategy | None = None,
    if_not_exists: IfNotExists | None = None,
    after_seconds: int | None = None,
    raise_error: bool = True,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
    on_run_created: Callable[[RunCreateMetadata], None] | None = None,
    durability: Durability | None = None,
) -> list[dict] | dict[str, Any]:
```

**Description:**
Create a run, wait until it finishes, and return the final state.

**Parameters:**
Similar to `create()` with additional `raise_error` parameter.

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| raise_error | bool | No | True | Whether to raise exception on run error |

**Returns:**
| Type | Description |
|------|-------------|
| list[dict] \| dict[str, Any] | The output/final state of the run |

**Example:**
```python
final_state = await client.runs.wait(
    thread_id=None,
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "how are you?"}]},
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2498-2666`

---

### list

**Signature:**
```python
async def list(
    self,
    thread_id: str,
    *,
    limit: int = 10,
    offset: int = 0,
    status: RunStatus | None = None,
    select: list[RunSelectField] | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> list[Run]:
```

**Description:**
List runs for a thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The thread ID to list runs for |
| limit | int | No | 10 | Maximum number of results to return |
| offset | int | No | 0 | Number of results to skip |
| status | RunStatus \| None | No | None | Filter by status: "pending", "running", "error", "success", "timeout", "interrupted" |
| select | list[RunSelectField] \| None | No | None | Specific run fields to include |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| list[Run] | List of runs for the thread |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2668-2716`

---

### get

**Signature:**
```python
async def get(
    self,
    thread_id: str,
    run_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Run:
```

**Description:**
Get a run by ID.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The thread ID |
| run_id | str | Yes | - | The run ID |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Run | Run object |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2718-2751`

---

### cancel

**Signature:**
```python
async def cancel(
    self,
    thread_id: str,
    run_id: str,
    *,
    wait: bool = False,
    action: CancelAction = "interrupt",
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> None:
```

**Description:**
Cancel a run.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The thread ID |
| run_id | str | Yes | - | The run ID to cancel |
| wait | bool | No | False | Whether to wait until run has completed |
| action | CancelAction | No | "interrupt" | Cancel action: "interrupt" or "rollback" |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| None | No return value |

**Example:**
```python
await client.runs.cancel(
    thread_id="thread_id_to_cancel",
    run_id="run_id_to_cancel",
    wait=True,
    action="interrupt"
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2753-2809`

---

### join

**Signature:**
```python
async def join(
    self,
    thread_id: str,
    run_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> dict:
```

**Description:**
Block until a run is done and return the final state of the thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The thread ID to join |
| run_id | str | Yes | - | The run ID to join |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| dict | Final thread state |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2811-2846`

---

### join_stream

**Signature:**
```python
def join_stream(
    self,
    thread_id: str,
    run_id: str,
    *,
    cancel_on_disconnect: bool = False,
    stream_mode: StreamMode | Sequence[StreamMode] | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
    last_event_id: str | None = None,
) -> AsyncIterator[StreamPart]:
```

**Description:**
Stream output from a run in real-time until done. Output is not buffered, so any output produced before this call will not be received.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The thread ID |
| run_id | str | Yes | - | The run ID |
| cancel_on_disconnect | bool | No | False | Whether to cancel run when stream disconnects |
| stream_mode | StreamMode \| Sequence[StreamMode] \| None | No | None | Stream mode(s) to use (subset of modes used when creating run) |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |
| last_event_id | str \| None | No | None | Last event ID to use for stream |

**Returns:**
| Type | Description |
|------|-------------|
| AsyncIterator[StreamPart] | Stream of parts |

**Example:**
```python
async for part in client.runs.join_stream(
    thread_id="thread_id_to_join",
    run_id="run_id_to_join",
    stream_mode=["values", "debug"]
):
    print(part)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2848-2905`

---

### delete

**Signature:**
```python
async def delete(
    self,
    thread_id: str,
    run_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> None:
```

**Description:**
Delete a run.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | The thread ID |
| run_id | str | Yes | - | The run ID to delete |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes | None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| None | No return value |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2907-2939`

---

## CronClient

Client for managing recurrent runs (cron jobs) in LangGraph. Allows scheduling recurring runs to occur automatically.

### create_for_thread

**Signature:**
```python
async def create_for_thread(
    self,
    thread_id: str,
    assistant_id: str,
    *,
    schedule: str,
    input: Input | None = None,
    metadata: Mapping[str, Any] | None = None,
    config: Config | None = None,
    context: Context | None = None,
    checkpoint_during: bool | None = None,
    interrupt_before: All | list[str] | None = None,
    interrupt_after: All | list[str] | None = None,
    webhook: str | None = None,
    multitask_strategy: str | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Run:
```

**Description:**
Create a cron job for a specific thread.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| thread_id | str | Yes | - | Thread ID to run cron job on |
| assistant_id | str | Yes | - | Assistant ID or graph name |
| schedule | str | Yes | - | Cron schedule expression |
| input | Input \| None | No | None | Input to the graph |
| metadata | Mapping[str, Any] \| None | No | None | Metadata for cron job runs |
| config | Config \| None | No | None | Configuration for assistant |
| context | Context \| None | No | None | Static context (v0.6.0+) |
| checkpoint_during | bool \| None | No | None | Whether to checkpoint during run |
| interrupt_before | All \| list[str] \| None | No | None | Nodes to interrupt before |
| interrupt_after | All \| list[str] \| None | No | None | Nodes to interrupt after |
| webhook | str \| None | No | None | Webhook URL |
| multitask_strategy | str \| None | No | None | Multitask strategy |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Run | The cron run |

**Example:**
```python
cron_run = await client.crons.create_for_thread(
    thread_id="my-thread-id",
    assistant_id="agent",
    schedule="27 15 * * *",
    input={"messages": [{"role": "user", "content": "hello!"}]},
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:2970-3052`

---

### create

**Signature:**
```python
async def create(
    self,
    assistant_id: str,
    *,
    schedule: str,
    input: Input | None = None,
    metadata: Mapping[str, Any] | None = None,
    config: Config | None = None,
    context: Context | None = None,
    checkpoint_during: bool | None = None,
    interrupt_before: All | list[str] | None = None,
    interrupt_after: All | list[str] | None = None,
    webhook: str | None = None,
    multitask_strategy: str | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Run:
```

**Description:**
Create a stateless cron run (without a specific thread).

**Parameters:**
Similar to `create_for_thread()` but without thread_id.

**Returns:**
| Type | Description |
|------|-------------|
| Run | The cron run |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3054-3129`

---

### delete

**Signature:**
```python
async def delete(
    self,
    cron_id: str,
    *,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> None:
```

**Description:**
Delete a cron job.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| cron_id | str | Yes | - | The cron ID to delete |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| None | No return value |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3131-3158`

---

### search

**Signature:**
```python
async def search(
    self,
    *,
    assistant_id: str | None = None,
    thread_id: str | None = None,
    limit: int = 10,
    offset: int = 0,
    sort_by: CronSortBy | None = None,
    sort_order: SortOrder | None = None,
    select: list[CronSelectField] | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> list[Cron]:
```

**Description:**
Search for cron jobs.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str \| None | No | None | Assistant ID or graph name to filter |
| thread_id | str \| None | No | None | Thread ID to filter |
| limit | int | No | 10 | Maximum number of results |
| offset | int | No | 0 | Number of results to skip |
| sort_by | CronSortBy \| None | No | None | Field to sort by: "cron_id", "assistant_id", "thread_id", "created_at", "updated_at", "next_run_date" |
| sort_order | SortOrder \| None | No | None | Sort order: "asc" or "desc" |
| select | list[CronSelectField] \| None | No | None | Specific cron fields to include |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| list[Cron] | List of cron jobs matching search criteria |

**Example:**
```python
cron_jobs = await client.crons.search(
    assistant_id="my_assistant_id",
    thread_id="my_thread_id",
    limit=5,
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3160-3239`

---

### count

**Signature:**
```python
async def count(
    self,
    *,
    assistant_id: str | None = None,
    thread_id: str | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> int:
```

**Description:**
Count cron jobs matching filters.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| assistant_id | str \| None | No | None | Assistant ID to filter by |
| thread_id | str \| None | No | None | Thread ID to filter by |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| int | Number of crons matching criteria |

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3241-3267`

---

## StoreClient

Client for interacting with the graph's shared storage. Provides key-value storage for persisting data across graph executions.

### put_item

**Signature:**
```python
async def put_item(
    self,
    namespace: Sequence[str],
    /,
    key: str,
    value: Mapping[str, Any],
    index: Literal[False] | list[str] | None = None,
    ttl: int | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> None:
```

**Description:**
Store or update an item in the persistent store.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| namespace | Sequence[str] | Yes | - | List of strings representing namespace path (labels cannot contain periods) |
| key | str | Yes | - | Unique identifier for the item within namespace |
| value | Mapping[str, Any] | Yes | - | Dictionary containing the item's data |
| index | Literal[False] \| list[str] \| None | No | None | Search indexing control: None (use defaults), False (disable), or list of field paths to index |
| ttl | int \| None | No | None | Time-to-live in minutes, or None for no expiration |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| None | No return value |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | When namespace label contains a period ('.') |

**Example:**
```python
await client.store.put_item(
    ["documents", "user123"],
    key="item456",
    value={"title": "My Document", "content": "Hello World"}
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3287-3337`

---

### get_item

**Signature:**
```python
async def get_item(
    self,
    namespace: Sequence[str],
    /,
    key: str,
    *,
    refresh_ttl: bool | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> Item:
```

**Description:**
Retrieve a single item from the store.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| namespace | Sequence[str] | Yes | - | List of strings representing namespace path |
| key | str | Yes | - | Unique identifier for the item |
| refresh_ttl | bool \| None | No | None | Whether to refresh TTL on read (None uses store default) |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| Item | The retrieved item with namespace, key, value, created_at, updated_at |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | When namespace label contains a period ('.') |

**Example:**
```python
item = await client.store.get_item(
    ["documents", "user123"],
    key="item456",
)
# {
#     'namespace': ['documents', 'user123'],
#     'key': 'item456',
#     'value': {'title': 'My Document', 'content': 'Hello World'},
#     'created_at': '2024-07-30T12:00:00Z',
#     'updated_at': '2024-07-30T12:00:00Z'
# }
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3339-3394`

---

### delete_item

**Signature:**
```python
async def delete_item(
    self,
    namespace: Sequence[str],
    /,
    key: str,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> None:
```

**Description:**
Delete an item from the store.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| namespace | Sequence[str] | Yes | - | List of strings representing namespace path |
| key | str | Yes | - | Unique identifier for the item |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| None | No return value |

**Example:**
```python
await client.store.delete_item(
    ["documents", "user123"],
    key="item456",
)
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3396-3430`

---

### search_items

**Signature:**
```python
async def search_items(
    self,
    namespace_prefix: Sequence[str],
    /,
    filter: Mapping[str, Any] | None = None,
    limit: int = 10,
    offset: int = 0,
    query: str | None = None,
    refresh_ttl: bool | None = None,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> SearchItemsResponse:
```

**Description:**
Search for items within a namespace prefix.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| namespace_prefix | Sequence[str] | Yes | - | List of strings representing namespace prefix |
| filter | Mapping[str, Any] \| None | No | None | Optional key-value pairs to filter results |
| limit | int | No | 10 | Maximum number of items to return |
| offset | int | No | 0 | Number of items to skip |
| query | str \| None | No | None | Optional natural language search query |
| refresh_ttl | bool \| None | No | None | Whether to refresh TTL on items (None uses store default) |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| SearchItemsResponse | Dictionary with 'items' key containing list of matching items |

**Example:**
```python
items = await client.store.search_items(
    ["documents"],
    filter={"author": "John Doe"},
    limit=5,
    offset=0
)
# {'items': [{'namespace': [...], 'key': '...', 'value': {...}, ...}, ...]}
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3432-3506`

---

### list_namespaces

**Signature:**
```python
async def list_namespaces(
    self,
    prefix: list[str] | None = None,
    suffix: list[str] | None = None,
    max_depth: int | None = None,
    limit: int = 100,
    offset: int = 0,
    headers: Mapping[str, str] | None = None,
    params: QueryParamTypes | None = None,
) -> ListNamespaceResponse:
```

**Description:**
List namespaces with optional match conditions.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| prefix | list[str] \| None | No | None | Optional prefix to filter namespaces |
| suffix | list[str] \| None | No | None | Optional suffix to filter namespaces |
| max_depth | int \| None | No | None | Optional maximum depth of namespaces to return (deeper ones truncated) |
| limit | int | No | 100 | Maximum number of namespaces to return |
| offset | int | No | 0 | Number of namespaces to skip |
| headers | Mapping[str, str] \| None | No | None | Optional custom headers |
| params | QueryParamTypes \| None | No | None | Optional query parameters |

**Returns:**
| Type | Description |
|------|-------------|
| ListNamespaceResponse | Dictionary with 'namespaces' key containing list of namespace paths |

**Example:**
```python
namespaces = await client.store.list_namespaces(
    prefix=["documents"],
    max_depth=3,
    limit=10,
)
# {'namespaces': [['documents', 'user123', 'reports'], ...]}
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py:3508-3565`

---

## Auth Class

### Auth

**Signature:**
```python
class Auth:
    def __init__(self) -> None:
```

**Description:**
Add custom authentication and authorization management to your LangGraph application. Supports custom user authentication protocols and fine-grained authorization rules for different resources and actions.

**Attributes:**
- `on` (_On): Entry point for authorization handlers
- `types`: Reference to auth type definitions
- `exceptions`: Reference to auth exception definitions

**Configuration:**
To use, create a separate Python file and add the path to your `langgraph.json`:

```json
{
  "dependencies": ["."],
  "graphs": {"agent": "./my_agent/agent.py:graph"},
  "env": ".env",
  "auth": {"path": "./auth.py:my_auth"}
}
```

**Example:**
```python
from langgraph_sdk import Auth

my_auth = Auth()

@my_auth.authenticate
async def authenticate(authorization: str) -> str:
    # Verify token and return user_id
    result = await verify_token(authorization)
    if result != "user_id":
        raise Auth.exceptions.HTTPException(
            status_code=401, detail="Unauthorized"
        )
    return result

@my_auth.on
async def authorize_default(ctx: Auth.types.AuthContext, value: dict) -> bool:
    return False  # Reject all requests by default

@my_auth.on.threads.create
async def authorize_thread_create(
    ctx: Auth.types.AuthContext,
    value: Auth.types.ThreadsCreate
) -> None:
    assert value.get("metadata", {}).get("owner") == ctx.user.identity
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/__init__.py:13-188`

---

#### authenticate

**Signature:**
```python
def authenticate(self, fn: Authenticator) -> Authenticator:
```

**Description:**
Register an authentication handler function. The handler verifies credentials and returns user scopes.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| fn | Authenticator | Yes | - | Authentication handler function that returns user representation (string user_id, dict with identity/permissions, or object) |

**Returns:**
| Type | Description |
|------|-------------|
| Authenticator | The registered handler function |

**Raises:**
| Exception | When |
|-----------|------|
| ValueError | If an authentication handler is already registered |

The handler can accept any of these parameters by name:
- `request` (Request): Raw ASGI request object
- `path` (str): Request path
- `method` (str): HTTP method
- `path_params` (dict[str, str]): URL path parameters
- `query_params` (dict[str, str]): URL query parameters
- `headers` (dict[bytes, bytes]): Request headers
- `authorization` (str | None): Authorization header value

**Example:**
```python
@auth.authenticate
async def authenticate(authorization: str) -> Auth.types.MinimalUserDict:
    user = await verify_token(authorization)
    return {
        "identity": user["id"],
        "permissions": user.get("permissions", []),
        "display_name": user["name"],
    }
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/__init__.py:189-265`

---

#### on

**Description:**
Entry point for authorization handlers. Provides decorators for:
- Global handlers (@auth.on)
- Resource-specific handlers (@auth.on.threads)
- Resource and action specific handlers (@auth.on.threads.create)

**Handler Parameters:**
- `ctx` (AuthContext): Contains request context and authenticated user info
- `value`: The data being authorized (type varies by endpoint)

**Handler Return Values:**
- `None` or `True`: Accept the request
- `False`: Reject with 403 error
- `FilterType`: Apply filtering rules to the response

**Available Resources:**
- `auth.on.threads`: Thread operations (create, read, update, delete, search, create_run)
- `auth.on.assistants`: Assistant operations (create, read, update, delete, search)
- `auth.on.crons`: Cron operations (create, read, update, delete, search)
- `auth.on.store`: Store operations (put, get, search, delete, list_namespaces)

**Example:**
```python
# Global handler
@auth.on
async def reject_unhandled(ctx: Auth.types.AuthContext, value: dict) -> bool:
    return False

# Resource-specific handler
@auth.on.threads
async def check_thread_access(ctx: Auth.types.AuthContext, value: dict) -> bool:
    return value.get("created_by") == ctx.user.identity

# Resource and action specific handler
@auth.on.threads.delete
async def prevent_thread_deletion(ctx: Auth.types.AuthContext, value: dict) -> bool:
    return "admin" in ctx.user.permissions

# Store auth (enforcing user creds in namespace)
@auth.on.store
async def check_store_access(ctx: Auth.types.AuthContext, value: dict) -> None:
    assert value["namespace"][0] == ctx.user.identity
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/__init__.py:113-181`

---

## Schema Types

### Type Aliases

**Json**
```python
Json = dict[str, Any] | None
```
Represents a JSON-like structure, can be None or a dictionary with string keys.

**RunStatus**
```python
RunStatus = Literal["pending", "running", "error", "success", "timeout", "interrupted"]
```
Status of a run execution.

**ThreadStatus**
```python
ThreadStatus = Literal["idle", "busy", "interrupted", "error"]
```
Status of a thread.

**StreamMode**
```python
StreamMode = Literal["values", "messages", "updates", "events", "tasks", "checkpoints", "debug", "custom", "messages-tuple"]
```
Streaming mode options.

**ThreadStreamMode**
```python
ThreadStreamMode = Literal["run_modes", "lifecycle", "state_update"]
```
Thread-specific streaming modes.

**DisconnectMode**
```python
DisconnectMode = Literal["cancel", "continue"]
```
Behavior on disconnection.

**MultitaskStrategy**
```python
MultitaskStrategy = Literal["reject", "interrupt", "rollback", "enqueue"]
```
Strategy for handling multiple tasks.

**OnConflictBehavior**
```python
OnConflictBehavior = Literal["raise", "do_nothing"]
```
Behavior on conflict.

**OnCompletionBehavior**
```python
OnCompletionBehavior = Literal["delete", "keep"]
```
Action after completion.

**Durability**
```python
Durability = Literal["sync", "async", "exit"]
```
Durability mode for graph execution.

**IfNotExists**
```python
IfNotExists = Literal["create", "reject"]
```
Behavior if thread doesn't exist.

**CancelAction**
```python
CancelAction = Literal["interrupt", "rollback"]
```
Action to take when cancelling a run.

**SortOrder**
```python
SortOrder = Literal["asc", "desc"]
```
Sorting order.

---

### TypedDict Classes

#### Config

```python
class Config(TypedDict, total=False):
    tags: list[str]
    recursion_limit: int
    configurable: dict[str, Any]
```

Configuration options for a call.

---

#### Checkpoint

```python
class Checkpoint(TypedDict):
    thread_id: str
    checkpoint_ns: str
    checkpoint_id: str | None
    checkpoint_map: dict[str, Any] | None
```

Represents a checkpoint in the execution process.

---

#### GraphSchema

```python
class GraphSchema(TypedDict):
    graph_id: str
    input_schema: dict | None
    output_schema: dict | None
    state_schema: dict | None
    config_schema: dict | None
    context_schema: dict | None
```

Defines the structure and properties of a graph.

---

#### Assistant

```python
class Assistant(TypedDict):
    assistant_id: str
    graph_id: str
    config: Config
    context: Context
    created_at: datetime
    metadata: Json
    version: int
    name: str
    description: str | None
    updated_at: datetime
```

Represents an assistant with configuration and metadata.

---

#### AssistantVersion

```python
class AssistantVersion(TypedDict):
    assistant_id: str
    graph_id: str
    config: Config
    context: Context
    created_at: datetime
    metadata: Json
    version: int
    name: str
    description: str | None
```

Represents a specific version of an assistant.

---

#### AssistantsSearchResponse

```python
class AssistantsSearchResponse(TypedDict):
    assistants: list[Assistant]
    next: str | None
```

Paginated response for assistant search results.

---

#### Interrupt

```python
class Interrupt(TypedDict):
    value: Any
    id: str
```

Represents an interruption in the execution flow.

---

#### Thread

```python
class Thread(TypedDict):
    thread_id: str
    created_at: datetime
    updated_at: datetime
    metadata: Json
    status: ThreadStatus
    values: Json
    interrupts: dict[str, list[Interrupt]]
```

Represents a conversation thread.

---

#### ThreadTask

```python
class ThreadTask(TypedDict):
    id: str
    name: str
    error: str | None
    interrupts: list[Interrupt]
    checkpoint: Checkpoint | None
    state: ThreadState | None
    result: dict[str, Any] | None
```

Represents a task within a thread.

---

#### ThreadState

```python
class ThreadState(TypedDict):
    values: list[dict] | dict[str, Any]
    next: Sequence[str]
    checkpoint: Checkpoint
    metadata: Json
    created_at: str | None
    parent_checkpoint: Checkpoint | None
    tasks: Sequence[ThreadTask]
    interrupts: list[Interrupt]
```

Represents the state of a thread.

---

#### ThreadUpdateStateResponse

```python
class ThreadUpdateStateResponse(TypedDict):
    checkpoint: Checkpoint
```

Response from updating a thread's state.

---

#### Run

```python
class Run(TypedDict):
    run_id: str
    thread_id: str
    assistant_id: str
    created_at: datetime
    updated_at: datetime
    status: RunStatus
    metadata: Json
    multitask_strategy: MultitaskStrategy
```

Represents a single execution run.

---

#### RunCreate

```python
class RunCreate(TypedDict):
    thread_id: str | None
    assistant_id: str
    input: dict | None
    metadata: dict | None
    config: Config | None
    context: Context | None
    checkpoint_id: str | None
    interrupt_before: list[str] | None
    interrupt_after: list[str] | None
    webhook: str | None
    multitask_strategy: MultitaskStrategy | None
```

Parameters for initiating a background run.

---

#### Cron

```python
class Cron(TypedDict):
    cron_id: str
    assistant_id: str
    thread_id: str | None
    end_time: datetime | None
    schedule: str
    created_at: datetime
    updated_at: datetime
    payload: dict
    user_id: str | None
    next_run_date: datetime | None
    metadata: dict
```

Represents a scheduled task.

---

#### Item

```python
class Item(TypedDict):
    namespace: list[str]
    key: str
    value: dict[str, Any]
    created_at: datetime
    updated_at: datetime
```

Represents a single document or data entry in the graph's Store.

---

#### SearchItem

```python
class SearchItem(Item, total=False):
    score: float | None
```

Item with optional relevance score from search operations.

---

#### SearchItemsResponse

```python
class SearchItemsResponse(TypedDict):
    items: list[SearchItem]
```

Response structure for searching items.

---

#### ListNamespaceResponse

```python
class ListNamespaceResponse(TypedDict):
    namespaces: list[list[str]]
```

Response structure for listing namespaces.

---

#### StreamPart

```python
class StreamPart(NamedTuple):
    event: str
    data: dict
    id: str | None = None
```

Represents a part of a stream response.

---

#### Send

```python
class Send(TypedDict):
    node: str
    input: dict[str, Any] | None
```

Represents a message to be sent to a specific node in the graph.

---

#### Command

```python
class Command(TypedDict, total=False):
    goto: Send | str | Sequence[Send | str]
    update: dict[str, Any] | Sequence[tuple[str, Any]]
    resume: Any
```

Commands to control graph execution flow and state.

---

#### RunCreateMetadata

```python
class RunCreateMetadata(TypedDict):
    run_id: str
    thread_id: str | None
```

Metadata for a run creation request.

---

## Exception Classes

### LangGraphError

**Signature:**
```python
class LangGraphError(Exception):
    pass
```

**Description:**
Base exception for all LangGraph SDK errors.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:13-14`

---

### APIError

**Signature:**
```python
class APIError(httpx.HTTPStatusError, LangGraphError):
    message: str
    request: httpx.Request
    body: object | None
    code: str | None
    param: str | None
    type: str | None
```

**Description:**
Base exception for API-related errors. Inherits from both httpx.HTTPStatusError and LangGraphError.

**Attributes:**
- `message` (str): Error message
- `request` (httpx.Request): The HTTP request that caused the error
- `body` (object | None): Response body if available
- `code` (str | None): Error code from response
- `param` (str | None): Parameter that caused error
- `type` (str | None): Error type from response

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:17-60`

---

### APIResponseValidationError

**Signature:**
```python
class APIResponseValidationError(APIError):
    response: httpx.Response
    status_code: int
```

**Description:**
Raised when API response data is invalid for expected schema.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:62-79`

---

### APIStatusError

**Signature:**
```python
class APIStatusError(APIError):
    response: httpx.Response
    status_code: int
    request_id: str | None
```

**Description:**
General API status error with HTTP status code.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:82-93`

---

### APIConnectionError

**Signature:**
```python
class APIConnectionError(APIError):
    pass
```

**Description:**
Raised when connection to the API fails.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:96-100`

---

### APITimeoutError

**Signature:**
```python
class APITimeoutError(APIConnectionError):
    pass
```

**Description:**
Raised when a request times out.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:103-105`

---

### BadRequestError

**Signature:**
```python
class BadRequestError(APIStatusError):
    status_code: Literal[400] = 400
```

**Description:**
HTTP 400 Bad Request error.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:108-109`

---

### AuthenticationError

**Signature:**
```python
class AuthenticationError(APIStatusError):
    status_code: Literal[401] = 401
```

**Description:**
HTTP 401 Unauthorized error - authentication failed.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:112-113`

---

### PermissionDeniedError

**Signature:**
```python
class PermissionDeniedError(APIStatusError):
    status_code: Literal[403] = 403
```

**Description:**
HTTP 403 Forbidden error - insufficient permissions.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:116-117`

---

### NotFoundError

**Signature:**
```python
class NotFoundError(APIStatusError):
    status_code: Literal[404] = 404
```

**Description:**
HTTP 404 Not Found error - resource doesn't exist.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:120-121`

---

### ConflictError

**Signature:**
```python
class ConflictError(APIStatusError):
    status_code: Literal[409] = 409
```

**Description:**
HTTP 409 Conflict error - resource conflict.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:124-125`

---

### UnprocessableEntityError

**Signature:**
```python
class UnprocessableEntityError(APIStatusError):
    status_code: Literal[422] = 422
```

**Description:**
HTTP 422 Unprocessable Entity error - validation failed.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:128-129`

---

### RateLimitError

**Signature:**
```python
class RateLimitError(APIStatusError):
    status_code: Literal[429] = 429
```

**Description:**
HTTP 429 Too Many Requests error - rate limit exceeded.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:132-133`

---

### InternalServerError

**Signature:**
```python
class InternalServerError(APIStatusError):
    pass
```

**Description:**
HTTP 500+ Internal Server Error - server-side error.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py:136-137`

---

## Auth Types

### MinimalUser

**Signature:**
```python
@typing.runtime_checkable
class MinimalUser(typing.Protocol):
    @property
    def identity(self) -> str:
        ...
```

**Description:**
Protocol for user objects. Must expose the identity property.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/types.py:150-161`

---

### MinimalUserDict

**Signature:**
```python
class MinimalUserDict(typing.TypedDict, total=False):
    identity: typing_extensions.Required[str]
    display_name: str
    is_authenticated: bool
    permissions: Sequence[str]
```

**Description:**
Dictionary representation of a user with identity, display name, authentication status, and permissions.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/types.py:164-178`

---

### BaseUser

**Signature:**
```python
@typing.runtime_checkable
class BaseUser(typing.Protocol):
    @property
    def is_authenticated(self) -> bool: ...

    @property
    def display_name(self) -> str: ...

    @property
    def identity(self) -> str: ...

    @property
    def permissions(self) -> Sequence[str]: ...

    def __getitem__(self, key): ...
    def __contains__(self, key): ...
    def __iter__(self): ...
```

**Description:**
The base ASGI user protocol with authentication and permission properties.

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/types.py:181-216`

---

### StudioUser

**Signature:**
```python
class StudioUser:
    def __init__(self, username: str, is_authenticated: bool = False) -> None:
        ...
```

**Description:**
User object populated from authenticated requests from the LangGraph Studio. Can be disabled in `langgraph.json` config.

**Attributes:**
- `username` (str): The username
- `is_authenticated` (bool): Whether user is authenticated
- `identity` (property): Returns username
- `display_name` (property): Returns username
- `permissions` (property): Returns list with "authenticated" if authenticated

**Example:**
```python
@auth.on
async def allow_developers(ctx: Auth.types.AuthContext, value: Any) -> None:
    if isinstance(ctx.user, Auth.types.StudioUser):
        return None  # Allow Studio users
    return False
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/types.py:218-268`

---

### AuthContext

**Signature:**
```python
@dataclass(slots=True)
class AuthContext:
    permissions: Sequence[str]
    user: BaseUser
    resource: Literal["runs", "threads", "crons", "assistants", "store"]
    action: Literal["create", "read", "update", "delete", "search", "create_run", "put", "get", "list_namespaces"]
```

**Description:**
Complete authentication context with resource and action information. Contains the authenticated user, their permissions, and details about the resource/action being accessed.

**Attributes:**
- `permissions` (Sequence[str]): Permissions granted to authenticated user
- `user` (BaseUser): The authenticated user
- `resource`: The resource being accessed
- `action`: The action being performed on the resource

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/types.py:381-417`

---

### HTTPException

**Signature:**
```python
class HTTPException(Exception):
    def __init__(
        self,
        status_code: int = 401,
        detail: str | None = None,
        headers: Mapping[str, str] | None = None,
    ) -> None:
        ...
```

**Description:**
HTTP exception that can be raised in auth handlers to return a specific HTTP error response.

**Parameters:**
| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| status_code | int | No | 401 | HTTP status code |
| detail | str \| None | No | None | Error message (defaults to status code phrase) |
| headers | Mapping[str, str] \| None | No | None | Additional HTTP headers |

**Example:**
```python
raise Auth.exceptions.HTTPException(status_code=401, detail="Unauthorized")
raise Auth.exceptions.HTTPException(status_code=404, detail="Not found")
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/exceptions.py:9-56`

---

## Complete Usage Example

```python
from langgraph_sdk import get_client, Auth

# Initialize async client
async def main():
    async with get_client(url="http://localhost:8123") as client:
        # Create an assistant
        assistant = await client.assistants.create(
            graph_id="agent",
            name="My Assistant",
            metadata={"version": "1.0"}
        )

        # Create a thread
        thread = await client.threads.create(
            metadata={"user_id": "user123"}
        )

        # Stream a run
        async for chunk in client.runs.stream(
            thread_id=thread["thread_id"],
            assistant_id=assistant["assistant_id"],
            input={"messages": [{"role": "user", "content": "Hello!"}]},
            stream_mode=["values", "updates"]
        ):
            print(f"Event: {chunk.event}, Data: {chunk.data}")

        # Get thread state
        state = await client.threads.get_state(thread["thread_id"])
        print(f"Final state: {state['values']}")

        # Store data
        await client.store.put_item(
            ["users", "user123"],
            key="preferences",
            value={"theme": "dark", "language": "en"}
        )

        # Search stored items
        items = await client.store.search_items(
            ["users"],
            filter={"theme": "dark"}
        )

        # Create a cron job
        cron = await client.crons.create(
            assistant_id=assistant["assistant_id"],
            schedule="0 9 * * *",  # Daily at 9 AM
            input={"task": "daily_summary"}
        )

# Run
import asyncio
asyncio.run(main())
```

---

## Additional Resources

- **Source Code**: `/home/user/langgraph/libs/sdk-py/langgraph_sdk/`
- **Main Files**:
  - `client.py`: Client implementations
  - `schema.py`: Type definitions
  - `auth/__init__.py`: Authentication system
  - `errors.py`: Exception classes

For more information, see the LangGraph documentation.
