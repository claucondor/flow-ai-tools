# AGENTS.md

This file provides guidance to AI coding agents (Claude Code, Codex, Cursor, Copilot, and others)
when working in this repository. It is loaded into agent context automatically — keep it concise.

## Overview

This repository is a Claude Code plugin marketplace for the Flow blockchain ecosystem. It ships
one plugin, `flow-dev`, containing eleven skills that provide domain knowledge for Cadence and
Flow development. Content is Markdown only — there is no code to build, compile, or test.

**Target users**: Cadence/Flow developers who install this marketplace into Claude Code to get
specialized assistance with smart contract development, auditing, querying, and deployment.

## How Skills Work

Skills use a three-level progressive disclosure system:

1. **Metadata** (~100 words) — The `name` and `description` in YAML frontmatter. Always loaded
   into the agent's context. This is how the agent decides whether to activate a skill.
2. **SKILL.md body** (~200 words) — Loaded when the skill triggers. Contains overview, quick
   start, and a navigation map pointing to reference files.
3. **Reference files** (200–300 lines each) — Loaded on demand when the agent needs detailed
   information on a specific topic.

This design keeps context efficient: metadata is always present, the skill body loads only when
relevant, and references load only when needed for the specific task.

## Repository Layout

```
.claude-plugin/marketplace.json          Marketplace catalog, registers plugins
plugins/flow-dev/
    .claude-plugin/plugin.json           Plugin metadata
    skills/<skill-name>/
        SKILL.md                         Skill entry point with YAML frontmatter
        references/                      Topic-focused markdown loaded on demand
scripts/install.sh                       One-liner installer (Flow CLI + MCP + plugin)
CLAUDE.md                                Symlink to AGENTS.md (backwards compat)
README.md                                User-facing install + skill catalog
CODEOWNERS                               PR review ownership (all paths)
```

## Maintenance Discipline

Before adding a new pitfall, anti-pattern, or common-error entry to any skill
reference: **grep first.**

```bash
grep -ri "your symptom keywords" plugins/flow-dev/skills/
```

If a matching entry exists, add a one-line cross-reference instead of
duplicating. See `CONTRIBUTING.md` for the full rule and the canonical-pointer
format.

Canonical assignments for known duplicate clusters are tracked in
`MAINTENANCE.md`. When updating a behavior, land the change in the canonical
file first; cross-refs remain short pointers.

Heading convention: `## Common Pitfalls` (not "Anti-Patterns", not "Common
Mistakes", not "Gotchas"). Exceptions: `## Known issues` (for upstream CLI
bugs, tracked in per-skill `known-bugs.md` files) and `## Cadence-Specific
Gotchas` (audit-checklist's domain-specific framing). Anti-patterns still use
`## Anti-Patterns` headers where appropriate; pitfalls are the consumer-error
prevention category specifically.

Upstream CLI bugs (distinct from developer-error pitfalls) live in their own
`known-bugs.md` files alongside affected skills, not in `## Common Pitfalls`
sections.

## Plugin and Skills

One plugin is registered in `.claude-plugin/marketplace.json`:

- **flow-dev** (`plugins/flow-dev/`) — v1.0.0, category `blockchain`

It contains exactly these thirteen skills (each has its own `SKILL.md` plus a `references/` directory):

| Skill | Reference count |
|---|---|
| `cadence-lang` | 14 |
| `cadence-tokens` | 3 |
| `cadence-audit` | 3 |
| `cadence-scaffold` | 3 |
| `cadence-testing` | 7 |
| `flow-react-sdk` | 4 |
| `flow-project-setup` | 2 |
| `flow-cli` | 5 |
| `flow-dev-setup` | 8 |
| `flow-defi` | 4 |
| `flow-tokenomics` | 5 |
| `flow-crossvm` | 7 |
| `flow-actions` | 6 |

Descriptions and trigger phrases live in each `SKILL.md` frontmatter.

## Skill Routing Guide

When a developer asks for help, use this table to determine which skill(s) to activate:

| Developer need | Primary skill | May also need |
|---|---|---|
| Write/understand Cadence code (syntax, types, patterns) | `cadence-lang` | |
| Build an NFT or FT token contract | `cadence-tokens` | `cadence-lang` |
| Review or audit existing Cadence code | `cadence-audit` | `cadence-lang` |
| Generate a new contract, transaction, or DeFi tx from scratch | `cadence-scaffold` | `cadence-lang`, `cadence-tokens` |
| Build React frontend on Flow | `flow-react-sdk` | |
| Set up a Flow project, configure flow.json, deploy | `flow-project-setup` | |
| Install dev tools (Flow CLI, emulator, VS Code, EVM tooling) | `flow-dev-setup` | `flow-project-setup` |
| Design or architect a DeFi protocol on Flow | `flow-defi` | |
| Design token economics for a Flow protocol | `flow-tokenomics` | `flow-defi`, `cadence-tokens` |
| Compose Source / Swapper / Sink / PriceOracle into atomic DeFi transactions | `flow-actions` | `cadence-lang`, `flow-defi` |
| Build a scheduled DeFi executor (DCA, auto-rebalancer, recurring strategy) | `flow-actions` | `cadence-lang`, `cadence-audit` |
| Implement a custom Source / Sink / Swapper / PriceOracle connector for a protocol | `flow-actions` | `cadence-lang`, `cadence-scaffold` |
| Write unit tests for Cadence contracts | `cadence-testing` | `cadence-lang` |
| Debug failing Cadence tests / add coverage | `cadence-testing` | `cadence-lang`, `cadence-audit` |
| Generate random numbers on Flow (revertibleRandom, commit-reveal, VRF) | `cadence-lang` | `cadence-audit`, `cadence-scaffold`, `flow-react-sdk` |
| Call an EVM contract from a Cadence transaction (COA, `coa.call`, ABI encoding) | `flow-crossvm` | `cadence-lang`, `cadence-audit` |
| Read ERC20 balances or storage from a Cadence script | `flow-crossvm` | `cadence-lang` |
| Bridge native FLOW between Cadence and EVM, or hold EVM assets under Cadence custody | `flow-crossvm` | `cadence-lang`, `flow-defi` |
| Write cross-VM integration tests (Cadence + EVM together) | `cadence-testing` | `flow-crossvm`, `cadence-lang` |
| Audit CrossVM code for capability, atomicity, or CU-ceiling pitfalls | `cadence-audit` | `flow-crossvm`, `cadence-lang` |

## Install and Validate Commands

There is no build or test target. Validate structural changes with:

- `claude plugin validate .` — schema-validates `marketplace.json` and each `plugin.json`
  (documented in `README.md` § Contributing).

End-user install flow (from `scripts/install.sh` and `README.md`):

- `sh -ci "$(curl -fsSL https://raw.githubusercontent.com/onflow/flow-ai-tools/main/scripts/install.sh)"`
- `/plugin marketplace add onflow/flow-ai-tools` then `/plugin install flow-dev@flow-ai-tools`
- `claude mcp add --scope user cadence-mcp -- flow mcp` (Cadence MCP, added by `install.sh`)

## Conventions and Gotchas

- **No code, only Markdown.** Every change is to `.md` files, `marketplace.json`, or
  `plugin.json`. There is no language toolchain in this repo.
- **Kebab-case names.** Plugin and skill directory names must be kebab-case and match the
  `name` field in their respective `plugin.json` / SKILL.md frontmatter.
- **SKILL.md frontmatter is required.** Each skill needs YAML with `name` and `description`.
  The `description` must include both trigger phrases and (when relevant) non-trigger
  redirects to the correct skill — see existing skills for the pattern.
- **Reference files stay 200–300 lines.** If a topic exceeds 300 lines, split into multiple
  files rather than truncating.
- **Cadence 1.0 syntax only.** All Cadence code examples in skill content must follow
  Cadence 1.0. Include `✅` / `❌` patterns where applicable.
- **Register a new skill in three places.** Create `SKILL.md` + `references/`, update the
  Skill Routing Guide table above, update the skill catalog in `README.md`.
- **Register a new plugin in four places.** Create `plugins/<name>/.claude-plugin/plugin.json`,
  add skills under `skills/`, add an entry to the `plugins` array in
  `.claude-plugin/marketplace.json`, update `README.md`.
- **`.gitignore` excludes `.claude` and `docs/plans/`.** Don't commit local Claude state or
  planning documents.
- **Ownership.** `CODEOWNERS` routes every path to the team listed there for PR review.

## Files Not to Modify

- `scripts/install.sh` — public one-liner endpoint referenced from the README; breaking
  changes affect existing users running the curl command.

## Content Sources

Skills in this marketplace were derived from:

- [onflow/cadence-rules](https://github.com/onflow/cadence-rules) — Cadence language rules,
  security patterns, DeFi Actions framework
- [onflow/flow-cli](https://github.com/onflow/flow-cli) — Flow CLI query patterns and FindLabs API
- Flow official documentation — cadence-lang.org, developers.flow.com
- Security audit best practices for Cadence smart contracts
