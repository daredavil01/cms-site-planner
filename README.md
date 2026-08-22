# cms-site-planner

A Claude Code plugin that turns a website brief into a phased development plan and the documents
that keep the build honest afterwards.

It plans one stack: **Astro SSR on Cloudflare Workers, Supabase for Postgres/Auth/Storage, RLS as
the access boundary, and a custom admin in the same app.** The rules it carries are specific
enough to check because the stack is fixed — that is the trade, and it is deliberate.

**It writes documents, not code.**

## What it produces

| File | Holds |
|---|---|
| `implementation_plan.md` | Phases, each with a version, a risk rationale and a runnable checkpoint |
| `CLAUDE.md` | The rules for this project — only the ones the selected modules make true |
| `status.md` | Phase tracker, verification records, paths never exercised, open items |
| `CHANGELOG.md` | Open `[Unreleased]`, phase-based versioning |
| `lessons.md` | Append-only record of what the build teaches |

## Modules

The core is always planned. These are asked about and only planned if chosen:

`i18n` (multi-locale JSONB content) · `media` (ID-referenced media library) · `blog` · `forms`
(public-write surfaces) · `gallery` · `roles` (more than one kind of editor).

A module that is off contributes no tables, no phase content, and **no rules** — a monolingual
site never receives a translation rule it will not use.

## Install

```bash
claude plugin marketplace add /path/to/cms-site-planner
```

Then install `cms-site-planner` from that marketplace. Validate a local checkout with:

```bash
claude plugin validate /path/to/cms-site-planner
```

## Use

In an empty directory for the new site:

> Plan a new CMS-backed website. The brief is in `./brief.md`.

or, with no brief at all:

> Plan a new website for a community health charity — I'll answer questions.

The skill reads whatever you point it at, asks only about the gaps, and writes the five files.

## Where the rules came from

Two production sites on this stack. Every rule in `skills/cms-site-planner/references/` is stated
as **Rule · Why · Evidence**, and the evidence is the load-bearing part — a rule without the
failure it prevents gets argued away by the next person who finds it inconvenient.

## Scope

Greenfield only. It does not audit or re-plan a project that already has code.
