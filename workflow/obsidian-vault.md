---
name: obsidian-vault
description: >
  Search, create, and manage notes in an Obsidian vault with wikilinks and index
  notes. Use when the user wants to write to their Obsidian vault, search their
  notes, create a new note with proper wikilinks, or build an index note for a topic.
  Triggers: "obsidian", "write to my vault", "create a note", "add to my notes",
  "search my vault", "obsidian vault", "wikilinks", "index note", "daily note",
  "zettelkasten".
source: adapted from mattpocock/skills
version: "1.0.0"
---

# Obsidian Vault Skill

## Vault conventions

Obsidian uses Markdown files with `[[wikilinks]]`. Good notes are:
- **Atomic** — one idea per note
- **Linked** — connected to related notes with `[[wikilinks]]`
- **Evergreen** — written to be useful indefinitely, not time-stamped
- **Titled clearly** — the title IS the note (no "Notes on X" — just "X")

---

## Step 0 — Locate the vault

```bash
# Common vault locations — ask the user if none found
VAULT=""
for candidate in \
    "$HOME/Documents/Obsidian" \
    "$HOME/Obsidian" \
    "$HOME/Notes" \
    "$HOME/Documents/Notes" \
    "$HOME/Library/Mobile Documents/iCloud~md~obsidian/Documents"; do
  if [ -d "$candidate" ]; then
    VAULT="$candidate"
    echo "Found vault: $VAULT"
    ls "$VAULT" | head -10
    break
  fi
done

if [ -z "$VAULT" ]; then
  echo "Vault not found. Ask the user for the path."
fi
```

---

## Step 1 — Search the vault

```bash
# Full-text search
VAULT="$HOME/Documents/Obsidian"  # set to actual vault path
QUERY="your search term"

grep -rn "$QUERY" "$VAULT" --include="*.md" \
  | grep -v ".obsidian\|.trash" \
  | head -20

# Find notes by title
find "$VAULT" -name "*.md" | grep -v ".obsidian\|.trash" \
  | xargs grep -l "$QUERY" 2>/dev/null | head -10

# Find all notes linking to a specific note
grep -rn "\[\[$QUERY\]\]" "$VAULT" --include="*.md" \
  | grep -v ".obsidian" | head -20
```

---

## Step 2 — Create a new note

```bash
VAULT="$HOME/Documents/Obsidian"
NOTE_TITLE="Your Note Title"
NOTE_PATH="$VAULT/${NOTE_TITLE}.md"

# Check if it already exists
if [ -f "$NOTE_PATH" ]; then
  echo "Note already exists: $NOTE_PATH"
  cat "$NOTE_PATH"
else
  cat > "$NOTE_PATH" << EOF
# ${NOTE_TITLE}

## Summary
[One paragraph summary of the main idea]

## Key concepts
- [[Related Note 1]]
- [[Related Note 2]]

## Details
[The actual content]

## References
- [Source or link]

---
Tags: #topic #category
Created: $(date +%Y-%m-%d)
EOF
  echo "Created: $NOTE_PATH"
fi
```

---

## Step 3 — Create an index / MOC (Map of Content)

An index note collects wikilinks to all notes on a topic:

```bash
VAULT="$HOME/Documents/Obsidian"
TOPIC="TypeScript"
INDEX_PATH="$VAULT/Index - ${TOPIC}.md"

# Auto-generate from existing notes
{
  echo "# Index: $TOPIC"
  echo ""
  echo "## Notes on this topic"
  find "$VAULT" -name "*.md" | grep -v ".obsidian\|.trash\|Index" \
    | xargs grep -l "$TOPIC" 2>/dev/null \
    | while read f; do
        name=$(basename "$f" .md)
        echo "- [[$name]]"
      done
  echo ""
  echo "---"
  echo "Updated: $(date +%Y-%m-%d)"
} > "$INDEX_PATH"

echo "Created index: $INDEX_PATH"
cat "$INDEX_PATH"
```

---

## Step 4 — Add a daily note entry

```bash
VAULT="$HOME/Documents/Obsidian"
DAILY_DIR="$VAULT/Daily"
mkdir -p "$DAILY_DIR"
TODAY=$(date +%Y-%m-%d)
DAILY_PATH="$DAILY_DIR/$TODAY.md"

if [ ! -f "$DAILY_PATH" ]; then
  cat > "$DAILY_PATH" << EOF
# $TODAY

## Today's focus
- 

## Notes
- 

## Done
- 

## Links
EOF
fi

# Append an entry
echo "" >> "$DAILY_PATH"
echo "### $(date +%H:%M) — [Entry title]" >> "$DAILY_PATH"
echo "[Content]" >> "$DAILY_PATH"

echo "Updated: $DAILY_PATH"
```

---

## Step 5 — Wikilink conventions

When writing notes, always:
- Link to existing notes with `[[exact note title]]`
- Create a link even if the target doesn't exist yet — Obsidian treats it as a future note
- Use `[[Note Title|display text]]` for aliased links
- Tag with `#tag` at the bottom, not in the body
- Use `## ` headers to structure, not bold titles

```markdown
# Dependency Injection

Dependency injection is a pattern where [[Inversion of Control]] is applied
at the dependency level. Instead of a class creating its own [[Repository]] instance,
the repository is passed in via the [[Constructor Injection]] pattern.

This makes the class testable — you can pass an [[In-Memory Repository]] in tests.

Related: [[Service Locator]], [[Factory Pattern]], [[Clean Architecture]]

Tags: #patterns #testing #architecture
```
