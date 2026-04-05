---
name: llmops
description: >
  Use this skill for LLMOps tasks: prompt engineering, model versioning (MLflow/DVC),
  model serving (FastAPI/Triton), feature store patterns, A/B testing for models,
  LLM observability (Langfuse/Phoenix/Helicone), guardrails and safety, fine-tuning
  and RLHF guidance, LLM evaluation frameworks, and AI cost optimisation.
  Triggers: "prompt engineering", "fine-tuning", "RLHF", "model versioning", "MLflow",
  "model serving", "A/B test model", "LLM observability", "Langfuse", "guardrails",
  "LLM evaluation", "evals", "AI cost", "token optimisation", "model deployment",
  "inference", "LLMOps".
compatibility: "Claude Code (claude.ai, Claude Desktop) — requires bash/filesystem access"
version: "1.0.0"
---

# LLMOps Skill

## Why this skill exists

LLM systems fail in ways traditional software doesn't: silent quality regressions,
prompt injection, hallucinations, runaway costs, and non-deterministic behaviour.
This skill provides **production-grade LLM engineering patterns** — from prompt
design through eval pipelines, observability, guardrails, and cost control.

---

## 0. LLM stack audit

```bash
# Detect LLM libraries in use
cat package.json requirements.txt 2>/dev/null | python3 - <<'EOF'
import sys, re
text = sys.stdin.read()
libs = ['openai','anthropic','langchain','llamaindex','llama-index',
        'langfuse','phoenix','helicone','trulens','guidance',
        'transformers','sentence-transformers','torch','litellm']
for lib in libs:
    if lib.lower() in text.lower():
        print(f"Found: {lib}")
EOF

# Find all LLM call sites
grep -rn "client\.chat\|anthropic\.messages\|openai\.\|ChatOpenAI\|ChatAnthropic\|llm\.invoke" \
  --include="*.py" --include="*.ts" . | grep -v "node_modules\|test" | head -20

# Check for hardcoded prompts (should be in prompt registry)
grep -rn "system.*prompt\|SYSTEM_PROMPT\|system_message" \
  --include="*.py" --include="*.ts" . | grep -v "node_modules" | head -20
```

---

## 1. Prompt engineering

### Prompt template registry
```python
# prompts/registry.py — centralise all prompts for versioning + testing
from dataclasses import dataclass
from typing import Optional

@dataclass
class PromptTemplate:
    name: str
    version: str
    system: str
    user: str
    model: str = "claude-sonnet-4-20250514"
    max_tokens: int = 1024
    temperature: float = 0.0   # deterministic for most tasks

    def render(self, **kwargs) -> dict:
        return {
            "system": self.system.format(**kwargs),
            "user":   self.user.format(**kwargs),
        }

PROMPTS = {
    "extract_entities_v2": PromptTemplate(
        name="extract_entities",
        version="2.0.0",
        system="""You are a precise information extraction assistant.
Extract structured data from the provided text.
Return ONLY valid JSON matching the schema. No markdown, no explanation.
Schema: {schema}""",
        user="Extract entities from:\n\n{text}",
        temperature=0.0,
    ),

    "summarise_v1": PromptTemplate(
        name="summarise",
        version="1.0.0",
        system="You are a concise summarisation assistant. Target length: {target_length} words.",
        user="Summarise:\n\n{document}",
        temperature=0.3,
        max_tokens=512,
    ),
}
```

### Prompt engineering patterns
```python
# Pattern 1: XML tags for structured output (Claude-specific)
EXTRACTION_PROMPT = """
Extract the following from the customer message.

<customer_message>
{message}
</customer_message>

Return your response in this exact XML format:
<extraction>
  <intent>refund|support|billing|other</intent>
  <sentiment>positive|neutral|negative</sentiment>
  <urgency>high|medium|low</urgency>
  <summary>One sentence summary</summary>
</extraction>

Return ONLY the XML. No other text.
"""

# Pattern 2: Chain-of-thought for complex reasoning
COT_PROMPT = """
Analyse the following support ticket and determine the correct routing.

<ticket>{ticket_content}</ticket>

Think step by step:
1. What is the primary issue?
2. What product/feature is affected?
3. What is the customer's technical level?
4. What team should handle this?

After thinking, provide your final answer as JSON:
{"team": "...", "priority": "...", "reasoning": "..."}
"""

# Pattern 3: Few-shot examples
FEW_SHOT_PROMPT = """
Classify customer sentiment as positive, neutral, or negative.

Examples:
Input: "Your product completely changed my workflow!"
Output: positive

Input: "I can't figure out how to export my data."
Output: neutral

Input: "This is the third time I've had to contact support for the same issue."
Output: negative

Now classify:
Input: {customer_message}
Output:"""
```

---

## 2. LLM client wrapper (production)

```python
# llm/client.py — production wrapper with retry, timeout, observability
import anthropic
import logging
import time
from typing import Optional
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

logger = logging.getLogger(__name__)

class LLMClient:
    def __init__(self, api_key: str, langfuse_client=None):
        self.client = anthropic.Anthropic(api_key=api_key, timeout=60.0)
        self.langfuse = langfuse_client

    @retry(
        stop=stop_after_attempt(3),
        wait=wait_exponential(multiplier=1, min=4, max=30),
        retry=retry_if_exception_type((anthropic.RateLimitError, anthropic.APIConnectionError)),
        reraise=True,
    )
    async def complete(
        self,
        system:     str,
        user:       str,
        model:      str = "claude-sonnet-4-20250514",
        max_tokens: int = 1024,
        temperature:float = 0.0,
        trace_name: Optional[str] = None,
        **kwargs,
    ) -> LLMResponse:
        start = time.perf_counter()
        trace = self.langfuse.trace(name=trace_name) if self.langfuse and trace_name else None

        try:
            response = self.client.messages.create(
                model=model,
                max_tokens=max_tokens,
                temperature=temperature,
                system=system,
                messages=[{"role": "user", "content": user}],
                **kwargs,
            )
            latency = time.perf_counter() - start

            result = LLMResponse(
                content=response.content[0].text,
                input_tokens=response.usage.input_tokens,
                output_tokens=response.usage.output_tokens,
                model=model,
                latency_ms=int(latency * 1000),
            )

            # Track cost
            cost = self._estimate_cost(model, result.input_tokens, result.output_tokens)
            logger.info("LLM call", extra={"model": model, "input_tokens": result.input_tokens,
                "output_tokens": result.output_tokens, "latency_ms": result.latency_ms, "cost_usd": cost})

            if trace:
                trace.generation(name=trace_name, model=model, input=user,
                    output=result.content, usage={"input": result.input_tokens, "output": result.output_tokens})

            return result

        except anthropic.APIStatusError as e:
            logger.error("LLM API error", extra={"status_code": e.status_code, "message": str(e)})
            raise

    def _estimate_cost(self, model: str, input_tokens: int, output_tokens: int) -> float:
        # Prices per million tokens (update as pricing changes)
        PRICES = {
            "claude-sonnet-4-20250514": {"input": 3.0, "output": 15.0},
            "claude-haiku-4-5-20251001": {"input": 0.25, "output": 1.25},
        }
        p = PRICES.get(model, {"input": 3.0, "output": 15.0})
        return (input_tokens * p["input"] + output_tokens * p["output"]) / 1_000_000
```

---

## 3. LLM evaluation (evals)

```python
# evals/eval_runner.py
import json
from dataclasses import dataclass
from typing import Callable
import anthropic

@dataclass
class EvalCase:
    id:       str
    input:    dict
    expected: str | dict
    tags:     list[str] = None

@dataclass
class EvalResult:
    case_id:  str
    passed:   bool
    score:    float
    actual:   str
    expected: str
    reason:   str

class EvalRunner:
    def __init__(self, llm_client: LLMClient):
        self.client = llm_client

    async def run_suite(self, cases: list[EvalCase], prompt_fn: Callable, scorer: Callable) -> EvalReport:
        results = []
        for case in cases:
            prompt = prompt_fn(**case.input)
            actual = await self.client.complete(**prompt)
            score  = await scorer(expected=case.expected, actual=actual.content)
            results.append(EvalResult(case_id=case.id, passed=score >= 0.8,
                score=score, actual=actual.content, expected=str(case.expected), reason=""))

        passed = sum(1 for r in results if r.passed)
        return EvalReport(results=results, pass_rate=passed/len(results), total=len(results))

# LLM-as-judge scorer (for open-ended outputs)
async def llm_judge_scorer(expected: str, actual: str, criteria: str = "") -> float:
    client = anthropic.Anthropic()
    response = client.messages.create(
        model="claude-haiku-4-5-20251001",
        max_tokens=128,
        system=f"""You are an objective evaluator. Score the response 0.0-1.0.
Criteria: {criteria or 'accuracy and completeness'}
Return ONLY a JSON: {{"score": 0.0-1.0, "reason": "..."}}""",
        messages=[{"role": "user", "content": f"Expected:\n{expected}\n\nActual:\n{actual}"}],
    )
    data = json.loads(response.content[0].text)
    return data["score"]
```

```bash
# Run eval suite
python -m pytest evals/ -v --tb=short 2>/dev/null

# Compare eval results across prompt versions
python evals/compare.py --baseline v1.0 --candidate v2.0 2>/dev/null
```

---

## 4. Langfuse observability

```python
# llm/observability.py
from langfuse import Langfuse
from langfuse.decorators import langfuse_context, observe

langfuse = Langfuse(
    public_key=os.environ["LANGFUSE_PUBLIC_KEY"],
    secret_key=os.environ["LANGFUSE_SECRET_KEY"],
    host=os.environ.get("LANGFUSE_HOST", "https://cloud.langfuse.com"),
)

# Decorator-based tracing
@observe(name="classify_ticket")
async def classify_ticket(ticket_text: str, user_id: str) -> TicketClassification:
    langfuse_context.update_current_observation(
        input={"ticket": ticket_text},
        metadata={"user_id": user_id},
    )

    response = await llm_client.complete(
        system=PROMPTS["classify_ticket"].system,
        user=ticket_text,
        model="claude-haiku-4-5-20251001",
    )

    result = TicketClassification.model_validate_json(response.content)

    langfuse_context.update_current_observation(
        output=result.model_dump(),
        usage={"input": response.input_tokens, "output": response.output_tokens},
    )
    return result
```

---

## 5. Guardrails & safety

```python
# guardrails/input.py
import re

class InputGuardrails:
    PROMPT_INJECTION_PATTERNS = [
        r"ignore previous instructions",
        r"disregard your (system )?prompt",
        r"you are now (a )?different",
        r"pretend you are",
        r"forget everything",
        r"jailbreak",
        r"<\|.*\|>",          # token injection attempt
    ]

    PII_PATTERNS = {
        "credit_card": r"\b\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\b",
        "ssn":         r"\b\d{3}-\d{2}-\d{4}\b",
        "email":       r"\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Z|a-z]{2,}\b",
    }

    def check_injection(self, text: str) -> tuple[bool, str]:
        for pattern in self.PROMPT_INJECTION_PATTERNS:
            if re.search(pattern, text, re.IGNORECASE):
                return True, f"Potential prompt injection: {pattern}"
        return False, ""

    def redact_pii(self, text: str) -> str:
        for pii_type, pattern in self.PII_PATTERNS.items():
            text = re.sub(pattern, f"[{pii_type.upper()}_REDACTED]", text)
        return text

    def validate(self, user_input: str) -> GuardrailResult:
        injected, reason = self.check_injection(user_input)
        if injected:
            return GuardrailResult(allowed=False, reason=reason)

        if len(user_input) > 50_000:
            return GuardrailResult(allowed=False, reason="Input too long")

        cleaned = self.redact_pii(user_input)
        return GuardrailResult(allowed=True, cleaned_input=cleaned)

# Output guardrails
class OutputGuardrails:
    def check(self, output: str, expected_format: str = "any") -> GuardrailResult:
        if expected_format == "json":
            try: json.loads(output)
            except: return GuardrailResult(allowed=False, reason="Output is not valid JSON")

        # Detect hallucinated URLs
        urls = re.findall(r"https?://\S+", output)
        for url in urls:
            if not self._is_trusted_domain(url):
                return GuardrailResult(allowed=True, warning=f"Output contains unverified URL: {url}")

        return GuardrailResult(allowed=True)
```

---

## 6. Fine-tuning guidance

```python
# Fine-tuning data preparation
import json
from pathlib import Path

def prepare_fine_tune_dataset(examples: list[dict], output_path: str):
    """
    Format examples for OpenAI fine-tuning (JSONL).
    Each example: {"messages": [{"role": "system"/"user"/"assistant", "content": "..."}]}
    """
    valid, invalid = [], []

    for ex in examples:
        # Validate format
        if not all(k in ex for k in ["input", "output"]):
            invalid.append(ex)
            continue

        formatted = {
            "messages": [
                {"role": "system",    "content": ex.get("system", "You are a helpful assistant.")},
                {"role": "user",      "content": ex["input"]},
                {"role": "assistant", "content": ex["output"]},
            ]
        }
        # Count tokens (rough estimate)
        total_chars = sum(len(m["content"]) for m in formatted["messages"])
        if total_chars > 16000 * 4:  # ~16k tokens
            invalid.append({"reason": "too_long", **ex})
            continue
        valid.append(formatted)

    with open(output_path, "w") as f:
        for item in valid:
            f.write(json.dumps(item) + "\n")

    print(f"Valid: {len(valid)}  Invalid: {len(invalid)}  Written: {output_path}")
    return valid

# Fine-tuning quality checks
def audit_dataset(jsonl_path: str):
    examples = [json.loads(l) for l in Path(jsonl_path).read_text().splitlines()]
    print(f"Examples: {len(examples)}")

    assistant_lengths = [len(ex["messages"][-1]["content"]) for ex in examples]
    print(f"Avg output length: {sum(assistant_lengths)/len(assistant_lengths):.0f} chars")

    # Check for duplicates
    seen = set()
    dupes = sum(1 for ex in examples if (k := ex["messages"][1]["content"]) in seen or seen.add(k))
    print(f"Duplicate inputs: {dupes}")

    # Minimum recommended dataset sizes by use case
    print("\nRecommended minimums:")
    print("  Classification: 50+ examples")
    print("  Extraction: 100+ examples")
    print("  Generation (style): 500+ examples")
```

---

## 7. Model versioning (MLflow)

```python
# mlops/model_registry.py
import mlflow
from mlflow.tracking import MlflowClient

mlflow.set_tracking_uri(os.environ["MLFLOW_TRACKING_URI"])
client = MlflowClient()

def register_prompt_version(name: str, prompt_template: dict, eval_results: dict):
    with mlflow.start_run(run_name=f"prompt-{name}"):
        # Log prompt as artifact
        mlflow.log_dict(prompt_template, "prompt.json")

        # Log eval metrics
        mlflow.log_metrics({
            "eval_pass_rate":    eval_results["pass_rate"],
            "avg_latency_ms":    eval_results["avg_latency_ms"],
            "avg_cost_per_call": eval_results["avg_cost"],
        })

        # Log parameters
        mlflow.log_params({
            "model":       prompt_template["model"],
            "temperature": prompt_template["temperature"],
            "max_tokens":  prompt_template["max_tokens"],
        })

        # Register model
        run_id = mlflow.active_run().info.run_id
        model_uri = f"runs:/{run_id}/prompt"
        mlflow.register_model(model_uri, name)

def promote_to_production(name: str, version: int):
    client.transition_model_version_stage(
        name=name, version=version, stage="Production",
        archive_existing_versions=True,
    )
```

---

## 8. Cost monitoring and optimisation

```python
# llmops/cost_tracker.py
from dataclasses import dataclass, field
from collections import defaultdict
import time

# Pricing per 1M tokens (USD) — update as providers change pricing
MODEL_PRICING = {
    "claude-sonnet-4-20250514":    {"input": 3.00,  "output": 15.00},
    "claude-haiku-4-5-20251001":   {"input": 0.25,  "output": 1.25},
    "gpt-4o":                      {"input": 2.50,  "output": 10.00},
    "gpt-4o-mini":                 {"input": 0.15,  "output": 0.60},
}

class CostTracker:
    def __init__(self):
        self.calls = defaultdict(list)

    def record(self, model: str, input_tokens: int, output_tokens: int, feature: str):
        price = MODEL_PRICING.get(model, {"input": 3.0, "output": 15.0})
        cost = (input_tokens * price["input"] + output_tokens * price["output"]) / 1_000_000
        self.calls[feature].append({"model": model, "cost": cost,
            "input_tokens": input_tokens, "output_tokens": output_tokens, "ts": time.time()})

    def report(self) -> dict:
        summary = {}
        for feature, calls in self.calls.items():
            summary[feature] = {
                "total_cost":    sum(c["cost"] for c in calls),
                "call_count":    len(calls),
                "avg_cost":      sum(c["cost"] for c in calls) / len(calls),
                "total_tokens":  sum(c["input_tokens"] + c["output_tokens"] for c in calls),
            }
        return summary

# Cost optimisation strategies
OPTIMISATION_STRATEGIES = {
    "model_routing": """
Route by complexity:
  Simple classification/extraction → claude-haiku (10× cheaper than sonnet)
  Complex reasoning/generation    → claude-sonnet
  One-off analysis, internal tools → claude-sonnet or opus

def route_model(task_complexity: str) -> str:
    return {
        'simple':  'claude-haiku-4-5-20251001',
        'medium':  'claude-sonnet-4-20250514',
        'complex': 'claude-opus-4-6',
    }[task_complexity]
""",
    "prompt_compression": """
Remove redundant context. Target: reduce input tokens by 20-40%.
- Summarise long documents before passing to LLM
- Remove examples once the model performs well (fine-tune instead)
- Use references instead of repeating the same context

from transformers import pipeline
summariser = pipeline("summarization", model="facebook/bart-large-cnn")
short_context = summariser(long_doc, max_length=200, min_length=50)[0]['summary_text']
""",
    "caching": """
Cache deterministic calls (temperature=0, same input).
Cache TTL by call type:
  Product descriptions: 24h
  FAQ answers: 1h
  User-specific:   skip cache
""",
}
```

---

## 9. A/B testing models / prompts

```python
# ab/router.py — traffic splitting for model experiments
import hashlib

class ABRouter:
    def __init__(self, experiments: list[dict]):
        # experiments = [{"name": "sonnet_v2", "weight": 10, "config": {...}}, ...]
        self.experiments = experiments
        self.total_weight = sum(e["weight"] for e in experiments)

    def get_variant(self, user_id: str) -> dict:
        # Deterministic — same user always gets same variant
        hash_val = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
        bucket = hash_val % self.total_weight
        cumulative = 0
        for exp in self.experiments:
            cumulative += exp["weight"]
            if bucket < cumulative:
                return exp
        return self.experiments[-1]

# Usage
router = ABRouter([
    {"name": "control",   "weight": 80, "config": {"model": "claude-haiku-4-5-20251001",  "prompt_version": "v1"}},
    {"name": "treatment", "weight": 20, "config": {"model": "claude-sonnet-4-20250514", "prompt_version": "v2"}},
])

variant = router.get_variant(request.user_id)
result  = await llm_client.complete(model=variant["config"]["model"], ...)
track_event("llm_call", {"variant": variant["name"], "latency": result.latency_ms, "cost": result.cost})
```
