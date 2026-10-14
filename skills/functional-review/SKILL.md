---
name: functional-review
description: Run a functional UX review of a feature, design or diff to find issues that hurt real users — dead ends, data coherence problems, and persona gaps — using three fresh reviewers, one per lens. Use when the user says /functional-review, asks "does this work for users", or wants the UX, flow or usability of a feature reviewed.
---

# Functional review

A review of what real users hit: flows that dead-end, data that contradicts itself, and states that work for one kind of user but break for another. Not a security audit, not a performance pass, not a code-style review.

## Inputs

- **Target** — what gets reviewed: code (components plus the API or data code behind them), a diff, or a design or prototype.
- **Spec** (optional) — the stories and designed states the target must deliver, for example a design doc. When it's given, every lens also checks that each designed state exists in the target and behaves as specified.
- **Personas** — the project's list of user types. Look for `docs/PERSONAS.md` in the target's repo. If there's none, the persona-gap reviewer works the user types out from the code alone.

## The three lenses

Each lens owns its own territory, set by its last line. A finding goes to the lens that owns it.

### 1. Loop closer — states with nowhere to go
Every state a user can reach needs a meaningful next step.
- Error states offer retry, recovery or navigation, not just a message.
- Success states lead to what comes next.
- Empty states say why they're empty and give a path forward.
- Loading and polling states have a timeout with a graceful exit.

Owns every dead end that hits **all** users.

### 2. Data coherence — wrong or inconsistent data
- Data the user already gave (amounts, choices, settings) informs the UI. Collected but ignored is a finding.
- A default that silently beats a value the user entered is a finding.
- A boolean read as "missing" when it means a confirmed "no" is a finding.
- Conditional logic that misreads real inputs (for example `every` where `some` was meant, on sparse data) is a finding.
- The same fact shown with different values in two places is a finding.

Owns every finding about **what** is shown, whoever sees it.

### 3. Persona gap — failures that hit some users but not all
- Start from the personas file if there is one. Then find every user type the code supports: role and permission checks, read-only flags, account state (new, set up, active), plan or tier, and feature flags.
- A user type the target supports that the file doesn't list is a finding. Only when the target is the whole feature's code: a listed type the code never handles is a finding too. A design or a diff only covers part of the app, so personas it doesn't touch are not findings.
- Walk each user type through the key flows.
- Report failures that hit **some user types but not all**: a read-only user reaching a control that changes things, a user who can reach a state but can't act in it, or a flow that degrades for some types.

Owns every failure that hits some user types but not all, except wrong or inconsistent data, which stays with data coherence.

## How to run it

1. **Pin the inputs.** Name the target, the spec if there is one, and the personas file if there is one.
2. **Run the three lenses with fresh context**: three new agents in parallel (not forks), each given the target, the spec and its lens prompt below — plus the personas file for the persona-gap reviewer — and nothing else from this conversation. If you can't spawn agents here (for example, you are already a subagent), run the lenses one after another yourself, and say so in the output. In that case you are also the reviewer, so list every finding you dropped, and why, under "How it ran".
3. **Synthesize without filtering.** Merge findings that are the same issue, keeping the highest severity. Never drop a finding or lower its severity: the reviewers set severity, not the session that asked for the review.

**Evidence rule (shared by every prompt).** Each finding cites evidence: `file:line` for code, `doc §section` for a design, or, for something missing, the state or story that should exist and where it was looked for. Each prompt tells its reviewer to drop findings it can't back with evidence; the synthesis never drops anything.

**Loop-closer prompt:**
```
Target: [TARGET]. Spec: [SPEC or "none"].
You are performing a loop-closer review. Find every state a user can reach (loading, error, empty, success, rate-limited, timed out, already running, and so on) and check that each one gives the user a meaningful next step: retry, recovery, navigation, or what comes next. Report dead ends that hit every user type; leave failures that hit some user types but not all to another reviewer. If a spec is given, also report every designed state that is missing or behaves differently. If the target has no user-facing states, return "N/A: <reason>".
For each finding give: severity (Critical: can't complete the primary task; High: dead end or wrong answer in a common flow; Medium: degraded or misleading in a secondary flow; Low: minor friction), what the user experiences, the evidence (file:line, doc §section, or the missing state and where you looked), and a one-sentence fix. Drop any finding you can't back with evidence, and any that needs unusual conditions to happen. No security, performance or style.
```

**Data-coherence prompt:**
```
Target: [TARGET]. Spec: [SPEC or "none"].
You are performing a data-coherence review. Find data that is available but unused, defaults that silently override what the user entered, booleans misread as missing, conditional logic that gives wrong results for real inputs, and the same fact shown with different values in different places. Where the target has an API or data layer, check what it returns alongside errors, not just what gets rendered. If a spec is given, also report data the spec says to show or use that the target ignores. If the target shows no data, return "N/A: <reason>".
For each finding give: severity (Critical: can't complete the primary task; High: dead end or wrong answer in a common flow; Medium: degraded or misleading in a secondary flow; Low: minor friction), what the user experiences, the evidence (file:line, doc §section, or the missing data and where you looked), and a one-sentence fix. Drop any finding you can't back with evidence, and any that needs unusual conditions to happen. No security, performance or style.
```

**Persona-gap prompt:**
```
Target: [TARGET]. Spec: [SPEC or "none"]. Personas: [PERSONAS FILE or "none"].
You are performing a persona-gap review. Start from the personas file if one is given, then identify every kind of user the target supports: role and permission checks, read-only flags, account state, plan or tier, feature flags. Report any user type the target supports that the file doesn't list; if the target is the whole feature's code, also report listed types it never handles. Don't report personas a design or diff simply doesn't touch. Walk each type through the primary flows. Report failures that hit some user types but not all: a dead end, a control they can reach but shouldn't use, or a degraded flow with no alternative. Leave wrong or inconsistent data to another reviewer, even when only one user type sees it. If there is only one kind of user, return "N/A: <reason>".
For each finding give: severity (Critical: can't complete the primary task; High: dead end or wrong answer in a common flow; Medium: degraded or misleading in a secondary flow; Low: minor friction), the user type, what they experience, the evidence (file:line, doc §section, or the missing state and where you looked), and a one-sentence fix. Drop any finding you can't back with evidence, and any that needs unusual conditions to happen. No security, performance or style.
```

## Output

### Summary
Two to four sentences: name the feature, say what it does well, and state the core failure mode in plain language.

### Findings

| Lens | Severity | Finding | Evidence |
| --- | --- | --- | --- |
| Loop closer | High | The error state has no retry, so the user is stuck | `src/Pay.tsx:88` |

A lens that returned N/A gets one row saying so, with its reason.

### Details
One paragraph per finding: what the user experiences, the evidence, why it's wrong, and the fix in one sentence.

### Top 3 actions
Numbered, most impactful first, one sentence each, naming the specific change.

### How it ran
- Fresh agents in parallel, or the lenses in sequence (and why).
- The target and spec each reviewer was given, and the personas file given to the persona-gap reviewer.
- The framing line of each lens prompt as sent (the line starting "You are performing").

## Leave out

- Security findings (auth, injection, data exposure).
- Performance findings.
- Code style or refactoring suggestions.
- Hypothetical issues that need unusual conditions.
