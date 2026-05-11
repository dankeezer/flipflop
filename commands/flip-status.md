Show FlipFlop progress statistics for the Obsidian vault at YOUR_VAULT_PATH.

## Step 1 — Gather Stats

Run the following script:

```
python3 -c "
import os, re, glob
from datetime import datetime, date

vault = 'YOUR_VAULT_PATH'
daily = re.compile(r'^\d{4}-\d{2}-\d{2}\.md$')
excluded_folders = ['Archive', 'Templates', 'Meta']  # update for your vault
excluded_files = []
default_stale_days = 365
today = date.today()

flopped = []
stale = []
flipped = []
eligible = []
excluded = {'daily': 0, 'folder': 0, 'archived': 0, 'checklist': 0}

for path in glob.glob(vault + '/**/*.md', recursive=True) + glob.glob(vault + '/*.md'):
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
            flopped_match = re.search(r'^Flopped:\s*\"?(\d{4}-\d{2}-\d{2})\"?', content, re.MULTILINE)
            threshold_match = re.search(r'^stale_after_days:\s*(\d+)', content, re.MULTILINE)
            threshold = int(threshold_match.group(1)) if threshold_match else default_stale_days
            reference_date = date.fromisoformat(flopped_match.group(1)) if flopped_match else date.fromtimestamp(mtime)
            if (today - reference_date).days >= threshold:
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
stale.sort()    # oldest first
flipped.sort()  # oldest first

total = len(flopped) + len(stale) + len(flipped) + len(eligible)
pct = round(len(flopped) / total * 100) if total > 0 else 0
bar_filled = round(pct / 5)
bar = '█' * bar_filled + '░' * (20 - bar_filled)

print('---STATS---')
print(f'TOTAL:{total}')
print(f'FLOPPED:{len(flopped)}')
print(f'STALE:{len(stale)}')
print(f'FLIPPED:{len(flipped)}')
print(f'ELIGIBLE:{len(eligible)}')
print(f'PCT:{pct}')
print(f'BAR:{bar}')
print('---RECENT_FLOPS---')
for mtime, title in flopped[:5]:
    dt = datetime.fromtimestamp(mtime).strftime('%Y-%m-%d')
    print(f'{dt}|{title}')
print('---STALE---')
for mtime, title in stale[:5]:
    dt = datetime.fromtimestamp(mtime).strftime('%Y-%m-%d')
    print(f'{dt}|{title}')
print('---DRAFTS---')
for mtime, title in flipped[:5]:
    dt = datetime.fromtimestamp(mtime).strftime('%Y-%m-%d')
    print(f'{dt}|{title}')
print('---EXCLUDED---')
print(f'daily:{excluded[\"daily\"]}')
print(f'folder:{excluded[\"folder\"]}')
print(f'archived:{excluded[\"archived\"]}')
print(f'checklist:{excluded[\"checklist\"]}')
"
```

## Step 2 — Present the Stats

Parse the output and display a clean summary in this format:

---

**FlipFlop Status — YYYY-MM-DD**

`[bar]` PCT% complete

| Status | Count |
|--------|-------|
| Flopped (done) | N |
| Stale (needs re-flip) | N |
| Flipped (drafts) | N |
| Eligible (waiting) | N |
| **Total in play** | **N** |

**Recently flopped** (last 5):
- YYYY-MM-DD — Note Title
- ...

**Stale notes** (N total, oldest first):
- YYYY-MM-DD — Note Title
- ... up to 5

**Drafts in progress** (N total, oldest first):
- YYYY-MM-DD — Note Title
- ... up to 5

**Excluded from pool:**
- N daily notes
- N archived notes
- N folder-excluded notes
- N completed checklists

---

If there are no recent flops, say so: "No notes flopped yet."
If there are no stale notes, omit that section.
If there are no drafts in progress, omit that section.
If the percentage is 0%, add an encouraging note: "Ready to start — run /flip to begin."
If the percentage is 100%, congratulate the user.

## Step 3 — Offer Next Action

Based on the stats, suggest what to do next:

- If there are flipped drafts: "You have N draft(s) in progress — run /flop to finalize the most recent."
- If stale > 0: "N stale note(s) — run /flip to revisit the oldest one."
- If eligible > 0: "N notes waiting — run /flip to enrich the next one."
- If eligible = 0 and flipped = 0 and stale = 0: "All notes are flopped. Nothing left to flip."
