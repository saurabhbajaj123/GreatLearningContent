# Week 8 Agentic Systems Notes

These notes collect the explanations from this chat about the notebook `Week_8_Advanced_Python_OOPs_for_Intelligent_Agentic_Systems_(1).ipynb`.

## Objective

The notebook's objective is to build **ShopSmart**, an intelligent support system that:

- is structured and modular
- supports async workflows
- integrates LLMs with tools and company knowledge
- handles HR, IT, and customer support requests
- is resilient, scalable, extensible, reliable, and cost-efficient

In simple terms, the goal is not just to build a chatbot. It is to build an **agentic support system** that can understand requests, route them, use tools, and respond in a grounded way.

## Why `BaseModel`, `Field`, and `EmailStr` are used

`BaseModel`, `Field`, and `EmailStr` come from Pydantic.

- `BaseModel` is used to define structured data models.
- `Field` is used to add validation rules, defaults, and descriptions.
- `EmailStr` is a special type that validates email addresses.

These are useful because the project relies on clean, validated request objects instead of loose dictionaries.

Example:

```python
from pydantic import BaseModel, Field, EmailStr

class UserRequest(BaseModel):
    name: str = Field(..., min_length=2)
    email: EmailStr
    issue: str = Field(..., description="User support issue")
```

In this example:

- `BaseModel` gives the object structure and validation
- `Field` says `name` is required and must be at least 2 characters
- `EmailStr` ensures `email` looks like a real email address

## Why the other imports are needed

The import block supports several responsibilities:

- core Python utilities: `os`, `re`, `json`, `datetime`, `asyncio`, typing helpers
- structured data and validation: Pydantic models
- LLM access: OpenAI and Azure OpenAI clients
- prompt construction: `ChatPromptTemplate`
- retrieval: `Document`, `TextLoader`, `Chroma`, `RecursiveCharacterTextSplitter`
- tools: `Tool`, `StructuredTool`
- notebook/Colab secrets: `google.colab.userdata`

Some imports appear duplicated, such as repeated `BaseModel`, `Field`, and `Chroma` imports.

## Why `asyncio` and `nest_asyncio` are used

- `asyncio` is Python's async framework. It lets the app run non-blocking tasks such as API calls and LLM requests.
- `nest_asyncio` is often used in notebooks so async code can run inside an already-running event loop.

In notebooks, trying to start a new event loop can fail. `nest_asyncio` helps avoid that issue.

Example idea:

```python
async def fetch_data():
    await some_api_call()
```

And in a notebook, trying to run async code without the notebook-friendly event-loop patch may raise an error such as:

```python
RuntimeError: This event loop is already running
```

## Why ticket classes inherit from `BaseModel`

The notebook defines ticket classes as Pydantic models so incoming tickets have:

- a fixed structure
- validation
- clearer contracts between components

This means the system can treat tickets as reliable structured objects instead of unvalidated free-form data.

## The `Ticket` model

The base `Ticket` model stores common fields shared by all support requests:

- `ticket_id`
- `created_at`
- `created_by`
- `input`
- `intent`
- `sentiment`
- `status`
- `priority`
- `response`
- `error`

This acts as the common support-ticket container that gets enriched as the workflow progresses.

Example:

```python
ticket = Ticket(
    ticket_id="T-1001",
    created_at=datetime.now(),
    created_by="alice",
    input="My laptop won't connect to VPN."
)
```

At creation time, the ticket may only contain the basic request data. Later workflow steps may enrich it with:

- `intent="IT_support"`
- `sentiment="frustrated"`
- `response="Please restart the VPN client and try again."`

## Why some fields are optional

Some fields are optional because that information may not exist yet when the ticket is first created.

For example:

- `intent` is discovered later
- `sentiment` is inferred later
- `response` is generated later
- `error` is only filled if something goes wrong

So the ticket starts small and becomes richer over time.

That is why fields like `intent`, `sentiment`, `response`, and `error` are optional: they are often filled in after the first ticket object is created.

## Why there is a base `Ticket` plus specialized ticket classes

The notebook defines `Ticket` and then specialized versions like:

- `CustomerSupportTicket`
- `HRTicket`
- `ITTicket`

This is done because:

- all tickets share common fields
- each domain needs extra fields specific to its own workflow

Examples:

- customer support may need `order_data`, `product_id`, `refund_amount`
- HR may need `employee_id`, `department`, `request_type`
- IT may need `system`, `severity`, `assigned_to`

This avoids repeating shared fields while still keeping domain-specific data organized.

A simple way to decide what fields belong in each subclass when building from scratch is to ask:

1. What must be known to create the ticket?
2. What must be known to classify or route it?
3. What must be known to resolve it?
4. What must be stored for tracking later?

For example:

- an IT ticket may need `system`, `severity`, and `assigned_to`
- an HR ticket may need `employee_id`, `department`, and `request_type`
- a customer support ticket may need `order_data`, `product_id`, and `refund_amount`

## How domain-specific organization and validation work in the current code

The current code already does **clean organization** through inheritance:

- `Ticket` stores shared fields
- each subclass adds its own fields

Basic validation is happening because Pydantic checks:

- required vs optional fields
- field types like `str`, `float`, `bool`, `datetime`

But the code does **not yet strongly enforce business rules**.

Examples of what is not fully enforced yet:

- `status`, `priority`, and `severity` are plain strings, so invalid values could still be passed
- `issue_type` comments describe allowed values, but those values are not strictly validated
- there are no validators like "if issue type is refund, `refund_amount` must be present"

So the code currently has strong **schema organization**, but only partial **business-rule validation**.

## What OOP technique is being used

The main OOP technique is **inheritance**.

Examples:

- `CustomerSupportTicket(Ticket)`
- `HRTicket(Ticket)`
- `ITTicket(Ticket)`

Other OOP ideas also show up:

- **abstraction**: `Ticket` represents the general idea of a support request
- **specialization**: subclasses add domain-specific fields
- **encapsulation**: related ticket data is grouped inside one object
- **polymorphism**: different ticket types can still be treated as a `Ticket`

## Inheritance, abstraction, encapsulation, and polymorphism using `Ticket`

### Inheritance

Child classes reuse and extend the parent class.

Example:

- `HRTicket` inherits all common fields from `Ticket`
- then adds HR-specific fields

```python
class Ticket(BaseModel):
    ticket_id: str
    input: str

class HRTicket(Ticket):
    employee_id: Optional[str] = None
    request_type: Optional[str] = None
```

So `HRTicket` automatically gets `ticket_id` and `input`, then adds HR-specific information.

### Abstraction

`Ticket` represents the general concept of a support request, without caring yet whether it belongs to HR, IT, or customer support.

### Encapsulation

Each ticket object groups related data together instead of scattering it across unrelated variables.

Example:

```python
it_ticket = ITTicket(
    ticket_id="101",
    created_at=datetime.now(),
    created_by="Alice",
    input="VPN is down",
    system="VPN",
    severity="high"
)
```

All the important ticket data lives inside one object instead of separate standalone variables.

### Polymorphism

A function that accepts a `Ticket` can also accept `HRTicket`, `ITTicket`, or `CustomerSupportTicket`, because each one is still a kind of `Ticket`.

Example:

```python
def print_ticket_summary(ticket: Ticket):
    print(ticket.ticket_id, ticket.input, ticket.status)
```

This function can work with any subclass because each specialized ticket is still a `Ticket`.

## `SmartLLM` class overview

`SmartLLM` is the notebook's main orchestrator for the LLM-powered workflow.

It combines:

- the chat model
- the embedding model
- the vector DB
- retrieval
- guardrails
- tool registration
- planning
- final answer generation

Its high-level flow is:

1. take the user query
2. apply a guardrail
3. retrieve relevant docs
4. ask the LLM to make a plan
5. run tools if needed
6. ask the LLM to generate the final answer

### Important `SmartLLM` pieces

- `self.llm`: main chat model
- `self.embedder`: embedding model
- `self.vector_db`: vector store helper
- `self.retriever`: retrieval interface
- `self.tools`: registered tools
- `self.router_llm`: optional separate model for planning
- `self.guardrail`: optional input-sanitization function
- `self.system_policy`: behavior rules for the assistant
- `self.plan_prompt`: prompt used to plan tool usage
- `self.answer_prompt`: prompt used to generate final grounded responses

### `answer()` vs `aanswer()`

- `answer()` is the synchronous version
- `aanswer()` is the asynchronous version

Both perform the same overall workflow, but `aanswer()` uses async LLM calls.

Simple request flow:

1. user sends a query
2. guardrail sanitizes the query
3. retriever fetches relevant internal knowledge
4. planner decides whether tools are needed
5. tools execute if needed
6. final answer is generated using the context and tool results

## The `simple_guardrail` function

`simple_guardrail(text)` is a basic privacy filter.

It:

- strips extra whitespace
- redacts email addresses
- redacts phone numbers

So it helps avoid sending raw personal information deeper into the system.

Example:

```python
"My email is test@example.com and my number is 415-555-1212"
```

becomes:

```python
"My email is [redacted-email] and my number is [redacted-phone]"
```

## `DocumentChunker` and `VectorDB`

These two classes support retrieval-augmented generation.

### `DocumentChunker`

`DocumentChunker` breaks large documents into smaller chunks.

This helps because smaller chunks:

- embed better
- retrieve more accurately
- fit more easily into prompts

### `VectorDB`

`VectorDB` wraps a Chroma vector database.

It can:

- build the vector store from documents
- add more documents later
- expose a retriever

Together, `DocumentChunker` and `VectorDB` let the assistant search internal knowledge semantically instead of using only keyword matching.

Simple example:

1. a refund-policy document is loaded
2. `DocumentChunker` splits it into smaller chunks
3. `VectorDB` stores those chunks as embeddings
4. the user asks, "Can I get a refund for my delayed order?"
5. the retriever fetches the most relevant policy chunks
6. the LLM answers using those chunks as grounding context

## Decorators: `log_step` and `retry_async`

The notebook defines two decorators:

- `log_step`
- `retry_async`

### `log_step`

This decorator logs:

- function input
- function output

It works for both sync and async functions.

Example usage:

```python
@log_step
def run(self, context):
    ...
```

When the function runs, the decorator prints:

- the function name
- the input arguments
- the returned result

### `retry_async`

This decorator retries an async function if it fails.

It:

- attempts the call multiple times
- waits between retries
- raises the last exception if all retries fail

This adds resilience for temporary failures like network issues or API timeouts.

Example usage:

```python
@retry_async(times=3, delay=2)
async def fetch_data():
    ...
```

If the first call fails, the decorator waits and retries automatically before finally giving up.

## What `@wraps` does

`@wraps` is used inside decorators to preserve the original function's identity.

Without `@wraps`, a decorated function may look like the wrapper instead of the original function.

Problems without `@wraps`:

- function name may become `wrapper`
- docstrings may be lost
- debugging becomes harder
- introspection tools become less useful

With `@wraps`, Python preserves metadata like:

- `__name__`
- `__doc__`
- `__module__`
- `__wrapped__`

So a decorated function still looks like the original function for debugging and tooling.

### Example without `@wraps`

```python
def log_decorator(func):
    def wrapper(*args, **kwargs):
        print("Running function")
        return func(*args, **kwargs)
    return wrapper

@log_decorator
def greet(name: str):
    """Say hello."""
    return f"Hello, {name}"
```

Then:

```python
print(greet.__name__)   # wrapper
print(greet.__doc__)    # None
```

### Example with `@wraps`

```python
from functools import wraps

def log_decorator(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print("Running function")
        return func(*args, **kwargs)
    return wrapper

@log_decorator
def greet(name: str):
    """Say hello."""
    return f"Hello, {name}"
```

Then:

```python
print(greet.__name__)   # greet
print(greet.__doc__)    # Say hello.
```

So `@wraps` preserves the original function metadata even though a wrapper is still being used underneath.

## `SupportTicket`, `CheckOrderArgs`, and `InitiateRefundArgs`

These are Pydantic models used in the workflow.

### `SupportTicket`

`SupportTicket` represents the support case as it moves through the workflow.

It stores:

- the user's input
- inferred intent
- fetched order data
- sentiment
- generated response
- error information

Example:

```python
ticket = SupportTicket(input="Where is my order ORD-123?")
```

Later, the system may enrich the same object with:

```python
ticket.intent = "check_order"
ticket.order_data = {"order_id": "ORD-123", "status": "Shipped"}
ticket.response = "Your order has shipped and should arrive in 2 days."
```

### `CheckOrderArgs`

This defines the expected arguments for the order-status tool:

- required `order_id`

### `InitiateRefundArgs`

This defines the expected arguments for the refund tool:

- required `order_id`
- required `amount`

These argument schemas are useful because tools need structured, validated inputs.

You can think of the difference like this:

- `SupportTicket` is the full case file
- `CheckOrderArgs` and `InitiateRefundArgs` are the forms required by specific tools

## `WorkflowStep` and `GenerateResponseStep`

`WorkflowStep` is a base class that defines the common contract for workflow steps.

It says every step must implement:

```python
run(context) -> SupportTicket
```

`GenerateResponseStep` is one concrete workflow step.

It:

- receives a `SupportTicket`
- calls the LLM wrapper
- stores the answer in `context.response`
- returns the updated ticket

This shows:

- inheritance
- abstraction
- composition
- a pipeline-style design where the same ticket object is passed through multiple steps

Simple pipeline example:

1. `SupportTicket(input="Where is my order ORD-123?")` enters the workflow
2. one step may classify intent
3. another step may call a tool to fetch order data
4. `GenerateResponseStep` fills `context.response`
5. the enriched ticket comes out the other end

## How to use the async answer method

To use `aanswer()` instead of `answer()`, the calling function must also become async.

Example idea:

```python
@log_step
async def run(self, context: SupportTicket) -> SupportTicket:
    context.response = await self.llm.aanswer(context.input)
    return context
```

Why:

- `aanswer()` is async
- async functions must be awaited
- the caller must therefore also be `async def`

If the whole pipeline is not async yet, another option is to keep:

- `run()` for sync
- `arun()` for async

For example:

```python
class GenerateResponseStep(WorkflowStep):
    def __init__(self, llm_wrapper: SmartLLM):
        self.llm = llm_wrapper

    @log_step
    def run(self, context: SupportTicket) -> SupportTicket:
        context.response = self.llm.answer(context.input)
        return context

    @log_step
    async def arun(self, context: SupportTicket) -> SupportTicket:
        context.response = await self.llm.aanswer(context.input)
        return context
```

## Why the raw tool functions are needed

The notebook defines raw functions such as:

- `_check_order_status_fn(order_id)`
- `_initiate_refund_fn(order_id, amount)`

These are needed because they represent the **real business actions** the system can perform.

Examples:

- check order status
- initiate a refund

Without them, the LLM could only talk about those actions, not actually perform them.

Example outputs:

```python
_check_order_status_fn("ORD-1001")
```

may return:

```python
{"order_id": "ORD-1001", "status": "Shipped", "eta_days": 2}
```

and:

```python
_initiate_refund_fn("ORD-1001", 200.0)
```

may return:

```python
{"ok": True, "order_id": "ORD-1001", "refund_id": "RF-12345"}
```

If the refund amount is too high, the function can enforce policy instead of blindly approving it.

## How the LLM calls those functions

The LLM does **not** execute Python functions directly.

The flow is:

1. the user asks a question
2. the LLM sees the list of available tools
3. the LLM returns a structured plan saying which tool to call
4. Python code reads that plan
5. Python code finds the matching registered tool
6. Python code executes the tool
7. the tool result is fed back into the final LLM response

The actual tool execution happens in `_exec_tools()` at the line:

```python
out = tool.run(args)
```

So:

- the LLM chooses the tool
- the Python program runs it
- the LLM then turns the result into a natural-language answer

Concrete example:

1. user asks: "Where is my order `ORD-123`?"
2. planner returns a tool call such as:

```json
{
  "intent": "order_status",
  "tool_calls": [
    {"name": "check_order_status", "args": {"order_id": "ORD-123"}}
  ]
}
```

3. `_exec_tools()` looks up the tool by name
4. `tool.run(args)` executes the wrapped Python function
5. the result comes back
6. the final answer prompt uses that result to write a response such as "Your order has shipped and should arrive in 2 days."

## Simple explanation of the tool-calling flow

In plain English:

1. the user asks a question
2. the LLM decides a tool is needed
3. the LLM returns instructions, not code execution
4. your Python code reads those instructions
5. your Python code looks up the right tool
6. your Python code actually runs `tool.run(args)`
7. the underlying function returns a result
8. the result is sent back to the LLM
9. the LLM writes the final reply

## Why `StructuredTool.from_function(...)` is needed

The raw functions alone are not enough for the agent workflow.

`StructuredTool.from_function(...)` wraps a Python function into a proper tool object with:

- a tool name
- a description
- a structured argument schema
- the actual underlying function

Example:

- raw function = engine
- structured tool = labeled machine with instructions

This is necessary because the LLM/tool system needs to know:

- what the tool is called
- what it does
- what arguments it expects
- how to validate and execute it

So:

- `_check_order_status_fn` is the underlying logic
- `check_order_status_tool = StructuredTool.from_function(...)` is the agent-friendly wrapper

In simple terms:

- raw function = engine
- structured tool = engine plus a label, instructions, and an input form

That wrapper is what lets the LLM planning system discover the function and ask the Python code to run it safely.

## Short recap

This notebook combines several important ideas:

- Pydantic models for structured and validated data
- OOP concepts like inheritance, abstraction, and encapsulation
- decorators for logging and retry behavior
- async support
- RAG through chunking, embeddings, and vector search
- tool calling so the LLM can trigger real business logic
- a workflow design where tickets get enriched step by step

Overall, it is a small example of an **agentic support system** rather than a plain chatbot.
