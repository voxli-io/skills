# GitHub testing setup

Setup is complete only after one real test finishes scoring and you have read its conversation.
Voxli dispatches a GitHub Actions workflow, which runs the same bridge you use locally. The bot must be reachable from the runner. You prepare the files; the user installs the GitHub App, configures credentials, and adds the GitHub agent in Voxli.

## 1. Inspect the working bridge

If there is no working bridge yet, read `get_voxli_guide("local-testing-setup")` or [local-testing-setup.md](local-testing-setup.md) first. Connect the adapter and verify a local test before moving to GitHub. Reuse its conversation loop, tool recording, per-test state, and [latency measurement](local-testing-setup.md#basic-latency). A bot started in the same job can use localhost; a bot running only on the user's laptop needs a reachable endpoint or an appropriate self-hosted runner.

This guide covers a workflow started by Voxli. Its process exit code reports bridge failures. Assertion scores arrive separately in Voxli; a green Actions job does not establish that assertions passed.

## 2. Prepare the workflow and settings

Save this as `.github/workflows/voxli-test.yml` and put the working `run_test.py` at the repository root. Adapt dependency installation, bot startup, and the command to the project's runtime. Include any additional dependencies used by the adapter.

```yaml
name: Run Voxli Test

on:
  workflow_dispatch:
    inputs:
      test_result_ids:
        description: "JSON array of assigned test result IDs"
        required: true
        type: string

permissions:
  contents: read

jobs:
  run-test:
    runs-on: ubuntu-latest
    # Leave time for interactive tester turns, but bound runner usage.
    timeout-minutes: 120
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Install dependencies
        run: python -m pip install requests python-dotenv

      - name: Run test
        env:
          VOXLI_API_TOKEN: ${{ secrets.VOXLI_API_TOKEN }}
          TEST_RESULT_IDS: ${{ inputs.test_result_ids }}
          BOT_API_KEY: ${{ secrets.BOT_API_KEY }}
          BOT_API_URL: ${{ vars.BOT_API_URL }}
          BOT_MODEL: ${{ vars.BOT_MODEL }}
        run: python run_test.py
```

`workflow_dispatch.inputs.test_result_ids` receives the JSON array from Voxli. Pass it through the step's environment as `TEST_RESULT_IDS`; do not embed the input in shell code. The job timeout bounds the whole batch. The sample bridge also bounds each conversation and exits nonzero on errors or limits.

Use the local guide's settings mapping. Replace the example `BOT_*` names with the bot's actual settings. Bot credentials go in secrets; endpoint and plain options go in variables. No real `.env` file is needed on the runner. Keep the placeholder `.env.example` in the repository. If overriding `VOXLI_API_URL`, map it from a variable and use a Voxli key from that same environment.

For several bots, keep one GitHub agent per bot. They can share a workflow when the bridge selects the bot from agent metadata. Preserve that routing if it exists. A bridge can read `GET /test-results/{id}` for `agent.id`, then `GET /agents/{id}` for the metadata. If the workflow uses fixed bot settings, a separate workflow per bot is also fine. Use GitHub Environments for separate credentials, and check their protection rules if dispatches wait for approval.

## 3. Make the workflow available

The workflow must exist on the repository's default branch for `workflow_dispatch`. After that, the GitHub agent's Branch may select another branch that also contains the workflow and bridge. See [GitHub's manual workflow instructions](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow).

Follow the user's publishing instructions. If they asked for local preparation only, leave these files local, hand off the exact remaining setup values and commands, and stop before UI setup or `run_tests`. Report that dispatch verification awaits publication. Otherwise, use the repository's normal review and merge process before trying a dispatch. Do not assume saving a local workflow makes it available to GitHub.

## 4. Hand off the UI setup with exact values

Check what is already configured. Supply the actual repository, workflow filename, branch, and required secret and variable names before asking the user to complete missing steps:

1. **Install the App:** in Voxli, open **Settings → Integrations**, click **Add integration**, choose **GitHub**, and grant the Voxli GitHub App access to the intended repository.
2. **Create and store credentials:** in **Settings → Developer**, create a Voxli API key under **API keys** (`sk-vox-...`). In the GitHub repository's **Settings → Secrets and variables → Actions**, save it as the `VOXLI_API_TOKEN` secret. Add the bot's credentials and plain settings using the names in the prepared workflow. For multiple bots, use the selected GitHub Environment for bot-specific settings.
3. **Add the agent:** on Voxli's **Agents** page, click **Connect agent** and choose **GitHub**. Set **Repository** to the chosen `owner/repo`, **Workflow File** to `voxli-test.yml` (the filename inside `.github/workflows/`), and **Branch** to the branch containing the bridge. Check the branch value explicitly; use the repository's default branch unless another branch is intended.

The MCP does not create or configure agents, GitHub integrations, or API keys. Ask the user to complete the missing UI steps and confirm. **Do not ask for secret values in chat.** Continue verification when the setup is confirmed or observable through available tools.

## 5. Verify one test

1. Use `list_agents` to identify the configured GITHUB agent.
2. Read the existing scenarios and tests. Agree on one test that uses a known tool if applicable, then call `run_tests` with `agent`, `scenarios: [{"scenario": "<scenario-id>", "tests": ["<test-id>"]}]`, and a `note` explaining the GitHub connection check. Use IDs returned by discovery tools.
3. Confirm that the intended workflow starts on the intended branch in GitHub Actions. If no GitHub tools are available, hand this check to the user. Poll `get_run_group` with the returned `run_group_id` and `status_only: true` every 15 to 30 seconds until `done` is true. Investigate a failed job or stalled run instead of polling indefinitely.
4. Read the overview with `get_run_group`, then `get_run` for each returned `run_id`. Collect each repeat's `result_id` and pass them to `get_test_results` as `result_ids`, with `include_conversation_types: ["message", "tool", "internal-event", "public-event", "action"]`. Check real bot replies, expected tool calls and responses, and completed scoring. `get_run` omits conversations; `include_conversation: true` alone omits tool metadata.
5. Report the run link, score, Actions outcome, and any limits. Without GitHub access, ask the user to confirm the final Actions outcome and mark that status unverified until confirmed. A failed assertion can still demonstrate a working bridge. A green job without a scored conversation is not enough.

## Troubleshoot

- **No workflow starts:** check App access to the repository, the workflow filename, selected branch, and presence of `workflow_dispatch` on the default branch. A rejected dispatch leaves the run's tests pending without an error. After fixing the cause, call `cancel_run` and start a fresh run.
- **401 from Voxli:** check that `VOXLI_API_TOKEN` is present, valid, and belongs to the same Voxli environment as `VOXLI_API_URL`. Use an API key, not a saved local login token.
- **Job starts but no turns happen:** verify the JSON input is mapped to `TEST_RESULT_IDS`, the adapter is implemented, and the bot is reachable with those settings. Read the bridge's error output.
- **Job hits its timeout or bridge exits:** use `get_run_group` to find unfinished work and `cancel_run` on each affected `run_id`. Fix the cause, then start a fresh run. Canceling a GitHub job alone does not complete a Voxli run.
- **Job waits for approval:** check the selected GitHub Environment's protection rules.

For exact REST schemas, follow `voxli://api-spec` and its returned resource links. Developer docs: [GitHub](https://app.voxli.io/docs/developers/github-integration) and [Local](https://app.voxli.io/docs/developers/local).
