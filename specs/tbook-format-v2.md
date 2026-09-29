# `.tbook` File Format Specification — version 2

**Format version:** 2
**Status:** Candidate (ratified 2026-09-26, second round 2026-09-28)
**Last updated:** 2026-09-28
**Media type:** `application/vnd.tbook+zip`
**File extension:** `.tbook`
**License:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
**Relation to version 1:** a redesign, not an extension. Version 1
([`tbook-format.md`](tbook-format.md)) stays valid and is not deprecated by this
document; a version-2 file is not readable by a version-1 consumer and vice versa
([§10](#10-versioning-and-compatibility)).

> **Release status.** Candidate. The author ratified every open choice of the
> draft on 2026-09-26 ([Ratified decisions](#ratified-decisions-2026-09-26)) and
> a second round of implementation questions on 2026-09-28
> ([Ratified decisions (2026-09-28)](#ratified-decisions-2026-09-28)).
> The reference converter writes version 2 by default (`--format v1` remains
> available); v2-capable readers open both versions. The status becomes Final
> after v2-capable Android and desktop releases ship; the copy of this
> specification on the website is published only then.

> **Background.** This document specifies the direction adopted by the 2026-09
> format review (panel report §5.1: the `v2-json` container and link model, the
> paragraph text model and learner slots of `paragraph-multigranular`, the
> `source` block of `epub-overlay`, and the seven preconditions of §4.3). Where
> the panel left a choice open the draft made one; each is recorded, with its
> outcome, under [Ratified decisions](#ratified-decisions-2026-09-26). Numbers
> appear only where they justify a requirement, each with its report section.

---

## 1. Introduction

A `.tbook` file is a self-contained, offline-first container for a
**language-learning ebook**. It pairs the original text of a book with a
**per-sentence translation** into one or more target languages, plus
**word-level alignment** between source and translation.

The defining feature is unchanged from version 1: **tap-to-translate**. A reader
displays the book in its source language; when the user taps a source word, the
reader shows the sentence's translation with exactly the translated word(s)
highlighted. Everything needed for this is in the file.

Version 2 changes *how* that data is laid out and *what a consumer can trust*:

| | Version 1 | Version 2 |
|---|---|---|
| Canonical text object | the sentence (`src` string) | the **paragraph** (`text` string); sentences are code-point ranges into it ([§4.5](#45-text-and-sents)) |
| Languages | inline in every sentence (`tr` map) | one **overlay** entry per (chapter, language); the chapter **skeleton** is language-free ([§2.1](#21-archive-layout)) |
| Tappable words | producer-emitted char ranges (`words`) | derived by a **normative tokenizer** `tbook-w2` on both sides ([§5](#5-tokenizer-tbook-w2)); an explicit inventory only where the tokenizer does not fit |
| Alignment | chunk objects `{t:[a,b], w:[…]}` with char ranges | **one link per target token** — `int`, `int[]`, `null` (unaligned) or `[]` (inserted) — plus an exact escape list `x` for spans that are not whole tokens ([§6.4](#64-a--links), [§6.5](#65-x--escape-chunks)) |
| Quality | `q` coverage score (informative, blind to correctness) | producer-written **status** `s`, **verdict** `v`, overlay-level **`gates[]`**, and an **alignment-pair digest** ([§6.7](#67-s--status), [§6.8](#68-v-and-gates--verdicts), [§9.2](#92-aligndigest--the-alignment-pair-digest)) |
| Identity | ordinal chapter ids `ch1`, `ch2` | **content-derived** chapter and paragraph ids, locators, `source` identity block ([§3.5](#35-spine--chapter-index), [§4.3](#43-paragraph-id), [§3.6](#36-source--source-identity)) |
| Offsets | "character offsets" (code points in practice) | **code points, NFC**, declared in the manifest ([§7](#7-offsets-and-normalization)) |
| Container | ZIP of JSON | ZIP of JSON with `mimetype` sentinel, per-entry SHA-256 `digests`, shipped JSON Schema, `requires[]` capability list ([§2](#2-container), [§3.2](#32-formatversion-and-requires)) |
| Learner payload | none above/below the word | optional **`units`** (source expressions), **`groups`** (target lexemes), **`parts`** (sub-token spans), and a 0-byte **derived run/split labelling** rule ([§4.9](#49-units), [§6.10](#610-groups), [§6.11](#611-parts), [§8.3](#83-derived-labelling-runs-and-splits)) |

### 1.1 Design model: source as pivot, paragraph as text, sentence as range

A `.tbook` is **multi-language by design**. The **source language is the pivot**;
every target ("gloss") language aligns word-by-word *to the source*, never to
another target. Adding a language costs one translation per sentence (N−1) and,
in version 2, is an **append** to the archive rather than a rewrite
([§2.4](#24-appending-a-language)).

The **paragraph is the canonical text object**: a skeleton stores each
paragraph's text once and sentences are ranges into it, so segmentation is a
*view* — a boundary may fall mid-word and the paragraph still renders, because
the consumer displays `text`, not a join of sentences ([Annex A.7](#a7-worked-example-s9)).
Alignment is **by meaning**, from every target token to the source words it
renders, produced *during* translation; the consumer has no aligner.

### 1.2 Terminology

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHOULD**, **SHOULD NOT**,
**MAY** and **OPTIONAL** are to be interpreted as described in RFC 2119.

| Term | Meaning |
|------|---------|
| **Source** | The original language of the book. One per file. |
| **Target** / **gloss** | A language the book is translated into. One or more per file. |
| **Skeleton** | The language-free chapter entry `text/chN.json` ([§4](#4-skeleton--textchnjson)). |
| **Overlay** | The per-(chapter, language) entry `gloss/chN.<lang>.json` ([§6](#6-overlay--glosschnlangjson)). |
| **Block** | A paragraph-like text object: a chapter paragraph, a table cell, or a footnote paragraph. Carries `text` and `sents`. |
| **Sentence** | A code-point range `[a, b)` into a block's `text`; the unit of translation. |
| **Word** | A **source** token produced by the tokenizer (or the block's `words` override) whose first code point lies in a sentence; identified by its 0-based index within that sentence. |
| **Token** | A **target** token of a translation `t`, produced by the same tokenizer (or the translation's `words` override); identified by its 0-based index. |
| **Link** | The `a[j]` entry of target token `j`: the source word index(es) it renders, `null`, or `[]`. |
| **Escape chunk** | An entry of `x`: a code-point range of `t` that is not exactly one token, with its source word indices. |
| **Unit** | A producer-asserted multi-word expression on the **source** side, shared by all languages ([§4.9](#49-units)). |
| **Group** | A producer-asserted set of target tokens forming one lexeme ([§6.10](#610-groups)). |
| **Part** | A producer-asserted sub-token span of a target token with its own link ([§6.11](#611-parts)). |
| **Status** `s` | A producer-written per-translation state ([§6.7](#67-s--status)). |
| **Gate** / **verdict** `v` | A quality check that ran over an overlay, and its per-translation result ([§6.8](#68-v-and-gates--verdicts)). |
| **Locator** | `chapterId/paragraphId/sentenceIndex/wordIndex` — a stable reference to a word ([§3.5.2](#352-locators)). |
| **Producer** | A tool that writes `.tbook` files. |
| **Consumer** | A tool that reads `.tbook` files. |

---

## 2. Container

A `.tbook` file **MUST** be a ZIP archive (PKZIP, APPNOTE 6.3.x). Rules beyond
plain ZIP:

- Every entry **MUST** use compression method `0` (STORED) or `8` (DEFLATE). No
  other method, no encryption, no ZIP64 structures (a file needing ZIP64 is out
  of scope).
- Entry names **MUST** be unique within the archive, use `/` as separator, be
  case-sensitive, and be valid UTF-8 without a leading `/` or any `..` segment.
  A consumer **MUST** reject an archive whose central directory lists the same
  name twice.
- Every JSON entry **MUST** be UTF-8 without a byte-order mark and **MUST** be
  Unicode NFC ([§7](#7-offsets-and-normalization)).
- The first local file entry **MUST** be `mimetype` ([§2.2](#22-the-mimetype-entry)).
- `manifest.json` **MUST** be the **last** local file entry, immediately
  followed by the central directory ([§2.4](#24-appending-a-language)).
- Entries **MUST** be addressable by name via the central directory; consumers
  **MUST NOT** depend on local-header order for anything except the two rules
  above.

### 2.1 Archive layout

```
book.tbook  (ZIP)
├── mimetype                       required — first entry, STORED (§2.2)
├── schema/tbook-2.schema.json     required — JSON Schema of every JSON entry (§3.9)
├── cover.jpg                      optional — cover image bytes
├── images/
│   └── img1.jpg                   optional — body images referenced by figures (§4.10)
├── text/
│   ├── ch1.json                   required (≥1) — chapter skeleton, language-free (§4)
│   ├── ch2.json
│   └── notes.json                 optional — footnote bodies, skeleton shape (§4.12)
├── gloss/
│   ├── ch1.ru.json                one per (chapter, language) — overlay (§6)
│   ├── ch1.tr.json
│   ├── ch2.ru.json
│   └── notes.ru.json              optional — footnote bodies, overlay shape (§6.12)
└── manifest.json                  required — LAST entry: metadata, spine, digests (§3)
```

Entry names other than `mimetype` and `manifest.json` are **conventional**: a
consumer **MUST** discover skeletons, overlays, notes and the schema only through
the manifest (`spine[].text`, `spine[].gloss`, `notes`, `schema`), never by
pattern. Producers **MAY** add entries; consumers **MUST** ignore entries the
manifest does not reference. The `chN` numbering is the reading order and
carries no identity — identity is the content-derived `spine[].id`
([§3.5.1](#351-chapter-id)).

### 2.2 The `mimetype` entry

The first local file entry **MUST** be named `mimetype`, **MUST** be STORED, and
**MUST** contain exactly the 25 ASCII bytes `application/vnd.tbook+zip` — no
trailing newline, no byte-order mark. Its local header **MUST** carry no extra
field, so that the string starts at byte offset 38 of the file (the same
sniffing rule as EPUB OCF).

A consumer **MAY** identify a `.tbook` by these 63 leading bytes before opening
the ZIP. A consumer **MUST NOT** rely on the file extension.

### 2.3 Compression

Producers **SHOULD** DEFLATE JSON entries and STORE already compressed media.
Consumers **MUST** support both methods and **MUST** read entries by random
access; whole-archive streaming need not work.

### 2.4 Appending a language

Adding a target language that the file does not yet have is an **append**,
defined as follows (a language already listed in `languages.targets` is not
appended; it is replaced per [§2.5](#25-replacing-a-language)):

1. Truncate the archive at the local-header offset of the existing
   `manifest.json` (which, by [§2](#2-container), is the last local entry; this
   discards only the manifest and the central directory).
2. Append the new `gloss/<chN>.<lang>.json` entries (and `gloss/notes.<lang>.json`
   if the book has footnotes).
3. Append a rewritten `manifest.json` with the language added to
   `languages.targets` and to each `spine[].gloss`, the new entries added to
   `digests`, and a new `meta.runs[]` record ([§3.10](#310-meta--provenance)).
4. Write a new central directory and end-of-central-directory record.

The skeleton entries are untouched, which is possible only because every
skeleton field — including `units` — is derived from the source alone
([§4.9](#49-units); report §4.3, M10: with language-dependent units 0 of 13
skeletons survived the append). A producer **MUST NOT** leave a superseded
`manifest.json` in the archive: after the append the name **MUST** occur exactly
once ([§2](#2-container)).

<a id="append-nonconformant-input"></a>**Non-conformant input.** If
`manifest.json` is not the last local entry of the input, or the central
directory lists it more than once, step 1 would discard or keep the wrong
bytes. A producer **MUST** then refuse to append, and its error **MUST** name
the entry that follows the manifest (for a duplicate, the other occurrence).
Rewriting the whole file into a conformant archive is a separate operation that
the user requests explicitly; a producer **MUST NOT** perform it silently as
part of an append.

The overlay entry names of an append, and the rules for `meta`, are those of
[§2.5](#25-replacing-a-language) and [§3.10](#310-meta--provenance).

<a id="25-replacing-a-language"></a>
### 2.5 Replacing a language

A producer **MAY** replace a target language that is already present — for
example to re-translate it with another model. Replacement is **not** an append
([§2.4](#24-appending-a-language)): the old overlays sit before the manifest, so
the producer rewrites the archive. It **MUST** do so as follows:

1. Every entry that is not a replaced overlay (step 2) — `mimetype`,
   the schema, the cover, images, every skeleton, `text/notes.json`, every
   overlay of every other language, every overlay of the replaced language
   for a chapter the run does not cover (step 3), and every entry the
   manifest does not reference — **MUST** be written byte-identical to the input (same name,
   same compression method, same uncompressed and compressed bytes), in the
   input's local-entry order. `mimetype` stays the first entry and STORED
   ([§2.2](#22-the-mimetype-entry)).
2. The overlay entries of the replaced language for the chapters the run
   covers (`spine[].gloss[lang]`) and its footnote overlay
   (`notes.gloss[lang]`) are removed and the new overlays written in their
   place; an overlay for a chapter that had none in that language is written
   after the last carried entry.
3. A replacement **MAY** be partial: only the chapters the run covers get new
   overlays. A chapter the run does not cover **MUST** keep its existing
   overlay for the replaced language, copied verbatim like any entry of step 1,
   with its `spine[].gloss` key and its digest unchanged. A chapter that had no
   overlay in that language and is not covered still has none, which is legal
   ([§3.4](#34-languages), decision 10).
4. `manifest.json` is rewritten as the **last** entry, exactly once: the
   `digests` of the replaced overlays are dropped and those of the new ones
   added, every other digest unchanged; `languages.targets` keeps its order;
   each new overlay carries its own `alignDigest` ([§9.2](#92-aligndigest--the-alignment-pair-digest)).
   `meta` follows [§3.10](#310-meta--provenance): the replaced language's
   earlier `meta.runs[]` records stay and a new record is added, whose
   `targets` lists the replaced language.
5. A new central directory and end-of-central-directory record are written. The
   ZIP archive comment is **not** preserved.

<a id="overlay-entry-names"></a>**Overlay entry names.** An appended or
replacing overlay is named after its skeleton: `text/X.json` →
`gloss/X.<lang>.json`; the footnote overlay is `gloss/notes.<lang>.json`. The
names remain conventional for consumers ([§2.1](#21-archive-layout)).

The non-conformant-input rule of [§2.4](#append-nonconformant-input) applies to
a replacement too. A single run **MAY** replace some languages and add others;
it is then a replacement (a rewrite) as a whole.

---

## 3. `manifest.json`

A single JSON object describing the book, its languages, its chapters, and the
integrity of every other entry. The example is the one-chapter fixture book of
[Annex B](#annex-b--conformance-tests):

```json
{
  "formatVersion": 2,
  "requires": ["text/paragraph", "align/token", "tokenizer/tbook-w2"],
  "title": "Canonical sentences",
  "authors": ["Arthur Conan Doyle"],
  "languages": { "source": "en", "targets": ["ru", "tr", "de"] },
  "text": { "tokenizer": "tbook-w2", "offsets": "codepoint", "normalization": "NFC" },
  "cover": null,
  "notes": {},
  "spine": [
    {
      "id": "cbedb517c",
      "title": "Canonical sentences",
      "level": 0,
      "text": "text/ch1.json",
      "gloss": { "ru": "gloss/ch1.ru.json", "tr": "gloss/ch1.tr.json", "de": "gloss/ch1.de.json" },
      "paragraphs": 11,
      "sentences": 15
    }
  ],
  "source": {
    "kind": "epub",
    "sha256": "8f4e3c1a9b2d7e6f5a4b3c2d1e0f9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f",
    "identifier": "urn:uuid:5c9f1b6e-2d4a-4e8b-9c3f-7a1d2e3f4b5c",
    "textSha256": "1d2c3b4a5f6e7d8c9b0a1f2e3d4c5b6a7f8e9d0c1b2a3f4e5d6c7b8a9f0e1d2c",
    "docs": [ { "href": "OEBPS/ch01.xhtml", "chars": 4180, "sha256": "a1b2c3d4e5f60718293a4b5c6d7e8f9012345678a1b2c3d4e5f60718293a4b5c" } ]
  },
  "schema": "schema/tbook-2.schema.json",
  "digests": {
    "schema/tbook-2.schema.json": "sha256:2c4e6a8b0d1f3a5c7e9b1d3f5a7c9e1b3d5f7a9c1e3b5d7f9a1c3e5b7d9f1a3c",
    "text/ch1.json": "sha256:6d345441cb4fa35e2e1c71ccf0b4a12874c2dfd3679cbcd52cfec965dc21e141",
    "gloss/ch1.ru.json": "sha256:9e8d7c6b5a4f3e2d1c0b9a8f7e6d5c4b3a2f1e0d9c8b7a6f5e4d3c2b1a0f9e8d"
  },
  "meta": {
    "createdAt": "2026-09-14T10:00:00Z",
    "runs": [ { "at": "2026-09-14T10:00:00Z", "producer": { "name": "tbook-converter", "version": "2.0.0" },
                "targets": ["ru", "tr", "de"], "options": { "tokenizer": "tbook-w2", "lexcheck": true } } ]
  }
}
```

(Hex values are layout placeholders except `digests["text/ch1.json"]`, the real
digest of the [Annex C](#annex-c--complete-minimal-example) skeleton; `digests`
is shown truncated.)

### 3.1 Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `formatVersion` | integer | **Yes** | **MUST** be `2`. Checked before any other field ([§10.1](#101-formatversion)). |
| `requires` | array of string | **Yes** | Feature ids a consumer **MUST** implement to open the file ([§3.2](#32-formatversion-and-requires)). |
| `title` | string | **Yes** | Book title. **MAY** be empty. |
| `authors` | array of string | **Yes** | Authors in display order. **MAY** be empty. |
| `languages` | object | **Yes** | `{ "source": tag, "targets": [tag, …] }` ([§3.4](#34-languages)). |
| `text` | object | **Yes** | Tokenizer, offset unit and normalization declaration ([§3.3](#33-text--tokenizer-offsets-normalization)). |
| `cover` | string \| null | No | Entry name of the cover image; absent or `null` = no cover. Format detected from bytes, never from the extension. |
| `notes` | object | **Yes** | Footnote entries: `{}` when the book has no footnotes, else `{ "text": entry, "gloss": { lang: entry } }` ([§3.7](#37-notes)). **Always an object**, never a string or `null` ([§10.4](#104-version-1-consumers-opening-a-version-2-file)). |
| `spine` | array of `ChapterRef` | **Yes** | Reading order; **MUST** contain ≥ 1 entry ([§3.5](#35-spine--chapter-index)). |
| `source` | object | No | Identity of the source document the book was converted from ([§3.6](#36-source--source-identity)). |
| `schema` | string | **Yes** | Entry name of the shipped JSON Schema ([§3.9](#39-schema--the-shipped-json-schema)). |
| `digests` | object | **Yes** | `entryName → "sha256:<64 lowercase hex>"` for every entry except `mimetype` and `manifest.json` ([§9.1](#91-digests--entry-integrity)). |
| `meta` | object | No | Provenance and service metadata; carries no format semantics ([§3.10](#310-meta--provenance)). |

Consumers **MUST** ignore unknown manifest fields.

### 3.2 `formatVersion` and `requires`

`formatVersion` is the on-disk format major version and is a constant `2` for
every file conforming to this document. A consumer that does not implement
version 2 **MUST** refuse the file ([§10](#10-versioning-and-compatibility)).

`requires` lists **feature ids** a consumer must implement to render the file
correctly. A consumer **MUST** refuse a file whose `requires` contains an id it
does not implement, and **SHOULD** show that id to the user. Registered ids:

| Feature id | Meaning | In version 2.0 files |
|---|---|---|
| `text/paragraph` | Paragraph text with sentence ranges ([§4.5](#45-text-and-sents)). | **MUST** be listed |
| `align/token` | Per-target-token links with `x` escapes and `words` overrides ([§6.4](#64-a--links)–[§6.6](#66-words--target-token-inventory-override)). | **MUST** be listed |
| `tokenizer/tbook-w2` | The tokenizer of [§5](#5-tokenizer-tbook-w2). | **MUST** be listed |

Rules:

- A producer **MUST** list exactly the ids whose absence in a consumer would
  produce a *wrong* rendering or wrong highlights. Purely additive slots that a
  consumer can ignore without harm (`units`, `groups`, `parts`, `source`,
  `spans`, `notes`, `figure`, `table`, `s`, `v`) **MUST NOT** be listed.
- A future revision that introduces a mandatory feature **MUST** register a new
  id here; it **MUST NOT** change the meaning of an existing id.
- Ids are compared byte-for-byte; unregistered ids **MUST** use an `x-` prefix
  or a reverse-DNS name.

### 3.3 `text` — tokenizer, offsets, normalization

```json
{ "tokenizer": "tbook-w2", "offsets": "codepoint", "normalization": "NFC" }
```

| Field | Value | Meaning |
|---|---|---|
| `tokenizer` | `"tbook-w2"` | The tokenizer applied to every block `text` and every translation `t` that carries no `words` override ([§5](#5-tokenizer-tbook-w2)). Under version 2 this is the only registered value; `requires` repeats it as `tokenizer/tbook-w2`. |
| `offsets` | `"codepoint"` | The unit of every offset in the file — `sents`, `words`, `spans`, `notes[].p`, `x`, `parts` ([§7](#7-offsets-and-normalization)). Constant. |
| `normalization` | `"NFC"` | Every string in every JSON entry is NFC. Constant. |

These declare rules that are mandatory anyway, so that a validator and a future
version can see what a file assumes.

### 3.4 `languages`

```json
{ "source": "en", "targets": ["ru", "tr", "de"] }
```

Language codes are BCP 47 tags; bare lowercase ISO 639-1 is the common case.
Consumers **MUST** treat tags as opaque, case-sensitive keys matched exactly
against `spine[].gloss`, overlay `lang` and `notes.gloss` keys. `targets` is
the display order, **SHOULD** be non-empty, and **MUST NOT** contain `source`
or duplicates. A listed target **SHOULD** have an overlay in every `spine`
entry; where one is missing (a partial conversion) the consumer **MUST** render
the source text with no translation, and a validator **SHOULD** warn.

### 3.5 `spine` — chapter index

`ChapterRef`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **Yes** | Content-derived chapter id ([§3.5.1](#351-chapter-id)). Unique within the file. |
| `title` | string | **Yes** | Navigation title. Producer fallback: `"Chapter N"`. |
| `level` | integer ≥ 0 | No | Nesting depth for a hierarchical table of contents. Default `0`. |
| `text` | string | **Yes** | Entry name of the skeleton ([§4](#4-skeleton--textchnjson)). |
| `gloss` | object | **Yes** | `lang → entry name` of the overlays ([§6](#6-overlay--glosschnlangjson)). **MAY** be empty. |
| `paragraphs`, `sentences`, `words` | integer | No | Counts, so a consumer can weight reading progress by text rather than by bytes. Informative. |

The chapter title is *also* re-emitted by the reference producer as the
skeleton's first `heading`-role paragraph, so it is translated and tappable. A
consumer that renders `ChapterRef.title` above the text **SHOULD** skip its own
copy when the skeleton opens with a `heading` paragraph.

#### 3.5.1 Chapter id

The chapter id is derived from content that re-conversions of the same book
leave untouched. Measured on the two real re-conversion pairs in the reference
corpus, a hash over *all* paragraph texts survived 0 of 58 chapters (one
heading paragraph is inserted per chapter by the current producer), while the
recipe below survived 12 of 12 and 45 of 46 (report §4.3, verification). The
recipe is therefore **normative** so that two producers emit the same id for the
same chapter:

1. `normTitle` = the chapter title, NFC-normalized, with every maximal run of
   Unicode `White_Space` characters replaced by one U+0020, leading and
   trailing whitespace removed, then full Unicode case folding applied
   (`toCasefold`).
2. `body₁ … bodyₙ` = the `text` of the first **three** paragraphs of the
   skeleton whose role is `body` (fewer if the chapter has fewer), NFC, verbatim.
3. `bytes` = UTF-8 of `normTitle`, then for each `bodyᵢ`: U+000A followed by
   UTF-8 of `bodyᵢ`. (No trailing newline.)
4. `id` = `"c"` + the first 8 lowercase hex digits of SHA-256(`bytes`).
5. If the result equals an id already assigned to an earlier `spine` entry,
   append `-2` for the second occurrence, `-3` for the third, and so on, in
   spine order.

The skeleton entry **MUST** repeat the id in its top-level `id` field, and every
overlay of the chapter **MUST** repeat it in `chapter` ([§6.1](#61-overlay-object-and-binding));
a mismatch is a binding error and the consumer **MUST** reject the entry.

Example ([Annex B](#annex-b--conformance-tests)): title `Canonical sentences`
→ `normTitle` = `canonical sentences`; body paragraphs S1, S2, S3 →
`id` = `cbedb517c`.

#### 3.5.2 Locators

A **locator** names a word: `chapterId/paragraphId/sentenceIndex/wordIndex`,
e.g. `cbedb517c/04537723/0/3` is `wait` in S1. Shorter locators name a
sentence (`…/0`), a paragraph, or a chapter. Consumers **SHOULD** store reading
positions and bookmarks as locators, never as array indices.

<a id="locator-precision"></a>**Precision.** A consumer that anchors reading
positions by paragraph **MAY** store the paragraph-precision locator
`chapterId/paragraphId/0/0`; it is conformant and denotes the start of that
paragraph. A chapter-only locator `chapterId` denotes the position **before the
first paragraph** of the chapter — the cover or title position of a chapter
that opens with one.

**Resolution rule (normative).** To resolve a locator: (1) find the spine entry
with the chapter id; (2) within its skeleton find the paragraph id; (3) index
the sentence and word. If step (1) fails — the chapter id is unknown, as after a
re-conversion whose title or opening paragraphs changed — the consumer **MUST**
fall back to searching the paragraph id **across all chapters** and resolve to
the first match in spine order. Paragraph ids survive re-conversion far more
often than any chapter-level recipe (99.40 % / 99.96 % on the same two pairs,
report §4.3), which is why the fallback is mandatory and not advisory. A locator whose chapter resolves but whose paragraph does not **SHOULD** degrade to the first paragraph of that chapter — the chapter start, including its title (a leading `heading` paragraph, [§3.5](#35-spine--chapter-index)); one whose chapter is unknown too degrades to the book start. It **MUST NOT** resolve to a wrong word.

### 3.6 `source` — source identity

An **optional** block recording which source document — and which *edition* of
it — the book was converted from. It is the only mechanism that can say whether
a gloss belongs to the text a reader holds.

| Field | Type | Description |
|---|---|---|
| `kind` | string | `"epub"`, `"fb2"`, or another producer-defined kind. |
| `sha256` | 64 hex | SHA-256 of the source file bytes — exact-file identity. |
| `identifier` | string | The source's own identifier (`dc:identifier` for EPUB), if any. |
| `title`, `language` | string | The source's declared title and language, if any. |
| `textSha256` | 64 hex | **Edition fingerprint** that survives repackaging ([recipe below](#textsha256-recipe)). |
| `docs` | array | One record per source document in reading order: `{ "href", "chars", "sha256", "title"? }` — `sha256` over the document's text stream, `chars` its code-point length. |

<a id="textsha256-recipe"></a>**`textSha256` recipe.** `stream(k)` = the text
nodes under document `k`'s body concatenated in document order (descendants of
`script`, `style`, `template` excluded), line endings normalized to U+000A, no
other change. `docs[k].sha256` = SHA-256(UTF-8(`stream(k)`)). `textSha256` =
SHA-256 of UTF-8 of NFC(`stream(k)`) with every `White_Space` code point removed,
for all `k` in order, joined by U+0000 — so the same text in an EPUB 2 and an
EPUB 3 wrapper has one `textSha256` and two `sha256`s.

`source` is informative: a consumer **MUST NOT** require it, and **MAY** compare
it with a source file it holds to warn about an edition mismatch.

### 3.7 `notes`

`notes` is **REQUIRED** and is always a JSON object:

- `{}` — the book has no footnotes.
- `{ "text": "text/notes.json", "gloss": { "ru": "gloss/notes.ru.json" } }` —
  footnote bodies in skeleton shape ([§4.12](#412-footnote-bodies--textnotesjson))
  and one overlay per language ([§6.12](#612-footnote-overlays--glossnoteslangjson)).
  `gloss` keys **MUST** be a subset of `languages.targets`.

The object form is mandatory even when empty so that every version-2 file
presents the *same* shape to a version-1 consumer, whose `notes` field is a
string ([§10.4](#104-version-1-consumers-opening-a-version-2-file)).

### 3.8 `cover`

Optional, as in version 1: when present and non-null the entry **MUST** exist
and hold image bytes; the extension is nominal.

### 3.9 `schema` — the shipped JSON Schema

Every file **MUST** ship, byte-identical, the published JSON Schema (draft
2020-12, `$id` `https://tbook.dev/schema/tbook-2.schema.json`) at the entry
named by `manifest.schema`. Its `$defs` `manifest`, `textChapter`, `textNotes`,
`glossChapter`, `glossNotes` are normative for **shape** (types, required keys,
enumerations, patterns); cross-entry invariants — binding, counts, ranges,
digests — are normative in this prose ([§11](#11-conformance)). Consumers need
not run a schema validator on open; the copy serves tooling and
self-description.

### 3.10 `meta` — provenance

`meta` is unchanged from version 1 §3.4 in shape and rules: an **optional**,
**informative** object with registered keys `createdAt`, `updatedAt`, `runs[]`
(`RunRecord` = `at`, `producer{name, version, commit, url}`, `targets`,
`provider`, `models`, `prompts`, `options`), open to namespaced producer keys.

Additional version-2 rules: `meta` **MUST NOT** carry format semantics —
tokenizer, offset unit, statuses, verdicts and ids live in their normative
fields, and a consumer **MUST NOT** derive rendering or tap behaviour from
`meta`. A run that appends a language ([§2.4](#24-appending-a-language)) or
replaces one ([§2.5](#25-replacing-a-language)) **MUST** carry the existing
`meta` over and append its own record, whose `targets` lists only the added or
replaced language(s). It **MAY** set `meta.updatedAt` to the run's time and
**MAY** cap `meta.runs` to the most recent 20 records (the new one included);
everything else in `meta` — `createdAt`, the remaining records, unknown keys —
**MUST** be carried over unchanged.
`runs[].options.tokenizer` **SHOULD** record the aligner's tokenizer id, which
**MUST** equal `text.tokenizer`. No secrets, no machine identity, keep it small.

---

## 4. Skeleton — `text/chN.json`

The skeleton is the **language-free** chapter: paragraph text, sentence ranges,
roles, emphasis, footnote markers, figures, tables and optional source-side
units. It contains nothing that depends on the set of target languages.

```json
{
  "id": "cbedb517c",
  "paragraphs": [
    { "id": "0f4552b1", "role": "heading", "text": "A SCANDAL IN BOHEMIA", "sents": [[0, 20]] },
    { "id": "04537723", "text": "“Then I can wait in the next room.”", "sents": [[0, 35]] },
    { "id": "781304bb",
      "text": "“‘Oh, at his new offices. He did tell me the address. Yes, 17 King Edward Street, near St. Paul’s.’",
      "sents": [[0, 23], [23, 51], [51, 88], [88, 96], [96, 99]] },
    { "id": "ff1de7b8",
      "text": "“Still, if I had married Lord St. Simon, of course I’d have done my duty by him.",
      "sents": [[0, 80]],
      "spans": [{ "s": 25, "e": 39, "k": "i" }],
      "units": [{ "s": 0, "w": [8, 9], "k": "idiom" }] }
  ]
}
```

### 4.1 Chapter object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **Yes** | The chapter id; **MUST** equal the referencing `spine[].id` ([§3.5.1](#351-chapter-id)). |
| `paragraphs` | array of `Block` | **Yes** | Ordered paragraphs. **MAY** be empty. |

### 4.2 The `Block` object

A block is a paragraph, a table cell, or a footnote paragraph. All three share
this shape.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **Yes** for paragraphs; No for cells | Content-derived paragraph id ([§4.3](#43-paragraph-id)). |
| `role` | string | No | `body` (default), `subtitle`, `heading`, `sceneBreak`, `figure`, `table`. Unknown → `body` ([§4.4](#44-role)). |
| `text` | string | **Yes** | The whole block text, NFC, displayed verbatim. Empty for `sceneBreak` and `table`. |
| `sents` | array of `[a, b]` | **Yes** | Sentence ranges into `text` ([§4.5](#45-text-and-sents)). **MAY** be empty. |
| `words` | array of array of `[a, b]` | No | Per-sentence source word ranges, **only** when the block was not tokenized with `text.tokenizer` ([§4.6](#46-words--source-word-inventory-override)). |
| `spans` | array of `Span` | No | Inline emphasis ([§4.7](#47-spans--inline-emphasis)). |
| `notes` | array of `NoteRef` | No | Footnote markers ([§4.8](#48-notes--footnote-markers)). |
| `units` | array of `Unit` | No | Source-side multi-word units ([§4.9](#49-units)). |
| `figure` | object | No | `{ "image": entry, "alt"?: string }`; only on a `figure` block ([§4.10](#410-figure)). |
| `table` | object | No | `{ "rows": [[Block, …], …] }`; only on a `table` block ([§4.11](#411-table)). |
| `header` | boolean | No | Table cells only: header cell. Default `false`. |

### 4.3 Paragraph id

`id` = the first 8 lowercase hex digits of SHA-256 over UTF-8 of the block's
NFC `text`, verbatim. If the result equals the id of an earlier paragraph **of
the same chapter**, append `-2`, `-3`, … in document order (66 paragraphs of the
reference sample repeat, mostly `* * *` and single-word replies). Ids of
different chapters **MAY** coincide; a locator always carries the chapter id.

Cells and footnote paragraphs **MAY** carry ids under the same recipe (scoped
to the table, or to the note); overlays bind to them by position.

Examples: `“Then I can wait in the next room.”` → `04537723`; the S9 paragraph
→ `781304bb`.

### 4.4 `role`

Roles are presentational hints exactly as in version 1 §4.1: `body`,
`subtitle`, `heading`, `sceneBreak`, `figure`, `table`. A `sceneBreak` or
`table` block **MUST** have `text` `""` and `sents` `[]`. A consumer **MAY**
render any role as body text without losing tap-to-translate; it **MUST**
treat an unknown role as `body`.

### 4.5 `text` and `sents`

`text` is the paragraph as displayed. The producer has already normalized it
(whitespace collapsed, non-breaking spaces converted, trimmed, NFC). A
consumer **MUST** render `text` verbatim and **MUST** treat it as the
coordinate space of `sents`, `words`, `spans` and `notes[].p`
([§7](#7-offsets-and-normalization)).

`sents[k] = [a, b]` is the half-open code-point range of sentence `k`:

- `0 ≤ a < b ≤ length(text)`; ranges **MUST** be sorted and **MUST NOT**
  overlap; gaps between ranges are allowed (inter-sentence whitespace).
- A boundary **MAY** fall anywhere, including inside a word; the consumer never
  joins sentences, so a mid-word cut only affects which sentence a word belongs
  to (next rule).
- **Word inventory.** The words of sentence `k` are the tokens of `text`
  ([§5](#5-tokenizer-tbook-w2)) whose **first code point** lies in `[a, b)`, in
  text order; the index of a word in that list is its **word index**, the value
  every link refers to. A token that starts in one range and ends in the next
  belongs to the range containing its start.
- Every token of `text` **MUST** start inside some sentence range (a token in
  a gap would be untappable); a validator **MUST** report a violation. A consumer that meets one **MUST** attach it to the preceding sentence (to the first sentence when none precedes), so word indices stay stable.

In the S9 paragraph the boundary at 23 cuts `offices` (17–24); the token
belongs to sentence 0 as word 4, sentence 1 (23–51) has words `He did tell me
the address`, and sentence 4 (96–99, `s.’`) has **no** words.

A consumer that interleaves a gloss after each sentence (bilingual mode) **MUST NOT** cut a word on the page: it splices after the last word of the sentence even when that word extends past `sents[k][1]`.

### 4.6 `words` — source word inventory override

When present, `words[k]` is the explicit list of word ranges `[a, b]` (block
coordinates, code points) of sentence `k`, replacing the tokenizer for this
block. It exists for scripts the tokenizer cannot segment (Chinese, Japanese,
Thai — a producer with a segmenter emits it) and for producers whose word
inventory differs from `tbook-w2`.

- `words` **MUST** have exactly `length(sents)` elements; each range **MUST**
  lie inside its sentence, satisfy `a < b`, be sorted and non-overlapping, and
  contain no U+0009 or U+000A.
- A consumer **MUST** use `words` when present and **MUST NOT** re-tokenize the
  block; implementing this override is part of `align/token`.

### 4.7 `spans` — inline emphasis

`{ "s": a, "e": b, "k": "i" | "b" }`, block coordinates, `0 ≤ s < e ≤ length(text)`.
Semantics as version 1 §5.5: spans **MAY** overlap, a consumer unions styles,
ignores unknown `k`, and clamps out-of-range offsets. Independent of words and
alignment.

### 4.8 `notes` — footnote markers

`{ "p": offset, "id": noteId, "label": string }` — `p` is an insertion point
into `text`, `0 ≤ p ≤ length(text)`; `id` is a key of `text/notes.json`; the
marker text is **not** part of `text`. Semantics as version 1 §5.6. An `id`
absent from `text/notes.json` **SHOULD** be ignored.

### 4.9 `units`

A **unit** is a producer-asserted multi-word expression on the **source** side:

```json
{ "s": 0, "w": [8, 9], "k": "idiom" }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `s` | integer | **Yes** | Sentence index within the block. |
| `w` | array of integer | **Yes** | ≥ 2 word indices of that sentence, ascending, **MAY** be discontiguous. |
| `k` | string | No | Kind: `idiom`, `phrasal`, `fixed`, `name`, `number`, `mwe`. Unknown kinds are legal and render as `mwe`. |

Rules:

- A unit **MUST** be derivable from the source text alone (a tagging pass over
  the source, a lexicon, or an editor). It **MUST NOT** depend on any target
  language or on the set of targets present: the skeleton would otherwise be
  rewritten whenever a language is added, and the append of
  [§2.4](#24-appending-a-language) would be impossible (report §4.3, M10).
- Units are an **optional learner slot**. A consumer ignoring them loses only
  the unit label and the union highlight of [§8.4](#84-unit-mode).
- Overlapping units are legal; on a tap, the smallest unit containing the word
  wins.

### 4.10 `figure`

On a `figure`-role block: `{ "image": "images/img1.jpg", "alt": "" }`. The
block's own `text`/`sents` are the caption — translated and tappable like any
paragraph, or empty for a caption-less image. `image` **MUST** name an existing
entry; format is detected from bytes.

### 4.11 `table`

On a `table`-role block (whose `text` is `""`): `{ "rows": [[cell, …], …] }`,
`rows` non-empty. A **cell** is a `Block` (with `text`, `sents`, optional
`header`, `spans`, `notes`, `words`, `units`). Rows **MAY** differ in cell
count; a consumer renders by row. A consumer ignoring tables loses their
content, as in version 1.

### 4.12 Footnote bodies — `text/notes.json`

```json
{
  "n1": { "label": "1", "kind": "note",
          "paragraphs": [ { "id": "ea90ea77", "text": "The two attendants brought up the rear.", "sents": [[0, 39]] } ] }
}
```

A map `noteId → Note`; `Note` = `{ "label": string, "kind"?: "note" | "citation",
"paragraphs": Block[] }`. Unknown `kind` → `note`. Note ids are opaque and
unique within the file. The overlay side is [§6.12](#612-footnote-overlays--glossnoteslangjson).

---

## 5. Tokenizer `tbook-w2`

The tokenizer is **normative for both sides**: it produces the source **words**
of every block and the target **tokens** of every translation, unless a `words`
override is present ([§4.6](#46-words--source-word-inventory-override),
[§6.6](#66-words--target-token-inventory-override)). It is what lets the file
drop every per-word character range: the consumer regenerates them.

### 5.1 Definition

A token is a maximal match of the following regular expression, applied
left-to-right with no overlap, to the NFC text:

```
[\p{L}\p{N}][\p{L}\p{M}\p{N}]*(?:['’\-][\p{L}\p{N}][\p{L}\p{M}\p{N}]*|[.,:]\p{N}[\p{L}\p{M}\p{N}]*)*
```

pinned to the **Unicode 15.1.0** character database for the General_Category
properties `L`, `M`, `N`. The three joiners are U+0027 APOSTROPHE, U+2019 RIGHT
SINGLE QUOTATION MARK and U+002D HYPHEN-MINUS; the three numeric separators are
U+002E, U+002C and U+003A. The expression contains no lookaround, no
backreference and no case-insensitive flag, so RE2 (Go), `java.util.regex`,
JavaScript `/u`, Rust `regex` and Python `regex` implement it identically.

In words: a token starts with a letter or a number and continues with letters,
marks and numbers; an apostrophe or hyphen joins two such runs (`don’t`,
`Bow-Street-Zellen`, `этом-то`, `l'avons`); `.`, `,` and `:` join two runs only
when the right-hand run starts with a digit (`30,000`, `3.14`, `5:15`).
Whitespace, punctuation and symbols — including `£`, `§`, `%` — are never part
of a token.

### 5.2 Properties

- **Digits are tappable.** `4` and `17` in S4/S9 are words; `£` is not.
- **Superset of the version-1 word.** Every version-1 word lies inside exactly
  one `tbook-w2` token (verified on 995,021 taps, [Annex A.3](#a3-links)).
- **Version pin.** `tbook-w2` is frozen. Any change to the expression or to the
  Unicode version is a **new tokenizer id** and a new `requires` entry; it
  **MUST NOT** be made under the name `tbook-w2`, because target token indices
  in every overlay depend on it.
- **Platform tables (informative).** A consumer uses its platform's Unicode
  tables, which may predate 15.1; only code points whose General_Category
  changed since differ, and none occur in the reference corpus. A producer
  **MUST** use 15.1.0 tables or later and **SHOULD** ship a `words` override
  where a block's tokenization would differ under 15.1.0.

### 5.3 Examples

| Text | Tokens |
|---|---|
| `“That’s the worst of it, Mr. Holmes, I don’t know.”` | `That’s` `the` `worst` `of` `it` `Mr` `Holmes` `I` `don’t` `know` |
| `“‘Is £ 4 a week.` | `Is` `4` `a` `week` |
| `В этом-то всё и дело.` | `В` `этом-то` `всё` `и` `дело` |
| `„Er sieht nicht gerade wie eine Zierde für die Bow-Street-Zellen aus, was?“` | `Er` `sieht` `nicht` `gerade` `wie` `eine` `Zierde` `für` `die` `Bow-Street-Zellen` `aus` `was` |
| `. Да, Кинг-Эдвард-стрит, 17, рядом с С` | `Да` `Кинг-Эдвард-стрит` `17` `рядом` `с` `С` |

### 5.4 Lazy tokenization (consumer requirement)

Tokenizing is cheap per string (microseconds) but materializing token arrays
for every language of a chapter is not: doing so eagerly for the eight
languages of the reference sample costs more heap than the whole version-1
chapter tree (14,486,956 B versus 12,784,448 B, report §4.3, M4). Therefore a
consumer **MUST** tokenize per language on demand — the active language's
translations when they are displayed, or on tap — and **MUST NOT** tokenize
every overlay of a chapter on open.

---

## 6. Overlay — `gloss/chN.<lang>.json`

One overlay per (chapter, language). It binds to the skeleton by chapter id and
paragraph ids, and carries per sentence the translation text and one link per
target token.

```json
{
  "lang": "ru",
  "chapter": "cbedb517c",
  "gates": ["langcheck", "lexcheck"],
  "alignDigest": "sha256:4bfa91a5b38848ba41ed5bf471da900413b0acdb6c68928dfaa74c0b6e832918",
  "paragraphs": [
    { "id": "0f4552b1", "s": [ { "t": "СКАНДАЛ В БОГЕМИИ", "a": [1, [0, 2], 3] } ] },
    { "id": "04537723", "s": [ { "t": "«Тогда я могу подождать в соседней комнате.»", "a": [0, 1, 2, 3, 4, [5, 6], 7] } ] },
    { "id": "781304bb", "s": [
        { "t": "«О, в его новом офисе", "a": [0, 1, 2, 3, 4] },
        { "t": ". Он действительно сказал мне адрес", "a": [0, 1, [1, 2], 3, [4, 5]] },
        { "t": ". Да, Кинг-Эдвард-стрит, 17, рядом с С", "a": [0, [2, 3, 4], 1, 5, 5, 6] },
        { "t": ". Павлом»", "a": [0] },
        { "t": ".»" }
    ] },
    { "id": "ff1de7b8", "s": [
        { "t": "«Всё же, если бы я вышла замуж за лорда Сент-Саймона, я, конечно, исполнила бы свой долг перед ним.",
          "a": [0, 0, 1, 3, 2, 4, 4, 4, 5, [6, 7], 10, [8, 9], [10, 12], [10, 11], 13, 14, 15, 16] }
    ] }
  ]
}
```

(The `alignDigest` shown is the real value for a one-paragraph overlay holding
only the S1 sentence, [Annex C](#annex-c--complete-minimal-example); a
four-paragraph overlay has a different digest.)

### 6.1 Overlay object and binding

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `lang` | string | **Yes** | The overlay's language tag; **MUST** equal the `spine[].gloss` key that names this entry. |
| `chapter` | string | **Yes** | **MUST** equal the skeleton's `id`. |
| `gates` | array of string | **Yes** | Quality gates that ran over every translation of this overlay ([§6.8](#68-v-and-gates--verdicts)). **MAY** be empty. |
| `alignDigest` | string | **Yes** | `"sha256:<64 hex>"` over the alignment pairs ([§9.2](#92-aligndigest--the-alignment-pair-digest)). |
| `paragraphs` | array of `GlossParagraph` | **Yes** | Parallel to the skeleton's `paragraphs`. |

`GlossParagraph` is `{ "id": paragraphId, "s": [Translation | null, …] }` for a
text block, or `{ "id": paragraphId, "rows": [[GlossCell, …], …] }` for a
`table` block, where `GlossCell` = `{ "s": [Translation | null, …] }`.

**Binding rules — a consumer MUST reject the overlay (treat the chapter as
having no translation in this language, and report the error) if any fails:**

1. `chapter` ≠ skeleton `id`.
2. `length(paragraphs)` ≠ skeleton `length(paragraphs)`.
3. For any `p`: `paragraphs[p].id` ≠ skeleton `paragraphs[p].id`.
4. For any text block: `length(s)` ≠ `length(sents)`; for any table:
   row count, or any row's cell count, or any cell's `length(s)` differs.
5. `lang` ≠ the `spine[].gloss` key that names this entry.
6. Shape mismatch: a text block whose gloss paragraph carries `rows`, or a
   `table` block whose gloss paragraph carries `s`.

A `null` entry means "not translated". A rejected overlay **MUST NOT** be
rendered partially; permutations that pass all four rules are caught by
`digests` and `alignDigest` ([§9](#9-integrity)).

### 6.2 The `Translation` object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `t` | string | No | Translation text, NFC, displayed verbatim; the coordinate space of `x`, `words`, `parts`. Absent = no translation text (then only `s`/`v` may be present). |
| `a` | array of `Link` | No | One link per target token, in token order ([§6.4](#64-a--links)). Absent = no links known. |
| `x` | array | No | Escape chunks ([§6.5](#65-x--escape-chunks)). |
| `words` | array of `[a, b]` | No | Target token inventory override ([§6.6](#66-words--target-token-inventory-override)). |
| `s` | string | No | Status; absent = ok ([§6.7](#67-s--status)). |
| `v` | object | No | Verdict of a gate ([§6.8](#68-v-and-gates--verdicts)). |
| `lang` | string | No | Actual language of `t` when it differs from the overlay's ([§6.9](#69-lang)). |
| `groups` | array | No | Producer-asserted target lexemes ([§6.10](#610-groups)). |
| `parts` | array | No | Producer-asserted sub-token spans ([§6.11](#611-parts)). |

`a`, `x`, `words`, `groups`, `parts` **MUST NOT** appear without `t`.

### 6.3 Absent translations

A `null` entry in `s[]`, a `Translation` without `t`, or an empty `t` is "no
translation available": the consumer renders the source without a gloss and
shows the status, if any ([§6.7](#67-s--status)). A producer **SHOULD** write
`null` for a sentence it never attempted and a `Translation` with `s` for one
it attempted and did not ship.

### 6.4 `a` — links

`a[j]` is the link of target token `j` (token order, [§5](#5-tokenizer-tbook-w2)
or `words`):

| Value | Meaning | Highlighted by a tap on… |
|---|---|---|
| `int` `i` | token `j` renders source word `i` | word `i` |
| `int[]` | token `j` renders several source words (compound, contraction, fused idiom) | any of them |
| `null` | **unaligned**: the producer does not know what `j` renders | nothing |
| `[]` | **inserted**: `j` has no source counterpart (article, particle, auxiliary) | nothing |

Invariants:

- `length(a) ≤ number of tokens of t`; missing trailing entries are `null`. A consumer **MUST** ignore entries beyond the token count; a validator **MUST** report them.
- Every index **MUST** satisfy `0 ≤ i < wordCount(sentence)`; an `int[]`
  **MUST** be ascending with no duplicates and **MUST** have ≥ 1 element when it
  is not the inserted marker `[]`.
- A source word **MAY** appear in the links of several tokens (one-to-many).
- `null` and `[]` are different statements. A producer **MUST** write `[]` only
  when it knows the token was inserted; a migration or an aligner that cannot
  tell **MUST** write `null`.

Examples (S1, source words `0=Then 1=I 2=can 3=wait 4=in 5=the 6=next 7=room`):

```json
{ "t": "“O zaman bitişik odada bekleyebilirim.”", "a": [0, 0, [5, 6], [4, 7], [1, 2, 3]] }
```

Tapping `wait` (3) highlights «bekleyebilirim»; so does tapping `I` or `can` —
a fan-in the `parts` slot can refine ([§6.11](#611-parts)). In de,
`"a": [0, 2, 1, [4, 5], [6, 7], 3]`: «im» renders `in the`, «Nebenzimmer»
`next room`. S5 (de): `"a": [0, 2, 1, null, null, 3, 4, 5, 6, [7, 8, 9], 2, [10, 11]]`
— `gerade` and `wie` are unaligned, not inserted; `look` (2) is rendered by
tokens 1 and 10, «sieht … aus».

### 6.5 `x` — escape chunks

An escape chunk records an alignment whose target span is **not exactly one
token**: a part of a token, or a run across tokens. Each entry is
`[a, b, i₁, i₂, …]` — a half-open code-point range of `t` followed by zero or
more source word indices:

```json
{ "t": "«Всё же, если бы я вышла замуж за лорда Сент-Саймона, я, конечно, исполнила бы свой долг перед ним.",
  "a": [0, 0, 1, 3, 2, 4, 4, 4, 5, null, 10, [8, 9], [10, 12], [10, 11], 13, 14, 15, 16],
  "x": [[40, 44, 6], [45, 52, 7]] }
```

Had the producer aligned the halves of «Сент-Саймона» separately (`Сент` ←
`St`, `Саймона` ← `Simon`), token 9 would be `null` in `a` and the two sub-token
spans would live in `x`. (The reference file links the whole token, `[6, 7]`.)

Rules:

- `0 ≤ a < b ≤ length(t)`; the range **MUST** contain at least one letter or
  digit; indices follow the rules of `a`.
- An escape chunk **MUST NOT** overlap a token whose `a` entry is a link
  (`int` or `int[]`); it **MAY** overlap `null` and `[]` tokens. It **MUST NOT**
  coincide with a whole token (that is a link, not an escape).
- `x` is the **exception path** (0.02 %–0.56 % of corpus chunks,
  [Annex A.3](#a3-links)). A producer **SHOULD** prefer `words`
  ([§6.6](#66-words--target-token-inventory-override)) for a whole translation
  with a different token inventory, and `parts` ([§6.11](#611-parts)) to refine
  an already linked token.

### 6.6 `words` — target token inventory override

When present, `words = [[a, b], …]` is the explicit token list of `t`
(code-point ranges, sorted, non-overlapping, `a < b`, no U+0009/U+000A) and `a`
indexes **it** instead of the tokenizer output. It is REQUIRED for a target the
tokenizer cannot segment: under `tbook-w2` a Chinese translation is one token
per punctuation-delimited run, and 「所以基本上你是在给你老爸施压」 would receive a
single link listing every source word (report §4.3, verification); with `words`
a segmenter's tokens carry ordinary per-token links. A consumer **MUST**
implement this override (it is part of `align/token`) and **MUST NOT**
re-tokenize a translation that carries `words`.

### 6.7 `s` — status

`s` is a **producer-written** state of one translation. Absent = ok.

| Value | Meaning | `t` |
|---|---|---|
| `raw` | Translated; the alignment pass did not run or failed after retries; shipped without links. | present |
| `unaligned` | No links, cause not recorded. **The only value a migration tool may write** ([Annex A.4](#a4-statuses)). | present |
| `skipped` | Deliberately not translated or aligned by policy (e.g. bibliographic citations). | optional |
| `rejected` | A gate judged the translation wrong and it was not repaired. | optional |
| `unverified` | A gate flagged it; regeneration was exhausted; shipped as is. | present |
| `wrongLang` | The text is not in the overlay's language (a language check). `lang` **SHOULD** name the detected language. | present |
| `free` | A free rendering: sentence-level meaning is right, per-word taps are unreliable. | present |

Rules:

- **Only the producer writes `s`.** A consumer **MUST NOT** derive a status
  (e.g. by a script test) and drive its UI from it: the panel's majority-script
  check mis-flagged 51 of 1,361 cells, all Roman-numeral headings such as
  «I.», «II.» (report §4.3). A migration tool writes `unaligned` only.
- **A consumer MUST surface a non-ok status** — a badge, a dimmed highlight,
  or a hidden gloss — so a learner never takes «СКАНДАЛ В БОГЕМИИ» under the
  `tr` key for Turkish (S8; 9.43 % of the shipped sample's sentences carried
  such a cell, report §2.2 C19). For `wrongLang` and `rejected` it **MAY** hide
  `t`. Unknown values **MUST** be surfaced like `unverified`. A consumer **MUST NOT** hide `t` for a status it does not know.
- An ok `s` says nothing about correctness; that is what `v` and `gates` say.

### 6.8 `v` and `gates` — verdicts

`gates` (overlay level) lists the quality gates that ran over **every**
translation of the overlay. `v` (per translation) is the result of a gate that
did not pass the cell:

```json
{ "t": "Двое сопровождающих замыкали шествие.", "a": [1, [0, 2], 2],
  "s": "free", "v": { "ok": false, "by": "judge", "why": "free rendering; brought/rear have no counterpart" } }
```

| Field | Type | Required | Description |
|---|---|---|---|
| `ok` | boolean | **Yes** | `false` = flagged; `true` = explicitly re-verified after a repair. |
| `by` | string | **Yes** | The gate id; **MUST** be listed in `gates`. |
| `why` | string | No | Short human-readable reason. |

Registered gate ids: `langcheck` (language identification), `lexcheck`
(dictionary support of alignment pairs), `judge` (LLM semantic verification).
Others **MUST** be namespaced (`x-…`).

Together the two fields make **absence meaningful**: no `v` under
`gates: ["judge"]` means judged and passed; no `v` under `gates: []` means never
checked. A consumer **SHOULD** show which gates ran and **MUST** surface
`v.ok == false` like a non-ok status.

### 6.9 `lang`

The actual language of `t` when a language check found it differs from the
overlay's `lang`; informative, always accompanied by `s: "wrongLang"`.

### 6.10 `groups`

A **group** is a producer-asserted set of target tokens that form one lexeme,
with the source words they render together:

```json
{ "t": "„Er sieht nicht gerade wie eine Zierde für die Bow-Street-Zellen aus, was?“",
  "a": [0, 2, 1, null, null, 3, 4, 5, 6, [7, 8, 9], 2, [10, 11]],
  "groups": [["lexeme", [1, 10], [2]]] }
```

Each entry is `[kind, targetTokens[], sourceWords[]]`; kinds: `lexeme`
(a discontinuous lexeme such as the separable verb `sieht … aus`), `phrase`
(a contiguous multi-token rendering), `name`. A group **MUST** be consistent
with `a` (every listed token's link contains at least one of the listed
words). A group takes precedence over the derived labelling of
[§8.3](#83-derived-labelling-runs-and-splits) for the words it lists. When several groups list the tapped word, the first in list order wins. A
consumer ignoring `groups` loses only the label.

### 6.11 `parts`

A **part** refines an already linked token into sub-token spans with their own
links — the slot for morpheme-level alignment of agglutinative targets and for
elisions:

```json
{ "t": "“O zaman bitişik odada bekleyebilirim.”", "a": [0, 0, [5, 6], [4, 7], [1, 2, 3]],
  "parts": [[4, 0, 5, 3], [4, 5, 10, 2], [4, 10, 12, []], [4, 12, 14, 1]] }
```

Each entry is `[token, a, b, link]`: `a`, `b` are code-point offsets **relative
to the token's start**, `0 ≤ a < b ≤ tokenLength` (the tokenizer token, or the `words` override range when one is present); `link` is an `int`, an
ascending `int[]`, or `[]` for an inserted morpheme. Here `bekle` ← `wait`,
`yebil` ← `can`, `ir` (aorist) inserted, `im` ← `I`.

Rules: parts of one token **MUST NOT** overlap; every non-empty part link
**MUST** be a subset of the token's `a` link; a token with parts **MUST** have
an `int` or `int[]` link in `a`. Tap rule: **smallest containing wins** — if a
part of token `k` links to the tapped word, the consumer highlights the part
span(s), otherwise the whole token ([§8.2](#82-word-rung)). A consumer
ignoring `parts` highlights whole tokens, exactly as without them.

`parts` and `x` are disjoint by rule: `x` carries spans of tokens that have
**no** link; `parts` refines tokens that **have** one.

### 6.12 Footnote overlays — `gloss/notes.<lang>.json`

```json
{ "lang": "ru", "gates": [], "alignDigest": "sha256:0000000000000000000000000000000000000000000000000000000000000000",
  "notes": { "n1": [ { "id": "ea90ea77", "s": [ { "t": "Двое сопровождающих замыкали шествие.", "a": [1, [0, 2], 2] } ] } ] } }
```

Same rules as a chapter overlay, binding per note id: every key of `notes`
**MUST** exist in `text/notes.json`; each note's paragraph list follows
[§6.1](#61-overlay-object-and-binding) rules 2 and 4 against the note's
`paragraphs`. Note paragraphs bind **by position** ([§4.3](#43-paragraph-id),
decision 6): rule 3 does **not** apply, and a consumer **MUST NOT** reject a
notes overlay because a `GlossParagraph.id` differs from the id of the note
paragraph at the same index (a validator **MAY** warn). The `lang`
check of rule 5 (decision 17) **does** apply: `lang` **MUST** equal the
`notes.gloss` key that names the entry, and a consumer **MUST** reject the
overlay (notes shown without translation in that language) if it does not. Citations **MAY** be left untranslated (`null` or
`{ "s": "skipped" }`). For the digest, notes are processed in ascending
code-point order of their ids ([§9.2](#92-aligndigest--the-alignment-pair-digest)).

---

## 7. Offsets and normalization

**This section is normative.** Every offset in a version-2 file — `sents`,
`words` (both sides), `spans[].{s,e}`, `notes[].p`, `x[][0..1]`, `parts[][1..2]`
— is a **Unicode code-point** offset (half-open `[a, b)` for ranges; an
insertion point for `p`) into the relevant NFC string. Every string in every
JSON entry **MUST** be NFC; a producer **MUST** normalize before computing any
offset or digest.

Consumers whose strings are UTF-16 (Kotlin/JVM, JavaScript) **MUST** convert
code-point offsets to UTF-16 indices before slicing — once per block or
translation, whenever the string contains a code point above U+FFFF. The pass
is a no-op on the reference corpus (no astral characters) but it **MUST**
exist: version 1's "clamp and hope" is what this section replaces. Ranges the
consumer derives with its own tokenizer are already in native units; only
offsets *read from the file* need conversion.

A consumer **MUST** still clamp every offset it reads into the valid range of
its string and skip degenerate ranges (`b ≤ a`) rather than fail: alignment is
model-generated and a file may carry rare imperfections that pass validation.

---

## 8. Tap-to-translate algorithm

Given a tap at code point `p` of a rendered block, gloss language `L`:

### 8.1 Resolve

1. Find sentence `k` with `sents[k][0] ≤ p < sents[k][1]`; none → nothing.
2. Build the word inventory of `k` ([§4.5](#45-text-and-sents) or `words`);
   find word `i` whose range contains `p`; none → nothing.
3. `tr = overlay(L).paragraphs[para].s[k]` (or the cell's `s[k]`). `null`, no
   `t`, or empty `t` → show "no translation" and stop. A non-ok `s` or
   `v.ok == false` **MUST** be surfaced ([§6.7](#67-s--status)); the consumer
   **MAY** stop here for `wrongLang`/`rejected`.

### 8.2 Word rung

4. `toks = tokens(tr.t)` (or `tr.words`), computed lazily and cached.
5. Highlight set `H` = every token `j` with `a[j] == i` or `i ∈ a[j]`; for each
   such `j` that has `parts` linking `i`, replace the whole token by those part
   spans (smallest containing wins); plus `t[a:b)` of every `x` chunk whose
   indices contain `i`.

### 8.3 Derived labelling: runs and splits

This rule costs 0 bytes and applies to every file. A quarter of all taps in the
reference sample land on a source word rendered by several target tokens
(26.3 % contiguous, a further 4.8 % discontiguous; report §0 and §5.1), and a
learner cannot tell a multi-word rendering from a smear unless the reader says
which it is.

6. If `H` contains ≥ 2 whole tokens: if a `groups` entry lists `i`, label the
   highlight with that group's kind (`lexeme`: one word rendered by
   `sieht … aus`). Otherwise derive: partition the highlighted token indices
   into maximal blocks of **consecutive** indices. One block → a **run**
   ("one rendering of *word*": S2 `just` → «этом-то всё»). Two or more blocks →
   a **split** ("split rendering": S5 `look` → «sieht … aus»). A consumer
   **MAY** show the label; it **MUST NOT** merge the blocks' text. Only tokens highlighted whole count toward this decision: part spans and `x` spans do not, so a highlight with fewer than two whole tokens carries no label.

### 8.4 Unit mode

7. If `i` belongs to a `units` entry of sentence `k` (smallest containing
   unit), a consumer **MAY** offer a unit view: the union of step 5 over all
   members, labelled with the unit's kind and source text (S6 `of`/`course` →
   es «por supuesto», fr «bien sûr», ru «конечно»). The word view of step 5
   **MUST** remain reachable.

### 8.5 Reference (TypeScript)

```typescript
type Link = number | number[] | null;
interface Translation { t?: string; a?: Link[]; x?: number[][]; words?: [number, number][];
                        parts?: [number, number, number, Link][]; }

/** Code-point ranges of tr.t to highlight for source word i (§8.2). */
function highlightRanges(tr: Translation, toks: [number, number][], i: number): [number, number][] {
  const out: [number, number][] = [];
  const hits = (l: Link) => l !== null && (l === i || (Array.isArray(l) && l.includes(i)));
  (tr.a ?? []).forEach((link, j) => {
    if (j >= toks.length || !hits(link)) return;
    const [ta, tb] = toks[j];
    const parts = (tr.parts ?? []).filter(p => p[0] === j && hits(p[3]));
    if (parts.length) parts.forEach(p => out.push([ta + p[1], ta + p[2]]));
    else out.push([ta, tb]);
  });
  for (const ch of tr.x ?? []) if (ch.slice(2).includes(i)) out.push([ch[0], ch[1]]);
  return out.map(([a, b]) => [clamp(a, 0, n(tr)), clamp(b, 0, n(tr))] as [number, number])
            .filter(([a, b]) => b > a).sort((p, q) => p[0] - q[0]);
}
/** §8.3: "run" for one block of consecutive token indices, "split" for several. */
function label(tokenIdx: number[]): "run" | "split" | null {
  if (tokenIdx.length < 2) return null;
  const s = [...tokenIdx].sort((a, b) => a - b);
  return s.every((v, k) => k === 0 || v === s[k - 1] + 1) ? "run" : "split";
}
```

`toks` comes from the consumer's tokenizer (or `tr.words`), converted to native
units where needed ([§7](#7-offsets-and-normalization)); `n(tr)` is `tr.t`'s
length in those units.

### 8.6 Language switch

Switching gloss language loads another overlay; the skeleton and its
tokenization **MUST** be reused, not re-read, and the previously active overlay
**SHOULD** stay parsed (bounded cache) so switching back is not a second decode.

---

## 9. Integrity

### 9.1 `digests` — entry integrity

`manifest.digests` maps every entry except `mimetype` and `manifest.json` to
`"sha256:"` + lowercase hex SHA-256 of its **uncompressed bytes**. A producer
**MUST** write it complete and recompute it on every write, append
([§2.4](#24-appending-a-language)) or replacement ([§2.5](#25-replacing-a-language)).
A validator **MUST** verify all.

<a id="referenced-entries"></a>**Referenced entries.** The entries a file
*references* are: the chapter skeletons (`spine[].text`), the overlays (every
value of `spine[].gloss`), the footnote entries (`notes.text` and every value of
`notes.gloss`), `cover` when non-null, `schema`, and every key of `digests`. An
image named only inside chapter text (`figure.image`, [§4.10](#410-figure)) is
verified when it is displayed, not at import.

<a id="digest-at-import"></a>**At import.** A consumer **SHOULD** verify the
digest of every referenced entry when it imports a file, and **MUST** then
refuse the **whole file** on any mismatch or on a missing referenced entry.

<a id="digest-lazy"></a>**After open.** A consumer that verifies lazily — when
an entry is first read after the file was opened — **MUST** refuse only the
damaged entry, report the error, and keep the rest of the book usable:

| Damaged entry | Consequence |
|---|---|
| chapter skeleton | the chapter is not shown (an error is shown in its place) |
| overlay | that language is not shown for that chapter, as for a binding failure ([§6.1](#61-overlay-object-and-binding)) |
| footnote bodies (`notes.text`) | footnotes are unavailable |
| footnote overlay (a `notes.gloss` value) | footnotes stay readable; that language's footnote translations are not shown |
| image or cover | the image is not shown |

<a id="digest-missing"></a>**Entry without a digest.** A referenced entry that
has no key in `digests` makes the file non-conformant
([§11.1](#111-a-conformant-file-must)); a validator **MUST** report it. A
consumer reads such an entry unverified and **SHOULD** log a warning; it
**MUST NOT** refuse the file or the entry for this reason alone.

### 9.2 `alignDigest` — the alignment-pair digest

A permuted alignment — every link shuffled within its sentence — is
structurally valid: every index is in range, coverage is unchanged, and the
version-1 validator accepted such a file byte-for-byte identically (report §2.2,
C10). `alignDigest` is the structural defence: a digest over the **texts** of
the aligned pairs, which any party can recompute from the file, and which the
producer computes independently from its aligner's output — so an index-level
drift between the aligner's pairs and the serialized links is caught at write
time, and a later permutation is caught by any validator.

**Recipe.** `canonical` is the UTF-8 string built by visiting, in order:
`paragraphs` in entry order (for a table, rows then cells; for a notes overlay,
note ids in ascending code-point order, then paragraphs); each sentence with a
`t`; and for each:

1. for each token `j` in order with a non-`null` link: if the link is `[]`,
   append U+0009, `token(j)`, U+000A; else for each source index `i` in
   ascending order append `word(i)`, U+0009, `token(j)`, U+000A;
2. then for each `x` chunk in list order: for each index `i` ascending (or once
   with an empty word if there are none) append `word(i)`, U+0009, `t[a:b)`,
   U+000A.

`word(i)` and `token(j)` are the NFC texts of the source word and target token
(tokenizer or override). `parts` and `groups` are excluded. `alignDigest` =
`"sha256:"` + hex SHA-256(`canonical`). A sentence with no pairs contributes
nothing; an overlay with no pairs digests the empty string.

For a one-paragraph overlay holding only S1 `ru` the canonical string is

```
Then\tТогда\nI\tя\ncan\tмогу\nwait\tподождать\nin\tв\nthe\tсоседней\nnext\tсоседней\nroom\tкомнате\n
```

and `alignDigest` is
`sha256:4bfa91a5b38848ba41ed5bf471da900413b0acdb6c68928dfaa74c0b6e832918`.
Swapping the links of «Тогда» and «комнате» yields
`sha256:f7552db092fe0c64ff64ab2d69079e20a36e4fb3f9efd8c51dd6ab99e5e24d7b`.

A producer **MUST** compute `alignDigest` from its aligner's pair list and
verify it against the serialized overlay before writing; a validator **MUST**
recompute it from the file. A consumer **MAY** verify it, on import or when it
loads the overlay.

<a id="aligndigest-mismatch"></a>A consumer that verifies `alignDigest` and finds
a mismatch **MUST** reject the overlay exactly as for a binding failure
([§6.1](#61-overlay-object-and-binding), decision 17): the chapter (or, for a
notes overlay, the footnotes) has no translation in that language, the error is
reported, and the overlay is never rendered partially. An overlay **without**
`alignDigest` is accepted by a consumer — it is read unverified — although the
field is REQUIRED and a validator reports its absence.

### 9.3 What each check defends

| Attack or accident | Caught by |
|---|---|
| Entry bytes corrupted or replaced | `digests` |
| Two overlays' contents swapped, or an overlay renamed to another chapter | `digests`; `chapter` ≠ skeleton id; paragraph id mismatch ([§6.1](#61-overlay-object-and-binding)) |
| Overlay paragraphs shifted by one | paragraph id / count mismatch |
| Links permuted or drifted inside a sentence | `alignDigest` |
| Gloss built from another edition of the book | `source.textSha256` / `docs[].sha256` ([§3.6](#36-source--source-identity)) |

None of these is a cryptographic signature; a producer that wants provenance
**MAY** sign the manifest by an external mechanism.

---

## 10. Versioning and compatibility

### 10.1 `formatVersion`

`manifest.formatVersion` is the on-disk major version. A consumer **MUST** read
it before decoding any other field (decode the manifest as a generic JSON
object first): `1` → the version-1 code path; `2` → this document; anything
else, or a missing field → refuse, stating the version. A missing
`formatVersion` **MUST NOT** default to any version.

### 10.2 Additive change under version 2

Within version 2, changes **MUST** be additive and backward-compatible:

- New OPTIONAL fields **MAY** be added anywhere; consumers **MUST** ignore
  unknown fields and unreferenced entries.
- A new field that a consumer must understand to render *correctly* **MUST**
  come with a new `requires` id ([§3.2](#32-formatversion-and-requires)).
- Meanings of existing fields, the tokenizer `tbook-w2`, the id recipes and the
  digest recipes **MUST NOT** change; a change to any of them is version 3.
- The spec **MUST** be updated before the reference producer emits a change.

### 10.3 Version-2 consumers opening a version-1 file

A version-2 consumer **SHOULD** keep a version-1 code path selected by
`formatVersion`, or convert on import with the migration of
[Annex A](#annex-a--migration-from-version-1-normative-for-migration-tools).

### 10.4 Version-1 consumers opening a version-2 file

Shipped version-1 consumers do not check `formatVersion` (report §2.2, C20),
so the version-2 manifest is shaped to fail in them **at manifest parse, before
any text is rendered**: `notes` is an object where version 1 has a string
([§3.7](#37-notes)); `author`, `sourceLang`, `targetLangs`, `chapters` are
absent. Making `notes` an object *unconditionally* is what makes the failure
consistent — with a nullable `notes`, a footnote-free book opened as an empty
library entry while a footnoted one was refused (report §4.3, verification). A
version-1 consumer tolerant of missing fields sees zero chapters and renders
nothing. In no case does it render wrong text or wrong highlights.

---

## 11. Conformance

### 11.1 A conformant file MUST

1. Be a ZIP per [§2](#2-container): STORED/DEFLATE only, unique names,
   `mimetype` first and STORED with the exact 25 bytes, `manifest.json` last.
2. Carry a `manifest.json` with `formatVersion` `2`, `requires` ⊇ the three
   ids of [§3.2](#32-formatversion-and-requires), all REQUIRED fields of
   [§3.1](#31-fields), `notes` an object, and `digests` complete and correct.
3. Ship the published JSON Schema at `manifest.schema`, and have every JSON
   entry validate against its definition.
4. Have every skeleton repeat its `spine[].id`; paragraph ids follow
   [§4.3](#43-paragraph-id); `sents` sorted, non-overlapping, in range; every
   token starting inside a sentence; `sceneBreak`/`table` blocks empty;
   `figure.image` existing; `words` overrides well-formed.
5. Have every overlay satisfy the binding rules of
   [§6.1](#61-overlay-object-and-binding), every link and escape in range
   ([§6.4](#64-a--links), [§6.5](#65-x--escape-chunks)), `x` disjoint from
   linked tokens, `parts` consistent with `a`, `v.by` ∈ `gates`, and a correct
   `alignDigest`.
6. Have every string NFC and every offset a code point ([§7](#7-offsets-and-normalization)).
7. Carry only source-derived `units`, and no format semantics in `meta`.

### 11.2 A conformant producer MUST

1. Tokenize source and target with `tbook-w2` (Unicode ≥ 15.1.0) or emit a
   `words` override; write one link per target token; write `[]` only for a
   known insertion.
2. Write `s` only from pipeline state; never derive `wrongLang` by script
   heuristics without a real language check.
3. Compute `alignDigest` from the aligner's pairs and verify it against the
   serialized overlay; recompute `digests` on every write or append.
4. Append a language per [§2.4](#24-appending-a-language) without touching
   skeleton entries, and replace one per [§2.5](#25-replacing-a-language)
   keeping every other entry byte-identical; refuse either on an input whose
   `manifest.json` is not last or not unique, naming the entry that follows it.

### 11.3 A conformant consumer MUST

1. Check `formatVersion` first, then `requires`; refuse unknown ids by name.
2. Discover entries only through the manifest; ignore unknown fields/entries.
3. Reject an overlay that fails binding; never render it partially.
4. Convert code-point offsets to native units where needed; clamp; skip
   degenerate ranges.
5. Apply `words` overrides on both sides; tokenize lazily per language; reuse
   the skeleton across a language switch.
6. Surface non-ok `s` and `v.ok == false`; never derive statuses itself.
7. Resolve locators, wherever it stores or accepts them, with the paragraph-id fallback.
8. Implement the word rung including `x` and `parts`; never merge the text of
   a split's blocks (showing the run/split label is OPTIONAL, [§8.3](#83-derived-labelling-runs-and-splits)).
9. When it verifies integrity: refuse the whole file on a digest mismatch found
   at import, refuse only the damaged entry on one found after open
   ([§9.1](#91-digests--entry-integrity)), and reject an overlay whose
   `alignDigest` mismatches as for a binding failure ([§9.2](#92-aligndigest--the-alignment-pair-digest)).

### 11.4 Validation

The reference producer validates on assembly. An independent validator checks
[§11.1](#111-a-conformant-file-must) in full — schema, cross-entry invariants,
`digests`, `alignDigest` — and runs the tests of
[Annex B](#annex-b--conformance-tests). Structural validity still does not
prove alignment *correctness*; only a semantic gate does, and the file now
records whether one ran (`gates`).

---

## 12. Reference implementations

The reference converter writes version 2 by default since 2026-09-26; no
version-2 reader has been released yet. Mapping: producer and validator in `tbook_converter` (`internal/tbook`,
`internal/segment` for `tbook-w2`, `internal/align` for links, `x` and the
digest; `cmd/tbook-migrate`, `cmd/tbook-validate`); consumers in
`tbook_desktop` (`src-tauri/src/{models,tbook}.rs`,
`src/lib/{render,align,tokenize}.ts`) and `TReader` (`data/model`, `data/book`,
`ui/reader`). The Final document will map every section to its file, as
version 1 §11 does.

---

## Annex A — Migration from version 1 (normative for migration tools)

A version-1 file converts to version 2 **losslessly** except for
[A.6](#a6-what-is-not-carried). On the reference sample (8 languages, 836,600
taps) and a single-language book (158,421 taps), 99.9876 % and 100 % of taps
highlight exactly what version 1 did; the whole residue is the 104 taps (13
fragments × 8 languages) on the mid-word cuts that
[A.7](#a7-worked-example-s9) repairs (report §4.3, M8).

### A.1 Mapping

| Version 1 | Version 2 |
|---|---|
| `manifest.chapters[]` `{id, title, file}` | `spine[]` `{id (§3.5.1), title, text, gloss{…}}`; `chN` ordinals dropped |
| `author` / `sourceLang` / `targetLangs` | `authors: [author]` / `languages.source` / `languages.targets` |
| `notes: "notes.json"` / `null` | `notes: {text, gloss{…}}` / `{}` |
| `cover`, `meta` | unchanged; `meta.runs[]` gains a migration record with `options.tokenizer` |
| `chapters/chN.json` `paragraphs[p]` (sentence list) | `text/chN.json` `paragraphs[p]` `{id, text, sents}` ([A.2](#a2-paragraph-text-and-sentence-ranges)) |
| `paragraphStyles[p]` | `paragraphs[p].role` (omitted when `body`) |
| `figures[] {para, image, alt}` / `tables[] {para, rows}` | `paragraphs[para].figure` / `.table` (cells become blocks) |
| `Sentence.src` | a range of the paragraph `text` |
| `Sentence.words` | dropped — derived by `tbook-w2` ([A.3](#a3-links)); emitted as `words` only if a v1 word is not contained in one v2 token (0 cases in the corpus) |
| `Sentence.spans`, `Sentence.notes` | block `spans`, `notes`, offsets re-based by the sentence start |
| `tr[L].text` | `gloss/chN.L.json` `…s[k].t`; `""` → `null` |
| `tr[L].align[] {t, w}` | `a` links + `x` escapes ([A.3](#a3-links)) |
| `tr[L].q` | dropped — coverage is `count(a[j] ≠ null) / tokens(t)`, derivable |
| `notes.json` | `text/notes.json` + `gloss/notes.L.json` |

### A.2 Paragraph text and sentence ranges

`text` = the v1 sentences joined with one U+0020, **except** no space when the
previous sentence ends in a letter or digit and the next starts with a lowercase
letter, or the previous ends in a letter + U+2019 and the next begins with a
lone `s` — the signature of the v1 segmenter's mid-word cuts (13 in the sample,
17 in an es-source book, 0 false joins in five books). `sents[k]` = each
sentence's range after the join; `id` per [§4.3](#43-paragraph-id).

### A.3 Links

For each v1 chunk `{t:[a,b], w}` of translation `text`:

1. Map every source index in `w` to the v2 word that **contains** the v1
   word's range (`sentence-relative`); a v1 word whose token now starts in
   another sentence (the mid-word cuts) has no v2 counterpart and is dropped.
2. If `[a, b)` equals exactly one `tbook-w2` token `j` of `text`: set `a[j]` to
   the mapped indices (`int` if one, ascending `int[]` if several); if `a[j]` is
   already set, take the union. If the mapped set is empty (orphan), leave
   `null`.
3. Otherwise, if `[a, b)` contains a letter or digit: append
   `[a, b, …indices]` to `x`. Fragments with no letter or digit are dropped
   (they highlighted nothing in v1).
4. **Digit pass-through:** a still-`null` token that is a digit-only string
   identical to a source word of the sentence is linked to that word (S4 `4` →
   «4»; 530 links in the sample).

Every v1 word lies in exactly one `tbook-w2` token (0 failures in 995,021
taps), so step 1 never fails. Step 3 fires for 0.02 % (English-source books) to
0.56 % (an older es-source book) of chunks — elisions split by the v1 aligner
(`l` + `avons`), digit-adjacent fragments (`1888'di` → `di`), em-dash joins
(report §4.3, verification).

### A.4 Statuses

A migration tool **MUST** write `s: "unaligned"` for a translation that has
text, at least one token, and no links after [A.3](#a3-links), and **MUST NOT**
write any other status: `raw`, `wrongLang`, `skipped` and `rejected` are
pipeline facts a v1 file does not record (991 sample cells and 1,435 cells of a
de→ru book conflate four causes; report §4.3, verification). `gates` = `[]`;
no `v` is written. `[]` links are never written by migration.

### A.5 Ids and digests

Ids per [§3.5.1](#351-chapter-id) and [§4.3](#43-paragraph-id); `digests` and
`alignDigest` from the emitted bytes. Version-1 saved positions are index-based;
a consumer **SHOULD** map `(chapterIndex, paragraphIndex)` to a locator once, on
first open after migration.

### A.6 What is not carried

- `q` (derivable).
- The 13 sample fragments' orphaned chunk references: a v1 chunk whose only
  source word was a mid-word fragment (`s`, `t`) becomes `null`; a v1 chunk
  spanning the fragment and a real word keeps the real word (S9: «Он» ← `He`).

### A.7 Worked example (S9)

Version 1 stored `sample ch3 p103` as five sentences — `“‘Oh, at his new
office` | `s. He did tell me the addres` | `s. Yes, 17 King Edward Street, near
S` | `t. Paul’` | `s.’` — rendered by consumers as «office s. He did tell me the
addres s.». Version 2 stores the paragraph of [§4](#4-skeleton--textchnjson)
(`781304bb`) with `sents` `[[0,23],[23,51],[51,88],[88,96],[96,99]]` and the
`ru` overlay of [§6](#6-overlay--glosschnlangjson). Taps: `offices` → «офисе»;
`He` → «Он» (v1: ← {`s`, `He`}); `address` → «адрес»; `17` → «17»; `St` → «С»;
`Paul’s` → «Павлом»; sentence 4 has no words.

### A.8 Version 2 → version 1 (informative)

A down-converter is possible for prose: `src = text[a:b)` per sentence,
`words` from the tokenizer, `{t, w}` chunks from `a` (token ranges) and `x`.
It drops `[]` semantics, `s`, `v`, `gates`, `units`, `groups`, `parts`, ids,
`source` and `alignDigest`, and must re-synthesize `notes` as a string.

---

## Annex B — Conformance tests

### B.1 The fixture book

A one-chapter version-2 book, title `Canonical sentences`, chapter id
`cbedb517c`, eleven paragraphs in this order, one sentence each unless noted:

| # | Paragraph id | Text (source) | Fixture |
|---|---|---|---|
| 0 | `04537723` | `“Then I can wait in the next room.”` | S1 |
| 1 | `2b606e75` | `That is just my point.` | S2 |
| 2 | `1bfb4bea` | `“That’s the worst of it, Mr. Holmes, I don’t know.”` | S3 |
| 3 | `8d89345a` | `“‘Is £ 4 a week.` | S4 |
| 4 | `0e2c5231` | `“He doesn’t look a credit to the Bow Street cells, does he?”` | S5 |
| 5 | `ff1de7b8` | `“Still, if I had married Lord St. Simon, of course I’d have done my duty by him.` | S6 |
| 6 | `ea90ea77` | `The two attendants brought up the rear.` | S7 |
| 7 | `0f4552b1` | `A SCANDAL IN BOHEMIA` (role `heading`) | S8 |
| 8 | `781304bb` | the S9 paragraph, 5 sentences ([§4](#4-skeleton--textchnjson)) | S9 |
| 9 | `0982e735` | `“If your Majesty would condescend to state your case,” …` | S10a |
| 10 | `b51a0a09` | `With hardly a word spoken, … threw across his case of cigars, …` | S10b |

Overlays `ru`, `tr`, `de` carry the translations of the panel fixture
(`canonical-sentences.json`) migrated per [Annex A](#annex-a--migration-from-version-1-normative-for-migration-tools);
the `a` arrays quoted below are the expected migration output.

### B.2 Expected taps (word rung, [§8.2](#82-word-rung))

| Tap | Expected highlight | Label ([§8.3](#83-derived-labelling-runs-and-splits)) |
|---|---|---|
| S1 `wait` | ru «подождать»; tr «bekleyebilirim»; de «warten» | — |
| S1 `I`, `can` (tr) | «bekleyebilirim» (fan-in; with `parts` of [§6.11](#611-parts): «im», «yebil») | — |
| S2 `just` (ru, `a: [1,[0,1,2],2,1,[3,4]]`) | «этом-то», «всё» | run |
| S2 `is` (ru) | «В», «этом-то», «и» | split |
| S3 `of` (tr, `a: [1,[2,3],3,4,5,6,[7,8,9]]`) | «kötüsü», «de» | run |
| S3 `of` (es, `a: [0,0,[1,3],2,5,6,8,9,[7,9]]`) | «lo» | — |
| S3 `don’t` (tr) | «bilmiyorum» | — |
| S4 `4` (tr, `a: [[0,2,3],1,2]`; de, `a: [0,0,1,2,2,3]`) | «4» | — |
| S4 `4` (ru, `a: [0,[0,2],2,3]`) | nothing (numeral parked on `Is` by the v1 aligner; the file carries it faithfully) | — |
| S4 `Is` (ru) | «Четыре», «фунта» | run |
| S5 `look` (de) | «sieht», «aus» | split; `lexeme` when `groups` present |
| S5 `look` (ru, `a: [1,2,null,0,4,9,5,[6,7,8],[10,11]]`) | «похоже» | — |
| S6 `of` / `course` (ru) | «конечно» | — (unit mode: same) |
| S7 `brought`, `rear` (ru, `a: [1,[0,2],2]`) | nothing; `s: "free"` badge when present | — |
| S7 `attendants` (ru) | «сопровождающих», «замыкали» | run |
| S8 any word (`s: "wrongLang"`, `lang: "ru"` on the tr/de cells) | consumer **MUST** surface the status; MAY hide «СКАНДАЛ В БОГЕМИИ» | — |
| S9 `offices` / `He` / `address` / `17` / `St` / `Paul’s` (ru) | «офисе» / «Он» / «адрес» / «17» / «С» / «Павлом» | — |
| S10a `case` (ru) | «дело» | — |
| S10b `case` (word 19, ru) | «футляр» | — |
| S10b `case` (word 26, "spirit case") | nothing — the reference alignment links «шкафчик» to word 24 (`a`), a drift the format carries and a `judge` gate would flag | — |

### B.3 Permuted-alignment test

Take the `ru` overlay; within one sentence, permute the links of `a` (e.g. S1:
`[7,1,2,3,4,[5,6],0]`). The file **MUST** still pass every shape and range
check of [§11.1](#111-a-conformant-file-must) items 1–5 except the last, and
the validator **MUST** fail it on `alignDigest`
([§9.2](#92-aligndigest--the-alignment-pair-digest)). Re-computing `alignDigest`
for the permuted file makes it pass — the test is a defence against drift and
corruption, not against a hostile producer.

### B.4 Tamper tests

From a valid fixture file, each of the following **MUST** be refused by a
validator and by a consumer that verifies `digests`:

1. **Replace**: overwrite `gloss/ch1.ru.json` with the `tr` overlay's bytes →
   `digests` mismatch; without digests, `lang` ≠ `spine` key.
2. **Swap**: exchange two skeletons or two overlays of a two-chapter book →
   `digests` mismatch; `chapter` ≠ skeleton id.
3. **Shift**: delete an overlay's first `GlossParagraph` and append a copy of
   the last → counts still equal, ids mismatch at index 0
   ([§6.1](#61-overlay-object-and-binding) rule 3) → **MUST** reject even
   without digests.
4. **Duplicate name**: a second `manifest.json` → refused ([§2](#2-container)).

### B.5 Mis-binding test

Point `spine[0].gloss.ru` at an overlay whose `chapter` is another chapter's
id, or whose `paragraphs[3].id` differs → the consumer **MUST** show the chapter
with no `ru` translation and report the error; it **MUST NOT** gloss
positionally.

### B.6 Tokenizer vectors

`“‘Is £ 4 a week.` → `Is|4|a|week`; `Bow-Street-Zellen`, `30,000`, `5:15`,
`l'avons`, `don’t`, `этом-то` → one token each; `£`, `—`, `…` → none;
`St. Paul’s` → `St|Paul’s`.

### B.7 Version-1 consumer test

A version-1 consumer opening the fixture **MUST** fail at manifest parse or list
a zero-chapter book; it **MUST NOT** display any paragraph.

---

## Annex C — Complete minimal example

A one-paragraph book (manifest as in [§3](#3-manifestjson) with a single
target `ru`; `notes: {}`, `cover: null`).

`text/ch1.json` — 117 bytes; `digests["text/ch1.json"]` =
`sha256:6d345441cb4fa35e2e1c71ccf0b4a12874c2dfd3679cbcd52cfec965dc21e141`:

```json
{"id":"cbedb517c","paragraphs":[{"id":"04537723","text":"“Then I can wait in the next room.”","sents":[[0,35]]}]}
```

`gloss/ch1.ru.json`:

```json
{"lang":"ru","chapter":"cbedb517c","gates":[],
 "alignDigest":"sha256:4bfa91a5b38848ba41ed5bf471da900413b0acdb6c68928dfaa74c0b6e832918",
 "paragraphs":[{"id":"04537723","s":[{"t":"«Тогда я могу подождать в соседней комнате.»","a":[0,1,2,3,4,[5,6],7]}]}]}
```

Tapping `wait` (3) highlights «подождать»; `the` (5) or `next` (6) highlights
«соседней» (link `[5,6]`); `room` (7) highlights «комнате». Nothing is inserted
or unaligned; `gates` is empty, so the absence of `v` means "unchecked", not
"passed". (The chapter id reuses the fixture value for illustration; a real
one-paragraph chapter with this title hashes differently, since the recipe
takes up to three body paragraphs.)

---

## Ratified decisions (2026-09-26)

Choices the draft made that the panel did not settle, as ratified by the author
on 2026-09-26. All stand as drafted except item 16.

1. **Manifest key shapes** — ratified: `authors` (array) and `languages{source, targets}` replace v1's `author`/`sourceLang`/`targetLangs` ([§3.1](#31-fields), [§3.4](#34-languages)).
2. **`requires` scope** — ratified: `units`, `groups`, `parts` are not gated by `requires`; `words` overrides are part of `align/token` ([§3.2](#32-formatversion-and-requires)).
3. **`x` versus `parts` boundary** — ratified: `x` only for spans of unlinked tokens, `parts` only refines linked tokens ([§6.5](#65-x--escape-chunks), [§6.11](#611-parts)).
4. **Overlay binding by per-paragraph `id`** — ratified, rather than a single skeleton digest ([§6.1](#61-overlay-object-and-binding)).
5. **Chapter-id byte recipe** — ratified: full case folding, `White_Space` collapse, U+000A separator, `-n` suffixes in spine order ([§3.5.1](#351-chapter-id)).
6. **Paragraph-id repeat suffix** — ratified: scoped to the chapter; cells and note paragraphs bound positionally ([§4.3](#43-paragraph-id)).
7. **`textSha256`** — ratified: whitespace removal + NFC only, no quote/dash fold ([§3.6](#36-source--source-identity)).
8. **Status vocabulary** — ratified: `unaligned` added, `untranslated` removed, `t` optional ([§6.7](#67-s--status)).
9. **`v` is one object per translation** — ratified; per-link verdicts are not representable ([§6.8](#68-v-and-gates--verdicts)).
10. **A missing overlay for a listed target is legal** — ratified: partial conversion, validator warning ([§3.4](#34-languages)).
11. **`alignDigest` lives in the overlay** — ratified: `[]` links count as empty-source pairs; `parts`/`groups` excluded ([§9.2](#92-aligndigest--the-alignment-pair-digest)).
12. **Shipped schema is REQUIRED** — ratified; Unicode 15.1.0 pinned in the tokenizer id ([§3.9](#39-schema--the-shipped-json-schema), [§5](#5-tokenizer-tbook-w2)).
13. **Mimetype** — ratified: `application/vnd.tbook+zip` with the offset-38 rule ([§2.2](#22-the-mimetype-entry)).
14. **No ZIP64, no encryption** — ratified as MUST NOTs ([§2](#2-container)).
15. **Migration writes `unaligned` only for a translation with ≥ 1 token** — ratified ([Annex A.4](#a4-statuses)).
16. **Derived labelling** — changed: showing the run/split label is **MAY** for consumers, not SHOULD; merging a split's blocks stays forbidden ([§8.3](#83-derived-labelling-runs-and-splits)).
17. **`lang` mismatch and shape mismatch are binding errors** — ratified ([§6.1](#61-overlay-object-and-binding) rules 5–6).
18. **A token starting in a gap joins the preceding sentence** in a consumer; a validator reports it — ratified ([§4.5](#45-text-and-sents)).
19. **Bilingual splicing never cuts a word** — ratified ([§4.5](#45-text-and-sents)).
20. **Run/split labelling counts whole tokens only; first matching group wins** — ratified ([§8.3](#83-derived-labelling-runs-and-splits), [§6.10](#610-groups)).
21. **Over-long `a` is ignored by consumers**, reported by validators; an unknown status never hides `t` — ratified ([§6.4](#64-a--links), [§6.7](#67-s--status)).
22. **Locator degradation** to the chapter's first paragraph, then the book start — ratified ([§3.5.2](#352-locators)).

## Ratified decisions (2026-09-28)

Second ratification round, on questions raised while implementing the readers
and the append path. Numbered G1–G10 as raised.

1. **G1 — Digest mismatch after open**: a consumer that verifies lazily refuses only the damaged entry (chapter not shown; overlay → that language not shown for the chapter; footnote bodies → notes unavailable; footnote overlay → that language's footnote translations not shown; image/cover not shown); at import a consumer SHOULD verify every referenced entry and refuse the whole file on any mismatch ([§9.1](#digest-lazy)).
2. **G2 — Entry without a digest**: the consumer reads it unverified and SHOULD log a warning; validators report it ([§9.1](#digest-missing)).
3. **G3 — `alignDigest` mismatch**: a consumer that verifies MUST reject the overlay as for a binding failure (decision 17); an overlay without `alignDigest` is accepted ([§9.2](#aligndigest-mismatch)).
4. **G4 — Referenced entries** are chapter texts, overlays, footnote entries, cover, schema and every key of `digests`; images named only inside chapter text are verified when displayed ([§9.1](#referenced-entries)).
5. **G5 — Locators**: `chapterId/paragraphId/0/0` is conformant for readers that anchor by paragraph; a chapter-only locator denotes the position before the first paragraph; "first paragraph of the chapter" in the degradation rule means the chapter start including its title ([§3.5.2](#locator-precision)).
6. **G6 — Notes binding** is by position (decision 6): §6.1 rule 3 does not apply to note paragraphs ([§6.12](#612-footnote-overlays--glossnoteslangjson)).
7. **G7 — The `lang` binding check** (decision 17) applies to footnote overlays too ([§6.12](#612-footnote-overlays--glossnoteslangjson)).
8. **G8 — Append and `meta`**: an append MAY update `meta.updatedAt` and MAY cap `meta.runs` to the most recent 20 records; everything else in `meta` is carried over unchanged ([§3.10](#310-meta--provenance), [§2.4](#24-appending-a-language)).
9. **G9 — Replacing a language** — alternative chosen: a producer MAY replace a target already present by rewriting the archive; every skeleton and every other language's overlay stays byte-identical, digests and `alignDigest` of the replaced language are updated, its earlier `meta.runs` stay and a new run is added; partial replacement is legal (decision 10); the ZIP comment is not preserved; overlay names derive from the skeleton name ([§2.5](#25-replacing-a-language), [overlay entry names](#overlay-entry-names)).
   *Refinement (2026-09-28, author's decision):* in a partial replacement a chapter the run does not cover keeps its existing overlay for the language, byte-identical with its digest unchanged; only covered chapters get new overlays ([§2.5](#25-replacing-a-language) step 3).
10. **G10 — Non-conformant input**: when `manifest.json` is not last or is listed twice, a producer MUST refuse to append and name the entry that follows the manifest; rewriting the whole file is a separate explicit operation ([§2.4](#append-nonconformant-input)).

---

## License

This specification is licensed under the
[Creative Commons Attribution 4.0 International License (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).
You are free to share and adapt it for any purpose, including commercially,
provided you give appropriate attribution.
