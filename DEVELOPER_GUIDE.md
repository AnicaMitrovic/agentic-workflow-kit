# Agentic Workflow — Developer Guide

How to build a feature with Claude Code in a repo set up with `SETUP_PROMPT.md`: which
skill to call at each step, what you do yourself, how to iterate, and when to stop.

The core idea: **AI writes the code; quality comes from running it through a fixed
process with human checkpoints.** You stop improvising prompts and start improving the
process.

## The four rules

1. **You manage the context.** The model only knows what is in its context. Point it at
   files in the task brief and keep `docs/usage-context.md` current — that is how it learns
   what only you know (real users, real data sizes, what must be fast).
2. **Every output is a first draft.** One pass cannot build, deduplicate, optimise and find
   edge cases at once. Plan for several improvement passes.
3. **Four human checkpoints, never skipped:** the plan, the code structure, the review
   findings, the final diff. Even for small features.
4. **The same prompts every time.** Each step is a skill in `.claude/skills/`. When a prompt
   is wrong, change the skill file and commit the reason — not the prompt of the day.

And two habits that hold it together:

- **Fresh session for every step** (`/clear` or a new terminal). A fresh session explores the
  repo differently and finds things the last one missed. Steps talk through files in
  `.planning/<feature>/`, never through chat history.
- **Ship mode at the end.** When reviews only find minor things, one last round reports
  blockers only. That is your clear stopping point.

## What is in the repo

| Path | What it is | Who changes it |
|---|---|---|
| `CLAUDE.md` | The map, loaded every session: rules, commands, pointers | You, rarely; keep it under ~100 lines |
| `AGENTS.md` | Tells Codex to follow `CLAUDE.md` | Nobody |
| `docs/usage-context.md` | What only you know: users, sizes, priorities, out of scope | **You** — fill its TODOs first |
| `docs/architecture.md` | How the system is built | Updated with the code |
| `.claude/settings.json` + `.claude/hooks/` | Format on edit; no finishing while verify is red | Rarely |
| `.claude/skills/` | `task-brief`, `plan-feature`, `implement`, `iterate`, `review-diff` | You, when tuning prompts |
| `.claude/agents/` | `review-crashes`, `review-cleanup`, `review-performance` | You, when tuning prompts |
| `.planning/<feature>/` | task, plan, review rounds, findings log, summary | Per feature |
| verify command (`npm run verify`, `scripts/verify.sh`, …) | The quality gate: format, lint, types, tests, build | Rarely |

## Once per machine / repo

- Install Claude Code; optionally Codex for second-model reviews. On Windows, Git Bash
  must be installed (hooks run in it).
- First session in the repo: **accept the workspace-trust prompt**, or the project hooks do
  not run. `/hooks` should list the two project hooks (format, verify). A higher total is
  normal — enabled plugins add their own.
- Skills and agents load at session start. After adding or changing them, start a fresh
  session. Type `/` to see skills; to check agents, ask "which project subagents can you
  use?" (the `/agents` wizard no longer exists).

## The 11-step loop

```
 1 Task brief (you) ─► 2 Plan (AI) ─► 3 Review plan (you) ─► 4–5 Implement + tests (AI)
        ▲                     ▲                                        │
        │                     └──── plan wrong: edit task.md, re-plan ◄┘
        │
 6 Test it yourself ─► 7 Review structure (you) ──bad──► back to 4 (fresh session)
        │ok
 8 Review code (AI, 3 agents) ─► 9 Decide findings (you) ──fixes──► /iterate ─► back to 8
        │ only minor left → 1 round in ship mode
10 Read the full diff (you) ─► 11 Commit, PR, merge, deploy (you)
```

| # | Who | How | Output | Done when |
|---|---|---|---|---|
| 1 | You | `/task-brief <feature-slug>`, then fill it in (10–15 min) | `.planning/<f>/task.md` | Every acceptance criterion is testable |
| 2 | AI | Fresh session: `/plan-feature .planning/<f>/task.md` | `plan.md` | — |
| 3 | You | Read the plan against the criteria and your codebase knowledge | Approved plan | You would build it this way. Commit task + plan |
| 4–5 | AI | Fresh session: `/implement .planning/<f>/plan.md` | Code + tests | Verify hook lets it finish (green) |
| 6 | You | Run the feature like a user, including the brief's edge cases | Notes | Works as a user expects |
| 7 | You | Does it fit the architecture, existing patterns, the right folders? | — | Structure is right |
| 8 | AI | Fresh session: `/review-diff .planning/<f>` | `review-round-N.md`, rows in `findings.md` | — |
| 9 | You | Fill **Decision** + **Reason** for each finding in `findings.md` | Decisions | Every row decided |
| 10 | You | Read the whole diff as if someone else wrote it | — | You would sign it |
| 11 | You | Commit, PR, merge, deploy; write `summary.md` | Shipped | — |

### Step by step

**1 — Task brief.** `/task-brief license-list-page` creates the skeleton; you write it.
The quality of everything after depends on this file:
- **Goal:** what the user can do after it ships, 1–2 sentences.
- **Acceptance criteria:** testable — "shows a 409 message next to the org field", not
  "handles errors". Each one becomes a test.
- **Context the AI cannot know:** who uses it, how often, real sizes, what must be fast.
- **Files to read first:** the 3–6 files that matter, each with why. Precise pointers
  beat letting the model search the whole repo.
- **Out of scope:** what not to touch.

**2 — Plan.** `/clear`, then `/plan-feature .planning/<f>/task.md`. It writes `plan.md`
and stops. No code.

**3 — Review the plan.** Check: every criterion has a test, the files make sense, the risks
section is honest, nothing unclear is left. If it is wrong, **fix `task.md`** (the cause)
and re-plan in a fresh session — do not argue with the plan in chat. Commit `task.md` and
the approved `plan.md`.

**4–5 — Implement.** `/clear`, then `/implement .planning/<f>/plan.md`. It writes code and
tests and cannot finish while verify is red. If it says the plan is wrong, go back to 3.

**6 — Test it yourself.** Run the app. Try the acceptance criteria and the edge cases. Write
down what is broken.

**7 — Review the structure.** Not line-by-line yet: right place, right layering, existing
patterns followed, nothing over-engineered. Only you catch a wrong design — AI review finds
code issues inside the design it was given.

If 6 or 7 fail: describe the problem (in `task.md` under a "Build loop 2" note, or in the
prompt) and go back to step 4 in a fresh session. Count these as **build loops**.

**8 — AI review.** `/clear`, then `/review-diff .planning/<f>`. Three agents (crashes,
cleanup, performance) review the diff in parallel; the result is saved as the next
`review-round-N.md` and appended to `findings.md`.

**Second model (recommended):** open Codex in the repo and paste the body of
`.claude/skills/review-diff/SKILL.md` (below the `---` header), with two edits: replace
`$ARGUMENTS` with the feature folder path, and add "You have no subagents: review from all
three angles (crashes, cleanup, performance) yourself. Save your output as
review-round-N-codex.md and add your findings to findings.md." Check that it did; otherwise
save its answer yourself. Different models miss different things.

**9 — Decide the findings.** For every row in `findings.md`:

| Round | ID | Severity | Decision | Reason |
| --- | --- | --- | --- | --- |
| 1 | C-3 | blocker | fix | null deref when token missing |
| 1 | P-1 | should-fix | reject | list is max ~200 items (usage-context) |

- **fix** or **reject**, always with a reason. Rejected findings stay in the log — the next
  round reads it, so the same wrong idea does not come back.
- A rejection "because of real usage" means `docs/usage-context.md` is missing a fact.
  Add it there.
- Then fix the accepted ones (next section) and return to step 8 in a fresh session.

**10 — Final read.** The full diff, once more, as a reviewer of someone else's code.

**11 — Publish.** Commit (code + `.planning/<f>/`), PR, merge, deploy through the normal
pipeline. Write `summary.md` (template in `.planning/README.md`).

## How to iterate

### Which tool for which situation

| Situation | Do this |
|---|---|
| Implementation done, before your own review | Optional: `/clear` → `/iterate` (general pass: duplication, patterns, untested branches). One or two passes. |
| Accepted findings in `findings.md` | `/clear` → `/iterate .planning/<f>/findings.md` — fixes exactly the rows marked **fix** in the latest round |
| One small, obvious fix | Ask for it directly in a fresh session, naming the finding ID |
| Your test or structure review failed (6–7) | Back to `/implement` with the problem described — a build loop |
| The plan itself was wrong | Fix `task.md` → `/plan-feature` again |
| Verify keeps failing and Claude gives up | The hook releases after 3 blocks so it can explain. Read why, fix the cause or the brief. Never weaken the check. |

### When to stop reviewing

1. Run review rounds (8 → 9 → fix → 8) until **a round finds only minor items**. Expect
   at least three rounds on a small feature; findings per round should fall
   (e.g. 11 → 6 → 3 → 1).
2. Then one **ship-mode** round: `/review-diff .planning/<f> ship`. It reports only what
   must be fixed and proposes simple solutions.
3. Fix those, run verify, go to step 10.

If findings do not fall, the problem is usually upstream: a vague brief, a missing
usage-context fact, or a structure problem you should have caught in step 7.

## Context habits

| Habit | In practice |
|---|---|
| One task per session | `/clear` between plan, implement, each iteration, each review round |
| Hand-pick the files | "Files to read first" in every brief |
| Give it what only you know | Keep `docs/usage-context.md` current; every skill reads it |
| Keep `CLAUDE.md` short | It is loaded every session; rules and pointers only |
| Hand off through files | task → plan → review rounds → findings. Nothing lives only in chat |
| Sub-agents for exploration and reviews | They read in their own context and return only results |
| Watch for drift | Answers contradict earlier decisions or ignore rules → context is full. Write state to a file, start fresh |

Repeated review rounds in fresh sessions are not repetition: each one loads different parts
of the repo — they are new samples.

## Starting a fresh session

`/clear` (same terminal is fine) and make **the skill call your first and only prompt**.
`CLAUDE.md` loads automatically and each skill names the files to read — no warm-up needed.
To start with it pre-typed: `claude "/plan-feature .planning/<f>/task.md"`.

When no skill fits, use the pattern **files to read → what to do → when to stop**:

```
# Targeted fix of one finding
Read CLAUDE.md and .planning/<f>/findings.md.
Fix finding C-3 only (round 2). Run the verify command until green. Do not commit.

# Build loop after your own test or structure review
/implement .planning/<f>/plan.md
The code exists already. Problem found in manual testing: <what you saw, how to reproduce>.
Fix that within the plan; if it means the plan is wrong, stop and explain.

# Multi-session work (setup, long tasks)
Read .planning/SETUP_STATUS.md, then do the next phase.
```

Avoid:
- "Continue where we left off" or "as we discussed": the session has no memory of it.
- Retelling history in the prompt. Write the state into `task.md`, `findings.md` or a
  status file and point to it.
- Role-play preambles ("you are a senior engineer…"). They belong in `CLAUDE.md` or the skill.
- Combining steps ("plan and implement"). One session, one step.

**Test before `/clear`:** could someone who never saw this chat do the next step from the
files alone? If not, write the missing piece into the right file first.

## Measure it

Per feature, `.planning/<f>/summary.md`:

```
Feature: <name>
Human time: 45 min | AI time: 1 h 50 min | Total: 2 h 35 min
Build loops: 2 | Review rounds: 4 (+1 ship mode)
Findings per round: 11 / 6 / 3 / 1 | Accepted: 14 | Rejected: 7
Claude vs Codex: Codex found 2 issues Claude missed (C-7, P-2)
Lines written by hand: 0
What went wrong: <one honest paragraph>
Prompt changes after this feature: <what and why>
```

Note the start time when you write the brief. A reference point: ~15 min writing the task,
~40 min AI planning and implementing, ~90 min over six review rounds — about two hours idea
to ship, with roughly 20 minutes of human work per hour.

## Tuning the process

After each feature, 15–30 minutes of retro:
- Which findings kept coming back? → sharpen the reviewer agent or add a `CLAUDE.md` rule.
- Which findings did you always reject? → add the missing fact to `docs/usage-context.md`.
- Where did a build loop start? → improve the task-brief or plan-feature prompt.

Edit the skill or agent file, commit with a message that says **why**
(`prompt: review-performance ignores lists under 500 items — usage-context`). The process
improves in git history, not per feature.

**Later, not day one:** two or three features in parallel, each in its own git worktree
with its own session. Master one agent first.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| Skill missing after `/` | Skills load at session start → new session. Check `.claude/skills/<name>/SKILL.md` exists and has frontmatter |
| Agent not used | Same: fresh session. Check `.claude/agents/<name>.md`. Ask "which project subagents can you use?" |
| Hooks do not run | Workspace trust not accepted, or the session started in another folder. `/hooks` to check |
| `/hooks` shows more than 2 | Plugin hooks — normal |
| Claude "finished" with red verify | It hit the 3-block release; read its explanation, fix the cause |
| Every stop takes long | Verify runs only when files changed since the last green run; a slow verify is a project problem (speed up tests/build) |
| Reviews keep suggesting the same rejected idea | Is the rejection reason in `findings.md`, and the fact in `usage-context.md`? |
| Answers contradict earlier decisions | Context drift → write state to `.planning/<f>/`, `/clear`, continue |

## Cheat sheet

Every AI step runs in a fresh session: `/clear` first, then the command.

```
1    /task-brief <slug>                        you fill task.md
2    /plan-feature .planning/<f>/task.md
3                                              you review the plan, commit task + plan
4–5  /implement .planning/<f>/plan.md
6–7                                            you test, you review structure → bad: back to 4
8    /review-diff .planning/<f>                + Codex with the same prompt
9                                              you decide each finding in findings.md
     /iterate .planning/<f>/findings.md        back to 8
     /review-diff .planning/<f> ship           only minor left: blockers only
10–11                                          you read the full diff, commit, PR, merge,
                                               write summary.md
```
