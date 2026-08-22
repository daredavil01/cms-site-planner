# Core rules

Always applicable, whatever modules were selected. Copy these into the generated `CLAUDE.md`,
keeping the evidence — **a rule stated without the failure it prevents gets argued away by the
next person who finds it inconvenient**, usually correctly, because a rule with no cost attached
looks like superstition.

Format: **Rule** · *Why* · *Evidence*.

---

## 1. Verify on the deploy runtime, not the dev server

**Rule.** A change to middleware, auth, cookies or any server endpoint is not verified until it
has run under the production runtime.

```bash
npm run build && npm run preview
```

Then hit `http://127.0.0.1:8787/api/health`.

*Why:* the dev server runs on Node; production runs on workerd. Different JavaScript runtimes.

*Evidence:* `@supabase/ssr` throwing `cookies.getAll is not a function` on workerd — a failure
the dev server never surfaced, because on Node that code path never ran. A green dev server is
not evidence.

> On Windows the dev/preview process holds a lock on `dist/`. Stop it before rebuilding, or the
> build fails with `EPERM`.

## 2. Ship a health endpoint before any page

**Rule.** Build `/api/health` first. It exercises the riskiest integration end to end, returns
`200`/`503` with **named** per-check results, and sets `cache-control: no-store`.

*Why:* every stack has one unknown that decides whether the plan works at all. Find out on day
one, when the answer is cheap. It also gives every later deploy one URL to smoke-test.

*Evidence:* discovering the workerd/cookie-adapter incompatibility in rule 1 costs an afternoon
before any pages exist, and a rewrite after five pages do.

**Never identify a state by string-matching a library's error message.** Matching a literal like
`'Auth session missing!'` is a silent breakage waiting for a minor version bump.

## 3. Runtime compatibility flags fail at request time, not build time

**Rule.** Treat `nodejs_compat` and friends as a deployment dependency and verify them with the
health endpoint, not with a successful build.

*Evidence:* the database client reaches for Node built-ins (`crypto`, `stream`, `buffer`).
Without the flag the build is green and the Worker cannot start — a build-green, request-red
failure, confusing precisely because the usual signal was positive.

## 4. RLS is the access control

**Rule.** Public reads use the anon key and see only published rows because the policies say so.
**Do not add a redundant `.eq('published', true)` in a query helper.**

*Why:* a redundant filter hides the case where a policy is wrong. The app would look correct
while the database was leaking.

*Corollary:* **test the policy, not the query.** Probe the database directly under a simulated
anonymous session.

*Evidence:* an empty table makes a policy test pass for the wrong reason — a `select` returning no
rows proves nothing about whether the policy would have filtered them. Seed a published row and an
unpublished one, and assert the anon session sees exactly the first.

## 5. The privileged credential is used by nothing

**Rule.** The service-role key has zero code references, enforced structurally rather than by
review. Four layers, none of them a habit anyone has to remember:

1. Declare it in the typed env schema as `context: 'server', access: 'secret'`, so a client
   import is a **build error**.
2. Provide no privileged client to import — the server client uses the anon key, always.
3. Keep the key out of CI entirely. A key never present in CI cannot leak from CI.
4. Grep the build output for it anyway (rule 6).

```js
env: {
  schema: {
    PUBLIC_SUPABASE_URL:      envField.string({ context: 'client', access: 'public' }),
    PUBLIC_SUPABASE_ANON_KEY: envField.string({ context: 'client', access: 'public' }),
    SUPABASE_SERVICE_ROLE_KEY: envField.string({
      context: 'server',
      access: 'secret',
      optional: true,
    }),
  },
},
```

*Evidence:* the trade worth copying is *"planned a lint guard on the service key → built
`access: 'secret'` instead → a client import is now a build error, not a rule to remember."*
Prefer that substitution whenever it is available.

## 6. CI checks; it does not deploy

**Rule.** CI runs type-check → build → scan the build output for leaked credentials. Deployment
stays a platform concern.

```yaml
- name: Assert no secret key in build output
  run: |
    found=0
    if grep -rlE 'eyJ[A-Za-z0-9_-]+\.[A-Za-z0-9_-]*c2VydmljZV9yb2xl' dist/ 2>/dev/null; then
      echo "::error::A legacy service-role JWT appears in the build output."
      found=1
    fi
    if grep -rlE 'sb_secret_[A-Za-z0-9_-]{20,}' dist/ 2>/dev/null; then
      echo "::error::A secret key (sb_secret_) appears in the build output."
      found=1
    fi
    [ "$found" -eq 0 ] && echo "OK: no secret key in dist/"
    exit $found
```

*Evidence:* both details in there were bugs first.

- **Match every credential format the provider issues.** The legacy JWT pattern alone was
  silently useless against a project using the newer opaque format. A guard that cannot fire is
  worse than no guard, because it looks like coverage.
- **Require entropy after a known prefix.** The client library ships the literal string
  `sb_secret_` inside its own key-detection code, so a bare-prefix match fails every build.
  `{20,}` is what makes it match real keys only.

Store even non-secret build values in repository secrets rather than the workflow file — not
because they are sensitive, but so they can be rotated without a commit.

**Do not declare a script you have not installed.** Either wire up lint and format in the first
phase or delete the lines; a project that declared both and installed neither left everyone
reading a guardrail that could not fire.

## 7. When you think you need the service key, you need a policy or an RPC

**Rule.** Reach for a `security definer` function before a credential that bypasses RLS. See the
worked example in `schema.md`. Return **aggregates, never rows**; pin `search_path`; `revoke`
before you `grant`.

*Evidence:* a homepage counter over a table that grants anon `insert` and no `select` returns
`0` from an anonymous session — **not because the table is empty, but because RLS filters every
row.** It would have read zero forever and looked healthy doing it.

## 8. Auth fails closed

**Rule.** Gate the admin area server-side in middleware, before any HTML is produced. Use
`getUser()`, which revalidates against the auth server — not `getSession()`, which trusts a
cookie the client could have forged. Wrap it in `try/catch` and redirect to login on failure.

*Why:* an unhandled throw becomes a 500, and a 500 on an auth check is the kind of thing that
gets "fixed" by removing the check.

**Read the whole cookie jar off the raw `Cookie` header.** A large auth token splits across
numbered cookies (`sb-<ref>-auth-token.0`, `.1`, …); reading only the names you know breaks
sessions for exactly the users whose token happens to spill.

The role in `locals` is **UI hinting only**. RLS enforces permission on every statement.

*Evidence:* a gate is **two** behaviours — refusing the unauthenticated and admitting the
authenticated — and a check that only ever exercised the first shipped as half a gate. Verify both.
The split-cookie failure has the same shape: it breaks sessions for exactly the subset of users
whose token spills, so it is invisible to whoever tested it.

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
helper takes the **request-scoped** client as its first parameter, never a module-level
singleton, so one request's auth context cannot leak into another; and sort embedded child rows
after the round trip, because the API does not guarantee their order.

**Type the insert payload** so a schema change breaks the build rather than the insert — and
note what the payload omits: a status column is never set from the client, so a submission
cannot arrive pre-marked as handled.

**Validate JSONB; do not trust it.** Free-form JSONB written by an admin UI is user input with
extra steps. Narrow it with an assertion only when runtime shape validation immediately follows,
and use `flatMap` so a malformed entry drops itself rather than the page.

## 10. Build-time values are not runtime values

**Rule.** No `vars` block in the Wrangler config for `PUBLIC_*` values.

*Why:* they are inlined at build time.

*Evidence:* setting them as runtime vars **half-works** — the server renders fine and the browser
is handed `undefined`. That is the worst of both outcomes, because the page looks like it is
working to whoever is checking it.

Use `||`, not `??`, for config that CI may supply as an empty string — `??` passes `''` through.

## 11. JS only where it earns it

**Rule.** Measure before shipping a framework island. Before reaching for one, check whether a
plain link, a `<details>` disclosure, a `:target` selector or a form submission does the job —
those cost zero bytes, work with JS off, and come with keyboard and screen-reader behaviour
already correct.

*Evidence:* **184 KB of framework runtime to animate six numbers**, replaced by a 602-byte inline
script; **181 KB** for a contact form, replaced by a plain `POST`-to-self. Quantify like this:
"184 KB to animate six numbers" ends a discussion, "React felt heavy" starts one.

**Islands live under the authenticated area.** The public site is HTML. Labels, translations,
dates and derived numbers are computed server-side; an island receives finished strings and
numbers as props. Never import the browser database client from a server template — it drags the
client bundle into pages that otherwise ship no JS, and it does so silently.

In `.tsx`, the framework uses `className`, not `class`.

## 12. Migration discipline

**Rule.** The first migration applies **once, to a fresh project** — its `create policy`
statements throw on a re-run. Never edit it to make it re-runnable; write a new numbered file.

**Check before you push, every time:**

```bash
supabase migration list
```

Read the `remote` column. `supabase db push` applies everything the remote history omits, so
against a drifted database it re-runs migrations whose rows already exist.

**Applying SQL by hand is not applying a migration** — the dashboard SQL editor counts. The
apply is not finished until the repair has run:

```bash
supabase migration repair --linked --status applied <version>
```

*Evidence:* this rule was in place for four phases of one project and history drifted anyway,
because nothing verified it — and the second drift was created by someone who was not pushing,
because the rule had been written on the push side while the failure was on the apply side.

When the columns disagree, find out whether the **content** is there before choosing a fix:

```bash
supabase db query --linked "select count(*) from public.<table>"
```

That reads the real database with no container and no privileged key. Content present → repair.
Content genuinely absent → re-run. Guessing duplicates production data.

**A pending migration breaks writes, not reads.** The API rejects an entire update when one
column in the payload does not exist, so the public site looks fine while every admin save on
that table fails.

Migrations carrying non-Latin seed data must stay UTF-8 **without a BOM**. If you generate SQL
from a shell, read and write with explicit UTF-8 — a default encoding mangles every non-ASCII
string.

**A role granted through a dashboard is a role the migrations do not know about.** Grant roles
in a migration, or the next fresh project is missing them.

## 12a. Storage columns hold paths, never URLs

Applies to **any** project with a storage bucket, whether or not the `media` module is on.

**Rule.** A storage column stores a path. A URL is produced only by one helper, and **private
buckets stay out of that helper's type union.**

```ts
export type PublicBucket = 'site' | 'resources';
//  `files` is private and deliberately absent — it needs a signed URL.

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
`media` module is off** — the egress arithmetic (`uploaded size × views = the bill`) binds on a
dozen fixed photographs too, and its failure mode is images quietly ceasing to load.

## 13. Errors: verbose inside the gate, opaque at the gate

**Rule.** Surface the database's own error message to staff.

*Evidence:* `duplicate key value violates unique constraint programs_slug_key` is something an
editor can act on without help. "Something went wrong" is a support request.

Do the **exact opposite** on the login form. The raw auth error distinguishes "no such user"
from "wrong password", which is an account-enumeration hint. Show a generic failure there.

## 14. Never publish invented content

**Rule.** Anything that did not come from the client stays `published = false` until they sign
off. Build the empty state.

*Why:* **an empty state is safer than a plausible one.**

*Evidence:* three categories from a real prototype did the damage, and each fails differently.
Invented **impact figures** get quoted back to the organisation. An invented **testimonial
attributed to a named beneficiary** is a consent problem, not merely an accuracy one. Invented
**account or tax-registration numbers** read as a claim the organisation cannot back — and donors
rely on those for deductions, so a blank one must be suppressed rather than rendered empty.

Suppress a field that has no value rather than rendering it blank.

## 15. Design tokens, not inline styles

**Rule.** Colours, type and spacing are CSS custom properties. **No component hardcodes a hex
value.** Interactive controls are `min-height: 44px`; the target is WCAG 2.1 AA.

Mobile and accessibility are constraints, not polish. Do not shrink controls to make a layout
fit.

A visual prototype is a **read-only reference**.

*Evidence:* its inline styles, its CDN font `<link>`, its entrance animation, its bottom toolbar
and the typeface it loads but never uses are all prototype decisions, and every one of them ships
by default if nobody names them as such. A prototype's asset loading is a prototype decision —
copy the design, not the delivery.

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

## 17. Commit at the boundary you would want to roll back to

**Rule.** Commit and push at each phase boundary.

*Evidence:* a deploy pipeline that triggers on push does nothing whatsoever for work that was
never pushed. One project shipped three phases as a single 51-file commit, and the deployed
artifact had no commit behind it — so it could be neither reproduced nor rolled back.
