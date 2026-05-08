# Iterating on Tests with the User

Writing Voxli tests is collaborative. The user knows what their agent should do; we know how to express that as a Voxli test and how to read the results. **You should never have an opinion on what is worth testing or why** — those are the user's calls. Your job is to surface them through questions, then turn answers into well-formed tests.

This guide covers the workflow. For the artifact-level rules (persona, steps, end conditions, assertion patterns), see `writing-tests.md` and `writing-assertions.md`.

## The loop

1. **Discover** — ask the user what they want to test before writing anything (unless they already told you upfront).
2. **Write** — one focused test using `writing-tests.md` + `writing-assertions.md`.
3. **Run** — ask the user whether they want you to run it (via the Voxli MCP) or run it themselves.
4. **Read** — go through the results with the user. State facts, not judgements.
5. **Improve** — diagnose together, then adjust the test, the assertions, the agent, or the scope.
6. **Repeat or expand** — iterate on the same test, or move to the next scenario.

## 1. Discover

Before writing anything, ask. A few questions to lean on (pick the ones that fit; don't bombard):

- **What does this agent do?** Domain, primary use cases, what it's deployed for.
- **What's the most important behavior it must get right?** That's the candidate for the first blocker.
- **What's it most likely to get wrong?** That's a candidate for the next test.
- **Are there hard "must not" rules?** Safety, scope, data privacy.
- **Does it call tools, APIs, or hand off to other systems?** If yes, those are candidates for tool-call assertions.
- **Where will tests run?** Voxli-managed agent (e.g. EBBOT/WEBHOOK/DEMO), local agent, or staged via the MCP server only.
- **Is there an existing test or instruction format you'd like the test to look like?** Match the user's house style.

If the user gives a vague brief ("just test my agent"), narrow it together. Pick one concrete scenario and start there. Avoid generating a sweeping coverage plan upfront — that's premature.

## 2. Write

Use `writing-tests.md` for the instruction and `writing-assertions.md` for the assertions. Show the user the proposed instruction and assertion list **before** creating the test in Voxli, so they can flag a mismatch with their intent. This saves a round-trip.

Common things the user might want to change:
- The persona's tone or backstory
- A different opening line
- Different severity on a specific assertion
- An assertion the user thinks is too strict, too loose, or off-target

Adjust, then create the test in their scenario.

## 3. Run

Don't assume you should run it. Ask:

- *"Want me to run this via the MCP, or do you want to run it yourself in the Voxli UI?"*

Reasons to defer to the user:
- They may want to point a specific live agent at it.
- They may already have a test pipeline (CI, scheduled runs).
- Running consumes credits or hits external systems.
- The agent might not be ready for testing yet.

If they ask you to run it, use the standard MCP flow: `start_run`, then poll `get_test_results`.
Don't guess what agent to use, make sure you ask if unclear.
Users with local agents that are online usually want to use those.
Also consider running it with multiple iterations to detect flakyness.

## 4. Read

Once results land, lead with facts:

- Score, `overallPassed`, the link to the run in the UI.
- Each assertion: severity, pass/fail, the judge's stated reason.
- The conversation flow in 1-2 sentences.

Don't editorialise. Don't say "the agent has a bug" — say "this assertion failed: <reason>." See `interpreting-results.md` for the field-by-field breakdown.

## 5. Improve

**A failing assertion is information, not a verdict.** Before suggesting a change, work through what the failure could mean:

| Cause | Symptom |
|-------|---------|
| Genuine agent issue | The behavior the user expected didn't happen and the assertion correctly caught it. |
| Assertion too strict | The agent did something reasonable that the assertion didn't allow for. Common: blocker required a specific named entity the agent can't know about, or a phrasing the agent doesn't use. |
| Instruction unclear | The tester drifted, misread its persona, or made the agent answer something different. The transcript shows the tester not following the intended path. |
| Persona-test mismatch | The chosen persona is incompatible with the test's blocker. |
| Flake | The judge made a borderline call. Re-run; if it fluctuates, the assertion is too vague. |

Surface the candidates and ask the user which one matches their reading of the result. *"Looking at the transcript, the agent gave a category-level answer instead of a specific one. Two ways to read this: the agent is being too vague, or the assertion expected more specificity than this case warrants. Which feels right?"*

Then make the targeted change — don't rewrite the test wholesale.

## 6. Repeat or expand

Stop iterating on a test when:
- The user is satisfied with what it covers.
- It consistently passes (or fails) the same way across re-runs — that's a stable signal.
- Further changes would just shift it without adding signal.

Then ask the user what to cover next: a different scenario type, a known weak spot, or a regression check for a recent change.

## Things to avoid

- **Assuming what matters.** "This test should also check for tone" — only if the user said tone matters.
- **Treating one failure as a verdict on the agent.** A failed test is a flag for diagnosis, not a bug report.
- **Auto-running tests without asking.** Even after the user has run a few via you, ask before each new run unless they've said "go ahead and run it whenever".
- **Over-coverage on the first pass.** Five tests for a new scenario when the user asked for one. Build coverage iteratively, with the user.
- **Rewriting both the test and the suggested fix.** If you change the test in response to a failure, say what changed and why; let the user accept or reject.

## Quick discovery template

When the user just says "help me test my agent", this is a short script that gets to a first test fast without making assumptions:

1. Ask what the agent does in one sentence.
2. Ask for one concrete scenario they want to verify (or one they're worried about).
3. Ask what success would look like for that scenario.
4. Ask if there's anything the agent must not do in that scenario.
5. Draft the test, show it, run it (with permission), read it together.
