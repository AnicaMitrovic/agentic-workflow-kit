# Agentic Workflow Kit

A setup prompt and a developer guide for building software with Claude Code through a
fixed, repeatable process: AI writes the code, and quality comes from iterating it through
standard prompts with four human checkpoints.

## What is in the kit

| File | For | What it does |
|---|---|---|
| [`SETUP_PROMPT.md`](SETUP_PROMPT.md) | Claude | Sets up the workflow in any repository, phase by phase, with your approval between phases |
| [`DEVELOPER_GUIDE.md`](DEVELOPER_GUIDE.md) | You | How to run the loop: which skill to call when, how to iterate, when to stop |

The setup adds these files to a project:

```
CLAUDE.md                 # the map, loaded every session (rules and pointers, ~100 lines)
AGENTS.md                 # points Codex and other agents at CLAUDE.md
docs/usage-context.md     # what only you know: users, data sizes, priorities
docs/architecture.md      # how the system is built
.claude/settings.json     # hooks: format on edit, no finishing while verify is red
.claude/hooks/            # the two hook scripts (Node, cross-platform)
.claude/skills/           # task-brief, plan-feature, implement, iterate, review-diff
.claude/agents/           # review-crashes, review-cleanup, review-performance
.planning/<feature>/      # task, plan, review rounds, findings log, summary
<verify command>          # one quality gate: format, lint, types, tests, build
```

## Quick start

1. Clone this repo somewhere permanent.
2. Open a terminal in the project you want to set up and start `claude`.
3. Type:

   > Read `<path-to-kit>/SETUP_PROMPT.md` and follow it for this project. Also copy
   > `<path-to-kit>/DEVELOPER_GUIDE.md` into `docs/agentic-workflow.md` in Phase 5.

4. Approve each phase. Fill in the `TODO (developer)` lines in `docs/usage-context.md`.
5. In a fresh session, accept the workspace-trust prompt and check that `/hooks` lists the
   two project hooks and `/` lists the five skills.

## The loop in one screen

Every AI step runs in a fresh session: `/clear` first, then the command.

```
1    /task-brief <slug>                        you write task.md
2    /plan-feature .planning/<f>/task.md
3                                              you review the plan
4–5  /implement .planning/<f>/plan.md
6–7                                            you test and review structure
8    /review-diff .planning/<f>                + the same prompt in Codex
9                                              you decide each finding in findings.md
     /iterate .planning/<f>/findings.md        back to 8 until only minor findings
     /review-diff .planning/<f> ship           last round: blockers only
10–11                                          you read the full diff, commit, merge
```

One step per session; steps hand over through files in `.planning/`, never through chat
history. Details in [`DEVELOPER_GUIDE.md`](DEVELOPER_GUIDE.md).

## Requirements

- [Claude Code](https://code.claude.com/docs). Optional: Codex, for second-model reviews.
- Git. Node.js for the hook scripts (any project stack; on Windows also Git Bash).

## Status

The hook scripts are tested on Windows with Git Bash and Node 24. The skills and reviewer
agents are a starting point: tune them from what your reviews show, and commit each prompt
change with the reason.
