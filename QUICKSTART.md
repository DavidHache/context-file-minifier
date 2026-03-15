# Quickstart

## Option 1: Fetch directly (no install)

Paste this into your AI agent's chat (Kiro, Claude Code, Cline, Cursor, Windsurf, Copilot, etc.):

```
Fetch https://raw.githubusercontent.com/DavidHache/context-file-minifier/main/context-file-minifier.md and follow its instructions to minify my AI context files
```

The agent will detect your steering/context files and minify them automatically.

## Option 2: Clone locally

```bash
git clone https://github.com/DavidHache/context-file-minifier.git
```

Then tell your agent:

```
Read context-file-minifier/context-file-minifier.md and follow its instructions to minify .kiro/steering/
```

## Option 3: Copy the instruction file into your project

```bash
curl -o context-file-minifier.md https://raw.githubusercontent.com/DavidHache/context-file-minifier/main/context-file-minifier.md
```

Then tell your agent:

```
Read context-file-minifier.md and follow its instructions to minify CLAUDE.md
```

## What to expect

1. If your target files are in an active steering directory, the agent will move them to `.min-backup/` first to avoid contamination
2. The agent runs the full pipeline (verbatim identification → redundant content elimination → deduplication → formatting removal → semantic compression → symbolic encoding)
3. Validation checks run automatically
4. Minified files are written as `<name>.min.md` alongside the originals
5. A summary report prints to console with token savings per file

## After minification

- Review the `.min.md` files to confirm they look right
- To use them as your active steering files, copy them back to the original location and rename (remove `.min.md` extension)
- To restore originals, copy from `.min-backup/` back to the original path
- Clean up `.min-backup/` when you're done

## Forking this repo?

If you fork this project, update the GitHub URLs in these files to point to your fork:

- `README.md` — Usage section fetch URLs
- `QUICKSTART.md` — All fetch/clone/curl URLs

Replace `DavidHache/context-file-minifier` with `<your-username>/context-file-minifier` throughout.
