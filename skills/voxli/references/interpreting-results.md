# Interpreting Test Results

After running tests, use `get_test_results(run_id)` to fetch the results. This guide explains how to read and act on what you get back.

## Run Status

The top-level `status` field tells you where the run is:

| Status | Meaning |
|--------|---------|
| `running` | Tests are still executing. Call `get_test_results` again after a short wait. |
| `completed` | All tests have finished. Results are ready to read. |
| `error` | Something went wrong during execution. |

## Test Result Fields

Each entry in the `results` array represents one test execution:

| Field | Description |
|-------|-------------|
| `id` | Test result ID |
| `status` | `"completed"` or `"error"` |
| `score` | Weighted percentage (0-100) of passed assertions |
| `overallPassed` | `true` if all blocker assertions passed (or no blockers exist) |
| `testInstruction` | The instruction that was used for this test |
| `conversation` | Array of messages exchanged between tester and agent |
| `assertionResults` | Array of assertion evaluations |

## Reading Assertion Results

Each assertion result contains:

| Field | Description |
|-------|-------------|
| `criteria` | The assertion text that was evaluated |
| `severity` | `"blocker"`, `"medium"`, or `"low"` |
| `passed` | `true` or `false` |
| `explanation` | AI judge's reasoning for the pass/fail decision |

### What to look for

1. **Failed blockers** — these are the most critical. They mean the test failed overall and indicate a core requirement wasn't met.
2. **Failed medium assertions** — these reduce the score and point to expected behaviors the agent missed.
3. **Failed low assertions** — minor issues. Worth noting but not urgent.
4. **Explanations** — the AI judge explains why each assertion passed or failed. Read these to understand the root cause.

## Score Calculation

```
score = (sum of passed assertion weights / sum of all assertion weights) * 100
```

Weights: blocker = 4, medium = 2, low = 1.

### overallPassed Logic

- If the test has **any blocker assertions**, `overallPassed` is `true` only if **all blockers pass**.
- If the test has **no blocker assertions**, `overallPassed` is `true` by default.
- The numeric score does NOT determine `overallPassed` — only blocker assertions do.

This means a test can have a score of 72% and still pass (all blockers passed, some medium/low failed), or have a score of 80% and fail (one blocker failed).

## Reading Conversations

The `conversation` array shows every message in the exchange:

```json
[
  { "type": "message", "role": "assistant", "content": "Hi! How can I help you today?" },
  { "type": "message", "role": "user", "content": "I'd like to check on my order NS-28479." },
  { "type": "public-event", "name": "check_order", "metadata": { "order_id": "NS-28479" } },
  { "type": "message", "role": "assistant", "content": "Your order NS-28479 has been shipped!" }
]
```

- **`role: "assistant"`** — messages from the agent under test
- **`role: "user"`** — messages from the AI tester (Voxli)
- **`type: "public-event"`** — tool calls made by the agent, with arguments and results in `metadata`

### Using the conversation to diagnose failures

When an assertion fails, trace through the conversation to find where things went wrong:

1. Find the assertion that failed and read its explanation
2. Look for the relevant turn in the conversation
3. Check if the agent responded correctly, called the right tool, or missed a step

## Common Failure Patterns

### Agent gave wrong information
- **Symptom**: Blocker assertion fails with explanation saying the agent provided incorrect data
- **Cause**: Agent's knowledge base or tool response is wrong
- **Action**: Fix the agent's data source or tool integration

### Agent didn't call expected tool
- **Symptom**: Tool call assertion fails
- **Cause**: Agent's prompt or routing logic didn't trigger the tool
- **Action**: Update the agent's system prompt or tool configuration

### Agent couldn't understand the request
- **Symptom**: Conversation shows the agent asking for clarification repeatedly
- **Cause**: The request was phrased in a way the agent doesn't handle
- **Action**: Improve the agent's prompt to handle this phrasing, OR adjust the test instruction to be more realistic

### Conversation went off-track
- **Symptom**: Multiple assertions fail; conversation diverged from the expected flow
- **Cause**: An early misunderstanding caused the entire conversation to go in the wrong direction
- **Action**: Look at the first point of divergence — that's where to focus your fix

### Test instruction was ambiguous
- **Symptom**: The conversation doesn't follow the intended flow, but the agent's responses seem reasonable
- **Cause**: The test instruction wasn't specific enough
- **Action**: Rewrite the instruction with clearer steps and a more defined persona

## Iterating on Results

After diagnosing failures:

1. **Fix the agent** if the problem is in the agent's behavior, prompt, or tools
2. **Fix the test** if the instruction was unclear or the assertion was wrong
3. **Re-run** by calling `start_run` again with the same scenario and agent
4. **Compare** new results against the previous run to verify improvements

Use `repetitions` (2-3) when re-running to check that the fix is stable and not just a flaky pass.

## Reporting Results to the User

When presenting results, lead with the most important information:

1. **Overall pass/fail count** — "3 of 5 tests passed"
2. **Failed blocker assertions** — highlight these first
3. **Score summary** — per-test scores
4. **Specific failures** — what went wrong and where in the conversation
5. **Recommendations** — what to fix (agent-side or test-side)
