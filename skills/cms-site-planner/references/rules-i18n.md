# Module `i18n` — multi-locale content

Load only when the interview returned two or more locales. A single-locale project gets plain
`text` columns and **none** of this — do not plan a translation layer "in case".

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

## 8. A locale is not a specification

**Rule.** Do not assume a locale tag gives you correct date, number or calendar formatting for
the audience. Check what the runtime actually produces for the tags you ship, and pin the format
explicitly where it matters.

---

## Caveat for three or more locales

Everything above is proven at **two** languages. At N ≥ 3, two things change and neither has run
in production:

- **The toggle becomes a picker.** A plain `<a>` swapping between two languages is trivially a
  link; a chooser among five is a `<details>` disclosure containing links — still no JS, but a
  different control that needs its own keyboard and screen-reader check.
- **The fallback becomes an ordered chain**, not a single alternative. `t()` must walk a
  per-project priority list, and `fellBack` alone no longer tells the reader *which* language they
  are getting — bind `lang` and surface the actual language in the marker.

Carry both into the generated `status.md` under paths never exercised. An unexamined gap does not
get closed; an admitted one does.
