---
name: voxli
description: >
  Guides you through creating and running automated tests for conversational AI agents
  using Voxli. Helps write effective test instructions, assertions, and scenarios via
  the Voxli MCP server. Use when the user asks to "test my chatbot", "create Voxli tests",
  "write test assertions", "run tests with Voxli", "set up Voxli", or "check test results".
  Requires the Voxli MCP server to be connected.
metadata:
  author: Voxli
  version: 0.5.0
  mcp-server: voxli
---

# Voxli Testing Skill

Voxli runs simulated multi-turn conversations against your chatbot and scores each one with assertions.
This skill helps you create, run and analyse those tests via the Voxli MCP server.

**Writing tests is collaborative.** Don't assume what the user wants to test or why — ask. See `references/iterating-on-tests.md` for the discovery-write-run-read-improve loop.

Reference docs:

- `references/iterating-on-tests.md` — the collaborative process for writing and improving tests with the user
- `references/writing-tests.md` — how to write a good test instruction
- `references/writing-assertions.md` — how to write good assertions
- `references/interpreting-results.md` — how to read run output
- `references/local-testing-setup.md` — running tests locally against a local or hosted agent

## Prerequisites

Voxli MCP server must be connected (https://app.voxli.io/docs/developers/mcp):
- Claude Code: `claude mcp add voxli --transport http https://api.voxli.io/mcp`
- Cursor / VS Code: add `https://api.voxli.io/mcp`

The user authenticates via browser on first use.

## Workflow

1. **Discover** — list existing scenarios, agents, and the tests inside a scenario before creating anything new. If a scenario exposes `configuration.test_fields`, those are custom fields you can set on its tests.
2. **Create or update tests** — new tests are created without assertions; for existing ones, read current state before updating.
3. **Add assertions** — every test needs at least one blocker covering the core requirement.
4. **Run** — start a run against a scenario and an agent. You can run a subset of tests, or use repetitions (1-10) to detect flaky behavior. If using the Voxli CLI, target the LOCAL agent registered to the user's machine.
5. **Check results** — fetch results by run ID. If the run is still in progress, wait and fetch again.
6. **Iterate** — fix the agent or the test based on failures.

## Hard rules

- **No em dashes (—)** in any test text or assertion text. Use commas, periods, or hyphens.
