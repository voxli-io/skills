# Voxli Skills

Skills for AI coding assistants that add workflow knowledge for testing conversational AI agents with [Voxli](https://voxli.io).

## What it does

The Voxli MCP server gives your coding assistant tools to create tests, run scenarios, and read results. This skill adds the **workflow knowledge** — how to write good test instructions, choose assertion severity levels, follow the right tool call sequence, and interpret results.

With this skill installed, you can say things like:

- "Help me create tests for my customer support chatbot"
- "Run my Voxli tests and tell me what failed"
- "Set up local testing with Voxli"
- "Write assertions for this test"

## Prerequisites

You need the Voxli MCP server connected. See the [MCP setup guide](https://app.voxli.io/docs/developers/mcp) for instructions.

## Install

### Skills CLI (works with 40+ agents)

```bash
npx skills add voxli-io/skills
```

This installs the skill for all supported agents (Claude Code, Cursor, Codex, GitHub Copilot, and more).

### Claude Code (plugin marketplace)

```
claude plugin marketplace add voxli-io/skills
claude plugin install voxli@voxli
```

### Manual install

Clone this repo and copy the skill folder into your skills directory:

```bash
git clone https://github.com/voxli-io/skills.git
cp -r skills/skills/voxli ~/.claude/skills/voxli
```

## What's included

| File | Purpose |
|------|---------|
| `skills/voxli/SKILL.md` | Core skill — workflow, key rules, common mistakes |
| `skills/voxli/references/writing-tests.md` | Deep guide on writing test instructions with examples |
| `skills/voxli/references/writing-assertions.md` | Assertion criteria, severity framework, scoring |
| `skills/voxli/references/mcp-tools-reference.md` | Quick reference for all MCP tools |
| `skills/voxli/references/local-testing-setup.md` | CLI setup for local agent testing |
| `skills/voxli/references/interpreting-results.md` | How to read results and diagnose failures |

## Links

- [Voxli](https://voxli.io) — the testing platform
- [MCP setup guide](https://voxli.io/docs/developers/mcp) — connect the MCP server
- [Claude Code skills docs](https://code.claude.com/docs/en/skills) — how skills work in Claude Code
