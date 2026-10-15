# Core rules

Always applicable, whatever stack and modules were selected. Copy these into the generated
`CLAUDE.md`, together with the rules file for the chosen stack (`rules-supabase.md` or
`rules-cloudflare.md`), keeping the evidence — **a rule stated without the failure it prevents gets
argued away by the next person who finds it inconvenient**, usually correctly, because a rule with
no cost attached looks like superstition.

Format: **Rule** · *Why* · *Evidence*. Where a rule's mechanics differ by stack, the stack file
carries them and this file says so.

---

## 1. Verify on the deploy runtime, not the dev server

**Rule.** A change to middleware, auth, cookies, headers or any server endpoint is not verified until
it has run under the production runtime.

```bash
npm run build && npm run preview
```

Then hit `http://127.0.0.1:8787/api/health`.

*Why:* the dev server runs on Node; production runs on workerd. Different JavaScript runtimes. Node
tests are a third environment, closer to the dev server than to production.

*Evidence:* three failures, each invisible until workerd or a browser ran the code.

- `@supabase/ssr` threw `cookies.getAll is not a function` on workerd — on Node that code path never
  ran.
- A Worker's main module exported a constant beside its handlers. Every Node test passed; `wrangler
  dev` refused to start with "Incorrect type for map entry". workerd accepts only handlers from the
  main module — a test now asserts `index.ts` exports `default` and nothing else.
- `Referrer-Policy: no-referrer` makes browsers send `Origin: null` on same-origin form posts, so a
  page refused its own forms. Node tests never send `Origin` or `Sec-Fetch-Site`; only a browser
  found it (`rules-forms.md` §5).

**Never rebuild `dist/` under a running preview.** `wrangler dev` reloads mid-build and then answers
404 — or 500 from a Worker that can no longer fetch its own shell page. Stop, build, start again.

> On Windows the dev/preview process holds a lock on `dist/`. Stop it before rebuilding, or the
> build fails with `EPERM`.

## 2. Ship a health endpoint before any page

**Rule.** Build `/api/health` first. It exercises the riskiest integration end to end, returns
`200`/`503` with **named** per-check results, and sets `cache-control: no-store`.

*Why:* every stack has one unknown that decides whether the plan works at all. Find out on day
one, when the answer is cheap. It also gives every later deploy one URL to smoke-test, and every
script that starts a preview one URL to poll before it does anything else.

*Evidence:* discovering the workerd/cookie-adapter incompatibility in rule 1 costs an afternoon
before any pages exist, and a rewrite after five pages do. Separately, a preview launcher returned
before `wrangler dev` was listening (the build runs first), so the first cron trigger sent to it did
nothing; polling `/api/health` first fixed it.

**Never identify a state by string-matching a library's error message.** Matching a literal like
`'Auth session missing!'` is a silent breakage waiting for a minor version bump.

## 3. Runtime compatibility flags fail at request time, not build time

**Rule.** Treat `nodejs_compat` and friends as a deployment dependency and verify them with the
health endpoint, not with a successful build.

*Evidence:* the database client reaches for Node built-ins (`crypto`, `stream`, `buffer`).
Without the flag the build is green and the Worker cannot start — a build-green, request-red
failure, confusing precisely because the usual signal was positive.

## 4. Access control lives in one enforced place, and its tests probe that place

**Rule.** Decide where access is enforced — on `supabase`, Row-Level Security; on `cloudflare`, the
Worker's role guard plus the build's export allow-list — and enforce it **only** there. Do not add
a second, softer copy of the same check in a query helper, and test the enforcing layer directly.
The mechanics are in the stack file.

*Why:* a redundant filter hides the case where the real control is wrong. The app looks correct
while the data leaks.

*Evidence:* an empty table makes an access test pass for the wrong reason — a `select` returning no
rows proves nothing about whether anything would have been filtered. Seed a row that must be seen
and one that must not, and assert exactly the first. The same shape on the other stack: a role guard
is tested by a **role matrix** — every route for every role — because a guard that was only ever
tested for the role that may enter has never been tested at all.

## 5. Secrets live in the platform's secret store, and nothing that ships can reach them

**Rule.** A secret is set with the platform's secret command and declared server-only in the typed
environment, so a client import is a build error. Never put one in `vars`, a committed file, or the
terminal history.

```bash
# Values arrive as JSON on stdin: nothing lands in a file or the shell history.
wrangler secret bulk --env production < /dev/stdin
```

Local values live in `.dev.vars`, which is gitignored, and hold made-up values only.

*Evidence:* three local-environment traps, each of which cost a session.

- **`wrangler types` reads `.dev.vars`.** Local-only variables became required members of the
  committed `Env` type and broke the tests' environment object. Move `.dev.vars` aside while
  regenerating types; declare local-only names in a hand-written type instead.
- **A key twice in `.dev.vars` uses the last one.** A real bot-check secret added below the
  always-pass test secret made every local form fail. `wrangler dev --var KEY:VALUE` overrides one
  value for one run without editing the file.
- On `supabase`, the privileged credential has its own four-layer rule — `rules-supabase.md` §2.

## 6. CI checks; it does not deploy

**Rule.** CI runs lint → typecheck → build → test → scan the build output. Deployment stays a
platform concern.

*Evidence:*

- **A guard that cannot fire is worse than no guard**, because it looks like coverage. A credential
  scan that matched only the provider's legacy key format was silently useless against the newer
  one; a bare-prefix match failed every build because the client library ships the prefix in its
  own code. Match every format the provider issues, and require entropy after a known prefix. The
  `supabase` patterns are in `rules-supabase.md` §3.
- **Do not declare a script you have not installed.** Either wire up lint and format in the first
  phase or delete the lines; a project that declared both and installed neither left everyone
  reading a guardrail that could not fire.
- **CI that never runs checks nothing.** One build configured CI in its first phase and pushed
  nothing until the last, so CI never ran once while every phase reported lint, typecheck and tests
  green — locally. Push every phase (rule 17), even when the
  owner has decided to deploy only at the end.
- **Pin the toolchain to what its plugins support.** The newest TypeScript major was outside the
  peer range of both the ESLint parser and the Astro checker, and npm resolved an ancient
  `eslint-plugin-astro` under the newest ESLint without an error. Check peer ranges before taking a
  major.

Store even non-secret build values in repository secrets rather than the workflow file — not
because they are sensitive, but so they can be rotated without a commit.

## 7. The rules that protect people have tests

**Rule.** Every rule that protects personal data, prevents a double booking or double charge, or
enforces a role has an automated test, written in the phase that creates the rule. The test suite
exists from phase 0; each phase in the plan names the tests that must exist when it closes.

*Why:* those are the rules whose failure nobody sees. A wrong role check renders a page that looks
right to whoever is allowed to see it; a race condition never shows up for one person clicking.

*Evidence:* the first two sites on the `supabase` stack had no test suite, and `/api/health` plus
manual verification tables stood in for one. On a later build that had a suite, the record of
what it caught that nothing else did is the argument:

- two simultaneous requests for the same resource both passing the availability check, caught by a
  `Promise.all` race test (`rules-booking.md` §4);
- a list filter that never matched, because a computed SQL column had no type affinity — "a test
  caught it; the browser check did not";
- an admin list view that answered 500 when it had nothing to audit, caught only by the role-matrix
  test;
- a staff form that, submitted twice, reported its own new record as a conflict.

## 8. Auth fails closed

**Rule.** Gate the staff area server-side, before any HTML is produced, on every request. Wrap the
check in `try/catch` and treat any failure as "signed out". The mechanics — which call to trust,
which token to verify — are in the stack file.

*Why:* an unhandled throw becomes a 500, and a 500 on an auth check is the kind of thing that gets
"fixed" by removing the check.

*Evidence:* a gate is **two** behaviours — refusing the unauthenticated and admitting the
authenticated — and a check that only ever exercised the first shipped as half a gate. Verify both.
On the `cloudflare` stack the same lesson came back from production configuration: a mistyped
Access team domain served no signing keys, so the admin refused everyone — a failure no local test
could catch, because localhost uses a stand-in (`rules-cloudflare.md` §6).

Any role shown in the UI is **hinting only**. The enforcing layer from rule 4 decides on every
request.

## 9. Reads degrade; writes return a Result

**Rule.** Query helpers return `null` / `[]` on error rather than throwing, so one missing row
degrades to an empty section instead of a 500 on the whole page. Writes return a discriminated
union.

```ts
export type WriteResult = { ok: true } | { ok: false; error: 'invalid' | 'failed' };
```

*Why:* a failed read should cost one section, not the page. A failed write is an outcome the UI
is expected to handle, and `throw` is the wrong channel for it.

**Isolate partial failures.** When a section aggregates N independent things, let each fail
independently.

*Evidence:* six homepage counters, six independent queries — one failing table drops its own
number and leaves the other five, rather than emptying the section. The blast radius of a failure
should match its actual scope.

Three more house rules, written at the top of the data-access layer where they will be read:
order by `sort_order` everywhere (the admin's drag-reorder is meaningless otherwise); every
helper takes the **request-scoped** client or binding as its first parameter, never a module-level
singleton, so one request's auth context cannot leak into another; and sort embedded child rows
after the round trip, because the API does not guarantee their order.

**Type the insert payload** so a schema change breaks the build rather than the insert — and
note what the payload omits: a status column is never set from the client, so a submission
cannot arrive pre-marked as handled.

**Validate JSON columns; do not trust them.** Free-form JSON written by an admin UI is user input
with extra steps. Narrow it with an assertion only when runtime shape validation immediately
follows, and use `flatMap` so a malformed entry drops itself rather than the page.

## 10. Build-time values are not runtime values

**Rule.** No `vars` block in the Wrangler config for `PUBLIC_*` values.

*Why:* they are inlined at build time.

*Evidence:* setting them as runtime vars **half-works** — the server renders fine and the browser
is handed `undefined`. That is the worst of both outcomes, because the page looks like it is
working to whoever is checking it.

Use `||`, not `??`, for config that CI may supply as an empty string — `??` passes `''` through.

**Generated Worker types are a build artefact with opinions.** Regenerate them (`wrangler types`)
after every change to the Wrangler config, and know two things they do:

- they write **literal** types for vars, so a var holding `"[TBD]"` makes every test value a type
  error — generate with `--strict-vars=false`;
- they declare runtime globals that clash with the DOM library Astro uses. Keep Worker code in its
  own folder with its own `tsconfig.json`, exclude it from the main one, and typecheck both.

## 11. JS only where it earns it

**Rule.** Measure before shipping a framework island. Before reaching for one, check whether a
plain link, a `<details>` disclosure, a `:target` or `:has()` selector or a form submission does the
job — those cost zero bytes, work with JS off, and come with keyboard and screen-reader behaviour
already correct.

*Evidence:* **184 KB of framework runtime to animate six numbers**, replaced by a 602-byte inline
script; **181 KB** for a contact form, replaced by a plain `POST`-to-self. Quantify like this:
"184 KB to animate six numbers" ends a discussion, "React felt heavy" starts one.

The same rule held for a whole admin, built as server-rendered JSX with plain HTML forms and no
client framework. List filters are GET forms, so their state is in the URL by construction. Menus
are `<details>`; a star rating is radios plus `:has()`; the only client code is small scripts for
theme, clipboard and upload optimising. Where a design names a client framework
for the admin, record the deviation as a decision and add an island only for the screen that
needs one.

**Public islands are the exception.** The public site is HTML. Labels, translations, dates and
derived numbers are computed server-side or at build; an island receives finished strings and
numbers as props. Never import a browser database client from a server template — it drags the
client bundle into pages that otherwise ship no JS, and it does so silently.

In `.tsx`, React uses `className`, not `class`.

## 12. Migration discipline

**Rule.** Migrations are numbered, applied once, and never edited after they have run anywhere.
To change something, write the next file. The apply commands, and how to find out what a remote
database has really applied, are in the stack file.

*Why:* a migration that was edited after it ran describes a database that does not exist.

*Evidence:* on `supabase`, history drifted twice in one project despite a written rule, because
nothing verified it (`rules-supabase.md` §6). On `cloudflare`, a restore that ran tables in creation
order failed against a real database while its unit test passed, because migrations that rebuild a
table move it after its children (`rules-cloudflare.md` §7).

**A seed runs after every migration**, so a migration that drops or renames a column changes the
seed in the same session — and seed rows added after a database was seeded never reach it unless
loaded by hand.

**Line endings are content.** Migrations and seeds that carry multi-line text (a consent notice, a
template) must stay LF and UTF-8 **without a BOM**. A checkout under `core.autocrlf=true` rewrote
SQL files to CRLF, and a test comparing multi-line seed text with the text the app derives from it
failed. `.gitattributes` with `* text=auto eol=lf` from phase 0 prevents it; rule 22 covers scripted
edits.

**A role granted through a dashboard is a role the migrations do not know about.** Grant roles
in a migration, or the next fresh project is missing them.

## 12a. Storage columns hold keys, never URLs

Applies to **any** project with a storage bucket, whether or not the `media` module is on.

**Rule.** A storage column stores a path or key. A URL is produced only by one helper, and **private
buckets stay out of that helper's type union.**

```ts
export type PublicBucket = 'site' | 'resources';
//  `files` is private and deliberately absent — it needs a signed URL or a Worker route.

export function publicUrl(bucket: PublicBucket, path: string | null): string | null
```

*Why:* URLs in the database calcify a hostname and a scheme into your content, and one helper is
the single place to change when the CDN does.

*Evidence:* with the private bucket absent from the union, linking a private document from a
public page is a **type error** rather than something a reviewer has to notice. The type system is
doing security work, cheaply and permanently.

Have the helper return `null` for an empty path, so callers branch to a placeholder rather than
rendering a broken image.

**If this project serves any image from object storage, read `rules-media.md` §4 even when the
`media` module is off.** On `supabase` the egress arithmetic (`uploaded size × views = the bill`)
binds on a dozen fixed photographs too, and its failure mode is images quietly ceasing to load.

## 13. Errors: verbose inside the gate, opaque at the gate, absent from the logs

**Rule.** Surface the database's own error message to staff, on screen.

*Evidence:* `duplicate key value violates unique constraint programs_slug_key` is something an
editor can act on without help. "Something went wrong" is a support request.

Do the **exact opposite** on the login form. The raw auth error distinguishes "no such user"
from "wrong password", which is an account-enumeration hint. Show a generic failure there.

**Public endpoints answer with error codes and never echo what was submitted.** A release check
posts deliberately invalid details carrying a canary string and fails if the canary appears in any
answer.

**A log line is not the screen.** Log error names and codes, never `err.message` unfiltered. A
security review found an admin error handler logging `err.message`, and some messages quote their
input — a `JSON.parse` failure quotes the text it read, which can be a person's name. The handler
now drops quoted parts, and a test checks that a quoted name reaches neither the log nor the page.

## 14. Never publish invented content

**Rule.** Anything that did not come from the client stays unpublished until they sign off. Build
the empty state.

*Why:* **an empty state is safer than a plausible one.**

*Evidence:* three categories from a real prototype did the damage, and each fails differently.
Invented **impact figures** get quoted back to the organisation. An invented **testimonial
attributed to a named beneficiary** is a consent problem, not merely an accuracy one. Invented
**account or tax-registration numbers** read as a claim the organisation cannot back — and donors
rely on those for deductions, so a blank one must be suppressed rather than rendered empty.

Suppress a field that has no value rather than rendering it blank.

## 14a. Unconfirmed facts are `[TBD]`, and the release check refuses them

**Rule.** A fact the client has not confirmed — a phone number, fee, opening hour, registration
number, domain — is seeded as the literal marker `[TBD]`, never as a sample value from a design or
prototype. The release check scans the built output and API answers and **fails while any `[TBD]`
remains**. Number the open questions (Q1, Q2, …) and say which phase's final values each one blocks.

*Why:* an unanswered question should block launch values, not code. The marker lets every phase
proceed while making the gap impossible to ship.

*Evidence:* a design export carried a plausible phone number, domain, opening hours, registration
number, prices and people's names — every one a sample. Listing them in the plan as "never ship"
and seeding `[TBD]` instead held for the whole build, and the release check went from failing to
passing on the day the client's answers landed. Four mechanics that came out of it:

- **A `[TBD]` must never become a link.** `href={undefined}` rendered anchors with no target. An
  unconfirmed phone or email renders as text through a dynamic tag; a test checks every link has
  an `href`.
- **Unconfirmed numbers are `null`, not `[TBD]`**, and the feature that needs them answers
  "unavailable" until they are set. Tests set their own made-up values, and before filling an
  answer into the seed, grep the tests for it — tests that exercise the "missing" path depend on
  the value being unset.
- **Optional settings are seeded empty, not `[TBD]`.** A marker reaches the built output through
  translation dictionaries or structured data even where no page shows the setting, and fails the
  release check for a value nobody needs.
- **The client's answers go in a seed-side file, not a migration.** On a fresh database migrations
  run before the seed, so an `UPDATE` in a migration finds no rows and the seed then writes `[TBD]`
  back. An answers file runs after the seed and guards each `UPDATE` on the seed value, so it can
  rerun without undoing an editor's later change. A test runs it twice.

## 14b. Regulated wording is a test, not a review

Include only when the interview named an advertising or professional code (medical, legal,
financial, charity regulation).

**Rule.** The code's banned claims become a word list, and a test fails the build when the seed or
the built HTML contains one. Wording for sensitive services is neutral everywhere it can leak:
page titles, messages, calendar files, confirmation screens.

*Evidence:* a superlatives-and-guarantees word list also matched the client's own disclaimers —
"we do our best", "cannot be guaranteed". The copy was reworded and the change recorded next to the
seed; **loosening the rule to let a disclaimer through would have let the next claim through too.**
The same seed test checks title and description lengths, and caught an over-long description in the
phase that wrote it.

## 15. Design tokens, not inline styles; every screen checked at phone width

**Rule.** Colours, type and spacing are CSS custom properties. **No component hardcodes a hex
value.** Interactive controls are `min-height: 44px`; the target is WCAG 2.1 AA unless the
interview set a higher one. Dark mode is one token block (`:root[data-theme='dark']`), set before
first paint.

Mobile and accessibility are constraints, not polish. Do not shrink controls to make a layout
fit.

**Every new screen is checked at 375 px**: `document.documentElement.scrollWidth` must equal
`clientWidth`. *Evidence:* three separate horizontal overflows, each from a different cause — an
absolutely placed `.sr-only` cell escaping an unpositioned `overflow-x: auto` box (761 px), a
three-column grid of radio pills (40 px too wide), and a long URL in a `pre-wrap` message
(1,076 px). An element scan for `right > innerWidth` missed the URL, because the overflow was a text
run; the document width did not.

**Contrast is per theme.** A status chip passed in dark mode and measured 4.1:1 in light.

A visual prototype is a **read-only reference**.

*Evidence:* its inline styles, its CDN font `<link>`, its entrance animation, its bottom toolbar,
the typeface it loads but never uses, its own runtime script and its sample data are all prototype
decisions, and every one of them ships by default if nobody names them as such. A prototype's asset
loading is a prototype decision — copy the design, not the delivery.

## 16. Comments carry decisions, including the reversals

**Rule.** When you abandon the obvious approach, record it **next to the code that replaced it**
— what was tried, what it cost, why it is gone.

*Why:* a commit message is findable only if you already suspect there is something to find. A
comment is read by the next person to touch the line, which is exactly the person about to
re-make the decision.

*Evidence:* the artifact this rule produces looks like the comment in rule 11 — "184 KB of
framework runtime to animate six numbers, replaced by a 602-byte inline script." Quantified next
to the replacement, it ends the discussion. In a commit message, it is invisible to the person
about to reintroduce the island.

## 17. Commit and push at the boundary you would want to roll back to

**Rule.** Commit **each phase separately**, and push it.

*Evidence:* a deploy pipeline that triggers on push does nothing whatsoever for work that was
never pushed. One project shipped three phases as a single 51-file commit, and the deployed
artifact had no commit behind it — so it could be neither reproduced nor rolled back. A later
project repeated half of it: sessions that built several phases committed them together, and nothing
was pushed for the whole build, so CI never ran once (rule 6).

## 18. The deploy config is code: test it, and read what the deploy printed

**Rule.** Tests read the Wrangler config files as text and assert the values that must agree across
Workers and environments. After every deploy, read the variable list the deploy command printed —
not the file you think it deployed.

*Why:* tests construct their own environment, so they never see the real one.

*Evidence:* a security review at the end of a build found two configuration mistakes that had
been in the config since early phases, with every test green: a variable meant to hold the public
site's origin held the admin's, so every link in outgoing messages would have opened a sign-in page;
and the sender address was on a domain the email service cannot send from, so every alert would have
failed. A config test now imports both `wrangler.jsonc`
files with `?raw`, strips whole-line `//` comments (a `//` inside a URL must stay), and checks them.
Separately, the first preview deploy sent an old sender address while the file was being edited —
visible only in what the deploy printed.

**Check which account you are about to deploy to.** `npx wrangler whoami` before every deploy: one
machine held a personal Cloudflare account and the client's, and Wrangler picked the personal one
first. Name the client's account in the generated `CLAUDE.md` and in `deployment.md`.

## 19. Request logs are a data store

**Rule.** Turn off platform request logging that records URLs and headers, unless the interview
decided otherwise and the privacy notice says so.

```jsonc
"observability": { "enabled": true, "logs": { "invocation_logs": false } }
```

*Why:* a request log keeps whatever the request carried — a private-page token in the path, the
visitor's IP address, the staff sign-in token in a header.

*Evidence:* found by the security review in rule 18, not by any test: neither Worker had set
`observability`, so Workers Logs could keep every private-page token, every IP address (which the
design otherwise stored only as a salted hash) and the staff sign-in token. `observability` is
inherited by environments, so the preview Worker gets the same setting. The Workers' own log lines
remain, and rule 13 governs what they say.

## 20. Every response path sets its own security headers, and a test checks each path

**Rule.** List the paths a response can take — static asset, Worker-rendered page, API, admin — and
make sure each one sets CSP, HSTS, framing and referrer headers. Generate the CSP; never hand-edit
hashes.

*Evidence:* the paths do not share headers the way it looks like they should.

- **`_headers` rules apply to static assets and to responses fetched through `env.ASSETS`, not to
  responses a Worker builds itself.** A Worker-rendered private page got none until it set them in
  code; routing a static page through `env.ASSETS.fetch()` to add one header kept the rest.
- **Astro's `security.csp` hashes only the scripts and styles it processes.** An `is:inline`
  script is blocked by the generated policy: use a processed `<script>`, or add the hash through
  `security.csp.scriptDirective.hashes`. A dist test checks every inline script's hash against the
  page's policy.
- **Per-page CSP additions must run before the layout renders the policy.**
  `Astro.csp.insertScriptResource` called inside a nested component did nothing; call it in the page
  frontmatter, and insert `'self'` along with the new source, because an inserted resource replaces
  the default.
- **A processed `<script>` placed after the layout's closing tag renders after `</html>`.** Browsers
  cope; the validator stops there.
- **`frame-ancestors 'none'` blocks same-origin iframes too**, so browser checks that load pages in
  a frame see nothing; navigate page by page.

## 21. Check the built output, not the template

**Rule.** Search and accessibility rules are asserted against what ships — the built `dist/` on the
`cloudflare` stack, rendered pages from the running preview on `supabase` — and a release-check
script runs the same checks against the preview and, at launch, against production.

What the tests assert, at minimum: a unique `<title>` of 60 characters or fewer and a description
on every page; canonicals without a query string; the expected JSON-LD types; every link has an
`href`; the sitemap lists every public page; `robots.txt` blocks private routes; preview builds are
`noindex` until launch flips one variable; no `[TBD]` (rule 14a); no banned words (rule 14b).

*Evidence:* the checks that only the output could fail.

- A test searching built HTML for an attribute name matched the **inlined language script**, which
  contains the same string once a second language is on. Check for the element, and run the dist
  tests once with every optional feature switched on.
- Lighthouse on the phone profile showed a slow LCP caused by the font loader preloading italic
  files the site never uses. Requesting the styles in use only cut it by about a second. Check the
  preload links after any font change.
- The W3C Nu validator caught what neither tests nor eyes did: `role="switch"` on a submit button,
  a label holding two controls, `aria-pressed` on a link, `figcaption` in the middle of a `figure`.

## 22. Scripted edits keep line endings and escapes

**Rule.** When an edit is made by a script rather than an editor, write the script to a file first,
open files with explicit UTF-8 and `newline=''`, and check line endings afterwards.

*Why:* the shell and the scripting language each reinterpret text on the way through, and the
damage is silent until a test compares strings.

*Evidence:* across one Windows build —

- Python's text mode wrote CRLF, turning a seed and several source files over, and a string
  comparison test caught it. Python's default encoding was cp1252 and failed on a UTF-8 byte.
- A regex written through a Python heredoc arrived with `\b` as a backspace character.
- Heredocs with backticks and `${}` broke in Git Bash repeatedly; `cat > file` with no input waited
  on stdin until the command timed out, and nothing after it ran.
- Git Bash rewrote an argument starting with `/` into a Windows path; `MSYS_NO_PATHCONV=1` stops it.
