---
name: write-a-skill
description: >
  Create a new Claude Code skill file with proper structure, clear trigger
  description, progressive disclosure, and bundled resources. Use when the user
  wants to create a new skill, document a repeatable workflow, or package an
  AI-assisted process for reuse.
  Triggers: "write a skill", "create a skill", "new skill", "document this workflow
  as a skill", "package this as a skill", "skill template".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Write a Skill Skill

## What makes a good skill

A skill is a **reusable, self-contained instruction set** that Claude follows
to complete a specific category of task. Great skills are:

- **Specific** — one clear job, not "do everything"
- **Actionable** — concrete steps, not vague guidance
- **Tool-using** — includes bash commands, not just prose
- **Self-verifying** — includes checks that confirm the task succeeded
- **Well-triggered** — the description makes it clear exactly when to use it

---

## Step 1 — Interview the user

Ask:
1. What specific task should this skill perform?
2. When should Claude use this skill? (trigger phrases)
3. What does success look like? (how do we know it worked?)
4. Are there common failure modes to guard against?
5. Does this need any bash commands, or is it pure reasoning?

---

## Step 2 — Write the YAML front matter

```yaml
---
name: skill-name                   # kebab-case, matches filename
description: >
  One paragraph that answers: WHAT does this skill do, WHEN should it trigger,
  and WHAT are the exact trigger phrases. This is what Claude reads to decide
  whether to activate the skill.
  Triggers: "exact phrase 1", "exact phrase 2", "keyword combination".
compatibility: "Claude Code (claude.ai, Claude Desktop)"
version: "1.0.0"
---
```

**Trigger phrase rules:**
- Include 5–10 exact phrases users would naturally say
- Cover synonyms: "refactor" and "clean up" and "improve"
- Be specific enough not to trigger accidentally

---

## Step 3 — Write the skill body

Structure:
```markdown
# [Skill Name]

## Why this skill exists
[One paragraph: what problem does this solve? Why is it better than ad-hoc?]

## Step 1 — [First action]
[What to do. Include bash commands where relevant.]

## Step 2 — [Second action]
[Continue...]

## Verification
[How to confirm the task succeeded]
```

**Rules for the body:**
- Every step that involves the filesystem should have a bash command
- Include the exact output format or template where relevant
- Add a "what to do if this fails" note for any step that commonly fails
- Keep each step atomic — one thing, then verify, then next thing

---

## Step 3 — Progressive disclosure

Put the most critical information first. Claude reads skills top-to-bottom;
if the context window is tight, earlier content is more likely to be used.

Order:
1. Quick-reference (if the user just wants the answer fast)
2. Standard process (the normal happy path)
3. Edge cases and variations
4. Troubleshooting

---

## Step 4 — Write the file

```bash
# Name the file after the skill, in the right category
CATEGORY="workflow"  # core|architecture|frontend|backend|ai-ml|devops|governance|workflow
SKILL_NAME="my-new-skill"

cat > skills/${CATEGORY}/${SKILL_NAME}.md << 'EOF'
---
name: my-new-skill
description: >
  [Your description here]
  Triggers: "trigger phrase 1", "trigger phrase 2".
version: "1.0.0"
---

# [Skill Name]

## Why this skill exists
[Reason]

## Step 1 — ...
EOF
```

## Step 5 — Test the skill

Ask Claude Code to perform the task this skill covers. Check:
- Did Claude activate the skill (mentioned or followed its structure)?
- Did the output match the expected format?
- Were the bash commands correct for the current environment?
- Did the verification step confirm success?

Iterate on the description and trigger phrases until activation is reliable.
