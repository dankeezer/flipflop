Pick a random note from the Obsidian vault at YOUR_VAULT_PATH, interview the user about it one question at a time, then draft a Wikipedia-style wiki entry and write it to the file.

## Eligibility Rules

A note is eligible if it meets ALL of the following:
- Is a `.md` file somewhere in `YOUR_VAULT_PATH`
- Does NOT match the daily note pattern `YYYY-MM-DD.md`
- Is NOT inside excluded folders (see script below)
- Does NOT have `flipped` in its frontmatter tags
- Does NOT have `flopped` in its frontmatter tags — UNLESS it also has `stale`, in which case it is eligible for re-flip
- Does NOT have `archived` in its frontmatter tags
- Is NOT a purely completed checklist (all `- [x]` items, none unchecked)

To find a random eligible note, run:

```
python3 << 'PYEOF'
import os, re, glob, json, random

vault = 'YOUR_VAULT_PATH'
daily = re.compile(r'^\d{4}-\d{2}-\d{2}\.md$')
excluded_folders = ['Archive', 'Templates', 'Meta']  # update for your vault
excluded_files = []
INDEX_PATH = vault + '/.flipflop-index.json'
INDEX_THRESHOLD = 500

all_paths = glob.glob(vault + '/**/*.md', recursive=True)

if len(all_paths) >= INDEX_THRESHOLD:
    try:
        stored = json.load(open(INDEX_PATH)).get('notes', {})
        eligible = [rel for rel, e in stored.items() if e.get('status') in ('eligible', 'stale')]
        print(random.choice(eligible) if eligible else 'NONE')
    except:
        print('INDEX_MISSING')
else:
    eligible = []
    for path in all_paths:
        rel = os.path.relpath(path, vault)
        basename = os.path.basename(rel)
        if daily.match(basename): continue
        if any(rel.startswith(e) for e in excluded_folders): continue
        if basename in excluded_files: continue
        try:
            content = open(path).read()
            if re.search(r'tags:.*\n(  - .*\n)*  - flipped',  content, re.MULTILINE): continue
            if re.search(r'tags:.*\n(  - .*\n)*  - archived', content, re.MULTILINE): continue
            is_flopped = bool(re.search(r'tags:.*\n(  - .*\n)*  - flopped', content, re.MULTILINE))
            is_stale   = bool(re.search(r'tags:.*\n(  - .*\n)*  - stale',   content, re.MULTILINE))
            if is_flopped and not is_stale: continue
            items = re.findall(r'- \[(.)\]', content)
            if items and all(x == 'x' for x in items): continue
        except: pass
        eligible.append(rel)
    print(random.choice(eligible) if eligible else 'NONE')
PYEOF
```

If the script prints `INDEX_MISSING`, run `/flip-status` first to build the index, then retry.

## Steps

### 1. Pick and Open

Run the script above to select a note. Open it in Obsidian:
```
encoded=$(python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1]))" "<relative-path>")
open "obsidian://open?vault=YOUR_VAULT_NAME&file=$encoded"
```

Read the note content. Immediately update the note's frontmatter tags before doing anything else:

- **Standard flip** (untagged note): add `flipped`
- **Re-flip** (note tagged `flopped + stale`): remove `flopped`, add `flipped`, keep `stale`

### 2. Orient the User

Tell the user which note was picked and summarize whatever is already there. If the note carries `stale`, flag it as a revisit. Examples:

> Flipped: **Garrison Inn** — has an address and a few lines about a stay there. Empty otherwise.

> Re-flip: **Acme Plumbing** — flopped 2025-01-14, marked stale. Existing entry covers contact info and two jobs. Let's update it.

### 3. Interview — One Question at a Time

Ask questions one at a time. Do not present a list. Wait for each answer before asking the next.

For re-flips, lead with: "What's changed since this was last written?"

**Question sequencing:** Move broad to specific. Establish context before asking for specifics. Ask factual questions (what, where, when, who) before evaluative ones (what worked, what didn't). Save "what would you do differently?" for last.

**Session length:** Aim for 5–8 questions. Answers shorten noticeably after that — wrap up rather than push through.

Adapt questions to the note type:

- **Object/product**: what is it, what's it for, where did it come from, current status, model/brand/spec
- **Place**: what and where is it, why it matters, any relevant history or context, have you been there
- **Person**: who are they, relationship or connection, what context brought them into the vault
- **Project/task**: what is it, goal, current status, who's involved, next steps or open questions
- **Concept/idea**: what is it, why it matters, where you encountered it, what other ideas it connects to

Keep asking until you have enough for a solid entry. When ready: "I think I have enough — anything else before I write?"

### 4. Draft the Entry

**For reference content** (people, places, products, concepts, events) — write a Wikipedia-style entry:
- Lead with the subject definition: the opening sentence must say what the thing *is* before any personal framing
- Neutral point of view: no evaluative language, no opinion, describe facts plainly
- No original research: only write what the user confirmed during the interview
- Plain, direct language: short declarative sentences, no hedging or padding
- Footnotes for external sources: `[^1]: Description. https://url`

**For project notes, process documentation, or opinions** — adapt the format to the content. A project note might use Scope, Status, and Next Steps sections. A process note might use numbered steps. Do not force Wikipedia format onto content it doesn't suit.

Always include an opening paragraph (no heading) and `##` sections as the content warrants.

### 5. Link Pass

Count the total notes in the vault:
```
python3 -c "import glob; print(len(glob.glob('YOUR_VAULT_PATH/**/*.md', recursive=True)))"
```

**Below 500 notes — full fidelity:**
Get all note names and pass the complete list. Check each concept in the draft against the full list, read candidate notes to confirm.
```
find YOUR_VAULT_PATH -name "*.md" | sed 's|.*/||; s|\.md$||'
```

**500 notes or above — filtered:**
Load the index to get all note titles, then filter to those sharing meaningful words with concepts mentioned in the draft (exclude stop words: the, a, an, in, of, to, and, or, for, with, at, by). Pass only the filtered list to yourself. Still read candidate notes to confirm before linking.
```
python3 -c "
import json
stored = json.load(open('YOUR_VAULT_PATH/.flipflop-index.json')).get('notes', {})
for entry in stored.values(): print(entry.get('title',''))
"
```

Rules:
- Never use aliased links (`[[Note Name|alias]]`) — always use the exact note name, rewriting the sentence if needed
- If multiple notes have similar names, read both before deciding
- Do not self-link
- If a concept deserves a note but none exists, suggest it to the user — do not create it automatically

### 6. Write the File

Update the note with this structure:

```
---
[existing frontmatter fields preserved exactly]
tags:
  - [existing tags]
  - flipped
---
# Note Title

[wiki entry]

---

## Original Content

[original content verbatim]
```

If there was no frontmatter, create it with at minimum `Created` (today's date), `Type: Note`, and `tags: [flipped]`.

The `---` divider and `## Original Content` heading are always present, even if the original note was empty.

### 7. Open in Obsidian Again

Re-open the note so the user can see the finished draft:
```
encoded=$(python3 -c "import urllib.parse, sys; print(urllib.parse.quote(sys.argv[1]))" "<relative-path>")
open "obsidian://open?vault=YOUR_VAULT_NAME&file=$encoded"
```

### 8. Done

Tell the user the note has been saved as a draft (`#flipped`). When they're happy with it, they can run `/flop` to finalize it (`#flopped`).
