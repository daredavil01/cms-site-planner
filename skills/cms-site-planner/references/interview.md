# Interview

Ask with `AskUserQuestion`, batching related questions. Every question below has a **default**,
so an unanswered interview still yields a usable plan — but a default that changes the plan
must be stated in the summary and written into `status.md` as an assumption.

> **Ambiguity is a question, not a coin flip.** Where a brief is silent and the answer changes
> the phases, the tables or the module list, ask. A guess made here is invisible by the time it
> is wrong.

## 0. Read what exists first

Read any supplied brief first. Skip every question the brief already answers, and say which ones
you skipped.

- **Requirement documents** stay the source of truth. Do not restate them in the plan; cite their
  sections from each phase ("Requirements §4.1") and record the precedence: requirements win over a
  design, and the plan wins over neither.
- **A design export or prototype** (HTML screens, a design-tool export): list every screen and map
  it to the phase that builds it; list the screens and controls that are prototype-only and never
  built (screen viewers, state switchers, "viewing as" role pickers); and list every **sample value**
  it shows — phone, domain, hours, prices, registration numbers, names, statistics — as never to ship
  (`rules-core.md` §14a).
- **Conflicts between the design and the requirements** become numbered decisions (D1, D2, …), each
  with the default that applies until someone decides and the phase that needs the answer. They go
  into the plan and into `status.md`.

## 1. Organisation and audience

- What does the organisation do, and what is the site for?
- Who visits, and from where?
- **What devices and what network?** — this is a constraint, not colour. A phone-first audience
  on patchy connectivity sets the image budget, the JS budget and the touch-target minimum.
- **Is its advertising regulated?** Medical, legal and financial professions and charities often
  work under a code that bans claims, superlatives or testimonials. If so, name the code and its
  banned claims — they become a test (`rules-core.md` §14b) — and ask which services need neutral
  wording everywhere.

*Default:* mobile-first, WCAG 2.1 AA, `min-height: 44px` on interactive controls, no advertising
code.

## 2. Stack → `supabase` or `cloudflare`

Do not ask "which stack?". Ask the questions in `stacks.md` §2 that decide it:

- **Budget:** must it run at zero cost, with commercial use allowed? Will traffic be quiet for days at
  a time?
- **Public sign-in:** do members of the public have accounts?
- **Freshness:** must an edit be live on save, or is a publish step of a few minutes acceptable?
- **Per-visitor pages:** does any public page vary by visitor?

*Default:* `supabase`, unless the budget is free and traffic low or bursty — then `cloudflare`.
State the choice, and the answer that decided it, in the summary and in `status.md`.

Also ask, for either stack: **which Cloudflare account** will hold the site, and is another account
logged in on the machine that will deploy it? The answer goes into `deployment.md`.

## 3. Language → module `i18n`

- How many locales? Which ones, as BCP-47 tags?
- **Which is primary?** The primary audience's language is the default, not the translation.
- Who translates, and who reviews a translation before it publishes? Can some languages launch
  later than others?
- On `cloudflare`: should every language be indexed by search engines (path prefixes), or is one URL
  per page with a browser swap acceptable (`rules-i18n.md` §2a)?

*Default:* one locale, module off. One locale means plain `text` columns and no translation
layer anywhere — do not plan one "in case".

If two or more: module on. If **three or more**, say so in the plan — read the N≥3 caveat at
the end of `rules-i18n.md` and carry it into `status.md` as a known-unproven area.

## 4. Content types → the table list

What does the site publish? Walk the likely list rather than asking an open question: pages,
programmes or services, team or people, events, news or blog posts, photo galleries, documents
or downloads, testimonials, partners, FAQs, prices.

For each: **who writes it, how often, and does it need to be unpublished before it is finished?**
Anything with a draft state gets a `published` flag or a draft copy (`schema.md` §8).

**Which values are facts only the client can confirm?** Registration numbers, prices, opening
hours, contact details, statistics. Each unconfirmed one is `[TBD]` and a numbered open question
(§9).

## 5. Editing → module `roles`

- Who edits, and how many of them?
- Do different people need different access, or is everyone a full editor?
- **Does anything need an owner's approval before it publishes** — prices, legal facts, registration
  numbers?

*Default:* a single admin role, module off. Multi-role is an access-control surface, not a UI
feature — turning it on means every access rule gains a role predicate, so do not turn it on
speculatively.

## 6. Personal data → module `privacy`

- What does the site store about people beyond a contact message? Health, children, finances, legal
  matters, bookings tied to a name?
- Which data-protection law applies, and what retention period does it — or the client — require?
- Which staff may see which personal data?

*Default:* module off when only a contact form exists. On whenever the answers include sensitive
categories or a retention period; then `forms` is on too.

## 7. Modules

Ask these as one multi-select, with the schema consequence named in each option:

| Key | Turn on when | Consequence |
|---|---|---|
| `i18n` | two or more locales | translatable fields, one `t()` helper, `:lang()` typography, a self-hosted subset font per script; see `rules-i18n.md` |
| `media` | images or documents are reused across pages, or an editor uploads them | one `media` table, entities reference it by ID; see `rules-media.md` |
| `blog` | dated posts with an archive | `posts` table, published state, an index route with pagination |
| `forms` | the public submits anything | insert-only public writes, the five defences, idempotency; see `rules-forms.md` |
| `gallery` | photo grids | `gallery_items`, ordering, a lightbox that costs no JS |
| `roles` | more than one kind of editor | a role table, a role-matrix test, approval as data, an audit log; see `rules-roles.md` |
| `privacy` | sensitive personal data, or a data-protection regime | field encryption, retention jobs, view logging, an exposure checklist; see `rules-privacy.md` |
| `booking` | anyone reserves a time that must not be double-booked | slot locks under a unique index, an engine phase before any screen, private manage links; see `rules-booking.md` |
| `notifications` | the site sends email or prepares messages | own-domain senders, a message log, claimed scheduled sends, signed webhooks; see `rules-notifications.md` |

`booking` implies `forms` and `privacy`. Say so if the user picks it alone.

## 8. Constraints

- Launch date, and anything immovable behind it.
- **Hosting and database accounts, domain, DNS, bot-check keys, staff sign-in — provisioned already,
  or not?** Each unprovisioned item is a setup item in `status.md`, recorded as *confirmed by the
  owner* or *verified here* — the two are not the same.
- **Deploy cadence.** Deploy each phase to a preview environment, or build everything locally and
  deploy once at the end? The second moves launch to the last phase and makes `deployment.md` the
  only record of what production will need (`phases.md`).
- **Session size.** How big is one working session? Phases are sized to fit one.
- **Budget tier.** On a free or low tier, write down which limit binds — see `stacks.md` §4. If the
  `media` module is on, this sets a per-image ceiling — see `rules-media.md` §4.
- Accessibility target. *Default:* WCAG 2.1 AA.
- Analytics, search, maps, chat or anything third-party that would add a script to the public pages.

## 9. Content readiness and open questions

**What real copy and imagery exists today?**

Everything absent starts unpublished. Say so explicitly in the plan, because the alternative — a
plausible placeholder that ships — is the failure this rule exists to prevent. Invented impact
figures, invented testimonials attributed to a named person, and invented account or
tax-registration numbers are the three that do real damage. Build the empty state instead, and make
the final phase the one where the client's real content lands.

**Number every unanswered question** (Q1, Q2, …) and record which phase's final values it blocks.
Unanswered questions block launch values, not code: phases proceed on `[TBD]`, `NULL` limits and
empty states, and the release check refuses to launch until the answers land.
