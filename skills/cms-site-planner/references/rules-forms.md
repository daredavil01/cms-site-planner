# Module `forms` — public-write surfaces

Load only when the interview selected `forms`: the public submits anything — contact, enquiry,
registration, volunteer, newsletter. If what they submit is sensitive (health, children, finances),
`privacy` is on as well and its rules come on top of these.

Format: **Rule** · *Why* · *Evidence*.

---

## 1. Public-write surfaces are never read back by the public

**Rule.** The public can insert and nothing else. On `supabase`: an `insert … with check (true)`
policy and **no anon `select`** (`schema.md` §2). On `cloudflare`: the endpoint writes, no public
route reads, and the table is absent from the export allow-list (`rules-cloudflare.md` §1).

*Why:* these tables hold names and phone numbers of people who contacted the organisation.

A public page that needs a fact from such a table — a count — gets it from a function or endpoint
that returns the aggregate only (`schema.md` §3).

## 2. Every public form gets the same five defences

**Rule.** A bot check (Cloudflare Turnstile), a honeypot field, a rate limit keyed on a **salted
hash** of the IP address, a size cap, and a strict field allow-list that refuses unknown or repeated
fields. One shared function applies all five, so a new form cannot ship with four.

*Why:* each one stops a different thing, and a form missing one is the form that gets used.

*Evidence:* the rate limit counts **before** validation, so even an invalid post spends one of the
hour's tries. That is right — validation is what an attacker probes — and it has a cost to plan for:
the release check posts deliberately invalid details and uses one of the hour's five tries per run,
so clear the local rate-limit rows afterwards. Locally, Turnstile's always-pass test secret
(`1x0000000000000000000000000000000AA`) stands in for the real one.

## 3. Check the idempotency key before the bot check

**Rule.** A form that creates something carries an idempotency key generated when the form renders.
The endpoint looks the key up **first** — before the bot check and before any availability check —
and returns the original result for a repeat.

*Why:* a double tap or a retry on a bad connection must not create two records, and must not be
refused either.

*Evidence:* Turnstile tokens work once, so a retry carrying the same token fails the bot check even
though the first attempt succeeded — the key check has to come first for the retry to get its
record back. Separately, a staff form submitted twice found its own new record occupying the
resource and reported a conflict; the key check had been placed after the availability check. A test
that submits twice caught it.

## 4. Size caps count encoded bytes

**Rule.** Derive a free-text form's size cap from the longest legitimate message in the
**longest-encoding script** the site accepts, after percent-encoding.

*Evidence:* a letter in many non-Latin scripts is three UTF-8 bytes, each written as `%XX` — about
**9 characters per letter** once URL-encoded. A cap sized by testing in English refused an ordinary
comment written in another script. Size the cap by encoding a realistic message in each script the
site accepts.

## 5. Same-origin checks use `Sec-Fetch-Site`, and allow `Origin: null`

**Rule.** Refuse a form post whose `Sec-Fetch-Site` is anything but `same-origin`. Do not refuse
`Origin: null`.

*Why:* with `Referrer-Policy: no-referrer` — the right policy for private pages — browsers send
`Origin: null` on form posts, even to the same origin.

*Evidence:* a private page refused its own forms. Node tests never send these headers, so
every test passed; only a browser found it. A test harness that drives the app without a browser
(Playwright routing to an in-process app) sends no `Sec-Fetch-Site` at all and needs to add it.

## 6. Answers never echo what was submitted

**Rule.** Success and failure answers carry codes and fixed text only. One "thank you" state for
every topic, in place on the same page — never a URL per topic.

*Why:* a submitted value echoed in an error is a reflected-content bug, and a topic in a thank-you
URL is a disclosure in every browser history and analytics report.

*Evidence:* the release check posts invalid details with a canary string to every public endpoint
and fails if the canary appears in any answer (`rules-core.md` §13).

## 7. A form with several submit buttons needs a default

**Rule.** When a form has more than one submit button, put the intended default first — visually
hidden if the design puts it elsewhere.

*Evidence:* Enter in a text box clicks the **first** submit button in the form. In a block editor
whose first button was "Move up", pressing Enter moved a block.
