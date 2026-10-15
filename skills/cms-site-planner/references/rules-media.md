# Module `media` — the media library

Load only when the interview selected `media`. A site whose images are decorative and never
reused does not need this; a site where an editor uploads a photograph that appears on three
pages does.

---

## 1. Media is referenced by ID, never by path

**Rule.** There are **no `*_path` columns on content tables.** Every image, video and document
lives in one `media` table, and entities reference it by foreign key.

```
programs.cover_media_id     team_members.photo_media_id
posts.cover_media_id        gallery_items.media_id
settings.qr_media_id        resources.file_media_id
pages.media_ids[]           partners.logo_media_id
```

*Why:* alt text, captions, dimensions and publication state then live once, on the media row, and
correcting them corrects every page at once. A path column forces the same description to be
written at each use, which in practice means written once and left blank the other three times.

*Evidence for what this costs when the UI is deferred:* one project built the entire foreign-key
schema in its first phase and deferred the library **screen** for four more. With no browser, the
picker's only control was a file input, so the sole way to fill a media column was to upload a new
file. **The reuse the whole design exists for was unreachable from every screen that needed it.**
Plan the picker in the same phase as the columns, or the data model is unusable.

*Consequence for admin forms:* they write a media **ID** into the form field. If a path is going
into a column, the architecture has been reverted.

## 2. Storage columns hold paths, and one helper makes URLs

**Rule.** The `media` table's `storage_path` holds a path, never a URL. A URL is produced only by
one helper, and **private buckets stay out of that helper's type union.**

```ts
export type PublicBucket = 'media';
//  `files` is private and deliberately absent — it needs a signed URL.

export function publicUrl(bucket: PublicBucket, path: string | null): string | null
```

*Why:* URLs in the database calcify a hostname and a scheme into your content, and one helper
gives you a single place to change when the CDN does.

*Evidence:* with the private bucket absent from the union, linking a private document from a
public page is a **type error** rather than something a reviewer has to notice. The type system is
doing security work, cheaply and permanently.

Have the helper return `null` for an empty path, so callers branch to a placeholder rather than
rendering a broken image.

## 3. Alt text lives on the media row, and is never enforced

**Rule.** Alt text and captions are columns on `media`, translatable if the `i18n` module is on.
A missing one is **warned about, never blocked**.

*Why:* refusing to save an image without alt text means refusing an upload from a phone in the
field. A photograph that arrives without a description is a cost; one that does not arrive is a
lost record of the work.

Show the warning on the card and repeat it in the picker under the selected image, so the gap is
visible where someone can fix it.

If `i18n` is on: alt text is JSONB like every other translatable field, and **empty values are
stripped rather than stored as `""`** — otherwise the fallback never fires and the reader gets a
blank `alt` attribute instead of the other language.

## 4. Budget for egress, not storage

**Rule.** Set a per-image byte ceiling in the plan, derived from the hosting tier, and shrink
images in the browser before upload.

**What the ceiling protects depends on the stack.** On `supabase`, it protects the transfer
allowance, and everything below applies in full. On `cloudflare`, object-storage egress is free and
images are copied into the build and served as static assets, so the ceiling protects **page weight**
— LCP on a phone on a slow network — rather than a bill; derive it from the performance target
instead. Shrink in the browser on both.

The arithmetic that matters:

```
uploaded size × views = the bill
```

*Why:* on a typical free tier, storage is generous and **transfer is the allowance that runs
out**. A few hundred photographs will never fill a gigabyte of storage; one 2 MB hero image viewed
2,500 times spends an entire monthly 5 GB transfer allowance.

Two facts make it run out faster than it looks, and both are easy to miss:

- there may be **no `srcset` and no image processing** on remote images, so a file is delivered at
  exactly the bytes it was uploaded at, with no smaller variant to fall back on; and
- the browser fetches it from the storage host directly, so the CDN's unmetered bandwidth absorbs
  none of it.

Nothing reports this in advance. The failure mode of exhausted egress is the site's images quietly
ceasing to load.

**Derive the ceiling, do not guess it:** (monthly transfer allowance) ÷ (expected page views) ÷
(images per page), minus room for HTML and fonts. Write the derivation into the plan so the number
can be re-derived when traffic changes.

**Where the shrinking runs.** In the browser, and a server endpoint would be worse than useless:
it protects nothing — access control decides who may write either way — and the obvious server
implementation cannot exist, because the production runtime has no `document`, no `Image` and no 2D
canvas. The
browser is the only machine in the request path with a layout engine, and the file is already in
its memory.

**Three kinds of file pass through untouched**, each because re-encoding could only do harm:

- **SVG** is text and already small; rasterising throws away its one property.
- **An animated GIF** would come back as frame one — data loss disguised as a size win. Scan the
  bytes for a second Graphic Control Extension rather than trusting the MIME type, which says
  `image/gif` either way.
- **Anything already small enough**, which is the rule that keeps a payment QR code bit-identical:
  it is the one image where a compression artefact costs real money, and there were no bytes there
  to save.

**How the shrinking runs without a client framework.** Decode with `createImageBitmap` (it follows
EXIF orientation), draw to a canvas, `toBlob('image/webp')`, and put the result back into the file
input through a `DataTransfer`, so the form submits normally and server-rendered error pages keep
working — no `fetch`, no upload API. Re-encoding drops EXIF, including GPS location. The admin CSP
needs `img-src blob:` for the before/after preview. Some formats decode only in some browsers (HEIC
in Safari); fall back to the original.

**The server trusts nothing the browser computed.** It re-checks type, size and dimensions by reading
the file's own header, and hashes the bytes (SHA-256) to detect duplicates. *Evidence:* a canvas-made
JPEG placed into the input with `DataTransfer` and a `change` event exercises the whole optimiser in
a browser test; in a Node test, a minimal valid image header is enough to pass the server's checks.

**Nothing in that path may refuse an upload for a reason that is not the editor's problem.** A file
that will not decode, a browser without `createImageBitmap`, an encoder that returns something
*larger* than the original — all fall back to uploading the file as chosen. Check a hard input cap
first, so the fallback can never be unbounded. A result still over the ceiling uploads with a
warning rather than being blocked.

Record the byte size in a column. The quota that binds is the one nothing in the schema recorded.

## 5. Deleting media needs a "used in" answer first

**Rule.** Build the reverse index from every `*_media_id` foreign key, and show it on the card,
before shipping a delete control.

*Why:* it answers "is this still needed?" at the moment of deleting, which is the only moment
anybody asks.

**Name the cascade explicitly in the confirmation.** If one reference is `not null … on delete
cascade` while every other detaches and leaves an empty slot, those are two different outcomes and
the dialog must say which one it is. Boilerplate confirmation text hides exactly this.

## 6. The default that protects consent also hides the photograph

**Rule.** If uploads default to `published = false`, say so in the picker.

*Why:* an unpublished-by-default media row is right for consent — nobody's photograph goes public
by accident — and it means an editor who uploads an image and selects it sees nothing appear on
the site, with no error anywhere. Both halves are invisible. Make the state visible on the card.

## 7. Documents: compress in the browser, check on the server, serve through a route

Applies when editors upload PDFs or other documents.

**Rule.** Compress PDFs in the browser only when one is chosen, strip their metadata, and keep the
original if compression makes it larger. On the server, refuse anything that is not a real PDF,
exceeds the ceiling, or contains active content. Serve documents through a route rather than a
public bucket.

*Evidence:*

- **A WebAssembly Ghostscript build** in a Web Worker does the compression (its "ebook" preset keeps
  text sharp). It is large (around 16 MB) and AGPL-licensed — load it only in the admin, only when a
  PDF is chosen, and record the licence decision. Running it in a worker script, which is a static
  asset, needs no `wasm-unsafe-eval` in the page CSP.
- **Metadata stripping is incomplete by default.** The rewrite copies author, title, keywords, XMP
  and bookmarks — including a bookmark's JavaScript action. Drop outlines and empty the document info
  explicitly, then test the arguments against a PDF built with those fields before trusting them.
- **Scanning for scripts by bytes needs care.** `/JS` alone matches random binary about once per
  5 MB; look for `/S /JavaScript`, `/Launch`, `/EmbeddedFiles`, `/RichMedia` and `/XFA` after
  decoding `#xx` name escapes and inflating compressed object streams. **Refuse what you cannot read**
  (unknown filters, encryption) rather than pass it.
- **Serving through a route** (`/dl/<id>`) means hiding or replacing a file needs no rebuild, needs no
  public bucket, and can count downloads per day without recording anything about the person.

## 8. Images on the page

**Rule.** Check the format the image component actually falls back to, and give anchor targets room
below sticky headers.

*Evidence:*

- Astro's `<Picture>` defaulted its fallback `<img>` to **PNG** for a WebP source — a 2 MB file for a
  photo. Set `formats={['avif']}` and `fallbackFormat="webp"`. A `class` passed to `<Picture>` lands
  on the `img`, not the `picture`; wrap it for layout.
- **Lazy images move anchor links.** A `#section` link lands correctly only if the images above it
  have loaded; `loading="lazy"` images load later and push the heading away. Give anchors
  `scroll-margin-top` below any sticky header, and do not lazy-load images above common anchors.
