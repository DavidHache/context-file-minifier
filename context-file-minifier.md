# Context File Minifier — Instruction File

Minify AI context files for maximum token reduction while preserving behavioral equivalence.

Execute sections top-to-bottom, in order. All strategies run on every file — no partial runs, no options, no compression levels.

Pipeline: Active Steering Isolation → Ingestion → Verbatim Identification → Redundant Content Elimination → Content Deduplication → Emoji & Formatting Removal → Semantic Compression → Symbolic Encoding → Validation → Output Generation → Summary Report

### Logging Rule (applies to Sections 3–7)

Each strategy logs every change. Each entry: **Strategy**: `<strategy-name>`, **Description**: what changed, **Location**: line range. [multiple changes on same line]→log each separately. [partially modified block]→log specific parts changed, note what kept. [cross-file dedup]→include source file (removed) and authoritative file (kept).

## 0. Active Steering Isolation

AI agents auto-load context files from known steering directories (e.g., `.kiro/steering/`, `.claude/`, `.agents/`). Processing these files in-place means the agent reads them as active instructions during minification, contaminating its behavior. This section relocates target files before any processing begins.

### Detection

Check if target path (file or directory) resides within any known active steering location:

| Tool | Active steering path |
|------|---------------------|
| Kiro | `.kiro/steering/` |
| Claude Code | `.claude/` |
| Cline | `.cline/` |
| Cursor | `.cursorrules` (file) |
| Windsurf | `.windsurfrules` (file) |
| Generic | `.agents/` |

[target not inside active steering location]→skip this section, proceed to Ingestion.

### Relocation

1. Create `.min-backup/` at workspace root.
2. Move target into `.min-backup/`, preserving path relative to workspace root.
   - [target is directory]→move entire directory: `.kiro/steering/` → `.min-backup/.kiro/steering/`
   - [target is single file]→create parent dirs, move file: `.cursorrules` → `.min-backup/.cursorrules`
3. All subsequent pipeline sections (1–10) operate on files inside `.min-backup/`. Do NOT stop here — immediately proceed to Section 1 (Ingestion).
4. Output Generation (Section 9) writes `.min.md` files alongside the originals inside `.min-backup/`.

### Post-Pipeline User Instructions

After the pipeline completes, inform the user:

```
=== Steering File Relocation Notice ===

Original files moved to: .min-backup/<original-path>
Minified files written to: .min-backup/<original-path>.min.md

To use the minified versions as your active steering files:
  1. Copy the .min.md files back to their original steering location
  2. Rename them to replace the originals (remove .min.md extension)

To restore the originals:
  1. Copy files from .min-backup/<original-path> back to <original-path>

The .min-backup/ directory is yours to clean up when ready.
```

### Error Handling

| Condition | Action |
|-----------|--------|
| `.min-backup/` already exists | Warn user, prompt to confirm overwrite or abort. |
| Move fails (permissions, disk) | Return error: `Cannot relocate <path> to .min-backup/: <reason>`. Stop. |
| Target is a single file at root (e.g., `.cursorrules`) | Still relocate to `.min-backup/.cursorrules`. |

## 1. Ingestion

Accept file path or directory path.

### File Mode

1. Read file at given path.
2. Identify format. Supported:
   - Markdown (`.md`)
   - Plain text (`.txt`, no extension)
   - YAML front-matter with Markdown body (starts with `---`, YAML, closed by `---`, then Markdown)
3. Unsupported format→return error: `Unsupported format: <format-name> (<file-path>)`. Stop.
4. Missing path→return error: `File not found: <file-path>`. Stop.
5. Compute original token count using cl100k_base tokenizer. Record for summary report.

### Directory Mode

1. Recursively discover context files matching:
   - Exact filenames: `AGENTS.md`, `CLAUDE.md`, `.cursorrules`, `.clinerules`, `.windsurfrules`
   - All files within `.agents/`, `.claude/`, `.cline/`, or `.kiro/steering/` directories
2. For each discovered file, apply File Mode steps 2–5.
3. Format validation failure→log error, skip, continue.

### Auto-Discovery Mode

When no specific target path is provided (e.g., user says "minify my AI context files"):

1. Scan workspace root for all known context file locations listed in Directory Mode.
2. Collect all matches into a single file set.
3. Process as Directory Mode.

### Error Handling

| Condition | Action |
|-----------|--------|
| Path does not exist | Return error naming missing path. Stop. |
| Unsupported format | Return error naming format. Stop (file) or skip (directory). |
| Empty file | Valid. Produce empty `.min.md` with zero-token report. |
| Binary file detected | Return error: `Binary file not supported: <path>`. Stop/skip. |
| Permission denied | Return error: `Cannot read file: <path>`. Stop/skip. |
| No tokenizer access | Fall back: tokens ≈ words × 1.3. Note fallback in report. |

## 2. Verbatim Identification

Before any transformation, scan entire document and mark all protected regions. Sections 3–7 skip these — verbatim content is immutable.

### Scan Procedure

1. Walk document top-to-bottom.
2. Mark every region matching verbatim types below.
3. Record each by start/end line range.
4. Single span matching multiple types→mark once.

### Verbatim Content Types

Mark all as protected:

- Fenced code blocks (` ``` `/`~~~`, with or without language identifier)
- Indented code blocks (≥ 4 spaces or 1 tab, no other block-level context)
- Inline file paths (`src/utils/helper.ts`, `./config/.env`, `/usr/local/bin/`)
- Directory tree representations (ASCII art, indented path listings)
- Glob patterns (`**/*.ts`, `src/**/*.js`)
- Shell commands/CLI invocations — exact flags, arguments, argument order
- Build, test, deploy, environment setup commands
- Pipe chains, redirections, subshell expressions
- API contracts (URL patterns, HTTP methods, headers, body schemas, endpoint definitions, parameter lists, return types)
- Inline code spans (backtick-delimited)
- Code snippets defining required output structure
- Skeleton/boilerplate blocks agent must reproduce
- XML/HTML-style tags agent must emit (e.g., `<result>`, `<error>`, `<output>`)
- Placeholder patterns (e.g., `<original-name>.min.md`, `${VAR}`)
- Delimiter markers, sentinel strings, format tokens referenced by other instructions
- Ordered/numbered lists defining execution steps — preserve count, ordering, per-step detail
- Enumerated sub-categories within classification schemes
- Conditional logic sequences where step order affects outcome
- Nested instruction hierarchies where granularity carries behavioral meaning

### Immutability Rule

Verbatim regions frozen after this section. Sections 3–7:
1. Skip every verbatim region — no deletions, rewording, or symbolic substitution.
2. Preserve content byte-identical in final output.
3. Update line-range tracking when surrounding content removed/compressed (regions shift, content unchanged).

### Edge Cases

| Condition | Action |
|-----------|--------|
| Unclosed code fence (no closing ` ``` `) | Treat opening fence to EOF as verbatim. Log warning. |
| Ambiguous inline code vs. prose backtick | [enclosed in matching backticks]→verbatim inline code. |
| Nested code blocks (inside blockquote) | Mark innermost code block as verbatim. |

## 3. Redundant Content Elimination

Remove content AI knows from training or can observe from project environment. Skip verbatim regions. Strategy: `redundant-content-elimination`.

### Removal Targets

#### 3.1 Folder Structure Descriptions
Remove prose/ASCII-art directory layouts — agent can `ls`/`find` directly. Do NOT remove inside verbatim regions.

#### 3.2 Tech Stack from Manifests
Remove language/framework/library/version descriptions present in: `package.json`, `Cargo.toml`, `go.mod`, `requirements.txt`/`pyproject.toml`, `pom.xml`, `build.gradle`/`build.gradle.kts`

#### 3.3 Linter/Formatter-Enforced Style Rules
Remove style rules duplicating: ESLint (`.eslintrc.*`, `eslint.config.*`), Prettier (`.prettierrc.*`), rustfmt (`rustfmt.toml`), Black (`pyproject.toml [tool.black]`), gofmt/golangci-lint, Checkstyle, SpotBugs

#### 3.4 API Patterns Visible in Codebase
Remove API convention/pattern descriptions observable in existing code.

#### 3.5 Generic Best Practices
Remove generic advice AI knows from training (e.g., "write clean code", "follow SOLID", "use meaningful variable names", "handle errors appropriately", "follow DRY").

### Preserve (project-specific, not derivable)

- Naming conventions unique to project (e.g., "prefix hook files with `use-`", "suffix store files with `.store.ts`")
- File organization deviating from framework defaults
- Custom patterns differing from community norms
- Documented decisions for non-obvious approaches
- Known bugs, workarounds, fragile areas, compatibility constraints
- Architectural boundaries (e.g., "never import from `internal/` in `public/` modules")
- Performance constraints affecting implementation
- Exact build/test/lint/deploy commands
- Environment variable requirements, setup steps, CI/CD expectations

## 4. Content Deduplication

Eliminate concepts appearing more than once — within single file or across files (directory mode). Skip verbatim regions. Strategy: `content-deduplication`.

### 4.1 Single-File

1. Build concept index: distinct concepts per non-verbatim section.
2. Concept in multiple sections→keep fullest statement in most relevant section (heading/purpose most directly addresses it). Remove others.
3. [equally relevant]→keep in first occurrence.

### 4.2 Cross-File (Directory Mode)

1. After single-file dedup, build cross-file concept index.
2. Concept in multiple files→keep in most authoritative file (name/scope/purpose most directly owns it). Remove from others.
3. [ambiguous authority]→prefer broadest-scope file (e.g., `AGENTS.md` over sub-directory file).

### 4.3 Removable Patterns

- Summary/Overview/TL;DR/Recap sections restating body without new instructions→remove. Test: [deleting loses zero unique instructions]→remove.
- Illustrative examples not defining required output format→remove. Test: [not referenced by "produce output like this"/"follow this format"]→remove.
- Rationale/"why" explanations→remove unless rationale contains behavioral constraint (e.g., "use X because Y breaks under Z") or defines boundary condition not stated elsewhere.

## 5. Emoji & Formatting Removal

Strip decorative tokens with no behavioral information. Skip verbatim regions. Strategy: `emoji-formatting-removal`.

### 5.1 Emoji Removal
1. Remove all emoji from non-verbatim text (Unicode emoji ranges, ZWJ sequences, modifier sequences).
2. [emoji as bullet marker]→remove emoji, keep text, replace with `-`.
3. Do NOT touch emoji inside verbatim regions.

### 5.2 Bold/Italic Removal
Remove markers used for emphasis only. Preserve structural uses (definition list labels, table headers/cells defining schema, line-start labels like `**Strategy**: value`, `**Note:** ...`).
Decision: [changes only visual weight]→remove. [delineates label/key/structural element]→keep.

### 5.3 Horizontal Rules and Decorative Spacing
1. Remove horizontal rules (`---`, `***`, `___`) — headings delineate sections.
2. Collapse 3+ consecutive blank lines to 1.
3. Remove whitespace-only lines, tab art, repeated decorative characters (e.g., `========`, `--------` outside tables/code blocks).

### 5.4 Heading Preservation
Preserve all Markdown headings (`#` through `######`). Do not remove, re-level, or alter.

## 6. Semantic Compression

Convert verbose prose into terse imperatives. Skip verbatim regions. Strategy: `semantic-compression`.

### 6.1 Filler Phrase Removal

Remove every occurrence — keep verb/clause that follows, capitalize if starts sentence:
- "you should", "make sure to", "be sure to", "remember to" → delete, keep verb
- "it is important that", "please note that", "keep in mind that", "it is recommended that", "note that" → delete, keep clause

### 6.2 Article Dropping

Drop `a`, `an`, `the` where unambiguous.
Remove: before nouns in imperatives, in generic descriptions, purely grammatical filler.
Preserve: [removal creates singular/plural ambiguity] ("a single instance"), part of proper name ("The MIT License"), inside verbatim region, distinguishes specific referent.

### 6.3 Prose-to-Key:Value Conversion

Prose assigning values to named properties→`key: value`, one per line. Only when clearly describing named properties. Do not force on narrative/procedural text.

```
max_retry_count: 3
timeout: 30s
log_level: debug
```

### 6.4 Conditional Compression

| Verbose form | Compact form |
|-------------|-------------|
| If X then Y | X→Y |
| If X, then Y | X→Y |
| When X, Y | X→Y |
| In the case that X, do Y | X→Y |
| Unless X, do Y | ¬X→Y |
| X only if Y | Y→X |

Compress connective, not conditions/actions. [nested]→chain: X∧Y→Z

### 6.5 Technical Term Preservation

Preserve exactly — no synonyms, abbreviations, rewording: code identifiers, config keys/values, CLI flags/arguments, file names/paths, API endpoints, HTTP methods, headers, domain vocabulary, acronyms (REST, API, CLI, ORM, CI/CD), library names (React, Express, Prisma, tokio), error message strings. Restructure around terms — never alter.

## 7. Symbolic Encoding

Final transformation pass — natural language patterns→symbolic notation. Runs after Semantic Compression. Skip verbatim regions. Strategy: `symbolic-encoding`.

### 7.1 Sequential Process → Arrow Notation

Prose describing steps/flows/pipelines→arrow chains: extract concise labels, join with ` → ` in execution order.

Multi-branch — use line breaks:
```
input → validate
      → [valid] process → store
      → [invalid] reject → log error
```
Do not flatten branches. Preserve branching structure.

### 7.2 Conditionals → Bracket Notation

Extends Section 6.4 for remaining conditionals.

| Pattern | Encoding |
|---------|----------|
| When condition, do action | [condition]→action |
| Action applies only if condition | [condition]→action |
| Multiple conditions, same action | [cond1∧cond2]→action |
| Multiple conditions, different actions | [cond1]→action1, [cond2]→action2 |
| Negated condition | [¬condition]→action |

### 7.3 Required/Optional Markers

1. Obligation words ("must", "required", "shall", "mandatory")→prepend `[REQ]`, remove original.
2. Optionality words ("optional", "may", "can optionally", "if desired")→prepend `[OPT]`, remove original.
3. Place at start of line/clause. [already marked]→skip.

### 7.4 File References → @-Notation

Non-verbatim prose file references→`@filename` or `@path/to/file`. Drop "the file called"/"the file at"/"in the file". Do NOT convert inside verbatim regions.

### 7.5 No Symbol Legend

Do NOT add legend/glossary explaining `→`, `[REQ]`, `[OPT]`, `[COND]`, `@filename`. Consuming AI understands these.

## 8. Validation

After all transformations (Sections 3–7), verify behavioral equivalence. Do not proceed to Output Generation until all checks pass.

### Checks

#### 8.1 File Path Preservation
Extract every file path from original (inline, globs, directory refs). Confirm each verbatim in working document. [missing/altered]→fail.

#### 8.2 Code Block Preservation
Extract every fenced (` ``` `/`~~~`) and indented code block. Compare byte-identical. [missing/truncated/modified]→fail.

#### 8.3 Artifact Count Match
Count distinct output artifacts (file creation instructions, numbered deliverables, template definitions) in original and working document. [counts differ]→fail.

#### 8.4 Command Syntax Preservation
Extract every shell command, CLI invocation, build/test/deploy command. Confirm verbatim — exact flags, arguments, order. [missing/altered]→fail.

### Report

```
Validation Report: <file-path>
  file-paths-preserved:    PASS | FAIL
  code-blocks-preserved:   PASS | FAIL
  artifact-count-match:    PASS | FAIL
  command-syntax-preserved: PASS | FAIL
  overall:                 PASS | FAIL
```

[FAIL]→append:

```
  VIOLATION: <check-name>
    expected: <original content or count>
    source:   <file-path>, lines <start>–<end>
    issue:    <what is missing or altered>
```

### Remediation

[any check fails]:
1. Identify original content from violation detail.
2. Re-insert/restore in working document exactly as original.
3. Re-run all checks.
4. Repeat until all pass or 2 fix attempts made.
5. [still failing after 2]→proceed with failures noted in Summary Report.

## 9. Output Generation

Write minified output. Never modify originals.

### 9.1 Single File Mode
1. Write working document to `<original-name>.min.md` in same directory.
   - `docs/AGENTS.md` → `docs/AGENTS.min.md`
   - `.cursorrules` → `.cursorrules.min.md`
2. [existing output]→overwrite.

### 9.2 Directory Mode
1. Create `.min/` subdirectory at source root.
2. Mirror source structure inside `.min/`.
3. Write each minified file to mirrored path with `.min.md` extension.
   - `project/.agents/config.md` → `project/.min/.agents/config.min.md`
   - `project/CLAUDE.md` → `project/.min/CLAUDE.min.md`
4. Create intermediate directories as needed.
5. [existing `.min/`]→overwrite.

### 9.3 Invariants
- Originals remain byte-identical — read-only access only.
- Output = minified content only — no report, no log, no validation appended.
- Output encoding matches original.

## 10. Summary Report

Console only — do not write into `.min.md` files.

### 10.1 Token Counting
Use cl100k_base tokenizer. [no access]→words × 1.3, note fallback in header.

### 10.2 Report Format

```
=== Context File Minification Report ===

File: <file-path>
  Original:  <N> tokens
  Minified:  <N> tokens
  Savings:   <N> tokens (<P>%)

--- Aggregate ---
  Total Original:  <N> tokens
  Total Minified:  <N> tokens
  Total Savings:   <N> tokens (<P>%)

--- Optimizations by Strategy ---
<Strategy Name> (<count>):
  - <description> (<location>)
  ...

Validation: <PASS | FAIL> (<details>)
```

### 10.3 Per-File Section
For each file: `File: <path>` (relative), `Original: <N> tokens` (from Ingestion), `Minified: <N> tokens`, `Savings: <N> tokens (<P>%)`. Percentage = `(original - minified) / original × 100`, one decimal. Comma-separated counts (e.g., `2,450 tokens`).

### 10.4 Aggregate (Directory Mode Only)
[directory]→print after per-file entries: `Total Original` = sum per-file originals, `Total Minified` = sum per-file minified, `Total Savings` = difference, percentage = `(Total Savings / Total Original) × 100`, one decimal. [single file]→omit.

### 10.5 Optimizations by Strategy
Group by strategy, print display name and count, list each as `- <description> (<location>)`.

| Log strategy value | Display name |
|---|---|
| `redundant-content-elimination` | Redundant Content Elimination |
| `content-deduplication` | Content Deduplication |
| `emoji-formatting-removal` | Emoji & Formatting Removal |
| `semantic-compression` | Semantic Compression |
| `symbolic-encoding` | Symbolic Encoding |

[zero optimizations]→omit group.

### 10.6 Validation Status
[all passed]→`Validation: PASS (all checks passed)`. [any failed]→`Validation: FAIL (<failed check names>)`.

### 10.7 Report Exclusion
Console output only. No report content in `.min.md` files. Minified file = minified document only.
