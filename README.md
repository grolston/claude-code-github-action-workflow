# Claude Code Agent with AWS Bedrock in GitHub Actions

A versatile autonomous coding agent for your repository! No Claude Desktop needed! It runs
[Claude Code](https://code.claude.com/docs/en/github-actions) inside GitHub
Actions, with model inference served through **Amazon Bedrock**. It reacts to
GitHub issues and pull requests: implementing new issues as PRs, and iterating
on existing PRs in place when you comment on them.

The agent posts status back to the issue/PR as it works.

### ⭐ Pick the model and turn budget per task, with labels

You control **which Claude model** the agent uses and **how many turns** it may
take on a given issue or PR — just by applying labels. No config changes, no code:
add a label to the issue/PR, then trigger the agent (assign it or `@`-mention it).
Labels are read live at run time, so you can add them right before triggering.

- **Model** — apply a `model:<name>` label. Cheaper models cost less but suit
  simpler work; pricier ones handle harder tasks.
  - `model:haiku` (aka `model:cheap`) — cheapest; docs, renames, config, one-file fixes
  - `model:sonnet` — balanced; most feature/bugfix work (**default if no label**)
  - `model:opus` (aka `model:max`) — most capable; hard, multi-file, architectural work
  - `model:fable5` — Fable 5 (verify Bedrock availability first)
- **Turn budget** — apply a `max-turns:<n>` label, e.g. `max-turns:100`. This caps
  how many agent turns a run may take. Default is **60** if no label is applied.

**Examples**

- Small doc fix, keep it cheap: add `model:haiku`, then comment
  `@<agent-user> fix the typo in the README`.
- Big refactor that needs room to work: add `model:opus` **and**
  `max-turns:150`, then assign the issue to the agent.
- Everyday bugfix: add nothing — it runs on Sonnet 5 with 60 turns by default.

See the full [Labels](#labels) reference for all aliases and details.

Throughout this document, the agent's GitHub identity is referred to as
`<agent-user>`. Its actual value is configured once via the `GITHUB_AGENT_USER`
Actions variable (see below), so you never hardcode a specific username.

---

## What it does

- **New issue → new PR.** Assign an issue to the agent (or `@`-mention it) and it
  creates a feature branch, implements the change, runs tests, and opens a PR
  that references the issue.
- **Comment on a PR → updates that PR.** Mention the agent in a PR comment and it
  checks out that PR's existing feature branch, makes the requested changes, and
  **pushes to the same branch** — no new PR.
- **Reports back.** It maintains a progress comment while running and posts a
  summary of what it changed, or explains why it stopped (see *Behavior* below).

---

## Prerequisites

### 1. A GitHub App (the agent's identity)

The workflow authenticates as a dedicated GitHub App rather than the default
`GITHUB_TOKEN`, so it can push to branches and comment as a consistent identity.
The App must have:

| Permission     | Access         | Why |
| -------------- | -------------- | --- |
| Contents       | Read & write   | Push commits to feature branches |
| Pull requests  | Read & write   | Open PRs, comment, update |
| Issues         | Read & write   | Comment on issues, read labels |

The App must be **installed on the repo**, and you need its App ID, Client ID,
and a generated private key (stored as secrets, below).

### 2. AWS access to Bedrock (via OIDC)

The workflow assumes an IAM role using GitHub OIDC (no long-lived AWS keys). That
role needs:

- `bedrock:InvokeModel` / `bedrock:InvokeModelWithResponseStream` for the Claude
  models you intend to use.
- A trust policy allowing the GitHub OIDC provider for this repo.

The models must be **enabled in your Bedrock account** in the workflow's region.
Confirm each model's inference-profile ID before relying on it — some carry dated
suffixes (e.g. `us.anthropic.claude-haiku-4-5-20251001-v1:0`), others are
suffix-less (e.g. `us.anthropic.claude-opus-4-8`).

### 3. Repository secrets

Set these under **Settings → Secrets and variables → Actions**:

| Secret | Purpose |
| ------ | ------- |
| `AGENT_APP_ID` | GitHub App ID |
| `AGENT_CLIENT_ID` | GitHub App client ID |
| `AGENT_APP_PRIVATE_KEY` | GitHub App private key (PEM) |
| `AGENT_OIDC_ROLE_ARN` | ARN of the IAM role to assume for Bedrock |

### 4. Repository variable

Set this under **Settings → Secrets and variables → Actions → Variables**:

| Variable | Purpose |
| -------- | ------- |
| `GITHUB_AGENT_USER` | The agent's GitHub username / App slug |

This is the single place that defines the agent's identity. The job's trigger
conditions read it as `vars.GITHUB_AGENT_USER`, so changing this variable swaps
the agent user without editing the workflow. It must be an Actions **variable**,
not a workflow `env` — a job-level `if` cannot read the `env` context, but it can
read `vars`.

---

## Setup

1. Create/configure the GitHub App with the permissions above and install it on
   your repository.
2. Create the IAM OIDC role with Bedrock permissions and a trust policy for this
   repo; put its ARN in `AGENT_OIDC_ROLE_ARN`.
3. Enable the Claude models you want in Bedrock and confirm their
   inference-profile IDs match those in the workflow's model map.
4. Add the repository secrets.
5. Set the `GITHUB_AGENT_USER` variable.
6. Commit the workflow to `.github/workflows/` on the default branch.

---

## What triggers the workflow

The workflow listens to three events, and only runs when the trigger targets the
agent (`<agent-user>`, i.e. your `GITHUB_AGENT_USER` value):

| Event | Fires when | Agent runs if |
| ----- | ---------- | ------------- |
| `issues` (assigned) | An issue is assigned | Assignee is `<agent-user>` |
| `issue_comment` (created) | A comment is added to an issue **or a PR's conversation tab** | Comment body contains `@<agent-user>` |
| `pull_request_review_comment` (created) | A comment is added inline on a PR's diff | Comment body contains `@<agent-user>` |

Notes:

- Comments on a PR's **conversation tab** arrive as `issue_comment` events (PRs
  are issues in GitHub's model). Comments left **inline on the diff** are
  `pull_request_review_comment` events — that's why both are wired up.
- When the trigger is a comment on an existing PR, the workflow runs
  `gh pr checkout` so the agent works on that PR's branch and pushes to it.
- To avoid self-triggering loops, the job `if` ignores comments authored by the
  agent itself, comparing the comment author against
  `format('{0}[bot]', vars.GITHUB_AGENT_USER)`. This assumes the App's bot login
  is `<agent-user>[bot]`; if your App slug differs from the username, set
  `GITHUB_AGENT_USER` to the value that makes both the mention and the bot login
  resolve correctly (they're normally the same).

---

## How to use

**Implement an issue:**
Assign the issue to `<agent-user>`, or comment `@<agent-user> implement this`.
The agent opens a PR when tests pass.

> **Important:** The more detailed/precise the instructions are, the better you will do with max turns or token use. Creating an issue template for this will help.

**Iterate on a PR:**
On the open PR, comment `@<agent-user> <what to change>` — either in the
conversation tab or inline on a line of the diff. The agent updates that PR's
branch and replies with a summary.

---

## Labels

Labels on the issue/PR let you control model and turn budget per task. Labels are
read live from the API at run time, so adding a label just before triggering
works. The **first** matching label of each kind wins.

### Model selection — `model:<name>`

Claude Code only runs on Claude models, so these are the available tiers.
Approximate Bedrock on-demand cost per 1M tokens (input / output):

| Label | Aliases | Model | Cost (in/out) | Use for |
| ----- | ------- | ----- | ------------- | ------- |
| `model:haiku` | `model:cheap`, `model:haiku4-5` | Haiku 4.5 | $1 / $5 | Docs, renames, config, one-file fixes |
| `model:fable5` | `model:fable`, `model:fable-5` | Fable 5 | *confirm* | (Verify Bedrock availability/pricing) |
| `model:sonnet` | `model:sonnet5`, `model:sonnet-5` | Sonnet 5 | ~$3 / $15 | Most feature/bugfix work |
| `model:opus` | `model:max`, `model:opus4-8` | Opus 4.8 | $5 / $25 | Hard, multi-file, architectural work |

No `model:` label → **Sonnet 5** (balanced default). Label matching is
case-insensitive.

> **Important:** Validate the models are all available in the region you are using in the workflow. Update as necessary.

### Turn budget — `max-turns:<n>`

Caps how many agent turns a run may take. Default is **60**. Override per task
with e.g. `max-turns:100`. The value must be digits only and colon-tight
(`max-turns:100`, not `max-turns: 100`); anything missing or non-numeric falls
back to 60.

---

## Considerations

- **Permissions bypass.** The agent runs with `--dangerously-skip-permissions`,
  so it executes tools (git, file edits, and shell commands) without prompting.
  This is intended for the ephemeral, throwaway GitHub-hosted runner — the blast
  radius is the container. Do **not** reuse this flag on a persistent/self-hosted
  runner running as root without adding a non-root user.
  Update this to `--allowedTools Edit,Read,Write` once you validate your requirements.
- **Cost control.** Model choice is the biggest lever (5× spread from Haiku to
  Opus). `max-turns` caps worst-case spend per run. Prompt caching (on by default
  in Claude Code) discounts repeated context like `CLAUDE.md` up to ~90%, so keep
  shared context stable.
- **Fork PRs.** Pushing to a PR from a fork requires the App token (already used
  here); the default `GITHUB_TOKEN` cannot write to forks.
- **Model IDs drift.** Verify every model ID against what's enabled in your
  Bedrock account/region, or runs fail with a model-not-found / AccessDenied
  error.
- **Two failure classes.** A blocked tool that used to prompt is now a
  permissions issue (fixed by the skip flag). A `403` / `AccessDenied` is a real
  GitHub App scope or AWS IAM problem — fix the App permissions or the IAM role,
  not the workflow.

---

## Behavior on edge cases

The agent's prompt tells it to handle these explicitly rather than guessing:

- **Already implemented** — comments where it already exists (file paths), no PR.
- **Unclear requirements** — asks specific questions and stops, no PR.
- **Blocked / cannot complete** — comments what it tried, what's blocking, and
  what a human needs to do; names any partial branch. No PR with failing tests.
- **Partial scope** — completes what it can and lists done vs. remaining in the
  PR description and an issue comment.

It only opens a new PR when tests pass and the work is genuinely ready for review.
