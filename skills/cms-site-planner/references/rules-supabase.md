# Stack `supabase` — Astro SSR + Supabase

Load only when the stack is `supabase` (`stacks.md`). These rules were moved out of `rules-core.md`
when the skill gained a second stack; their evidence comes from the two production sites this skill
was first built from.

Format: **Rule** · *Why* · *Evidence*.

---

## 1. RLS is the access control

**Rule.** Public reads use the anon key and see only published rows because the policies say so.
**Do not add a redundant `.eq('published', true)` in a query helper.** This is `rules-core.md` §4
on this stack.

*Corollary:* **test the policy, not the query.** Probe the database directly under a simulated
anonymous session.

*Evidence:* an empty table makes a policy test pass for the wrong reason — a `select` returning no
rows proves nothing about whether the policy would have filtered them. Seed a published row and an
unpublished one, and assert the anon session sees exactly the first.

## 2. The privileged credential is used by nothing

**Rule.** The service-role key has zero code references, enforced structurally rather than by
review. Four layers, none of them a habit anyone has to remember:

1. Declare it in the typed env schema as `context: 'server', access: 'secret'`, so a client
   import is a **build error**.
2. Provide no privileged client to import — the server client uses the anon key, always.
3. Keep the key out of CI entirely. A key never present in CI cannot leak from CI.
4. Grep the build output for it anyway (§3).

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

## 3. Scan the build output for every key format Supabase issues

**Rule.** CI's last step greps `dist/` for both credential formats (`rules-core.md` §6).

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
  silently useless against a project using the newer opaque format.
- **Require entropy after a known prefix.** The client library ships the literal string
  `sb_secret_` inside its own key-detection code, so a bare-prefix match fails every build.
  `{20,}` is what makes it match real keys only.

## 4. When you think you need the service key, you need a policy or an RPC

**Rule.** Reach for a `security definer` function before a credential that bypasses RLS. See the
worked example in `schema.md` §3. Return **aggregates, never rows**; pin `search_path`; `revoke`
before you `grant`.

*Evidence:* a homepage counter over a table that grants anon `insert` and no `select` returns
`0` from an anonymous session — **not because the table is empty, but because RLS filters every
row.** It would have read zero forever and looked healthy doing it.

## 5. Auth mechanics on this stack

**Rule.** In middleware, use `getUser()`, which revalidates against the auth server — not
`getSession()`, which trusts a cookie the client could have forged (`rules-core.md` §8 for the
fail-closed part).

**Read the whole cookie jar off the raw `Cookie` header.** A large auth token splits across
numbered cookies (`sb-<ref>-auth-token.0`, `.1`, …); reading only the names you know breaks
sessions for exactly the users whose token happens to spill.

The role in `locals` is **UI hinting only**. RLS enforces permission on every statement.

*Evidence:* the split-cookie failure breaks sessions for exactly the subset of users whose token
spills, so it is invisible to whoever tested it.

## 6. Migration history drifts; check it before every push

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

## 7. The free tier pauses

**Rule.** If the project is on Supabase Free, record in `status.md` under blockers and risks that
the project pauses after **7 days of low activity**, and name the plan for it: a paid tier, or an
accepted risk with an owner.

*Why:* on this stack public pages are rendered from the database per request. A paused project is
not a slow site; it is a site whose every SSR page fails.

*Evidence:* a later zero-cost brief chose the `cloudflare` stack for this reason alone
(`stacks.md` §2). Neither of the two sites this file comes
from was on a tier where it applied, which is exactly why the first release of this skill did not
mention it.

## 8. Tests on this stack — designed, not yet run

`rules-core.md` §7 requires tests for every rule that protects people. Neither production site on
this stack had a suite, so this is a design carried as an unproven area, not a recipe:

- policy probes run against a local Supabase (`supabase start`), seeded with one row that must be
  visible and one that must not, under the anon role and under each staff role;
- write paths tested through the same request-scoped client the app uses, not a privileged one;
- rendered pages tested through the Astro container API or against `npm run preview`.

Carry "test suite design unproven on `supabase`" into the generated `status.md` under paths never
exercised. The `cloudflare` stack's suite (`rules-cloudflare.md` §5) is the evidence that a real
local database per test is cheap enough to be worth building.

## Known blind spots on this stack

Carry these into the generated `CLAUDE.md` §7.

- **A signed-in editor sees unpublished drafts on the public site**, because the same cookie-bound
  client serves both. Kept deliberately — it is free draft preview — but it means **"it looks fine
  to me" is not evidence about what visitors see.** Check in a private window.
- **There is no caching at the data layer.** Every request hits the database, and the base layout
  runs its chrome queries before the page's own. Deferred and tracked — not discovered later.

## Commands the generated `CLAUDE.md` lists

| Command | What it is for |
|---|---|
| `npm run dev` | Dev server — never evidence (`rules-core.md` §1) |
| `npm run lint`, `npm run typecheck`, `npm run build`, `npm test` | The CI steps, in CI's order |
| `npm run preview` | Build, then serve under workerd on port 8787 |
| `supabase migration list` | Before every push (§6) |
| `supabase db push` | Apply pending migrations to the linked project |
| `supabase migration repair --linked --status applied <version>` | After any hand-applied SQL |
