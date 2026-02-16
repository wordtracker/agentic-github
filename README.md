# Agentic

Reusable Claude Code skills and GitHub Actions composite actions for automated PR lifecycle management.

## Skills

- **plan-work** — creates GitHub issues through structured conversation
- **do-work** — implements work from a GitHub issue
- **pr-start** — implements a GitHub issue in CI, writes output files for the workflow to commit and open a PR
- **pr-start-local** — implements a GitHub issue locally, creates branch, commits, and opens a PR
- **pr-fix** — fixes code based on reviewer feedback
- **gh-respond** — answers questions and responds to PR or issue comments
- **pr-merge** — writes clean commit message for squash-merging approved PRs
- **commit** — atomic git commit conventions

## PR Workflow

A system where an AI agent handles the full PR lifecycle. All interactions require addressing the bot with `@claude` at the start of a review or comment. When a reviewer requests changes, the AI reads the feedback, fixes the code, and pushes for re-review. When approved, the AI writes a clean commit message and merges.

## The Players

- **You** (human): Create issues, trigger work, review PRs, approve or reject
- **Claude** (local): Runs on your machine. Helps plan issues, executes work, opens PRs
- **GClaude** (GitHub Actions): Runs in GitHub's cloud on issue and PR events. Implements, fixes, responds, or merges

## The Loop

```
You create a GitHub issue (or /plan-work helps you)
       ↓
@claude /pr-start on the issue (or /pr-start-local)
       ↓
Claude implements and opens a PR (Closes #issue)
       ↓
You review the PR
       ↓
@claude + request changes?
       ↓
GClaude fixes, writes commit msg + PR comment
       ↓
Workflow commits, pushes, and posts comment
       ↓
Back to review
```

## Step by Step

### 1. Planning (local, you + Claude)

You run `/plan-work`. Claude asks what you want to build, researches the codebase, asks clarifying questions, then creates a GitHub issue with the plan.

### 2. Execution (GitHub Actions or local)

Comment `@claude /pr-start` on the issue. GClaude implements the plan, commits with `issue: #N` in the body, and opens a PR with `Closes #N`.

Alternatively, run `/pr-start-local` to do this on your machine.

(`/do-work` can also be used standalone if you just want to implement an issue without the PR lifecycle.)

### 3. Review (GitHub, human)

A human reviews the PR. To trigger the bot, start a review or comment with `@claude`. Without `@claude`, nothing happens — the bot only responds when addressed directly.

### 4. pr-fix (GitHub Actions, GClaude)

Runs when a review with `changes_requested` starts with `@claude`:

1. Checks out the repo at the PR branch
2. Runs `gather-pr-context` to collect PR data, reviews, comments, commit history, and the linked issue into `context.txt`
3. Runs Claude with the pr-fix skill
4. Claude reads the review comments, finds the linked issue for original context
5. Claude fixes all requested changes in one pass
6. Claude writes commit message + PR comment to files
7. The workflow commits, pushes, and posts the comment

The loop returns to the review step.

### 5. gh-respond (GitHub Actions, GClaude)

Runs when a PR or issue comment starts with `@claude`:

1. Checks out the repo (PR branch for PRs, main for issues)
2. Gathers context (PR context or issue context depending on source)
3. Runs Claude with the gh-respond skill
4. Claude reads the comment, explores the codebase as needed, and writes a response
5. The workflow posts the response as a comment

### 6. pr-merge (GitHub Actions, GClaude)

Runs when a review with `approved` starts with `@claude`:

1. Checks out the repo at the PR branch
2. Runs `gather-pr-context` to collect PR data, reviews, comments, commit history, and the linked issue into `context.txt`
3. Runs Claude with the pr-merge skill
4. Claude reads the commit history and linked issue, writes a clean commit message
5. The workflow squash-merges the PR with that message

The messy history of fix commits disappears. Main gets one clean commit.

## UI Testing (Conditional)

For frontend work, GClaude can spin up a dev server and use Chrome DevTools to:

- Take screenshots
- Check console for errors
- Verify the UI matches expectations

This only happens when the plan involves UI work. The GitHub Actions runner has Chrome pre-installed. GClaude starts it in headless mode and connects via the Chrome DevTools MCP.

## Bot Identity

In GitHub Actions, the workflow sets git identity directly using `AGENTIC_BOT_NAME` and `AGENTIC_BOT_EMAIL` variables, and authenticates with `AGENTIC_BOT_TOKEN`. Claude never runs git or gh commands — the workflow handles all git and GitHub operations.

For local use, `bin/git-bot` is a wrapper script that sets the bot's identity via environment variables. This ensures commits, PRs, and comments are attributed to your machine account, not your personal GitHub account.

### Setting up a GitHub machine account

1. Create a new GitHub account for your bot (e.g. `yourname-bot`)
2. Add the bot account as a collaborator on your repos (with write access)
3. On the bot account, generate a fine-grained Personal Access Token (PAT):
   - Settings → Developer settings → Personal access tokens → Fine-grained tokens
   - Repository access: **All repositories** (covers any repo the bot is added to in the future)
   - Permissions:
     - **Contents**: Read and write (for git push)
     - **Pull requests**: Read and write (for creating/commenting/merging PRs)
     - **Metadata**: Read (auto-selected)
4. Note the bot's noreply email from https://github.com/settings/emails — it looks like `ID+username@users.noreply.github.com`

### Environment variables

`git-bot` requires three env vars:

| Variable | Value |
|---|---|
| `AGENTIC_BOT_NAME` | The bot's GitHub username (e.g. `yourname-bot`) |
| `AGENTIC_BOT_EMAIL` | The bot's noreply email (e.g. `ID+yourname-bot@users.noreply.github.com`) |
| `GH_TOKEN` | The bot's PAT |

**Locally**, set these in your shell profile or a `.env` file:

```bash
export AGENTIC_BOT_NAME="yourname-bot"
export AGENTIC_BOT_EMAIL="123456+yourname-bot@users.noreply.github.com"
export GH_TOKEN="ghp_..."
```

**In GitHub Actions**, add `AGENTIC_BOT_TOKEN` as a repo/org secret, and `AGENTIC_BOT_NAME` / `AGENTIC_BOT_EMAIL` as repo/org variables (Settings → Secrets and variables → Actions).

## Composite Actions

This repo provides 6 composite actions that can be composed into workflows:

- **setup-claude-agent** — Sets up Claude CLI, skills repo, dependencies, mkcert, and Chrome DevTools MCP
- **run-claude-skill** — Runs Claude with a specified skill and context (issue or PR). Inherits env vars from job.
- **git-commit-push** — Commits and pushes changes, optionally creating a new branch
- **post-comment** — Posts a comment to an issue or PR from `/tmp/comment.txt`
- **create-pr** — Creates a PR with optional issue linking from `/tmp/commit_msg.txt` and `/tmp/comment.txt`
- **squash-merge** — Squash merges a PR with custom commit message from `/tmp/commit_msg.txt`

### Environment Variable Inheritance

Composite actions automatically inherit environment variables from the job. Set env vars at the job level and they'll be available to all steps, including those in composite actions:

```yaml
jobs:
  pr-fix:
    env:
      VITE_PROXY_TARGET: ${{ secrets.VITE_PROXY_TARGET }}
      VITE_HOST: localhost
    steps:
      - uses: chriswickett/agentic/.github/actions/run-claude-skill@main
        # VITE_PROXY_TARGET and VITE_HOST are automatically available
```

Secrets and tokens must still be passed explicitly as inputs.

## Repo Setup

Before using these workflows with a client repo:

1. **Disable auto-merge**: Settings → General → Pull Requests. Make sure "Allow auto-merge" is OFF. GClaude controls when merges happen.

2. **Allow squash merging**: Same section, make sure "Allow squash merging" is ON.

3. **Add workflow files**: Create four workflow files in `.github/workflows/`. All workflows require `@claude` at the start of the triggering comment/review for security.

   `gh-commented.yml` — Responds to issue comments and new issues:
   ```yaml
   on:
     issue_comment:
       types: [created]
     issues:
       types: [opened]

   jobs:
     gh-respond:
       if: (github.event_name == 'issue_comment' && startsWith(github.event.comment.body, '@claude')) || (github.event_name == 'issues' && startsWith(github.event.issue.body, '@claude'))
       runs-on: ubuntu-latest
       timeout-minutes: 5
       env:
         VITE_PROXY_TARGET: ${{ secrets.VITE_PROXY_TARGET }}
         VITE_HOST: localhost
       steps:
         - uses: actions/checkout@v4

         - uses: chriswickett/agentic/.github/actions/setup-claude-agent@main

         - uses: chriswickett/agentic/.github/actions/run-claude-skill@main
           with:
             skill-name: gh-respond
             context-type: issue
             context-number: ${{ github.event.issue.number }}
             repository: ${{ github.repository }}
             claude-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}

         - uses: chriswickett/agentic/.github/actions/post-comment@main
           with:
             comment-file: /tmp/comment.txt
             issue-or-pr-number: ${{ github.event.issue.number }}
             comment-type: issue
             repository: ${{ github.repository }}
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}
   ```

   `gh-start-work.yml` — Implements an issue and opens a PR:
   ```yaml
   on:
     issue_comment:
       types: [created]

   jobs:
     pr-start:
       if: "!github.event.issue.pull_request && startsWith(github.event.comment.body, '@claude /pr-start')"
       runs-on: ubuntu-latest
       timeout-minutes: 15
       env:
         VITE_PROXY_TARGET: ${{ secrets.VITE_PROXY_TARGET }}
         VITE_HOST: localhost
       steps:
         - uses: actions/checkout@v4
           with:
             fetch-depth: 0

         - uses: chriswickett/agentic/.github/actions/setup-claude-agent@main

         - uses: chriswickett/agentic/.github/actions/run-claude-skill@main
           with:
             skill-name: pr-start
             context-type: issue
             context-number: ${{ github.event.issue.number }}
             repository: ${{ github.repository }}
             claude-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}

         - uses: chriswickett/agentic/.github/actions/git-commit-push@main
           with:
             branch-name-file: /tmp/pr_branch_name.txt
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}
             bot-name: ${{ vars.AGENTIC_BOT_NAME }}
             bot-email: ${{ vars.AGENTIC_BOT_EMAIL }}

         - uses: chriswickett/agentic/.github/actions/create-pr@main
           with:
             issue-number: ${{ github.event.issue.number }}
             repository: ${{ github.repository }}
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}
   ```

   `pr-changes-requested.yml` — Fixes code when reviewer requests changes:
   ```yaml
   on:
     pull_request_review:
       types: [submitted]

   jobs:
     pr-fix:
       if: github.event.review.state == 'changes_requested' && startsWith(github.event.review.body, '@claude')
       runs-on: ubuntu-latest
       timeout-minutes: 10
       env:
         VITE_PROXY_TARGET: ${{ secrets.VITE_PROXY_TARGET }}
         VITE_HOST: localhost
       steps:
         - uses: actions/checkout@v4
           with:
             ref: ${{ github.event.pull_request.head.ref }}
             fetch-depth: 0

         - uses: chriswickett/agentic/.github/actions/setup-claude-agent@main

         - uses: chriswickett/agentic/.github/actions/run-claude-skill@main
           with:
             skill-name: pr-fix
             context-type: pr
             context-number: ${{ github.event.pull_request.number }}
             repository: ${{ github.repository }}
             claude-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}

         - uses: chriswickett/agentic/.github/actions/git-commit-push@main
           with:
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}
             bot-name: ${{ vars.AGENTIC_BOT_NAME }}
             bot-email: ${{ vars.AGENTIC_BOT_EMAIL }}

         - uses: chriswickett/agentic/.github/actions/post-comment@main
           with:
             issue-or-pr-number: ${{ github.event.pull_request.number }}
             comment-type: pr
             repository: ${{ github.repository }}
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}
   ```

   `pr-approved.yml` — Squash merges when reviewer approves:
   ```yaml
   on:
     pull_request_review:
       types: [submitted]

   jobs:
     pr-merge:
       if: github.event.review.state == 'approved' && startsWith(github.event.review.body, '@claude')
       runs-on: ubuntu-latest
       timeout-minutes: 5
       env:
         VITE_PROXY_TARGET: ${{ secrets.VITE_PROXY_TARGET }}
         VITE_HOST: localhost
       steps:
         - uses: actions/checkout@v4
           with:
             ref: ${{ github.event.pull_request.head.ref }}
             fetch-depth: 0

         - uses: chriswickett/agentic/.github/actions/setup-claude-agent@main

         - uses: chriswickett/agentic/.github/actions/run-claude-skill@main
           with:
             skill-name: pr-merge
             context-type: pr
             context-number: ${{ github.event.pull_request.number }}
             repository: ${{ github.repository }}
             claude-token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}

         - uses: chriswickett/agentic/.github/actions/squash-merge@main
           with:
             pr-number: ${{ github.event.pull_request.number }}
             repository: ${{ github.repository }}
             github-token: ${{ secrets.AGENTIC_BOT_TOKEN }}
   ```

4. **Add secrets**: At repo level (Settings → Secrets and variables → Actions) or org level:
   - `CLAUDE_CODE_OAUTH_TOKEN`: OAuth token from `claude setup-token` (uses your Claude Pro/Max subscription)
   - `AGENTIC_BOT_TOKEN`: The bot account's PAT
   - `VITE_PROXY_TARGET` (optional): For frontend repos that need a proxy to a backend

5. **Add variables**: At repo level (Settings → Secrets and variables → Actions → Variables) or org level:
   - `AGENTIC_BOT_NAME`: The bot's GitHub username
   - `AGENTIC_BOT_EMAIL`: The bot's noreply email

## TODO

- Harden and expand the allowed commands list in `bin/bg`
- Setup script to generate client repo workflow files from agentic

