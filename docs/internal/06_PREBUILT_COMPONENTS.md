# LangGraph Prebuilt Components

## Overview

The `langgraph.prebuilt` module provides high-level APIs for creating and executing agents and tools in LangGraph workflows. These prebuilt components handle common patterns like:

- **ReAct Agents**: Tool-calling agents that reason and act in a loop
- **Tool Execution**: Robust tool invocation with error handling and state injection
- **Tool Validation**: Schema-based validation without execution
- **Conditional Routing**: Route based on tool calls
- **Human-in-the-Loop**: Interrupt patterns for human approval workflows

### Why Use Prebuilt Components?

Prebuilt components abstract away complexity while maintaining flexibility:

- **Faster Development**: Get production-ready agents in minutes
- **Best Practices**: Built-in error handling, parallel execution, and validation
- **Customizable**: Extend with hooks, custom state, and interceptors
- **Type-Safe**: Full type annotations for IDE support

### Module Structure

```python
from langgraph.prebuilt import (
    create_react_agent,    # Create ReAct agents
    ToolNode,              # Execute tools
    ValidationNode,        # Validate tool calls
    tools_condition,       # Route based on tool calls
    InjectedState,         # Inject graph state into tools
    InjectedStore,         # Inject persistent store into tools
    ToolRuntime,           # Runtime context for tools
)
```

---

## create_react_agent()

The `create_react_agent()` function creates a ReAct-style agent that calls tools in a loop until a stopping condition is met. It returns a compiled `CompiledStateGraph` ready for invocation.

### Function Signature

```python
def create_react_agent(
    model: str | LanguageModelLike | Callable[[StateSchema, Runtime[ContextT]], BaseChatModel],
    tools: Sequence[BaseTool | Callable | dict[str, Any]] | ToolNode,
    *,
    prompt: Prompt | None = None,
    response_format: StructuredResponseSchema | tuple[str, StructuredResponseSchema] | None = None,
    pre_model_hook: RunnableLike | None = None,
    post_model_hook: RunnableLike | None = None,
    state_schema: StateSchemaType | None = None,
    context_schema: type[Any] | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    interrupt_before: list[str] | None = None,
    interrupt_after: list[str] | None = None,
    debug: bool = False,
    version: Literal["v1", "v2"] = "v2",
    name: str | None = None,
) -> CompiledStateGraph
```

### Parameters

#### `model` (required)

The language model for the agent. Supports both static and dynamic model selection.

**Static Model:**
- A chat model instance (e.g., `ChatOpenAI("gpt-4")`)
- A string identifier (e.g., `"openai:gpt-4"`, `"anthropic:claude-3-7-sonnet-latest"`)

**Dynamic Model:**
- A callable with signature `(state, runtime) -> BaseChatModel`
- Returns different models based on runtime context
- Supports async with coroutines

```python
# Static model - simple string
graph = create_react_agent("openai:gpt-4", tools)

# Static model - instance
from langchain_openai import ChatOpenAI
model = ChatOpenAI(model="gpt-4", temperature=0)
graph = create_react_agent(model, tools)

# Dynamic model - context-based selection
from dataclasses import dataclass

@dataclass
class ModelContext:
    model_name: str = "gpt-3.5-turbo"

gpt4 = ChatOpenAI(model="gpt-4")
gpt35 = ChatOpenAI(model="gpt-3.5-turbo")

def select_model(state: AgentState, runtime: Runtime[ModelContext]) -> ChatOpenAI:
    model_name = runtime.context.model_name
    model = gpt4 if model_name == "gpt-4" else gpt35
    return model.bind_tools(tools)  # Must bind tools!

graph = create_react_agent(select_model, tools, context_schema=ModelContext)
```

**Important Notes:**
- If using a pre-bound model (with `.bind_tools()`), the bound tools must match the `tools` parameter
- Dynamic models must return models with tools bound via `.bind_tools()`
- String syntax requires `langchain` package: `pip install langchain`

#### `tools` (required)

A list of tools or a `ToolNode` instance.

```python
from langchain_core.tools import tool

@tool
def search(query: str) -> str:
    """Search the web for information."""
    return f"Results for: {query}"

@tool
def calculator(expression: str) -> str:
    """Evaluate a mathematical expression."""
    return str(eval(expression))

# Pass as list
graph = create_react_agent(model, tools=[search, calculator])

# Or create ToolNode with custom configuration
tool_node = ToolNode([search, calculator], handle_tool_errors=True)
graph = create_react_agent(model, tools=tool_node)

# Empty tools list creates LLM-only node
graph = create_react_agent(model, tools=[])
```

#### `prompt`

Optional prompt for the LLM. Can be a string, `SystemMessage`, callable, or `Runnable`.

```python
# Simple string prompt
graph = create_react_agent(
    model,
    tools,
    prompt="You are a helpful assistant that uses tools to answer questions."
)

# SystemMessage
from langchain_core.messages import SystemMessage
graph = create_react_agent(
    model,
    tools,
    prompt=SystemMessage(content="You are an expert researcher.")
)

# Callable - access full state
def dynamic_prompt(state: AgentState) -> list:
    message_count = len(state["messages"])
    return [
        SystemMessage(content=f"You are a helpful assistant. Messages so far: {message_count}"),
        *state["messages"]
    ]

graph = create_react_agent(model, tools, prompt=dynamic_prompt)

# Runnable
from langchain_core.runnables import RunnableLambda
prompt_runnable = RunnableLambda(lambda state: [
    SystemMessage(content="Custom prompt"),
    *state["messages"]
])
graph = create_react_agent(model, tools, prompt=prompt_runnable)
```

#### `response_format`

Optional schema for structured output. Requires model to support `.with_structured_output()`.

**Note:** The graph makes a **separate LLM call** after the agent loop to generate the structured response.

```python
from pydantic import BaseModel

class Answer(BaseModel):
    answer: str
    confidence: float
    sources: list[str]

# Basic usage
graph = create_react_agent(
    model,
    tools,
    response_format=Answer
)

# With custom prompt for structured output
graph = create_react_agent(
    model,
    tools,
    response_format=("Provide a detailed answer with sources", Answer)
)

# Access structured response
result = graph.invoke({"messages": [("user", "What is the capital of France?")]})
structured_answer = result["structured_response"]  # Answer instance
```

#### `pre_model_hook`

Optional node to add **before** the agent node. Useful for message management (trimming, summarization).

The hook must return a state update with **at least one** of:
- `messages`: Updates state messages (should overwrite with `RemoveMessage`)
- `llm_input_messages`: Temporary messages for LLM input (doesn't update state)

```python
from langchain_core.messages import RemoveMessage, trim_messages

# Message trimming
def trim_hook(state: AgentState):
    messages = state["messages"]
    trimmed = trim_messages(
        messages,
        max_tokens=1000,
        strategy="last",
        token_counter=ChatOpenAI(model="gpt-4"),
    )
    return {
        "messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *trimmed]
    }

graph = create_react_agent(model, tools, pre_model_hook=trim_hook)

# Use llm_input_messages to avoid updating state
def context_hook(state: AgentState):
    # Don't modify state, just change LLM input
    return {
        "llm_input_messages": [
            SystemMessage(content="Additional context"),
            *state["messages"]
        ]
    }

graph = create_react_agent(model, tools, pre_model_hook=context_hook)
```

**Important:**
- If returning `messages`, you should **overwrite** using `RemoveMessage(id=REMOVE_ALL_MESSAGES)`
- Either `messages` or `llm_input_messages` must be provided
- Only available in version `v2`

#### `post_model_hook`

Optional node to add **after** the agent node. Useful for validation, guardrails, or human-in-the-loop.

```python
def validate_hook(state: AgentState):
    last_message = state["messages"][-1]

    # Check for inappropriate content
    if "inappropriate" in last_message.content.lower():
        return {
            "messages": [AIMessage(content="I cannot help with that request.")]
        }

    return {}  # No changes

graph = create_react_agent(model, tools, post_model_hook=validate_hook)

# Human-in-the-loop example
from langgraph.types import interrupt

def approval_hook(state: AgentState):
    last_message = state["messages"][-1]

    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        # Request human approval for tool calls
        response = interrupt({"tool_calls": last_message.tool_calls})

        if not response.get("approved"):
            return {
                "messages": [AIMessage(content="Action cancelled by user.")]
            }

    return {}

graph = create_react_agent(model, tools, post_model_hook=approval_hook)
```

**Note:** Only available with `version="v2"`.

#### `state_schema`

Optional custom state schema. Must have `messages` and `remaining_steps` keys.

```python
from typing import Annotated
from typing_extensions import TypedDict
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages
from langgraph.managed import RemainingSteps

class CustomState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    remaining_steps: RemainingSteps
    user_id: str  # Custom field
    session_data: dict

graph = create_react_agent(
    model,
    tools,
    state_schema=CustomState
)

# Invoke with custom state
result = graph.invoke({
    "messages": [("user", "Hello")],
    "user_id": "123",
    "session_data": {"preference": "detailed"}
})
```

**Note:** `remaining_steps` limits the number of agent iterations. When it drops below 2 and tool calls are present, the agent returns "Sorry, need more steps to process this request."

#### `context_schema`

Optional schema for runtime context. Enables passing context that's available throughout the graph.

```python
from dataclasses import dataclass

@dataclass
class UserContext:
    user_id: str
    permissions: list[str]
    preferences: dict

graph = create_react_agent(
    model,
    tools,
    context_schema=UserContext
)

# Pass context at runtime
context = UserContext(
    user_id="user123",
    permissions=["read", "write"],
    preferences={"language": "en"}
)

result = graph.invoke(
    {"messages": [("user", "Hello")]},
    config={"context": context}
)
```

#### `checkpointer` and `store`

Optional persistence for state and data.

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.store.memory import InMemoryStore

# Checkpointer - saves conversation state
checkpointer = MemorySaver()
graph = create_react_agent(model, tools, checkpointer=checkpointer)

# Use threads for multiple conversations
config1 = {"configurable": {"thread_id": "conversation-1"}}
config2 = {"configurable": {"thread_id": "conversation-2"}}

graph.invoke({"messages": [("user", "Hi")]}, config=config1)
graph.invoke({"messages": [("user", "Hello")]}, config=config2)

# Store - persistent cross-thread storage
store = InMemoryStore()
graph = create_react_agent(model, tools, store=store)

# Tools can access store via InjectedStore
```

#### `interrupt_before` and `interrupt_after`

Optional lists of node names to interrupt execution.

```python
# Interrupt before agent node (before LLM call)
graph = create_react_agent(
    model,
    tools,
    interrupt_before=["agent"]
)

# Interrupt after tools node (after tool execution)
graph = create_react_agent(
    model,
    tools,
    interrupt_after=["tools"]
)

# Use with checkpointer for human-in-the-loop
checkpointer = MemorySaver()
graph = create_react_agent(
    model,
    tools,
    checkpointer=checkpointer,
    interrupt_before=["tools"]  # Pause before executing tools
)

config = {"configurable": {"thread_id": "1"}}

# First run - stops before tools
result = graph.invoke({"messages": [("user", "Search for information")]}, config)

# Review and approve, then continue
result = graph.invoke(None, config)  # Resume execution
```

Valid node names: `"agent"`, `"tools"`, `"pre_model_hook"`, `"post_model_hook"`, `"generate_structured_response"`

#### `version`

Determines tool execution strategy.

- `"v1"`: Tool node processes all tool calls in a single message (parallel execution within node)
- `"v2"`: Tool calls distributed across multiple node instances using `Send` API

```python
# v1 - simpler, all tools in one node call
graph = create_react_agent(model, tools, version="v1")

# v2 - more flexible, better for human-in-the-loop
graph = create_react_agent(model, tools, version="v2")  # Default
```

**Default:** `"v2"`

#### `name`

Optional name for the compiled graph. Useful when adding as a subgraph.

```python
# Create named agent
customer_support_agent = create_react_agent(
    model,
    tools,
    name="customer_support"
)

# Use as subgraph
from langgraph.graph import StateGraph

main_graph = StateGraph(State)
main_graph.add_node("support", customer_support_agent)
```

### Graph Structure

The ReAct agent graph has the following structure:

```
┌─────────────────────┐
│   pre_model_hook    │ (optional)
│  (message trimming) │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│       agent         │
│  (LLM with tools)   │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   post_model_hook   │ (optional)
│   (validation)      │
└──────────┬──────────┘
           │
      ┌────┴────┐
      │         │
  tool calls?   │ no tool calls
      │         │
      ▼         ▼
  ┌─────┐   ┌─────┐
  │tools│   │ END │
  └──┬──┘   └─────┘
     │
     │ (loops back)
     ▼
  [agent]
```

### Complete Examples

#### Basic ReAct Agent

```python
from langchain_anthropic import ChatAnthropic
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def get_weather(location: str) -> str:
    """Get the weather for a location."""
    # Placeholder implementation
    return f"The weather in {location} is sunny, 72°F"

@tool
def search_web(query: str) -> str:
    """Search the web for information."""
    return f"Search results for: {query}"

model = ChatAnthropic(model="claude-3-7-sonnet-latest")
graph = create_react_agent(model, tools=[get_weather, search_web])

# Run the agent
result = graph.invoke({
    "messages": [("user", "What's the weather in San Francisco?")]
})

print(result["messages"][-1].content)
```

#### Agent with Structured Output

```python
from pydantic import BaseModel, Field

class ResearchSummary(BaseModel):
    """Summary of research findings."""
    main_findings: list[str] = Field(description="Key findings from research")
    confidence: float = Field(description="Confidence level 0-1")
    sources: list[str] = Field(description="Sources consulted")

graph = create_react_agent(
    model,
    tools=[search_web],
    prompt="You are a research assistant. Search for information and provide a detailed summary.",
    response_format=ResearchSummary
)

result = graph.invoke({
    "messages": [("user", "Research the latest developments in quantum computing")]
})

# Access structured output
summary: ResearchSummary = result["structured_response"]
print(f"Findings: {summary.main_findings}")
print(f"Confidence: {summary.confidence}")
```

#### Agent with Custom State and Memory Management

```python
from typing import Annotated
from typing_extensions import TypedDict
from langchain_core.messages import BaseMessage, RemoveMessage, trim_messages
from langgraph.graph.message import add_messages, REMOVE_ALL_MESSAGES
from langgraph.managed import RemainingSteps
from langgraph.checkpoint.memory import MemorySaver

class CustomAgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    remaining_steps: RemainingSteps
    user_context: dict
    total_tool_calls: int

def message_trimmer(state: CustomAgentState):
    """Keep only last 10 messages."""
    messages = state["messages"]
    if len(messages) > 10:
        # Keep last 10
        kept_messages = messages[-10:]
        return {
            "messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *kept_messages]
        }
    return {}

checkpointer = MemorySaver()

graph = create_react_agent(
    model,
    tools=[get_weather, search_web],
    state_schema=CustomAgentState,
    pre_model_hook=message_trimmer,
    checkpointer=checkpointer,
    prompt="You are a helpful assistant with conversation memory."
)

# Use with custom state
config = {"configurable": {"thread_id": "user-123"}}

result = graph.invoke({
    "messages": [("user", "What's the weather in NYC?")],
    "user_context": {"location": "New York", "timezone": "EST"},
    "total_tool_calls": 0
}, config)

# Continue conversation
result = graph.invoke({
    "messages": [("user", "How about tomorrow?")]
}, config)
```

#### Agent with Human-in-the-Loop

```python
from langgraph.types import interrupt
from langgraph.checkpoint.memory import MemorySaver

def human_approval_hook(state: AgentState):
    """Request human approval before executing tools."""
    last_message = state["messages"][-1]

    if hasattr(last_message, "tool_calls") and last_message.tool_calls:
        # Show tool calls to human
        tool_info = [
            f"{call['name']}({call['args']})"
            for call in last_message.tool_calls
        ]

        # Request approval (this pauses execution)
        approval = interrupt({
            "message": "Approve these tool calls?",
            "tools": tool_info
        })

        # If not approved, return early
        if not approval.get("approved", False):
            return {
                "messages": [AIMessage(content="Action cancelled by user.")]
            }

    return {}

checkpointer = MemorySaver()
graph = create_react_agent(
    model,
    tools=[get_weather, search_web],
    post_model_hook=human_approval_hook,
    checkpointer=checkpointer
)

config = {"configurable": {"thread_id": "interactive-session"}}

# First call - will pause for approval
result = graph.invoke({
    "messages": [("user", "Check weather in Paris and search for tourist attractions")]
}, config)

# Human reviews and approves (simulated)
# In real system, this would come from UI
approval_state = {"approved": True}

# Resume with approval
result = graph.invoke(approval_state, config)
```

---

## ToolNode

The `ToolNode` class executes tools in LangGraph workflows. It handles parallel execution, error handling, state injection, and more.

### Constructor

```python
def __init__(
    self,
    tools: Sequence[BaseTool | Callable],
    *,
    name: str = "tools",
    tags: list[str] | None = None,
    handle_tool_errors: bool | str | Callable[..., str] | type[Exception] | tuple[type[Exception], ...] = _default_handle_tool_errors,
    messages_key: str = "messages",
    wrap_tool_call: ToolCallWrapper | None = None,
    awrap_tool_call: AsyncToolCallWrapper | None = None,
)
```

### Parameters

#### `tools` (required)

Sequence of tools to execute. Can be `BaseTool` instances or plain functions.

```python
from langchain_core.tools import tool
from langgraph.prebuilt import ToolNode

@tool
def add(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b

# Create from tools
tool_node = ToolNode([add, multiply])

# Functions are automatically converted to tools
def divide(a: float, b: float) -> float:
    """Divide a by b."""
    return a / b

tool_node = ToolNode([add, multiply, divide])
```

#### `handle_tool_errors`

Configuration for error handling. Supports multiple strategies:

```python
# Catch all errors, return default error message
tool_node = ToolNode(tools, handle_tool_errors=True)

# Custom error message string
tool_node = ToolNode(tools, handle_tool_errors="An error occurred. Please try again.")

# Catch specific exception types
tool_node = ToolNode(tools, handle_tool_errors=ValueError)
tool_node = ToolNode(tools, handle_tool_errors=(ValueError, TypeError))

# Custom error handler function
def custom_handler(e: Exception) -> str:
    if isinstance(e, ValueError):
        return "Invalid input provided"
    return f"Error: {str(e)}"

tool_node = ToolNode(tools, handle_tool_errors=custom_handler)

# Type-aware error handler
def typed_handler(e: ValueError | TypeError) -> str:
    if isinstance(e, ValueError):
        return "Value error occurred"
    return "Type error occurred"

tool_node = ToolNode(tools, handle_tool_errors=typed_handler)

# Disable error handling (errors propagate)
tool_node = ToolNode(tools, handle_tool_errors=False)
```

**Default behavior:**
- Catches `ToolInvocationError` (invalid arguments from model)
- Returns descriptive error message
- Re-raises other exceptions

#### `messages_key`

The state key containing messages (default: `"messages"`).

```python
from typing_extensions import TypedDict
from typing import Annotated
from langchain_core.messages import BaseMessage
from langgraph.graph.message import add_messages

class CustomState(TypedDict):
    chat_history: Annotated[list[BaseMessage], add_messages]

tool_node = ToolNode(tools, messages_key="chat_history")

# Use with custom state
result = tool_node.invoke({
    "chat_history": [AIMessage("", tool_calls=[...])]
})
```

#### `wrap_tool_call` and `awrap_tool_call`

Interceptors for tool execution. Enable retries, caching, logging, etc.

```python
from langgraph.prebuilt.tool_node import ToolCallRequest, ToolCallWrapper

# Logging wrapper
def log_wrapper(
    request: ToolCallRequest,
    execute: Callable[[ToolCallRequest], ToolMessage | Command]
) -> ToolMessage | Command:
    print(f"Executing tool: {request.tool_call['name']}")
    result = execute(request)
    print(f"Tool completed: {request.tool_call['name']}")
    return result

tool_node = ToolNode(tools, wrap_tool_call=log_wrapper)

# Retry wrapper
def retry_wrapper(request, execute):
    max_retries = 3
    for attempt in range(max_retries):
        try:
            result = execute(request)
            if isinstance(result, ToolMessage) and result.status != "error":
                return result
        except Exception as e:
            if attempt == max_retries - 1:
                raise
    return result

tool_node = ToolNode(tools, wrap_tool_call=retry_wrapper)

# Caching wrapper
cache = {}

def cache_wrapper(request, execute):
    # Create cache key
    key = (request.tool_call['name'], str(request.tool_call['args']))

    if key in cache:
        # Return cached result
        return ToolMessage(
            content=cache[key],
            tool_call_id=request.tool_call['id']
        )

    # Execute and cache
    result = execute(request)
    if isinstance(result, ToolMessage):
        cache[key] = result.content

    return result

tool_node = ToolNode(tools, wrap_tool_call=cache_wrapper)
```

### Input Formats

`ToolNode` accepts three input formats:

```python
from langchain_core.messages import AIMessage, ToolCall

# 1. Graph state with messages key
state_input = {
    "messages": [
        AIMessage(content="", tool_calls=[
            {"name": "add", "args": {"a": 1, "b": 2}, "id": "1", "type": "tool_call"}
        ])
    ]
}

# 2. List of messages
list_input = [
    AIMessage(content="", tool_calls=[
        {"name": "add", "args": {"a": 1, "b": 2}, "id": "1", "type": "tool_call"}
    ])
]

# 3. Direct tool calls (for testing/programmatic use)
tool_calls_input = [
    {"name": "add", "args": {"a": 1, "b": 2}, "id": "1", "type": "tool_call"},
    {"name": "multiply", "args": {"a": 3, "b": 4}, "id": "2", "type": "tool_call"}
]

# All three work
result1 = tool_node.invoke(state_input)
result2 = tool_node.invoke(list_input)
result3 = tool_node.invoke(tool_calls_input)
```

### Output Formats

Output format matches input format:

```python
# Dict input → Dict output
result = tool_node.invoke({"messages": [ai_message]})
# result = {"messages": [ToolMessage(...), ToolMessage(...)]}

# List input → List output
result = tool_node.invoke([ai_message])
# result = [ToolMessage(...), ToolMessage(...)]
```

### State Injection

Tools can access graph state using `InjectedState`:

```python
from typing import Annotated
from langgraph.prebuilt import InjectedState, ToolNode

@tool
def context_aware_tool(
    query: str,
    state: Annotated[dict, InjectedState]
) -> str:
    """Tool that accesses full graph state."""
    message_count = len(state["messages"])
    return f"Processing '{query}' (context: {message_count} messages)"

@tool
def specific_field_tool(
    query: str,
    user_id: Annotated[str, InjectedState("user_id")]
) -> str:
    """Tool that accesses specific state field."""
    return f"Query from user {user_id}: {query}"

tool_node = ToolNode([context_aware_tool, specific_field_tool])

# State is automatically injected
result = tool_node.invoke({
    "messages": [AIMessage("", tool_calls=[...])],
    "user_id": "user123"
})
```

### Store Injection

Tools can access persistent storage using `InjectedStore`:

```python
from typing import Annotated, Any
from langgraph.prebuilt import InjectedStore, ToolNode
from langgraph.store.memory import InMemoryStore

@tool
def save_preference(
    key: str,
    value: str,
    store: Annotated[Any, InjectedStore()]
) -> str:
    """Save user preference."""
    store.put(("preferences",), key, value)
    return f"Saved {key} = {value}"

@tool
def get_preference(
    key: str,
    store: Annotated[Any, InjectedStore()]
) -> str:
    """Get user preference."""
    result = store.get(("preferences",), key)
    return result.value if result else "Not found"

store = InMemoryStore()
tool_node = ToolNode([save_preference, get_preference])

# Use with graph
from langgraph.graph import StateGraph

graph = StateGraph(State)
graph.add_node("tools", tool_node)
compiled = graph.compile(store=store)  # Store injected automatically
```

### Runtime Injection

Tools can access runtime context using `ToolRuntime`:

```python
from langgraph.prebuilt import ToolRuntime, ToolNode

@tool
def runtime_aware_tool(query: str, runtime: ToolRuntime) -> str:
    """Tool that accesses runtime context."""

    # Access state
    messages = runtime.state["messages"]

    # Access tool call ID
    call_id = runtime.tool_call_id

    # Access config
    run_id = runtime.config.get("run_id")

    # Access context (if provided)
    user_id = runtime.context.get("user_id") if runtime.context else None

    # Access store
    if runtime.store:
        runtime.store.put(("logs",), call_id, {"query": query})

    # Stream output
    runtime.stream_writer.write(f"Processing: {query}")

    return f"Processed query with ID {call_id}"

tool_node = ToolNode([runtime_aware_tool])
```

### Error Handling Examples

```python
@tool
def risky_tool(value: int) -> str:
    """Tool that might fail."""
    if value < 0:
        raise ValueError("Value must be positive")
    if value > 100:
        raise TypeError("Value too large")
    return f"Processed: {value}"

# Catch all errors
tool_node = ToolNode([risky_tool], handle_tool_errors=True)

# Catch specific errors
tool_node = ToolNode([risky_tool], handle_tool_errors=ValueError)

# Custom error messages
def error_handler(e: ValueError | TypeError) -> str:
    if isinstance(e, ValueError):
        return "Please provide a positive value"
    return "Please provide a smaller value"

tool_node = ToolNode([risky_tool], handle_tool_errors=error_handler)

# Test error handling
result = tool_node.invoke([
    {"name": "risky_tool", "args": {"value": -5}, "id": "1", "type": "tool_call"}
])
# result[0].status == "error"
# result[0].content == "Please provide a positive value"
```

---

## ValidationNode

The `ValidationNode` validates tool calls against Pydantic schemas **without executing them**. Useful for:

- Structured data extraction
- Schema validation in multi-turn conversations
- Ensuring tool calls conform to complex schemas

**Note:** `ValidationNode` is deprecated. Consider using `create_agent` with custom tool error handling instead.

### Constructor

```python
def __init__(
    self,
    schemas: Sequence[BaseTool | type[BaseModel] | Callable],
    *,
    format_error: Callable[[BaseException, ToolCall, type[BaseModel]], str] | None = None,
    name: str = "validation",
    tags: list[str] | None = None,
)
```

### Basic Usage

```python
from pydantic import BaseModel, field_validator
from langgraph.prebuilt import ValidationNode
from langchain_core.messages import AIMessage

class SelectNumber(BaseModel):
    """Select a number."""
    a: int

    @field_validator("a")
    def a_must_be_37(cls, v):
        if v != 37:
            raise ValueError("Only 37 is allowed")
        return v

validation_node = ValidationNode([SelectNumber])

# Valid call
result = validation_node.invoke({
    "messages": [
        AIMessage("", tool_calls=[
            {"name": "SelectNumber", "args": {"a": 37}, "id": "1", "type": "tool_call"}
        ])
    ]
})
# result["messages"][0].content == '{"a": 37}'

# Invalid call
result = validation_node.invoke({
    "messages": [
        AIMessage("", tool_calls=[
            {"name": "SelectNumber", "args": {"a": 42}, "id": "1", "type": "tool_call"}
        ])
    ]
})
# result["messages"][0].additional_kwargs["is_error"] == True
# result["messages"][0].content contains validation error
```

### Re-prompting Pattern

Use validation with conditional edges to re-prompt on errors:

```python
from langgraph.graph import StateGraph, START, END
from langchain_anthropic import ChatAnthropic
from typing import Literal

class State(TypedDict):
    messages: Annotated[list, add_messages]

builder = StateGraph(State)

# Add LLM node
llm = ChatAnthropic(model="claude-3-5-haiku-latest").bind_tools([SelectNumber])
builder.add_node("model", llm)

# Add validation node
builder.add_node("validation", ValidationNode([SelectNumber]))

builder.add_edge(START, "model")

# Route to validation if tool calls present
def should_validate(state: list) -> Literal["validation", "__end__"]:
    if state["messages"][-1].tool_calls:
        return "validation"
    return END

builder.add_conditional_edges("model", should_validate)

# Re-prompt on validation errors
def should_reprompt(state: list) -> Literal["model", "__end__"]:
    for msg in reversed(state["messages"]):
        if msg.type == "ai":
            return END
        if msg.additional_kwargs.get("is_error"):
            return "model"  # Re-prompt
    return END

builder.add_conditional_edges("validation", should_reprompt)

graph = builder.compile()

# The graph will keep trying until valid
result = graph.invoke({
    "messages": [("user", "Select the number 37")]
})
```

### Custom Error Formatting

```python
def custom_error_formatter(
    error: BaseException,
    call: ToolCall,
    schema: type[BaseModel]
) -> str:
    """Custom error message."""
    return f"Validation failed for {call['name']}: {str(error)}. Please fix and retry."

validation_node = ValidationNode(
    [SelectNumber],
    format_error=custom_error_formatter
)
```

---

## tools_condition

A utility function for conditional routing based on tool calls. Routes to `"tools"` if tool calls present, otherwise to `END`.

### Function Signature

```python
def tools_condition(
    state: list[AnyMessage] | dict[str, Any] | BaseModel,
    messages_key: str = "messages",
) -> Literal["tools", "__end__"]
```

### Basic Usage

```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode, tools_condition

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph = StateGraph(State)
graph.add_node("llm", call_model)
graph.add_node("tools", ToolNode([my_tool]))

# Use tools_condition for routing
graph.add_conditional_edges(
    "llm",
    tools_condition,  # Returns "tools" or "__end__"
    {
        "tools": "tools",
        "__end__": END
    }
)

graph.add_edge("tools", "llm")  # Loop back after tools
graph.add_edge(START, "llm")

compiled = graph.compile()
```

### Custom Messages Key

```python
class CustomState(TypedDict):
    chat_history: Annotated[list, add_messages]

def custom_condition(state):
    return tools_condition(state, messages_key="chat_history")

graph.add_conditional_edges(
    "llm",
    custom_condition,
    {"tools": "tools", "__end__": END}
)
```

---

## Human Interrupt Utilities

Utilities for human-in-the-loop workflows with the Agent Inbox pattern.

**Note:** These classes are deprecated. Use `langchain.agents.interrupt` instead.

### HumanInterruptConfig

Defines allowed actions for human interrupts.

```python
from langgraph.prebuilt.interrupt import HumanInterruptConfig

config = HumanInterruptConfig(
    allow_ignore=True,   # Can skip this step
    allow_respond=True,  # Can provide text feedback
    allow_edit=False,    # Cannot edit content
    allow_accept=True    # Can accept/approve
)
```

### ActionRequest

Represents a requested action.

```python
from langgraph.prebuilt.interrupt import ActionRequest

request = ActionRequest(
    action="run_command",
    args={"command": "ls", "flags": ["-la"]}
)
```

### HumanInterrupt

The interrupt request sent to humans.

```python
from langgraph.prebuilt.interrupt import HumanInterrupt, HumanInterruptConfig, ActionRequest
from langgraph.types import interrupt

def tool_approval_node(state):
    """Request approval before executing tool."""
    tool_call = state["messages"][-1].tool_calls[0]

    # Create interrupt request
    human_interrupt = HumanInterrupt(
        action_request=ActionRequest(
            action=tool_call['name'],
            args=tool_call['args']
        ),
        config=HumanInterruptConfig(
            allow_ignore=True,
            allow_respond=True,
            allow_edit=True,
            allow_accept=True
        ),
        description=f"Approve execution of {tool_call['name']}?"
    )

    # Send interrupt and get response
    response = interrupt([human_interrupt])[0]

    # Process response
    if response['type'] == 'accept':
        return {}  # Proceed
    elif response['type'] == 'ignore':
        return {"messages": [AIMessage(content="Skipped by user")]}
    elif response['type'] == 'response':
        return {"messages": [HumanMessage(content=response['args'])]}
    elif response['type'] == 'edit':
        # Use edited args
        edited_args = response['args']
        return {"edited_tool_call": edited_args}
```

### HumanResponse

The response from the human.

```python
from langgraph.prebuilt.interrupt import HumanResponse

# Response types:
accept_response = HumanResponse(type="accept", args=None)
ignore_response = HumanResponse(type="ignore", args=None)
text_response = HumanResponse(type="response", args="Please use a different approach")
edit_response = HumanResponse(
    type="edit",
    args=ActionRequest(action="run_command", args={"command": "ls -la"})
)
```

---

## Dependency Injection

### InjectedState

Inject graph state into tool parameters.

```python
from typing import Annotated
from langgraph.prebuilt import InjectedState

@tool
def full_state_tool(
    query: str,
    state: Annotated[dict, InjectedState]
) -> str:
    """Access full graph state."""
    return f"Messages: {len(state['messages'])}, Query: {query}"

@tool
def specific_field_tool(
    query: str,
    user_id: Annotated[str, InjectedState("user_id")]
) -> str:
    """Access specific state field."""
    return f"User {user_id} queried: {query}"

@tool
def multiple_fields_tool(
    query: str,
    user_id: Annotated[str, InjectedState("user_id")],
    messages: Annotated[list, InjectedState("messages")]
) -> str:
    """Access multiple state fields."""
    return f"User {user_id}, {len(messages)} messages, query: {query}"
```

**Important:**
- Injected arguments are excluded from the tool schema shown to the LLM
- Only the LLM-controlled arguments (`query` above) appear in the schema
- Injection happens automatically during execution

### InjectedStore

Inject persistent store into tool parameters.

```python
from typing import Annotated, Any
from langgraph.prebuilt import InjectedStore
from langgraph.store.memory import InMemoryStore

@tool
def save_data(
    key: str,
    value: str,
    store: Annotated[Any, InjectedStore()]
) -> str:
    """Save data to persistent store."""
    store.put(("app_data",), key, value)
    return f"Saved {key}"

@tool
def load_data(
    key: str,
    store: Annotated[Any, InjectedStore()]
) -> str:
    """Load data from persistent store."""
    result = store.get(("app_data",), key)
    return result.value if result else "Not found"

# Setup
store = InMemoryStore()
tool_node = ToolNode([save_data, load_data])

# Compile graph with store
graph = StateGraph(State)
graph.add_node("tools", tool_node)
compiled = graph.compile(store=store)
```

**Requirements:**
- Graph must be compiled with `store` parameter
- Requires `langchain-core >= 0.3.8`

### ToolRuntime

Access complete runtime context including state, config, store, and more.

```python
from langgraph.prebuilt import ToolRuntime

@tool
def advanced_tool(query: str, runtime: ToolRuntime) -> str:
    """Tool with full runtime access."""

    # Access state
    messages = runtime.state["messages"]
    user_id = runtime.state.get("user_id")

    # Access tool call ID
    call_id = runtime.tool_call_id

    # Access config
    thread_id = runtime.config["configurable"].get("thread_id")

    # Access context (if provided to graph)
    context_data = runtime.context if runtime.context else {}

    # Access store
    if runtime.store:
        # Log this tool call
        runtime.store.put(
            ("tool_logs", thread_id),
            call_id,
            {"tool": "advanced_tool", "query": query}
        )

    # Stream intermediate results
    runtime.stream_writer.write(f"Processing query: {query}")

    return f"Processed '{query}' for user {user_id}"

# No Annotated needed - just use `runtime: ToolRuntime`
tool_node = ToolNode([advanced_tool])
```

**ToolRuntime Attributes:**
- `state`: Current graph state
- `tool_call_id`: ID of the current tool call
- `config`: `RunnableConfig` for this execution
- `context`: Runtime context (from graph configuration)
- `store`: Persistent storage (if provided)
- `stream_writer`: For streaming output

---

## Complete Examples

### Multi-Agent System with ReAct Agents

```python
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langchain_core.messages import BaseMessage, HumanMessage
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import create_react_agent
from langchain_core.tools import tool

# Define tools for each agent
@tool
def search_database(query: str) -> str:
    """Search internal database."""
    return f"Database results for: {query}"

@tool
def call_api(endpoint: str) -> str:
    """Call external API."""
    return f"API response from {endpoint}"

@tool
def generate_report(data: str) -> str:
    """Generate formatted report."""
    return f"Report:\n{data}"

# Create specialized agents
research_agent = create_react_agent(
    ChatOpenAI(model="gpt-4"),
    tools=[search_database, call_api],
    prompt="You are a research specialist. Gather comprehensive information.",
    name="researcher"
)

report_agent = create_react_agent(
    ChatOpenAI(model="gpt-4"),
    tools=[generate_report],
    prompt="You are a report writer. Create clear, structured reports.",
    name="reporter"
)

# Supervisor state
class SupervisorState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]
    next_agent: str

# Supervisor decides which agent to use
def supervisor_node(state: SupervisorState):
    """Supervisor decides next step."""
    last_message = state["messages"][-1]

    if "research" in last_message.content.lower():
        return {"next_agent": "researcher"}
    elif "report" in last_message.content.lower():
        return {"next_agent": "reporter"}
    else:
        return {"next_agent": "end"}

# Build supervisor graph
supervisor = StateGraph(SupervisorState)
supervisor.add_node("supervisor", supervisor_node)
supervisor.add_node("researcher", research_agent)
supervisor.add_node("reporter", report_agent)

supervisor.add_edge(START, "supervisor")

def route_to_agent(state: SupervisorState) -> Literal["researcher", "reporter", "__end__"]:
    next_agent = state.get("next_agent", "end")
    if next_agent == "end":
        return END
    return next_agent

supervisor.add_conditional_edges(
    "supervisor",
    route_to_agent,
    {
        "researcher": "researcher",
        "reporter": "reporter",
        END: END
    }
)

supervisor.add_edge("researcher", "supervisor")
supervisor.add_edge("reporter", "supervisor")

multi_agent = supervisor.compile()

# Use the system
result = multi_agent.invoke({
    "messages": [HumanMessage(content="Research quantum computing and generate a report")]
})
```

### Custom Tool with All Injection Types

```python
from typing import Annotated, Any
from langgraph.prebuilt import InjectedState, InjectedStore, ToolRuntime, ToolNode
from langgraph.store.memory import InMemoryStore
from langchain_core.tools import tool

@tool
def comprehensive_tool(
    user_query: str,
    # Inject full state
    full_state: Annotated[dict, InjectedState],
    # Inject specific field
    user_id: Annotated[str, InjectedState("user_id")],
    # Inject store
    store: Annotated[Any, InjectedStore()],
    # Inject runtime
    runtime: ToolRuntime
) -> str:
    """Tool demonstrating all injection types."""

    # Use full state
    message_count = len(full_state["messages"])

    # Use specific field
    user_greeting = f"Hello, user {user_id}"

    # Use store for persistence
    # Get user's query history
    history = store.get(("user_history", user_id), "queries") or []
    history = history.value if hasattr(history, 'value') else history

    # Save this query
    store.put(("user_history", user_id), "queries", history + [user_query])

    # Use runtime for context
    thread_id = runtime.config["configurable"].get("thread_id", "unknown")

    # Stream progress
    runtime.stream_writer.write(f"Processing query for user {user_id}...")

    result = f"""
Query: {user_query}
User: {user_id}
Messages in conversation: {message_count}
Previous queries: {len(history)}
Thread: {thread_id}
Tool call ID: {runtime.tool_call_id}
"""

    return result.strip()

# Setup
store = InMemoryStore()
tool_node = ToolNode([comprehensive_tool])

# Use in graph
from langgraph.graph import StateGraph
from langgraph.managed import RemainingSteps

class AppState(TypedDict):
    messages: Annotated[list, add_messages]
    remaining_steps: RemainingSteps
    user_id: str

graph = StateGraph(AppState)
graph.add_node("tools", tool_node)
# ... add other nodes ...

compiled = graph.compile(
    store=store,
    checkpointer=MemorySaver()
)

# Invoke
config = {"configurable": {"thread_id": "session-123"}}
result = compiled.invoke({
    "messages": [AIMessage("", tool_calls=[{
        "name": "comprehensive_tool",
        "args": {"user_query": "What's the weather?"},
        "id": "call-1",
        "type": "tool_call"
    }])],
    "user_id": "user-456"
}, config)
```

### Advanced Error Handling and Retry Logic

```python
from langgraph.prebuilt import ToolNode, ToolCallRequest
from langchain_core.messages import ToolMessage
from typing import Callable

# Tool that sometimes fails
@tool
def unreliable_api(endpoint: str) -> str:
    """Call an unreliable API."""
    import random
    if random.random() < 0.3:  # 30% failure rate
        raise ConnectionError("API temporarily unavailable")
    return f"Success: {endpoint}"

# Retry wrapper with exponential backoff
import time

def retry_with_backoff(
    request: ToolCallRequest,
    execute: Callable[[ToolCallRequest], ToolMessage]
) -> ToolMessage:
    """Retry tool execution with exponential backoff."""
    max_retries = 3
    base_delay = 1

    for attempt in range(max_retries):
        try:
            result = execute(request)

            # Check if result is an error
            if isinstance(result, ToolMessage) and result.status == "error":
                if attempt < max_retries - 1:
                    delay = base_delay * (2 ** attempt)
                    time.sleep(delay)
                    continue

            return result

        except ConnectionError as e:
            if attempt < max_retries - 1:
                delay = base_delay * (2 ** attempt)
                print(f"Attempt {attempt + 1} failed, retrying in {delay}s...")
                time.sleep(delay)
            else:
                # Final attempt failed
                return ToolMessage(
                    content=f"Failed after {max_retries} attempts: {str(e)}",
                    tool_call_id=request.tool_call['id'],
                    name=request.tool_call['name'],
                    status="error"
                )

    # Should never reach here
    return result

# Custom error handler for specific errors
def smart_error_handler(e: Exception) -> str:
    """Provide helpful error messages."""
    if isinstance(e, ConnectionError):
        return "The API is temporarily unavailable. Please try again in a few moments."
    elif isinstance(e, ValueError):
        return "Invalid input provided. Please check your arguments."
    elif isinstance(e, TimeoutError):
        return "The request timed out. The API may be experiencing high load."
    else:
        return f"An unexpected error occurred: {type(e).__name__}"

# Create ToolNode with retry and error handling
tool_node = ToolNode(
    [unreliable_api],
    wrap_tool_call=retry_with_backoff,
    handle_tool_errors=smart_error_handler
)

# Test it
result = tool_node.invoke([{
    "name": "unreliable_api",
    "args": {"endpoint": "/data"},
    "id": "test-1",
    "type": "tool_call"
}])

print(result[0].content)
```

### Streaming Agent with Progress Updates

```python
from langgraph.prebuilt import create_react_agent, ToolRuntime
from langchain_core.tools import tool

@tool
def long_running_task(task_description: str, runtime: ToolRuntime) -> str:
    """Execute a long-running task with progress updates."""
    import time

    steps = [
        "Initializing...",
        "Loading data...",
        "Processing...",
        "Generating results...",
        "Finalizing..."
    ]

    for i, step in enumerate(steps):
        # Stream progress update
        runtime.stream_writer.write(f"Step {i+1}/5: {step}")
        time.sleep(0.5)  # Simulate work

    return f"Completed: {task_description}"

graph = create_react_agent(
    ChatOpenAI(model="gpt-4"),
    tools=[long_running_task],
    prompt="Execute tasks and provide progress updates."
)

# Stream execution
for chunk in graph.stream(
    {"messages": [("user", "Process the quarterly reports")]},
    stream_mode="updates"
):
    print(chunk)
```

---

## Best Practices

### 1. Error Handling

Always configure appropriate error handling for tools:

```python
# For production: catch errors gracefully
tool_node = ToolNode(tools, handle_tool_errors=True)

# For development: let errors propagate for debugging
tool_node = ToolNode(tools, handle_tool_errors=False)

# For specific errors: handle only what you expect
tool_node = ToolNode(tools, handle_tool_errors=(ValueError, TypeError))
```

### 2. State Management

Use `InjectedState` to keep tools context-aware:

```python
# Good: Inject only what you need
@tool
def my_tool(
    query: str,
    user_id: Annotated[str, InjectedState("user_id")]
) -> str:
    return f"Query for {user_id}: {query}"

# Avoid: Accessing state from global scope
# (makes testing harder and violates separation of concerns)
```

### 3. Message Trimming

Implement `pre_model_hook` to manage message history:

```python
from langchain_core.messages import trim_messages, RemoveMessage, REMOVE_ALL_MESSAGES

def trim_hook(state):
    messages = trim_messages(
        state["messages"],
        max_tokens=2000,
        strategy="last",
        token_counter=model
    )
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES), *messages]}

graph = create_react_agent(model, tools, pre_model_hook=trim_hook)
```

### 4. Persistent Storage

Use checkpointers and stores appropriately:

```python
# Checkpointer: for conversation state
checkpointer = MemorySaver()  # or SqliteSaver, PostgresSaver

# Store: for cross-conversation data
store = InMemoryStore()  # or other store implementations

graph = create_react_agent(
    model,
    tools,
    checkpointer=checkpointer,  # Saves conversation state
    store=store                  # Persists data across conversations
)
```

### 5. Testing Tools

Test tools independently before using in agents:

```python
# Test tool directly
result = my_tool.invoke({"query": "test", "user_id": "123"})
assert result == expected

# Test with ToolNode
tool_node = ToolNode([my_tool])
result = tool_node.invoke([
    {"name": "my_tool", "args": {"query": "test"}, "id": "1", "type": "tool_call"}
])
assert result[0].content == expected
```

---

## Migration Notes

### From v1 to v2

Version 2 introduces several improvements:

```python
# v1: Single tool node processes all calls
graph_v1 = create_react_agent(model, tools, version="v1")

# v2: Tool calls distributed for better parallelization
graph_v2 = create_react_agent(model, tools, version="v2")

# v2 supports post_model_hook
graph_v2 = create_react_agent(
    model,
    tools,
    version="v2",
    post_model_hook=validation_hook  # Not available in v1
)
```

### Deprecation Warnings

Some components are deprecated:

```python
# Deprecated: ValidationNode
# Use create_agent with custom error handling instead
validation_node = ValidationNode([schema])  # Will show deprecation warning

# Deprecated: HumanInterrupt classes in langgraph.prebuilt
# Use langchain.agents.interrupt instead
from langchain.agents.interrupt import HumanInterrupt  # Preferred
```

---

## Summary

The `langgraph.prebuilt` module provides production-ready components for building agent workflows:

- **create_react_agent()**: High-level API for ReAct agents with extensive customization
- **ToolNode**: Robust tool execution with error handling and injection
- **ValidationNode**: Schema validation for structured extraction
- **tools_condition**: Standard routing logic for tool-calling workflows
- **Injection patterns**: Access state, store, and runtime in tools
- **Human-in-the-loop**: Interrupt patterns for human approval

These components handle common patterns so you can focus on building unique agent behaviors.
