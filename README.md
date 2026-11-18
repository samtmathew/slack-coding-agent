# Patch: a coding agent that lives in Slack

Patch is an AI coding agent you talk to in Slack. Report a bug or ask for a small
feature, and Patch reads the codebase, makes the change in its own sandboxed
machine, checks its work, and opens a pull request on GitHub after you approve
it with a button in Slack.

I built this project to learn **LangChain Deep Agents**, specifically
[managed deep agents](https://docs.langchain.com/langsmith/python/managed-deep-agents-overview)
hosted on LangSmith. It covers the main building blocks of a production agent:
instructions, skills, sandboxes, middleware, MCP tools, credentials,
human-in-the-loop approval, chat channels, schedules, and long-term memory.

## What it does

- **Fixes bugs:** finds the root cause, fixes every occurrence, and refuses to
  "fix" code that already behaves correctly.
- **Builds small features:** follows the existing code's patterns and keeps the
  docs in sync. It turns down requests too big for one pull request.
- **Opens pull requests:** creates a branch, pushes one commit, and opens a PR
  with a structured description. A person must approve every PR before it opens.
- **Works from Slack:** DM it or @mention it in a channel, and attach
  screenshots of bugs. It replies in the thread with the PR link and the diff.
- **Posts a PR roundup:** every weekday morning it lists open PRs in a Slack
  channel and flags any open more than 3 days.
- **Remembers:** facts about the repo are shared across everyone; each
  person's preferences (such as "open my PRs as drafts") stay private to them.

## How it works

```
Slack message ──► Patch (LangSmith deployment, Claude model)
                    │
                    ├─ middleware/checkout.py   clones the target repo into the sandbox
                    ├─ instructions.md          its job, rules, and how to verify work
                    ├─ skills/                  step-by-step procedures, loaded on demand
                    ├─ sandbox/                 isolated machine per conversation (git, node)
                    ├─ tools/mcp.py             GitHub MCP tools: branch, push, open PR
                    │       └─ approval required before github_create_pull_request
                    └─ memory.py                shared + per-user memory in Context Hub
                    │
                    ▼
            Pull request on GitHub, reply + diff in Slack
```

## Project structure

| Path | Purpose |
|---|---|
| `agent.py` | Agent definition: name, model, middleware, and the approval step before a PR opens |
| `instructions.md` | System prompt: the agent's job, rules, how it verifies changes, and how it replies |
| `skills/` | Procedures loaded only when needed: `fix-a-bug`, `add-a-feature`, `open-a-pull-request` |
| `middleware/checkout.py` | Clones the target repo into the sandbox before every turn |
| `sandbox/` | Per-conversation sandbox; `setup.sh` installs git and Node into its snapshot |
| `tools/mcp.py` | GitHub's MCP server, limited to the four tools the agent needs |
| `github_headers.py` | Workaround headers required by GitHub's MCP server |
| `channels/slack.py` | Slack integration (DMs and @mentions) |
| `schedules/pr_roundup.py` | Weekday-morning open-PR roundup posted to Slack |
| `memory.py` | Shared agent memory and private per-user memory |
| `identity.py` | Who may call the deployment (LangSmith API keys) |
| `config.py` | The target repository and where it is checked out |

## Prerequisites

- [uv](https://docs.astral.sh/uv/getting-started/installation/) (installs Python 3.14 and the dependencies)
- git
- A [LangSmith](https://smith.langchain.com) account with a workspace
- An [Anthropic](https://console.anthropic.com) API key
- A **public** GitHub repository for Patch to work on. To try it out, fork
  [ishaan-lal/simple-game](https://github.com/ishaan-lal/simple-game), a small
  browser Tetris game.
- A Slack workspace where you can install apps
- Chrome or Firefox for LangSmith Studio (Safari can block the local connection)

## Configuration

Copy the template and fill it in:

```bash
cp .env.example .env
```

| Variable | Required | What it is | Where to get it |
|---|---|---|---|
| `LANGSMITH_API_KEY` | Yes | LangSmith API key | LangSmith → **Settings → API Keys**. Create it in the same workspace as below |
| `LANGSMITH_WORKSPACE_ID` | Yes | ID of the LangSmith workspace the agent deploys into | LangSmith → **Settings → Workspaces** |
| `ANTHROPIC_API_KEY` | Yes | API key for the Claude model | [console.anthropic.com](https://console.anthropic.com) |
| `PATCH_REPO` | Yes | Repo the agent works on, as `owner/name` (must be public) | e.g. `your-username/simple-game` |
| `MDA_DEV_PATCH_GITHUB` | For PRs | Fine-grained GitHub token for `PATCH_REPO` | See [GitHub token](#github-token) below |

Two values live in code rather than `.env`:

- **Agent name**, `name="patch"` in `agent.py`. It is also the deployment
  name, so it must be unique in your LangSmith workspace.
- **Slack channel ID** for the PR roundup, `conversation_id` in
  `schedules/pr_roundup.py`. Open the channel in Slack → channel name →
  **About** → **Channel ID** (starts with `C`), then `/invite` the bot there.
  Schedule values must be literals, so this can't come from `.env`.

### GitHub token

Create a fine-grained token at
[github.com/settings/personal-access-tokens/new](https://github.com/settings/personal-access-tokens/new):

| Field | Value |
|---|---|
| Repository access | **Only select repositories** → your `PATCH_REPO` |
| Permissions → Contents | Read and write |
| Permissions → Pull requests | Read and write |

Paste it into `.env` as `MDA_DEV_PATCH_GITHUB`. The token never appears in code:
locally it's read from `.env`, and in production it's stored in LangSmith as a
named connection.

## Setup and deployment

`uv run mda dev` and `uv run mda deploy` both check that the `patch-github`
connection exists in your LangSmith workspace, and a connection can only be
created once a deployment exists. So the first time, deploy without the
connection:

```bash
uv sync

# 1. In tools/mcp.py, comment out the line:
#      "connection": connections.get("patch-github", {"type": "agent"}),
#    then deploy once so the deployment exists.
uv run mda deploy

# 2. Store the GitHub token in LangSmith as the `patch-github` connection.
uv run mda connections create patch-github --secret-from-env MDA_DEV_PATCH_GITHUB

# 3. Restore the `connection` line in tools/mcp.py and deploy again.
uv run mda deploy
```

After that, both commands work normally.

On deploy, the CLI prints a **Slack link**. Open it while signed in to your
Slack workspace and install the app. When the deploy finishes, the bot DMs
you. There is no Slack app to create and no bot token to manage.

## Running locally

```bash
uv run mda dev
```

LangSmith Studio opens in your browser. In the graph dropdown, pick `patch`,
not `patch--general-purpose`. If Chrome asks to let the site connect to devices
on your local network, click **Allow**.

## Using it

In a DM with the bot, or by @mentioning it in a channel:

> The level goes up way too fast. I'm on level 3 after only 10 lines. Can you fix it?

> Add a key that toggles the ghost piece on and off. Use G.

> From now on, open my PRs as drafts.

Patch makes the change, then shows **Approve / Reject** buttons before it opens
the pull request. Follow-up requests in the same thread add commits to the
same PR.

## Cleaning up

The deployment and its schedule keep running until you delete them:

```bash
uv run mda delete
uv run mda connections delete patch-github
```

Then revoke the GitHub token.

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Please use typing_extensions.TypedDict … on Python < 3.12` | The project needs Python 3.12+ (pinned to 3.14 in `.python-version`). Delete `.venv` and `.mda`, then run `uv sync` again |
| `LangSmith rejected LANGSMITH_API_KEY … (HTTP 403)` | The key is from a different workspace than `LANGSMITH_WORKSPACE_ID`, or an old key exported in your shell overrides `.env` (`unset LANGSMITH_API_KEY`) |
| Studio: "Failed to fetch" | The browser blocked the local connection. Allow **Local network access** for smith.langchain.com in Chrome's site settings |
| "Set PATCH_REPO in .env" / "Could not clone" | Set `PATCH_REPO` as `owner/name` and make sure the repo is public |
| `403 Resource not accessible by personal access token` | Set the token's **Contents** and **Pull requests** permissions to Read and write |
| `connections declared by this project are not in the workspace` | Create the `patch-github` connection (see [Setup and deployment](#setup-and-deployment)) |
| `connections create`: `No deployment named '…' exists` | Deploy once without the `connection` line first (see [Setup and deployment](#setup-and-deployment)) |
| Run never pauses before the PR | The `interrupt_on` key must be `github_create_pull_request` |
| Bot doesn't answer in a channel | `/invite` it first. In channels it answers @mentions; DMs always work |

## Built with

- [LangChain Deep Agents](https://docs.langchain.com/langsmith/python/managed-deep-agents-overview) (`managed-deepagents`) on LangSmith
- Anthropic Claude
- GitHub MCP server
- Slack

## Acknowledgements

Based on the LangChain managed deep agents workshop by
[Ishaan Lal](https://github.com/ishaan-lal), which I worked through to learn the
framework.
