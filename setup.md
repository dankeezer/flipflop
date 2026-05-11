Set up FlipFlop in the user's Obsidian vault.

## Step 1 — Discover

Before asking the user anything, gather the following automatically:

**Find Obsidian vaults:**
```
find ~ -name ".obsidian" -type d -maxdepth 6 2>/dev/null | sed 's|/.obsidian||'
```

**Determine command file destination** based on the tool you are running in:
- Claude Code → `.claude/commands/` inside the vault
- If unknown, note that you'll ask the user at confirmation time

**For each vault found**, read its top-level folder structure to identify likely excluded directories — folders whose names suggest they are not candidate notes (e.g. contain words like "archive", "template", "meta", "daily", "resource", "attachment", "system"):
```
ls "<vault-path>"
```

## Step 2 — Propose

Present your findings as a single confirmation block. Do not ask questions one at a time. Example format:

---

Here's what I found — let me know if anything needs changing:

| Setting | Detected value |
|---------|---------------|
| Vault path | `/Users/you/Documents/my-vault` |
| Vault name | `my-vault` |
| Excluded folders | `Archive`, `Templates`, `Daily Notes` |
| Installing commands to | `my-vault/.claude/commands/` |

Ready to install, or anything to adjust?

---

If multiple vaults were found, list them and ask which one to use before showing the table.

If the command destination is unknown (not running in a tool you recognise), add a row asking where to install: "Command destination — where does your AI tool load custom prompt files?"

## Step 3 — Install

Once the user confirms (or provides corrections), do the following:

1. Create the commands destination directory if it doesn't exist

2. Copy the three files from `commands/` in this repo to the destination, replacing placeholders as you write each file:
   - `YOUR_VAULT_PATH` → the confirmed vault path
   - `YOUR_VAULT_NAME` → the confirmed vault name
   - The `excluded_folders` list → the confirmed list, formatted as a Python list of strings

3. Run `/flip-status` to verify the installation:
```
python3 -c "
import os, glob, re
vault = '<confirmed-vault-path>'
count = len([p for p in glob.glob(vault + '/**/*.md', recursive=True) if not re.search(r'- (flipped|flopped)', open(p).read()) if True])
print(f'Found {count} candidate notes. FlipFlop is ready.')
" 2>/dev/null || echo "Setup complete — run /flip-status to see your vault stats."
```

## Step 4 — Hand Off

Tell the user:
- Which vault was configured
- Where the command files were installed
- How to start: "Run `/flip-status` to see your vault, then `/flip` to enrich your first note."
