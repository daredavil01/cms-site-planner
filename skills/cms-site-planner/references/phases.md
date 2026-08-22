# Deriving phases

Phase count and boundaries come from the project's scope. They are **not** a fixed template — but
derivation is not improvisation either, and four constraints below make the difference.

---

## The four constraints

1. **The first phase is always the foundation**, and it ends at a green `/api/health` under the
   production runtime. Not a page. Its purpose is to answer the riskiest question in the stack
   while the answer is still cheap (`rules-core.md` §1–3).
2. **The last phase is always launch, at `1.0.0`**, and it is where the client's real content
   lands. Everything before it runs on empty states and unpublished drafts.
3. **Every phase carries three things**: a version, a one-line **risk rationale** saying why *this*
   boundary exists, and a **runnable checkpoint** — a command whose output settles whether the
   phase is done. A phase with no rationale is a list of tasks with a heading; a phase with no
   checkpoint completes on "it renders".
4. **Emit the phase → version table explicitly**, in both `implementation_plan.md` and the
   generated `CLAUDE.md` §6. Because phase count varies per project, no mapping can be assumed —
   a reader who has to infer it will infer it wrongly.

## Versioning

Minor per phase, patch for everything else, `1.0.0` at launch:

| Phase | Version |
|---|---|
| 0 — foundation | `0.1.0` |
| 1 … n−1 | `0.2.0` … |
| n — launch | `1.0.0` |

This is phase-based, not semver-for-APIs — a website has no downstream consumers, so "breaking
change" has no useful meaning here. A phase that would land after launch is unversioned and lives
under a deferred heading.

## How to cut the boundaries

Order by **risk retired**, not by how the site is organised. Each boundary should be a point you
would be willing to roll back to.

- **Foundation first** — runtime, env schema, request-scoped client, health endpoint, CI, a
  `.gitattributes` with `* text=auto eol=lf`.
- **Schema before anything consumes it.** If the `i18n` module is on, the locale machinery belongs
  in the same phase as the schema, because every later phase depends on both. Ship a deliberately
  half-translated row as a fixture.
- **The riskiest public surface next**, then the remainder. Split public pages into two phases only
  when the first one proves a pattern the second repeats.
- **Admin after the public read paths work**, so the write path is built against a schema already
  known to render.
- **Do not defer the UI half of a data model past the phase that creates it.** A schema whose
  screen never arrived is a schema nothing can use (`rules-media.md` §1).
- **A launch-readiness phase before launch**, holding the checks that are only meaningful against
  the finished site: accessibility pass, metadata, performance, a real-device check.

### What forces an extra phase

A module with its own risk: `media` (upload path, egress budget), `roles` (a policy predicate in
every staff policy), `forms` (public-write surfaces and their insert-only policies). Two or three
such modules usually mean one extra phase, not three.

### What does not

Page count. Ten static pages that share one layout are one phase, not five.

---

## Worked example A — monolingual brochure site, one editor

Modules: none. Six content types, one admin, no uploads beyond a handful of fixed images.

| Phase | Goal | Risk retired | Version | Checkpoint |
|---|---|---|---|---|
| 0 | Foundation | Does the data client run on the deploy runtime at all? | `0.1.0` | `npm run build && npm run preview`, then `/api/health` → 200 |
| 1 | Schema + policies | Can an anonymous session see exactly the published rows and nothing else? | `0.2.0` | Policy probe under a simulated anon session against a seeded table |
| 2 | Public site | Does the whole read path work end to end? | `0.3.0` | Every route renders under `preview`; empty states verified in a private window |
| 3 | Admin CMS | Can an editor save, and does a failed save say something actionable? | `0.4.0` | Create/edit/publish one row of every type; gate refuses signed-out **and** admits signed-in |
| 4 | Launch readiness | Anything only measurable on the finished site | `0.5.0` | Accessibility pass at the stated target; metadata; real-device check |
| 5 | Content + launch | Is anything on the live site not from the client? | `1.0.0` | Every placeholder replaced or still `published = false`; deploy smoke-tested at `/api/health` |

## Worked example B — bilingual CMS, media library, blog, forms, galleries

Modules: `i18n`, `media`, `blog`, `forms`, `gallery`. Fifteen-odd tables, several editors.

| Phase | Goal | Risk retired | Version | Checkpoint |
|---|---|---|---|---|
| 0 | Foundation | Does the cookie-session adapter work on the deploy runtime? | `0.1.0` | `/api/health` → 200 with `database`, `auth`, `i18n` named and passing |
| 1 | Schema + locale core | Both are depended on by everything after; a fix here later is a migration everywhere | `0.2.0` | Policy probe; `t()` exercised against a deliberately half-translated row |
| 2 | Public core | The layout, chrome and one full content path | `0.3.0` | Both languages rendered; font network panel checked against the gating claim |
| 3 | Remaining public pages | Repeat of a proven pattern | `0.4.0` | Every route in both languages; galleries with JS disabled |
| 4 | Admin CMS + media library | The write path, and the picker without which the media FKs are unusable | `0.5.0` | Save every type; upload, reuse an existing image, and check the recorded byte size |
| 5 | Launch readiness | Accessibility, metadata, performance | `0.6.0` | Accessibility pass; egress arithmetic re-derived against real image sizes |
| 6 | Content + launch | Real content replaces every placeholder | `1.0.0` | Nothing invented is published; deploy smoke-tested |
| 7 | Deferred | Named now so it is tracked, not discovered | — | Data-layer caching; whatever the build named as untested |

Note phase 7. **Name the deferred work in the plan.** An admitted gap gets closed; an unexamined
one does not.

---

## Per-phase verification, written into the plan

No phase closes on "it renders". Each one closes with:

1. `npm run build && npm run preview` — the production runtime, not the dev server.
2. `curl http://127.0.0.1:8787/api/health` → 200, all checks named.
3. Every language exercised, including a deliberately half-translated row (if `i18n` is on).
4. A private window used for anything about what the public sees — a signed-in editor sees
   unpublished drafts, so "it looks fine to me" is not evidence.
5. The verification table in `status.md` filled in, **including what was not checked**.
6. `CHANGELOG.md` updated, version bumped, `status.md` header refreshed, `lessons.md` appended.
