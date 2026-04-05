# Claude Code Skills

> A production-grade skill library for Claude Code — covering the full software
> engineering lifecycle from security audits to LLM agents, with enterprise-level
> depth across every major technology stack.

## Quick start

```bash
# Clone and point Claude Code at this repo
git clone https://github.com/AdItYaSiNhG/Claude-code_SKILLS
cd Claude-code_SKILLS

# In your project's .claude/settings.json, add this skills directory
# or copy relevant skill files into your project's .claude/skills/ folder
```

---

## Repository structure

```
Claude-code_SKILLS/
│
├── core/                    # Foundational engineering quality
│   ├── security.md          # AppSec, secret scanning, OWASP, SAST, CVE triage
│   ├── code-quality.md      # Linting, type checking, complexity, license audit, a11y
│   ├── code-review.md       # PR review, architecture review, API review, SLA tracking
│   ├── testing.md           # Unit, integration, E2E, load, contract, mutation tests
│   └── code-cleanup.md      # Tech debt, dead code, i18n, feature flags, refactoring
│
├── architecture/            # System design and structural patterns
│   ├── code-structure.md    # Clean arch, DDD, hexagonal, CQRS, event sourcing, sagas
│   ├── infra-devops.md      # Terraform, Pulumi, CDK, GitHub Actions, ArgoCD, Ansible
│   ├── directory-layout.md  # Project scaffolding for TS/Python/Go/Java, path aliases
│   └── monorepo.md          # Turborepo, Nx, Changesets, affected builds, cache strategy
│
├── frontend/                # UI engineering
│   ├── ui.md                # React, Next.js, Vue 3, SvelteKit, PWA, micro-frontends
│   ├── browser-visual-qa.md # Cross-browser, visual regression, axe-core, responsive
│   └── design-system.md     # Design tokens, cva variants, shadcn/ui, Storybook, theming
│
├── backend/                 # Server-side engineering
│   ├── api.md               # REST, GraphQL, gRPC, tRPC, WebSocket, OpenAPI, FastAPI
│   ├── database.md          # PostgreSQL, Redis, MongoDB, Elasticsearch, Kafka, dbt
│   └── language-specific.md # Node.js, Python, Go, Java, Rust, C#, Ruby idioms
│
├── ai-ml/                   # AI engineering and LLMOps
│   ├── llmops.md            # Prompt engineering, evals, fine-tuning, cost tracking
│   ├── rag.md               # RAG pipelines, vector stores, hybrid search, re-ranking
│   └── agents.md            # Tool design, ReAct loops, HITL, multi-agent, evaluation
│
├── devops/                  # Operations and reliability
│   └── observability.md     # Structured logging, OTel tracing, Prometheus, SLOs, Sentry
│
├── governance/              # Cost, compliance, and policy
│   └── token-usage.md       # Session cost reports, model selection, context budgets
│
└── workflow/                # Developer productivity and process
    ├── write-a-prd.md             # Interview → codebase explore → PRD → GitHub issue
    ├── prd-to-issues.md           # Decompose PRD into atomic GitHub issues
    ├── tdd.md                     # Red → Green → Refactor with commit discipline
    ├── git-guardrails.md          # Block force push, protect main, Claude Code hooks
    ├── setup-pre-commit.md        # Husky + lint-staged + Prettier + type check
    ├── request-refactor-plan.md   # Safe incremental refactor plans with risk assessment
    ├── triage-issue.md            # Bug investigation → root cause → TDD fix plan
    ├── improve-codebase-architecture.md  # Shallow modules, testability, coupling audit
    ├── design-an-interface.md     # Use-case-first TypeScript interface design
    ├── write-a-skill.md           # Create new Claude Code skills
    ├── edit-article.md            # Structure, clarity, brevity for technical writing
    ├── scaffold-exercises.md      # Course/workshop exercise directory generation
    └── obsidian-vault.md          # Search, create, and manage Obsidian notes
```

---

## Skill index

### Core quality (5 skills)

| Skill | What it does | Key tools |
|-------|-------------|-----------|
| [security](core/security.md) | Full AppSec audit: secrets, SAST, deps, OWASP, headers, containers | gitleaks, semgrep, bandit, tfsec |
| [code-quality](core/code-quality.md) | Lint, type check, complexity, dead code, license audit, a11y | ESLint, ruff, mypy, radon, axe-core |
| [code-review](core/code-review.md) | Multi-lens PR review: correctness, security, perf, testing | git diff, madge, gh CLI |
| [testing](core/testing.md) | Full test pyramid: unit, integration, E2E, load, contract, mutation | Jest, Playwright, k6, Pact, Stryker |
| [code-cleanup](core/code-cleanup.md) | Tech debt quantification, dead deps, feature flag removal, i18n | depcheck, vulture, knip |

### Architecture (4 skills)

| Skill | What it does | Key tools |
|-------|-------------|-----------|
| [code-structure](architecture/code-structure.md) | Clean arch, DDD, CQRS, hexagonal, saga, plugin systems | madge, semgrep |
| [infra-devops](architecture/infra-devops.md) | IaC, CI/CD pipelines, GitOps, DR runbooks, FinOps | Terraform, CDK, GitHub Actions, ArgoCD |
| [directory-layout](architecture/directory-layout.md) | Project scaffolding for 6 languages, barrel files, path aliases | find, wc |
| [monorepo](architecture/monorepo.md) | Turborepo/Nx setup, Changesets versioning, affected builds, cache | turbo, nx, pnpm |

### Frontend (3 skills)

| Skill | What it does | Key tools |
|-------|-------------|-----------|
| [ui](frontend/ui.md) | React patterns, Next.js App Router, Vue 3, SvelteKit, Core Web Vitals | Lighthouse, bundlephobia |
| [browser-visual-qa](frontend/browser-visual-qa.md) | Cross-browser, visual regression, a11y, mobile viewports | Playwright, axe-core, pa11y |
| [design-system](frontend/design-system.md) | Token architecture, cva variants, shadcn/ui, Storybook, dark mode | tailwind, storybook, chromatic |

### Backend (3 skills)

| Skill | What it does | Key tools |
|-------|-------------|-----------|
| [api](backend/api.md) | REST, GraphQL, gRPC, tRPC, WebSocket, OpenAPI, auth, rate limiting | zod, fastify, pothos |
| [database](backend/database.md) | PostgreSQL tuning, Redis patterns, MongoDB, Kafka, dbt, Airflow | EXPLAIN ANALYZE, pgvector |
| [language-specific](backend/language-specific.md) | Production patterns for Node, Python, Go, Java, Rust, C#, Ruby | language-native tooling |

### AI/ML (3 skills)

| Skill | What it does | Key tools |
|-------|-------------|-----------|
| [llmops](ai-ml/llmops.md) | Prompt registry, evals, LLM observability, fine-tuning, cost tracking | Langfuse, MLflow, tenacity |
| [rag](ai-ml/rag.md) | Chunking, embeddings, pgvector/Pinecone, hybrid search, re-ranking | sentence-transformers, BM25 |
| [agents](ai-ml/agents.md) | Tool design, ReAct loops, loop detection, HITL, multi-agent, evals | Anthropic SDK, subprocess |

### DevOps (1 skill)

| Skill | What it does | Key tools |
|-------|-------------|-----------|
| [observability](devops/observability.md) | Structured logging, OTel tracing, Prometheus metrics, SLOs, Sentry | pino, OpenTelemetry, prom-client |

### Governance (1 skill)

| Skill | What it does | Key tools |
|-------|-------------|-----------|
| [token-usage](governance/token-usage.md) | Session cost reports, model selection guide, context budgets, ROI | Python scripts |

### Workflow (13 skills)

| Skill | What it does | Source |
|-------|-------------|--------|
| [write-a-prd](workflow/write-a-prd.md) | Interview → explore → PRD → GitHub issue | mattpocock/skills |
| [prd-to-issues](workflow/prd-to-issues.md) | Decompose PRD into atomic, dependency-linked issues | mattpocock/skills |
| [tdd](workflow/tdd.md) | Red → Green → Refactor with commit discipline | mattpocock/skills |
| [git-guardrails](workflow/git-guardrails.md) | Block force push, protect main/master, Claude Code hooks | mattpocock/skills |
| [setup-pre-commit](workflow/setup-pre-commit.md) | Husky + lint-staged + Prettier + type check on commit | mattpocock/skills |
| [request-refactor-plan](workflow/request-refactor-plan.md) | Phased refactor plans with risk and rollback | mattpocock/skills |
| [triage-issue](workflow/triage-issue.md) | Bug investigation → root cause → TDD-based fix plan | mattpocock/skills |
| [improve-codebase-architecture](workflow/improve-codebase-architecture.md) | Shallow modules, testability, coupling audit | mattpocock/skills |
| [design-an-interface](workflow/design-an-interface.md) | Use-case-first TypeScript interface design | mattpocock/skills |
| [write-a-skill](workflow/write-a-skill.md) | Create new skills with proper triggers and structure | mattpocock/skills |
| [edit-article](workflow/edit-article.md) | Structure, clarity, brevity for technical writing | mattpocock/skills |
| [scaffold-exercises](workflow/scaffold-exercises.md) | Course/workshop exercise directory generation | mattpocock/skills |
| [obsidian-vault](workflow/obsidian-vault.md) | Search, create, and manage Obsidian vault notes | mattpocock/skills |

---




# 🔌 Free MCP Servers for AI Agents

> A curated list of free / open-source MCP (Model Context Protocol) servers organized by development category. All entries are MIT/Apache licensed or have generous free tiers.

---

## 📑 Table of Contents

- [🎨 Frontend \& Design](#-frontend--design)
- [⚙️ Backend](#️-backend)
- [🤖 AI \& ML](#-ai--ml)
- [🔄 Workflows](#-workflows)

---

## 🎨 Frontend & Design

| Server | Language | License | Best For |
|---|---|---|---|
| [Figma MCP](https://github.com/figma/figma-developer-mcp) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | Free tier | Design-to-code: generates React/Vue/Angular from real Figma node tree |
| [Playwright MCP](https://github.com/microsoft/playwright-mcp) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | MIT | Browser automation via accessibility trees — verify UI at localhost |
| [Magic UI MCP](https://magicui.design/docs/mcp) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | Free | React + Tailwind component library — ship polished UI fast |
| [Context7 MCP](https://github.com/upstash/context7) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | Free tier | Fetches live, version-specific docs so AI never uses stale APIs |
| [Vercel MCP](https://vercel.com/docs/mcp) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | Free tier | Deploy, monitor, and manage Next.js/full-stack projects from chat |
| [Fetch MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/fetch) | ![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) | MIT | Converts web pages to LLM-friendly format — official Anthropic server |

> ⭐ **Top pick:** Figma MCP for design handoff · Playwright MCP for UI testing (12K+ GitHub stars)

---

## ⚙️ Backend

| Server | Language | License | Best For |
|---|---|---|---|
| [Supabase MCP](https://github.com/supabase-community/supabase-mcp) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | Free tier | Natural-language Postgres queries, auth, storage, and log debugging |
| [GitHub MCP](https://github.com/github/github-mcp-server) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | MIT | PRs, issues, code search, and Actions — all without leaving your editor |
| [Git MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/git) | ![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) | MIT | Local repo git ops — great for air-gapped or huge monorepos |
| [Filesystem MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/filesystem) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | MIT | Secure file read/write with access controls — official Anthropic server |
| [Kubectl MCP](https://github.com/rohitg00/kubectl-mcp-server) | ![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) | MIT | Natural-language kubectl — find crashed pods, stream logs, manage clusters |
| [FastAPI MCP](https://github.com/tadata-org/fastapi_mcp) | ![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) | Free / OSS | Exposes FastAPI endpoints as MCP tools with zero config |

> ⭐ **Top pick:** Supabase MCP for full backend control · GitHub MCP for repo automation

---

## 🤖 AI & ML

| Server | Language | License | Best For |
|---|---|---|---|
| [Run Python MCP](https://github.com/modelcontextprotocol/servers) | ![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) | Free / OSS | Sandboxed Python execution — run ML experiments and data analysis inline |
| [mcp-agent (LastMile)](https://github.com/lastmile-ai/mcp-agent) | ![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) | Free / OSS | Compose agents with map-reduce, orchestrator, and evaluator-optimizer patterns |
| [Brave Search MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/brave-search) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | Free tier | Independent search index for real-time AI grounding |
| [Sentry MCP](https://github.com/getsentry/sentry-mcp) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | Free tier | AI pulls live error traces + correlates with releases to propose fixes |
| [DuckDuckGo Search MCP](https://github.com/nickclyde/duckduckgo-mcp-server) | ![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) | MIT | Fully free web search — no API key needed, no rate-limit costs |
| [Postman MCP](https://mcp.postman.com) | ![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=flat&logo=javascript&logoColor=black) | Free tier | AI-assisted API testing and ML model endpoint validation |

> ⭐ **Top pick:** Run Python MCP for ML work · DuckDuckGo MCP for zero-cost web search

---

## 🔄 Workflows

| Server | Language | License | Best For |
|---|---|---|---|
| [Slack MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/slack) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | Free tier | 47 tools — search messages, post to channels, manage team workflows |
| [Google Calendar MCP](https://github.com/nspady/google-calendar-mcp) | ![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) | Free | Event creation, reminders, and scheduling via Google Calendar API |
| [Netlify MCP](https://github.com/netlify/netlify-mcp) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | Free tier | Manage JAMstack sites, builds, env vars, and preview deploys |
| [GitLab MCP](https://github.com/kopfrechner/gitlab-mr-mcp) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | MIT | CI/CD pipeline monitoring and MR automation for self-managed GitLab |
| [GitHub Actions MCP](https://github.com/Tiberriver256/mcp-server-github-actions) | ![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat&logo=typescript&logoColor=white) | MIT | Chat-driven CI — re-run workflows, watch statuses, debug failures |
| [Google Maps MCP](https://github.com/modelcontextprotocol/servers/tree/main/src/google-maps) | ![Python](https://img.shields.io/badge/Python-3776AB?style=flat&logo=python&logoColor=white) | Free tier | Spatial awareness for agents — POI data, delivery zones, location queries |

> ⭐ **Top pick:** Slack MCP for team automation · GitHub Actions MCP for CI/CD chat control

---

## 💡 Quick Setup Tips

```json
// claude_desktop_config.json — example multi-server setup
{
  "mcpServers": {
    "filesystem": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/your/path"]
    },
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_PERSONAL_ACCESS_TOKEN": "your-token" }
    },
    "supabase": {
      "command": "npx",
      "args": ["-y", "@supabase/mcp-server-supabase"],
      "env": { "SUPABASE_URL": "your-url", "SUPABASE_KEY": "your-key" }
    }
  }
}
```

> **Tip:** Start with 2–3 servers from your primary category. Running 10+ servers simultaneously can exhaust token budgets in longer conversations.

---

<div align="center">

**MCP is governed by the [Agentic AI Foundation](https://www.linuxfoundation.org/) under the Linux Foundation**  
Anthropic · OpenAI · Google · Microsoft · AWS are all member organizations

</div>



## Usage in Claude Code

### Option 1 — Project-level skills (recommended)
Copy the skills you need into your project's `.claude/skills/` directory:

```bash
# Copy a skill into your project
mkdir -p .claude/skills
cp path/to/Claude-code_SKILLS/core/security.md .claude/skills/
cp path/to/Claude-code_SKILLS/workflow/tdd.md .claude/skills/
```

### Option 2 — Global skills
Add this repo as a global skills source in your Claude Code config:

```json
// ~/.claude/settings.json
{
  "skillsDirectories": [
    "/path/to/Claude-code_SKILLS"
  ]
}
```

### Option 3 — Reference specific skills in prompts
```
@skills/core/security.md Run a full security audit on this repo
@skills/workflow/tdd.md  Implement the payment retry logic using TDD
@skills/ai-ml/rag.md     Build a RAG pipeline for our documentation
```

---

## Trigger reference

Claude Code reads skill descriptions to decide when to activate them. These
phrases reliably trigger each skill:

| You say... | Skill activated |
|-----------|----------------|
| "security audit", "find vulnerabilities", "scan for secrets" | core/security |
| "run code quality checks", "lint this", "check complexity" | core/code-quality |
| "review this PR", "review my code", "code review" | core/code-review |
| "write tests", "add unit tests", "E2E test" | core/testing |
| "clean up this code", "remove dead code", "tech debt" | core/code-cleanup |
| "architecture", "clean architecture", "DDD", "CQRS" | architecture/code-structure |
| "Terraform", "GitHub Actions", "deploy this", "IaC" | architecture/infra-devops |
| "project structure", "directory layout", "scaffold project" | architecture/directory-layout |
| "monorepo", "Turborepo", "Nx", "Changesets" | architecture/monorepo |
| "React component", "Next.js", "bundle size", "Core Web Vitals" | frontend/ui |
| "cross-browser", "visual regression", "a11y audit" | frontend/browser-visual-qa |
| "design system", "design tokens", "Storybook" | frontend/design-system |
| "REST API", "GraphQL", "rate limit", "API design" | backend/api |
| "PostgreSQL", "slow query", "Redis", "Kafka", "dbt" | backend/database |
| "Python pattern", "Go microservice", "Spring Boot" | backend/language-specific |
| "prompt engineering", "LLM evals", "fine-tuning", "LLMOps" | ai-ml/llmops |
| "RAG", "vector store", "embeddings", "semantic search" | ai-ml/rag |
| "agent", "tool calling", "agentic", "multi-agent" | ai-ml/agents |
| "structured logging", "Prometheus", "SLO", "incident runbook" | devops/observability |
| "token usage", "session cost", "AI spend", "model selection" | governance/token-usage |
| "write a PRD", "product requirements", "plan this feature" | workflow/write-a-prd |
| "break into issues", "decompose this spec" | workflow/prd-to-issues |
| "TDD", "test driven", "write tests first" | workflow/tdd |
| "git guardrails", "protect main branch" | workflow/git-guardrails |
| "set up pre-commit", "husky", "lint on commit" | workflow/setup-pre-commit |
| "refactor plan", "safe refactor" | workflow/request-refactor-plan |
| "triage this bug", "root cause", "investigate" | workflow/triage-issue |
| "improve architecture", "shallow modules", "reduce coupling" | workflow/improve-codebase-architecture |
| "design this interface", "API contract" | workflow/design-an-interface |
| "create a skill", "write a skill" | workflow/write-a-skill |
| "edit this article", "improve this post" | workflow/edit-article |
| "scaffold exercises", "create a course" | workflow/scaffold-exercises |
| "obsidian", "write to my vault", "create a note" | workflow/obsidian-vault |

---

## Coverage by technology

| Technology | Skills that cover it |
|-----------|---------------------|
| TypeScript / Node.js | core/*, frontend/*, backend/api, backend/language-specific, architecture/* |
| Python | core/*, backend/api, backend/language-specific, ai-ml/* |
| Go | backend/language-specific, architecture/code-structure |
| Java / Kotlin | backend/language-specific, architecture/infra-devops |
| Rust | backend/language-specific |
| React / Next.js | frontend/ui, frontend/browser-visual-qa, frontend/design-system |
| Vue / Svelte | frontend/ui |
| PostgreSQL | backend/database |
| Redis | backend/database |
| MongoDB | backend/database |
| Kafka | backend/database |
| Terraform | architecture/infra-devops |
| GitHub Actions | architecture/infra-devops |
| Docker / K8s | architecture/infra-devops, devops/observability |
| Claude / Anthropic API | ai-ml/llmops, ai-ml/rag, ai-ml/agents, governance/token-usage |
| OpenTelemetry | devops/observability |
| Prometheus / Grafana | devops/observability |

---

## Contributing

To add a new skill:

1. Identify which section it belongs to (or create a new one)
2. Follow the skill format:
   ```yaml
   ---
   name: skill-name
   description: >
     What it does. When to use it.
     Triggers: "exact phrase 1", "phrase 2".
   version: "1.0.0"
   ---
   ```
3. Include bash commands for every step that touches the filesystem
4. End with a verification step (how to confirm it worked)
5. Add to the index table in this README
6. Test by asking Claude Code to perform the task

See [workflow/write-a-skill.md](workflow/write-a-skill.md) for detailed guidance.

---

## Credits

- Core engineering skills (17)
- Workflow skills (13): adapted from [mattpocock/skills](https://github.com/mattpocock/skills)
  by Matt Pocock — original work on PRD writing, TDD, git guardrails, and developer
  workflow automation
