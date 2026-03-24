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
  version: 0.4.0
  mcp-server: voxli
---

# Voxli Testing Skill

## Overview

Voxli is an automated testing platform for conversational AI agents. It simulates multi-turn conversations against your chatbot, then scores each conversation on correctness, tone, tool usage, and more.

The core testing loop:

1. **Create a scenario** — a collection of related tests (e.g., "Order Flow").
2. **Write tests** — each test describes a conversation to simulate.
3. **Add assertions** — pass/fail checks evaluated after each conversation.
4. **Run the scenario** against an agent — Voxli plays the role of a user and talks to your chatbot.
5. **Inspect results** — review transcripts, assertion outcomes, and scores.
6. **Iterate** — refine tests or fix your agent, then run again.

## Prerequisites

The Voxli MCP server must be connected. If it isn't, the user needs to add it first:

- **Claude Code**: `claude mcp add voxli --transport http https://api.voxli.io/mcp`
- **Cursor / VS Code**: Add the Voxli MCP server URL `https://api.voxli.io/mcp`

The user will be prompted to authenticate via browser on first use.

## Core Workflow

When helping users test their chatbot, follow this sequence:

### Step 1: Discover what exists

Call `list_scenarios` and `list_agents` to see what the user already has. If they have existing scenarios, call `get_scenario_tests` to see the tests inside.

If a scenario has a linked configuration, `list_scenarios` returns a `configuration` object with `scenario_fields` and `test_fields`. Check `configuration.test_fields` to see what fields are available for tests in that scenario.

### Step 2: Create or update tests

If starting fresh, help the user create tests with `create_test`. If improving existing tests, use `get_test` to read the current state, then `update_test`.

If the scenario has configured test fields (visible from `list_scenarios`), pass a `fields` JSON string to `create_test` or `update_test` with key-value pairs matching the field definitions. Field types: `single_line_text` and `long_text` expect strings, `checkbox` expects booleans.

Key principle: **tests are created without assertions**. Always use `add_test_assertion` after creating a test.

### Step 3: Add assertions

After creating a test, add assertions with `add_test_assertion`. Every test should have at least one blocker assertion covering the core requirement.

### Step 4: Run the scenario

Call `start_run` with the scenario ID and agent ID. Optionally pass specific `test_ids` to run a subset, or `repetitions` (1-10) to detect flaky behavior. If the voxli cli is used, then use the LOCAL agent registered as the users computer.

### Step 5: Check results

Call `get_test_results` with the run ID. If the run status is still `"running"`, wait a moment and call again. Review each test result's score, `overallPassed`, assertion results, and conversation transcript.

### Step 6: Iterate

Based on the results, suggest improvements — either to the user's agent or to the tests themselves.

## Writing Good Instructions

Test instructions tell the AI tester what to do. They should follow this pattern:

1. **Define a persona** — give the tester a name and context ("You're Sarah. Your order number is NS-28479.")
2. **List steps** — numbered steps describing what to say and do
3. **Include an explicit end condition** — always tell the tester when to stop

**Good instruction:**
```
You're Sarah. Your order number is NS-28479, and your email is sarah@example.com.

Follow these steps:
1. Say "What is the status or my order?"
2. When asked, provide your order number and email.
3. Once you receive the order status, say "When is the delivery date?"

End: End the conversation after you receive the delivery date or after step 3.
```

**Bad instruction:**
```
Ask the chatbot about an order.
```
This is too vague — the tester doesn't know what order, what to ask, or when to stop.

For a deep dive with more examples, see `references/writing-tests.md`.

## Writing Good Assertions

Assertions are pass/fail checks evaluated by an AI judge after the conversation ends. Write them as specific, verifiable statements.

**Good assertions:**
- "The chatbot asked for the customer's email before looking up the order"
- "The chatbot provided the correct order status: shipped"
- "The chatbot responded in Swedish throughout the conversation"

**Bad assertions:**
- "The chatbot was helpful" (too vague)
- "Good response" (not verifiable)

### Severity levels

| Severity | Weight | Use for |
|----------|--------|---------|
| **blocker** | 4 | Safety rules, core functionality, must-do actions. If any blocker fails, the test fails. |
| **medium** | 2 | Expected behaviors, standard responses. |
| **low** | 1 | Tone, formatting, optional nice-to-haves. |

**Always include at least one blocker** for the test's core requirement. Use medium/low for supplementary checks.

For a deep dive with more examples, see `references/writing-assertions.md`.

## Choosing What to Test

Knowing *how* to write tests is half the picture - knowing *what types* of scenarios to cover is just as important. Consider these categories when building test coverage:

- **Happy path** — Standard flows where the user follows the expected path
- **Boundary enforcement / Refusal** — Requests the agent should decline or redirect
- **Context switching** — User changes topic mid-conversation
- **Guardrails** — Agent stays within its role, doesn't hallucinate or fabricate data
- **Abuse resistance** — Prompt injection, manipulation, offensive language
- **Edge cases** — Empty inputs, unexpected formats, ambiguous requests
- **Escalation** — When the agent should hand off to a human

Start with happy path tests for your core flows, then add boundary and guardrail tests for your highest-risk areas. For detailed descriptions, example instructions, and suggested assertions for each category, see `references/scenario-types.md`.

## Tool Calls and Events

During a test conversation, you can register tool calls and events alongside regular messages. There are three types:

| Type | Visible to simulated user | Use for |
|------|--------------------------|---------|
| `tool` | No | Agent actions: API calls, database lookups, function invocations |
| `internal-event` | No | Behind-the-scenes data: collected fields, classifications, knowledge articles |
| `public-event` | Yes | UI elements shown to the end user: forms, widgets, order status cards |

The visibility distinction matters for assertions. The AI judge only treats `message` and `public-event` as user-visible - so an assertion like "The chatbot showed the order status" will fail if the status only appears in a `tool` or `internal-event`.

For implementation details on how to register these, see `references/local-testing-setup.md`.

## Running Tests via MCP

The typical tool call sequence:

```
1. list_scenarios()          → find the scenario ID
                                → check configuration.test_fields for available fields
2. list_agents()             → find the agent ID
3. start_run(scenario_id, agent_id)  → starts the run, returns run_id
4. get_test_results(run_id)  → check status and results
```

If the run is still in progress, `get_test_results` shows `status: "running"`. Wait and call it again.

For selective runs, pass `test_ids` to `start_run`. For repetition testing, pass `repetitions` (up to 10).


## Interpreting Results

When reading results from `get_test_results`:

- **`status`** — the run's current state (`"running"`, `"completed"`, `"error"`)
- **`score`** — weighted percentage (0-100) of passed assertions
- **`overallPassed`** — `true` only if all blocker assertions passed. If there are no blockers, it's `true` by default.
- **`assertionResults`** — each assertion's criteria, severity, passed status, and explanation
- **`conversation`** — the full message transcript between tester and agent

**Score formula:** `score = (sum of passed assertion weights / sum of all assertion weights) * 100`

Weights: blocker=4, medium=2, low=1.

When a test fails, look at the failed assertions' explanations and the conversation transcript to diagnose the issue. Common causes: the agent didn't understand the request, called the wrong tool, or gave an incorrect response.

For more details, see `references/interpreting-results.md`.

## Local Testing with CLI

For testing agents running on the user's local machine:

1. Install: `npm install -g @voxli/cli`
2. Authenticate: `voxli auth`
3. Start listening: `voxli listen --command "python run_my_agent.py"`

The CLI registers the local machine as an agent in Voxli. When a test run targets this agent, the CLI picks up the work and runs the specified command.

The command receives two environment variables:
- `VOXLI_API_TOKEN` — authentication token
- `VOXLI_TEST_RESULT_IDS` — JSON array of test result IDs to execute

For setup details and code examples, see `references/local-testing-setup.md`.

## Common Mistakes

### Vague instructions
- **Problem**: "Test the chatbot" gives the AI tester no direction.
- **Fix**: Provide a persona, specific steps, and an end condition.

### Missing end condition
- **Problem**: The tester doesn't know when to stop, leading to aimless conversations.
- **Fix**: Always include "End: ..." at the bottom of the instruction.

### Vague assertions
- **Problem**: "The chatbot was good" is not something an AI judge can reliably evaluate.
- **Fix**: Use specific, observable criteria: "The chatbot provided the order tracking number."

### Wrong severity
- **Problem**: Using "low" for a critical safety check, or "blocker" for a formatting preference.
- **Fix**: Blocker = must-pass requirements. Medium = expected behavior. Low = nice-to-have.

### Too many steps in one test
- **Problem**: A single test with 15 steps is hard to debug when it fails.
- **Fix**: Split into multiple focused tests. Each test should cover one scenario or flow.

### Not adding assertions
- **Problem**: A test without assertions always "passes" but validates nothing.
- **Fix**: Every test needs at least one assertion. Start with a blocker for the core requirement.
