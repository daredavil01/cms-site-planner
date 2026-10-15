# Module `booking` — time that two people must not both get

Load only when the interview selected `booking`: visitors or staff reserve a time with a person, a
room or any resource that cannot be double-booked — appointments, classes, rentals, visits. A
booking holds a name and a phone number, so `privacy` and `forms` are on as well.

Format: **Rule** · *Why* · *Evidence*.

---

## 1. The engine is its own phase, and it ends at a test

**Rule.** Plan the booking engine — rules, availability, the atomic write, expiry — as a phase with
no screens, whose checkpoint is a concurrency test. The visitor's booking screens and the staff
screens are separate, later phases.

*Why:* double booking is the one failure the business cannot absorb, and it never shows up when one
person clicks through a screen. Retire it while the engine is small enough to test exhaustively.

## 2. Time is fixed blocks, and a booking locks each block it needs

**Rule.** Split the schedule into fixed blocks (15 minutes suits most appointment work). A booking
inserts one **lock row per block** it occupies, under a unique index on (resource, block start). The
booking, its locks and its first event are **one atomic write** — a D1 batch, or a Postgres
transaction. If any block is taken, the unique index fails the whole write and nothing is saved.

*Why:* the database refuses the second booking. No application check, however careful, can close the
window between "is it free?" and "book it" on its own.

*Evidence:* the schema must also refuse **timestamps off the block grid**. A lock at 10:20 does not
collide with one at 10:15, so a single off-grid time dodges the index entirely. Check the grid in a
constraint, and check that the site's time-zone offset is a multiple of the block size — an offset of
`:30` or `:45` breaks a 60-minute grid.

On Postgres an exclusion constraint over a time range can replace lock rows; that variant is
designed, not run — carry it into `status.md` as unproven if chosen.

**After a failed write, tell a repeat from a conflict.** The failed write saved nothing; look up the
idempotency key (a repeat — return the first booking) or a lock on one of the blocks (taken — say
so), and rethrow anything else.

## 3. Idempotency is checked first, everywhere a booking is created

**Rule.** Every form that creates a booking — public or staff — carries an idempotency key, and the
endpoint checks it before the bot check and before the availability check (`rules-forms.md` §3).

*Evidence:* a staff form submitted twice found its own new booking occupying the time and answered
"taken". The public endpoint already checked the key first; the staff path had been written without
it.

## 4. The concurrency test makes every pair overlap

**Rule.** Fire N booking requests at once (`Promise.all` of real requests against a real local
database) and assert exactly one succeeds. Choose the requests so that **every pair overlaps**.

*Evidence:* all the parallel requests passed the free-time check before any write ran; only the
unique index let exactly one through. Requests that do not all overlap pairwise can produce two
legitimate winners, and the test then fails for a reason that is not a bug — or passes for one that
is.

## 5. Status changes free locks in the same statement

**Rule.** Cancelling, declining or expiring a booking frees its locks **in the same write** — a
trigger on the status column is the simplest guarantee. Rescheduling frees the old blocks and locks
the new ones in one write. An expiry job frees unconfirmed requests on a schedule.

*Why:* a status change that forgets its locks leaves time permanently unbookable, and nobody notices
until a customer asks why a free afternoon shows as full.

**Everything that consumes the resource's time locks blocks the same way** — staff-blocked time,
visits elsewhere, events — with an entity column saying what owns the lock. Availability then needs
no special cases: every lock but the booking's own is taken.

## 6. A staff action and a customer action can race

**Rule.** Every change to an existing booking guards on the state it was read in, inside the write,
and reports "changed" when the guard fails (`rules-cloudflare.md` §3 for the D1 pattern).

*Why:* the customer reschedules on their phone while the front desk confirms the old time.

## 7. Unconfirmed rules mean "unavailable", not a guess

**Rule.** Durations, notice periods, booking windows and cutoffs the client has not confirmed are
`null`. While any is null, the booking endpoints answer "unavailable" with the contact phone number,
and the admin lists exactly what is missing (`rules-core.md` §14a). Tests set their own made-up rules.

## 8. The private manage page is a token plus a second factor

**Rule.** A booking's manage link carries a random token; only its hash is stored. Opening the
details needs a second factor the person knows (for example the last digits of their phone number),
with a lockout after a few wrong tries. After a correct answer, a short session in an `HttpOnly`,
`Secure`, `SameSite=Strict` cookie holding a random id. The link expires a set time after the
booking.

*Why:* links get forwarded, and databases get copied. Neither alone should expose a booking.

*Evidence:* three details that tests forced.

- **Count before comparing.** Increment the try counter first (`INSERT … ON CONFLICT DO UPDATE …
  RETURNING`) and compare only while it is under the limit. Read-then-compare let parallel guesses
  through; a test fires a dozen at once.
- **Derive the token from the idempotency key** with an HMAC/HKDF under a fixed label, so a retried
  request returns the same link while the database holds only the hash. Different kinds of private
  link use different labels, so a token can only ever match one table.
- **A staff reschedule is not a customer reschedule.** Record which it was in the event, or the
  customer's history reads "confirmed by the business" twice.

## 9. Calendar files and reminders use the booking's neutral name

**Rule.** The `.ics` file, reminders and confirmations name the service the way `rules-privacy.md`
§7 says — never a sensitive service's real name.

## 10. Switches turn booking off without breaking existing bookings

**Rule.** Online booking sits behind a switch, with sub-switches for kinds of booking that may launch
later. Turning the parent off turns its dependants off; endpoints check the switch on every request
(a short cache is fine); existing bookings, their manage pages and the admin keep working.

*Evidence:* a switch cached for a minute per database means a test must change the switch before its
first request, or use a fresh database.
