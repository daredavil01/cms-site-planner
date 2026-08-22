# CHANGELOG.md — template

> **To the skill.** Fill `{{PROJECT_NAME}}` and `{{DATE}}`. Ship it with one real entry — the
> creation of the document set — so the file starts as a record rather than a blank promise.
> Delete this blockquote from the generated file.

---

# Changelog

All notable changes to this project. Format follows
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning is **phase-based**, not
semver — see [`CLAUDE.md`](CLAUDE.md) §6.

Groups: **Added · Changed · Fixed · Removed · Security · Notes**.

Write entries for the person reading them later, not as a restatement of the diff — the diff
already has the *what*, and **the why matters more**. Record the failures too: an honest changelog
is a debugging tool, a promotional one is not.

What belongs here: user-visible changes, bugs that were subtle, decisions that diverged from the
plan, and anything security-relevant — RLS, the privileged-credential boundary, the admin gate.
Not every refactor; `git log` records those better.

What belongs in [`lessons.md`](lessons.md) instead: what was *learned*, including from work that
never shipped. The two files overlap on purpose.

---

## [Unreleased]

{{One line naming the phase in progress, once there is one.}}

### Added
- **Project documents.** [`implementation_plan.md`](implementation_plan.md) with
  {{PHASE_COUNT}} phases, [`CLAUDE.md`](CLAUDE.md) carrying the rules that apply to the modules
  selected ({{MODULE_LIST}}), this changelog, [`status.md`](status.md) and
  [`lessons.md`](lessons.md).

### Notes
- {{Any assumption applied because the brief was silent. Each one is also an open item in
  status.md; naming it here is what makes it findable later, when it turns out to be wrong.}}

---

## [0.0.0] — {{DATE}}

Repository created.
