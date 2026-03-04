# Local Testing Setup

Run Voxli tests against an agent running on your local machine, without CI round-trips. The Voxli CLI handles agent registration and polling automatically — you just provide a test command.

## What the CLI Does

- Registers your machine as a **local agent** in Voxli (visible only to you)
- Listens for test runs targeting your local agent
- Runs your command with the right environment variables when tests are assigned
- Works with both the Voxli web UI and MCP for triggering runs

## Setup

### 1. Install the CLI

```bash
npm install -g @voxli/cli
```

### 2. Authenticate

```bash
voxli auth
```

This opens a browser window to link your machine to your Voxli account. The CLI stores your credentials locally and automatically injects them when running your test command.

### 3. Write Your Test Command

Create a script that runs test conversations. The CLI passes these environment variables to your command:

| Variable | Description |
|----------|-------------|
| `VOXLI_API_TOKEN` | Your API token (injected automatically from `voxli auth`) |
| `VOXLI_TEST_RESULT_IDS` | JSON array of test result IDs to run |

Here's a minimal Python example:

```python
"""Run Voxli test conversations."""
import json
import os
import sys
import time
import requests


def get_agent_response(message: str) -> str:
    """Replace this with your actual agent integration."""
    # Call your chatbot/agent here and return its response
    return my_agent.chat(message)


def poll_next_message(endpoint, headers, timeout=30):
    start_time = time.time()
    while True:
        response = requests.post(endpoint, headers=headers)
        response.raise_for_status()
        data = response.json()
        if data["ready"]:
            return None if data.get("end_chat") else data["message"]
        if time.time() - start_time > timeout:
            raise TimeoutError("Timed out waiting for message")
        time.sleep(1)


def run_test(test_result_id, api_key, base_url):
    headers = {"Authorization": f"Bearer {api_key}"}
    conversation_url = f"{base_url}/test-results/{test_result_id}/conversation"
    next_message_url = f"{base_url}/test-results/{test_result_id}/next-message"

    # Get first message from Voxli
    tester_message = poll_next_message(next_message_url, headers)

    # Conversation loop
    while tester_message is not None:
        agent_response = get_agent_response(tester_message)

        # Record agent response
        requests.post(
            conversation_url,
            headers=headers,
            json={"type": "message", "content": agent_response}
        ).raise_for_status()

        # Get next message from Voxli
        tester_message = poll_next_message(next_message_url, headers)


def main():
    api_key = os.getenv("VOXLI_API_TOKEN")
    result_ids_raw = os.getenv("VOXLI_TEST_RESULT_IDS")
    base_url = os.getenv("VOXLI_API_URL", "https://api.voxli.io")

    if not api_key or not result_ids_raw:
        print("VOXLI_API_TOKEN and VOXLI_TEST_RESULT_IDS are required")
        sys.exit(1)

    for result_id in json.loads(result_ids_raw):
        run_test(result_id, api_key, base_url)


if __name__ == "__main__":
    main()
```

### 4. Start Listening

```bash
voxli listen --command "python run_test.py"
```

The `--command` flag specifies what the CLI runs when tests are assigned. Keep this terminal open — the CLI listens continuously for incoming test runs.

## How It Works

1. The CLI registers your machine as a local agent in Voxli
2. You trigger a test run from the Voxli UI or via MCP (`start_run` with your local agent's ID)
3. The CLI picks up the assigned tests and runs your `--command` with `VOXLI_TEST_RESULT_IDS` and `VOXLI_API_TOKEN`
4. Your script runs the conversation loop for each test
5. Voxli evaluates assertions and calculates scores once each conversation ends

## The Conversation Loop

Each test follows this pattern:

1. **Poll** `POST /test-results/{id}/next-message` — get the next message from the AI tester
2. **Respond** — pass the message to your agent and get a response
3. **Record** `POST /test-results/{id}/conversation` — send the agent's response back to Voxli
4. **Repeat** — poll again for the next message
5. **Stop** — when `end_chat: true` is returned, the conversation is over

The polling endpoint returns `{ "ready": false }` while Voxli is generating the next message. Keep polling until `ready` is `true`.

## Recording Tool Calls

If your agent calls tools during the conversation, you can record them for visibility in the results:

```python
requests.post(
    f"{base_url}/test-results/{result_id}/conversation",
    headers=headers,
    json={
        "type": "public-event",
        "name": "check_order",
        "metadata": {
            "order_id": "NS-28479",
            "result": {"status": "shipped"}
        }
    }
)
```

Tool calls recorded this way appear in the conversation transcript and can be verified by assertions.

## Reusing the Script for CI

The CLI test command uses the same contract as the GitHub integration — `VOXLI_API_TOKEN` and `VOXLI_TEST_RESULT_IDS`. If you write your test script for local development, you can reuse it in a GitHub Actions workflow with no changes.

## Troubleshooting

- **Agent not showing up in Voxli**: Make sure `voxli listen` is running. The agent appears as online only while the CLI is actively listening.
- **Tests not picked up**: Verify you selected your local agent when starting the run. Local agents are specific to your machine.
- **Authentication errors**: Run `voxli auth` again to refresh your credentials.
