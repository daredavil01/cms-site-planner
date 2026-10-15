# Deriving phases

Phase count and boundaries come from the project's scope. They are **not** a fixed template — but
derivation is not improvisation either, and the constraints below make the difference.

---

## The constraints

1. **The first phase is always the foundation**, and it ends at a green `/api/health` under the
   production runtime, with lint, typecheck, a test runner and CI wired up. Not a page. Its purpose
   is to answer the riskiest question in the stack while the answer is still cheap
   (`rules-core.md` §1–3).
2. **Launch is at `1.0.0`**, and it is where the client's real content lands. Everything before it
   runs on empty states, unpublished drafts and `[TBD]` markers. It is the last phase unless the
   plan has planned post-launch phases (see Versioning).
3. **Every phase carries**: a version; a one-line **risk rationale** saying why *this* boundary
   exists; the **requirement sections and design screens** it builds from; the **tests that must
   exist** when it closes (`rules-core.md` §7); a **runnable checkpoint** — a command whose output
   settles whether the phase is done; and the **open questions** (Q-numbers) that block its final
   values. A phase with no rationale is a list of tasks with a heading; a phase with no checkpoint
   completes on "it renders".
4. **Emit the phase → version table explicitly**, in both `implementation_plan.md` and the
   generated `CLAUDE.md` §6. Because phase count varies per project, no mapping can be assumed —
   a reader who has to infer it will infer it wrongly.
5. **Size every phase for one working session**, and say what a session is. A phase that does not
   finish is marked in progress with what is left, and finished first next session. Anything
   discovered outside the phase's scope goes into the notes column of `status.md`, not into the
   phase.

**Owner setup is not a phase.** Accounts, domain, DNS, keys and the client's first answers are a
setup checklist in `status.md`, each item marked *confirmed by the owner* or *verified here*.
*Evidence:* a setup checklist that kept the two apart showed the staff sign-in application as
"confirmed, not verified" through the whole build — and at the first deploy its configured team
domain turned out to be mistyped (`rules-cloudflare.md` §6).

## Versioning

Minor per phase, patch for everything else, `1.0.0` at launch:

| Phase | Version |
|---|---|
| 0 — foundation | `0.1.0` |
| 1 … n−1 | `0.2.0` … |
| n — launch | `1.0.0` |
| planned after launch | `1.1.0` … |

This is phase-based, not semver-for-APIs — a website has no downstream consumers, so "breaking
change" has no useful meaning here.

**Planned post-launch phases are versioned; unscoped ideas are not.** When the brief splits an MVP
from later features that are already specified, the later features are real phases at `1.1.0`,
`1.2.0`, … with the same five things as any phase. Work that is named but not yet scoped lives under
a deferred heading, unversioned.

**When the owner builds everything before deploying.** If the interview chose "build locally, deploy
once", launch moves to the end, after every planned phase, and the versions follow. A phase is then
done when it runs locally with lint, typecheck, tests and build passing; everything production will
need — migrations, bindings, variables, secrets, outside services — is written into `deployment.md`
in the phase that introduces it. **Pushing is not deploying**: every phase is still committed and
pushed, or CI never runs (`rules-core.md` §6, §17).

## How to cut the boundaries

Order by **risk retired**, not by how the site is organised. Each boundary should be a point you
would be willing to roll back to.

- **Foundation first** — runtime, env schema, request-scoped client or bindings, health endpoint,
  test runner, CI, a `.gitattributes` with `* text=auto eol=lf`. Check the toolchain's peer ranges
  before taking the newest majors (`rules-core.md` §6).
- **Schema before anything consumes it.** If the `i18n` module is on, the locale machinery belongs
  in the same phase as the schema, because every later phase depends on both. Ship a deliberately
  half-translated row as a fixture. On `cloudflare`, the export allow-list and its "no personal
  table" test belong here too.
- **The riskiest public surface next**, then the remainder. Split public pages into two phases only
  when the first one proves a pattern the second repeats.
- **Admin foundation before admin screens**: the gate, the role table and its matrix test, and the
  audit log, before any screen that edits content.
- **Admin after the public read paths work**, so the write path is built against a schema already
  known to render.
- **Data protection lands with the first public-write surface.** If `privacy` is on, the first form
  that stores personal data brings field encryption, retention, view logging and the test that the
  public side cannot read personal tables with it — not a later hardening phase.
- **Engine before screens.** A mechanism whose failure is invisible to one person clicking — double
  booking, a publish that must be atomic, a payment — gets its own phase with no screens, ending at
  the test that proves it (`rules-booking.md` §1). The visitor's screens and the staff screens come
  after, as separate phases.
- **Do not defer the UI half of a data model past the phase that creates it.** A schema whose
  screen never arrived is a schema nothing can use (`rules-media.md` §1).
- **A launch-readiness phase before launch**, holding the checks that are only meaningful against
  the finished site: accessibility pass, metadata, performance on the phone profile, a real-device
  check on a low-end phone, the release-check script, email DNS, backups and a tested restore.
- **A quality-pass phase at the end of the build** (before launch, or as the last planned phase):
  regression across every feature and every language, a security and privacy review against the
  written exposure rules (`rules-privacy.md` §1), and the documents staff will need — a user guide
  and an operations runbook. *Evidence:* the review in a fully tested build found five problems no
  test had, all in configuration, logging and copy.

### What forces an extra phase

A module with its own risk: `media` (upload path, egress budget), `roles` (a predicate in every
access rule), `forms` (public-write surfaces), `privacy` (encryption and retention), `booking`
(always three: engine, visitor screens, staff screens), `notifications` (scheduled sends and their
duplicates). Two or three of the lighter modules usually mean one extra phase, not three.

### What does not

Page count. Ten static pages that share one layout are one phase, not five.

---

## Worked example A — monolingual brochure site, one editor (`supabase`)

Modules: none. Six content types, one admin, no uploads beyond a handful of fixed images.

| Phase | Goal | Risk retired | Version | Checkpoint |
|---|---|---|---|---|
| 0 | Foundation | Does the data client run on the deploy runtime at all? | `0.1.0` | `npm run build && npm run preview`, then `/api/health` → 200 |
| 1 | Schema + policies | Can an anonymous session see exactly the published rows and nothing else? | `0.2.0` | Policy probe under a simulated anon session against a seeded table |
| 2 | Public site | Does the whole read path work end to end? | `0.3.0` | Every route renders under `preview`; empty states verified in a private window |
| 3 | Admin CMS | Can an editor save, and does a failed save say something actionable? | `0.4.0` | Create/edit/publish one row of every type; gate refuses signed-out **and** admits signed-in |
| 4 | Launch readiness | Anything only measurable on the finished site | `0.5.0` | Accessibility pass at the stated target; metadata; real-device check |
| 5 | Content + launch | Is anything on the live site not from the client? | `1.0.0` | Every placeholder replaced or still unpublished; deploy smoke-tested at `/api/health` |

## Worked example B — bilingual CMS, media library, blog, forms, galleries (`supabase`)

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

## Worked example C — multilingual booking site with an MVP and later features (`cloudflare`)

Modules: `i18n` (four locales), `media`, `forms`, `roles`, `privacy`, `booking`, `notifications`,
`blog`. An MVP launch, then planned features. Tests are named per phase; only the checkpoint is
shown here.

| Phase | Goal | Risk retired | Version | Checkpoint |
|---|---|---|---|---|
| 0 | Foundation and layout | Does the static build plus a Worker run under `wrangler dev`, with tests and CI? | `0.1.0` | `/api/health` → 200; render test checks every header and footer link |
| 1 | Schema and seed | Draft/published readers, atomic publish, an export that cannot see personal tables | `0.2.0` | Tests: publish all-or-nothing; allow-list contains no personal table |
| 2 | Public pages and SEO | The whole read path from exported content | `0.3.0` | Dist tests: titles, canonicals, JSON-LD, banned words, sitemap, every link has an `href` |
| 3 | Languages | Every string switches, and missing text falls back visibly | `0.4.0` | Pseudo-translation pass finds no unswitched text |
| 4 | Admin foundation | The gate verifies the token itself; roles; audit | `0.5.0` | Tests: bad signature, audience and expiry refused; role matrix over every route |
| 5 | Content and publishing | The write path and the publish rebuild | `0.6.0` | An edit appears on the built site after Publish |
| 6 | Enquiry form and data protection | The first stored personal data, encrypted, retained, logged | `0.7.0` | Tests: no plain text stored; staff-only inbox; retention deletes only expired rows |
| 7 | Booking engine | Double booking | `0.8.0` | Concurrent overlapping requests → exactly one success |
| 8 | Visitor booking screens | Book, manage, reschedule, cancel on a phone | `0.9.0` | Full flow at 375 px under `preview`; manage-page lockout test |
| 9 | Staff booking screens | Staff can run a day from the admin | `0.10.0` | Every status transition; illegal ones refused |
| 10 | Launch readiness and launch | Release checks, backups, email DNS, production | `1.0.0` | Release check passes against production with no `[TBD]` |
| 11–15 | Planned features (articles, documents, templates and message log, scheduled reminders, reports) | One risk each | `1.1.0`–`1.5.0` | Each with its own runnable checkpoint |
| 16 | Quality pass and handover | Security and privacy review, regression in every language, staff guide | `1.6.0` | Review recorded with each rule's enforcement; every finding fixed or owned |

If the owner chooses to build everything before deploying, phase 10 keeps its checks but loses its
launch, and launch becomes a final step after phase 16.

---

## Per-phase verification, written into the plan

No phase closes on "it renders". Each one closes with:

1. `npm run lint`, `npm run typecheck`, `npm test` and `npm run build` passing — the CI steps, in
   CI's order — and the phase's named tests present.
2. `npm run preview` — the production runtime, not the dev server — and
   `curl http://127.0.0.1:8787/api/health` → 200, all checks named.
3. Every language exercised, including a deliberately half-translated row (if `i18n` is on).
4. A private window used for anything about what the public sees — a signed-in editor may see
   unpublished drafts, so "it looks fine to me" is not evidence.
5. Every new screen at 375 px, document width equal to viewport width (`rules-core.md` §15), and as
   every role that can open it (if `roles` is on).
6. The verification table in `status.md` filled in, **including what was not checked**.
7. `deployment.md` updated if the phase added a migration, binding, variable, secret or outside
   service.
8. `CHANGELOG.md` updated, version bumped, `status.md` header refreshed, `lessons.md` appended if
   anything surprised you, and the phase committed **on its own** and pushed.
