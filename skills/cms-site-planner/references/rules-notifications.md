# Module `notifications` — email, reminders and staff alerts

Load only when the interview selected `notifications`: the site sends email or prepares messages —
confirmations, reminders, staff alerts, digests. A contact form that only stores its submission does
not need this.

Format: **Rule** · *Why* · *Evidence*.

---

## 1. Send from the organisation's own domain, with SPF, DKIM and DMARC

**Rule.** Every sender address is on the organisation's domain (third-party senders on a subdomain,
to keep SPF short). Publish SPF and each provider's DKIM records; start DMARC at `p=none` with
reports, then move to `quarantine` and `reject` after a few weeks. A config test checks every sender
address is on that domain (`rules-core.md` §18).

*Evidence:* a sender configured as a free-mail address would have failed every alert, because the
email service sends only from the zone's own domain. Nothing local reported it — locally the send
binding just writes a file.

## 2. An unconfigured provider is a logged failure, not a crash

**Rule.** When the provider key or sender is missing, each message is recorded as failed with a
reason such as `not_configured`, and the action that triggered it still succeeds.

*Why:* development and preview then run without real keys, and production without a key shows up as
a list of failed messages rather than as broken bookings.

## 3. Every message is a row in a log

**Rule.** One log row per action — prepared, copied, opened, queued, sent, delivered, bounced,
failed — with the template, version, language, trigger and a masked recipient. Message text, if
kept, is encrypted (`rules-privacy.md` §3) and retained no longer than the records it belongs to.
Failed sends show a warning and a "send again" action.

*Evidence:* once previews are log rows too, tests that list a record's messages must filter them
out, and timelines must hide them — or they fill up with "prepared".

## 4. Scheduled sends claim their work atomically

**Rule.** A scheduled job claims each message before sending it, with a unique key per message
(type, date, recipient, channel), so overlapping runs split the work and a rerun sends nothing.

```sql
INSERT INTO send_claims (kind, day, recipient, channel)
SELECT kind, day, recipient, channel FROM due_messages WHERE send_at <= ?
ON CONFLICT DO NOTHING
RETURNING kind, day, recipient, channel;
```

*Evidence:* `RETURNING` gives back only the rows this run inserted. The `SELECT` needs a `WHERE`, or
SQLite parses `ON CONFLICT` as a join's `ON`. To claim existing rows without `UPDATE … LIMIT`, use
`UPDATE … WHERE id IN (SELECT … LIMIT n) AND claimed_at IS NULL RETURNING …`.

## 5. Test scheduled jobs at simulated times, one job at a time

**Rule.** Jobs read the scheduled time from the trigger, never the wall clock, so a test can run the
evening job at noon. In a test for one job, switch the others off.

*Evidence:* at a late send time every earlier message is due too, and a test of one digest counted
another job's reminder in its total. Count by subject or use case, not totals. A second trap:
records stamped with the real time and claims stamped with the scheduled time do not compare in
tests that simulate the future.

**Relative wording goes stale.** "Sends at 6:00 pm" read wrong at 9 pm, before the next run. Check
time-based screens after their time as well as before.

## 6. Provider webhooks are verified over the raw body

**Rule.** Read the raw body before parsing, verify the signature with a constant-time comparison
(`crypto.subtle.verify`), refuse unsigned or stale requests, and answer 2xx to valid events you
ignore.

*Evidence:* providers retry anything that is not 2xx, so an ignored event type answered 400 comes
back forever. For Svix-signed webhooks (Resend and others): the key is the base64 part after
`whsec_`, the signed text is `<id>.<timestamp>.<raw body>`, and the signature header holds
space-separated `v1,<base64>` entries.

## 7. Templates are versioned, and approved versions never change

**Rule.** One template per use case, with text per language and variables in braces. Saving refuses
an unknown variable or empty text; preview uses sample values. A draft is the next version; approving
makes it the version in use; approved versions are made immutable by trigger, so the log can always
show what was sent.

## 8. Messaging apps without an API: copy first, then open

**Rule.** Where messages go through an app with no affordable sending API, the admin builds a
prefilled link for staff to open. The button writes the full text to the clipboard **and then**
opens the link, both synchronously in the same click handler.

*Why:* the app sometimes opens with an empty message (desktop apps, business accounts, long text in
non-Latin scripts); with the text already copied, staff paste it. Both calls in one click keep the
browser's user-activation. If clipboard access is refused, select the text in a box for a manual
copy, and keep nothing in browser storage afterwards.

## 9. Know the sending limits before planning reminders

**Rule.** Write the provider's daily and monthly limits into the plan next to the expected volume of
confirmations, reminders and digests (`stacks.md` §4). A reminder feature on a free tier is sized by
its daily cap, not its monthly one.
