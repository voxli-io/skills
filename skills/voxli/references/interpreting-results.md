# Interpreting Test Results

After running tests, use this guide to interpret the results.

## Reading Order

Work from the overview down to the detail:

1. **`get_run_group`**: the whole run. `metrics` folds every run in the group once (score, blockers, warnings, hallucinations, and the workspace's own metrics), and `runs` gives one line per scenario and personality with its `score`. Quote group numbers from here: a group value is not the average of its runs' values.
2. **`get_run`**: one run's tests. Each test lists its repeats with `status`, `score`, and each failed assertion with the judge's explanation.
3. **`get_test_results`**: the detail behind specific `result_id`s from `get_run`: every assertion result, and the conversation when you ask for it.

## Status

`done` on `get_run_group` is `true` once every run has finished. Until then, metrics are partial and `pending_results` counts the repeats still going, so wait for `done` before quoting numbers. A canceled repeat never counts toward metrics, including one a retry replaced. To wait, call `get_run_group` with `status_only: true`: it returns only `done` and `pending_results`, so checking every 15 to 30 seconds is cheap.

| Run `status` | Meaning |
|--------------|---------|
| `new`, `pending`, `running` | Still going. |
| `completed` | Every test finished. |
| `failed` | The run stopped on an error. |
| `canceled` | Stopped before it finished. |

A repeat's `status` is `pending`, `running`, `completed`, `failed` (the conversation errored, and it scores 0) or `canceled`. There is no pass/fail verdict for a test or a run: read the score and the failed assertions.

## Test Result Fields

`get_test_results` returns one entry per result:

| Field | Description |
|-------|-------------|
| `result_id` | Test result ID |
| `test_name`, `scenario_name`, `personality_name`, `agent_name` | What ran, as whom, on which agent |
| `status` | See above |
| `score` | Weighted percentage (0-100) of passed assertions |
| `test_instruction` | The instruction that was used for this test |
| `assertion_results` | Array of assertion evaluations |
| `conversation` | Only with `include_conversation` or `include_conversation_types` |

## Reading Assertion Results

Each assertion result contains:

| Field | Description |
|-------|-------------|
| `criteria` | The assertion text that was evaluated |
| `severity` | `"blocker"`, `"medium"`, or `"low"` |
| `passed` | `true` or `false` |
| `explanation` | AI judge's reasoning for the pass/fail decision |

### What to look for

1. **Failed blockers**: these are the most critical. They mean a core requirement wasn't met.
2. **Failed medium assertions**: these reduce the score and point to expected behaviors the agent missed.
3. **Failed low assertions**: minor issues. Worth noting but not urgent.
4. **Explanations**: the AI judge explains why each assertion passed or failed. Read these to understand the root cause.

### Score and Blockers

The score and the failed blockers are separate signals. A test can score 72% with no failed blocker (some medium and low assertions failed), or 80% with one. Report both.

## Reading Conversations

The `conversation` array shows every message in the exchange:

```json
[
  { "type": "message", "role": "chatbot", "content": "Hi! How can I help you today?" },
  { "type": "message", "role": "tester", "content": "I'd like to check on my order NS-28479." },
  { "type": "internal-event", "name": "intent_classified", "metadata": { "intent": "order_status" } },
  { "type": "tool", "name": "check_order", "metadata": { "order_id": "NS-28479", "result": { "status": "shipped" } } },
  { "type": "public-event", "name": "order_status_card", "metadata": { "order_id": "NS-28479", "status": "shipped" } },
  { "type": "message", "role": "chatbot", "content": "Your order NS-28479 has been shipped!" }
]
```

- **`type: "message"`, `role: "chatbot"`** — messages from the agent under test (visible to the tester)
- **`type: "message"`, `role: "tester"`** — messages from the tester (Voxli, simulating the user)
- **`type: "tool"`** — tool calls made by the agent (e.g. API calls, lookups). Arguments and return values in `metadata`. Not visible to the tester, but visible to the AI judge.
- **`type: "internal-event"`** — behind-the-scenes data such as classifications, intent detection, or collected fields. Not visible to the tester, but visible to the AI judge.
- **`type: "public-event"`** — UI elements shown to the end user (forms, status cards, widgets). Visible to both the tester and the judge. Payload in `metadata`.

### Using the conversation to diagnose failures

When an assertion fails, trace through the conversation to find where things went wrong:

1. Find the assertion that failed and read its explanation
2. Look for the relevant turn in the conversation
3. Check if the agent responded correctly, called the right tool, or missed a step

## Iterating on Results

After diagnosing failures:

1. **Fix the agent** if the problem is in the agent's behavior, prompt, or tools
2. **Fix the test** if the instruction was unclear or the assertion was wrong
3. **Re-run** by calling `run_tests` on the same agent with the fields of the run group's `setup` (from `get_run_group`)
4. **Compare** new results against the previous run to verify improvements: put both run groups in one comparison (`create_comparison`, or `update_comparison` with every column for one that exists) and read it with `get_comparison`

Use `repetitions` (2-3) when re-running to check that the fix is stable and not just a flaky pass.

## Reporting Results to the User

When presenting results, lead with the most important information:

1. **Score and blockers**: the run's score, blocker and warning counts from `get_run_group`
2. **Failed blocker assertions**: highlight these first
3. **Score summary**: per-run and per-test scores
4. **Specific failures**: what went wrong and where in the conversation
5. **Recommendations**: what to fix (agent-side or test-side)

Try to keep this short and to the point, no unecessary fluff.
Users want to know why something failed and what they can do about it.