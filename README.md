# skills

Danny's Claude Code skills.

| Skill | What it does |
| --- | --- |
| [`ship`](skills/ship/SKILL.md) | Runs a coding task through gates: Design → Plan → Plan review → Build → Simplify → Review → Verify → PR. It stops at an open PR with green CI and a gate report. |

## Install

### 1. Install Matt Pocock's skills first

`ship` calls four of them: `code-review` and `tdd` always, `prototype` and `grilling` when you're around to answer their questions. Install them before `ship`. If they're missing, `ship` names them at the start and runs those steps by hand, which works less well.

In an interactive `claude` session:

```
/plugin marketplace add mattpocock/skills
/plugin install mattpocock-skills@mattpocock
```

Tested with `mattpocock-skills` v1.3.1. `ship` refers to these skills by their full names (`mattpocock-skills:code-review` and so on), because Claude Code also has a built-in `/code-review`.

### 2. Install ship

Clone the repo and symlink the skill, so a `git pull` updates it:

```bash
git clone https://github.com/dansnacks/skills.git
ln -s "$PWD/skills/skills/ship" ~/.claude/skills/ship
```

If you'd rather copy it (`cp -R skills/skills/ship ~/.claude/skills/`), re-copy it after every pull, or Claude keeps loading the old version.

### 3. Check it loaded

Start a new Claude Code session and type `/`. Both `ship` and `mattpocock-skills:code-review` should be in the list.

## Optional

- **`functional-review`** — a UX review across three lenses: dead ends, data coherence and persona gaps. `ship` uses it if it's installed, and folds the same checks into its own review if not. It isn't in this repo yet.
- **A project verify skill** — if your repo has `.claude/skills/verify/SKILL.md`, `ship` adds its commands to the verify set.

## How ship works

- **Size first.** S gets an inline design and one review. M gets every gate. L gets split into PR-sized slices.
- **Fresh adversarial eyes.** Subagents with isolated context review the plan before any code and the diff after it. They are told the work is broken and asked to find how.
- **Simplify before review,** so the reviewers read the code that ships.
- **One fix loop.** Every fix (except copy, comment or formatting changes) gets a fresh review of what changed. Review findings, a failed verify and red CI all share one cap of 3 rounds.
- **Ends at an open PR with green CI.** A person merges. The repo's own rules (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`) win wherever they cover branching, tests, commits, PRs or merging.
- **Unattended runs** write their assumptions into the design doc and list open questions in the final report.
