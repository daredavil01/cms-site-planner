# status.md — template

> **To the skill.** Fill every `{{placeholder}}`. The phase rows must match
> `implementation_plan.md` exactly. Delete this blockquote from the generated file.

---

# {{PROJECT_NAME}} — Status

**Owner:** {{OWNER}} · **Last updated:** {{DATE}} · **Commit:** {{SHA}} · **Version:** {{VERSION}}

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
| ✅ | Done and verified under `npm run preview` |
| 🟨 | Started, not verified |
| ⬜ | Not started |
| ⚠ | Done, with a caveat recorded below |

## Phase overview

| Phase | Name | Version | State |
|---|---|---|---|
{{PHASE_ROWS — one per phase, all ⬜ at generation}}

---

{{Per phase, one section:}}

## Phase {{N}} — {{NAME}} ⬜

**Goal:** {{from implementation_plan.md}}

**Checkpoint:** {{the runnable command}}

{{Sub-items as they are built. Record what was built, not what was planned — the plan is in the
other file.}}

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

- No automated test suite exists. `/api/health` and this table stand in for one.
{{ADDITIONAL — e.g. an unproven locale count, a write path with no fixture, a module interaction
designed but never run}}

## Blockers and risks

| # | Item | Impact | Owner | Status |
|---|---|---|---|---|
| | | | | |

## Open items requiring client input

Each with what is blocked and what answer unblocks it. **This is the maintained copy** —
`implementation_plan.md` §6 duplicates it and goes stale first.

| # | Question | Blocks | Asked | Answered |
|---|---|---|---|---|
{{OPEN_ITEMS from the interview — every default you applied without an answer belongs here}}

## Change log

Brief, dated, one line per update to *this file*. The project's changelog is
[`CHANGELOG.md`](CHANGELOG.md).

| Date | Change |
|---|---|
| {{DATE}} | Created from the implementation plan. All phases ⬜. |

## How to update this file

At every phase boundary, in this order: mark the phase row · add a verification-record row
including what was not checked · move anything newly known into blockers or open items · refresh
the header (date, commit, version) · then update [`CHANGELOG.md`](CHANGELOG.md) and
[`lessons.md`](lessons.md).
