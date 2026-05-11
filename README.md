# FlipFlop

FlipFlop is a progressive wiki-building system for Obsidian vaults. It uses a conversational AI assistant to transform rough, unfinished notes into polished Wikipedia-style reference entries through a structured interview process.

The workflow is simple: you run `/flip`, the AI picks a random unfinished note, reads whatever's there, and interviews you about it one question at a time. When it has enough, it writes a clean wiki-style entry and saves it back to the file. Run `/flop` to mark it done. Over time, your vault accumulates a body of well-written, consistently structured reference entries — a personal Wikipedia built from conversations.

## How it works

- **`/flip`** — picks a random eligible note, opens it, interviews you, writes the entry
- **`/flop`** — promotes the most recently drafted note from `#flipped` to `#flopped`
- **`/flip-status`** — shows vault-wide progress stats, drafts in progress, and stale notes

Notes move through a tag lifecycle: `untagged → #flipped → #flopped`. After a configurable period (default 1 year), flopped notes are stamped `#stale` and re-enter the eligible pool for a revisit interview.

See [DESIGN.md](DESIGN.md) for the full specification.

## Requirements

- An [Obsidian](https://obsidian.md) vault (a folder of `.md` files with YAML frontmatter)
- An AI assistant with read and write access to the vault's files
- Python 3 (for the vault scanning scripts)

No Obsidian plugins required. No database. No special configuration.

## Setup

Point your AI assistant at this repo and say **"set up FlipFlop for my vault"** — it will read `setup.md`, detect your vault and folder structure automatically, propose a configuration for you to confirm, and install the command files in one step.

To set up manually, copy the three files from `commands/` wherever your AI tool loads custom prompt files, then replace the two placeholders in each:
- `YOUR_VAULT_PATH` — absolute path to your vault folder
- `YOUR_VAULT_NAME` — your vault's name as it appears in Obsidian

Update the `excluded_folders` list to match your vault structure, then run `/flip-status` to verify.

## Usage

```
/flip          — enrich the next random note
/flop          — finalize the current draft
/flip-status   — check progress across the vault
```

Run `/flip` and `/flop` in alternating sessions, or batch several `/flip` sessions before flopping. The system is designed to work at whatever pace fits your habits.

## Web Usage (no file access required)

FlipFlop also works with web-based AI assistants that can't read or write files directly. The interview and writing steps are identical — the only difference is that the finished entry is presented as a text block you copy back into Obsidian manually.

**Option 1 — Persistent project context (recommended):**
Most web LLMs support a persistent workspace where you can set system-level instructions that apply to every new chat. Paste the contents of `web-flip.md` there. Every new chat in that workspace has FlipFlop ready — open a new chat, paste your note, and start the interview. (Example: Claude Projects, ChatGPT Custom GPTs, Gemini Gems.)

**Option 2 — Direct URL:**
Once this repo is public, you can point any capable web LLM at it directly:
> "Fetch the FlipFlop flip instructions from github.com/dankeezer/flipflop and flip this note for me: [paste note]"

**Option 3 — One-time paste:**
Copy the contents of `web-flip.md`, paste it into any chat session, then paste your note. Works with any LLM.

The web workflow produces the same Wikipedia-style output as the CLI version. You copy the result back into Obsidian and update the tags manually (`flipped` → `flopped`, add `Flopped: "YYYY-MM-DD"`).

## Staleness

Flopped notes don't stay flopped forever. After 365 days (configurable per-note with `stale_after_days` in frontmatter), `/flip-status` stamps a note as `#stale` and it re-enters the eligible pool. The re-flip interview focuses on what's changed since the note was last written.

## Design

The full specification — tag lifecycle, writing principles, interview guidelines, link pass, implementation notes, and a worked example — is in [DESIGN.md](DESIGN.md).

FlipFlop is LLM-agnostic. The prompt templates work with any AI assistant that can read and write files — the writing principles, tag lifecycle, and note structure are the invariant core.

## Output format

FlipFlop-enriched notes follow Wikipedia's core content policies: lead with the subject definition, neutral point of view, no original research, plain declarative sentences. This format is well-suited for LLM querying as well as human reading — a vault of enriched notes is directly queryable via the [Obsidian MCP server](https://github.com/MarkusMcNugen/mcp-obsidian) or similar integrations.
