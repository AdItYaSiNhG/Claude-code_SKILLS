---
name: governance-cost
description: Claude Code and AI API cost dashboards, per-team chargeback reports, budget alerts, and spend forecasting. Use to build finance-ready cost visibility, enforce per-team budgets, and identify cost optimisation opportunities.
---

The user needs cost visibility, reporting, or budget controls for AI tool usage. Apply the framework below.

## Cost Visibility Architecture

```
Data sources:
  Claude Code sessions  → ~/.claude/ JSONL files (developer workstations)
  Anthropic API         → /v1/usage endpoint (server-side calls)
  AWS Bedrock           → Cost Explorer with tag filters
  Azure OpenAI          → Azure Monitor metrics

Aggregation:
  Collect → Normalise → Store (PostgreSQL / BigQuery) → Dashboard (Grafana / Metabase)

Dimensions to track:
  By model tier (Haiku / Sonnet / Opus)
  By team / cost centre
  By project / repository
  By task type (code review / generation / analysis)
  By day / week / month
```

## Data Collection Agent

```python
#!/usr/bin/env python3
"""
Collects Claude Code usage from developer machines and pushes to central DB.
Run as a cron job on each developer's machine (or via MDM script).
"""
import json, os, httpx
from pathlib import Path
from datetime import date

def collect_and_ship():
    sessions_dir = Path.home() / ".claude" / "projects"
    today = date.today().isoformat()
    records = []

    for jsonl in sessions_dir.glob("*/sessions/*.jsonl"):
        project_hash = jsonl.parts[-3]
        for line in jsonl.read_text().splitlines():
            msg = json.loads(line)
            if usage := msg.get("usage"):
                records.append({
                    "date": today,
                    "user": os.environ.get("USER"),
                    "project": project_hash,
                    "model": msg.get("model", "unknown"),
                    "input_tokens": usage.get("input_tokens", 0),
                    "output_tokens": usage.get("output_tokens", 0),
                    "cache_read_tokens": usage.get("cache_read_input_tokens", 0),
                    "cost_usd": (
                        usage.get("input_tokens", 0) * 0.000003 +
                        usage.get("output_tokens", 0) * 0.000015 +
                        usage.get("cache_read_input_tokens", 0) * 0.0000003
                    ),
                })
    
    if records:
        httpx.post(
            f"{os.environ['TELEMETRY_ENDPOINT']}/claude-usage",
            json=records,
            headers={"Authorization": f"Bearer {os.environ['TELEMETRY_TOKEN']}"},
        )

collect_and_ship()
```

## Central Cost Database Schema

```sql
CREATE TABLE ai_usage (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    recorded_at   DATE NOT NULL,
    user_id       TEXT NOT NULL,
    team          TEXT NOT NULL,           -- populated via LDAP/HR lookup
    cost_centre   TEXT NOT NULL,           -- for finance chargeback
    project       TEXT,
    model         TEXT NOT NULL,
    input_tokens  INTEGER NOT NULL DEFAULT 0,
    output_tokens INTEGER NOT NULL DEFAULT 0,
    cache_read_tokens INTEGER NOT NULL DEFAULT 0,
    cost_usd      NUMERIC(10, 6) NOT NULL,
    source        TEXT NOT NULL            -- 'claude_code' | 'api' | 'bedrock'
);

-- Indexes for common query patterns
CREATE INDEX idx_usage_team_date ON ai_usage (team, recorded_at DESC);
CREATE INDEX idx_usage_user_date ON ai_usage (user_id, recorded_at DESC);
CREATE INDEX idx_usage_model ON ai_usage (model, recorded_at DESC);
```

## Cost Reports

### Weekly team report (SQL → Slack/email)
```sql
SELECT
    team,
    cost_centre,
    SUM(cost_usd)                            AS total_cost_usd,
    SUM(input_tokens + output_tokens)        AS total_tokens,
    SUM(cost_usd) / COUNT(DISTINCT user_id)  AS cost_per_user,
    COUNT(DISTINCT user_id)                  AS active_users,
    ROUND(
        SUM(CASE WHEN model LIKE '%haiku%' THEN cost_usd ELSE 0 END) /
        NULLIF(SUM(cost_usd), 0) * 100, 1
    )                                        AS haiku_pct    -- higher = more efficient
FROM ai_usage
WHERE recorded_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY team, cost_centre
ORDER BY total_cost_usd DESC;
```

### Month-to-date vs budget
```sql
SELECT
    team,
    SUM(cost_usd)                              AS mtd_spend_usd,
    b.monthly_budget_usd,
    ROUND(SUM(cost_usd) / b.monthly_budget_usd * 100, 1) AS pct_budget_used,
    b.monthly_budget_usd - SUM(cost_usd)       AS remaining_budget_usd,
    -- Forecast: extrapolate MTD to full month
    SUM(cost_usd) / EXTRACT(DAY FROM CURRENT_DATE) * 
    EXTRACT(DAY FROM DATE_TRUNC('month', CURRENT_DATE) + INTERVAL '1 month - 1 day') 
                                               AS forecasted_monthly_usd
FROM ai_usage
JOIN team_budgets b USING (team)
WHERE recorded_at >= DATE_TRUNC('month', CURRENT_DATE)
GROUP BY team, b.monthly_budget_usd
ORDER BY pct_budget_used DESC;
```

## Budget Alerts

```python
# Run daily via cron or CI job
from anthropic_cost_db import get_mtd_spend, get_budget

ALERT_THRESHOLDS = [0.70, 0.90, 1.00]  # 70%, 90%, 100%

for team in get_all_teams():
    spend = get_mtd_spend(team)
    budget = get_budget(team)
    ratio = spend / budget

    for threshold in ALERT_THRESHOLDS:
        if ratio >= threshold:
            send_slack(
                channel=f"#{team}-engineering",
                message=(
                    f"💰 AI cost alert: *{team}* is at "
                    f"*{ratio:.0%}* of monthly budget "
                    f"(${spend:.0f} / ${budget:.0f}). "
                    f"Forecast: ${forecast_monthly(team):.0f}."
                )
            )
            break  # Only send highest applicable alert
```

## Chargeback Report (Finance-ready)

```python
import pandas as pd

def generate_chargeback_report(month: str) -> pd.DataFrame:
    """
    Produces a finance-ready CSV: one row per cost centre,
    columns: cost_centre, team, model_breakdown, total_usd
    """
    df = query_db(f"""
        SELECT
            cost_centre,
            team,
            SUM(CASE WHEN model LIKE '%haiku%' THEN cost_usd END) AS haiku_usd,
            SUM(CASE WHEN model LIKE '%sonnet%' THEN cost_usd END) AS sonnet_usd,
            SUM(CASE WHEN model LIKE '%opus%' THEN cost_usd END)   AS opus_usd,
            SUM(cost_usd)                                           AS total_usd
        FROM ai_usage
        WHERE DATE_TRUNC('month', recorded_at) = '{month}'
        GROUP BY cost_centre, team
        ORDER BY total_usd DESC
    """)
    df.to_csv(f"chargeback_{month}.csv", index=False)
    return df
```

## Optimisation Opportunities (auto-detect)

```sql
-- Users spending >$X/day who could switch to Haiku
SELECT user_id, team, SUM(cost_usd) AS daily_cost_usd,
       ROUND(SUM(CASE WHEN model LIKE '%sonnet%' THEN cost_usd END) / SUM(cost_usd) * 100) AS sonnet_pct
FROM ai_usage
WHERE recorded_at = CURRENT_DATE - 1
  AND model LIKE '%sonnet%'
GROUP BY user_id, team
HAVING SUM(cost_usd) > 5   -- $5+/day threshold
ORDER BY daily_cost_usd DESC;
-- → outreach: "Consider routing routine tasks to Haiku (10× cheaper)"
```

## Output Format

1. For a cost question: show the SQL query + sample output formatted as a table
2. For a budget question: include the budget vs actuals comparison with forecast
3. For a chargeback report: produce the CSV-ready query
4. Always express costs in USD (not just tokens) — finance teams need dollars
