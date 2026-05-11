# FlipFlop — Design Document

**Version 1.3**

FlipFlop is a progressive wiki-building system for Obsidian vaults. It uses a conversational AI assistant to transform rough, unstructured notes into polished Wikipedia-style reference entries through a structured interview process. The system is LLM-agnostic and requires only that the AI has read and write access to the vault's markdown files.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Core Concepts](#core-concepts)
4. [Tag Lifecycle](#tag-lifecycle)
5. [Eligibility Rules](#eligibility-rules)
6. [The Flip Command](#the-flip-command)
7. [The Flop Command](#the-flop-command)
8. [The Flip-Status Command](#the-flip-status-command)
9. [Writing Principles](#writing-principles)
10. [Note Structure](#note-structure)
11. [Interview Guidelines](#interview-guidelines)
12. [Link Pass](#link-pass)
13. [Implementation Notes](#implementation-notes)
14. [Example](#example)

**Changes in v1.3:** Link pass fidelity/performance threshold (500 notes), index file spec, stale write cap (20/run), glob duplication fix.

**Changes in v1.2:** Stale tag lifecycle, `Flopped` frontmatter date, per-note `stale_after_days`, re-flip mode in `/flip`, staleness check and stale stats in `/flip-status`, question sequencing guidance, content type filter, session length guidance, LLM compatibility note.

---

## Overview

A personal knowledge vault accumulates notes over time — quick captures, rough drafts, placeholder stubs, half-finished thoughts. Most of these never get cleaned up. FlipFlop addresses this by making note enrichment feel like a conversation rather than a writing task.

The workflow is simple:

- **`/flip`** — the AI picks a random unfinished note, opens it, reads whatever's there, and interviews you about it one question at a time. When it has enough, it writes a clean wiki-style entry and saves it back to the file.
- **`/flop`** — the AI finds the most recently drafted note and promotes it to finalized status.

Over time, the vault accumulates a body of well-written, consistently structured reference entries — a personal Wikipedia built from conversations.

---

## Prerequisites

- An **Obsidian vault** (a folder of `.md` files with YAML frontmatter)
- An **AI assistant** with read and write access to the vault's files
- Obsidian notes should use YAML frontmatter with a `tags` field

No plugins are required. No database. No special Obsidian configuration.

---

## Core Concepts

### Notes as the Unit of Work

Each note in the vault is a candidate for enrichment. Notes can be anything: a product you own, a place you've been, a project you're working on, a concept you want to document, a person, an event. The system makes no assumptions about content type — the interview adapts to whatever the note is about.

### Progressive Enrichment

FlipFlop doesn't try to enrich everything at once. Each `/flip` session handles one note. Over many sessions, the vault gradually fills in. Notes that have already been enriched are skipped automatically.

### Preserved Original Content

The original content of a note is never discarded. After enrichment, the new wiki entry sits above the original content, separated by a horizontal rule. This means the enrichment is always reversible and the raw source material is always accessible.

---

## Tag Lifecycle

FlipFlop uses three tags to track note status:

| Tag | Meaning |
|-----|---------|
| *(no tag)* | Eligible — not yet enriched |
| `flipped` | Draft in progress — interview complete, entry written, pending review |
| `flopped` | Finalized — reviewed and accepted |
| `flopped` + `stale` | Finalized but outdated — eligible for re-flip |

The standard lifecycle is:

```
[untagged] → (flip) → [flipped] → (flop) → [flopped]
```

Over time, finalized notes may become outdated. When a note has been flopped for longer than its staleness threshold, the system adds `stale` as an additional tag. Stale notes re-enter the eligible pool:

```
[flopped] → (threshold elapsed) → [flopped + stale] → (flip) → [flipped + stale] → (flop) → [flopped]
```

Notes about this lifecycle:

- A note tagged `flipped` is immediately excluded from future `/flip` picks, preventing duplicate work even if the session is interrupted.
- A note tagged `flopped` (without `stale`) is excluded — it is considered current.
- `stale` is always additive — a stale note carries both `flopped` and `stale`. The `flopped` tag is removed when re-flip begins.
- If a re-flip session is interrupted, the note will carry both `flipped` and `stale`. `/flip-status` correctly surfaces this in the drafts column. `/flop` clears both tags when finalizing.
- Tags other than `flipped`, `flopped`, and `stale` are ignored by FlipFlop and left untouched.

### Staleness Threshold

The staleness threshold defaults to **365 days** from the date the note was last flopped. This is tracked via a `Flopped` frontmatter field written by `/flop`:

```yaml
Flopped: "2026-05-10"
```

Individual notes can override the threshold with an optional `stale_after_days` frontmatter field:

```yaml
stale_after_days: 180
```

A note about a contractor's pricing might warrant 180 days; a note about a plant species might warrant 1825. If the field is absent, the default of 365 applies. `/flip-status` is responsible for reading these fields and stamping `stale` on notes that have crossed their threshold.

---

## Eligibility Rules

A note is eligible for `/flip` if it meets **all** of the following:

1. It is a `.md` file anywhere in the vault
2. Its filename does **not** match the daily note pattern `YYYY-MM-DD.md`
3. It is **not** inside excluded folders (configure these per vault — typically archive, template, and meta folders)
4. Its frontmatter does **not** contain `flipped` in the tags list
5. Its frontmatter does **not** contain `flopped` in the tags list — **unless** it also contains `stale`, in which case it is eligible for re-flip
6. Its frontmatter does **not** contain `archived` in the tags list
7. It is **not** a purely completed checklist (all checklist items checked, none unchecked) — these are task lists, not reference notes

Selection from the eligible pool should be random. This prevents bias toward recently created or alphabetically early notes.

---

## The Flip Command

### Step 1 — Select and Open

Pick a random eligible note from the vault. Open it in Obsidian (if the integration supports it). Read its full content.

Immediately update the note's frontmatter tags before doing anything else — this prevents duplicate work if the session is interrupted:

- **Standard flip** (untagged note): add `flipped`
- **Re-flip** (note tagged `flopped + stale`): remove `flopped`, add `flipped`, keep `stale`

### Step 2 — Orient the User

Tell the user which note was picked and summarize whatever is already there. Keep it brief. If the note carries `stale`, flag it as a revisit. Examples:

> Flipped: **Hotel Frederick** — has an address and a few lines about a stay there. Empty otherwise.

> Flipped: **Sous Vide Circulator** — just a model number and a link. No other content.

> Flipped: **Q3 Website Redesign** — has a rough outline and some stakeholder names. No current status.

> Re-flip: **Acme Plumbing** — flopped 2025-01-14, marked stale. Existing entry covers contact info and two jobs. Let's update it.

### Step 3 — Interview

Ask questions one at a time. Do not present a list of questions. Wait for each answer before asking the next.

Adapt the questions to the note type (see [Interview Guidelines](#interview-guidelines)). For re-flips, lead with "What's changed since this was last written?" before asking follow-up questions.

Keep asking until you have enough material to write a solid entry. When ready, say so and ask if there's anything else before writing.

### Step 4 — Write the Entry

Write a Wikipedia-style wiki entry following the [Writing Principles](#writing-principles) below.

Write the file automatically — do not ask for approval before writing. The user can review after.

### Step 5 — Link Pass

Get all note names in the vault. For each concept mentioned in the draft, check whether a note exists that is genuinely about that concept. Add wikilinks (`[[Note Name]]`) where a clear match exists.

Rules:
- Never use aliased links (`[[Note Name|display text]]`) — always use the exact note name, rewriting the surrounding sentence if needed
- If two notes have similar names, read both briefly before deciding which to link
- If a concept deserves a note but none exists, suggest it to the user — do not create it automatically

### Step 6 — Save the File

Write the enriched note using the [Note Structure](#note-structure) below. The original content is preserved verbatim after the divider.

### Step 7 — Confirm

Open the note in Obsidian. Tell the user the note has been saved as a draft (`#flipped`). When they're happy with it, they can run `/flop` to finalize.

---

## The Flop Command

Flop is intentionally minimal.

### Step 1 — Find the Note

Find the most recently modified note in the vault that has `flipped` in its frontmatter tags.

### Step 2 — Promote

In the note's frontmatter:
- Replace `flipped` with `flopped` in the tags list
- Remove `stale` from the tags list if present
- Write or update the `Flopped` field with today's date: `Flopped: "YYYY-MM-DD"`

No confirmation needed — act immediately.

### Step 3 — Confirm

Open the note in Obsidian. Tell the user which note was flopped.

---

## The Flip-Status Command

`/flip-status` reports progress across the entire vault — how many notes have been flopped, how many are in draft, and how many are still waiting. It is the only command that writes `stale` tags: as part of its scan, it checks all flopped notes against their staleness threshold and stamps any that have crossed it.

### Output

The command displays:

- A **progress bar** showing the percentage of in-play notes that have been flopped
- A **count table** breaking down flopped, flipped (draft), stale, and eligible notes
- The **5 most recently flopped notes** by file modification date
- **Drafts in progress** — total count, oldest 5 shown
- **Stale notes** — total count, oldest 5 shown
- A breakdown of **excluded notes** (daily notes, folder-excluded, archived, completed checklists)
- A **suggested next action** based on current state

Example output:

```
FlipFlop Status — 2026-05-10

██░░░░░░░░░░░░░░░░░░ 11% complete

Status              Count
Flopped (done)        22
Flipped (drafts)       3
Stale (needs re-flip)  8
Eligible (waiting)   164
Total in play        197

Recently flopped:
- 2026-05-09 — Hotel Frederick
- 2026-05-09 — Acme Sous Vide Circulator
- 2026-05-09 — Q3 Website Redesign
- 2026-05-08 — Blue Ridge Trail Notes
- 2026-05-08 — Standing Desk

Drafts in progress (3, oldest first):
- 2026-03-12 — Blue Ridge Trail Notes
- 2026-04-28 — Standing Desk
- 2026-05-10 — Conference Room Booking Policy

Stale notes (8, oldest first):
- 2024-09-22 — Rental Car Policy
- 2024-11-03 — Dr. Chen
- 2025-01-14 — Acme Plumbing
- 2025-02-28 — Hotel Frederick
- 2025-04-01 — Q3 Website Redesign

Excluded from pool:
- 31 daily notes
- 12 folder-excluded notes
- 0 archived
- 3 completed checklists

→ 3 drafts in progress — run /flop to finalize the most recent.
→ 8 stale notes — run /flip to revisit the oldest one.
→ 164 notes waiting — run /flip to enrich the next one.
```

### Staleness Check

Before reporting, `/flip-status` scans all flopped notes. For each one, it reads the `Flopped` frontmatter field and the optional `stale_after_days` field (defaulting to 365). If the elapsed time exceeds the threshold, it adds `stale` to the note's tags. This is the only place `stale` is written — no other command stamps it.

Stale notes are counted separately in the stats table and listed in their own section (oldest 5 shown, with total count). Drafts follow the same pattern. Both are sorted oldest first — the longest-waiting items are most pressing. They are included in "total in play" alongside flopped, flipped, and eligible notes.

### Implementation

Scan every `.md` file in the vault using the same eligibility logic as `/flip`. Categorize each file as flopped, stale, flipped, eligible, or excluded. Sort flopped, stale, and flipped lists by the `Flopped` date or file modification time (most recent first).

```python
import os, re, glob
from datetime import datetime, date

vault = '/path/to/vault'
daily = re.compile(r'^\d{4}-\d{2}-\d{2}\.md$')
excluded_folders = ['Archive', 'Templates', 'Meta']  # configure per vault
excluded_files = []
default_stale_days = 365

flopped, stale, flipped, eligible = [], [], [], []
excluded = {'daily': 0, 'folder': 0, 'archived': 0, 'checklist': 0}
today = date.today()

for path in glob.glob(vault + '/**/*.md', recursive=True):
    rel = os.path.relpath(path, vault)
    basename = os.path.basename(rel)

    if daily.match(basename):
        excluded['daily'] += 1
        continue
    if any(rel.startswith(e) for e in excluded_folders):
        excluded['folder'] += 1
        continue
    if basename in excluded_files:
        excluded['folder'] += 1
        continue

    try:
        content = open(path).read()
        mtime = os.path.getmtime(path)
        title = basename.replace('.md', '')

        if re.search(r'tags:.*\n(  - .*\n)*  - archived', content, re.MULTILINE):
            excluded['archived'] += 1
            continue

        is_flopped = bool(re.search(r'tags:.*\n(  - .*\n)*  - flopped', content, re.MULTILINE))
        is_flipped = bool(re.search(r'tags:.*\n(  - .*\n)*  - flipped', content, re.MULTILINE))
        is_stale = bool(re.search(r'tags:.*\n(  - .*\n)*  - stale', content, re.MULTILINE))

        items = re.findall(r'- \[(.)\]', content)
        if not is_flopped and not is_flipped and items and all(x == 'x' for x in items):
            excluded['checklist'] += 1
            continue

        if is_flopped and not is_stale:
            # Check if this note should be stamped stale
            flopped_match = re.search(r'^Flopped:\s*"?(\d{4}-\d{2}-\d{2})"?', content, re.MULTILINE)
            threshold_match = re.search(r'^stale_after_days:\s*(\d+)', content, re.MULTILINE)
            threshold = int(threshold_match.group(1)) if threshold_match else default_stale_days
            # Use Flopped field if present; fall back to file mtime for notes flopped before v1.2
            if flopped_match:
                reference_date = date.fromisoformat(flopped_match.group(1))
            else:
                reference_date = date.fromtimestamp(mtime)
            if (today - reference_date).days >= threshold:
                # Stamp stale
                new_content = re.sub(
                    r'(tags:.*\n(  - .*\n)*  - flopped)',
                    r'\1\n  - stale',
                    content, flags=re.MULTILINE
                )
                open(path, 'w').write(new_content)
                is_stale = True

        if is_flopped and is_stale:
            stale.append((mtime, title))
        elif is_flopped:
            flopped.append((mtime, title))
        elif is_flipped:
            flipped.append((mtime, title))
        else:
            eligible.append(title)
    except:
        pass

flopped.sort(reverse=True)
stale.sort(key=lambda x: x[0])  # oldest first for stale
flipped.sort(reverse=True)
total = len(flopped) + len(stale) + len(flipped) + len(eligible)
pct = round(len(flopped) / total * 100) if total > 0 else 0
```

---

## Writing Principles

FlipFlop entries follow Wikipedia's core content policies, adapted for personal knowledge vaults.

### Lead with the Subject

The opening sentence must define what the thing *is* before any personal context or framing. The subject comes first.

**Wrong:**
> The Hotel Frederick was where we stayed during our trip to Boonville.

**Right:**
> The Hotel Frederick is a boutique hotel located at 501 High Street in Boonville, Missouri.

### Neutral Point of View (NPOV)

No evaluative language, advocacy, or personal opinion. Describe facts plainly.

**Wrong:**
> The Acme Sous Vide Circulator is an excellent product with outstanding build quality.

**Right:**
> The Acme Sous Vide Circulator is a 1200-watt immersion circulator with a stainless steel housing, a touchscreen display, and Wi-Fi connectivity for app-based temperature control.

### No Original Research

Only write what the user confirmed during the interview. Do not infer, extrapolate, or fill in gaps from general knowledge about the subject. If you use an external source to verify a fact, cite it as a footnote.

### Plain, Direct Language

Short declarative sentences. Avoid hedging, padding, and narrative framing. Each sentence should convey one fact.

**Wrong:**
> It's worth noting that the project was, at least in part, a response to some concerns that had been raised by stakeholders over the course of the preceding quarter.

**Right:**
> The project was initiated in response to stakeholder concerns raised in Q2.

### Footnotes for External Sources

Any website, product page, or external document referenced during the interview should be cited as a Markdown footnote at the bottom of the entry:

```markdown
[^1]: Hotel Frederick website. https://example.com/hotel-frederick
[^2]: Acme Circulator product page. https://example.com/acme-sv1200
```

---

## Note Structure

The written file must follow this exact structure:

```markdown
---
[existing frontmatter fields, preserved exactly]
tags:
  - [existing tags, preserved]
  - flipped
---
# Note Title

[wiki entry — one or more paragraphs and ## sections]

---

## Original Content

[original note content, preserved verbatim]
```

Rules:
- If the note had no frontmatter, create it with at minimum `Created` (today's date), `Type: Note`, and `tags: [flipped]`
- The `---` divider and `## Original Content` heading are always present, even if the original note was empty
- The `# Note Title` heading uses the same name as the file (without `.md`)
- Existing frontmatter fields (aliases, links, references, custom fields) are preserved exactly as written — only the tags list is modified

---

## Interview Guidelines

Adapt questions to the note type. Below are the key questions for common note types. Ask them one at a time, in natural conversational order — not as a list.

### Question Sequencing

Follow a broad-to-specific funnel within each session:

1. **Context first** — establish what the thing is and where it fits before asking for specifics
2. **Facts before opinion** — get the objective information (what, where, when, who) before asking evaluative questions (what worked, what didn't, what would you do differently)
3. **Evaluative questions last** — once context and facts are established, close with the user's assessment or open questions

Asking "what went wrong?" before establishing what the thing is produces shallow answers. Earning the evaluative question by building context first produces richer ones.

### Object / Product
- What is this and what is it for?
- Where did it come from? (purchased, gifted, found, inherited?)
- What is its current status? (in use, in storage, broken, sold?)
- Any history or provenance worth noting?
- Is there a model number, brand, or spec worth including?

### Place
- What is this place and where is it?
- Why does it appear in your vault — personal significance, a trip, a recommendation?
- Any relevant history, context, or details about the place itself?
- Have you been there? What was the context?

### Person
- Who is this person?
- What is your relationship or connection to them?
- What context brought them into your vault?

### Project or Task
- What is this project and what is it trying to accomplish?
- What is its current status?
- Who is involved?
- What are the next steps or open questions?

### Concept or Idea
- What is this concept?
- Why does it matter to you?
- Where did you encounter it?
- What other ideas or notes does it connect to?

### General Guidance
- If the existing note content already answers a question, skip it
- If an answer raises a new question, follow it
- Stop when you have enough for a solid entry — not every field needs to be filled
- Aim for 5–8 questions per session. Answers shorten noticeably after that; wrapping up is better than pushing through fatigue
- When ready to write, say: "I think I have enough — anything else before I write?"

### Content Type and Format

Wikipedia format is appropriate for **reference content** — things the user will look up later: places, people, products, concepts, events, organizations. The lead-paragraph-first structure serves this use case well.

It is a poor fit for **project notes**, **process documentation**, and **opinion pieces**. A note about an ongoing renovation project, a recipe, or a personal reflection should not be written in NPOV encyclopedia style. When `/flip` selects a note of this type, adapt the output structure to fit the content rather than forcing it into Wikipedia format. A project note might use Scope, Status, and Next Steps sections; a process note might use numbered steps; an opinion note might use the user's own voice directly.

---

## Link Pass

After drafting the entry, scan for concepts that might have notes in the vault.

### Fidelity vs. Performance Threshold

The link pass uses different strategies depending on vault size:

**Below 500 notes** — full fidelity mode. Get all note names from the vault and pass the complete list to the AI. The AI checks every name against the draft and reads candidate notes to confirm matches. At this scale the list is small (~12KB or less) and every potential connection is worth catching.

**500 notes or above** — filtered mode. Instead of passing all note names, pre-filter the list before giving it to the AI: keep only notes whose titles share meaningful words with the draft content (exclude common words: the, a, in, of, etc.). This keeps the candidate list manageable while preserving the most relevant matches. The AI still reads candidate notes to confirm before linking — only the initial list is narrowed.

The 500-note threshold applies to total vault size, not just eligible notes.

### Process

1. Count total notes in the vault to determine which mode to use
2. Get note names (filtered or full, per above)
3. For each concept, place, person, or object mentioned in the draft, check whether any note name closely matches
4. If a match looks plausible, read the note briefly to confirm it is genuinely about the same thing
5. Add a wikilink only if you are confident the note is the right one

### Rules

- Link to the note name exactly as it appears in the filesystem — no aliases
- If a concept has multiple candidate notes with similar names, read all of them before deciding
- Do not link to the note being written (no self-links)
- If no note matches a concept that clearly deserves one, suggest a stub to the user — do not create it automatically

---

## Implementation Notes

### For LLMs with File System Access

If the AI has shell or file system access (e.g., Claude Code, a custom agent with file tools), the following operations are needed:

**Finding eligible notes:**
```python
import os, re, glob, random

vault = '/path/to/vault'
daily = re.compile(r'^\d{4}-\d{2}-\d{2}\.md$')
excluded_folders = ['Archive', 'Templates', 'Meta']  # configure per vault
excluded_files = []

eligible = []
# glob '/**/*.md' with recursive=True matches root-level files too — no second glob needed
for path in glob.glob(vault + '/**/*.md', recursive=True):
    rel = os.path.relpath(path, vault)
    if daily.match(os.path.basename(rel)):
        continue
    if any(rel.startswith(e) for e in excluded_folders):
        continue
    if os.path.basename(rel) in excluded_files:
        continue
    try:
        content = open(path).read()
        if re.search(r'tags:.*\n(  - .*\n)*  - flipped', content, re.MULTILINE):
            continue
        if re.search(r'tags:.*\n(  - .*\n)*  - archived', content, re.MULTILINE):
            continue
        is_flopped = bool(re.search(r'tags:.*\n(  - .*\n)*  - flopped', content, re.MULTILINE))
        is_stale = bool(re.search(r'tags:.*\n(  - .*\n)*  - stale', content, re.MULTILINE))
        # flopped without stale = done, skip; flopped + stale = eligible for re-flip
        if is_flopped and not is_stale:
            continue
        items = re.findall(r'- \[(.)\]', content)
        if items and all(x == 'x' for x in items):
            continue
    except:
        pass
    eligible.append(rel)

print(random.choice(eligible))
```

**Opening a note in Obsidian:**
```bash
encoded=$(python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1]))" "relative/path/to/note.md")
open "obsidian://open?vault=VaultName&file=$encoded"
```

**Finding the most recently modified `#flipped` note (for /flop):**
```python
import os, re, glob

vault = '/path/to/vault'
candidates = []
for path in glob.glob(vault + '/**/*.md', recursive=True):
    try:
        content = open(path).read()
        if re.search(r'tags:.*\n(  - .*\n)*  - flipped', content, re.MULTILINE):
            candidates.append((os.path.getmtime(path), path))
    except:
        pass
candidates.sort(reverse=True)
print(candidates[0][1] if candidates else 'NONE')
```

**Writing the `Flopped` date and clearing `stale` (for /flop):**

When promoting a note, write the `Flopped` field to frontmatter (add it if absent, update it if present), and remove the `stale` tag if present:

```python
from datetime import date

def flop_note(path):
    content = open(path).read()
    today = date.today().isoformat()

    # Update or add Flopped field
    if re.search(r'^Flopped:', content, re.MULTILINE):
        content = re.sub(r'^Flopped:.*$', f'Flopped: "{today}"', content, flags=re.MULTILINE)
    else:
        content = re.sub(r'^(tags:)', f'Flopped: "{today}"\n\\1', content, flags=re.MULTILINE)

    # Replace flipped with flopped in tags
    content = re.sub(r'(  - )flipped\n', r'\1flopped\n', content)

    # Remove stale tag if present
    content = re.sub(r'  - stale\n', '', content)

    open(path, 'w').write(content)
```

### Index File

At vault sizes above ~500 notes, reading every `.md` file on each command invocation becomes slow. FlipFlop uses an optional index file to cache note metadata and avoid full rescans.

**Location:** `{vault}/.flipflop-index.json` — add to `.gitignore` if the vault is version-controlled.

**Structure:**
```json
{
  "version": 1,
  "built_at": "2026-05-10T12:00:00",
  "notes": {
    "01 - Note Box/Hotel Frederick.md": {
      "mtime": 1746835200.0,
      "title": "Hotel Frederick",
      "status": "flopped",
      "flopped_date": "2026-05-09",
      "stale_after_days": null
    }
  }
}
```

Status values: `eligible`, `flipped`, `flopped`, `stale`, `excluded`.

**Rebuild strategy:** On each command run, `os.stat` every `.md` file (fast — no file reads). Compare stored mtimes to current mtimes. Re-read and re-categorize only files that have changed, been added, or been deleted. Write the updated index back to disk.

**Activation threshold:** Commands use the index when the vault contains 500 or more `.md` files. Below that threshold, a direct full scan is fast enough and simpler to reason about.

**Stale write cap:** When stamping `stale` tags, write at most 20 files per invocation. This prevents a batch of hundreds of simultaneous writes from conflicting with Obsidian Sync on a mature vault. Remaining eligible-for-stale notes will be stamped on subsequent runs.

```python
import json, os, re, glob
from datetime import date

INDEX_PATH = '{vault}/.flipflop-index.json'
STALE_WRITE_CAP = 20

def load_index(vault):
    try:
        return json.load(open(INDEX_PATH.format(vault=vault)))
    except:
        return {'version': 1, 'notes': {}}

def save_index(vault, index):
    index['built_at'] = date.today().isoformat()
    json.dump(index, open(INDEX_PATH.format(vault=vault), 'w'), indent=2)

def refresh_index(vault, index, excluded_folders, excluded_files, default_stale_days=365):
    today = date.today()
    current_paths = set(
        p for p in glob.glob(vault + '/**/*.md', recursive=True)
        if not any(os.path.relpath(p, vault).startswith(e) for e in excluded_folders)
        and os.path.basename(p) not in excluded_files
    )
    stored = index.get('notes', {})
    stale_writes = 0

    # Remove deleted files
    for rel in list(stored):
        if os.path.join(vault, rel) not in current_paths:
            del stored[rel]

    # Re-read changed or new files
    for path in current_paths:
        rel = os.path.relpath(path, vault)
        mtime = os.path.getmtime(path)
        if rel in stored and stored[rel]['mtime'] == mtime:
            # Check stale promotion even if file unchanged
            if stored[rel]['status'] == 'flopped' and stale_writes < STALE_WRITE_CAP:
                threshold = stored[rel].get('stale_after_days') or default_stale_days
                ref = stored[rel].get('flopped_date')
                ref_date = date.fromisoformat(ref) if ref else date.fromtimestamp(mtime)
                if (today - ref_date).days >= threshold:
                    _stamp_stale(path)
                    stored[rel]['status'] = 'stale'
                    stored[rel]['mtime'] = os.path.getmtime(path)
                    stale_writes += 1
            continue
        stored[rel] = _categorize(path, rel, mtime, today, default_stale_days,
                                   stale_writes < STALE_WRITE_CAP)
        if stored[rel]['status'] == 'stale':
            stale_writes += 1

    index['notes'] = stored
    return index
```

### LLM Compatibility

FlipFlop-enriched notes are optimized for LLM querying as well as human reading. The Wikipedia format — lead paragraph first, factual, neutral, cross-linked with `[[wikilinks]]` — produces structured, navigable text that performs well as input to LLM retrieval and synthesis workflows.

This aligns with the **LLM Wiki pattern** (Karpathy, 2025), which builds Wikipedia-style Obsidian entries from ingested documents. FlipFlop produces the same artifact via a different source: user interview rather than document ingestion. The two approaches are complementary — LLM Wiki handles existing written material; FlipFlop captures tacit knowledge that was never written down.

A vault of FlipFlop-enriched notes is directly queryable via the Obsidian MCP server or similar LLM-vault integrations, with no additional preparation.

### For LLMs without File System Access

If the AI cannot directly read and write files, FlipFlop can still be run semi-manually:

1. The user opens a note in Obsidian and pastes its content into the chat
2. The AI conducts the interview and writes the enriched entry as a chat response
3. The user copies the output back into the note and adds `flipped` to the tags manually
4. `/flop` becomes a manual tag change in Obsidian

This is lower-friction than writing from scratch, though it loses the automation of file selection and writing.

### Vault Configuration to Specify

When setting up FlipFlop for a new vault, define:

- **Vault path** — absolute path to the vault folder
- **Vault name** — as it appears in Obsidian (for the `obsidian://open` URI)
- **Excluded folders** — folders to skip during note selection (e.g. `Archive`, `Templates`, `Daily Notes`)
- **Excluded files** — specific filenames to always skip (e.g. `README.md`, `AGENTS.md`)

---

## Example

### Before (raw note: `Hotel Frederick.md`)

```markdown
---
Created: "2024-09-15"
Type: Note
tags:
  - travel
  - missouri
---
# Hotel Frederick

501 High St, Boonville MO

stayed here on Katy Trail trip
really old building
great bar downstairs
good location
```

### After flip (`Hotel Frederick.md`)

```markdown
---
Created: "2024-09-15"
Type: Note
tags:
  - travel
  - missouri
  - flipped
---
# Hotel Frederick

The Hotel Frederick is a boutique hotel located at 501 High Street in downtown Boonville, Missouri. Built in 1905 by local miller and banker Charles Augustus Sombart and named for his son Frederick, the building is considered the finest example of Romanesque Revival architecture in the region. It is listed on the National Register of Historic Places.

## Accommodations

The hotel has 33 guest rooms. The building served for several decades as the Boonville Retirement Center before closing in 1994 and falling into disrepair. A $4 million restoration by owners Bill and Maggie Haw was completed in 2007. Amenities include a restaurant and lounge on the main level and The Brick Room, a lower-level bar with live entertainment on weekends. Bike rentals are available on-site.

## Access

The hotel is located in downtown Boonville adjacent to the Katy Trail, a long-distance rail trail that follows the Missouri River. Free parking is available on-site.

[^1]: Hotel Frederick website. https://www.hotelfrederick.com

---

## Original Content

501 High St, Boonville MO

stayed here on Katy Trail trip
really old building
great bar downstairs
good location
```

### After flop

Two changes: the tag is promoted and the `Flopped` date is written:

```yaml
Flopped: "2026-05-10"    ← added
tags:
  - travel
  - missouri
  - flopped              ← was: flipped
```

---

*FlipFlop was designed for use with Claude Code but is intended to be portable to any AI assistant with vault file access. The writing principles, note structure, and tag lifecycle are the invariant core — the implementation details vary by environment.*
