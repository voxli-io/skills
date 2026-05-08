# Interpreting Test Results

After running tests, use this guide to interpret the results.

## Run Status

The top-level `status` field tells you where the run is:

| Status | Meaning |
|--------|---------|
| `running` | Tests are still executing. |
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

### overallPassed Logic

- If the test has **any blocker assertions**, `overallPassed` is `true` only if **all blockers pass**.
- If the test has **no blocker assertions**, `overallPassed` is `true` by default.
- The numeric score does NOT determine `overallPassed` — only blocker assertions do.

This means a test can have a score of 72% and still pass (all blockers passed, some medium/low failed), or have a score of 80% and fail (one blocker failed).

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

Try to keep this short and to the point, no unecessary fluff.
Users want to know why something failed and what they can do about it.