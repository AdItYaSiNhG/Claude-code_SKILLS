---
name: ai-agents
description: >
  Use this skill for AI agent and agentic system tasks: tool-calling agent design,
  ReAct/plan-and-execute patterns, multi-agent orchestration, agent memory systems,
  human-in-the-loop checkpoints, agent evaluation, tool design, and production agent
  reliability (retries, timeouts, cost guardrails, loop detection).
  Triggers: "agent", "tool calling", "agentic", "multi-agent", "ReAct", "plan and
  execute", "autonomous", "agent loop", "agent memory", "agent tools", "orchestrator",
  "LangGraph", "AutoGen", "CrewAI", "human in the loop".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# AI Agents Skill

## Why this skill exists

Agents fail in ways that are expensive: infinite loops, runaway API costs,
incorrect tool calls with side effects, and hallucinated tool results. This skill
provides **production-safe agent patterns** — correct tool design, loop detection,
cost guardrails, human-in-the-loop checkpoints, and evaluation frameworks.

---

## 1. Tool design principles

```python
# Good tools are: narrow, safe, idempotent, with typed schemas
# Bad tools are: broad ("do_everything"), stateful without undo, loosely typed

# tools/search.py
from anthropic import Anthropic
import json

client = Anthropic()

TOOLS = [
    {
        "name": "search_codebase",
        "description": "Search for files or code patterns in the current repository. Returns matching file paths and line snippets. Use for finding where something is defined or used.",
        "input_schema": {
            "type": "object",
            "properties": {
                "pattern": {
                    "type": "string",
                    "description": "grep-compatible regex pattern to search for"
                },
                "file_glob": {
                    "type": "string",
                    "description": "Optional glob to restrict search (e.g. '*.ts', 'src/**/*.py')",
                    "default": "*"
                },
                "max_results": {
                    "type": "integer",
                    "description": "Maximum results to return",
                    "default": 20,
                    "maximum": 50
                }
            },
            "required": ["pattern"]
        }
    },
    {
        "name": "read_file",
        "description": "Read the contents of a specific file. Returns the file content as text.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {
                    "type": "string",
                    "description": "Relative path to the file from repo root"
                },
                "start_line": {"type": "integer", "description": "Optional: start line (1-indexed)"},
                "end_line":   {"type": "integer", "description": "Optional: end line (inclusive)"}
            },
            "required": ["path"]
        }
    },
    {
        "name": "write_file",
        "description": "Write or update a file. Creates parent directories if needed. DESTRUCTIVE — use only when explicitly asked to modify files.",
        "input_schema": {
            "type": "object",
            "properties": {
                "path":    {"type": "string", "description": "File path relative to repo root"},
                "content": {"type": "string", "description": "Full file content to write"}
            },
            "required": ["path", "content"]
        }
    },
    {
        "name": "run_command",
        "description": "Execute a shell command. Use for running tests, linters, or build commands. Do NOT use for git push, rm -rf, or any destructive operations.",
        "input_schema": {
            "type": "object",
            "properties": {
                "command":    {"type": "string", "description": "Shell command to run"},
                "timeout_sec":{"type": "integer", "default": 30, "maximum": 120}
            },
            "required": ["command"]
        }
    }
]
```

---

## 2. ReAct agent loop

```python
# agents/react_agent.py
import subprocess, json, re
from pathlib import Path
from anthropic import Anthropic

client = Anthropic()

def execute_tool(name: str, inputs: dict) -> str:
    """Execute a tool call and return string result."""
    try:
        if name == "search_codebase":
            pattern  = inputs["pattern"]
            glob     = inputs.get("file_glob", "*")
            max_r    = inputs.get("max_results", 20)
            result   = subprocess.run(
                ["grep", "-rn", "--include", glob, pattern, "."],
                capture_output=True, text=True, timeout=10
            )
            lines = result.stdout.splitlines()[:max_r]
            return "\n".join(lines) or "No matches found"

        elif name == "read_file":
            path = Path(inputs["path"])
            if not path.exists(): return f"File not found: {path}"
            text  = path.read_text(errors="replace")
            lines = text.splitlines()
            s, e  = inputs.get("start_line", 1) - 1, inputs.get("end_line", len(lines))
            return "\n".join(lines[s:e])

        elif name == "write_file":
            path = Path(inputs["path"])
            path.parent.mkdir(parents=True, exist_ok=True)
            path.write_text(inputs["content"])
            return f"Written: {path} ({len(inputs['content'])} bytes)"

        elif name == "run_command":
            # Safety check — block destructive commands
            cmd = inputs["command"]
            BLOCKED = ["git push", "rm -rf", "DROP TABLE", "format c:", "> /dev/"]
            if any(b in cmd for b in BLOCKED):
                return f"BLOCKED: command contains disallowed operation"
            result = subprocess.run(cmd, shell=True, capture_output=True,
                                    text=True, timeout=inputs.get("timeout_sec", 30))
            output = (result.stdout + result.stderr).strip()
            return output[:4000] or "(no output)"

        else:
            return f"Unknown tool: {name}"

    except subprocess.TimeoutExpired:
        return f"Tool timed out after {inputs.get('timeout_sec', 30)}s"
    except Exception as e:
        return f"Tool error: {type(e).__name__}: {e}"


def run_agent(task: str, max_iterations: int = 15,
              budget_usd: float = 0.50) -> str:
    """
    Run a ReAct agent loop until task complete or limits hit.
    Guards: max_iterations, budget cap, loop detection.
    """
    messages = [{"role": "user", "content": task}]
    total_cost   = 0.0
    seen_states  = set()

    for iteration in range(max_iterations):
        response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4096,
            tools=TOOLS,
            messages=messages,
        )

        # Cost tracking (sonnet: $3/$15 per M tokens)
        cost = (response.usage.input_tokens * 3 + response.usage.output_tokens * 15) / 1_000_000
        total_cost += cost
        print(f"[iter {iteration+1}] stop={response.stop_reason} cost=${total_cost:.4f}")

        # Budget guard
        if total_cost > budget_usd:
            return f"STOPPED: budget ${budget_usd} exceeded at iteration {iteration+1}"

        # Done — no tool calls
        if response.stop_reason == "end_turn":
            text_blocks = [b.text for b in response.content if hasattr(b, "text")]
            return "\n".join(text_blocks)

        # Process tool calls
        messages.append({"role": "assistant", "content": response.content})
        tool_results = []

        for block in response.content:
            if block.type != "tool_use":
                continue

            # Loop detection — same tool + same inputs twice = infinite loop
            state_key = f"{block.name}:{json.dumps(block.input, sort_keys=True)}"
            if state_key in seen_states:
                tool_results.append({
                    "type": "tool_result",
                    "tool_use_id": block.id,
                    "content": "LOOP DETECTED: identical call made before. Try a different approach.",
                    "is_error": True,
                })
                continue
            seen_states.add(state_key)

            result = execute_tool(block.name, block.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": result,
            })

        messages.append({"role": "user", "content": tool_results})

    return f"STOPPED: reached max {max_iterations} iterations"
```

---

## 3. Human-in-the-loop checkpoints

```python
# agents/hitl_agent.py — pause before destructive/irreversible actions

REQUIRES_APPROVAL = {"write_file", "run_command", "send_email", "deploy", "delete_record"}

def ask_human(tool_name: str, tool_inputs: dict) -> bool:
    """Returns True if human approves the action."""
    print(f"\n⚠️  Agent wants to call: {tool_name}")
    print(f"   Inputs: {json.dumps(tool_inputs, indent=2)}")
    answer = input("   Approve? [y/N] ").strip().lower()
    return answer == "y"

def execute_tool_with_hitl(name: str, inputs: dict, auto_approve: bool = False) -> str:
    if name in REQUIRES_APPROVAL and not auto_approve:
        if not ask_human(name, inputs):
            return f"SKIPPED: user declined {name}"
    return execute_tool(name, inputs)
```

---

## 4. Multi-agent orchestration

```python
# agents/orchestrator.py — supervisor pattern
import asyncio

class AgentOrchestrator:
    """Supervisor delegates subtasks to specialised sub-agents."""

    AGENTS = {
        "researcher": "You are a research agent. Search and read files to gather information. Do NOT write files.",
        "writer":     "You are a code writing agent. Write and modify files based on provided specifications. Always run tests after writing.",
        "reviewer":   "You are a code review agent. Read code and provide structured feedback. Do NOT modify files.",
    }

    async def run(self, task: str) -> str:
        # Step 1: Supervisor plans subtasks
        plan_response = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1024,
            system="""You are a task planner. Given a task, break it into subtasks
and assign each to: researcher, writer, or reviewer.
Return JSON: {"subtasks": [{"agent": "...", "task": "...", "depends_on": []}]}""",
            messages=[{"role": "user", "content": task}],
        )
        plan = json.loads(plan_response.content[0].text)

        results = {}
        for subtask in plan["subtasks"]:
            # Wait for dependencies
            deps_done = all(d in results for d in subtask.get("depends_on", []))
            if not deps_done:
                await asyncio.sleep(0.1)

            agent_name = subtask["agent"]
            agent_task = subtask["task"]
            if results:
                agent_task += f"\n\nContext from previous steps:\n{json.dumps(results, indent=2)}"

            result = await self._run_subagent(agent_name, agent_task)
            results[f"{agent_name}:{agent_task[:30]}"] = result

        # Step 2: Synthesise results
        synthesis = client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=2048,
            system="Synthesise the results of multiple agent subtasks into a coherent final answer.",
            messages=[{"role": "user", "content": f"Original task: {task}\n\nResults: {json.dumps(results)}"}],
        )
        return synthesis.content[0].text

    async def _run_subagent(self, agent_name: str, task: str) -> str:
        system_prompt = self.AGENTS[agent_name]
        return run_agent(task)  # uses the ReAct loop above
```

---

## 5. Agent memory

```python
# agents/memory.py — different memory types
import json
from pathlib import Path
from datetime import datetime

class AgentMemory:
    """
    Three memory tiers:
    - Working: current conversation context (managed by message list)
    - Episodic: past task summaries (persisted to disk)
    - Semantic: extracted facts/knowledge (vector store)
    """

    def __init__(self, agent_id: str, memory_dir: str = ".agent_memory"):
        self.agent_id  = agent_id
        self.mem_dir   = Path(memory_dir) / agent_id
        self.mem_dir.mkdir(parents=True, exist_ok=True)
        self.episodic_file = self.mem_dir / "episodes.jsonl"

    def save_episode(self, task: str, result: str, metadata: dict = None):
        episode = {
            "timestamp": datetime.now().isoformat(),
            "task":      task[:200],
            "result":    result[:500],
            "metadata":  metadata or {},
        }
        with open(self.episodic_file, "a") as f:
            f.write(json.dumps(episode) + "\n")

    def get_relevant_episodes(self, task: str, limit: int = 3) -> list[dict]:
        if not self.episodic_file.exists(): return []
        episodes = [json.loads(l) for l in self.episodic_file.read_text().splitlines()]
        # Simple keyword match — replace with vector search for production
        task_words = set(task.lower().split())
        scored = [(e, len(task_words & set(e["task"].lower().split()))) for e in episodes]
        return [e for e, score in sorted(scored, key=lambda x: -x[1]) if score > 0][:limit]

    def build_memory_context(self, task: str) -> str:
        episodes = self.get_relevant_episodes(task)
        if not episodes: return ""
        lines = ["Relevant past work:"]
        for ep in episodes:
            lines.append(f"- {ep['timestamp'][:10]}: {ep['task']} → {ep['result'][:100]}")
        return "\n".join(lines)
```

---

## 6. Agent evaluation

```python
# evals/agent_eval.py
import time
from dataclasses import dataclass

@dataclass
class AgentEvalCase:
    id:          str
    task:        str
    success_criteria: list[str]   # assertions to check in final output
    max_iterations: int = 10
    max_cost_usd:   float = 0.20
    should_call_tools: list[str] = None  # tools that must be called

@dataclass
class AgentEvalResult:
    case_id:     str
    passed:      bool
    iterations:  int
    cost_usd:    float
    final_output: str
    failures:    list[str]

async def run_agent_eval(case: AgentEvalCase) -> AgentEvalResult:
    start = time.perf_counter()
    calls_made = []

    # Intercept tool calls for assertion
    original_execute = execute_tool
    def tracked_execute(name, inputs):
        calls_made.append(name)
        return original_execute(name, inputs)

    output = run_agent(case.task, max_iterations=case.max_iterations,
                       budget_usd=case.max_cost_usd)

    failures = []
    for criterion in case.success_criteria:
        if criterion.lower() not in output.lower():
            failures.append(f"Missing: '{criterion}' in output")

    if case.should_call_tools:
        for tool in case.should_call_tools:
            if tool not in calls_made:
                failures.append(f"Tool not called: {tool}")

    return AgentEvalResult(
        case_id=case.id,
        passed=len(failures) == 0,
        iterations=len(calls_made),
        cost_usd=0.0,
        final_output=output,
        failures=failures,
    )
```

---

## 7. Production agent checklist

```
Safety
  [ ] All destructive tools require explicit human approval in production
  [ ] Max iterations hard cap (never unbounded loops)
  [ ] Per-session USD budget cap with early exit
  [ ] Loop detection (identical tool call twice → error + different approach)
  [ ] Tool input validation before execution
  [ ] Blocked command list for shell tools (rm -rf, git push --force, DROP, etc.)

Observability
  [ ] Every tool call logged with inputs + outputs + latency
  [ ] Total token usage and cost tracked per session
  [ ] Agent trace stored (Langfuse / Phoenix) for debugging
  [ ] Failure mode: graceful degradation, not silent hang

Reliability
  [ ] Tool calls wrapped in try/except — errors returned as tool_result, not crashes
  [ ] Timeout on every external call
  [ ] Retry with backoff on rate limits only (not logic errors)
  [ ] Result truncation for large tool outputs (>4000 chars)

Testing
  [ ] Eval suite covers: happy path, tool failure, budget exceeded, loop
  [ ] Regression tests for past failure modes
  [ ] Cost-per-task benchmarked and alerted on regression
```
