Show FlipFlop progress statistics for the Obsidian vault at YOUR_VAULT_PATH.

## Step 1 — Gather Stats

Run the following script:

```
python3 << 'PYEOF'
import os, re, glob, json
from datetime import datetime, date

vault = 'YOUR_VAULT_PATH'
daily = re.compile(r'^\d{4}-\d{2}-\d{2}\.md$')
excluded_folders = ['Archive', 'Templates', 'Meta']  # update for your vault
excluded_files = []
INDEX_PATH = vault + '/.flipflop-index.json'
INDEX_THRESHOLD = 500
STALE_WRITE_CAP = 20
DEFAULT_STALE_DAYS = 365
today = date.today()

def is_excluded(rel, basename):
    if daily.match(basename): return 'daily'
    if any(rel.startswith(e) for e in excluded_folders): return 'folder'
    if basename in excluded_files: return 'folder'
    return None

def parse_note(path):
    content = open(path).read()
    mtime = os.path.getmtime(path)
    if re.search(r'tags:.*\n(  - .*\n)*  - archived', content, re.MULTILINE):
        return 'archived', mtime, None, None
    is_flopped = bool(re.search(r'tags:.*\n(  - .*\n)*  - flopped', content, re.MULTILINE))
    is_flipped = bool(re.search(r'tags:.*\n(  - .*\n)*  - flipped', content, re.MULTILINE))
    is_stale  = bool(re.search(r'tags:.*\n(  - .*\n)*  - stale',   content, re.MULTILINE))
    items = re.findall(r'- \[(.)\]', content)
    if not is_flopped and not is_flipped and items and all(x == 'x' for x in items):
        return 'checklist', mtime, None, None
    fd = re.search(r'^Flopped:\s*"?(\d{4}-\d{2}-\d{2})"?', content, re.MULTILINE)
    sa = re.search(r'^stale_after_days:\s*(\d+)', content, re.MULTILINE)
    flopped_date = fd.group(1) if fd else None
    stale_after  = int(sa.group(1)) if sa else None
    if is_flopped and is_stale:  return 'stale',    mtime, flopped_date, stale_after
    if is_flopped:                return 'flopped',  mtime, flopped_date, stale_after
    if is_flipped:                return 'flipped',  mtime, None, None
    return 'eligible', mtime, None, None

def stamp_stale(path):
    try:
        content = open(path).read()
        new = re.sub(r'(tags:.*\n(  - .*\n)*  - flopped)', r'\1\n  - stale', content, flags=re.MULTILINE)
        open(path, 'w').write(new)
        return os.path.getmtime(path)
    except:
        return None

def check_stale(status, flopped_date, mtime, stale_after):
    if status != 'flopped': return False
    threshold = stale_after or DEFAULT_STALE_DAYS
    ref = date.fromisoformat(flopped_date) if flopped_date else date.fromtimestamp(mtime)
    return (today - ref).days >= threshold

all_paths = glob.glob(vault + '/**/*.md', recursive=True)
use_index = len(all_paths) >= INDEX_THRESHOLD

flopped, stale, flipped, eligible = [], [], [], []
excl = {'daily': 0, 'folder': 0, 'archived': 0, 'checklist': 0}
stale_writes = 0

if use_index:
    try:    index = json.load(open(INDEX_PATH))
    except: index = {'version': 1, 'notes': {}}
    stored = index.setdefault('notes', {})
    current_rels = {os.path.relpath(p, vault) for p in all_paths}
    for rel in list(stored):
        if rel not in current_rels: del stored[rel]

    for path in all_paths:
        rel      = os.path.relpath(path, vault)
        basename = os.path.basename(rel)
        exc = is_excluded(rel, basename)
        if exc:
            excl[exc] += 1
            stored.pop(rel, None)
            continue
        try:
            mtime = os.path.getmtime(path)
            entry = stored.get(rel)
            if entry and entry.get('mtime') == mtime:
                if stale_writes < STALE_WRITE_CAP and check_stale(
                        entry['status'], entry.get('flopped_date'), mtime, entry.get('stale_after_days')):
                    new_mtime = stamp_stale(path)
                    if new_mtime:
                        entry.update({'status': 'stale', 'mtime': new_mtime})
                        mtime = new_mtime
                        stale_writes += 1
            else:
                status, mtime, fd, sa = parse_note(path)
                if status in ('archived', 'checklist'):
                    excl['archived' if status == 'archived' else 'checklist'] += 1
                    stored.pop(rel, None)
                    continue
                if stale_writes < STALE_WRITE_CAP and check_stale(status, fd, mtime, sa):
                    new_mtime = stamp_stale(path)
                    if new_mtime:
                        status = 'stale'
                        mtime = new_mtime
                        stale_writes += 1
                stored[rel] = {'mtime': mtime, 'title': basename.replace('.md',''),
                               'status': status, 'flopped_date': fd, 'stale_after_days': sa}
            status = stored[rel]['status']
            title  = stored[rel].get('title') or basename.replace('.md','')
        except: continue
        if   status == 'flopped':  flopped.append((mtime, title))
        elif status == 'stale':    stale.append((mtime, title))
        elif status == 'flipped':  flipped.append((mtime, title))
        elif status == 'eligible': eligible.append(title)

    index['built_at'] = today.isoformat()
    json.dump(index, open(INDEX_PATH, 'w'), indent=2)

else:
    for path in all_paths:
        rel      = os.path.relpath(path, vault)
        basename = os.path.basename(rel)
        exc = is_excluded(rel, basename)
        if exc:
            excl[exc] += 1
            continue
        try:
            status, mtime, fd, sa = parse_note(path)
            if status in ('archived', 'checklist'):
                excl['archived' if status == 'archived' else 'checklist'] += 1
                continue
            if stale_writes < STALE_WRITE_CAP and check_stale(status, fd, mtime, sa):
                new_mtime = stamp_stale(path)
                if new_mtime:
                    status = 'stale'
                    mtime = new_mtime
                    stale_writes += 1
            title = basename.replace('.md','')
            if   status == 'flopped':  flopped.append((mtime, title))
            elif status == 'stale':    stale.append((mtime, title))
            elif status == 'flipped':  flipped.append((mtime, title))
            elif status == 'eligible': eligible.append(title)
        except: pass

flopped.sort(reverse=True)
stale.sort()
flipped.sort()

total = len(flopped) + len(stale) + len(flipped) + len(eligible)
pct = round(len(flopped) / total * 100) if total else 0
bar = '█' * round(pct/5) + '░' * (20 - round(pct/5))

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
    print(f'{datetime.fromtimestamp(mtime).strftime("%Y-%m-%d")}|{title}')
print('---STALE---')
for mtime, title in stale[:5]:
    print(f'{datetime.fromtimestamp(mtime).strftime("%Y-%m-%d")}|{title}')
print('---DRAFTS---')
for mtime, title in flipped[:5]:
    print(f'{datetime.fromtimestamp(mtime).strftime("%Y-%m-%d")}|{title}')
print('---EXCLUDED---')
print(f'daily:{excl["daily"]}')
print(f'folder:{excl["folder"]}')
print(f'archived:{excl["archived"]}')
print(f'checklist:{excl["checklist"]}')
PYEOF
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
