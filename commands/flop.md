Finalize the most recently modified #flipped note in the Obsidian vault at YOUR_VAULT_PATH by promoting it to #flopped.

## Steps

1. Find the most recently modified note tagged `flipped`:
```
python3 -c "
import os, re, glob

vault = 'YOUR_VAULT_PATH'
candidates = []
for path in glob.glob(vault + '/**/*.md', recursive=True) + glob.glob(vault + '/*.md'):
    try:
        content = open(path).read()
        if re.search(r'tags:.*\n(  - .*\n)*  - flipped', content, re.MULTILINE):
            candidates.append((os.path.getmtime(path), path))
    except:
        pass

if not candidates:
    print('NONE')
else:
    candidates.sort(reverse=True)
    print(candidates[0][1])
"
```

2. If no note is found, tell the user there are no `#flipped` notes to finalize.

3. In the note's frontmatter, make the following changes immediately — no confirmation needed:
   - Replace `flipped` with `flopped` in the tags list
   - Remove `stale` from the tags list if present
   - Write or update the `Flopped` field with today's date: `Flopped: "YYYY-MM-DD"`

4. Open the note in Obsidian:
```
encoded=$(python3 -c "import urllib.parse, os, sys; print(urllib.parse.quote(os.path.relpath(sys.argv[1], 'YOUR_VAULT_PATH')))" "<full-path>")
open "obsidian://open?vault=YOUR_VAULT_NAME&file=$encoded"
```

5. Tell the user which note was flopped.
