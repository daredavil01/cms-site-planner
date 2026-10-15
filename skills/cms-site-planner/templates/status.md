# status.md — template

> **To the skill.** Fill every `{{placeholder}}`. The phase rows must match
> `implementation_plan.md` exactly. Delete this blockquote from the generated file.

---

# {{PROJECT_NAME}} — Status

**Owner:** {{OWNER}} · **Last updated:** {{DATE}} · **Commit:** {{SHA}} · **Version:** {{VERSION}}

**Current phase:** {{phase number and name, or "none started"}}

{{If the owner builds everything before deploying: "**Owner decision ({{DATE}}):** every phase is
built and verified locally; the site deploys once, after phase {{N}}. What production needs
accumulates in [`deployment.md`](deployment.md)."}}

> **This file is authoritative for current state.** Where [`CLAUDE.md`](CLAUDE.md) or
> [`implementation_plan.md`](implementation_plan.md) disagree with it about what is done, they are
> wrong and get corrected.
>
> It names an owner because a document stating current state without one will lie — most
> confidently to the newest person on the project.
>
> **A status document is not evidence about the repository.** Before marking a row ✅, check the
> repository, not this file.

## Legend

| Mark | Meaning |
|---|---|
| ✅ | Done and verified under `npm run preview`, with the phase's tests passing |
| 🟨 | In progress — started, not finished or not verified |
| ⛔ | Blocked — the blocker is named in the notes |
| ⬜ | Not started |
| ⚠ | Done, with a caveat recorded below |

## Phase overview

| Phase | Name | Version | State | Blockers and notes |
|---|---|---|---|---|
{{PHASE_ROWS — one per phase, all ⬜ at generation. The notes column is where out-of-scope work
found during a phase goes, so it is not lost and does not leak into the phase.}}

## Setup checklist (owner)

Accounts and services the build depends on. **Confirmed** means the owner said so; **verified**
means someone checked it from here. Only verified items are done.

| Item | Confirmed | Verified | Needed by phase |
|---|---|---|---|
{{One row per item from the interview: repository, Cloudflare account (and that `wrangler whoami`
shows it), domain and DNS, databases and buckets, bot-check keys, staff sign-in, email sending
domain, the client's first answers. Unknown items start ⬜ in both columns.}}

---

{{Per phase, one section:}}

## Phase {{N}} — {{NAME}} ⬜

**Goal:** {{from implementation_plan.md}}

**Checkpoint:** {{the runnable command}}

{{Sub-items as they are built. Record what was built and which tests prove it, not what was planned
— the plan is in the other file. When a phase stops unfinished, list exactly what is left.}}

---

## Verification record

Filled in as each phase closes. **Record what was *not* checked as well** — "never exercised" and
"never built" look identical in a tracking document, and only one of them is a bug.

| Date | Phase | What was verified | How | Result |
|---|---|---|---|---|
| | | | | |

### Paths never exercised

{{The honest list. Start it at generation with whatever the plan already knows is untested, and
add to it every phase. An admitted gap gets closed; an unexamined one does not.}}

{{ADDITIONAL — e.g. the chosen stack's unproven areas from references/stacks.md §3, an unproven
locale count, a write path with no fixture, a module interaction designed but never run}}

## Decisions

Conflicts between the design and the requirements, and choices the documents left open. A default
applied without a decision is still recorded here, with what it changed — and, when it departs from
a requirement document, which document should be updated.

| # | Decision | Needed by | Status |
|---|---|---|---|
{{D1, D2, … from implementation_plan.md §1.2, each "Open (default: …)" at generation. Later:
"Default applied in phase N: …" or "Decided by <who> on <date>: …".}}

## Blockers and risks

| # | Item | Impact | Owner | Status |
|---|---|---|---|---|
{{Anything known at generation — an unprovisioned account, a free-tier limit that binds, a stack
risk such as a pausing database.}}

## Open questions requiring client input

Each with what is blocked and what answer unblocks it. **This is the maintained copy** —
`implementation_plan.md` §7 duplicates it and goes stale first.

| # | Question | Blocks (phase, values) | Asked | Answered |
|---|---|---|---|---|
{{OPEN_ITEMS from the interview, numbered Q1, Q2, … — every default you applied without an answer
belongs here}}

## Deployment

What production needs — migrations, bindings, variables, secrets, outside services, the deploy order
and the checks afterwards — is in [`deployment.md`](deployment.md). Every phase that adds one of
those updates it.

## Session log

One row per working session, newest first.

| Date | Phase | Summary |
|---|---|---|
| {{DATE}} | — | Created from the implementation plan. All phases ⬜. |

## How to update this file

At the end of every session, in this order: mark the phase row (and its notes) · add a
verification-record row including what was not checked · move anything newly known into blockers,
decisions or open questions · add a session-log row · refresh the header (date, commit, version,
current phase) · then update [`deployment.md`](deployment.md) if needed, [`CHANGELOG.md`](CHANGELOG.md)
and [`lessons.md`](lessons.md).
