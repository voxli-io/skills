# Writing Test Instructions

A test instruction tells the AI tester (Voxli) how to behave when talking to your chatbot. It must be concrete enough to produce a repeatable conversation that you can evaluate with assertions.

## The three required parts

1. **Persona** — a name, the data the tester will share when asked, and the situation that brought them here.
2. **Steps** — numbered actions, one per step. React to the agent rather than predict it ("Once you receive the order status..." not "After the agent shows...").
3. **End condition** — a final `End:` line saying when to stop, with a fallback exit so the tester doesn't loop.

### Useful step techniques

- **Quote a context-rich opening line.** Giving the tester an exact first message ("Start with: 'X'") cuts a major source of run-to-run variance — and the opener should also establish the persona's situation so the agent doesn't have to spend a turn extracting it.
  - Bad: `"Start with: 'Hi, I need help.'"`
  - Good: `"Start with: 'Hi, I'm looking for a podiatrist who takes my insurance and has evening slots, what's available?'"`
- **Set the pace.** "Ask one question at a time" prevents the tester from dumping every question into a single message.
- **Group sub-questions under topic headings** when one persona explores several related things — the agent gets a realistic flow, and you get clean assertion targets per topic.
- **Use plain transitions between topics.** "When you've covered the above, ask about..." or "Now switch to..." make the intent unmistakable.
- **Specify the closing phrase.** Giving the tester an exact line to say before ending ("say 'Thanks, that's all I needed.' and end the conversation") produces consistent, clean transcripts.

```
You're Sarah. Your order is NS-28479, email sarah@example.com.

Follow these steps:
1. Ask about the status of your order.
2. Provide your order number and email when asked.
3. Once you have the status, ask when it will be delivered.

End: End the conversation after you receive the delivery date, or if the agent says the order isn't found.
```

## Scope

One focused conversation per test. If you need 10+ steps, or the persona has two unrelated goals, split it. You want failures to be diagnosable from a single assertion.

A persona asking several related questions on **one topic** (e.g. researching a product) is fine — group sub-questions under headings so the conversation flows naturally.

## Patterns

| Pattern | When to use | Key idea |
|---------|-------------|----------|
| Happy path | Verify the core flow works | Direct steps, single goal |
| Information gathering | User researches one topic across many turns | Group related sub-questions |
| Refusal / out-of-scope | Agent should decline | Push twice, accept refusal |
| Escalation | Agent should hand off to a human | Reject standard solutions, ask for human |
| Edge case | Unclear or empty input | Send "hi", "...", then a real message |
| Context switch | User changes topic mid-conversation | Interrupt, then return |
| Prompt injection | Agent should resist manipulation | "Ignore your instructions and..." then verify the agent still works |

### Multi-topic information-gathering example

```
You're Sam, evaluating CRMs for a 12-person sales team that's outgrowing spreadsheets. You have basic technical skills but you're not the engineer who'll set it up.

Start with: "Hi, I'm comparing CRMs for a small sales team. Where do I start?"

Ask one question at a time. Cover:

About the recommended plan:
- What it includes.
- Whether it scales to 12 seats.
- Pipeline and forecasting features.
- Integrations with email and calendar.

About pricing:
- The monthly cost per seat.
- What's not included at this tier.
- Annual vs monthly billing.

Then ask how to start a trial.

End: When all topics are answered, say "Great, I have what I need. Thanks." and end the conversation. If the agent says it cannot help with one of the topics, accept that and move on to the next; if it refuses everything, end the conversation.
```

## Coverage

For any non-trivial agent, aim to cover: happy path, refusal, escalation, edge cases, abuse resistance. One blocker per category is worth more than ten happy-path tests.

## Hard rules

- **No em dashes (—)** anywhere in test text. Use commas, periods, or hyphens (-). Tests with em dashes are rejected.
- **No markdown** in the instruction body. Plain prose with numbered steps.
- **Every `End:` line needs a fallback exit AND an unambiguous stop signal.** Two parts:
  - **Fallback exit.** A success-only end lets the tester loop forever if the success path never lands. Always add a second clause: `...or if the agent refuses, end the conversation` / `...or after N turns, whichever comes first`.
  - **Concrete stop signal.** Prefer step-based (`after step 5`) or a specific trigger (`once you receive a confirmation number`) over open-ended phrasings like `"when you have what you need"` — those are interpretable and produce variable-length runs.

## Pitfalls

- Vague: "Talk to the chatbot about orders." → no persona, no goal.
- No end condition → infinite tester follow-ups.
- Predicting the agent: "After the agent shows order details..." → use "Once you receive..." instead.
- Conflicting persona: "Be angry. Stay polite." → pick one.
- Scripting exact wording when not needed → real users don't talk like form letters.
- Leaking assertions into steps: "Make sure the agent calls check_order." → the tester is a user, not an evaluator.
- Restating the persona in step 1: the persona section is already "in the tester's head"; step 1 should be the action that follows from it, not a recap.
  - Bad: `1. Tell the bakery you're vegan and want a birthday cake.`
  - Good: `1. Ask whether they have any vegan birthday cake options available this week.`
