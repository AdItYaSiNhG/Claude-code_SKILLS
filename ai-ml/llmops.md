---
name: ai-ml-llmops
description: LLMOps — deploying, evaluating, monitoring, and governing large language models in production. Covers model selection, prompt versioning, A/B testing, Langfuse/Helicone observability, guardrails, and cost governance.
---

The user is operating or improving LLMs in production. Apply the following framework.

## Model Selection Decision Tree

```
Latency < 500ms required?
  Yes → claude-haiku-4-5, gpt-4o-mini, gemini-flash
  No  → continue

Context > 100K tokens?
  Yes → claude-sonnet-4-6, gemini-1.5-pro
  No  → continue

Code generation primary?
  Yes → claude-sonnet-4-6, gpt-4o, deepseek-coder-v2
  No  → continue

On-premise / air-gapped required?
  Yes → Llama 3.3-70B, Mistral-Large (self-hosted)
  No  → claude-sonnet-4-6 (default recommendation)
```

**Model tier guidance for Claude Code users:**
- Routine tasks (summarise, classify, extract) → Haiku (10× cheaper than Sonnet)
- Complex reasoning, code review, analysis → Sonnet
- Multi-step agentic workflows → Sonnet with extended thinking
- Never use Opus for automated pipelines — reserve for human-in-the-loop tasks

## Prompt Management

### Version-controlled prompts
```
prompts/
  v1/
    system.txt
    user_template.jinja2
  v2/
    system.txt
    user_template.jinja2
  current -> v2/        # symlink, easy rollback
```

### Prompt registry (LangSmith / Langfuse)
```python
from langfuse import Langfuse
lf = Langfuse()

# Fetch versioned prompt (cached, refreshes every 60s)
prompt = lf.get_prompt("order-classifier", version=3)
compiled = prompt.compile(order_text=order.text)
```

### Prompt hardening checklist
- [ ] System prompt defines role, output format, and refusal behaviour
- [ ] User input sanitised before interpolation (strip `<`, `>`, XML-like tags)
- [ ] Output format enforced via structured output / JSON mode / tool use
- [ ] Jailbreak / injection test suite runs in CI (≥50 adversarial inputs)
- [ ] PII detection before logging any prompt/response pair

## Evaluation Framework

### Eval types and when to run
| Eval type | Trigger | Threshold |
|---|---|---|
| Unit evals | Every commit | >95% pass |
| Regression evals | Pre-deploy | No degradation vs baseline |
| Shadow evals | Continuous (10% traffic) | Alert on >5% drop |
| Human evals | Monthly or major prompt change | Team-defined rubric |

### Running evals with Langfuse
```python
from langfuse import Langfuse
from langfuse.decorators import observe

@observe()
async def classify_order(text: str) -> str:
    response = await anthropic.messages.create(
        model="claude-haiku-4-5-20251001",
        messages=[{"role": "user", "content": text}]
    )
    return response.content[0].text

# Score in the eval run
lf.score(trace_id=trace.id, name="accuracy", value=1.0)
```

### LLM-as-judge eval pattern
```python
JUDGE_PROMPT = """
You are an evaluator. Rate the following response on a scale of 0-1.

Criteria: {criteria}
Question: {question}
Response: {response}

Return ONLY a JSON object: {{"score": <float>, "reason": "<string>"}}
"""

async def judge_response(question, response, criteria):
    result = await call_llm(JUDGE_PROMPT.format(**locals()))
    return json.loads(result)
```

## Observability (Langfuse / Helicone)

### Mandatory instrumentation
```python
# Every LLM call must emit:
# - model name and version
# - input/output token counts
# - latency (p50, p95, p99)
# - trace ID (for request correlation)
# - user ID (for per-user cost attribution)
# - session ID (for multi-turn conversations)

from langfuse.decorators import observe, langfuse_context

@observe(name="generate-order-summary")
async def generate_summary(order_id: str, user_id: str):
    langfuse_context.update_current_trace(
        user_id=user_id,
        metadata={"order_id": order_id}
    )
    ...
```

### Alerts to configure
- p95 latency > 5s → page on-call
- Error rate > 2% → page on-call
- Daily cost > budget threshold → Slack alert
- Token usage spike > 3× moving average → Slack alert

## Guardrails

```python
# Input guardrails (run before LLM)
from llm_guard import scan_prompt
from llm_guard.input_scanners import PromptInjection, Toxicity, Secrets

sanitized, results, is_valid = scan_prompt(
    scanners=[PromptInjection(), Toxicity(), Secrets()],
    prompt=user_input,
)
if not is_valid:
    raise GuardrailViolation(results)

# Output guardrails (run after LLM)
from llm_guard.output_scanners import Relevance, Sensitive, NoRefusal

output, results, is_valid = scan_output(
    scanners=[Relevance(), Sensitive(), NoRefusal()],
    prompt=user_input,
    output=llm_response,
)
```

## Cost Governance

```
Daily cost report:
  Total spend by model tier
  Top 10 users by token consumption
  Cost per feature/endpoint
  Projected monthly spend

Levers to reduce cost:
  1. Route simple tasks to Haiku (biggest lever)
  2. Cache repeated prompts (semantic cache with Redis + embeddings)
  3. Compress prompts (remove whitespace, use abbreviations in system prompts)
  4. Reduce max_tokens to realistic output length
  5. Batch requests where latency is non-critical
```

## Output Format

For LLMOps tasks:
1. Model recommendation with cost/latency reasoning
2. Code for the specific SDK (Anthropic, OpenAI, etc.) — not pseudocode
3. Eval script the user can run immediately
4. Cost estimate at expected volume
