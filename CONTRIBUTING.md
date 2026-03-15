# Contributing to Context File Minifier

Thanks for your interest in contributing. This project is a structured prompt, not traditional software, so contributions look a bit different than usual.

## Ways to Contribute

- Report issues with minification behavior (missed optimizations, broken verbatim regions, validation failures)
- Suggest new compression strategies or improvements to existing ones
- Improve documentation and examples
- Share before/after results from real-world context files
- Add support for additional context file formats or discovery patterns

## Getting Started

1. Fork the repository
2. Create a feature branch (`git checkout -b my-contribution`)
3. Make your changes
4. Test against real context files to verify behavioral equivalence is preserved
5. Commit with a clear message describing what changed and why
6. Open a pull request

## Guidelines

- Keep the instruction file lean. Every token in the minifier is a token spent before the real work starts.
- Changes to the pipeline (Sections 0–10) should preserve or improve token reduction without breaking behavioral equivalence.
- If adding a new compression strategy, include before/after examples showing the token savings.
- Verbatim preservation rules exist for a reason. If you think something should be compressible, make the case in your PR description.

## Testing Your Changes

There's no automated test suite — this is a prompt, not code. To validate changes:

1. Pick a real-world context file (AGENTS.md, CLAUDE.md, .cursorrules, etc.)
2. Run the original instruction file against it with your preferred AI agent
3. Run your modified version against the same file
4. Compare: token savings should be equal or better, and no behavioral information should be lost

## Reporting Issues

When reporting a minification issue, include:

- The original context file (or a representative snippet)
- The minified output
- What went wrong (lost information, broken code block, missed optimization, etc.)
- Which AI agent you used

## Code of Conduct

Be kind, be constructive, be respectful. We're all here to make AI context files less wasteful.

## License

By contributing, you agree that your contributions will be licensed under the MIT-0 License.
