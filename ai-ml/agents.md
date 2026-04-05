---
name: ai-ml-agents
description: Designing, implementing, and debugging multi-step LLM agent systems. Covers tool use, orchestration frameworks (LangGraph, CrewAI, custom), state management, error recovery, human-in-the-loop patterns, and agent safety.
---

The user is building or debugging an agentic system. Apply the framework below.

## Agent Architecture Patterns

### Pattern selection
```
Single agent + tools    → 1-5 tools, linear task, deterministic
Supervisor + workers    → parallel subtasks, different specialisations
LangGraph state machine → complex branching, conditional routing, retries
Human-in-the-loop       → high-stakes actions (send email, deploy, payment)
Multi-agent debate       → fact-checking, adversarial validation
```

### When NOT to use agents
- Task is deterministic and codable → write a function
- Single LLM call with structured output suffices → don't add orchestration
- Latency budget < 2s → agents add too much overhead

## Tool Design

```python
# Anthropic tool use (Claude)
tools = [
    {
        "name": "search_orders",
        "description": "Search orders by customer email or order ID. Returns a list of matching orders with status and total.",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "Email address or order ID to search for"},
                "limit": {"type": "integer", "default": 10, "maximum": 50}
            },
            "required": ["query"]
        }
    }
]
```

### Tool design rules
- Description must explain WHEN to use the tool, not just what it does
- Parameters must be typed with constraints (`maximum`, `enum`, `pattern`)
- Always include error response shape in the description
- Destructive tools (delete, send, pay) must require explicit confirmation parameter
- Tools should be idempotent where possible

## Orchestration with LangGraph

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict, Annotated
import operator

class AgentState(TypedDict):
    messages: Annotated[list, operator.add]
    order_context: dict | None
    attempts: int

def should_continue(state: AgentState) -> str:
    last = state["messages"][-1]
    if last.tool_calls:
        return "tools"
    if state["attempts"] >= 3:
        return "escalate"
    return END

graph = StateGraph(AgentState)
graph.add_node("agent", call_llm)
graph.add_node("tools", execute_tools)
graph.add_node("escalate", human_escalation)

graph.add_conditional_edges("agent", should_continue)
graph.add_edge("tools", "agent")
graph.set_entry_point("agent")

app = graph.compile(checkpointer=MemorySaver())  # enables interruption + resume
```

## Error Recovery

```python
# Retry with exponential backoff
import tenacity

@tenacity.retry(
    stop=tenacity.stop_after_attempt(3),
    wait=tenacity.wait_exponential(multiplier=1, min=1, max=10),
    retry=tenacity.retry_if_exception_type((RateLimitError, APITimeoutError)),
    before_sleep=tenacity.before_sleep_log(logger, logging.WARNING),
)
async def call_agent_step(state: AgentState) -> AgentState: ...

# Tool error handling — always return structured errors to the agent
async def execute_tool(tool_call) -> dict:
    try:
        result = await dispatch_tool(tool_call)
        return {"status": "success", "data": result}
    except ToolError as e:
        # Let the agent decide how to recover
        return {"status": "error", "error_code": e.code, "message": str(e)}
```

## Human-in-the-Loop (HiTL)

```python
# LangGraph interrupt for approval
from langgraph.types import interrupt

def request_approval(state: AgentState):
    action_summary = summarise_action(state)
    approval = interrupt({
        "type": "approval_required",
        "action": action_summary,
        "risk_level": "high",
    })
    if not approval["approved"]:
        return {**state, "cancelled": True}
    return state

# Trigger HiTL before any:
# - External API calls with side effects (email, payment, deploy)
# - Database writes that are not reversible
# - Actions affecting > N records
```

## Agent Safety Rules

```
1. Principle of least authority — tools only access what the task requires
2. Sandbox external code execution (E2B, Modal, or Docker container)
3. Rate limit tool calls per agent session (max N calls per minute)
4. Log every tool call with inputs/outputs for audit trail
5. Set absolute token budget per session (max_tokens × max_steps)
6. Never interpolate user input directly into shell commands or SQL
7. Require confirmation for irreversible actions regardless of instruction
```

## Observability

```python
from langfuse.decorators import observe

@observe(name="agent-session")
async def run_agent(task: str, user_id: str):
    langfuse_context.update_current_trace(
        user_id=user_id,
        tags=["agent", "orders"],
    )
    # Each tool call auto-traced as a child span
    ...

# Track per session:
# - Total tokens consumed
# - Number of tool calls
# - Number of LLM calls (steps)
# - Wall-clock duration
# - Success / failure / escalation outcome
```

## Output Format

For agent design tasks:
1. Recommend the architecture pattern with justification
2. Show the state schema and graph definition
3. Show tool definitions (full schema, not pseudocode)
4. Include error recovery and HiTL gates explicitly
5. Note the safety controls applied
