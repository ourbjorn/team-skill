---
name: team
description: Use when running inside Rex and work should be split across several Claude Code agents working in parallel in visible panes, such as an orchestrator or lead with workers, an agent team, spawning agents, fanning out a feature into backend/frontend/test parts, or a parallel review. Not for quick lookups or research, where in-process subagents are cheaper.
---

# Team: orchestrate Claude Code agent teams in Rex

You are the **lead**. You split work, start workers, wait for them, check their
results and merge. Each worker is a real Claude Code session in its own Rex tab
and its own git worktree and branch. The user can watch any of them, type to
them, or approve their prompts.

Rex gives you the key signal for free: Claude Code reports its state
(`working`, `done`, `blocked` with a reason, `error`) to Rex. The `team`
script reads that state, so you never have to read screens to find out whether
a worker has finished.

All commands go through the bundled script, `scripts/team` in this skill's
base directory (shown as "Base directory for this skill" when the skill
loads). In the commands below, `$T` stands for:

```bash
bash <base directory of this skill>/scripts/team
```

Use the full path in every command, because shell variables don't carry over
between your Bash calls.

Run it from inside the repository. If a command reports missing requirements,
run `$T doctor` and pass its ✗ lines and install hints on to the user. Don't
install tools yourself unless the user asks. State lives in `<repo>/.team/<team>/`;
worktrees go in `<repo>/.worktrees/` (both are git-excluded automatically).

## Quick reference

| Command | Effect |
|---|---|
| `$T up TEAM --plan FILE` | Start a team from a plan file (see Team plans): every worker, its start point and its paths. Checks the whole plan first; on any problem nothing is created |
| `$T up TEAM [--base REF] SPEC...` | Start a team from a list. SPEC is `ROLE` or `ROLE@LABEL` (worker `ROLE-LABEL`). Branches `team/TEAM/<worker>` start from REF (default HEAD) |
| `$T add TEAM SPEC [--base REF] [PATH...]` | Add a worker to a running team, e.g. a later wave stacked on an earlier worker's branch |
| `$T own TEAM WORKER PATH...` | Paths the worker may change (dirs, files, globs). `$T own TEAM` lists all and shows overlaps |
| `$T check TEAM` | Files changed outside each worker's paths, and files changed by 2+ workers. Exit 1 on problems |
| `$T task TEAM ROLE [FILE]` | Set the worker's task (file, or stdin with a heredoc) |
| `$T assign TEAM ROLE...` | Tell workers to start their task |
| `$T wait TEAM [ROLE...] --for 1800` | Block until each is done, blocked or failed. Exit 0 = all done, 2 = some blocked, 3 = failed or gone, 124 = timed out |
| `$T status TEAM` | State, what it is waiting on, report present, commits ahead |
| `$T read TEAM ROLE` | The worker's `.team-report.md` (add `--screen 80` for raw terminal output) |
| `$T send TEAM ROLE "one line"` | Follow-up instruction. One line only: each newline submits a separate prompt |
| `$T stop TEAM ROLE` | Interrupt the worker (Esc) |
| `$T focus TEAM ROLE` | Show that worker to the user in the Rex app |
| `$T inbox` | Every Claude pane in Rex that is blocked, done or failed |
| `$T role ls` / `new` / `show` / `edit` / `rm` | Manage role files (see Creating roles) |
| `$T down TEAM` | Close the session, remove clean worktrees. Branches are kept for merging |

Built-in roles: `plan`, `be`, `fe`, `test`, `review`. Any other name gets a
generic worker. A repo can override or add roles in `.team/roles/<role>.md`.
A role file sets its worker's model with frontmatter (`model: sonnet` between
`---` lines). Built-in defaults: `plan`, `be` and `review` use opus; `fe` and
`test` use sonnet.

## The loop

1. **Decide whether a team is worth it.** Use one when there are 2 or more parts
   that can proceed independently, touch different files, and each take more
   than a few minutes. Otherwise do the work yourself or use a subagent. Start
   with 2–3 workers; more workers mean more merging and more prompts for the user.
2. **Work out the team from the task.** Read the code (or the diff) first. The
   workers follow from how the work divides: one per independent part, each
   owning its paths, with a role and model that fit the part.
3. **Propose the team before starting it.** Show the user the plan file and,
   for each worker, one line on what it does. Also show the waves, if any (see
   Team plans), and what stays with you. Start only after the user agrees,
   then apply their changes to the plan. Don't run `$T up` before that.
4. **Start the team:** `$T up TEAM --plan TEAM.plan`. If it reports problems,
   nothing was created: fix the plan and run it again. If the output says
   **ACTION NEEDED** (folder trust), tell the user which tabs need
   "Yes, I trust this folder". Wait for them to confirm before assigning.
5. **Write one task per worker** with `$T task`, using the recipe below. Its
   "You own" section lists the same paths as the plan.
6. **Assign:** `$T assign TEAM <workers>`. It refuses while declared paths
   overlap, and tells each worker which paths it may change.
7. **Wait:** `$T wait TEAM --for 1800`. Run it in the background when you have
   other work to do; you are notified when it exits.
8. **Handle the exit code:**
   - `0`: read each report with `$T read`.
   - `2` (blocked): show the user `$T status` and run `$T focus` for each
     blocked worker. The user answers the prompt; then wait again.
   - `3`: read the screen (`$T read TEAM ROLE --screen 80`) and decide: send a
     fix-up with `$T send`, or report the failure to the user.
   - `124`: show `$T status` and ask the user whether to keep waiting.
9. **Run `$T check TEAM`** before merging. A worker that changed files outside
   its paths, or a file changed by two workers, has to be resolved first. Ask
   the worker to revert, or merge that file by hand with the user.
10. **Verify by exit code yourself.** Merge the branches into a scratch branch
   or run the tests in a worktree. A worker's "tests pass" is a claim; an exit
   code of 0 that you observed is evidence.
11. **Merge** `team/TEAM/<worker>` branches as the user prefers, then `$T down TEAM`.

## Several workers with the same role

Split one role across workers when its work divides along paths that don't
intersect: modules, services, feature folders, independent endpoints.

```
be@orders   src/orders/  tests/orders/
be@billing  src/billing/ tests/billing/
fe          web/src/
```
(as a plan file; `$T own TEAM WORKER PATH...` changes paths later)

- **Split only if each part has its own directories.** If they share a file
  that both need to change (shared types, a router, a DI registration, a
  migration list), give that file to exactly one worker. Put what the others
  need in the Contract. Or do that shared change yourself before assigning.
- **Overlap checks are conservative.** A glob is compared by its directory part,
  so `src/**/*.a.ts` and `src/**/*.b.ts` count as overlapping. Use directories
  where you can; use `assign --force` only when you have checked the globs by hand.
- **Don't split two ways when the task is small.** Every extra worker adds a
  merge and more prompts for the user.

## Team plans

A plan file describes the whole team: one worker per line, written as
`SPEC [--base REF] [PATH...]`. Text after `#` is a comment. Paths can't contain
spaces, and quotes are optional. Save it as `<repo>/.team/<team>.plan`.

```
# csv-export.plan
be@api      src/api/export/ tests/api/export/      # endpoint + streaming
fe          web/src/export/                        # button + download flow
test        tests/e2e/export/                      # end-to-end coverage
```

`$T up csv-export --base main --plan .team/csv-export.plan` checks everything
first and lists all problems: bad names, duplicates, unknown refs, existing
branches, overlapping paths. If there is any problem, nothing is created.
`--base` (default HEAD) is where branches start; a line's own `--base`
overrides it.

**Waves, for work that depends on other work.** Workers start from a fixed
commit, so a worker never sees changes another worker makes later. When part B
needs part A:
1. Wave 1: start A (and anything else independent). Wait for it, then check
   its branch builds and its tests pass.
2. Wave 2: `$T add TEAM be@b --base team/TEAM/be-a src/b/` stacks B on A's
   branch. `check` and `status` then compare B against A's branch, so A's
   changes don't count as B's.
3. If A gets more commits later, B doesn't have them. Finish A before stacking.

Plan waves up front and put them in the proposal. Only wave 1 goes in the plan
file; later waves are `add` commands you run after the earlier wave is done.

**Splitting a finished branch into smaller branches** is a good fit:
- Base the team on the branch's base (`--base main`) and give each worker one
  slice: the paths it owns.
- Its task says to take those files from the big branch
  (`git checkout <big-branch> -- <paths>`), make the slice build and pass tests
  on its own, and commit. The task also asks for the slice's ticket text
  (title, description, acceptance criteria) in the report.
- Slices that need another slice's changes go in a later wave, based on that
  slice's branch.
- **Verify completeness:** merge every slice branch into a scratch branch and
  `git diff <big-branch>`. It should be empty, apart from changes the user
  agreed to leave out.
- A file that must be split by hunk doesn't fit file ownership. Give it to one
  slice, or split it yourself.

## Task file recipe

Each `.team-task.md` has these sections, in order:

```markdown
## Goal
One or two sentences: what this role delivers.
## You own
Files/directories this role may change.
## Do not touch
Files other roles own.
## Contract
Exact interfaces shared with other roles (endpoint, request/response shape,
types, events), copied verbatim into every task that depends on them.
## Done when
Checkable acceptance criteria, including the command(s) that must exit 0.
```

Workers cannot see each other's tasks or your conversation. Everything a
worker needs is in its task file, or in the repo at the base commit.

## Creating roles

When the user asks for a new role, or a team needs one that `$T role ls` doesn't
have:

1. **Look at the repo first:** stack, folder layout, and the real build, test
   and lint commands (package.json scripts, *.csproj, Makefile, CI config).
2. **Write the body** with these parts, in order:
   ```markdown
   # Role: <name>
   One paragraph: what this worker owns and delivers.
   ## You do
   Kinds of change, and where in this repo (real paths).
   ## You don't
   What is out of scope; for read-only roles: "Do not modify code."
   ## Done means
   The exact commands from this repo that must exit 0 before reporting.
   ```
   Leave out the task-file and report rules; the script adds those to every role.
3. **Choose the model:** `opus` for judgement-heavy roles (planning, review,
   security, tricky backend); `sonnet` for well-specified building and testing;
   `haiku` for mechanical work (docs, renames, formatting).
4. **Save it.** Repo scope is the default: `.team/roles/`, committable, shared
   with the user's team. Use `--user` only when the user wants the role in
   all their repos (saved in `~/.claude/team/roles/`). Never edit the
   built-in role files: updates replace them. `$T role edit <built-in>`
   copies one to the user's roles first.
   ```bash
   $T role new security --model opus - <<'EOF'
   # Role: security
   ...
   EOF
   ```
   To start from an existing role: `$T role new be-dotnet --from be -` with the
   adapted body, or without a body to copy it as-is.
5. **Show the user the file** (`$T role show <name>`) and offer to adjust it.

## Rules

- **Answering prompts is the user's job.** Never answer a worker's permission,
  trust or login prompt with `send` or keystrokes; the script refuses to type
  into a blocked worker for this reason. Report it and use `$T focus`.
- **Send one line at a time.** Long instructions go in the task file, then:
  `$T send TEAM ROLE "Re-read .team-task.md; I updated the Contract section."`
- **Wait on status, not on screens.** Use `$T wait`, never sleep and capture in
  a loop. Use `--screen` only to diagnose a failure.
- **Use the script, not raw `rex` calls** against team panes. It tracks the
  block IDs and remembers each worker's state before your last message, so
  `wait` does not return early.
- **Never send to your own pane.**

## Common mistakes

| Mistake | Fix |
|---|---|
| Two workers edit the same file, then merge conflicts | Give each file one owner with `$T own` and in "You own". Put shared types in the Contract and let one worker write them. `$T check` before merging |
| Worker builds against code another worker hasn't finished | Put the interface in the Contract; consumers stub against it |
| Reporting "done" from worker reports alone | Run the tests yourself and report the exit code |
| Starting 5 workers for a 20-minute task | Use 2–3 workers, or none |
| `$T up` fails with "already exists" | `$T status TEAM` to resume it, or `$T down TEAM` first |

## Worker permissions

Workers start in auto mode and load `<repo>/.team/settings.json` (view it with
`$T rules`):
- **Allow rules:** safe git, search, test and build commands run without a prompt.
- **Deny rules:** `git push`, `merge`, `rebase`, `switch`, `reset --hard`,
  `worktree`, `rm -rf`, `sudo` are always refused.

Merging and pushing are your job as lead, done with the user's agreement. When
a worker reports a denied command it needed, tell the user. Don't add allow
rules yourself unless the user asks.

`$T down` merges rules the user approved with "don't ask again" into the shared
list. Mention the `learned from …` lines it prints to the user.

`TEAM_CLAUDE_ARGS` overrides the flags; a `--model` there also overrides the
role file's model for that run, e.g.
`TEAM_CLAUDE_ARGS="--model haiku --permission-mode auto" $T add feat-x docs`.
