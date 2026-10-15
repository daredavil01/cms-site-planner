# Module `i18n` — multi-locale content

Load only when the interview returned two or more locales. A single-locale project gets plain
`text` columns and **none** of this — do not plan a translation layer "in case".

§2 is how languages are served on the `supabase` stack; §2a is the `cloudflare` stack, whose content
pages are static. Everything else applies to both.

---

## 1. The primary audience's language is the default, not the translation

**Rule.** The locale most of the audience reads is the site default and the first key in every
JSONB object. The second language is the alternate.

*Why:* it decides the fallback direction, the font that must always load, and which language an
untranslated row degrades *to*. Getting it backwards means the majority audience is the one
seeing fallback markers.

Name the primary locale explicitly in the generated `CLAUDE.md`. It is the single fact that the
rest of this file depends on, and it is the one a new contributor will assume wrongly.

## 2. One URL, all languages

**Rule.** Serve every language from the same URL. Resolve as **explicit cookie → `Accept-Language`
→ default**. Handle `?lang=` server-side: set the cookie, then redirect (303) to the clean URL.

```ts
const requested = url.searchParams.get('lang');
if (isLang(requested)) {
  cookies.set(LANG_COOKIE, requested, {
    path: '/', maxAge: LANG_COOKIE_MAX_AGE, sameSite: 'lax',
    httpOnly: false,           // the toggle reads it to render its own label
  });
  const clean = new URL(url);
  clean.searchParams.delete('lang');
  return redirect(clean.pathname + clean.search, 303);
}
```

*Why:* the URL stays clean and shareable, the choice persists, and — the part that pays off
elsewhere — **the language control can be a plain `<a>`**, because the server does all the work.
That is a whole island avoided.

Give the cookie a long life (a year), so the choice survives between visits on a shared phone.

Emit `hreflang` alternates plus `x-default`, and set `og:locale` to the region the audience is
actually in, not the region the language is stereotypically associated with.

*Considered and rejected:* path prefixes (`/en`, `/xx`). They double the URL surface and split
inbound links across two addresses — a large change bought for a weak signal.

## 2a. On a static build, choose between path prefixes and a browser swap — and say what it costs

**Rule.** Static content pages have no per-request server, so §2's cookie resolution is unavailable
for them. Ask, and record the choice as a decision:

| Option | Gains | Costs |
|---|---|---|
| **One page per locale** under a path prefix | Every language indexed; works without JS; `hreflang` possible | URL surface multiplies; inbound links split |
| **Primary-language pages + per-page translation files swapped in the browser** | One URL per page; a language is a data file, not a build target | Other languages largely unindexed (canonical is the primary page); non-primary text needs JS; layout shift to manage |

For the browser swap, resolve `?lang=` → stored choice → default, store only the language code, and
update the URL with `history.replaceState` so a shared link opens in the same language. Pages the
Worker renders itself (private pages, forms) read the same choice and render that language directly.

*Evidence for the swap, all from layout shift:*

- **Hiding only the text does not stop layout shift.** `visibility: hidden` on the translated
  elements kept their boxes, so the swap to longer text and then the late script font moved
  everything — CLS of 0.6–0.9 on translated pages on the phone profile. Hiding the whole `body` until
  the text is swapped **and** `document.fonts.ready` resolves gave CLS 0. Start the font load with
  the text fetch (`document.fonts.load(...)` with a sample of the script), not after it, and show the
  primary language if either fails.
- **A footer under script-rendered content jumps** when the content arrives; reserve the space
  (`min-height`) until it does.
- **Content that waits for the translation should wait for the language event**, or it fetches
  twice — once in the default language and again after the swap.
- **Text a script composes cannot use the swap's snapshot.** A swap that restores the primary
  language by cloning a snapshot drops anything a script changed. Script-rendered flows keep their
  strings in a JSON block and re-render on the language event.

## 3. Translatable columns are JSONB, read through exactly one helper

**Rule.** Every translatable column is JSONB keyed by BCP-47 tag: `{"<primary>": …, "<alt>": …}`.
Nothing indexes those objects directly; everything goes through `t(field, lang)`.

*Why:* the fallback lives in exactly one place, and adding a language becomes a data change
rather than a migration.

*The tradeoff, stated honestly:* JSONB gives up per-locale database constraints and makes sorting
or filtering by translated text clumsy. For a content site that is a good trade. For a product
that must search across translations, it may not be.

**Widen the type once, at the boundary.** Generated database types give every JSONB column the
type `Json`, which does not assign to a locale-field type. Widen once at the helper's signature
and validate the shape at runtime, rather than casting at every call site — casting everywhere
costs readability and buys the same amount of safety: none.

```ts
export type I18nInput = I18nField | Json;
```

**On `cloudflare` the proven shape is different, and the rule is the same.** Each content table has a
translation table with one row per locale and per state (draft, published), read through one helper
that picks the visible row (`schema.md` §8). The fallback still lives in exactly one place, and
adding a language is still a data change.

## 4. Fallbacks must fail visibly-but-safely

Two rules that look similar and are **opposites**.

- **A missing content field returns `''`**, never `undefined`. A half-filled row should render as
  an empty section, not print `undefined` on a public page.
- **A missing UI string falls back to its own key.** A missing `nav.home` should read as an
  obvious `nav.home` in the navigation so somebody fixes it.

The distinction: **content can legitimately be absent; chrome cannot.** Make absence quiet where
it is expected and loud where it is a bug — a blank gap in the navigation is invisible until a
user reports it.

## 5. Bind `lang` to the text, not to the page

**Rule.** When a field falls back to another language, the element rendering it declares *that*
language.

*Why:* screen-reader pronunciation and script-specific font rules both follow the element, not
the document. A page in one script containing one paragraph of another, marked with the page's
language, is read aloud as if the second script were the first.

*Consequence for the API:* this forces the helper to return a structure, not a string.

```ts
export interface Translated {
  text: string;
  fellBack: boolean;   // true when `text` came from another language
  lang: Lang;          // the language `text` is actually in — bind this to lang=
}
```

`fellBack` also gives the UI what it needs to show a quiet *(English)*-style marker, so the
reader knows why the text switched.

Note the failure this prevents at the template level: an absent translated field can take the
`<h1>` with it. Check what a half-translated row does to the page structure, not only to the
prose.

## 6. Never store `""` for an untranslated field — drop the key

**Rule.** The admin save path strips empty values rather than writing them.

*Why:* an empty string is a **present** value, so the fallback never fires. A half-translated row
then renders a blank gap where the other language should have appeared — exactly the failure
rule 4 was designed to prevent, reintroduced from the write side.

*Evidence:* both halves were needed. The plain-text path drops blank strings; the rich-text path
additionally strips `<p><br></p>`, because **an emptied contenteditable is not empty** — every
browser leaves that markup behind, and storing it makes a body with no text count as translated.

## 7. Script-specific typography goes on `:lang()`, and the font is not free

**Rule.** Apply per-script sizing with `:lang()` selectors, never classes.

*Why:* it then follows the element's real language automatically, which is what makes rule 5 pay
off.

```css
:lang(en) { font-size: 16px; line-height: 1.65; letter-spacing: -0.01em; }
:lang(hi) { font-size: 18px; line-height: 1.75; letter-spacing: normal; }
```

A second script often needs more size and leading than Latin — marks above and below the baseline
need the room — and the Latin tracking-tightening actively hurts it.

**Budget for the font.** A second script's font file is large and mostly wasted unless it is
`unicode-range` gated, which requires **self-hosting**: a CDN `<link>` will not give you
per-subset ranges.

```css
@font-face {
  font-family: 'SiteScript';
  src: url('/fonts/site-script-subset.woff2') format('woff2');
  unicode-range: U+0900-097F, U+A8E0-A8FF, U+1CD0-1CFF;
  font-display: swap;
}
```

*Evidence, and the whole mechanism:* **seven characters** of the second script in the site chrome
— a two-character logo mark and a five-character language-toggle label — pulled the entire 121 KB
file onto every page in the *other* language. Gating only helps if the page truly has no glyphs
from that script. Check the network panel before assuming a language you are not rendering costs
nothing.

With the browser swap (§2a), load a script's font only once that language is chosen. Either way,
check what the font loader preloads (`rules-core.md` §21).

## 8. A locale is not a specification

**Rule.** Do not assume a locale tag gives you correct date, number or calendar formatting for
the audience. Check what the runtime actually produces for the tags you ship, and pin the format
explicitly where it matters.

**Fallback text needs fallback formatting.** When a string falls back to another language, format
the dates and numbers around it in the language of the text actually shown. *Evidence:* a missing
translation returned primary-language text with dates formatted in the requested language — a
sentence in two languages. Check for the locale's own published row before choosing the format.

## 9. Test every language without waiting for translators

**Rule.** Before real translations exist, run a pseudo-translation pass: copy every published
primary-language row to each other locale with a visible per-language prefix, switch the languages
on, build, and flag any visible translatable text without the prefix. Then delete the pseudo rows and
compare row counts against a snapshot taken first.

*Why:* it finds every string that will not switch — hard-coded labels, script-composed text, alt
text, titles — while the only cost is a script.

*Evidence:* prefix structured (JSON) bodies by walking the parsed value. A string replace on
`"items": ["` prefixed only the first list item and looked exactly like missing translations.

## 10. Translations publish after review, keyed so they survive edits

**Rule.** A translation has its own draft and published state, a reviewer and a review date, and
publishes only when reviewed. Structured bodies (block editors, lists) key each translatable piece by
a **stable id** generated when the block is created, never by position.

*Why:* when blocks move, positional keys attach every translation to the wrong paragraph.

*Evidence:* give generated ids a shape that cannot collide with fixed keys (hex ids cannot equal
`title` or `summary`), so one lookup serves both.

---

## Caveat for three or more locales

The `supabase` approach (§2) is proven at **two** languages in production. The `cloudflare` browser
swap (§2a) was built and regression-tested at four with pseudo-translations, but no real
translation had shipped. At N ≥ 3, two things change:

- **The toggle may become a picker.** At four languages a row of plain links worked — native names,
  no flags, each link carrying its own `lang`, in a labelled group, each 44 px tall. Somewhere past
  that, a chooser becomes a `<details>` disclosure containing links — still no JS, but a different
  control that needs its own keyboard and screen-reader check.
- **The fallback becomes an ordered chain**, not a single alternative. `t()` must walk a
  per-project priority list, and `fellBack` alone no longer tells the reader *which* language they
  are getting — bind `lang` and surface the actual language in the marker.

Carry both into the generated `status.md` under paths never exercised. An unexamined gap does not
get closed; an admitted one does.
