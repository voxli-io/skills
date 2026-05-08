# Local Testing Setup

Run Voxli tests against an agent on the user's own machine using the Voxli CLI. The CLI registers the machine as a local agent, listens for assigned test runs, and invokes a user-provided command.

For exact API payloads, endpoints, and a working code sample, point the user to the official Voxli documentation and the `@voxli/cli` README. This file only explains the concepts the skill needs to know.

## Core concepts

### Local agent
The CLI registers the user's machine as an "agent" inside Voxli, visible only to that user. Tests can target it the same way they target any other agent.

### Listener command
`voxli listen --command "<your command>"` keeps the CLI running. When Voxli assigns tests to this machine, the CLI executes the command, passing test IDs in via environment variables.

### Environment contract
The user's command receives:

| Variable | Purpose |
|----------|---------|
| `VOXLI_API_TOKEN` | Auth token, injected by the CLI from `voxli auth` |
| `VOXLI_TEST_RESULT_IDS` | JSON array of test result IDs the command should run |

The same contract is used by the GitHub integration, so a script written for local development works in CI without changes.

### Conversation loop
For each test result ID, the user's command runs a loop:

1. Poll Voxli for the next message from the simulated user.
2. Pass the message to the agent under test, capture the response.
3. Send the agent's response back to Voxli.
4. Optionally record `tool`, `internal-event`, or `public-event` entries (see the visibility table in `SKILL.md`) before sending the response.
5. Stop when Voxli signals end-of-chat.

Voxli evaluates assertions and computes the score after the conversation ends.

## Setup steps (high level)

1. `npm install -g @voxli/cli`
2. `voxli auth` (browser-based login).
3. Write a test command that implements the conversation loop above.
4. `voxli listen --command "<command>"` — keep this terminal open. The user starts this themselves; an assistant should not.

For working code, exact endpoints, and request/response shapes, refer to the Voxli documentation.

## Troubleshooting (quick)

- **Local agent missing in Voxli** → `voxli listen` isn't running.
- **Tests not picked up** → wrong agent selected when starting the run.
- **Auth errors** → re-run `voxli auth`.
