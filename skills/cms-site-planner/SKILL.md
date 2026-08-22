---
name: cms-site-planner
description: Turn a website brief into a phased development plan plus the project's rules and tracking documents. Use when planning a new CMS-backed website, starting a new client site, or asked for a phased development plan, implementation plan, or project setup docs for an Astro + Cloudflare Workers + Supabase build. For greenfield projects only — not for changes inside a site that already exists.
---

# CMS site planner

Plans a new website on a fixed stack — Astro SSR on Cloudflare Workers, Supabase for Postgres,
Auth and Storage, Row-Level Security as the access boundary, and a custom admin in the same app.
Optional capabilities are chosen per project. **This skill writes documents, never code.**

## What it emits

Five files, into the new project's directory:

```
implementation_plan.md   phases, each with a version, a rationale and a checkpoint
CLAUDE.md                the rules for this project — only the ones that apply
status.md                phase tracker, verification records, open items
CHANGELOG.md             open [Unreleased], phase-based versioning
lessons.md               append-only record of what the build teaches
```

## Procedure

1. **Read what exists.** If the user pointed at a brief, a `dev-docs/` folder or any
   requirements document, read it first. Never re-ask something the brief already answers.
2. **Interview the gaps.** Work through `references/interview.md`. Ask with `AskUserQuestion`,
   batching related questions. Stop asking once every module decision and every phase boundary
   is determined.
3. **Derive the phases.** Follow `references/phases.md`.
4. **Emit the five files** from `templates/`, filling every placeholder.
5. **Summarise**: which modules were selected, what you assumed, and what still needs the
   client — the last of these goes into `status.md` as well, not only into the chat.

## Where to read what

| Doing this | Read first |
|---|---|
| Interviewing | `references/interview.md` |
| Choosing tables, policies, buckets | `references/schema.md` |
| Deriving phases | `references/phases.md` |
| Writing the project's `CLAUDE.md` | `references/rules-core.md`, plus `references/rules-<module>.md` for each selected module that has one |

Module keys the interview returns: `i18n` · `media` · `blog` · `forms` · `gallery` · `roles`.
Only `i18n` and `media` carry their own rules file; the rest contribute schema and phase scope
only. A module that was not selected contributes nothing — no tables, no phase content, and
**no rules in the generated `CLAUDE.md`**.

## Hard stops

- Ambiguity is a question, not a coin flip. If a silent brief would change the plan, ask.
- Never emit invented content as published. Placeholder copy starts `published = false`, and
  an empty state is safer than a plausible one.
- Every phase carries a version, a one-line risk rationale, and a runnable checkpoint.
- Write every generated file as UTF-8 **without a BOM**, and never leave a `{{placeholder}}`,
  a `TODO` or a `…` in the output.
