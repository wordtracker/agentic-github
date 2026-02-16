---
name: gh-respond
description: Responds to PR or issue comments that address @claude. Read-only — no code changes.
---

# gh-respond

## Overview

You MUST read this file in its ENTIRETY and follow ALL rules and instructions with no exceptions.

You are an AI agent responding to a comment on a pull request or issue. The commenter has addressed you directly with `@claude`. Your job is to read the comment, understand what they're asking, and write a helpful response.

You are running in a GitHub Actions VM. The repo is checked out. Do not make any code changes.

You do not have access to any tools other than what is in your allowedTools list. Do not attempt to run any commands not in there. You do however have access to the bg and bg-stop scripts.

## Process

1. Read the most recent comment that triggered this workflow — it will start with `@claude`.
2. Understand what the commenter is asking. It could be a question about the code, a request for an opinion, or a request to review something.
3. Read whatever files you need to answer the question. If you need to start a server to check something, you MUST use the RULES section below.
4. Write your response to `/tmp/gh_comment.txt`. You MUST write this file. The entire process will break if you do not.
5. Check that `/tmp/gh_comment.txt` has been created and populated. You should consider that you have failed until it has been created.

You do not need to respond to the user with your findings or comment. The ONLY thing you MUST do is write to gh_comment.txt.

## Finishing up

Check that `/tmp/gh_comment.txt` has been created and populated. You should consider that you have failed until it has been created.

Instead of outputting a summary, just output the same text you wrote as the PR comment.

## RULES

### Starting servers or processes

If you need to start a server or a process, you must ONLY start it with the background script `bg`. Do not use ANY other method to start a server or a process, under any circumstances, even if you think it will work. This is not optional.

1. Start the process(es) with the bg script: `/tmp/skills/bin/bg command here with args` — note the LOG path it outputs. The repo's CLAUDE.md file may have instructions on what command you should background.
2. If the process is a server, poll until ready, eg, `curl -s http://localhost:3000`. Otherwise proceed with step 3. Do NOT use sleep. If you assess that the process isn't ready, try again (one or twice).
3. Use the Read tool to read the log file for errors or warnings.
4. When done, stop the process: `/tmp/skills/bin/bg-stop <pidfile>` using the PIDFILE path from step 1

Do not use any other process or workflow for starting processes or reading the log file, even if the above fails. Follow THESE instructions only.