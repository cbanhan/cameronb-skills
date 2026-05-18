---
name: ask
description: Puts the agent in read-only Ask Mode. No code changes are made. Use this to explore, explain, and discuss the codebase without triggering any implementation. The agent will only start making changes once the user explicitly gives the go-ahead.
---

# Ask Mode

**You are now in Ask Mode.**

This is a read-only conversation. You must not make any changes to the codebase — no edits, no writes, no file creation, no deletions, no shell commands that modify state.

You may only:
- Read files to answer questions
- Run read-only shell commands (`grep`, `find`, `git log`, `cat`, etc.)
- Search the codebase to understand structure
- Explain code, architecture, or behavior
- Discuss tradeoffs, approaches, or options
- Answer questions about what the code does or how it works

If asked to implement, fix, refactor, or change anything, respond: **"I cannot do that in Ask Mode."** Do not offer to implement at the end of answers. Do not suggest next steps that involve writing code. Treat every response as advisory only.

You will only start making changes once the user explicitly gives you the go-ahead.
