---
name: ship
description: Run a non-trivial coding task through a gated senior-engineer loop — Design → Plan → Plan-review → Build → Simplify → Review → Verify → PR — with a hard gate between phases, ending at an open PR with green CI and a gate report. Use when the user types /ship, asks to "run this through the loop", wants design-before-code discipline, or starts a task big enough to warrant a spec and an independent review before it's called done.
---

# Ship

A gated loop around a coding task. It ends at an **open PR with green CI and a gate report**. A person merges.

The repo's own rules (`CLAUDE.md`, `AGENTS.md`, `CONTRIBUTING.md`) win wherever they cover branching, tests, commits, PRs or merging. This skill fills the gaps they leave.

## Start: preflight and gate plan

1. **Check dependencies.** This skill calls `mattpocock-skills:grilling`, `mattpocock-skills:prototype`, `mattpocock-skills:code-review` and `mattpocock-skills:tdd`, plus the built-in `simplify`. If any of them isn't in your skill list, name it, point the user to the install steps in this skill's README (`/plugin marketplace add mattpocock/skills`, then `/plugin install mattpocock-skills@mattpocock`), and do that gate by hand from its description. Never drop a gate because a skill is missing. `functional-review` is optional: use it if it's installed.
2. **Size the task.** This sets how much ceremony each gate gets:
   - **S** — about one file, no new mechanism, and nothing that would trigger Gate 3's soundness lens (any such trigger makes it M). S runs Gates 0, 1 (inline: a short paragraph in chat, no doc, still approved), 4, 6 (fresh attack only), 7 and 8. In place of Gate 2, state the verify set in one line. Gate 3 and Gate 5 are skipped and say so.
   - **M** — the default. Every gate as written below.
   - **L** — more than one PR's worth of work. Design the whole thing once and land the design doc first, in its own PR or as the first slice, so every later slice branches from a base that has it. Then split the work into PR-sized slices; each runs Gates 0 and 2–8 on its own branch. Keep a progress checklist in the design doc and tick it in each slice's PR, so the work survives a context reset.
3. **State the gate plan** in one block: the size, which conditional pieces apply and why, and whether the user is reachable (see Unattended runs). Every gate you skip later gets one line saying why.

When unsure, pick the larger size. A task that grows mid-way gets re-sized out loud.

## Gate 0 — Branch

Follow the repo's branching rule. Without one: fetch, branch from the up-to-date base branch, and name the branch for the change. Never commit to the base branch. The design doc and the code go on this branch.

## Gate 1 — Design (no code yet)

1. Orient: read the structure, recent commits and the files the task touches.
2. **Name the incumbent.** List the generic capabilities the design needs (rate limiting, dedupe, caching, locking, retry, …). For each one, grep for an existing implementation and name it, or state that none exists. A new mechanism must say why the incumbent doesn't fit. Reuse is decided here; later gates only polish code that already exists.
3. If the request is vague, ask clarifying questions one at a time. Ask about mechanisms before parameters: don't ask for a threshold on a mechanism that hasn't been approved (step 5).
4. **User-facing change:** write user stories ("As a <role>, I want <capability> so that <benefit>") and sketch the UX: the primary flow, *every* state (empty, loading, error, success, partial), the interaction model and the key copy. This is the contract that Gates 3, 6 and 7 check against.
5. Propose 2–3 approaches that differ in **mechanism**, with trade-offs and a recommendation. The same code with a knob toggled counts as one approach. Get approval.
6. M and L: write the design to `docs/specs/YYYY-MM-DD-<topic>-design.md` (or the repo's spec folder), sized to the task, and commit it on the branch.

**Gate:** no planning until the design is approved.

## Gate 2 — Plan (orchestrator writes it)

1. Break the design into steps. Each step names exact file paths, the concrete change, and the command that proves it. No "TBD", no "similar to above".
2. Name the verify set now: the repo's required test suites for the touched areas, the build, the commands in the repo's own verify skill (`.claude/skills/verify/SKILL.md`) if it has one, and — where the repo asks for one — a regression test that fails before the change. Gate 7 runs this set and can't shrink it.
3. Self-review for placeholders, contradictions, missing steps and scope creep. This is a filter that clears the obvious before Gate 3, not a substitute for it.

**Gate:** no code until the plan is concrete and has passed Gate 3.

## Gate 3 — Adversarial plan review

Spawn a **fresh subagent** (a new agent, not a fork; a fork inherits this conversation): give it the design and the plan only. Tell it the plan is flawed and its job is to find how:
- Where is it underspecified?
- Which paths or APIs are guessed?
- What step is missing or out of order?
- What will the implementer have to invent?
- What is the riskiest assumption?

Skip only at size S, and say so.

**+ UX lens — user-facing work.** Does the UX serve each story? Is every state designed? Look for dead ends, persona gaps, responsive, dark mode and accessibility. When the interaction is new and the user is reachable, build a throwaway with `mattpocock-skills:prototype` (it asks the user questions) and click through it. If `functional-review` is installed, run it in a fresh subagent.

**+ Soundness lens — new or changed data model, migration, multi-module contract, security or permission boundary, concurrency, or external API shape.** Even if built perfectly, is the design correct? Spawn a fresh subagent told the design is unsound, asking "what breaks this?" — edge cases, failure modes, data integrity, contracts. When the user is reachable and the open questions are theirs to answer, `mattpocock-skills:grilling` (an interview with the user) can follow it.

A finding that changes the design's mechanism goes back to Gate 1 for approval. Other findings are fixed in the plan; a materially changed plan gets one more Gate 3 pass on what changed.

**Gate:** no code until the plan has had adversarial fresh eyes, and each applicable lens has run or is stated N/A.

## Delegate-build path (decided here, per slice)

The orchestrator builds by default: the agent that holds the design and the trade-offs is the best implementer. Consider handing a slice to a cheaper model only when **all** of these hold:
- the slice names exact paths and a concrete change;
- it mirrors an existing pattern, with no new API or contract;
- it involves no architecture, security, concurrency or cross-module judgement;
- it has a verify command, and its blast radius is low or reversible.

If a slice qualifies, ask the user which model to use: offer the options with a one-line trade-off each and your recommendation, then reuse their answer for later slices. Never pick the model yourself. Never delegate in an unattended run.

Each delegated slice gets a handoff containing the approved design, its plan slice, the Gate 3 findings, the non-goals and the verify command. Review never tiers down. A slice that fails Gate 6 or 7 twice comes back to the orchestrator.

## Gate 4 — Build

- Find the root cause before fixing anything. Failed fixes count toward the fix loop's 3-round cap.
- Touch only what the task requires.
- Where the plan names a regression test, write it first and watch it fail. `mattpocock-skills:tdd` fits here.

## Gate 5 — Simplify (conditional)

Applies when the build added real new code: new functions or modules, or repeated patterns. Skip it when the diff is under about 30 changed lines or a pure rename or move, and say so.

Run `simplify` for reuse, simplification, efficiency and altitude cleanups. It runs **before** review, so the reviewers read the code that ships.

## Gate 6 — Adversarial review (the key gate)

Run these in parallel:

1. **Fresh attack.** Spawn a fresh subagent (not a fork), given only the spec (or the S-size inline design) and the diff against the base branch. Tell it the diff is broken and ask it two things:
   - (a) **Conformance:** does the diff build what the spec asks, story by story?
   - (b) **Defects:** edge cases, error paths, races, regressions, security holes and wrong assumptions.
2. **Standards and spec (M and L).** Run `mattpocock-skills:code-review`, passing **both** the fixed point (the base branch; it reviews the three-dot diff against the merge-base) and the design doc path, and say the design doc is the only spec — otherwise it looks for issue references in commit messages first. If it stops — for example to ask for `docs/agents/issue-tracker.md` — answer it from this context and carry on. A run that doesn't finish counts as a failed gate, not a skipped one.
3. **UX drift (user-facing).** If `functional-review` is installed, run it in a fresh subagent against the Gate 1 UX. Otherwise fold the UX states into the attack prompt in item 1.

Fix every finding, or decline it with a reason. Declines go in the report, and any declined high-severity finding goes to the next fresh review with your reason, so a reviewer — not you — accepts the decline.

**Gate:** no verify until every finding is fixed or declined.

## The fix loop (Gates 6, 7 and CI)

- Every fix gets a fresh review of **only what changed**, with the original finding attached. The only exception is a change to copy, comments or formatting.
- A failed verify or red CI goes back to Gate 4, then a Gate 6 review of the change.
- **One counter covers the whole loop:** a round is one pass of fix → review → verify, whether it started from a review finding, a failed verify or red CI. Cap it at **3 rounds**. If it is still failing after that, stop, report what is failing and why (Status: stopped), and question the design instead of patching again.

## Gate 7 — Verify

Run the verify set from Gate 2 fresh and read the actual output. Report real pass and fail counts; "should pass" is not evidence.

For user-facing work, also drive the real UI (preview tools, the dev server, or the repo's preview environment). Walk the stories and every designed state, and capture evidence: screenshots, console and network.

## Gate 8 — PR

1. Commit using the repo's message style. Rebase onto a fresh base branch only if the repo's rules say to.
2. Push once per fix round, not per commit, then open the PR. The body says what changed and why, and includes the evidence. Add before/after screenshots for any visible change.
3. Wait for CI. Red CI goes back to the fix loop.
4. **Stop at green CI.** Don't merge unless the user asks.

## The report (last message)

```
Status: done | stopped: <what is failing, after <n> rounds>
Size: S | M | L     Branch: <name>     PR: <link>     CI: <green | red | pending>
Gates
  0 Branch     <branch> from <base>@<sha>
  1 Design     docs/specs/<file> | inline (S)  — approved by user | assumed (see open questions)
  2 Plan       <n> steps | one-line verify set (S)
  3 Plan review  <n> findings: <n> fixed, <n> declined | skipped: <reason>
     UX lens   ran | N/A: <reason>      Soundness lens  ran | N/A: <reason>
  4 Build      local | delegated: <slices → model>
  5 Simplify   ran | skipped: <reason>
  6 Review     attack: <n> found, <n> fixed, <n> declined · code-review: <n> standards / <n> spec | N/A (S) · UX drift: ran | N/A
  7 Verify     <command> → <passed>/<total>  (one line per command) · UI evidence: <where>
  Fix rounds   <n> of 3
Reviewer prompts   for each fresh subagent: its inputs and its framing line, quoted
Declined findings  <finding → reason → reviewer accepted? >, or none
Open questions     <only for unattended runs>
```

## Unattended runs

When the user isn't reachable to approve the design or answer questions:
- Pick the recommended approach and write every assumption into the design doc.
- Never delegate the build.
- Run every gate as normal and stop at the open PR. Never merge.
- List every decision the user would have made under **Open questions** in the report.

## Red flags

- "I'll build a small helper for this" — did you name the incumbent in Gate 1?
- "The reviewer said it looks good" — did you tell it the work was broken, with isolated context? A neutral ask gets a rubber stamp; re-run it adversarially.
- "I'll tidy it up after review" — Simplify comes before review, or the reviewed code isn't the code that ships.
- A gate missing from the report — every gate shows as ran or skipped with a reason. A gate you can silently drop isn't a gate.
