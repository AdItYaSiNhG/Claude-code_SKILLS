---
name: edit-article
description: >
  Edit and improve articles, blog posts, documentation, or technical writing by
  restructuring sections, improving clarity, tightening prose, fixing flow, and
  removing fluff — while preserving the author's voice.
  Triggers: "edit this article", "improve this post", "tighten this prose",
  "edit my writing", "proofread", "make this clearer", "fix the flow",
  "improve readability", "edit this doc", "rewrite this section".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Edit Article Skill

## Editing philosophy

The goal is to make the reader's experience better — not to impose a style,
not to show off vocabulary, not to rewrite the author's voice. Every edit
should serve: clarity, brevity, or flow. If it doesn't serve one of those three,
don't make it.

---

## Step 1 — Read the full piece first

Read without editing. Understand:
- Who is the intended audience?
- What is the single main point?
- What structure does it currently have?
- Where does the reader's attention drop off?

If the user hasn't specified an audience, ask.

---

## Step 2 — Structural review

Before touching sentences, fix structure. Good structure makes every other edit
easier:

**Check:**
- Does the opening hook earn the reader's attention in the first 2 sentences?
- Is the main point clear before paragraph 3?
- Are sections in the right order? (most important → supporting → conclusion)
- Are there sections that belong elsewhere, or that should be cut entirely?
- Does the ending land, or does it just stop?

**Output a structural plan before editing:**
```
Current structure:
  Intro (weak hook) → Background (too long) → Main point → Examples → Conclusion

Proposed structure:
  Hook with main point → Key insight → Supporting evidence → What this means for reader
  
Sections to cut: [background section — reader doesn't need this before the main point]
Sections to move: [examples before the main point to create tension]
```

Get agreement from the user before restructuring.

---

## Step 3 — Prose editing pass

Go sentence by sentence. Apply these rules in order:

**Clarity — say exactly what you mean:**
```
Before: "It is important to note that the implementation of this feature
        may potentially lead to certain performance considerations."
After:  "This feature may slow down page loads by 200ms."
```

**Brevity — cut every word that doesn't earn its place:**
```
Before: "In order to be able to achieve the desired outcome..."
After:  "To achieve..."

Before: "It is worth noting that..."
After:  [just say the thing]

Before: "Due to the fact that..."
After:  "Because..."
```

**Active voice — subject does the action:**
```
Before: "The error was thrown by the function."
After:  "The function throws an error."
```

**Concrete over abstract:**
```
Before: "This improves performance."
After:  "This reduces database queries from 47 to 3 per page load."
```

---

## Step 4 — Flow pass

Read the edited version aloud (or simulate it). Check:
- Do sentences vary in length? (monotonous = all same length)
- Do paragraphs connect? (each should follow logically from the last)
- Are transitions explicit where needed?
- Does the pace match the content? (technical sections can be slower; intros should be fast)

---

## Step 5 — Final checks

```
[ ] Opening sentence earns attention without clickbait
[ ] Main point clear within first 3 paragraphs
[ ] No section longer than it needs to be
[ ] No jargon unexplained for the target audience
[ ] No hedge words unless the uncertainty is real (maybe, perhaps, might, could)
[ ] Conclusion says something — not just "in summary, we covered X"
[ ] Code blocks (if technical) are correct and runnable
[ ] Links work (if any)
```

---

## Output format

Return the edited version in full. For significant structural changes, add a
brief change log at the top:

```
## Changes made
- Moved main point to paragraph 1 (was paragraph 4)
- Cut background section (500 words) — unnecessary for target audience
- Replaced 3 passive-voice paragraphs with active voice
- Tightened intro from 180 to 60 words
- Added concrete numbers to performance claims
```

Then the full edited article.
