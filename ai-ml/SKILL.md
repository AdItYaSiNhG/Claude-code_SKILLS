---
name: ai-ml
description: AI/ML engineering skills covering LLMOps, RAG pipelines, agent systems, and ML observability. Use for production LLM system design, vector store integration, prompt management, evaluation frameworks, and agentic workflow orchestration.
license: Internal enterprise use
---

Three sub-skills cover the AI/ML surface area:

| Sub-skill file | Invoke when… |
|---|---|
| `llmops.md` | deploying, monitoring, or governing LLMs in production |
| `rag.md` | building or tuning retrieval-augmented generation pipelines |
| `agents.md` | designing or debugging multi-step agent systems and tool use |

## When multiple sub-skills apply

- New AI feature from scratch → `rag.md` (retrieval) + `agents.md` (orchestration) + `llmops.md` (deployment)
- Production quality issue → `llmops.md` first (evals, observability)
- Knowledge base search → `rag.md` only

## Enterprise Context

Enterprise AI systems fail on: hallucination at scale, prompt injection, missing evals, and uncontrolled costs. These skills treat all four as first-class concerns, not afterthoughts.
