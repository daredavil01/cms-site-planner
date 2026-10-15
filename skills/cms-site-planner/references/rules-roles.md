# Module `roles` — more than one kind of editor

Load only when the interview selected `roles`. A single admin role needs none of this — and turning
it on means every access rule gains a role predicate, so do not turn it on speculatively.

Format: **Rule** · *Why* · *Evidence*.

---

## 1. One table maps every screen and route to the roles that may use it

**Rule.** Roles and permissions live in one table in code (screen or route → allowed roles), read by
the gate on every request and by the navigation. On `supabase` every staff policy also carries the
role predicate (`schema.md` §2); on `cloudflare` the admin Worker's guard reads the table.

*Why:* permissions scattered across handlers are permissions nobody can review.

## 2. A role-matrix test walks every route for every role

**Rule.** One test iterates the table and requests every route as every role, asserting allowed
roles get through and every other role gets 403.

*Evidence:* the matrix test caught failures no feature test did — including a list screen that
answered 500 when it had nothing to log, which only showed for a role that saw an empty list.

## 3. The role comes from the server's own lookup

**Rule.** After authentication, look the account up in the application's own users table, refuse
unknown or deactivated accounts, and take the role from there — never from the client, and never
from a claim the identity provider happens to pass along. Removing someone means deactivating them
the same day, as well as removing them from the identity provider.

Anything that hides a section by role in the UI — navigation, help pages — is convenience, not
protection; the route keeps its own guard.

## 4. Approval is data: `approved_at >= updated_at`

**Rule.** Where some changes need an owner's approval (prices, legal facts, registration numbers),
store `approved_by` and `approved_at` on the row. A change by anyone else leaves `approved_at` older
than `updated_at`; the owner's own change sets both to the same time; publishing copies a row only
when `approved_at >= updated_at`.

*Why:* approval then survives edits correctly without a separate workflow table, and "awaiting
approval" is a query.

Decide per field whether a visibility change resets approval, and write the decision down.

## 5. Translators edit their languages and cannot publish

**Rule.** If `i18n` is on and translators are a role: each translator has a list of allowed locales,
every translation write checks it, and publishing is refused to the role.

## 6. Every change writes an audit row in the same write as the change

**Rule.** Each mutating route writes an audit row (who, what, which record, when) **in the same
atomic write** as the change. Data-layer functions accept the caller's extra statements, so the
audit row cannot be forgotten or committed separately. Sign-ins and views of personal data are
audited too when `privacy` is on.

*Evidence:* an audit row written after the change can be lost when the request fails between the
two; one written in the same batch cannot.

## 7. Every role is exercised locally

**Rule.** Development seeds one made-up account per role, and switching role is one setting (on
`cloudflare`, the `DEV_ACCESS_EMAIL` stand-in — `rules-cloudflare.md` §6). Check each new screen as
every role that can open it, and as one that cannot.
