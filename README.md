# skills

Danny's Claude Code skills.

| Skill | What it does |
| --- | --- |
| [`ship`](skills/ship/SKILL.md) | Runs a coding task through gates: Design → Plan → Plan review → Build → Simplify → Review → Verify → PR. It stops at an open PR with green CI and a gate report. |
| [`functional-review`](skills/functional-review/SKILL.md) | A UX review by three fresh reviewers in parallel: dead ends, data that contradicts itself, and users who get stuck. `ship` uses it for user-facing work. |

## Install

### 1. Install Matt Pocock's skills first

`ship` calls four of them: `code-review` and `tdd` always, `prototype` and `grilling` when you're around to answer their questions. Install them before `ship`. If they're missing, `ship` names them at the start and runs those steps by hand, which works less well.

In an interactive `claude` session:

```
/plugin marketplace add mattpocock/skills
/plugin install mattpocock-skills@mattpocock
```

Tested with `mattpocock-skills` v1.3.1. `ship` refers to these skills by their full names (`mattpocock-skills:code-review` and so on), because Claude Code also has a built-in `/code-review`.

### 2. Install the skills

Clone the repo and symlink each skill, so a `git pull` updates them:

```bash
git clone https://github.com/dansnacks/skills.git
mkdir -p ~/.claude/skills
ln -s "$PWD/skills/skills/ship" ~/.claude/skills/ship
ln -s "$PWD/skills/skills/functional-review" ~/.claude/skills/functional-review
```

If you'd rather copy them, copy both (`cp -R skills/skills/ship skills/skills/functional-review ~/.claude/skills/`) and re-copy after every pull, or Claude keeps loading the old versions.

### 3. Remove old copies

If you have an older `functional-review` as a command, delete `~/.claude/commands/functional-review.md` so it doesn't shadow the skill. Same for any older `ship` in `~/.claude/skills/` or `~/.claude/commands/`.

### 4. Check it loaded

Start a new Claude Code session and type `/`. `ship`, `functional-review` and `mattpocock-skills:code-review` should all be in the list. `ship` still works without `functional-review`; it folds the UX checks into its own review.

## Optional

- **A project verify skill** — if your repo has `.claude/skills/verify/SKILL.md`, `ship` adds its commands to the verify set.

## How ship works

- **Size first.** S gets an inline design and one review. M gets every gate. L gets split into PR-sized slices.
- **Fresh adversarial eyes.** Subagents with isolated context review the plan before any code and the diff after it. They are told the work is broken and asked to find how.
- **Simplify before review,** so the reviewers read the code that ships.
- **One fix loop.** Every fix (except copy, comment or formatting changes) gets a fresh review of what changed. Review findings, a failed verify and red CI all share one cap of 3 rounds.
- **Ends at an open PR with green CI.** A person merges. The repo's own rules (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`) win wherever they cover branching, tests, commits, PRs or merging.
- **Unattended runs** write their assumptions into the design doc and list open questions in the final report.
