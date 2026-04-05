---
name: governance-tokens
description: Claude Code token usage analysis, session cost reporting, and prompt optimisation. Use to diagnose expensive sessions, build per-user/team usage reports, select the right model tier, and compress prompts without losing quality.
---

The user wants to understand, measure, or reduce token consumption. Apply the framework below.

## Token Anatomy in Claude Code Sessions

```
Session cost = (input_tokens × input_rate) + (output_tokens × output_rate) + (cache_write_tokens × cache_write_rate) + (cache_read_tokens × cache_read_rate)

For claude-sonnet-4-6 (2025 pricing, approximate):
  Input:        $3.00 / 1M tokens
  Output:       $15.00 / 1M tokens
  Cache write:  $3.75 / 1M tokens
  Cache read:   $0.30 / 1M tokens    ← use this for repeated context
```

**Key insight**: output tokens are 5× more expensive than input. Long, verbose responses are the primary cost lever.

## Usage Profiling

### Extracting Claude Code session data
```bash
# Claude Code stores session data in ~/.claude/
# Sessions log: ~/.claude/projects/<project-hash>/sessions/

# Quick session cost summary (requires jq)
cat ~/.claude/projects/*/sessions/*.jsonl | \
  jq -s '
    [.[] | select(.type == "assistant")] |
    {
      total_sessions: length,
      input_tokens: [.[].usage.input_tokens] | add,
      output_tokens: [.[].usage.output_tokens] | add,
      cache_read_tokens: [.[].usage.cache_read_input_tokens // 0] | add,
      estimated_cost_usd: (
        ([.[].usage.input_tokens] | add) * 0.000003 +
        ([.[].usage.output_tokens] | add) * 0.000015 +
        ([.[].usage.cache_read_input_tokens // 0] | add) * 0.0000003
      )
    }
  '
```

### 5-hour session cost report
```python
#!/usr/bin/env python3
"""Generate a cost report for the last N hours of Claude Code usage."""
import json, glob, os
from datetime import datetime, timedelta
from pathlib import Path

def session_report(hours: int = 5):
    cutoff = datetime.now() - timedelta(hours=hours)
    sessions_dir = Path.home() / ".claude" / "projects"
    
    totals = {"input": 0, "output": 0, "cache_read": 0, "cache_write": 0}
    session_count = 0
    
    for jsonl in sessions_dir.glob("*/sessions/*.jsonl"):
        if datetime.fromtimestamp(jsonl.stat().st_mtime) < cutoff:
            continue
        session_count += 1
        for line in jsonl.read_text().splitlines():
            msg = json.loads(line)
            if usage := msg.get("usage", {}):
                totals["input"]       += usage.get("input_tokens", 0)
                totals["output"]      += usage.get("output_tokens", 0)
                totals["cache_read"]  += usage.get("cache_read_input_tokens", 0)
                totals["cache_write"] += usage.get("cache_write_input_tokens", 0)
    
    cost = (
        totals["input"]       * 0.000003  +
        totals["output"]      * 0.000015  +
        totals["cache_read"]  * 0.0000003 +
        totals["cache_write"] * 0.00000375
    )
    
    print(f"Sessions (last {hours}h): {session_count}")
    print(f"Input tokens:      {totals['input']:,}")
    print(f"Output tokens:     {totals['output']:,}")
    print(f"Cache read tokens: {totals['cache_read']:,}  (saved: ${totals['cache_read'] * 0.0000027:.4f})")
    print(f"Estimated cost:    ${cost:.4f}")
    print(f"Cache hit rate:    {totals['cache_read'] / max(totals['input'], 1) * 100:.1f}%")

session_report(hours=5)
```

## Model Tier Selection Guide

| Task | Recommended model | Rationale |
|---|---|---|
| Quick code completion, autocomplete | `claude-haiku-4-5-20251001` | 10× cheaper than Sonnet, sufficient for simple tasks |
| Code review, bug fixing, refactoring | `claude-sonnet-4-6` | Best quality/cost balance |
| Complex architecture, multi-file reasoning | `claude-sonnet-4-6` | Required capability level |
| Long document analysis (>50K tokens) | `claude-sonnet-4-6` | Only Sonnet has sufficient context |
| Critical production incidents | `claude-sonnet-4-6` | Quality over cost in emergencies |

**Routing rule**: default to Haiku for any task a junior developer could solve; escalate to Sonnet only when Haiku struggles.

## Prompt Optimisation

### Token reduction techniques
```
1. Remove whitespace in system prompts
   Before: "You are a helpful assistant.\n\nYou should always..."
   After:  "You are a helpful assistant. You should always..."
   Saving: ~10–15% on system prompt tokens

2. Use structured formats (XML/JSON) instead of prose instructions
   Before: "Please look at the code, then think about what it does, then..."
   After:  "<task>review</task><focus>security</focus>"
   Saving: 30–50% on instruction tokens

3. Reference, don't repeat
   Bad:  Paste entire file for every question about the same file
   Good: Use claude code's file referencing — @ mention the file once

4. Summarise conversation history
   When session > 50K tokens, ask Claude to summarise context:
   "Summarise our progress in 200 words, then we'll continue."

5. Limit output length explicitly
   Add to system prompt: "Respond concisely. Max 500 words unless asked for more."
```

### Context caching (API use)
```python
# Mark stable context for caching (system prompts, large documents)
messages=[
    {
        "role": "user",
        "content": [
            {
                "type": "text",
                "text": large_codebase_context,
                "cache_control": {"type": "ephemeral"}   # cache this block
            },
            {"type": "text", "text": user_question}
        ]
    }
]
# Cache hit saves 90% on those tokens across turns
```

## Token Budget Alerts

```python
# Add to CI or developer tooling
TOKEN_BUDGET_DAILY = 500_000   # adjust per team

current_usage = get_daily_token_usage()   # from session report above
if current_usage > TOKEN_BUDGET_DAILY * 0.8:
    send_slack_alert(
        f"⚠️ Token usage at {current_usage/TOKEN_BUDGET_DAILY:.0%} of daily budget "
        f"({current_usage:,} / {TOKEN_BUDGET_DAILY:,} tokens)"
    )
```

## Output Format

1. For a usage question: run the session report script, show real numbers
2. For an optimisation question: show before/after token counts with % saving
3. For a model selection question: use the tier table with justification
4. Always include the estimated cost in USD, not just token counts
