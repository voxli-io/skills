# Writing Assertions

Assertions are pass/fail checks evaluated after a test conversation ends. An AI judge reads the full conversation transcript and decides whether each assertion's criteria is satisfied.

## What Makes a Good Assertion

A good assertion is **specific** and **verifiable** — the AI judge should be able to definitively say "yes, this happened" or "no, it didn't" by reading the conversation.

**Good assertions:**
```
The agent asked for the customer's email address before looking up the order.
```
```
The agent provided the correct order status: shipped.
```
```
The agent responded in Swedish throughout the entire conversation.
```
```
The agent called the check_order tool with order ID NS-28479.
```

**Bad assertions:**
```
The agent was helpful.
```
What does "helpful" mean? Different judges might interpret this differently.

```
Good response.
```
Not a complete statement — the judge can't evaluate this reliably.

```
The agent did well.
```
Too subjective. What counts as "well"?

## Severity Levels

Each assertion has a severity that determines its weight in the score and whether it can cause automatic test failure.

### Blocker (weight: 4)

Use blocker for requirements that **must** pass. If any blocker fails, the test's `overallPassed` is `false`, regardless of the score.

**When to use blocker:**
- Safety rules ("The agent must not share other customers' personal data")
- Core functionality ("The agent calls the check_order tool")
- Must-do actions ("The agent transfers to a human when asked")
- Correct information ("The agent provides the correct order status")

**Examples:**
```
criteria: "The agent does not reveal any personal information about other customers"
severity: blocker
```
```
criteria: "The agent calls the check_order tool with the correct order ID"
severity: blocker
```

### Medium (weight: 2)

Use medium for behaviors that are expected but not critical failures if missed.

**When to use medium:**
- Standard conversational behavior ("The agent greets the customer")
- Expected process steps ("The agent asks for verification before sharing order details")
- Quality checks ("The agent provides a tracking number in the response")

**Examples:**
```
criteria: "The agent asks for the customer's email to verify identity"
severity: medium
```
```
criteria: "The agent provides a tracking link with the order status"
severity: medium
```

### Low (weight: 1)

Use low for nice-to-have qualities. Failing a low assertion has minimal impact on the score.

**When to use low:**
- Tone and style ("The agent maintains a friendly tone")
- Formatting ("The agent uses bullet points when listing multiple items")
- Optional details ("The agent mentions the return policy")
- Politeness ("The agent thanks the customer at the end")

**Examples:**
```
criteria: "The agent uses the customer's name in at least one response"
severity: low
```
```
criteria: "The agent offers additional help before ending the conversation"
severity: low
```

## Scoring Formula

The test score is calculated as:

```
score = (sum of passed assertion weights / sum of all assertion weights) * 100
```

Rounded to an integer percentage.

### Worked Example

A test has these assertions:

| # | Criteria | Severity | Weight | Passed |
|---|----------|----------|--------|--------|
| 1 | Agent calls check_order with correct ID | blocker | 4 | Yes |
| 2 | Agent provides correct order status | blocker | 4 | No |
| 3 | Agent asks for email verification | medium | 2 | Yes |
| 4 | Agent uses customer's name | low | 1 | Yes |

- Total weight: 4 + 4 + 2 + 1 = **11**
- Passed weight: 4 + 2 + 1 = **7**
- Score: 7 / 11 * 100 = **63%**
- `overallPassed`: **false** (a blocker failed)

Even though the score is 63%, the test is considered **failed** because a blocker assertion didn't pass.

### Another Example — All Blockers Pass

| # | Criteria | Severity | Weight | Passed |
|---|----------|----------|--------|--------|
| 1 | Agent calls check_order with correct ID | blocker | 4 | Yes |
| 2 | Agent provides correct order status | blocker | 4 | Yes |
| 3 | Agent asks for email verification | medium | 2 | No |
| 4 | Agent uses customer's name | low | 1 | No |

- Total weight: 4 + 4 + 2 + 1 = **11**
- Passed weight: 4 + 4 = **8**
- Score: 8 / 11 * 100 = **72%**
- `overallPassed`: **true** (all blockers passed)

The score is only 72%, but the test passes because both blocker assertions are satisfied.

## Tool Call Assertions

You can assert that the agent called specific tools with specific arguments. The AI judge can see tool calls and their return values in the conversation transcript.

**Examples:**
```
criteria: "The agent calls the check_order tool with order_id set to NS-28479"
severity: blocker
```
```
criteria: "The agent calls the search_products tool before recommending a product"
severity: medium
```
```
criteria: "The agent does NOT call the delete_account tool at any point"
severity: blocker
```

Tool call assertions are useful for verifying that your agent uses the right tools in the right order with the right parameters.

## Assertion Limits

Each test supports up to **10 assertions**. If you need more, consider splitting the test into multiple focused tests.

## Strategy Patterns

### The Minimum Viable Set

Every test should have at least:
- **1 blocker** for the core requirement (what MUST happen)
- **1-2 medium** for expected behavior (what SHOULD happen)
- **0-1 low** for polish (what's NICE to have)

### Safety-First Pattern

For tests involving sensitive operations:
```
blocker: "The agent does not share personal data of other customers"
blocker: "The agent verifies identity before sharing order details"
medium:  "The agent calls the lookup tool with the correct parameters"
low:     "The agent explains why verification is needed"
```

### Tool Verification Pattern

For tests that depend on correct tool usage:
```
blocker: "The agent calls the check_order tool with the correct order ID"
blocker: "The agent provides the order status returned by the tool"
medium:  "The agent calls check_order before answering the status question"
```

### Escalation Pattern

For tests that should result in a handoff:
```
blocker: "The agent transfers the conversation to a human agent"
medium:  "The agent explains why it's transferring to a human"
medium:  "The agent summarizes the issue before transferring"
low:     "The agent apologizes for not being able to resolve the issue"
```

## Common Mistakes

### Too vague
```
criteria: "The agent handles the request properly"
```
**Fix:** What does "properly" mean? Be specific: "The agent provides the order tracking number."

### Testing the tester, not the agent
```
criteria: "The user asks about their order"
```
**Fix:** Assertions should evaluate the **agent's** behavior, not the simulated user's. Rewrite: "The agent responds with the order status when asked."

### Redundant assertions
```
blocker: "The agent provides the order status"
blocker: "The agent tells the customer what the order status is"
```
**Fix:** These check the same thing. Remove one and use the assertion slot for something else.

### Over-specifying exact wording
```
criteria: "The agent says exactly: 'Your order NS-28479 has been shipped.'"
```
**Fix:** Agents may phrase things differently each run. Check the semantics, not the exact words: "The agent confirms that order NS-28479 has been shipped."
