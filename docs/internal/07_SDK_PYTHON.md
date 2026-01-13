# LangGraph Python SDK Documentation

## Overview

The LangGraph Python SDK provides programmatic access to the LangGraph API server, enabling you to build, deploy, and manage stateful, multi-actor applications. The SDK offers both asynchronous and synchronous clients for interacting with core LangGraph resources including Assistants, Threads, Runs, Cron jobs, and persistent storage.

**Key Features:**
- **Assistants Management**: Create and manage versioned graph configurations
- **Thread Handling**: Manage multi-turn conversational threads with persistent state
- **Run Control**: Execute graphs with streaming, waiting, and background execution modes
- **State Management**: Get and update thread state with checkpoint support
- **Scheduled Execution**: Configure recurring runs with cron-like scheduling
- **Persistent Storage**: Key-value store for cross-thread memory and data sharing
- **Authentication**: Flexible auth configuration with custom headers and API keys
- **Streaming Support**: Real-time streaming of graph execution results

## Installation

```bash
pip install -U langgraph-sdk
```

## Getting Started

### Creating a Client

The SDK provides two main entry points for creating clients:

#### Async Client (Recommended)

```python
from langgraph_sdk import get_client

# Connect to a remote server
client = get_client(url="http://localhost:8123")

# Connect to local server (auto-detected if running via langgraph-cli)
client = get_client()  # Defaults to http://localhost:8123
```

#### Synchronous Client

```python
from langgraph_sdk import get_sync_client

# For synchronous operations
sync_client = get_sync_client(url="http://localhost:8123")
```

### Client Parameters

**`get_client()`** accepts the following parameters:

- **`url`** (str | None): Base URL of the LangGraph API server
  - If `None`, attempts in-process connection (only works inside a LangGraph server)
  - Defaults to `http://localhost:8123` if running locally
- **`api_key`** (str | None): API key for authentication
  - String: use this exact API key
  - `None`: explicitly skip loading from environment
  - Not provided (default): auto-load from environment variables:
    1. `LANGGRAPH_API_KEY`
    2. `LANGSMITH_API_KEY`
    3. `LANGCHAIN_API_KEY`
- **`headers`** (Mapping[str, str] | None): Additional HTTP headers
- **`timeout`** (TimeoutTypes | None): HTTP timeout configuration
  - Can be a float (seconds), httpx.Timeout instance, or tuple (connect, read, write, pool)
  - Defaults: connect=5s, read=300s, write=300s, pool=5s

### Basic Usage Example

```python
from langgraph_sdk import get_client

async def main():
    # Create client
    client = get_client(url="http://localhost:8123")

    # List all assistants
    assistants = await client.assistants.search()
    assistant = assistants[0]

    # Create a thread
    thread = await client.threads.create()

    # Run the assistant and stream results
    input_data = {
        "messages": [{"role": "human", "content": "What's the weather in LA?"}]
    }

    async for chunk in client.runs.stream(
        thread_id=thread['thread_id'],
        assistant_id=assistant['assistant_id'],
        input=input_data
    ):
        print(chunk)

    # Close the client when done
    await client.aclose()

# Or use as a context manager
async def main_with_context():
    async with get_client(url="http://localhost:8123") as client:
        # Your code here
        pass
```

## Client Architecture

### LangGraphClient

The main client class provides access to all SDK functionality through specialized sub-clients:

```python
class LangGraphClient:
    assistants: AssistantsClient  # Manage graph configurations
    threads: ThreadsClient        # Manage conversation threads
    runs: RunsClient             # Control graph execution
    crons: CronClient            # Schedule recurring runs
    store: StoreClient           # Key-value storage
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py` (lines 291-308)

### Sub-Clients Overview

- **AssistantsClient**: Versioned configurations for your graphs
- **ThreadsClient**: Multi-turn interactions with persistent state
- **RunsClient**: Individual graph invocations (stateful or stateless)
- **CronClient**: Scheduled, recurring executions
- **StoreClient**: Shared, persistent data storage

## Assistants API

Assistants are versioned configurations of your graph. They encapsulate the graph ID, configuration, and metadata.

### Get an Assistant

```python
assistant = await client.assistants.get("assistant_id_123")

# Response structure:
# {
#     'assistant_id': 'my_assistant_id',
#     'graph_id': 'agent',
#     'created_at': '2024-06-25T17:10:33.109781+00:00',
#     'updated_at': '2024-06-25T17:10:33.109781+00:00',
#     'config': {},
#     'context': {},
#     'metadata': {'created_by': 'system'},
#     'version': 1,
#     'name': 'my_assistant',
#     'description': 'Assistant description'
# }
```

### Create an Assistant

```python
assistant = await client.assistants.create(
    graph_id="agent",
    name="My Custom Assistant",
    description="An assistant for handling customer queries",
    config={
        "configurable": {
            "model_name": "gpt-4",
            "temperature": 0.7
        }
    },
    context={
        "organization_id": "org_123",
        "permissions": ["read", "write"]
    },
    metadata={"team": "customer_support"},
    if_exists="do_nothing"  # or "raise"
)
```

**Parameters:**
- `graph_id`: The ID of the graph (defined in `langgraph.json`)
- `config`: Runtime configuration for the graph
- `context`: Static context passed to the assistant
- `metadata`: Additional metadata
- `assistant_id`: Optional custom ID
- `if_exists`: `"raise"` (default) or `"do_nothing"`
- `name`: Assistant name
- `description`: Optional description

### Update an Assistant

```python
updated = await client.assistants.update(
    assistant_id="my_assistant_id",
    name="Updated Assistant Name",
    config={"configurable": {"temperature": 0.9}},
    metadata={"team": "sales"}
)
```

### Search Assistants

```python
assistants = await client.assistants.search(
    limit=10,
    offset=0,
    sort_by="created_at",  # or "assistant_id", "graph_id", "name", "updated_at"
    sort_order="desc",     # or "asc"
    metadata={"team": "customer_support"},
    select=["assistant_id", "name", "created_at"]  # Select specific fields
)
```

### Get Assistant Graph Structure

```python
# Get graph structure
graph = await client.assistants.get_graph(
    assistant_id="my_assistant_id",
    xray=False  # Set to True or int for subgraph depth
)

# Response includes nodes and edges:
# {
#     'nodes': [
#         {'id': '__start__', 'type': 'schema', 'data': '__start__'},
#         {'id': 'agent', 'type': 'runnable', 'data': {...}},
#         {'id': '__end__', 'type': 'schema', 'data': '__end__'}
#     ],
#     'edges': [
#         {'source': '__start__', 'target': 'agent'},
#         {'source': 'agent', 'target': '__end__'}
#     ]
# }
```

### Get Assistant Schemas

```python
schemas = await client.assistants.get_schemas("my_assistant_id")

# Returns GraphSchema with:
# - input_schema: JSON schema for graph input
# - output_schema: JSON schema for graph output
# - state_schema: JSON schema for graph state
# - config_schema: JSON schema for configuration
# - context_schema: JSON schema for context
```

### Get Subgraphs

```python
subgraphs = await client.assistants.get_subgraphs(
    assistant_id="my_assistant_id",
    namespace="subgraph_namespace",  # Optional
    recurse=True  # Include nested subgraphs
)
```

### Delete an Assistant

```python
await client.assistants.delete("assistant_id_to_delete")
```

### Get Assistant Versions

```python
versions = await client.assistants.get_versions(
    assistant_id="my_assistant_id",
    limit=10,
    offset=0
)
```

## Threads API

Threads maintain the state of a graph across multiple interactions. They persist graph state between runs.

### Create a Thread

```python
# Basic thread creation
thread = await client.threads.create()

# With custom ID and metadata
thread = await client.threads.create(
    thread_id="custom-thread-id",
    metadata={
        "user_id": "user_123",
        "session_id": "session_456"
    },
    if_exists="do_nothing",  # or "raise"
    graph_id="agent",
    ttl=43200  # Time-to-live in minutes (30 days)
)

# With custom TTL configuration
thread = await client.threads.create(
    thread_id="my-thread",
    ttl={
        "ttl": 1440,  # 1 day in minutes
        "strategy": "delete"  # Delete thread after TTL expires
    }
)

# Response:
# {
#     'thread_id': 'my_thread_id',
#     'created_at': '2024-07-18T18:35:15.540834+00:00',
#     'updated_at': '2024-07-18T18:35:15.540834+00:00',
#     'metadata': {'user_id': 'user_123'},
#     'status': 'idle',
#     'values': {},
#     'interrupts': {}
# }
```

### Get a Thread

```python
thread = await client.threads.get("thread_id_123")
```

### Update Thread Metadata

```python
updated_thread = await client.threads.update(
    thread_id="my-thread-id",
    metadata={"status": "active", "priority": "high"},
    ttl=86400  # Update TTL to 60 days
)
```

### Search Threads

```python
threads = await client.threads.search(
    metadata={"user_id": "user_123"},
    status="busy",  # or "idle", "interrupted", "error"
    limit=20,
    offset=0,
    sort_by="created_at",  # or "thread_id", "status", "updated_at"
    sort_order="desc",
    select=["thread_id", "metadata", "status"]  # Select specific fields
)
```

### Count Threads

```python
count = await client.threads.count(
    metadata={"user_id": "user_123"},
    status="idle"
)
```

### Delete a Thread

```python
await client.threads.delete("thread_id_to_delete")
```

### Get Thread State

```python
# Get current state
state = await client.threads.get_state(
    thread_id="my_thread_id"
)

# Get state at specific checkpoint
state = await client.threads.get_state(
    thread_id="my_thread_id",
    checkpoint_id="checkpoint_123"
)

# Include subgraph states
state = await client.threads.get_state(
    thread_id="my_thread_id",
    subgraphs=True
)

# Response structure:
# {
#     'values': {'messages': [...]},  # Current state values
#     'next': [],  # Next nodes to execute
#     'checkpoint': {
#         'thread_id': '...',
#         'checkpoint_ns': '',
#         'checkpoint_id': '...'
#     },
#     'metadata': {...},
#     'created_at': '2024-07-25T15:35:44.184703+00:00',
#     'parent_checkpoint': {...},
#     'tasks': [],
#     'interrupts': []
# }
```

### Update Thread State

```python
response = await client.threads.update_state(
    thread_id="my_thread_id",
    values={"messages": [{"role": "user", "content": "New message"}]},
    as_node="my_node"  # Update as if this node just executed
)

# Returns:
# {
#     'checkpoint': {
#         'thread_id': '...',
#         'checkpoint_ns': '',
#         'checkpoint_id': '...',
#         'checkpoint_map': {}
#     }
# }
```

### Get State History

```python
# Get all historical states
history = await client.threads.get_history(
    thread_id="my_thread_id",
    limit=10,
    before=None,  # Optional checkpoint to query before
    metadata=None,  # Optional metadata filter
    select=None    # Optional field selection
)

# Iterate through history
for state in history:
    print(f"Checkpoint: {state['checkpoint']['checkpoint_id']}")
    print(f"Values: {state['values']}")
```

## Runs API

Runs represent individual executions of an assistant. They can be stateful (on threads) or stateless.

### Stream a Run

The most common way to execute a graph with real-time output:

```python
input_data = {
    "messages": [{"role": "user", "content": "Hello!"}]
}

async for chunk in client.runs.stream(
    thread_id="my_thread_id",  # or None for stateless
    assistant_id="agent",
    input=input_data,
    stream_mode="values",  # or ["values", "updates", "events"]
    metadata={"session": "abc123"},
    config={
        "configurable": {"model_name": "gpt-4"}
    },
    context={"user_tier": "premium"},
    interrupt_before=["human_review_node"],
    interrupt_after=["data_processing_node"],
    multitask_strategy="interrupt",  # or "reject", "rollback", "enqueue"
    on_disconnect="cancel"  # or "continue"
):
    if chunk.event == "metadata":
        print(f"Run started: {chunk.data['run_id']}")
    elif chunk.event == "values":
        print(f"State update: {chunk.data}")
    elif chunk.event == "end":
        print("Run completed")
```

**Stream Modes:**
- `"values"`: Stream complete state values
- `"messages"`: Stream message updates
- `"updates"`: Stream state updates
- `"events"`: Stream all events
- `"tasks"`: Stream task start/finish events
- `"checkpoints"`: Stream checkpoints as created
- `"debug"`: Stream detailed debug information
- `"custom"`: Stream custom events
- `"messages-tuple"`: Stream messages as tuples

**Parameters:**
- `stream_subgraphs`: Stream output from subgraphs (default: False)
- `stream_resumable`: Make stream resumable after disconnection (default: False)
- `checkpoint`: Resume from specific checkpoint
- `checkpoint_id`: (deprecated) Checkpoint ID to resume from
- `interrupt_before`: List of nodes to interrupt before
- `interrupt_after`: List of nodes to interrupt after
- `feedback_keys`: Keys for feedback collection
- `webhook`: URL for webhook notifications
- `multitask_strategy`: How to handle concurrent runs
  - `"reject"`: Reject new runs when busy
  - `"interrupt"`: Interrupt current run
  - `"rollback"`: Rollback and start new run
  - `"enqueue"`: Queue the new run
- `if_not_exists`: `"create"` or `"reject"` for missing threads
- `after_seconds`: Delay before starting (for scheduling)
- `on_disconnect`: `"cancel"` or `"continue"`
- `on_completion`: `"delete"` or `"keep"` (for stateless runs)
- `durability`: `"sync"`, `"async"`, or `"exit"`
  - `"sync"`: Checkpoint synchronously after each step
  - `"async"`: Checkpoint asynchronously while executing next step
  - `"exit"`: Only checkpoint at the end

### Create a Background Run

```python
run = await client.runs.create(
    thread_id="my_thread_id",
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Process this"}]},
    metadata={"priority": "high"},
    multitask_strategy="enqueue"
)

# Response:
# {
#     'run_id': 'run_123',
#     'thread_id': 'my_thread_id',
#     'assistant_id': 'agent',
#     'created_at': '2024-07-25T15:35:42.598503+00:00',
#     'updated_at': '2024-07-25T15:35:42.598503+00:00',
#     'status': 'pending',  # or "running", "success", "error", "timeout", "interrupted"
#     'metadata': {'priority': 'high'},
#     'multitask_strategy': 'enqueue'
# }
```

### Wait for Run Completion

Execute a run and wait for it to complete, returning the final state:

```python
final_state = await client.runs.wait(
    thread_id="my_thread_id",  # or None for stateless
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Analyze this"}]},
    metadata={"request_id": "req_123"},
    raise_error=True  # Raise exception if run fails
)

# Returns the final state values:
# {
#     'messages': [
#         {'role': 'user', 'content': 'Analyze this'},
#         {'role': 'assistant', 'content': 'Analysis complete...'}
#     ]
# }
```

### List Runs

```python
runs = await client.runs.list(
    thread_id="my_thread_id",
    limit=10,
    offset=0,
    status="success",  # Filter by status
    select=["run_id", "status", "created_at"]
)
```

### Get a Run

```python
run = await client.runs.get(
    thread_id="my_thread_id",
    run_id="run_123"
)
```

### Cancel a Run

```python
await client.runs.cancel(
    thread_id="my_thread_id",
    run_id="run_123",
    action="interrupt",  # or "rollback" to delete run and checkpoints
    wait=True  # Wait for cancellation to complete
)
```

### Join a Run

Block until a run completes and get the final state:

```python
result = await client.runs.join(
    thread_id="my_thread_id",
    run_id="run_123"
)
```

### Join and Stream a Run

Stream output from an existing run until completion:

```python
async for chunk in client.runs.join_stream(
    thread_id="my_thread_id",
    run_id="run_123",
    stream_mode=["values", "updates"],
    cancel_on_disconnect=False
):
    print(chunk)
```

### Delete a Run

```python
await client.runs.delete(
    thread_id="my_thread_id",
    run_id="run_123"
)
```

## State API

State management is handled through the Threads API. See the [Get Thread State](#get-thread-state) and [Update Thread State](#update-thread-state) sections above.

### Key State Operations

**Get Current State:**
```python
state = await client.threads.get_state(thread_id="my_thread")
current_values = state['values']
next_nodes = state['next']
```

**Update State:**
```python
await client.threads.update_state(
    thread_id="my_thread",
    values={"counter": 42, "status": "active"},
    as_node="processing_node"
)
```

**Get State History:**
```python
history = await client.threads.get_history(
    thread_id="my_thread",
    limit=5
)
```

**Resume from Checkpoint:**
```python
async for chunk in client.runs.stream(
    thread_id="my_thread",
    assistant_id="agent",
    checkpoint=state['checkpoint'],  # Resume from specific checkpoint
    input={"action": "continue"}
):
    print(chunk)
```

## Crons API

Schedule recurring runs with cron-like syntax.

### Create a Cron Job for a Thread

```python
cron = await client.crons.create_for_thread(
    thread_id="my-thread-id",
    assistant_id="agent",
    schedule="0 9 * * *",  # Every day at 9 AM (cron format)
    input={"task": "daily_summary"},
    metadata={"type": "daily_report"},
    context={"report_type": "summary"},
    config={"configurable": {"verbose": True}},
    interrupt_before=["review_step"],
    interrupt_after=["email_step"],
    webhook="https://my.webhook.com/cron-complete",
    multitask_strategy="enqueue"
)
```

### Create a Stateless Cron Job

```python
cron = await client.crons.create(
    assistant_id="agent",
    schedule="*/30 * * * *",  # Every 30 minutes
    input={"task": "health_check"},
    metadata={"type": "monitoring"}
)
```

**Cron Schedule Format:**
Standard cron syntax: `minute hour day month day_of_week`
- `0 9 * * *` - Every day at 9:00 AM
- `*/15 * * * *` - Every 15 minutes
- `0 0 * * 0` - Every Sunday at midnight
- `0 12 1 * *` - First day of every month at noon

### Get a Cron Job

```python
cron = await client.crons.get("cron_id_123")
```

### Search Cron Jobs

```python
crons = await client.crons.search(
    assistant_id="agent",
    thread_id="my-thread-id",
    limit=10,
    offset=0,
    sort_by="next_run_date",  # or "cron_id", "assistant_id", "created_at", "updated_at"
    sort_order="asc"
)
```

### Update a Cron Job

```python
updated = await client.crons.update(
    cron_id="cron_123",
    schedule="0 10 * * *",  # Change to 10 AM
    metadata={"updated": True}
)
```

### Delete a Cron Job

```python
await client.crons.delete("cron_id_123")
```

## Store API

The Store provides persistent, key-value storage for sharing data across threads and graph executions.

### Put an Item

```python
await client.store.put_item(
    namespace=["users", "user_123"],
    key="preferences",
    value={
        "theme": "dark",
        "language": "en",
        "notifications": True
    },
    index=["theme", "language"],  # Index these fields for search
    ttl=43200  # Time-to-live in minutes
)
```

**Parameters:**
- `namespace`: List of strings forming a hierarchical path
- `key`: Unique identifier within the namespace
- `value`: Dictionary containing the data
- `index`: Fields to index for search (None = use defaults, False = disable, list = specific fields)
- `ttl`: Optional expiration time in minutes

### Get an Item

```python
item = await client.store.get_item(
    namespace=["users", "user_123"],
    key="preferences",
    refresh_ttl=True  # Refresh TTL on read
)

# Response:
# {
#     'namespace': ['users', 'user_123'],
#     'key': 'preferences',
#     'value': {'theme': 'dark', 'language': 'en', 'notifications': True},
#     'created_at': '2024-07-30T12:00:00Z',
#     'updated_at': '2024-07-30T12:00:00Z'
# }
```

### Search Items

```python
results = await client.store.search_items(
    namespace_prefix=["users"],
    filter={"theme": "dark"},  # Filter by indexed fields
    query="notification settings",  # Natural language search (if supported)
    limit=10,
    offset=0,
    refresh_ttl=False
)

# Response:
# {
#     'items': [
#         {
#             'namespace': ['users', 'user_123'],
#             'key': 'preferences',
#             'value': {...},
#             'created_at': '...',
#             'updated_at': '...',
#             'score': 0.95  # Relevance score for natural language queries
#         },
#         ...
#     ]
# }
```

### Delete an Item

```python
await client.store.delete_item(
    namespace=["users", "user_123"],
    key="preferences"
)
```

### List Namespaces

```python
namespaces = await client.store.list_namespaces(
    prefix=["users"],
    suffix=None,
    max_depth=3,
    limit=100,
    offset=0
)

# Response:
# {
#     'namespaces': [
#         ['users', 'user_123', 'settings'],
#         ['users', 'user_456', 'preferences'],
#         ...
#     ]
# }
```

### Store Usage Patterns

**Cross-Thread Memory:**
```python
# Store user context accessible across threads
await client.store.put_item(
    namespace=["memory", "user_123"],
    key="conversation_summary",
    value={"topics": ["weather", "news"], "sentiment": "positive"}
)

# Retrieve in any thread
memory = await client.store.get_item(
    namespace=["memory", "user_123"],
    key="conversation_summary"
)
```

**Shared Configuration:**
```python
# Store application configuration
await client.store.put_item(
    namespace=["config", "app"],
    key="feature_flags",
    value={"new_ui": True, "beta_features": False}
)
```

## Authentication

### API Key Authentication

The SDK supports multiple ways to provide API keys:

**Environment Variables:**
```bash
# Preferred methods (in order of precedence)
export LANGGRAPH_API_KEY="your-api-key"
export LANGSMITH_API_KEY="your-api-key"
export LANGCHAIN_API_KEY="your-api-key"
```

**Explicit API Key:**
```python
client = get_client(
    url="http://localhost:8123",
    api_key="your-api-key"
)
```

**Disable API Key:**
```python
# Explicitly skip API key loading
client = get_client(
    url="http://localhost:8123",
    api_key=None
)
```

### Custom Headers

Add custom headers for additional authentication or metadata:

```python
client = get_client(
    url="http://localhost:8123",
    headers={
        "X-Custom-Auth": "bearer-token",
        "X-Request-ID": "req-123",
        "X-User-Agent": "my-app/1.0"
    }
)
```

**Note:** Reserved headers like `x-api-key` cannot be set via custom headers.

### Server-Side Authentication

For server-side authentication and authorization, use the `Auth` class:

```python
from langgraph_sdk import Auth

auth = Auth()

@auth.authenticate
async def authenticate(authorization: str) -> str:
    """Verify token and return user ID"""
    user_id = verify_token(authorization)
    if not user_id:
        raise auth.exceptions.HTTPException(
            status_code=401,
            detail="Invalid token"
        )
    return user_id

@auth.on.threads.create
async def authorize_thread_create(ctx: Auth.types.AuthContext, value: dict):
    """Control who can create threads"""
    # Only allow if user owns the metadata
    assert value.get("metadata", {}).get("owner") == ctx.user.identity

@auth.on.store
async def authorize_store(ctx: Auth.types.AuthContext, value: dict):
    """Ensure users can only access their own data"""
    assert ctx.user.identity in value["namespace"]
```

Configure in `langgraph.json`:
```json
{
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "auth": {
    "path": "./auth.py:auth"
  }
}
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/__init__.py`

## Streaming

Streaming provides real-time updates as your graph executes.

### Stream Modes

Different modes provide different levels of detail:

```python
# Single mode
async for chunk in client.runs.stream(
    thread_id="my_thread",
    assistant_id="agent",
    input={"query": "hello"},
    stream_mode="values"
):
    print(chunk)

# Multiple modes
async for chunk in client.runs.stream(
    thread_id="my_thread",
    assistant_id="agent",
    input={"query": "hello"},
    stream_mode=["values", "updates", "events", "debug"]
):
    if chunk.event == "values":
        print(f"State: {chunk.data}")
    elif chunk.event == "updates":
        print(f"Update: {chunk.data}")
    elif chunk.event == "events":
        print(f"Event: {chunk.data}")
    elif chunk.event == "debug":
        print(f"Debug: {chunk.data}")
```

### Stream Part Structure

Each chunk is a `StreamPart` NamedTuple:

```python
StreamPart(
    event="values",  # Event type
    data={...},      # Event data
    id="evt_123"     # Optional event ID
)
```

### Common Stream Events

- **`metadata`**: Run metadata (run_id, thread_id)
- **`values`**: Complete state values
- **`updates`**: Incremental state updates
- **`events`**: Graph execution events
- **`tasks`**: Task start/finish events
- **`checkpoints`**: Checkpoint creation events
- **`end`**: Run completion
- **`error`**: Error information

### Resumable Streams

Enable resumable streaming to replay from disconnection:

```python
async for chunk in client.runs.stream(
    thread_id="my_thread",
    assistant_id="agent",
    input={"query": "hello"},
    stream_resumable=True  # Enable resumable streaming
):
    print(chunk)
```

### Streaming with Callbacks

```python
def on_run_created(metadata):
    print(f"Run created: {metadata['run_id']}")

async for chunk in client.runs.stream(
    thread_id="my_thread",
    assistant_id="agent",
    input={"query": "hello"},
    on_run_created=on_run_created
):
    print(chunk)
```

### Subgraph Streaming

Stream output from nested subgraphs:

```python
async for chunk in client.runs.stream(
    thread_id="my_thread",
    assistant_id="agent",
    input={"query": "hello"},
    stream_subgraphs=True  # Include subgraph output
):
    print(chunk)
```

## Error Handling

The SDK provides structured exceptions for different error scenarios.

### Exception Hierarchy

```python
LangGraphError (base exception)
└── APIError
    ├── APIConnectionError
    │   └── APITimeoutError
    ├── APIStatusError
    │   ├── BadRequestError (400)
    │   ├── AuthenticationError (401)
    │   ├── PermissionDeniedError (403)
    │   ├── NotFoundError (404)
    │   ├── ConflictError (409)
    │   ├── UnprocessableEntityError (422)
    │   ├── RateLimitError (429)
    │   └── InternalServerError (500+)
    └── APIResponseValidationError
```

**Source:** `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py`

### Exception Attributes

All API exceptions include:
- **`message`**: Error message
- **`request`**: Original HTTP request
- **`response`**: HTTP response (if available)
- **`status_code`**: HTTP status code
- **`body`**: Response body
- **`code`**: Error code (if provided)
- **`type`**: Error type (if provided)

### Error Handling Examples

**Basic Error Handling:**
```python
from langgraph_sdk import get_client
from langgraph_sdk.errors import (
    NotFoundError,
    AuthenticationError,
    RateLimitError,
    APITimeoutError
)

try:
    client = get_client(url="http://localhost:8123")
    assistant = await client.assistants.get("nonexistent_id")
except NotFoundError as e:
    print(f"Assistant not found: {e.message}")
except AuthenticationError as e:
    print(f"Authentication failed: {e.message}")
    print(f"Status: {e.status_code}")
except APITimeoutError as e:
    print(f"Request timed out: {e.message}")
except RateLimitError as e:
    print(f"Rate limit exceeded: {e.message}")
    print(f"Retry after checking headers")
```

**Handling Run Errors:**
```python
try:
    final_state = await client.runs.wait(
        thread_id="my_thread",
        assistant_id="agent",
        input={"query": "test"},
        raise_error=True  # Raise exception if run fails
    )
except Exception as e:
    print(f"Run failed: {e}")

    # Get the failed run details
    runs = await client.runs.list(
        thread_id="my_thread",
        limit=1,
        status="error"
    )
    if runs:
        print(f"Error details: {runs[0]}")
```

**Graceful Degradation:**
```python
from langgraph_sdk.errors import APIConnectionError, APITimeoutError

try:
    async for chunk in client.runs.stream(
        thread_id="my_thread",
        assistant_id="agent",
        input={"query": "hello"},
        on_disconnect="continue"  # Continue run on disconnect
    ):
        print(chunk)
except (APIConnectionError, APITimeoutError):
    print("Connection lost, but run continues on server")

    # Later, rejoin the stream
    async for chunk in client.runs.join_stream(
        thread_id="my_thread",
        run_id=last_run_id
    ):
        print(chunk)
```

**Context Manager for Cleanup:**
```python
try:
    async with get_client(url="http://localhost:8123") as client:
        result = await client.runs.wait(
            thread_id=None,
            assistant_id="agent",
            input={"query": "test"}
        )
except Exception as e:
    print(f"Error: {e}")
# Client is automatically closed even if error occurs
```

## Complete Examples

### Example 1: Simple Conversational Agent

```python
from langgraph_sdk import get_client

async def run_conversation():
    async with get_client(url="http://localhost:8123") as client:
        # Get or create assistant
        assistants = await client.assistants.search(graph_id="agent", limit=1)
        assistant_id = assistants[0]["assistant_id"]

        # Create a thread for the conversation
        thread = await client.threads.create(
            metadata={"user_id": "user_123", "session": "conv_001"}
        )
        thread_id = thread["thread_id"]

        # First message
        print("User: What's the weather in San Francisco?")
        async for chunk in client.runs.stream(
            thread_id=thread_id,
            assistant_id=assistant_id,
            input={
                "messages": [
                    {"role": "user", "content": "What's the weather in San Francisco?"}
                ]
            },
            stream_mode="values"
        ):
            if chunk.event == "values":
                messages = chunk.data.get("messages", [])
                if messages:
                    last_message = messages[-1]
                    if last_message.get("role") == "assistant":
                        print(f"Assistant: {last_message.get('content')}")

        # Follow-up message (state persisted in thread)
        print("\nUser: How about in New York?")
        async for chunk in client.runs.stream(
            thread_id=thread_id,
            assistant_id=assistant_id,
            input={
                "messages": [
                    {"role": "user", "content": "How about in New York?"}
                ]
            },
            stream_mode="values"
        ):
            if chunk.event == "values":
                messages = chunk.data.get("messages", [])
                if messages:
                    last_message = messages[-1]
                    if last_message.get("role") == "assistant":
                        print(f"Assistant: {last_message.get('content')}")
```

### Example 2: Background Processing with Status Monitoring

```python
from langgraph_sdk import get_client
import asyncio

async def process_documents():
    async with get_client(url="http://localhost:8123") as client:
        assistant_id = "document_processor"

        # Create thread for the processing job
        thread = await client.threads.create(
            metadata={"job_type": "document_processing"}
        )
        thread_id = thread["thread_id"]

        # Start background run
        run = await client.runs.create(
            thread_id=thread_id,
            assistant_id=assistant_id,
            input={
                "documents": ["doc1.pdf", "doc2.pdf", "doc3.pdf"],
                "operation": "extract_and_summarize"
            },
            metadata={"priority": "high"}
        )

        print(f"Started run: {run['run_id']}")

        # Poll for status
        while True:
            current_run = await client.runs.get(
                thread_id=thread_id,
                run_id=run["run_id"]
            )

            status = current_run["status"]
            print(f"Status: {status}")

            if status in ["success", "error", "timeout"]:
                break

            await asyncio.sleep(2)  # Check every 2 seconds

        # Get final state
        if status == "success":
            state = await client.threads.get_state(thread_id)
            print(f"Results: {state['values']}")
        else:
            print(f"Run failed with status: {status}")
```

### Example 3: Human-in-the-Loop with Interrupts

```python
from langgraph_sdk import get_client

async def human_in_loop_workflow():
    async with get_client(url="http://localhost:8123") as client:
        assistant_id = "approval_workflow"

        thread = await client.threads.create()
        thread_id = thread["thread_id"]

        # Start run with interrupt before approval node
        print("Starting workflow...")
        async for chunk in client.runs.stream(
            thread_id=thread_id,
            assistant_id=assistant_id,
            input={"request": "Deploy to production", "changes": ["feature_x"]},
            interrupt_before=["approval_node"],
            stream_mode="values"
        ):
            if chunk.event == "values":
                print(f"State: {chunk.data}")

        # Check for interrupts
        state = await client.threads.get_state(thread_id)
        if state["next"] == ["approval_node"]:
            print("\nWorkflow paused for approval")
            print(f"Changes to approve: {state['values'].get('changes')}")

            # Simulate human approval
            approval = input("Approve? (yes/no): ")

            # Update state with approval decision
            await client.threads.update_state(
                thread_id=thread_id,
                values={"approved": approval.lower() == "yes"},
                as_node="approval_node"
            )

            # Resume the workflow
            print("\nResuming workflow...")
            final_state = await client.runs.wait(
                thread_id=thread_id,
                assistant_id=assistant_id,
                input=None  # Continue from current state
            )

            print(f"Final result: {final_state}")
```

### Example 4: Scheduled Reports with Cron

```python
from langgraph_sdk import get_client

async def setup_daily_report():
    async with get_client(url="http://localhost:8123") as client:
        assistant_id = "report_generator"

        # Create a persistent thread for reports
        thread = await client.threads.create(
            thread_id="daily-reports",
            metadata={"type": "scheduled_reports"}
        )

        # Schedule daily report at 9 AM
        cron = await client.crons.create_for_thread(
            thread_id=thread["thread_id"],
            assistant_id=assistant_id,
            schedule="0 9 * * *",  # Every day at 9 AM
            input={
                "report_type": "daily_summary",
                "recipients": ["team@company.com"]
            },
            metadata={"automated": True},
            webhook="https://api.company.com/report-complete"
        )

        print(f"Scheduled cron job: {cron['cron_id']}")
        print(f"Next run: {cron['next_run_date']}")

        # List all scheduled jobs
        all_crons = await client.crons.search()
        print(f"\nAll scheduled jobs: {len(all_crons)}")
        for job in all_crons:
            print(f"  - {job['cron_id']}: {job['schedule']}")
```

### Example 5: Multi-User Application with Store

```python
from langgraph_sdk import get_client

async def multi_user_app():
    async with get_client(url="http://localhost:8123") as client:
        user_id = "user_123"
        assistant_id = "chat_assistant"

        # Load user preferences from store
        try:
            prefs = await client.store.get_item(
                namespace=["users", user_id],
                key="preferences"
            )
            user_context = prefs["value"]
        except:
            # First time user - set defaults
            user_context = {
                "language": "en",
                "theme": "light",
                "model": "gpt-4"
            }
            await client.store.put_item(
                namespace=["users", user_id],
                key="preferences",
                value=user_context
            )

        # Create thread with user context
        thread = await client.threads.create(
            metadata={"user_id": user_id}
        )

        # Run with user preferences
        async for chunk in client.runs.stream(
            thread_id=thread["thread_id"],
            assistant_id=assistant_id,
            input={"message": "Hello!"},
            context=user_context,  # Pass user preferences
            stream_mode="values"
        ):
            if chunk.event == "values":
                print(chunk.data)

        # Save conversation summary to store
        state = await client.threads.get_state(thread["thread_id"])
        await client.store.put_item(
            namespace=["conversations", user_id],
            key=thread["thread_id"],
            value={
                "summary": "User greeted the assistant",
                "message_count": len(state["values"].get("messages", [])),
                "last_active": state["created_at"]
            }
        )

        # Search user's conversations
        user_conversations = await client.store.search_items(
            namespace_prefix=["conversations", user_id],
            limit=10
        )
        print(f"\nUser has {len(user_conversations['items'])} conversations")
```

### Example 6: Error Recovery and Retry Logic

```python
from langgraph_sdk import get_client
from langgraph_sdk.errors import (
    APITimeoutError,
    RateLimitError,
    InternalServerError
)
import asyncio

async def run_with_retry(max_retries=3):
    async with get_client(url="http://localhost:8123") as client:
        assistant_id = "agent"

        for attempt in range(max_retries):
            try:
                # Try to run
                result = await client.runs.wait(
                    thread_id=None,
                    assistant_id=assistant_id,
                    input={"query": "Important request"},
                    on_disconnect="continue"  # Continue if disconnected
                )

                print(f"Success: {result}")
                return result

            except APITimeoutError:
                print(f"Timeout on attempt {attempt + 1}/{max_retries}")
                if attempt < max_retries - 1:
                    await asyncio.sleep(2 ** attempt)  # Exponential backoff

            except RateLimitError as e:
                print(f"Rate limited: {e.message}")
                # Parse retry-after header if available
                wait_time = 60  # Default wait
                if e.response and "retry-after" in e.response.headers:
                    wait_time = int(e.response.headers["retry-after"])
                print(f"Waiting {wait_time} seconds...")
                await asyncio.sleep(wait_time)

            except InternalServerError:
                print(f"Server error on attempt {attempt + 1}/{max_retries}")
                if attempt < max_retries - 1:
                    await asyncio.sleep(5)

        raise Exception(f"Failed after {max_retries} attempts")
```

### Example 7: Stateless Processing Pipeline

```python
from langgraph_sdk import get_client

async def process_batch():
    async with get_client(url="http://localhost:8123") as client:
        assistant_id = "data_processor"
        items = ["item1", "item2", "item3"]

        # Process each item in a stateless run
        results = []
        for item in items:
            print(f"Processing {item}...")

            result = await client.runs.wait(
                thread_id=None,  # Stateless run
                assistant_id=assistant_id,
                input={"data": item},
                on_completion="delete"  # Clean up temporary thread
            )

            results.append(result)
            print(f"  Result: {result}")

        print(f"\nProcessed {len(results)} items")
        return results
```

## Best Practices

### 1. Resource Management

Always close clients or use context managers:

```python
# Preferred: Context manager
async with get_client(url="http://localhost:8123") as client:
    # Use client
    pass

# Alternative: Manual cleanup
client = get_client(url="http://localhost:8123")
try:
    # Use client
    pass
finally:
    await client.aclose()
```

### 2. Thread Reuse

Reuse threads for related conversations:

```python
# Create once
thread = await client.threads.create(metadata={"user_id": "user_123"})

# Reuse for multiple runs
for message in user_messages:
    await client.runs.wait(
        thread_id=thread["thread_id"],
        assistant_id=assistant_id,
        input={"message": message}
    )
```

### 3. Error Handling

Always handle potential errors:

```python
from langgraph_sdk.errors import NotFoundError, APIError

try:
    state = await client.threads.get_state(thread_id)
except NotFoundError:
    # Thread doesn't exist, create it
    thread = await client.threads.create(thread_id=thread_id)
except APIError as e:
    # Handle other API errors
    logger.error(f"API error: {e.message}")
```

### 4. Streaming for Long Operations

Use streaming for long-running operations to get real-time feedback:

```python
async for chunk in client.runs.stream(
    thread_id=thread_id,
    assistant_id=assistant_id,
    input=input_data,
    stream_mode=["values", "events"]
):
    # Process updates in real-time
    if chunk.event == "events":
        print(f"Progress: {chunk.data}")
```

### 5. Store Namespacing

Use hierarchical namespaces for organization:

```python
# Good: Hierarchical organization
await client.store.put_item(
    namespace=["app", "users", user_id, "settings"],
    key="preferences",
    value=data
)

# Avoid: Flat namespace
await client.store.put_item(
    namespace=["settings"],
    key=f"user_{user_id}_preferences",  # Don't encode hierarchy in key
    value=data
)
```

## Additional Resources

- **Main Repository**: `/home/user/langgraph`
- **SDK Source**: `/home/user/langgraph/libs/sdk-py/langgraph_sdk/`
- **Client Implementation**: `/home/user/langgraph/libs/sdk-py/langgraph_sdk/client.py`
- **Schema Definitions**: `/home/user/langgraph/libs/sdk-py/langgraph_sdk/schema.py`
- **Authentication**: `/home/user/langgraph/libs/sdk-py/langgraph_sdk/auth/__init__.py`
- **Error Types**: `/home/user/langgraph/libs/sdk-py/langgraph_sdk/errors.py`

---

**SDK Version**: 0.3.1

This documentation covers the core functionality of the LangGraph Python SDK. For the latest updates and additional features, refer to the source code and official documentation.
