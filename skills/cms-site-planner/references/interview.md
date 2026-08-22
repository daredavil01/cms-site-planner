# Interview

Ask with `AskUserQuestion`, batching related questions. Every question below has a **default**,
so an unanswered interview still yields a usable plan — but a default that changes the plan
must be stated in the summary and written into `status.md` as an assumption.

> **Ambiguity is a question, not a coin flip.** Where a brief is silent and the answer changes
> the phases, the tables or the module list, ask. A guess made here is invisible by the time it
> is wrong.

Read any supplied brief first. Skip every question the brief already answers, and say which
ones you skipped.

## 1. Organisation and audience

- What does the organisation do, and what is the site for?
- Who visits, and from where?
- **What devices and what network?** — this is a constraint, not colour. A phone-first audience
  on patchy connectivity sets the image budget, the JS budget and the touch-target minimum.

*Default:* mobile-first, WCAG 2.1 AA, `min-height: 44px` on interactive controls.

## 2. Language → module `i18n`

- How many locales? Which ones, as BCP-47 tags?
- **Which is primary?** The primary audience's language is the default, not the translation.

*Default:* one locale, module off. One locale means plain `text` columns and no translation
layer anywhere — do not plan one "in case".

If two or more: module on. If **three or more**, say so in the plan — read the N≥3 caveat at
the end of `rules-i18n.md` and carry it into `status.md` as a known-unproven area.

## 3. Content types → the table list

What does the site publish? Walk the likely list rather than asking an open question: pages,
programmes or services, team or people, events, news or blog posts, photo galleries, documents
or downloads, testimonials, partners, FAQs.

For each: **who writes it, how often, and does it need to be unpublished before it is finished?**
Anything with a draft state gets a `published` boolean and a policy that reads it.

## 4. Editing → module `roles`

- Who edits, and how many of them?
- Do different people need different access, or is everyone a full editor?

*Default:* a single admin role, module off. Multi-role is a policy surface, not a UI feature —
turning it on means every policy gains a role predicate, so do not turn it on speculatively.

## 5. Modules

Ask these as one multi-select, with the schema consequence named in each option:

| Key | Turn on when | Consequence |
|---|---|---|
| `i18n` | two or more locales | JSONB translatable columns, one `t()` helper, `:lang()` typography, a self-hosted subset font per script |
| `media` | images or documents are reused across pages, or an editor uploads them | one `media` table, one bucket, entities reference it by FK; see `rules-media.md` |
| `blog` | dated posts with an archive | `posts` table, published state, an index route with pagination |
| `forms` | the public submits anything | insert-only anon policies, no anon `select`; a `Result` write path |
| `gallery` | photo grids | `gallery_items`, ordering, a lightbox that costs no JS |
| `roles` | more than one kind of editor | a role column plus a role predicate in every staff policy |

## 6. Constraints

- Launch date, and anything immovable behind it.
- **Hosting and database accounts — provisioned already, or not?** An unprovisioned account is a
  phase-0 blocker, and it belongs in `status.md` under open items.
- **Budget tier.** On a free or low tier the binding limit is usually egress, not storage. If
  the `media` module is on, this sets a per-image ceiling — see `rules-media.md`.
- Accessibility target. *Default:* WCAG 2.1 AA.
- Analytics, search, or anything third-party that would add a script to the public pages.

## 7. Content readiness

**What real copy and imagery exists today?**

Everything absent starts `published = false`. Say so explicitly in the plan, because the
alternative — a plausible placeholder that ships — is the failure this rule exists to prevent.
Invented impact figures, invented testimonials attributed to a named person, and invented
account or tax-registration numbers are the three that do real damage. Build the empty state
instead, and make the final phase the one where the client's real content lands.
