# Local testing setup

Setup is complete only after one real test finishes scoring and you have read its conversation.
A bridge relays Voxli's simulated user turns to the user's bot and records the bot's replies and tool calls. You write and check the bridge; the user authenticates and starts the listener in their terminal.

## 1. Inspect the bot

Read the repository before asking questions. Find its runtime and entry point, how it keeps conversation state, where tools execute or traces are exposed, and which settings and credentials it needs. Reuse an existing bridge, settings loader, and bot selection command when present. Ask only for what the repository cannot answer. Use the project's existing runtime and package runner when adapting the Python example below to another language.

## 2. Write the bridge

Save the script below as `run_test.py`. Install `requests` and `python-dotenv` in the project's environment and record them in its dependency file. For a standalone Python project: `python -m pip install requests python-dotenv`.

Adapt `get_agent_response(session_id, turn)` to call the real bot. Keep state across turns and isolate it for each `session_id` (the assigned test ID). The full tester turn contains `message` or `action`, plus `message_id` when available. For text, pass `turn.get("message") or ""`. For an action, apply its `name` and `arguments` in the bot. A text-only adapter should raise an error for unsupported actions. Put timeouts on the bot's own network calls too; the loop cannot interrupt a blocking adapter.

Return a list of Voxli conversation entries in their original order. Each reply is a `message` entry; tools and events use the shapes below. Keep all replies when the bot sends several, tools before the replies using them, and button or handoff events where they occurred. The stub raises `NotImplementedError` until connected. The sample uses automatic completion: it stops at `end_chat`, then you check scoring through the MCP.

<!-- Keep this code block identical to the docs-site github-example.py sample. -->
```python
"""Relay Voxli conversations to your bot locally or in GitHub Actions.

Requires requests and python-dotenv. Adapt get_agent_response before running.
VOXLI_API_TOKEN and TEST_RESULT_IDS come from the CLI or workflow environment.
"""

import json
import os
from pathlib import Path
import time

from dotenv import load_dotenv
import requests

SESSION_DEADLINE_SECONDS = 90 * 60
MAX_TURNS = 200
POLL_INTERVAL_SECONDS = 2
HTTP_TIMEOUT_SECONDS = 60


def get_agent_response(session_id: str, turn: dict) -> list[dict]:
    """Call your bot and return ordered Voxli conversation entries.

    Keep state per session_id. Handle turn["action"] when present, otherwise
    send turn.get("message") or "". Put tool entries before replies using them.
    """
    raise NotImplementedError("Connect your bot and its tool execution traces here")


def load_bot_settings() -> None:
    path = Path(os.environ.get("BOT_ENV", Path(__file__).with_name(".env")))
    if "BOT_ENV" in os.environ and not path.is_file():
        raise FileNotFoundError(f"Bot settings file does not exist: {path}")
    load_dotenv(path, override=False)


def voxli_request(
    client: requests.Session,
    method: str,
    url: str,
    body: dict | None = None,
    timeout: float = HTTP_TIMEOUT_SECONDS,
) -> requests.Response:
    response = client.request(method, url, json=body, timeout=timeout)
    response.raise_for_status()
    return response


def remaining_timeout(deadline: float) -> float:
    remaining = deadline - time.monotonic()
    if remaining <= 0:
        raise TimeoutError("Conversation deadline reached")
    return min(HTTP_TIMEOUT_SECONDS, remaining)


def poll_next_message(client: requests.Session, endpoint: str, deadline: float) -> dict:
    while True:
        data = voxli_request(
            client, "POST", endpoint, timeout=remaining_timeout(deadline)
        ).json()
        remaining_timeout(deadline)
        if data["ready"]:
            return data
        time.sleep(min(POLL_INTERVAL_SECONDS, remaining_timeout(deadline)))


def run_test(test_result_id: str, client: requests.Session, base_url: str) -> None:
    endpoint = f"{base_url}/test-results/{test_result_id}"
    deadline = time.monotonic() + SESSION_DEADLINE_SECONDS
    for turn_number in range(MAX_TURNS + 1):
        turn = poll_next_message(client, f"{endpoint}/next-message", deadline)
        if turn.get("end_chat"):
            print(f"Test {test_result_id}: conversation ended; check scoring in Voxli")
            return
        if turn_number == MAX_TURNS:
            raise RuntimeError(f"Conversation exceeded {MAX_TURNS} turns")

        entries = get_agent_response(test_result_id, turn)
        if not entries:
            raise ValueError("Bot returned no messages or events")
        for entry in entries:
            voxli_request(
                client,
                "POST",
                f"{endpoint}/conversation",
                entry,
                timeout=remaining_timeout(deadline),
            )


def main() -> int:
    try:
        # Bot env files must not supply CLI/workflow inputs.
        api_key = os.environ["VOXLI_API_TOKEN"]
        test_result_ids = json.loads(os.environ["TEST_RESULT_IDS"])
        base_url = os.getenv("VOXLI_API_URL", "https://api.voxli.io").rstrip("/")
        load_bot_settings()
        if not api_key or not base_url:
            raise ValueError("VOXLI_API_TOKEN and VOXLI_API_URL must not be empty")
        if not isinstance(test_result_ids, list) or not test_result_ids:
            raise ValueError("TEST_RESULT_IDS must be a nonempty JSON array")
        if any(
            not isinstance(item, str) or not item.strip() for item in test_result_ids
        ):
            raise ValueError("TEST_RESULT_IDS must contain nonempty strings")
    except (KeyError, ValueError, OSError) as error:
        print(f"Bridge configuration failed: {error}")
        return 1

    failed = False
    with requests.Session() as client:
        client.headers["Authorization"] = f"Bearer {api_key}"
        for test_result_id in test_result_ids:
            try:
                run_test(test_result_id, client, base_url)
            except Exception as error:
                failed = True
                print(f"Test {test_result_id}: bridge failed: {error}")
                try:
                    voxli_request(
                        client,
                        "PATCH",
                        f"{base_url}/test-results/{test_result_id}",
                        {"status": "failed"},
                    )
                except Exception as status_error:
                    print(
                        f"Test {test_result_id}: could not mark failed: {status_error}"
                    )
    return 1 if failed else 0


if __name__ == "__main__":
    raise SystemExit(main())
```

The script starts with `POST /test-results/{id}/next-message`. It polls `ready: false`, passes each text or action turn to the adapter, and records its entries through `POST /test-results/{id}/conversation`. Voxli already records each tester turn. Do not post it a second time. If the bot greets first, create its session and record that greeting before the first poll. For traces that arrive after the chat, see manual completion below.

Deadlines, turn limits, HTTP errors, and adapter failures produce a nonzero exit. The script tries to mark the affected test `failed` through `PATCH /test-results/{id}`, then attempts the remaining assigned tests. An exit code of zero means the conversations ended; scoring still needs verification.

## 3. Record tool calls

**If the bot executes tools, record each call before the reply that uses its output.** Read the bot's execution hooks or returned traces. Never infer calls from reply text. If the platform exposes only replies or observed actions, report that limit and do not claim internal tool verification is complete.

For new bridges, use this tool shape followed by the reply (illustrative values):

```json
[
  {
    "type": "tool",
    "name": "lookup_order",
    "metadata": {
      "id": "call_1",
      "arguments": {"order_id": "example-order"},
      "response": {"status": "shipped"}
    }
  },
  {"type": "message", "content": "Your order has shipped."}
]
```

Use the actual call ID, arguments, and response. If the runtime has no call IDs, assign a unique ID when capturing each execution. The API accepts arbitrary JSON metadata, so an existing bridge can preserve fields such as `input` and `output`. Tool-call assertions and hallucination detection read that metadata. Evidence must appear at or before the claim's turn; recording only a tool name loses it.

If a hosted integration already records tools in Voxli, verify its session is linked to this test and omit those same calls from the bridge's entries. Do not record them twice.

## 4. Configure bot settings

Keep bot credentials, its endpoint, and options in `.env` beside `run_test.py`. Add `.env` and `bots/*.env` to `.gitignore`; track a placeholder `.env.example` with the required names. Use the names the actual bot expects. For example:

```dotenv
BOT_API_URL=https://your-bot.example/chat
BOT_API_KEY=replace-locally
BOT_MODEL=your-model
```

Real environment variables take precedence over the file. The sample uses `python-dotenv`; keep the project's settings loader if it already has one. Never put `VOXLI_API_TOKEN` or `TEST_RESULT_IDS` in a bot env file. The sample reads these from the environment before loading bot settings. Do not ask for secret values in chat.

The CLI supplies the two required variables:

| Variable | Purpose |
|----------|---------|
| `VOXLI_API_TOKEN` | Authentication injected by the CLI after `voxli auth` |
| `TEST_RESULT_IDS` | JSON array of assigned test result IDs |

`VOXLI_API_URL` is an optional override for the bridge's default `https://api.voxli.io`. When using another Voxli environment, set it before both `voxli auth` and `voxli listen` so the CLI and bridge use the same API. That API controls which app the login flow redirects to.

| Setting | Local source | GitHub Actions source |
|---------|--------------|-----------------------|
| Bot credentials, such as `BOT_API_KEY` | Gitignored `.env` | Secret |
| Bot endpoint and plain options, such as `BOT_API_URL` and `BOT_MODEL` | `.env` or environment | Variable |
| Voxli token (`VOXLI_API_TOKEN`) | Injected by the CLI | Secret containing a Voxli API key |
| Assigned test result IDs (`TEST_RESULT_IDS`) | Injected by the CLI | Workflow input `test_result_ids` |
| Voxli API URL (`VOXLI_API_URL`), when overridden | Environment shared by CLI and bridge | Variable for the same Voxli environment as the key |

For several bots, keep one named listener per bot. Reuse the project's bot selector, such as `--bot support`. The sample selects a `bots/<name>.env` file with `BOT_ENV`, relative to the command's working directory:

```bash
BOT_ENV=bots/support.env voxli listen --name support --command "python run_test.py"
```

`--name` identifies the local agent; it does not select a settings file. Give each bot its own placeholder env example too.

## 5. Ask the user to start listening

Prepare the script and settings first. If the user requested file preparation only, stop here and hand off the files and remaining commands, marking live verification as pending. Otherwise, give the user the exact commands for their project's environment and working directory. For the standalone example:

```bash
npm install -g @voxli/cli
voxli auth
voxli listen --name my-bot --command "python run_test.py"
```

**The user must start `voxli listen` in their terminal. Do not start it yourself.** Wait for confirmation, then call `list_agents` and find the LOCAL agent named by `--name`. `voxli auth` also registers an agent named after the machine; do not target that one. `list_agents` does not report online status, so ask the user to confirm it in the app's **Local agents** menu (the terminal icon in the top bar). Keep that terminal open. The CLI registers the machine with `POST /agents/register` and polls for work; the bridge does not register itself.

## 6. Verify one test

1. Read existing scenarios and tests. Agree on one test with the user; choose one that exercises a known tool if the bot uses tools. Read `writing-tests` and `writing-assertions` through `get_voxli_guide` before creating anything missing.
2. Call `run_tests` with `agent`, `scenarios: [{"scenario": "<scenario-id>", "tests": ["<test-id>"]}]`, and a `note` explaining the connection check. Use IDs returned by discovery tools.
3. Poll `get_run_group` with the returned `run_group_id` and `status_only: true` every 15 to 30 seconds until `done` is true. If the bridge exits or progress stalls, diagnose it instead of polling indefinitely.
4. Read `get_run_group` once for the overview, then `get_run` for each returned `run_id`. Collect each repeat's `result_id` and pass them to `get_test_results` as `result_ids`, with `include_conversation_types: ["message", "tool", "internal-event", "public-event", "action"]`. Check real bot replies and the known tool's arguments and response before the reply using them. `get_run` omits conversations; `include_conversation: true` alone omits tool metadata.
5. Report the returned run link, score, and any integration limits. A failed assertion can still mean the bridge works. Missing replies, missing expected tools, or incomplete scoring mean setup needs more work.

## Troubleshoot

- **Agent missing or offline:** verify the user's listener is running and authenticated to the same workspace. A local agent is offline after five minutes without a heartbeat.
- **Work never reaches the bot:** check that `run_tests` targeted this listener's agent, and that `BOT_ENV` points at the intended bot settings.
- **Authentication error:** rerun `voxli auth` in the correct Voxli environment. An inherited `VOXLI_API_TOKEN` overrides saved CLI credentials; remove a stale override first.
- **Bridge crashes or hits a limit:** the sample marks the test failed when it can. If the process was killed or the status update failed, read the run overview and call `cancel_run` for each affected unfinished `run_id`. Fix the cause, then start a fresh run.
- **`ready: false` keeps returning:** it is normal while an interactive tester waits for a person. For automated tests, check run status and the last recorded reply. Respect the deadline; do not keep retrying a failed run.

## Reference

### Events and actions

Return events alongside messages with `type`, `name`, and JSON `metadata`, using the same conversation endpoint. For example:

```json
{"type": "internal-event", "name": "handoff", "metadata": {"queue": "support"}}
```

`internal-event` is visible to scoring; `public-event` is also visible to the tester. See the visibility table in `get_voxli_guide("writing-assertions")` or [writing-assertions.md](writing-assertions.md).

To expose a button or form, record an event with `metadata.actions`: each action needs a unique snake_case `name`, a `description`, optional primitive JSON Schema `properties`, and optional `required` property names. Actions last for the current bot turn; re-register them if still available on the next turn. The adapter receives the chosen action in `turn["action"]` and must apply it. Do not send its serialized text as a chat message. See [Tools, Events, and Actions](https://app.voxli.io/docs/developers/tool-calls) for the full action schema and example.

### Basic latency

Start with the time from sending a tester message until the bot call returns. Measure it around the bot call with `time.monotonic()` and convert seconds to milliseconds, or use the platform's reported latency. Put the number in `metadata.metrics.latency_ms` on the reply. For example, this entry uses an illustrative latency of 850 milliseconds:

```json
{
  "type": "message",
  "content": "How can I help?",
  "metadata": {"metrics": {"latency_ms": 850}}
}
```

If one bot call produces several replies, attach its latency only to the first message. Omit unavailable measurements; do not substitute zero. The sample preserves the adapter's metadata.

In **Settings → Metrics**, create a metric with **Key** `latency_ms`, **Label** "Response time", **Unit** **Milliseconds**, and **Aggregates** **avg**, then choose its placement in **Layout**. Match the key exactly. See [Custom Metrics](https://app.voxli.io/docs/documentation/custom-metrics) for the settings. Each metadata payload is limited to 100 KB.

### Late traces

For data that arrives after the chat, set `completionMode` to `manual` through `PATCH /test-results/{id}` **before the conversation ends**. After `end_chat: true`, insert late tools or events through `POST /test-results/{id}/conversation`, using `before` or `after` with an existing message ID to put them in the correct order. Merge metadata with `PATCH /test-results/{id}/conversation/{message_id}`. Finish all writes, then call `POST /test-results/{id}/finish` to queue scoring and poll the run as above. Insertion alone does not delay scoring. Once processing starts, positioned inserts and metadata updates are rejected, and later appends may be missed by scoring.

### Schemas and next step

The guide contains the basic bridge contract even when the MCP client cannot browse resources. For exact endpoint schemas, read `voxli://api-spec`, follow its tag resources `voxli://api-spec/{tag}`, then the returned `voxli://api-spec/{tag}/{operationId}` links. Do not guess operation IDs.

Developer docs: [Local](https://app.voxli.io/docs/developers/local), [Tools, Events, and Actions](https://app.voxli.io/docs/developers/tool-calls).

To run the bridge in GitHub Actions, read `get_voxli_guide("github-testing-setup")` or [github-testing-setup.md](github-testing-setup.md).
