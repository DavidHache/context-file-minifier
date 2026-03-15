# Context File Minifier

<p align="center">
  <img src="image/context-file-minifier-logo.png" alt="Context File Minifier" width="400">
</p>

A Markdown instruction file that guides an AI agent to minify AI context files (AGENTS.md, CLAUDE.md, .cursorrules, .clinerules, etc.) for maximum token reduction while preserving behavioral equivalence.

> **Note:** This project is experimental. Results vary across AI models and context file complexity. Always review minified output before replacing your originals.

## What It Is

This is not software. It's a structured prompt — a runbook an AI agent reads and follows to compress your steering files. Hand it to any capable LLM alongside a target file or directory, and it will produce a `.min.md` version with significantly fewer tokens.

## Why

Bloated context files waste tokens and can actually reduce AI task success rates. Research from ETH Zurich found that bloated context files can reduce task success rates compared to providing no repository context at all, while increasing inference cost by over 20% ([Augment Code](https://www.augmentcode.com/blog/your-agents-context-is-a-junk-drawer), [InfoQ](https://www.infoq.com/news/2026/03/agents-context-file-value-review/)). Separately, an optimized CLAUDE.md reduced tokens by 60% (4,500 to 1,800) while cutting hallucinations by 40% ([SFEIR Institute](https://institute.sfeir.com/en/claude-code/claude-code-memory-system-claude-md/optimization/)). This instruction file codifies that optimization process so any AI agent can perform it consistently.

## Important: Active Steering Isolation

When you ask your AI agent to minify its own steering files (e.g., `.kiro/steering/`, `.claude/`, `.cursorrules`), those files are actively loaded as instructions. The agent would be reading them as rules to follow while simultaneously trying to compress them — contaminating its behavior during the minification run.

The minifier handles this automatically: it detects active steering files, moves them to a `.min-backup/` directory first, then processes them from there. After the pipeline completes, you choose whether to use the minified versions or restore the originals.

## How It Works

The agent executes an 11-section pipeline in order:

0. **Active Steering Isolation** — Detect if target files are in an active steering directory; relocate to `.min-backup/` to prevent the agent from reading them as live instructions during processing
1. **Ingestion** — Accept file/directory, validate format, count tokens
2. **Verbatim Identification** — Mark protected regions (code blocks, paths, commands, API contracts) before any transformation
3. **Redundant Content Elimination** — Remove content the AI already knows (folder descriptions, tech stack from manifests, linter-enforced rules, generic best practices)
4. **Content Deduplication** — Eliminate repeated concepts within and across files
5. **Emoji & Formatting Removal** — Strip decorative tokens (emoji, emphasis markers, horizontal rules)
6. **Semantic Compression** — Convert verbose prose to terse imperatives, drop filler phrases and articles
7. **Symbolic Encoding** — Convert to symbolic notation (→ for sequences, [COND]→action, [REQ]/[OPT], @filename)
8. **Validation** — Verify behavioral equivalence (all paths, code blocks, commands, artifact counts preserved)
9. **Output Generation** — Write `<name>.min.md` (single file) or mirror under `.min/` (directory)
10. **Summary Report** — Console output with per-file token counts, savings percentages, and all optimizations grouped by strategy

## Usage

Tell your AI agent to fetch the instruction file and let it find and minify your context files:

```
Fetch https://raw.githubusercontent.com/DavidHache/context-file-minifier/main/context-file-minifier.md and follow its instructions to minify my AI context files
```

The agent will detect your steering/context files automatically based on your tool (Kiro, Claude Code, Cursor, Cline, Windsurf, etc.), run the full pipeline, and produce minified versions. See [QUICKSTART.md](QUICKSTART.md) for more options.

### Supported Formats

- Markdown (`.md`)
- Plain text (`.txt`, no extension)
- YAML front-matter + Markdown body

### Supported Context File Patterns

- `AGENTS.md`, `CLAUDE.md`, `.cursorrules`, `.clinerules`, `.windsurfrules`
- All files within `.agents/`, `.claude/`, `.cline/`, or `.kiro/steering/` directories

## Output

- Single file: `<original-name>.min.md` in the same directory
- Directory: mirrored structure under `.min/` subdirectory
- Originals are never modified
- Console summary with token savings per file and per strategy

## Key Guarantees

- All code blocks preserved byte-identical
- All file paths, command syntax, and API contracts preserved verbatim
- Output artifact count matches original
- Validation runs automatically with fix-and-retry on failures
- Idempotent — re-minifying a minified file produces identical output

## Example: Real-World Results

Run against 5 Kiro steering files (`.kiro/steering/`) from a real project:

| File | Original | Minified | Savings |
|------|----------|----------|---------|
| AWS-cli.md | 160 | 107 | 53 (33.1%) |
| coding-practices.md | 146 | 100 | 46 (31.5%) |
| KISS-principle.md | 352 | 194 | 158 (44.9%) |
| Python.md | 159 | 74 | 85 (53.5%) |
| truth-serum.md | 395 | 212 | 183 (46.3%) |
| **Total** | **1,212** | **687** | **525 (43.3%)** |

Token counts estimated via words × 1.3 fallback. Files with high prose density (truth-serum.md, KISS-principle.md, Python.md) saw the largest reductions. Validation passed on all files.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines on how to help improve the minifier.

## Acknowledgments

This project was developed with assistance from Claude Opus 4.6 in [Kiro IDE](https://kiro.dev).

<img src="image/kiro-icon.svg" alt="Built with Kiro" width="32">

## License

This project is licensed under the [MIT-0 (MIT No Attribution)](LICENSE) license.

## File Structure

```
context-file-minifier.md   # The instruction file (the deliverable)
QUICKSTART.md              # Step-by-step getting started guide
README.md                  # This file
CONTRIBUTING.md            # Contribution guidelines
LICENSE                    # MIT-0 license
.gitignore                 # Excludes wip/ and working directories
```
