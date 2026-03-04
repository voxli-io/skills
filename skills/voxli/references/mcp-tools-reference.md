# MCP Tools Reference

Quick reference for all 12 Voxli MCP tools, organized by workflow phase.

## Browse

### `list_scenarios()`
List all available scenarios.
- **Params:** none
- **Returns:** `{ scenarios: [{ id, name, description, fields }] }`

### `list_agents()`
List all available agents.
- **Params:** none
- **Returns:** `{ agents: [{ id, name, type }] }`

### `get_scenario_tests(scenario_id)`
Get all enabled, non-archived tests in a scenario with full details.
- **Params:** `scenario_id` (string, required)
- **Returns:** `{ tests: [{ id, name, instruction, assertions: [{ criteria, severity }], fields }] }`

### `get_test(test_id)`
Get full details of a single test.
- **Params:** `test_id` (string, required)
- **Returns:** `{ id, name, description, instruction, assertions, isEnabled, scenario_id, fields }`

## Create & Update

### `create_test(scenario_id, name, instruction, fields?)`
Create a new test in a scenario. Tests are created without assertions — use `add_test_assertion` after.
- **Params:**
  - `scenario_id` (string, required) — scenario to add the test to
  - `name` (string, required) — test name
  - `instruction` (string, required) — what the AI tester should do
  - `fields` (string, optional) — JSON object of key-value configuration fields
- **Returns:** `{ id }` — the new test ID

### `update_test(test_id, name?, instruction?, fields?)`
Update an existing test. Only non-empty fields are applied. Does not handle assertions.
- **Params:**
  - `test_id` (string, required) — test to update
  - `name` (string, optional) — new name
  - `instruction` (string, optional) — new instruction (regenerates the case)
  - `fields` (string, optional) — JSON object to replace configuration fields
- **Returns:** `{ id, name }`

### `add_test_assertion(test_id, criteria, severity)`
Add an assertion to a test (max 10 per test).
- **Params:**
  - `test_id` (string, required)
  - `criteria` (string, required) — specific, verifiable statement
  - `severity` (string, required) — `"blocker"`, `"medium"`, or `"low"`
- **Returns:** `{ assertions: [{ criteria, severity }] }` — full updated list

### `update_test_assertion(test_id, assertion_index, criteria?, severity?)`
Update an existing assertion by 0-based index.
- **Params:**
  - `test_id` (string, required)
  - `assertion_index` (integer, required) — 0-based index
  - `criteria` (string, optional) — new criteria text
  - `severity` (string, optional) — `"blocker"`, `"medium"`, or `"low"`
- **Returns:** `{ assertions: [{ criteria, severity }] }` — full updated list

### `remove_test_assertion(test_id, assertion_index)`
Remove an assertion by 0-based index.
- **Params:**
  - `test_id` (string, required)
  - `assertion_index` (integer, required) — 0-based index
- **Returns:** `{ assertions: [{ criteria, severity }] }` — full updated list

## Run

### `start_run(scenario_id, agent_id, test_ids?, repetitions?)`
Start a test run for a scenario with a given agent.
- **Params:**
  - `scenario_id` (string, required)
  - `agent_id` (string, required)
  - `test_ids` (array of strings, optional) — specific tests to run; empty = all enabled tests
  - `repetitions` (integer, optional) — 1-10, default 1
- **Returns:** `{ run_id, status, scenario_id, scenario_name, agent_id, agent_name }`

## Analyze

### `get_latest_runs()`
Get the 10 most recent test runs with scenario and agent info.
- **Params:** none
- **Returns:** `{ runs: [{ id, status, scenario_id, scenario_name, agent_id, agent_name, created_at }] }`

### `get_test_results(run_id)`
Get the status and test results for a run.
- **Params:** `run_id` (string, required)
- **Returns:**
```json
{
  "run_id": "...",
  "status": "completed",
  "scenario_id": "...",
  "scenario_name": "...",
  "agent_id": "...",
  "agent_name": "...",
  "results": [
    {
      "id": "...",
      "status": "completed",
      "score": 85,
      "overallPassed": true,
      "testInstruction": "...",
      "conversation": [
        { "type": "message", "role": "assistant", "content": "..." },
        { "type": "message", "role": "user", "content": "..." }
      ],
      "assertionResults": [
        {
          "criteria": "...",
          "severity": "blocker",
          "passed": true,
          "explanation": "..."
        }
      ]
    }
  ]
}
```

## Common Workflows

### Create a test with assertions
```
1. create_test(scenario_id, "Order lookup", instruction)     → get test_id
2. add_test_assertion(test_id, "Agent provides order status", "blocker")
3. add_test_assertion(test_id, "Agent verifies identity first", "medium")
4. add_test_assertion(test_id, "Agent uses customer's name", "low")
```

### Run and check results
```
1. list_scenarios()                     → pick scenario_id
2. list_agents()                        → pick agent_id
3. start_run(scenario_id, agent_id)     → get run_id
4. get_test_results(run_id)             → check status, read results
```

### Update an assertion
```
1. get_test(test_id)                    → see current assertions with indices
2. update_test_assertion(test_id, 0, criteria="New criteria text")
```

### Remove and replace an assertion
```
1. get_test(test_id)                    → find the assertion index
2. remove_test_assertion(test_id, 2)    → remove assertion at index 2
3. add_test_assertion(test_id, "New check", "medium")  → add replacement
```
