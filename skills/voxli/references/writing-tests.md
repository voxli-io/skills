# Writing Test Instructions

Test instructions tell the AI tester how to behave during a conversation with the chatbot under test. A well-written instruction produces a predictable, repeatable conversation that you can reliably evaluate with assertions.

## The Three Essential Parts

Every test instruction needs three things:

1. **Persona** — who the tester is and what context they have
2. **Steps** — what to say and do, in order
3. **End condition** — when to stop the conversation

## Persona-Based Format

Start by establishing who the tester is. Give them a name, relevant details, and the context they need.

**Good:**
```
You're Sarah. Your order number is NS-28479, and your email is sarah@example.com.
You placed the order last week and haven't received a shipping update.
```

**Bad:**
```
You're a customer.
```
The bad version gives the tester nothing specific to work with — no name, no order number, no context.

## Step-by-Step Structure

Use numbered steps to describe the conversation flow. Each step should be a specific action.

**Good:**
```
Follow these steps:
1. Say "Can you help me with my order?"
2. When asked for your order number, provide NS-28479.
3. When asked for verification, give your email sarah@example.com.
4. Once you receive the order status, ask "when it will be delivered".
```

**Bad:**
```
Ask about your order and give your details when needed.
```
The bad version is ambiguous — "details" could mean anything, and there's no clear flow.

## Explicit End Conditions

Always tell the tester when to stop. Without an end condition, the conversation may continue aimlessly.

**Good end conditions:**
```
End: End the conversation after you receive the delivery date.
```
```
End: End the conversation after step 4, or if the agent says your order is not found.
```
```
End: Say "Thanks, goodbye" after getting your answer.
```

**Bad (no end condition):**
```
Ask about your order status.
```
This could result in an endless back-and-forth where the tester keeps asking follow-up questions.

## Example Instructions

### Order Lookup

```
You're Marcus. Your order number is NS-44821 and your email is marcus@example.com.

Follow these steps:
1. Ask the agent about the status of your order.
2. Provide your order number when asked.
3. Provide your email for verification when asked.
4. Once you receive the status, ask if you can change the shipping address.

End: End the conversation after the agent responds to your address change request.
```

### FAQ / Knowledge Base

```
You're Alex. You're considering signing up for the Premium plan but have questions.

Follow these steps:
1. Ask what features are included in the Premium plan.
2. Ask about the pricing and whether there's an annual discount.
3. Ask if you can cancel anytime or if there's a commitment period.

End: End the conversation after step 3.
```

### Escalation to Human

```
You're Jordan. You have a billing dispute that the chatbot can't resolve.

Follow these steps:
1. Explain that you were charged twice for order #BL-7291.
2. When the agent offers standard solutions, say you've already tried that and it didn't work.
3. Ask to speak with a human agent.

End: End the conversation after the agent either transfers you or provides a way to reach a human.
```

### Multi-Turn with Tool Usage

```
You're Priya. You want to book a flight from Stockholm to London for next Friday.

Follow these steps:
1. Tell the agent you want to book a flight from Stockholm to London for next Friday.
2. When presented with options, choose the cheapest one.
3. Confirm the booking when asked.
4. Ask for a confirmation number.

End: End the conversation after receiving the confirmation number, or after step 4.
```

### Edge Case — Empty Input Handling

```
You're Sam. You're going to test how the agent handles unclear messages.

Follow these steps:
1. Send just "hi".
2. When the agent asks how it can help, send "..." (just three dots).
3. When the agent responds, send a normal message: "I need help with my account."

End: End the conversation after the agent responds to your account request.
```

### Negative Test — Out-of-Scope Request

```
You're Dana. You're going to ask the agent something it shouldn't be able to help with.

Follow these steps:
1. Ask the agent to help you write a Python script.
2. If the agent refuses or redirects, accept the response.
3. If the agent tries to help, ask a follow-up coding question.

End: End the conversation after the agent's first response to your request, or after step 3.
```

## Common Pitfalls

### Too vague
```
Talk to the chatbot about orders.
```
**Why it fails:** The tester has no persona, no specific order, and no direction. Every run will produce a different, unpredictable conversation.

### No end condition
```
You're Sarah. Your order number is NS-28479.
Ask about your order status and provide your details when asked.
```
**Why it fails:** The tester might keep asking follow-up questions indefinitely. Always add "End: ..." to signal when to stop.

### Too many steps
A test with 15+ steps is hard to debug when a failure occurs somewhere in the middle. Split long flows into multiple focused tests instead.

### Conflicting instructions
```
You're angry about your order being late. Be polite and patient throughout.
```
**Why it fails:** The persona conflicts with the behavior instruction. Keep the persona and behavior consistent.

### Assuming agent behavior
```
1. Ask about your order.
2. After the agent shows the order details, ask to cancel it.
```
**Why it fails:** Step 2 assumes the agent will show order details. What if it asks for verification first? Write conditional steps: "Once you receive the order details, ask to cancel it."
