# Agentic Workflow Setup — prompt for any project

**How to use (for the developer):** open a terminal in the project root, start `claude`,
and type:

> Read `<path-to-this-file>/SETUP_PROMPT.md` and follow it for this project. Also copy
> `<path-to-this-folder>/DEVELOPER_GUIDE.md` into `docs/agentic-workflow.md` in Phase 5.

Everything below is addressed to Claude.

---

## Your job

Set up a repeatable agentic engineering workflow in this repository: context files, one
quality-gate command, two Claude Code hooks, five skills, three reviewer agents and a
`.planning/` folder. The method behind it:

1. **Context window management** — the model only knows what is in its context, so the
   developer decides what goes in (pointed files + `docs/usage-context.md`).
2. **Iterate all generated code** — every output is a first draft; improvement passes run
   in fresh sessions.
3. **A repeatable process with four human checkpoints** — plan, code structure, review
   findings, final code. Never skipped, even for small features.
4. **Standardized prompts** — one fixed prompt per step, stored as a skill and versioned
   in the repo. Reviews run three narrow reviewer agents in parallel, repeated until only
   minor findings remain, then one "ship mode" round.

The setup must work for **this** project's stack. Templates below are stack-neutral;
adapt commands and reviewer focus lists to what you find, and say what you adapted.

## Ground rules (follow throughout)

- **Phase by phase.** Do one phase, show the result, wait for approval before the next.
  One branch per phase (`chore/workflow-phase-N-…`) from an up-to-date default branch;
  `git fetch` first, never trust a stale local view.
- **Show files before committing.** Commit only after the developer has seen them. Push
  and open a PR only when asked; merge only when asked.
- **Ask before adding dependencies** (formatter, test runner, linter). Propose the
  conventional choice for the stack, then wait.
- **Never overwrite existing setup.** If `CLAUDE.md`, `AGENTS.md`, `.claude/settings.json`,
  `.claude/agents/*` or `.claude/skills/*` exist, read them and merge. Existing project
  rules and agents stay; report any name collision instead of replacing.
- **Never invent facts.** Anything only the developer knows (users, data sizes, priorities)
  is written as `**TODO (developer):** …` with the assumption you will use until it is
  answered, labelled `Assumed:`.
- **Check current docs, not memory,** for anything about Claude Code's own config: hook
  schema and stdin fields, skill frontmatter (`$ARGUMENTS`, `disable-model-invocation`,
  `argument-hint`), subagent frontmatter. Use the claude-code-guide agent or the docs at
  https://code.claude.com/docs. Report what you verified.
- **Cross-platform.** Detect the OS. On Windows, hook commands run in Git Bash (PowerShell
  if Git Bash is missing); `npm`/`npx` are `.cmd` shims that need a shell to spawn. Write
  hook scripts in Node (`.mjs`) when Node is installed — even in non-JS projects — because
  they then behave identically everywhere; otherwise bash.
- **Secrets.** Before any commit, check that `.env*`, keys and local config are ignored.
  Never print secret values; compare by hash or length if you must.
- **Keep a status file.** Create `.planning/SETUP_STATUS.md` in Phase 0 and update it at
  the end of every phase (phase, date, branch/PR, open TODOs). A fresh session must be able
  to continue from it without chat history.

---

## Phase 0 — Discover and fix prerequisites

1. Inspect the repo and report a short table: language(s) and versions, package manager,
   build command, formatter, linter, test runner, CI, whether it is a git repo, remote,
   default branch, OS, and existing `CLAUDE.md`/`.claude/` content.
2. Every one of these blocks the workflow. For each missing item, propose the fix and wait:
   - **No git repo** → `git init`, check ignores (secrets, build output, dependencies),
     consider a `.gitattributes` with `* text=auto eol=lf` (Windows + formatter check),
     first commit. Ask whether to add a remote.
   - **No formatter** → the stack's default (gofmt, Prettier, `dotnet format`/CSharpier,
     Ruff, rustfmt…). Match the existing code style to minimise churn. Commit the config
     first, then the formatting pass as its **own** commit (`style: …`, no behaviour
     change). Optionally add `.git-blame-ignore-revs` with that hash — and note that it only
     survives "Create a merge commit" merges (squash/rebase change the hash).
   - **No linter, or warnings that do not fail** → make warnings fail.
   - **No test runner** → the stack's default (Go testing, Vitest/Jest + Testing Library,
     xUnit, pytest…). Add one small real test to prove the setup.
3. Create `.planning/SETUP_STATUS.md`.

## Phase 1 — Context files

### `CLAUDE.md` — the map, loaded every session (max ~100 lines)

Rules and pointers only; details live in `docs/` and are read on demand. Include:

- One paragraph: what the project is and who it is for.
- "Read before non-trivial work": `docs/usage-context.md`, `docs/architecture.md`.
- Stack, commands (run, test, **the verify command**), folder map.
- Pointers instead of copies: "Auth lives in `internal/auth`; read it before touching login."
- **Rules that must never be broken** (numbered, specific, derived from the code and
  existing docs — security boundaries, layering, data that must never be logged).
- Conventions the code actually follows (naming, error handling, test style, comment style).
  If the code has an unusual convention a cleanup pass might "fix" (e.g. long teaching
  comments), state that it is intentional.
- Hooks: one line saying edits are auto-formatted and Claude cannot finish while verify is red.
- Git workflow (remotes, branch/PR habit, merge method).

If a `CLAUDE.md` already exists and is longer than ~100 lines, do not shorten it without
asking; propose what could move to `docs/`.

### `docs/usage-context.md` — what only the developer knows

Every skill and reviewer reads it; it is what stops a reviewer from demanding pagination
for a 200-item list. Sections:

```markdown
# Usage context

## What this is
## Who uses it            (roles, how many, how often, concurrently?)
## Data sizes             (rows, items per list, request sizes — real numbers)
## What must be right     (the things where a bug really hurts)
## What must be fast
## What can be simple
## Out of scope
```

Fill only what you can source (code, docs, README, git history) and cite where it came
from. Everything else: `**TODO (developer):** …` plus `Assumed: …`.

### `docs/architecture.md` — for humans and models

How the system is built: components and what calls what (an ASCII diagram is fine),
request/data flow, external contracts (APIs, schemas) with pointers to their source of
truth, error taxonomy, layering rules, how tests are organised. Only facts from the code.

### `AGENTS.md` — for Codex and other agents

```markdown
# AGENTS.md

Read [CLAUDE.md](CLAUDE.md) and follow it. It is the single source of truth for this
repo's rules, commands and conventions, for every coding agent.
```

## Phase 2 — Quality gate and hooks

### The verify command

One command that runs formatter check, linter, type check, tests and build, in that
order, and **fails on the first error or warning**. Expose it the way the stack expects:

| Stack | Typical gate |
|---|---|
| Node/TS | `npm run verify` = format:check && lint && typecheck && test && build |
| Go | `scripts/verify.sh`: `test -z "$(gofmt -l .)"`, `go vet ./...`, `golangci-lint run`, `go test -race ./...`, `go build ./...` |
| .NET | `dotnet format --verify-no-changes`, `dotnet build -warnaserror`, `dotnet test` |
| Python | `ruff format --check`, `ruff check`, `mypy`/`pyright`, `pytest` |

Monorepo: one gate that runs every part. Prove it fails: add a deliberate type error,
run it, confirm non-zero exit, remove the error.

### Two hooks in `.claude/settings.json` (committed)

Verify the schema in the current docs first. As of this writing: events `PostToolUse`
(matcher `"Edit|Write"`, stdin has `tool_input.file_path`) and `Stop` (block by printing
`{"decision":"block","reason":"…"}` on stdout with exit 0; Claude Code itself overrides a
Stop hook after 8 consecutive blocks). Project hooks only run after the developer accepts
the workspace-trust prompt.

```json
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/format-file.mjs\"",
            "timeout": 30
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "node \"$CLAUDE_PROJECT_DIR/.claude/hooks/verify-on-stop.mjs\"",
            "timeout": 300
          }
        ]
      }
    ]
  }
}
```

Requirements for the hook scripts:

- **format-file** formats only the edited file, only inside the project, respects the
  formatter's ignore files, and **never blocks** (a syntax error is left for verify).
- **verify-on-stop** runs the verify command; on red it blocks with the last ~80 lines of
  output as the reason plus "fix the cause, do not weaken the check". It **skips** when the
  working tree is unchanged since the last green run (question-only turns stay fast). It
  releases after **3** blocks in a row so Claude can explain instead of looping. Keep its
  state in the OS temp dir, not `.git` (in a worktree `.git` is a file).

Reference implementations (tested on Windows + Git Bash, Node 24). Adapt
`FORMATTERS` and `VERIFY_COMMAND` to the stack:

`.claude/hooks/format-file.mjs`

```js
// PostToolUse hook: formats the file Claude just edited or wrote, so formatting
// never shows up as a review finding. Never blocks Claude: if the formatter fails
// (e.g. a syntax error), the Stop hook's verify run reports it later.

import { spawnSync } from 'node:child_process'
import { readFileSync } from 'node:fs'
import path from 'node:path'

// Extension → formatter command; the file path is appended. Adapt to the stack.
// Commands go through a shell so Windows .cmd shims (npx) work.
const FORMATTERS = {
  '.go': 'gofmt -w',
  '.ts': 'npx --no-install prettier --write --ignore-unknown',
  '.tsx': 'npx --no-install prettier --write --ignore-unknown',
  '.js': 'npx --no-install prettier --write --ignore-unknown',
  '.mjs': 'npx --no-install prettier --write --ignore-unknown',
  '.jsx': 'npx --no-install prettier --write --ignore-unknown',
  '.json': 'npx --no-install prettier --write --ignore-unknown',
  '.css': 'npx --no-install prettier --write --ignore-unknown',
  '.md': 'npx --no-install prettier --write --ignore-unknown',
  // '.py': 'ruff format',
  // '.cs': 'dotnet csharpier',
}

const projectDir = process.env.CLAUDE_PROJECT_DIR ?? process.cwd()
const input = JSON.parse(readFileSync(0, 'utf8'))
const filePath = input.tool_input?.file_path
if (!filePath) process.exit(0)

const absolute = path.resolve(projectDir, filePath)
const relative = path.relative(projectDir, absolute)
if (relative.startsWith('..') || path.isAbsolute(relative)) process.exit(0) // outside project

const command = FORMATTERS[path.extname(absolute).toLowerCase()]
if (!command) process.exit(0)

// Relative path, so the formatter applies the project's ignore files and config.
spawnSync(`${command} "${relative}"`, { cwd: projectDir, shell: true, stdio: 'ignore' })
```

`.claude/hooks/verify-on-stop.mjs`

```js
// Stop hook: runs the verify command when Claude tries to finish, and blocks
// finishing while it is red. Claude gets the failing output and keeps working.
//
// - Skips verify when the working tree has not changed since the last green run,
//   so question-only turns do not pay for a full build.
// - Loop guard: after MAX_BLOCKS refusals in a row, lets Claude stop (with the
//   failure in its context) instead of looping forever on something it cannot fix.

import { spawnSync } from 'node:child_process'
import { createHash } from 'node:crypto'
import { existsSync, mkdirSync, readFileSync, writeFileSync } from 'node:fs'
import { tmpdir } from 'node:os'
import path from 'node:path'

const VERIFY_COMMAND = 'npm run verify' // e.g. 'bash scripts/verify.sh'
const MAX_BLOCKS = 3
const OUTPUT_TAIL_LINES = 80

const projectDir = process.env.CLAUDE_PROJECT_DIR ?? process.cwd()
const input = JSON.parse(readFileSync(0, 'utf8'))

// Per-session state lives in the OS temp dir (not .git: in a worktree .git is a file).
const stateDir = path.join(
  tmpdir(),
  'claude-hooks',
  createHash('sha256').update(projectDir).digest('hex').slice(0, 12),
)
mkdirSync(stateDir, { recursive: true })
const stateFile = path.join(stateDir, `stop-${input.session_id ?? 'default'}.json`)
const state = existsSync(stateFile)
  ? JSON.parse(readFileSync(stateFile, 'utf8'))
  : { blocks: 0, greenFingerprint: null }
const saveState = () => writeFileSync(stateFile, JSON.stringify(state))

function git(args) {
  return spawnSync('git', args, { cwd: projectDir, encoding: 'utf8' }).stdout ?? ''
}

// Identifies the current working tree: tracked changes plus untracked files' contents.
function fingerprint() {
  const hash = createHash('sha256')
  hash.update(git(['diff', 'HEAD', '--no-ext-diff', '--binary']))
  for (const file of git(['ls-files', '--others', '--exclude-standard', '-z']).split('\0')) {
    if (!file) continue
    hash.update(file)
    try {
      hash.update(readFileSync(path.join(projectDir, file)))
    } catch {
      // Deleted between listing and reading: the name alone still marks the change.
    }
  }
  return hash.digest('hex')
}

const current = fingerprint()
if (current === state.greenFingerprint) {
  state.blocks = 0
  saveState()
  process.exit(0)
}

const result = spawnSync(VERIFY_COMMAND, { cwd: projectDir, encoding: 'utf8', shell: true })

if (result.status === 0) {
  state.blocks = 0
  state.greenFingerprint = current
  saveState()
  process.exit(0)
}

if (state.blocks >= MAX_BLOCKS) {
  state.blocks = 0
  saveState()
  process.exit(0)
}

state.blocks += 1
saveState()

const output = `${result.stdout ?? ''}${result.stderr ?? ''}`
const tail = output.trimEnd().split(/\r?\n/).slice(-OUTPUT_TAIL_LINES).join('\n')
const reason = [
  `${VERIFY_COMMAND} failed (attempt ${state.blocks} of ${MAX_BLOCKS}). Work is not done until it passes.`,
  'Fix the cause (do not weaken the check, skip tests or disable lint rules), then finish again.',
  'If you cannot fix it, stop and explain why to the user.',
  '',
  tail,
].join('\n')

console.log(JSON.stringify({ decision: 'block', reason }))
```

**Test both hooks by piping JSON into them** (no live session needed):

- format: a messy file gets formatted; a file with a syntax error and an ignored file are
  left alone; missing `file_path` exits 0.
- stop: green tree → exit 0, no output; run again unchanged → skipped instantly; add a type
  error → `decision: block` with the real error, 3 times, released on the 4th; remove it.

Then add the hooks line to `CLAUDE.md`. Tell the developer to do the live check in a fresh
session: accept workspace trust, run `/hooks` (the count includes hooks from enabled
plugins — that is normal), ask Claude to add a type error and finish.

## Phase 3 — Skills and reviewer agents

Check in the current docs that `$ARGUMENTS` works in `SKILL.md` and which frontmatter
fields exist. Use `disable-model-invocation: true` (if supported) so the workflow skills run
only when the developer calls them. Name the review skill **`review-diff`** — `/review` may
be built in, and `code-review`/`security-review` skills may exist. Check for collisions with
existing skills and agents.

Replace `<VERIFY>` below with this project's verify command. Keep the prompt bodies as
written; they are meant to be tuned later from what the reviews show, with the change and
the reason committed.

### `.claude/skills/task-brief/SKILL.md` — step 1

```markdown
---
name: task-brief
description: Start a feature by creating .planning/<feature>/task.md from the standard task template.
argument-hint: <feature-slug>
disable-model-invocation: true
---
Create `.planning/$ARGUMENTS/task.md` from the template below. If the folder already
exists, stop and say so.

Write the template exactly. Do not invent the goal, acceptance criteria or usage context:
those are the developer's. You may add up to five suggested files under "Files to read
first", each marked `(suggested)`, if you are confident they matter.

Run `date` and add its output as the first line: `Started: <output>`.
Then stop and tell the developer to fill it in (10–15 minutes) before running
/plan-feature .planning/$ARGUMENTS/task.md in a fresh session.

Template:

# Task: <feature name>

## Goal
What the user can do after this ships, in one or two sentences.

## Acceptance criteria
- [ ] Concrete, testable statements. "Returns 409 when X", not "handles errors".

## Context the AI cannot know
- Who uses this and how often, realistic data sizes, what must be fast.

## Files to read first
- path/to/file — why it matters

## Out of scope
- What not to touch.
```

### `.claude/skills/plan-feature/SKILL.md` — step 2

```markdown
---
name: plan-feature
description: Create an implementation plan from a task brief. No code changes.
argument-hint: <path to task.md>
disable-model-invocation: true
---
Read CLAUDE.md, docs/usage-context.md, docs/architecture.md and the task brief at $ARGUMENTS.
Read every file listed under "Files to read first" before planning.

Write a plan to the same folder as plan.md containing:
1. Files to create or change, with one line each on why.
2. The order of changes.
3. How each acceptance criterion will be tested.
4. Risks, edge cases, and anything in the brief you find unclear.

Do not write any code. Stop after writing plan.md.
```

### `.claude/skills/implement/SKILL.md` — steps 4–5

```markdown
---
name: implement
description: Implement an approved plan, with tests.
argument-hint: <path to plan.md>
disable-model-invocation: true
---
Read CLAUDE.md, docs/usage-context.md and the approved plan at $ARGUMENTS, and the task.md
next to it. Read the files the plan names before changing them.

Implement the plan step by step. Write tests for every acceptance criterion.
Run <VERIFY> and fix everything until it passes.
If the plan turns out to be wrong, stop and explain instead of improvising.
Do not commit.
```

### `.claude/skills/iterate/SKILL.md` — improvement passes

```markdown
---
name: iterate
description: One improvement pass over the current uncommitted changes, or a fix of the accepted review findings.
argument-hint: "[path to findings.md]"
disable-model-invocation: true
---
Read CLAUDE.md and docs/usage-context.md.

If $ARGUMENTS names a findings.md: fix exactly the findings of the latest round whose
Decision is "fix", nothing else. Rejected findings stay as they are.

Otherwise: analyze the current changes with git diff.
The changes are `git diff HEAD` (staged and unstaged) plus every file listed by
`git ls-files --others --exclude-standard`. Read the new files in full.
Check for duplicated code, code that duplicates something elsewhere in the repo,
inconsistency with existing patterns, and untested branches.
Fix what you find.

In both cases run <VERIFY> until green. Do not commit.
List what you changed and why (finding IDs where they apply).
```

### `.claude/skills/review-diff/SKILL.md` — step 8, the superprompt

```markdown
---
name: review-diff
description: Standard code review of uncommitted changes with three parallel reviewer agents. Analysis only.
argument-hint: <.planning/feature folder> [ship]
disable-model-invocation: true
---
Arguments: $ARGUMENTS — the feature's .planning folder, optionally followed by "ship".

Please do an analysis of the code we wrote. Use git diff.
Files are not yet committed, I will do this soon.

The changes are `git diff HEAD` (staged and unstaged) plus every file listed by
`git ls-files --others --exclude-standard`. Read the new files in full.

I want to make sure we do not have any critical showstoppers that cause
crashes or unexpected behavior.
Also check if we have duplicated code and if there is any code that really
should be cleaned up.
Also check if there are potential performance issues with the code that
really should be optimized.

Everything works as expected as far as I have tested.
Important! Do not change code, only analyze.

Run the three reviewer agents in parallel: review-crashes, review-cleanup,
review-performance. Give each the feature folder so it can read task.md, plan.md and
findings.md. Merge their results and remove duplicates. Do not re-report a finding that
findings.md already rejects, unless you have new evidence — then say so.

If the arguments include "ship": Important: I want to ship this now. Only report things we
must fix, and propose simple solutions.

Output every finding as:
- ID (C-n crashes, Q-n cleanup, P-n performance), severity (blocker / should-fix / minor), file:line
- What is wrong and how it would show up for a user
- Suggested fix in one or two sentences

End with counts per severity. Save the whole output as the next
review-round-N.md in the feature folder (N = highest existing + 1), and append one row
per finding to the table in findings.md with the Decision column empty (create
findings.md from the template in .planning/README.md if it does not exist).
```

### Reviewer agents — `.claude/agents/`

Each agent: frontmatter `name`, `description` (when to use), `tools: Read, Grep, Glob, Bash`
(Bash only for `git diff`/`git log`; the body must say "analysis only, never edit files").
Body: read `CLAUDE.md`, `docs/usage-context.md`, the feature's `task.md`, `plan.md`,
`findings.md`; get the changes as `git diff HEAD` plus every file from
`git ls-files --others --exclude-standard` (read new files in full); report findings in the review-diff
format with its ID prefix; say "no findings" rather than inventing minor ones.

Focus lists — keep the generic items, add the stack-specific ones you find relevant:

- **`review-crashes.md`** — null/undefined/nil handling, error paths and swallowed errors,
  concurrency and races (goroutines, async/await, stale UI state, double submit), input
  validation, security (injection, XSS, secrets in code or logs, auth checks), resource
  leaks, behaviour that differs from the acceptance criteria in `task.md`.
- **`review-cleanup.md`** — duplication within the diff and against the rest of the repo
  (search for it), naming, dead code, inconsistency with existing patterns, deviations from
  `CLAUDE.md` rules, missing or weak tests for acceptance criteria. Respect conventions
  `CLAUDE.md` calls intentional.
- **`review-performance.md`** — queries or requests in loops, missing indexes, N+1,
  unnecessary allocations or re-renders, work on every request/render that could be done
  once, payload/bundle size. **Judge against `docs/usage-context.md`, not imaginary
  scale;** a finding that only matters at a scale the usage context rules out is not a
  finding.

Verification: subagents and skills load at session start, so the developer checks them in a
**fresh** session — type `/` to see the skills; ask "which project subagents can you use?"
(the `/agents` wizard has been removed); or run one: "use review-cleanup on the current diff".

## Phase 4 — `.planning/` and dry run

`.planning/README.md`:

```markdown
# .planning

One folder per feature, committed with the feature. Steps hand over through these files,
never through chat history.

.planning/<feature>/
  task.md            step 1  — written by the developer (/task-brief creates the skeleton)
  plan.md            step 2  — /plan-feature
  review-round-N.md  step 8  — /review-diff, one per round (-codex suffix for Codex rounds)
  findings.md        step 9  — the findings log, decisions by the developer
  summary.md         step 11 — metrics and retro
```

`findings.md` template (created by `/review-diff` on its first round if missing):

```markdown
# Findings log — <feature>

| Round | ID | Severity | Decision | Reason |
| --- | --- | --- | --- | --- |
```

`summary.md` template:

```markdown
Feature: <name>
Human time: … | AI time: … | Total: …
Build loops: … | Review rounds: … (+1 ship mode)
Findings per round: … | Accepted: … | Rejected: …
Claude vs Codex: …
Lines written by hand: …
What went wrong: <one honest paragraph>
Prompt changes after this feature: <what and why>
```

**Dry run** (fresh session, by the developer, with you guiding): a trivial change; confirm
the format hook formats it, a deliberate type error blocks finishing, all five skills and
three agents load.

## Phase 5 — Hand-off

- Copy the developer guide to `docs/agentic-workflow.md` (if its path was given) and link
  it from `CLAUDE.md` in one line.
- Update `.planning/SETUP_STATUS.md`: all phases, PRs, open TODOs (usage-context answers,
  anything not verified live).
- Recommend a first feature: small (half a day by hand), inside this repo only, ideally one
  that fixes something the setup uncovered.

## Gotchas learned the hard way

- A file the plan "refers to" but nobody saved does not exist for the next session. Put
  every template in the repo (they live in the skills) and keep `SETUP_STATUS.md` current.
- The developer may say "merged" before it is; check the PR state with `gh` before
  branching from main.
- Hooks count in `/hooks` includes plugin hooks; only the two project ones are yours.
- `localhost` vs `127.0.0.1`, CRLF vs LF, `.cmd` shims, `$CLAUDE_PROJECT_DIR` quoting —
  test on the developer's actual OS.
- A setup test that silences output (e.g. a spied `console.log`) can hide a real bug —
  report anything suspicious you see in passing as a candidate first feature, do not fix it
  during setup.
