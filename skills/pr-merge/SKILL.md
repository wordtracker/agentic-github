---
name: pr-merge
description: Writes a clean squash-merge commit message for an approved PR. Called by the pr-merge job in GitHub Actions.
---

# pr-merge

## Context

You are running in a GitHub Actions VM. The repo is checked out at the PR branch. You have access to Read, Write, and Bash tools. You can run `pnpm test` to validate tests. Do not attempt to use git or gh. The workflow handles the actual merge after you finish.

PR metadata, review comments, and commit history are provided in your prompt.

## Process

### 1. Gather context

Review the commit history and PR comments (provided in your prompt) to understand the full journey — initial implementation and any rejection cycles. The linked issue (the original plan) is also included in your context.

### 2. CRITICAL: Test Validation

Before writing ANY files, you MUST run tests:

1. Run `pnpm test`
2. Check the exit code

**IF exit code is 0 (all tests passed):**
- Write comment.txt explaining that all tests passed and the PR will be merged
- Proceed to write commit_msg.txt
- Proceed with merge process

**IF exit code is NOT 0 (any test failed):**
- Write comment.txt explaining which tests failed and why, and explaining that you cannot merge if tests fail
- DO NOT write commit_msg.txt
- DO NOT proceed with merge
- Your task is complete - exit

**IMPORTANT:** You MUST ALWAYS write comment.txt in both cases. Writing commit_msg.txt when tests have failed is a critical error. The commit_msg.txt file MUST ONLY exist if `pnpm test` exits with code 0. If you are unsure whether tests passed, run them again. Never assume tests passed without checking the exit code.

Tests are not optional. Tests are not suggestions. All tests must pass before merge.

### 3. Write the commit message

Write a clean commit message describing what was built. Follow the conventions in the commit skill at `../commit/SKILL.md`.

The message should read as if the work was done right the first time. No mention of rejections, fix cycles, or iterations. Just the final result.

The first line of the commit body (after the subject and blank line) must be `issue: #<issue-number>`.

## Output

Write these files:

- `/tmp/comment.txt` — ALWAYS write this. Comment explaining test results and what will happen (merge or not).
- `/tmp/commit_msg.txt` — ONLY write this if tests passed. The clean squash-merge commit message.

The workflow will post the comment, then merge the PR if commit_msg.txt exists.
