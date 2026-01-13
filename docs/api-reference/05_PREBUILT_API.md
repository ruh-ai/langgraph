# LangGraph Prebuilt API Reference

This document provides a comprehensive reference for the `langgraph.prebuilt` module, which exposes a higher-level API for creating and executing agents and tools.

## Table of Contents

- [Agent Creation](#agent-creation)
  - [create_react_agent](#create_react_agent)
- [Tool Execution](#tool-execution)
  - [ToolNode](#toolnode)
  - [tools_condition](#tools_condition)
- [Validation](#validation)
  - [ValidationNode](#validationnode)
- [State and Store Injection](#state-and-store-injection)
  - [InjectedState](#injectedstate)
  - [InjectedStore](#injectedstore)
  - [ToolRuntime](#toolruntime)
  - [ToolCallRequest](#toolcallrequest)
- [Tool Call Wrappers](#tool-call-wrappers)
  - [ToolCallWrapper](#toolcallwrapper)
  - [AsyncToolCallWrapper](#asynctoolcallwrapper)
- [Human Interaction](#human-interaction)
  - [HumanInterrupt](#humaninterrupt)
  - [HumanResponse](#humanresponse)
  - [ActionRequest](#actionrequest)
  - [HumanInterruptConfig](#humaninterruptconfig)
- [Utility Functions](#utility-functions)
  - [msg_content_output](#msg_content_output)
- [Exceptions](#exceptions)
  - [ToolInvocationError](#toolinvocationerror)

---

## Agent Creation

### create_react_agent

**Signature:**
```python
@deprecated(
    "create_react_agent has been moved to `langchain.agents`. "
    "Please update your import to `from langchain.agents import create_agent`.",
    category=LangGraphDeprecatedSinceV10,
)
def create_react_agent(
    model: str
    | LanguageModelLike
    | Callable[[StateSchema, Runtime[ContextT]], BaseChatModel]
    | Callable[[StateSchema, Runtime[ContextT]], Awaitable[BaseChatModel]]
    | Callable[[StateSchema, Runtime[ContextT]], Runnable[LanguageModelInput, BaseMessage]]
    | Callable[[StateSchema, Runtime[ContextT]], Awaitable[Runnable[LanguageModelInput, BaseMessage]]],
    tools: Sequence[BaseTool | Callable | dict[str, Any]] | ToolNode,
    *,
    prompt: Prompt | None = None,
    response_format: StructuredResponseSchema
    | tuple[str, StructuredResponseSchema]
    | None = None,
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
    **deprecated_kwargs: Any,
) -> CompiledStateGraph
```

**Description:**

Creates an agent graph that calls tools in a loop until a stopping condition is met. This is the main entry point for creating ReAct-style agents with LangGraph.

The agent follows this pattern:
1. The "agent" node calls the language model with the messages list (after applying the prompt)
2. If the resulting AIMessage contains `tool_calls`, the graph calls the "tools" node
3. The "tools" node executes the tools and adds responses as `ToolMessage` objects
4. The agent node calls the language model again
5. The process repeats until no more `tool_calls` are present

**Deprecation Notice:**

This function has been moved to `langchain.agents`. Please update your import to `from langchain.agents import create_agent`.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| model | `str \| LanguageModelLike \| Callable` | Yes | - | The language model for the agent. Supports:<br>- **Static model**: A chat model instance (e.g., `ChatOpenAI`) or string identifier (e.g., `"openai:gpt-4"`)<br>- **Dynamic model**: A callable with signature `(state, runtime) -> BaseChatModel` that returns different models based on runtime context. Coroutines are also supported for async model selection. |
| tools | `Sequence[BaseTool \| Callable \| dict[str, Any]] \| ToolNode` | Yes | - | A list of tools or a `ToolNode` instance. If an empty list is provided, the agent will consist of a single LLM node without tool calling. |
| prompt | `Prompt \| None` | No | `None` | Optional prompt for the LLM. Can be:<br>- `str`: Converted to `SystemMessage` and prepended to messages<br>- `SystemMessage`: Prepended to messages<br>- `Callable`: Function taking state and returning `LanguageModelInput`<br>- `Runnable`: Runnable taking state and returning `LanguageModelInput` |
| response_format | `StructuredResponseSchema \| tuple[str, StructuredResponseSchema] \| None` | No | `None` | Optional schema for the final agent output. If provided, output will be formatted to match the schema and returned in the `structured_response` state key. Can be:<br>- OpenAI function/tool schema<br>- JSON Schema<br>- TypedDict class<br>- Pydantic class<br>- Tuple `(prompt, schema)` where prompt is used with the model for structured generation<br><br>Requires model to support `.with_structured_output` |
| pre_model_hook | `RunnableLike \| None` | No | `None` | Optional node to add before the "agent" node. Useful for managing long message histories (e.g., message trimming, summarization). Must take current graph state and return a state update with at least one of `messages` or `llm_input_messages` keys. |
| post_model_hook | `RunnableLike \| None` | No | `None` | Optional node to add after the "agent" node. Useful for implementing human-in-the-loop, guardrails, validation, or other post-processing. Only available with `version="v2"`. |
| state_schema | `StateSchemaType \| None` | No | `None` | Optional state schema that defines graph state. Must have `messages` and `remaining_steps` keys. Defaults to `AgentState`. |
| context_schema | `type[Any] \| None` | No | `None` | Optional schema for runtime context. |
| checkpointer | `Checkpointer \| None` | No | `None` | Optional checkpoint saver object for persisting state (e.g., as chat memory) for a single thread. |
| store | `BaseStore \| None` | No | `None` | Optional store object for persisting data across multiple threads. |
| interrupt_before | `list[str] \| None` | No | `None` | Optional list of node names to interrupt before. Should be `"agent"` or `"tools"`. Useful for adding user confirmation before taking an action. |
| interrupt_after | `list[str] \| None` | No | `None` | Optional list of node names to interrupt after. Should be `"agent"` or `"tools"`. Useful for returning directly or running additional processing on an output. |
| debug | `bool` | No | `False` | Flag indicating whether to enable debug mode. |
| version | `Literal["v1", "v2"]` | No | `"v2"` | Determines the version of the graph to create:<br>- `"v1"`: The tool node processes a single message. All tool calls in the message are executed in parallel within the tool node.<br>- `"v2"`: The tool node processes a tool call. Tool calls are distributed across multiple instances of the tool node using the Send API. |
| name | `str \| None` | No | `None` | Optional name for the `CompiledStateGraph`. This name will be automatically used when adding ReAct agent graph to another graph as a subgraph node. |

**Returns:**

| Type | Description |
|------|-------------|
| `CompiledStateGraph` | A compiled LangChain `Runnable` that can be used for chat interactions. |

**Raises:**

| Exception | When |
|-----------|------|
| `ValueError` | If invalid version is specified, or if state_schema is missing required keys, or if chat history validation fails |
| `TypeError` | If model is not a ChatModel or RunnableBinding |
| `ImportError` | If using string model syntax without langchain installed |

**Example:**
```python
from langgraph.prebuilt import create_react_agent

def check_weather(location: str) -> str:
    '''Return the weather forecast for the specified location.'''
    return f"It's always sunny in {location}"

graph = create_react_agent(
    "anthropic:claude-3-7-sonnet-latest",
    tools=[check_weather],
    prompt="You are a helpful assistant",
)
inputs = {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
for chunk in graph.stream(inputs, stream_mode="updates"):
    print(chunk)
```

**Dynamic Model Example:**
```python
from dataclasses import dataclass

@dataclass
class ModelContext:
    model_name: str = "gpt-3.5-turbo"

# Instantiate models globally
gpt4_model = ChatOpenAI(model="gpt-4")
gpt35_model = ChatOpenAI(model="gpt-3.5-turbo")

def select_model(state: AgentState, runtime: Runtime[ModelContext]) -> ChatOpenAI:
    model_name = runtime.context.model_name
    model = gpt4_model if model_name == "gpt-4" else gpt35_model
    return model.bind_tools(tools)

graph = create_react_agent(
    select_model,
    tools=[my_tool],
    context_schema=ModelContext,
)
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/chat_agent_executor.py:273-991`

---

## Tool Execution

### ToolNode

**Signature:**
```python
class ToolNode(RunnableCallable):
    def __init__(
        self,
        tools: Sequence[BaseTool | Callable],
        *,
        name: str = "tools",
        tags: list[str] | None = None,
        handle_tool_errors: bool
        | str
        | Callable[..., str]
        | type[Exception]
        | tuple[type[Exception], ...] = _default_handle_tool_errors,
        messages_key: str = "messages",
        wrap_tool_call: ToolCallWrapper | None = None,
        awrap_tool_call: AsyncToolCallWrapper | None = None,
    ) -> None
```

**Description:**

A node for executing tools in LangGraph workflows. Handles tool execution patterns including function calls, state injection, persistent storage, and control flow. Manages parallel execution and error handling.

The ToolNode supports three input formats:
1. **Graph state with `messages` key**: Common representation for agentic workflows (supports custom messages key via `messages_key` parameter)
2. **Message List**: `[AIMessage(..., tool_calls=[...])]` - List of messages with tool calls in the last AIMessage
3. **Direct Tool Calls**: `[{"name": "tool", "args": {...}, "id": "1", "type": "tool_call"}]` - Bypasses message parsing for direct tool execution

Output format depends on input type:
- **For Regular tools**:
  - Dict input → `{"messages": [ToolMessage(...)]}`
  - List input → `[ToolMessage(...)]`
- **For Command tools**:
  - Returns `[Command(...)]` or mixed list with regular tool outputs

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| tools | `Sequence[BaseTool \| Callable]` | Yes | - | A sequence of tools that can be invoked by this node. Supports BaseTool instances and plain functions (automatically converted to tools with inferred schemas). |
| name | `str` | No | `"tools"` | The name identifier for this node in the graph. Used for debugging and visualization. |
| tags | `list[str] \| None` | No | `None` | Optional metadata tags to associate with the node for filtering and organization. |
| handle_tool_errors | `bool \| str \| Callable \| type[Exception] \| tuple[type[Exception], ...]` | No | `_default_handle_tool_errors` | Configuration for error handling during tool execution. Supports multiple strategies:<br>- `True`: Catch all errors and return a `ToolMessage` with the default error template<br>- `str`: Catch all errors and return a `ToolMessage` with this custom error message<br>- `type[Exception]`: Only catch exceptions of the specified type<br>- `tuple[type[Exception], ...]`: Only catch exceptions of the specified types<br>- `Callable[..., str]`: Catch exceptions matching the callable's signature and return the string result<br>- `False`: Disable error handling entirely, allowing exceptions to propagate<br><br>Defaults to a callable that catches tool invocation errors (invalid arguments) and returns a descriptive error message. |
| messages_key | `str` | No | `"messages"` | The key in the state dictionary that contains the message list. This same key will be used for the output `ToolMessage` objects. Allows custom state schemas with different message field names. |
| wrap_tool_call | `ToolCallWrapper \| None` | No | `None` | Sync wrapper function to intercept tool execution. Receives `ToolCallRequest` and execute callable, returns `ToolMessage` or `Command`. Enables retries, caching, request modification, and control flow. |
| awrap_tool_call | `AsyncToolCallWrapper \| None` | No | `None` | Async wrapper function to intercept tool execution. If not provided, falls back to `wrap_tool_call` for async execution. |

**Returns:**

N/A (Constructor)

**Raises:**

| Exception | When |
|-----------|------|
| N/A | Constructor does not raise exceptions directly |

**Properties:**

##### tools_by_name

**Signature:**
```python
@property
def tools_by_name(self) -> dict[str, BaseTool]
```

**Description:** Mapping from tool name to BaseTool instance.

**Returns:** Dictionary mapping tool names to their corresponding BaseTool instances.

**Methods:**

##### invoke / ainvoke

Inherited from `RunnableCallable`. Executes the tool node synchronously or asynchronously.

**Signature:**
```python
def invoke(
    self,
    input: list[AnyMessage] | dict[str, Any] | BaseModel,
    config: RunnableConfig | None = None,
) -> Any

async def ainvoke(
    self,
    input: list[AnyMessage] | dict[str, Any] | BaseModel,
    config: RunnableConfig | None = None,
) -> Any
```

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| input | `list[AnyMessage] \| dict[str, Any] \| BaseModel` | Yes | - | Input in one of the supported formats (graph state, message list, or direct tool calls) |
| config | `RunnableConfig \| None` | No | `None` | Configuration for the execution |

**Returns:**

| Type | Description |
|------|-------------|
| `Any` | Tool execution results (format depends on input type and tool behavior) |

**Example:**

Basic usage:
```python
from langchain.tools import ToolNode
from langchain_core.tools import tool

@tool
def calculator(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

tool_node = ToolNode([calculator])
```

State injection:
```python
from typing_extensions import Annotated
from langchain.tools import InjectedState

@tool
def context_tool(query: str, state: Annotated[dict, InjectedState]) -> str:
    """Some tool that uses state."""
    return f"Query: {query}, Messages: {len(state['messages'])}"

tool_node = ToolNode([context_tool])
```

Error handling:
```python
def handle_errors(e: ValueError) -> str:
    return "Invalid input provided"

tool_node = ToolNode([my_tool], handle_tool_errors=handle_errors)
```

Tool call wrapping:
```python
def retry_wrapper(request: ToolCallRequest, execute: Callable) -> ToolMessage | Command:
    for attempt in range(3):
        try:
            result = execute(request)
            if is_valid(result):
                return result
        except Exception:
            if attempt == 2:
                raise
    return result

tool_node = ToolNode([my_tool], wrap_tool_call=retry_wrapper)
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:610-1434`

---

### tools_condition

**Signature:**
```python
def tools_condition(
    state: list[AnyMessage] | dict[str, Any] | BaseModel,
    messages_key: str = "messages",
) -> Literal["tools", "__end__"]
```

**Description:**

Conditional routing function for tool-calling workflows. This utility function implements the standard conditional logic for ReAct-style agents: if the last `AIMessage` contains tool calls, route to the tool execution node; otherwise, end the workflow.

The function handles multiple state formats commonly used in LangGraph applications, making it flexible for different graph designs while maintaining consistent behavior.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| state | `list[AnyMessage] \| dict[str, Any] \| BaseModel` | Yes | - | The current graph state to examine for tool calls. Supported formats:<br>- Dictionary containing a messages key (for `StateGraph`)<br>- List of messages<br>- `BaseModel` instance with a messages attribute |
| messages_key | `str` | No | `"messages"` | The key or attribute name containing the message list in the state. This allows customization for graphs using different state schemas. |

**Returns:**

| Type | Description |
|------|-------------|
| `Literal["tools", "__end__"]` | Either `"tools"` if tool calls are present in the last `AIMessage`, or `"__end__"` to terminate the workflow. |

**Raises:**

| Exception | When |
|-----------|------|
| `ValueError` | If no messages can be found in the provided state format |

**Example:**

Basic usage in a ReAct agent:
```python
from langgraph.graph import StateGraph
from langchain.tools import ToolNode
from langchain.tools.tool_node import tools_condition
from typing_extensions import TypedDict


class State(TypedDict):
    messages: list


graph = StateGraph(State)
graph.add_node("llm", call_model)
graph.add_node("tools", ToolNode([my_tool]))
graph.add_conditional_edges(
    "llm",
    tools_condition,  # Routes to "tools" or "__end__"
    {"tools": "tools", "__end__": "__end__"},
)
```

Custom messages key:
```python
def custom_condition(state):
    return tools_condition(state, messages_key="chat_history")
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:1436-1513`

---

## Validation

### ValidationNode

**Signature:**
```python
@deprecated(
    "ValidationNode is deprecated. Please use `create_agent` from `langchain.agents` with custom tool error handling.",
    category=LangGraphDeprecatedSinceV10,
)
class ValidationNode(RunnableCallable):
    def __init__(
        self,
        schemas: Sequence[BaseTool | type[BaseModel] | Callable],
        *,
        format_error: Callable[[BaseException, ToolCall, type[BaseModel]], str]
        | None = None,
        name: str = "validation",
        tags: list[str] | None = None,
    ) -> None
```

**Description:**

A node that validates all tool requests from the last `AIMessage`. It can be used in `StateGraph` with a `'messages'` key.

This node does not actually **run** the tools, it only validates the tool calls, which is useful for extraction and other use cases where you need to generate structured output that conforms to a complex schema without losing the original messages and tool IDs (for use in multi-turn conversations).

**Deprecation Notice:**

ValidationNode is deprecated. Please use `create_agent` from `langchain.agents` with custom tool error handling.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| schemas | `Sequence[BaseTool \| type[BaseModel] \| Callable]` | Yes | - | A list of schemas to validate the tool calls with. These can be:<br>- A pydantic BaseModel class<br>- A BaseTool instance (the args_schema will be used)<br>- A function (a schema will be created from the function signature) |
| format_error | `Callable[[BaseException, ToolCall, type[BaseModel]], str] \| None` | No | `None` | A function that takes an exception, a ToolCall, and a schema and returns a formatted error string. By default, it returns the exception repr and a message to respond after fixing validation errors. |
| name | `str` | No | `"validation"` | The name of the node. |
| tags | `list[str] \| None` | No | `None` | A list of tags to add to the node. |

**Returns:**

| Type | Description |
|------|-------------|
| `Union[Dict[str, List[ToolMessage]], Sequence[ToolMessage]]` | A list of `ToolMessage` objects with the validated content or error messages. |

**Raises:**

| Exception | When |
|-----------|------|
| `ValueError` | If a tool does not have an args_schema defined, or if validation node only works with tools that have a pydantic BaseModel args_schema, or if no message found in input, or if last message is not an AIMessage |

**Properties:**

##### schemas_by_name

**Description:** Dictionary mapping schema names to their corresponding BaseModel classes.

**Type:** `dict[str, type[BaseModel]]`

**Example:**

Re-prompting the model to generate a valid response:
```python
from typing import Literal, Annotated
from typing_extensions import TypedDict

from langchain_anthropic import ChatAnthropic
from pydantic import BaseModel, field_validator

from langgraph.graph import END, START, StateGraph
from langgraph.prebuilt import ValidationNode
from langgraph.graph.message import add_messages

class SelectNumber(BaseModel):
    a: int

    @field_validator("a")
    def a_must_be_meaningful(cls, v):
        if v != 37:
            raise ValueError("Only 37 is allowed")
        return v

builder = StateGraph(Annotated[list, add_messages])
llm = ChatAnthropic(model="claude-3-5-haiku-latest").bind_tools([SelectNumber])
builder.add_node("model", llm)
builder.add_node("validation", ValidationNode([SelectNumber]))
builder.add_edge(START, "model")

def should_validate(state: list) -> Literal["validation", "__end__"]:
    if state[-1].tool_calls:
        return "validation"
    return END

builder.add_conditional_edges("model", should_validate)

def should_reprompt(state: list) -> Literal["model", "__end__"]:
    for msg in state[::-1]:
        # None of the tool calls were errors
        if msg.type == "ai":
            return END
        if msg.additional_kwargs.get("is_error"):
            return "model"
    return END

builder.add_conditional_edges("validation", should_reprompt)

graph = builder.compile()
res = graph.invoke(("user", "Select a number, any number"))
# Show the retry logic
for msg in res:
    msg.pretty_print()
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_validator.py:47-221`

---

## State and Store Injection

### InjectedState

**Signature:**
```python
class InjectedState(InjectedToolArg):
    def __init__(self, field: str | None = None) -> None
```

**Description:**

Annotation for injecting graph state into tool arguments. This annotation enables tools to access graph state without exposing state management details to the language model. Tools annotated with `InjectedState` receive state data automatically during execution while remaining invisible to the model's tool-calling interface.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| field | `str \| None` | No | `None` | Optional key to extract from the state dictionary. If `None`, the entire state is injected. If specified, only that field's value is injected. |

**Returns:**

N/A (Constructor)

**Raises:**

| Exception | When |
|-----------|------|
| N/A | Constructor does not raise exceptions directly |

**Example:**

```python
from typing import List
from typing_extensions import Annotated, TypedDict

from langchain_core.messages import BaseMessage, AIMessage
from langchain.tools import InjectedState, ToolNode, tool


class AgentState(TypedDict):
    messages: List[BaseMessage]
    foo: str


@tool
def state_tool(x: int, state: Annotated[dict, InjectedState]) -> str:
    '''Do something with state.'''
    if len(state["messages"]) > 2:
        return state["foo"] + str(x)
    else:
        return "not enough messages"


@tool
def foo_tool(x: int, foo: Annotated[str, InjectedState("foo")]) -> str:
    '''Do something else with state.'''
    return foo + str(x + 1)


node = ToolNode([state_tool, foo_tool])

tool_call1 = {"name": "state_tool", "args": {"x": 1}, "id": "1", "type": "tool_call"}
tool_call2 = {"name": "foo_tool", "args": {"x": 1}, "id": "2", "type": "tool_call"}
state = {
    "messages": [AIMessage("", tool_calls=[tool_call1, tool_call2])],
    "foo": "bar",
}
node.invoke(state)
# Returns:
# [
#     ToolMessage(content="not enough messages", name="state_tool", tool_call_id="1"),
#     ToolMessage(content="bar2", name="foo_tool", tool_call_id="2"),
# ]
```

**Notes:**

- `InjectedState` arguments are automatically excluded from tool schemas presented to language models
- `ToolNode` handles the injection process during execution
- Tools can mix regular arguments (controlled by the model) with injected arguments (controlled by the system)
- State injection occurs after the model generates tool calls but before tool execution

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:1576-1649`

---

### InjectedStore

**Signature:**
```python
class InjectedStore(InjectedToolArg):
    pass
```

**Description:**

Annotation for injecting persistent store into tool arguments. This annotation enables tools to access LangGraph's persistent storage system without exposing storage details to the language model. Tools annotated with `InjectedStore` receive the store instance automatically during execution while remaining invisible to the model's tool-calling interface.

The store provides persistent, cross-session data storage that tools can use for maintaining context, user preferences, or any other data that needs to persist beyond individual workflow executions.

**Parameters:**

N/A (No constructor parameters)

**Returns:**

N/A (Annotation class)

**Raises:**

| Exception | When |
|-----------|------|
| N/A | Annotation does not raise exceptions directly |

**Example:**

```python
from typing_extensions import Annotated
from langgraph.store.memory import InMemoryStore
from langchain.tools import InjectedStore, ToolNode, tool

@tool
def save_preference(
    key: str,
    value: str,
    store: Annotated[Any, InjectedStore()]
) -> str:
    """Save user preference to persistent storage."""
    store.put(("preferences",), key, value)
    return f"Saved {key} = {value}"

@tool
def get_preference(
    key: str,
    store: Annotated[Any, InjectedStore()]
) -> str:
    """Retrieve user preference from persistent storage."""
    result = store.get(("preferences",), key)
    return result.value if result else "Not found"
```

Usage with `ToolNode` and graph compilation:
```python
from langgraph.graph import StateGraph
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
tool_node = ToolNode([save_preference, get_preference])

graph = StateGraph(State)
graph.add_node("tools", tool_node)
compiled_graph = graph.compile(store=store)  # Store is injected automatically
```

Cross-session persistence:
```python
# First session
result1 = graph.invoke({"messages": [HumanMessage("Save my favorite color as blue")]})

# Later session - data persists
result2 = graph.invoke({"messages": [HumanMessage("What's my favorite color?")]})
```

**Notes:**

- Requires `langchain-core >= 0.3.8`
- `InjectedStore` arguments are automatically excluded from tool schemas presented to language models
- The store instance is automatically injected by `ToolNode` during execution
- Tools can access namespaced storage using the store's get/put methods
- Store injection requires the graph to be compiled with a store instance
- Multiple tools can share the same store instance for data consistency

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:1652-1724`

---

### ToolRuntime

**Signature:**
```python
@dataclass
class ToolRuntime(_DirectlyInjectedToolArg, Generic[ContextT, StateT]):
    state: StateT
    context: ContextT
    config: RunnableConfig
    stream_writer: StreamWriter
    tool_call_id: str | None
    store: BaseStore | None
```

**Description:**

Runtime context automatically injected into tools. When a tool function has a parameter named `tool_runtime` with type hint `ToolRuntime`, the tool execution system will automatically inject an instance containing state, tool_call_id, config, context, store, and stream_writer.

No `Annotated` wrapper is needed - just use `runtime: ToolRuntime` as a parameter.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| state | `StateT` | The current graph state |
| context | `ContextT` | Runtime context (from langgraph `Runtime`) |
| config | `RunnableConfig` | RunnableConfig for the current execution |
| stream_writer | `StreamWriter` | StreamWriter for streaming output (from langgraph `Runtime`) |
| tool_call_id | `str \| None` | The ID of the current tool call |
| store | `BaseStore \| None` | BaseStore instance for persistent storage (from langgraph `Runtime`) |

**Returns:**

N/A (Dataclass)

**Raises:**

| Exception | When |
|-----------|------|
| N/A | Dataclass does not raise exceptions directly |

**Example:**

```python
from langchain_core.tools import tool
from langchain.tools import ToolRuntime

@tool
def my_tool(x: int, runtime: ToolRuntime) -> str:
    """Tool that accesses runtime context."""
    # Access state
    messages = runtime.state["messages"]

    # Access tool_call_id
    print(f"Tool call ID: {runtime.tool_call_id}")

    # Access config
    print(f"Run ID: {runtime.config.get('run_id')}")

    # Access runtime context
    user_id = runtime.context.get("user_id")

    # Access store
    runtime.store.put(("metrics",), "count", 1)

    # Stream output
    runtime.stream_writer.write("Processing...")

    return f"Processed {x}"
```

**Notes:**

This is a marker class used for type checking and detection. The actual runtime object will be constructed during tool execution.

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:1516-1574`

---

### ToolCallRequest

**Signature:**
```python
@dataclass
class ToolCallRequest:
    tool_call: ToolCall
    tool: BaseTool | None
    state: Any
    runtime: ToolRuntime
```

**Description:**

Tool execution request passed to tool call interceptors. This dataclass encapsulates all the information needed to execute a tool call, including the tool call details, the tool instance, the current state, and the runtime context.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| tool_call | `ToolCall` | Tool call dict with name, args, and id from model output |
| tool | `BaseTool \| None` | BaseTool instance to be invoked, or None if tool is not registered with the `ToolNode`. When tool is `None`, interceptors can handle the request without validation. |
| state | `Any` | Agent state (`dict`, `list`, or `BaseModel`) |
| runtime | `ToolRuntime` | LangGraph runtime context (optional, `None` if outside graph) |

**Methods:**

##### override

**Signature:**
```python
def override(
    self, **overrides: Unpack[_ToolCallRequestOverrides]
) -> ToolCallRequest
```

**Description:**

Replace the request with a new request with the given overrides. Returns a new `ToolCallRequest` instance with the specified attributes replaced. This follows an immutable pattern, leaving the original request unchanged.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| **overrides | `Unpack[_ToolCallRequestOverrides]` | No | - | Keyword arguments for attributes to override. Supported keys:<br>- `tool_call`: Tool call dict with name, args, and id |

**Returns:**

| Type | Description |
|------|-------------|
| `ToolCallRequest` | New ToolCallRequest instance with specified overrides applied |

**Example:**

```python
# Modify tool call arguments without mutating original
modified_call = {**request.tool_call, "args": {"value": 10}}
new_request = request.override(tool_call=modified_call)

# Override multiple attributes
new_request = request.override(tool_call=modified_call)
```

**Notes:**

Direct attribute assignment is deprecated. Use the `override()` method instead to create a new instance with modified values.

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:126-190`

---

## Tool Call Wrappers

### ToolCallWrapper

**Signature:**
```python
ToolCallWrapper = Callable[
    [ToolCallRequest, Callable[[ToolCallRequest], ToolMessage | Command]],
    ToolMessage | Command,
]
```

**Description:**

Wrapper for tool call execution with multi-call support. The wrapper receives a request and an execute callable, and can call execute multiple times for retry logic, with potentially modified requests each time.

**Parameters:**

The wrapper function receives:

| Name | Type | Description |
|------|------|-------------|
| request | `ToolCallRequest` | ToolCallRequest with tool_call, tool, state, and runtime |
| execute | `Callable[[ToolCallRequest], ToolMessage \| Command]` | Callable to execute the tool (CAN BE CALLED MULTIPLE TIMES) |

**Returns:**

| Type | Description |
|------|-------------|
| `ToolMessage \| Command` | The final result |

**Raises:**

| Exception | When |
|-----------|------|
| Any | Exceptions from the wrapped execution or wrapper logic |

**Example:**

Passthrough (execute once):
```python
def handler(request, execute):
    return execute(request)
```

Modify request before execution:
```python
def handler(request, execute):
    modified_call = {**request.tool_call, "args": {**request.tool_call["args"], "value": request.tool_call["args"]["value"] * 2}}
    modified_request = request.override(tool_call=modified_call)
    return execute(modified_request)
```

Retry on error (execute multiple times):
```python
def handler(request, execute):
    for attempt in range(3):
        try:
            result = execute(request)
            if is_valid(result):
                return result
        except Exception:
            if attempt == 2:
                raise
    return result
```

Conditional retry based on response:
```python
def handler(request, execute):
    for attempt in range(3):
        result = execute(request)
        if isinstance(result, ToolMessage) and result.status != "error":
            return result
        if attempt < 2:
            continue
        return result
```

Cache/short-circuit without calling execute:
```python
def handler(request, execute):
    if cached := get_cache(request):
        return ToolMessage(content=cached, tool_call_id=request.tool_call["id"])
    result = execute(request)
    save_cache(request, result)
    return result
```

**Notes:**

The execute callable can be invoked multiple times for retry logic, with potentially modified requests each time. Each call to execute is independent and stateless.

When implementing middleware for `create_agent`, use `AgentMiddleware.wrap_tool_call` which provides properly typed state parameter for better type safety.

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:192-267`

---

### AsyncToolCallWrapper

**Signature:**
```python
AsyncToolCallWrapper = Callable[
    [ToolCallRequest, Callable[[ToolCallRequest], Awaitable[ToolMessage | Command]]],
    Awaitable[ToolMessage | Command],
]
```

**Description:**

Async wrapper for tool call execution with multi-call support. Same as `ToolCallWrapper` but for asynchronous execution.

**Parameters:**

The wrapper function receives:

| Name | Type | Description |
|------|------|-------------|
| request | `ToolCallRequest` | ToolCallRequest with tool_call, tool, state, and runtime |
| execute | `Callable[[ToolCallRequest], Awaitable[ToolMessage \| Command]]` | Async callable to execute the tool (CAN BE CALLED MULTIPLE TIMES) |

**Returns:**

| Type | Description |
|------|-------------|
| `Awaitable[ToolMessage \| Command]` | The final result (awaitable) |

**Raises:**

| Exception | When |
|-----------|------|
| Any | Exceptions from the wrapped execution or wrapper logic |

**Example:**

```python
async def async_handler(request, execute):
    for attempt in range(3):
        try:
            result = await execute(request)
            if is_valid(result):
                return result
        except Exception:
            if attempt == 2:
                raise
    return result
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:269-273`

---

## Human Interaction

### HumanInterrupt

**Signature:**
```python
@deprecated(
    "HumanInterrupt has been moved to `langchain.agents.interrupt`. Please update your import to `from langchain.agents.interrupt import HumanInterrupt`.",
    category=LangGraphDeprecatedSinceV10,
)
class HumanInterrupt(TypedDict):
    action_request: ActionRequest
    config: HumanInterruptConfig
    description: str | None
```

**Description:**

Represents an interrupt triggered by the graph that requires human intervention. This is passed to the `interrupt` function when execution is paused for human input.

**Deprecation Notice:**

HumanInterrupt has been moved to `langchain.agents.interrupt`. Please update your import to `from langchain.agents.interrupt import HumanInterrupt`.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| action_request | `ActionRequest` | The specific action being requested from the human |
| config | `HumanInterruptConfig` | Configuration defining what actions are allowed |
| description | `str \| None` | Optional detailed description of what input is needed |

**Returns:**

N/A (TypedDict)

**Raises:**

| Exception | When |
|-----------|------|
| N/A | TypedDict does not raise exceptions directly |

**Example:**

```python
# Extract a tool call from the state and create an interrupt request
request = HumanInterrupt(
    action_request=ActionRequest(
        action="run_command",  # The action being requested
        args={"command": "ls", "args": ["-l"]}  # Arguments for the action
    ),
    config=HumanInterruptConfig(
        allow_ignore=True,    # Allow skipping this step
        allow_respond=True,   # Allow text feedback
        allow_edit=False,     # Don't allow editing
        allow_accept=True     # Allow direct acceptance
    ),
    description="Please review the command before execution"
)
# Send the interrupt request and get the response
response = interrupt([request])[0]
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/interrupt.py:51-85`

---

### HumanResponse

**Signature:**
```python
class HumanResponse(TypedDict):
    type: Literal["accept", "ignore", "response", "edit"]
    args: None | str | ActionRequest
```

**Description:**

The response provided by a human to an interrupt, which is returned when graph execution resumes.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| type | `Literal["accept", "ignore", "response", "edit"]` | The type of response:<br>- `'accept'`: Approves the current state without changes<br>- `'ignore'`: Skips/ignores the current step<br>- `'response'`: Provides text feedback or instructions<br>- `'edit'`: Modifies the current state/content |
| args | `None \| str \| ActionRequest` | The response payload:<br>- `None`: For ignore/accept actions<br>- `str`: For text responses<br>- `ActionRequest`: For edit actions with updated content |

**Returns:**

N/A (TypedDict)

**Raises:**

| Exception | When |
|-----------|------|
| N/A | TypedDict does not raise exceptions directly |

**Example:**

```python
# Accept action
response = HumanResponse(type="accept", args=None)

# Ignore action
response = HumanResponse(type="ignore", args=None)

# Provide text response
response = HumanResponse(type="response", args="Please use a different approach")

# Edit action
response = HumanResponse(
    type="edit",
    args=ActionRequest(action="run_command", args={"command": "ls", "args": ["-la"]})
)
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/interrupt.py:87-105`

---

### ActionRequest

**Signature:**
```python
@deprecated(
    "ActionRequest has been moved to `langchain.agents.interrupt`. Please update your import to `from langchain.agents.interrupt import ActionRequest`.",
    category=LangGraphDeprecatedSinceV10,
)
class ActionRequest(TypedDict):
    action: str
    args: dict
```

**Description:**

Represents a request for human action within the graph execution. Contains the action type and any associated arguments needed for the action.

**Deprecation Notice:**

ActionRequest has been moved to `langchain.agents.interrupt`. Please update your import to `from langchain.agents.interrupt import ActionRequest`.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| action | `str` | The type or name of action being requested (e.g., `"Approve XYZ action"`) |
| args | `dict` | Key-value pairs of arguments needed for the action |

**Returns:**

N/A (TypedDict)

**Raises:**

| Exception | When |
|-----------|------|
| N/A | TypedDict does not raise exceptions directly |

**Example:**

```python
action = ActionRequest(
    action="run_command",
    args={"command": "ls", "args": ["-l"]}
)
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/interrupt.py:33-44`

---

### HumanInterruptConfig

**Signature:**
```python
@deprecated(
    "HumanInterruptConfig has been moved to `langchain.agents.interrupt`. Please update your import to `from langchain.agents.interrupt import HumanInterruptConfig`.",
    category=LangGraphDeprecatedSinceV10,
)
class HumanInterruptConfig(TypedDict):
    allow_ignore: bool
    allow_respond: bool
    allow_edit: bool
    allow_accept: bool
```

**Description:**

Configuration that defines what actions are allowed for a human interrupt. This controls the available interaction options when the graph is paused for human input.

**Deprecation Notice:**

HumanInterruptConfig has been moved to `langchain.agents.interrupt`. Please update your import to `from langchain.agents.interrupt import HumanInterruptConfig`.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| allow_ignore | `bool` | Whether the human can choose to ignore/skip the current step |
| allow_respond | `bool` | Whether the human can provide a text response/feedback |
| allow_edit | `bool` | Whether the human can edit the provided content/state |
| allow_accept | `bool` | Whether the human can accept/approve the current state |

**Returns:**

N/A (TypedDict)

**Raises:**

| Exception | When |
|-----------|------|
| N/A | TypedDict does not raise exceptions directly |

**Example:**

```python
config = HumanInterruptConfig(
    allow_ignore=True,
    allow_respond=True,
    allow_edit=False,
    allow_accept=True
)
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/interrupt.py:11-26`

---

## Utility Functions

### msg_content_output

**Signature:**
```python
def msg_content_output(output: Any) -> str | list[dict]
```

**Description:**

Convert tool output to `ToolMessage` content format. Handles `str`, `list[dict]` (content blocks), and arbitrary objects by attempting JSON serialization with fallback to str().

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| output | `Any` | Yes | - | Tool execution output of any type |

**Returns:**

| Type | Description |
|------|-------------|
| `str \| list[dict]` | String or list of content blocks suitable for `ToolMessage.content` |

**Raises:**

| Exception | When |
|-----------|------|
| N/A | Does not raise exceptions (uses fallback to str()) |

**Example:**

```python
# String output
result = msg_content_output("Hello")  # Returns: "Hello"

# Dict output (JSON serialized)
result = msg_content_output({"key": "value"})  # Returns: '{"key": "value"}'

# List of content blocks
result = msg_content_output([{"type": "text", "text": "Hello"}])  # Returns: [{"type": "text", "text": "Hello"}]

# Complex object (JSON serialized or str())
result = msg_content_output(MyObject())  # Returns JSON or str representation
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:299-326`

---

## Exceptions

### ToolInvocationError

**Signature:**
```python
class ToolInvocationError(ToolException):
    def __init__(
        self,
        tool_name: str,
        source: ValidationError,
        tool_kwargs: dict[str, Any],
        filtered_errors: list[ErrorDetails] | None = None,
    ) -> None
```

**Description:**

An error occurred while invoking a tool due to invalid arguments. This exception is only raised when invoking a tool using the `ToolNode`.

**Parameters:**

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| tool_name | `str` | Yes | - | The name of the tool that failed |
| source | `ValidationError` | Yes | - | The exception that occurred |
| tool_kwargs | `dict[str, Any]` | Yes | - | The keyword arguments that were passed to the tool |
| filtered_errors | `list[ErrorDetails] \| None` | No | `None` | Optional list of filtered validation errors excluding injected arguments |

**Returns:**

N/A (Exception constructor)

**Raises:**

| Exception | When |
|-----------|------|
| N/A | This is the exception being raised |

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| message | `str` | Formatted error message with tool name, kwargs, and error details |
| tool_name | `str` | The name of the tool that failed |
| tool_kwargs | `dict[str, Any]` | The keyword arguments that were passed to the tool |
| source | `ValidationError` | The original validation exception |
| filtered_errors | `list[ErrorDetails] \| None` | Optional list of filtered validation errors |

**Example:**

```python
try:
    tool_node.invoke(state)
except ToolInvocationError as e:
    print(f"Tool {e.tool_name} failed with args {e.tool_kwargs}")
    print(f"Error: {e.message}")
```

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:329-370`

---

## Additional Types and Internal Classes

### ToolCallWithContext

**Signature:**
```python
class ToolCallWithContext(TypedDict):
    tool_call: ToolCall
    __type: Literal["tool_call_with_context"]
    state: Any
```

**Description:**

ToolCall with additional context for graph state. This is an internal data structure meant to help the `ToolNode` accept tool calls with additional context (e.g. state) when dispatched using the Send API.

The Send API is used in create_agent to distribute tool calls in parallel and support human-in-the-loop workflows where graph execution may be paused for an indefinite time.

**Attributes:**

| Name | Type | Description |
|------|------|-------------|
| tool_call | `ToolCall` | The tool call dict with name, args, and id |
| __type | `Literal["tool_call_with_context"]` | Type to parameterize the payload |
| state | `Any` | The state provided as additional context |

**Notes:**

This is an internal data structure. Users typically don't need to create instances of this directly.

**Source:** `/home/user/langgraph/libs/prebuilt/langgraph/prebuilt/tool_node.py:276-297`

---

## Summary

The `langgraph.prebuilt` module provides high-level abstractions for building agent-based applications:

- **Agent Creation**: `create_react_agent()` provides a complete ReAct agent with tool calling support
- **Tool Execution**: `ToolNode` handles parallel tool execution with error handling and state injection
- **Validation**: `ValidationNode` validates tool calls without executing them (useful for extraction)
- **State Management**: `InjectedState`, `InjectedStore`, and `ToolRuntime` enable tools to access graph state and persistent storage
- **Routing**: `tools_condition()` provides standard conditional routing logic for tool-calling workflows
- **Human Interaction**: `HumanInterrupt`, `HumanResponse`, and related types support human-in-the-loop workflows

All components are designed to work together seamlessly to create production-ready agentic applications with minimal boilerplate code.
