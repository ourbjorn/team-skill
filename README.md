<img src="plugins/rex-x/.claude-plugin/icon.png" alt="" width="96" align="right">

# rex-x: Claude Code agent teams in Rex

`rex-x` is a Claude Code plugin with extensions for working in
[Rex](https://www.superlogical.com/rex/docs). Its first skill is **team**.

Run a **lead** Claude that splits work across several **worker** Claudes. Each
worker gets its own Rex tab and its own git worktree and branch. You can watch,
type to, or approve any worker at any time. The lead waits on Rex's built-in
agent status (`working`, `done`, `blocked`, `error`) instead of reading screens.

```
 you ──▶ lead (your Claude pane, uses /rex-x:team)
            │
            ▼
   Rex session "feat-x"
   ┌──────┬──────┬──────┐
   │  be  │  fe  │ test │   one tab + worktree + branch per worker
   └──────┴──────┴──────┘
```

## Install

**As a plugin** (recommended). In Claude Code:

```
/plugin marketplace add ourbjorn/team-skill
/plugin install rex-x@team-skill
```

Restart Claude Code, then run `/rex-x:team`, or just ask for a team (see Quick
start). To update later: `/plugin marketplace update team-skill`.

**For your whole organization:** an admin can add the marketplace and enable
the plugin for everyone in Claude Code's managed settings:

```json
{
  "extraKnownMarketplaces": {
    "team-skill": { "source": { "source": "github", "repo": "ourbjorn/team-skill" } }
  },
  "enabledPlugins": { "rex-x@team-skill": true }
}
```

**As a personal skill** (to hack on it). Clone the repo and link the skill
folder, so your edits go straight to the clone:

```bash
git clone https://github.com/ourbjorn/team-skill.git
ln -s "$PWD/team-skill/plugins/rex-x/skills/team" ~/.claude/skills/team
```

The skill is then `/team`. Don't install it both ways at once, or Claude sees
two copies of the skill.

**Where the skill lives** (for the `team` alias below): a linked skill is in
`~/.claude/skills/team`. A plugin install is under `~/.claude/plugins/`:

```bash
find ~/.claude/plugins -path '*rex-x*/skills/team/scripts/team' 2>/dev/null | tail -1
```

## Requirements

Check everything at once with `team doctor`. Every command also checks what it
needs and names anything missing, with an install hint.

| Required | Why | Install |
|---|---|---|
| [Rex](https://www.superlogical.com/rex/docs), app running | Sessions, tabs, agent status | From the Rex site |
| Claude Code (`claude`) | The lead and the workers | [code.claude.com/docs](https://code.claude.com/docs) |
| `git` 2.31 or newer | Worktrees and branches per worker | `xcode-select --install` or `brew install git` |
| `jq` | Reading Rex's JSON output | `brew install jq` |
| A git repository | Teams work inside one | |

| Recommended | Why |
|---|---|
| Run the lead Claude inside a Rex terminal | So you can watch and switch between workers |
| Remote Control on, in Rex settings | Lets `team focus` switch your screen to a worker |
| `rex skill setup` (run once) | Teaches Claude the rest of Rex. `team` itself doesn't need it |

## Quick start: let the lead drive

In a Claude session inside your repo:

> Use /rex-x:team to build the CSV export: a backend endpoint, a frontend button, and tests.

(Asking for "a team" in your own words works too.)

The lead reads the code, works out which workers are needed, and **shows you
the proposed team** (a plan file, plus a line per worker) before starting
anything. After you agree, it starts the team, writes one task per worker,
assigns them, waits, checks the results by running tests, and asks you how to
merge. Your part:

- **Approve prompts.** Blocked workers show ⚠ on their tab and in Rex's session
  picker. The lead tells you which ones, and never answers them for you.
- **Trust new worktree folders** if Claude asks. This only happens if no parent
  folder is trusted yet.
- **Decide on merges.** Branches are named `team/<team>/<role>`.

## Driving it by hand

First make the `team` command available (see Install for where the skill lives):

```bash
alias team="bash <skill folder>/scripts/team"

team up feat-x be fe                  # session + worktree + Claude per role
team task feat-x be < be-task.md      # or: team task feat-x be be-task.md
team task feat-x fe < fe-task.md
team assign feat-x be fe              # workers start
team wait feat-x --for 1800           # blocks until done / blocked / failed
team status feat-x                    # state, waiting on, report?, commits ahead
team read feat-x be                   # the worker's report
team down feat-x                      # close session, remove worktrees, keep branches
```

| Command | What it does |
|---|---|
| `up TEAM --plan FILE` | Create the team from a plan file (see Team plans) |
| `up TEAM [--base REF] SPEC...` | Create the team from a list. SPEC is `ROLE` or `ROLE@LABEL` |
| `add TEAM SPEC [--base REF] [PATH...]` | Add a worker to a running team |
| `task TEAM WORKER [FILE]` | Set a worker's task (file or stdin) |
| `own TEAM [WORKER PATH...]` | Set the paths a worker may change. With no worker: list all, show overlaps |
| `assign TEAM WORKER... [--force]` | Tell workers to start. Refuses while declared paths overlap |
| `send TEAM WORKER "msg"` | One-line follow-up (each newline submits a separate prompt) |
| `wait TEAM [WORKER...] [--for S]` | Exit codes: `0` done, `2` blocked, `3` failed or gone, `124` timeout |
| `status TEAM` | Table of all workers |
| `check TEAM` | Changes outside each worker's paths, and files changed by 2+ workers |
| `read TEAM WORKER [--screen N]` | Report, or the last N lines of the terminal |
| `stop TEAM WORKER` | Interrupt (Esc) |
| `focus TEAM WORKER` | Show the worker in the Rex app (needs Remote Control turned on) |
| `inbox` | Every Claude pane in Rex that needs you, across all sessions |
| `ls` | Teams in this repo |
| `rules` | The workers' shared allow/deny list |
| `doctor` | Check requirements: rex, jq, git 2.31+, claude, Rex server |
| `role ls\|new\|show\|edit\|rm` | Manage role files (see Creating roles) |
| `down TEAM [--force]` | Tear down. `--force` discards uncommitted worker changes |

## Team plans

A plan file describes a whole team: one worker per line, its paths, and
optionally where its branch starts.

```
# refactor-split.plan   (SPEC [--base REF] [PATH...]; # starts a comment)
be@core     src/core/
be@billing  src/billing/ tests/billing/
fe          web/src/settings/
```

```bash
team up refactor-split --base main --plan refactor-split.plan
```

- **Everything is checked first:** worker names, duplicates, git refs, existing
  branches, overlapping paths. All problems are listed at once, and if there
  are any, nothing is created.
- **`--base REF`** sets where branches start (default: HEAD). A plan line can
  set its own `--base` instead.
- **Waves for dependent work.** A worker starts from a fixed commit and never
  sees later changes, so work that depends on another worker goes in a later
  wave. It is stacked on that worker's branch once that work is done:
  ```bash
  team add refactor-split be@orders --base team/refactor-split/be-core src/orders/
  ```
  `check` and `status` then measure the stacked worker against its own start
  point.

You rarely write plans yourself: the lead writes one from its analysis and
shows it to you before starting.

## Several workers with the same role

A role can run as several workers, each on its own part of the code. Name them
`ROLE@LABEL`; each worker is called `ROLE-LABEL` and uses `ROLE`'s role file:

```bash
team up feat-x be@orders be@billing fe          # workers: be-orders, be-billing, fe
team own feat-x be-orders  src/orders/  tests/orders/
team own feat-x be-billing src/billing/ tests/billing/
team own feat-x                                 # list ownership, show overlaps
team assign feat-x be-orders be-billing fe      # refuses if paths overlap
team check feat-x                               # before merging
```

- **Paths** are repo-relative directories, files or globs (`*` also matches `/`).
- **`assign`** refuses while two workers' paths overlap, and tells each worker
  exactly which paths it may change. The overlap check is conservative: globs
  are compared by their directory part. Use `--force` if you've checked them.
- **`check`** lists files each worker changed outside its paths (committed or
  not), and files changed by more than one worker. It exits 1 on problems.
- **It works best** when parts have their own directories. A file that every
  part touches (shared types, router, DI setup) should belong to one worker,
  or be changed by the lead before assigning.

The lead does this for you when it splits work; ownership is also what goes
in each task's "You own" section.

## How workers behave

Each worker is told, through its system prompt, to:

1. Read `.team-task.md` in its worktree.
2. Work only inside that worktree and commit to its branch.
3. Write `.team-report.md` with Status, Summary, Files changed, Tests run and
   Open questions, then stop.

**Good task files** have the sections Goal, You own, Do not touch, Contract
(shared API and types, copied word for word into every task that depends on
them) and Done when (including a command that must exit 0). Workers see only
their task file and the repo, not your conversation.

## Roles

Built-in roles: `plan`, `be`, `fe`, `test`, `review`. Any other name gets the
generic `worker` prompt. To customize per repo, add `.team/roles/<role>.md`;
it takes precedence over the built-in file with the same name.

A role file can set the worker's model in frontmatter. Any `claude --model`
value works (`opus`, `sonnet`, `haiku`, or a full model ID):

```markdown
---
model: sonnet
---
# Role: security
You review branches for security issues. Do not modify code. ...
```

Built-in defaults: `plan`, `be` and `review` use `opus`; `fe` and `test` use
`sonnet`; `worker` uses Claude's default. To override a model for one run, put
`--model` in `TEAM_CLAUDE_ARGS`. `team up` prints the model each worker got.

### Creating roles

The easy way is to ask your Claude: *"Create a team role for a .NET backend
developer"*. The lead looks at your repo (stack, folders, the real test
commands), writes the role, picks a model and shows you the file.

By hand:

```bash
team role ls                                   # what exists, scope, model
team role new docs --model haiku               # template to fill in
team role new be-dotnet --from be              # copy a role (and its model)
team role new security --model opus - < security.md   # body from a file or stdin
team role show security                        # print it
team role edit security                        # open in $EDITOR
team role rm docs                              # delete (add --user for personal roles)
```

Roles come in three layers. The first match wins:

| Layer | Where | Use for |
|---|---|---|
| repo | `<repo>/.team/roles/` | Roles for one codebase. Committable, shared with your team |
| user | `~/.claude/team/roles/` (`--user`) | Your personal roles, in all your repos |
| built-in | inside the skill | The defaults. Replaced on update: `role edit` copies one to your user roles first |

The rest of `.team/` stays git-excluded. A good role has four parts: a one-paragraph mission, **You do**,
**You don't**, and **Done means** (the commands that must exit 0).

## Configuration

| Variable | Default | Use |
|---|---|---|
| `TEAM_CLAUDE_ARGS` | `--permission-mode auto` | Extra `claude` flags for workers. A `--model` here overrides the role file's `model:` |
| `TEAM_SETTINGS` | `<repo>/.team/settings.json` | Workers' shared allow/deny list. Set to `none` to disable |
| `TEAM_WORKTREES` | `<repo>/.worktrees` | Where worktrees go |
| `TEAM_POLL` | `3` | Seconds between status checks in `wait` |

## Permissions: few prompts, hard limits

Workers start in **auto mode** and load a **shared allow/deny list**, so they
rarely stop to ask you:

1. **Allow rules** (git status/diff/add/commit, ls/grep/find, common test and
   build commands) run with no prompt and no auto-mode check.
2. **Auto mode** decides everything else, and still asks you when it is unsure.
3. **Deny rules** always win, even in auto mode: `git push`, `merge`, `rebase`,
   `switch`, `reset --hard`, `worktree`, `rm -rf`, `sudo`.

The list lives in `<repo>/.team/settings.json`. It is copied from the skill's
`settings.json` the first time you start a team in a repo. Edit it freely, and
view it with `team rules`.

**Your approvals carry over.** When you pick "Yes, and don't ask again" in a
worker, Claude saves the rule in that worktree. `team down` merges those rules
into the shared list (and prints them as `learned from <role>: …`), so every
future worker in that repo inherits them. Rules that point into a worker's own
worktree path are skipped.

To get the old behaviour, where shell commands ask you first, use
`TEAM_CLAUDE_ARGS="--permission-mode acceptEdits"`.

## Files

```
<repo>/.team/roles/        your repo's roles                      (committable)
<repo>/.team/settings.json workers' shared allow/deny list        (git-excluded)
<repo>/.team/<team>/       session ID, block IDs, worktree paths  (git-excluded)
<repo>/.worktrees/<team>-<role>/   one worktree per worker        (git-excluded)
  ├── .team-task.md
  └── .team-report.md
```

## Tips and limits

- Start with 2–3 workers. More workers mean more prompts for you and more merging.
- Give each file exactly one owner, or you'll get merge conflicts.
- Workers are Claude Code only (roles use `--append-system-prompt`).
- If a test runner in the main repo picks up files from `.worktrees/`, set
  `TEAM_WORKTREES` to a folder outside the repo.
