# Contributing to flow-ai-tools

This repository contains Markdown-only plugin content for the Flow blockchain Claude Code plugin marketplace. All contributions are to `.md` files, `marketplace.json`, or `plugin.json` — there is no code to build or compile.

## Adding a pitfall entry

Before writing a new pitfall, grep the existing skill tree for the symptom or the underlying mechanism:

```bash
grep -ri "your symptom keywords" plugins/flow-dev/skills/
```

If a matching entry already exists, do NOT duplicate it. Add a short cross-reference instead:

```
> See canonical treatment in [filename](path).
> This entry is a context-specific summary; updates to the underlying behavior should land in the canonical file first.
```

Updates to underlying behavior (a new error mode, a new CLI version) should land in the CANONICAL file first, with cross-refs remaining short pointers. This prevents the "maintenance bomb" pattern where one factual change requires editing N files in sync.

When in doubt about which file is canonical, see [MAINTENANCE.md](MAINTENANCE.md).
