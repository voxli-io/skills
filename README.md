# Voxli Skill for Claude

A [Claude Code skill](https://code.claude.com/docs/en/skills) that teaches Claude how to write effective tests for conversational AI agents using [Voxli](https://voxli.io).

## What it does

The Voxli MCP server gives Claude tools to create tests, run scenarios, and read results. This skill adds the **workflow knowledge** — how to write good test instructions, choose assertion severity levels, follow the right tool call sequence, and interpret results.

With this skill installed, you can say things like:

- "Help me create tests for my customer support chatbot"
- "Run my Voxli tests and tell me what failed"
- "Set up local testing with Voxli"
- "Write assertions for this test"

## Prerequisites

You need the Voxli MCP server connected. More information on how to add it here https://app.voxli.io/docs/developers/mcp

## Install

### Claude Code (plugin marketplace)

```
/plugin marketplace add voxli-io/skills
```

Then browse and install:

1. Select **Browse and install plugins**
2. Select **voxli**
3. Select **Install now**

Or install directly:

```
/plugin install voxli@voxli
```

### Manual install

Clone this repo and copy the skill folder into your Claude skills directory:

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
| `skills/voxli/references/mcp-tools-reference.md` | Quick reference for all 12 MCP tools |
| `skills/voxli/references/local-testing-setup.md` | CLI setup for local agent testing |
| `skills/voxli/references/interpreting-results.md` | How to read results and diagnose failures |

## Links

- [Voxli](https://voxli.io) — the testing platform
- [Voxli MCP docs](https://voxli.io/docs/developers/mcp) — MCP server setup
- [Claude Code skills docs](https://code.claude.com/docs/en/skills) — how skills work
