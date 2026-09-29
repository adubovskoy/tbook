# `.tbook` File Format Specification

**Format version:** 1
**Status:** Stable
**Last updated:** 2026-07-28
**Media type (proposed):** `application/x-tbook+zip`
**File extension:** `.tbook`
**License:** [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)

> **2026-07-03.** Additive, backward-compatible extensions under version 1 (per
> [§9.1](#91-optional-fields-and-forward-compatibility)): **figures** (body
> images with translatable captions, [§4.2](#42-figures)), **tables**
> ([§4.3](#43-tables)), and **footnotes** (inline markers + book-level
> translatable note bodies, [§4.4](#44-footnotes--notesjson)). A consumer that
> ignores all of them still renders correct, tappable prose.
>
> **2026-07-19.** No format change: the reference producer now also emits each
> chapter's title as the chapter's **first `heading`-role paragraph**
> ([§4.1](#41-paragraphstyles--per-paragraph-roles)), so the displayed title is
> translated and tappable like any prose. A consumer that renders
> `ChapterRef.title` above the text SHOULD skip its own copy when the chapter
> already opens with a heading paragraph.
>
> **2026-07-28.** Additive extension under version 1: the optional
> **`manifest.meta`** object ([§3.4](#34-meta--provenance-and-service-metadata))
> — provenance and service metadata *about the file* (producer version, LLM
> provider and models, prompt versions, run settings), plus a namespaced place
> for any other producer-defined bookkeeping. Purely informative: a consumer
> never needs it to render.

---

## 1. Introduction

A `.tbook` file is a self-contained, offline-first container for a
**language-learning ebook**. It pairs the original text of a book with a
**per-sentence translation** into one or more target languages, plus
**word-level alignment** between source and translation.

The defining feature is **tap-to-translate**: a reader application displays the
book in its original ("source") language; when the user taps a source word, the
app shows the full-sentence translation with exactly the translated word(s)
highlighted. All data needed for this is baked into the file at conversion time,
so the reader works fully offline with no network or API calls.

The format also preserves **content and formatting fidelity** so the rendered
book resembles the source EPUB: per-paragraph **roles** (chapter subtitles,
headings, scene breaks), per-sentence inline **emphasis** (italic/bold) runs,
body **figures** with translatable captions, **tables** with translatable cells,
and **footnotes** (inline markers + translatable bodies). All of these are
**optional** ([§4.1](#41-paragraphstyles--per-paragraph-roles)–[§4.4](#44-footnotes--notesjson),
[§5.5](#55-spans--inline-emphasis)–[§5.6](#56-notes--footnote-markers)); a
consumer **MAY** ignore them and still render correct, tappable prose.

### 1.1 Design model: source as pivot ("hub and spoke")

A `.tbook` is **multi-language by design**. The **source language is the
pivot**, and every target ("gloss") language aligns word-by-word *to the
source* — not pairwise to every other target. Adding a language therefore costs
one extra translation per sentence (**N−1**, not N² pairwise combinations).

Because each translation is produced *from* the source sentence, the alignment
is generated directly during translation. There is **no statistical word
aligner** and **no sentence-alignment layer** in the consumer.

### 1.2 Terminology

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** in
this document are to be interpreted as described in RFC 2119.

| Term | Meaning |
|------|---------|
| **Source** | The original language of the book (e.g. English). One per file. |
| **Target** / **gloss** | A language the book is translated into. One or more per file. |
| **Sentence** | The atomic translatable unit. Carries source text, word offsets, and one translation per target. |
| **Word** | A tappable token in the source sentence, identified by character offsets. |
| **Chunk** (align entry) | A contiguous span of the *translation* text mapped to the source word(s) it translates. |
| **Role** | A paragraph's structural kind: body (default), subtitle, heading, scene break, figure, or table ([§4.1](#41-paragraphstyles--per-paragraph-roles)). |
| **Span** (emphasis) | An inline italic/bold run inside a sentence's source text ([§5.5](#55-spans--inline-emphasis)). |
| **Figure** | A body image attached to a paragraph whose sentences are its caption ([§4.2](#42-figures)). |
| **Note** | A footnote/endnote: an inline marker in a sentence ([§5.6](#56-notes--footnote-markers)) plus a book-level body ([§4.4](#44-footnotes--notesjson)). |
| **Producer** | A tool that writes `.tbook` files (reference: the Go CLI [`tbook_converter`](https://github.com/adubovskoy/tbook_converter)). |
| **Consumer** | A tool that reads `.tbook` files (reference: the desktop reader [`tbook_desktop`](https://github.com/adubovskoy/tbook_desktop)). |

---

## 2. Container

A `.tbook` file **MUST** be a standard **ZIP archive** (PKZIP / `.zip`). The
reference producer uses the `DEFLATE` compression method; consumers **MUST**
support `DEFLATE` and **SHOULD** support `STORE` (no compression).

- Entries **MUST** be addressable by name (the reference consumer uses
  random-access `ZipFile` lookups by entry name, not streaming order).
- Entry names use `/` as the path separator and are **case-sensitive**.
- All textual entries (`manifest.json`, chapter files) **MUST** be UTF-8 encoded
  JSON with no byte-order mark.

### 2.1 Archive layout

```
book.tbook  (ZIP)
├── manifest.json          required — metadata + chapter index
├── cover.jpg              optional — cover image bytes
├── notes.json             optional — book-level footnote bodies (§4.4)
├── images/
│   ├── img1.jpg           optional — body images referenced by figures (§4.2)
│   └── …
└── chapters/
    ├── ch1.json           required (≥1) — one file per chapter
    ├── ch2.json
    └── …
```

| Entry | Required | Description |
|-------|----------|-------------|
| `manifest.json` | **Yes** | Top-level metadata and the ordered chapter index. |
| `cover.jpg` | No | Cover image. Present iff `manifest.cover` is non-null. |
| `notes.json` | No | Footnote bodies. Present iff `manifest.notes` is non-null ([§4.4](#44-footnotes--notesjson)). |
| `images/<name>` | No | Body image bytes, referenced from chapter `figures` ([§4.2](#42-figures)). |
| `chapters/<id>.json` | **Yes** (≥1) | One chapter payload per entry in `manifest.chapters`. |

A consumer **MUST** read `manifest.json` first and discover chapter entries
**only** through `manifest.chapters[].file`. It **MUST NOT** assume any other
entry naming, ordering, or count. Producers **MAY** include additional entries;
consumers **MUST** ignore unknown entries.

---

## 3. `manifest.json`

A single JSON object describing the book and indexing its chapters.

```json
{
  "formatVersion": 1,
  "title": "Alien Harvest",
  "author": "Robert Sheckley",
  "sourceLang": "en",
  "targetLangs": ["ru"],
  "cover": "cover.jpg",
  "chapters": [
    { "id": "ch1", "title": "Captain Hoban's Prologue", "file": "chapters/ch1.json" },
    { "id": "ch2", "title": "1", "file": "chapters/ch2.json" }
  ]
}
```

### 3.1 Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `formatVersion` | integer | **Yes** | Format version. **MUST** be `1` for this spec. See [§9](#9-versioning). |
| `title` | string | **Yes** | Book title. **MAY** be empty. |
| `author` | string | **Yes** | Author. Producer default when unknown: `"Unknown"`. **MAY** be empty. |
| `sourceLang` | string | **Yes** | Source language code (the pivot). See [§3.2](#32-language-codes). |
| `targetLangs` | array of string | **Yes** | Target language codes present in the file, in display order. **SHOULD** be non-empty. |
| `cover` | string \| null | **Yes** | Entry name of the cover image (e.g. `"cover.jpg"`), or `null` if the book has no cover. |
| `notes` | string \| null | No | Entry name of the footnote-bodies file (`"notes.json"`), or `null`/absent if the book has no notes ([§4.4](#44-footnotes--notesjson)). |
| `meta` | object | No | Provenance and service metadata *about the file* (producer, models, prompt versions, run settings, producer-defined extras). Informative only — a consumer never needs it to render ([§3.4](#34-meta--provenance-and-service-metadata)). |
| `chapters` | array of `ChapterRef` | **Yes** | Ordered chapter index; reading order. **MUST** contain ≥1 entry. |

`ChapterRef`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | **Yes** | Stable chapter identifier. Producer uses `ch1`, `ch2`, … (1-based). |
| `title` | string | **Yes** | Human-readable chapter title. Producer fallback: `"Chapter N"`. |
| `file` | string | **Yes** | ZIP entry name of the chapter payload (e.g. `"chapters/ch1.json"`). |

### 3.2 Language codes

Language codes **SHOULD** be lowercase ISO 639-1 two-letter codes (e.g. `en`,
`ru`, `de`, `es`, `fr`, `it`, `pt`, `uk`, `pl`, `ja`, `zh`, `ko`, `tr`, `nl`,
`ar`). The format does not constrain the code set; consumers **SHOULD** treat
codes as opaque keys and **MUST** match them exactly (case-sensitive) against
the keys of each sentence's `tr` map ([§5.3](#53-tr--translations)).

Every code listed in `targetLangs` **MUST** appear as a key in the `tr` map of
every sentence (possibly with an empty translation; see
[§5.3.1](#531-empty-and-missing-translations), which states the same rule from
the producer's side).

### 3.3 Cover image

When `cover` is non-null, the named entry **MUST** exist and contain raw image
bytes. The reference producer always names the entry `cover.jpg`, but the bytes
are copied verbatim from the source EPUB and **MAY** be any web image format
(JPEG, PNG, …). The `.jpg` extension is therefore **nominal**: consumers
**SHOULD** detect the actual format from the bytes and **MUST NOT** rely on the
extension.

### 3.4 `meta` — provenance and service metadata

`meta` is an **optional** object carrying information *about the file* rather
than its content: which tool produced it, with which language model, under which
prompt versions and settings — plus any other bookkeeping a producer wants to
keep with the book. It answers "where did this translation come from, and can I
reproduce it?", which matters because the content of a `.tbook` is
model-generated: a file is only as good as the model, prompt, and options that
made it.

`meta` is **informative**. A consumer **MUST NOT** require it to render a book,
and **MUST** treat it as opaque data — display it, log it, or ignore it, but
never derive rendering, segmentation, or tap behavior from it.

```json
{
  "formatVersion": 1,
  "title": "Alien Harvest",
  "…": "…",
  "meta": {
    "createdAt": "2026-07-19T08:02:55Z",
    "updatedAt": "2026-07-28T09:11:04Z",
    "runs": [
      {
        "at": "2026-07-19T08:02:55Z",
        "producer": { "name": "tbook-converter", "version": "1.4.0", "commit": "5abd168cb55c" },
        "targets": ["ru"],
        "provider": "openrouter",
        "models": {
          "translate": "google/gemini-2.5-flash",
          "align": "google/gemini-2.5-flash",
          "judge": "anthropic/claude-opus-5",
          "embed": "sentence-transformers/LaBSE"
        },
        "prompts": { "translate": "v5", "align": "v8", "judge": "j3" },
        "options": { "alignMode": "hybrid", "glossary": true, "judge": true, "batchSize": 24 }
      },
      {
        "at": "2026-07-28T09:11:04Z",
        "producer": { "name": "tbook-converter", "version": "1.5.1" },
        "targets": ["de"],
        "provider": "claude",
        "models": { "translate": "claude-sonnet-5", "align": "claude-sonnet-5" },
        "prompts": { "translate": "v5", "align": "v8" },
        "options": { "alignMode": "hybrid", "glossary": true }
      }
    ],
    "x-mytool": { "anything": "here" }
  }
}
```

**Registered keys.** All are optional; a producer emits what it knows.

| Key | Type | Description |
|-----|------|-------------|
| `createdAt` | string | RFC 3339 timestamp (UTC, `Z`) of the **first** assembly of this file. |
| `updatedAt` | string | RFC 3339 timestamp of the **most recent** assembly. Absent when the file was never rewritten. |
| `runs` | array of `RunRecord` | The producer runs that contributed to this file, **oldest first**. |

`RunRecord` — every field optional:

| Field | Type | Description |
|-------|------|-------------|
| `at` | string | RFC 3339 timestamp of the run. |
| `producer` | object | The tool: `name` (short stable id, e.g. `"tbook-converter"`), `version`, `commit` (VCS revision), `url`. |
| `targets` | array of string | Target languages this run produced or updated — a subset of `targetLangs` ([§3.1](#31-fields)). A run that adds a language lists only that language. |
| `provider` | string | Opaque id of the inference backend (e.g. `"openrouter"`, `"claude"`, `"ollama"`). Not constrained by this spec. |
| `models` | object | Pass name → model id, for the passes that ran. |
| `prompts` | object | Pass name → the producer's internal prompt/contract version for that pass (an opaque string). |
| `options` | object | The settings the run was invoked with — the "which switches" record. Free-form scalars/arrays keyed by option name. |

Well-known pass names for `models` and `prompts` (producers **MAY** use others):
`translate`, `align`, `repair`, `judge`, `escalate`, `embed` (a local embedding
aligner's encoder).

**Rules.**

- When present, `meta` **MUST** be a JSON object. Unknown keys inside it are
  **legal by construction** — a validator **MUST NOT** reject them, and a
  consumer **MUST NOT** fail on them.
- Producers **SHOULD** namespace non-registered keys with an `x-` prefix or a
  reverse-DNS name (`"x-mytool"`, `"dev.tbook.stats"`), so producer-specific
  bookkeeping can never collide with a key a later version of this spec
  registers. Registered keys **SHOULD** keep the meaning defined above.
- **No secrets, no machine identity.** `.tbook` files are shared. A producer
  **MUST NOT** write API keys, tokens, or other credentials into `meta`, and
  **SHOULD NOT** write absolute filesystem paths, private endpoint hostnames,
  account names, or anything else identifying the machine that ran the
  conversion. Record *what* was configured (`"provider": "ollama"`), not *where*
  it lived.
- `runs` is **append-only**, oldest → newest. A producer that rewrites an
  existing file — e.g. adding a target language to an already-assembled book —
  **SHOULD** carry the existing `meta` over *including keys it does not
  understand* and append its own record, so provenance is never silently lost.
  It **MAY** collapse a record identical to the previous one but for its
  timestamp, and **MAY** drop the oldest records to stay small.
- **Keep it small.** `manifest.json` is parsed on every book open, and a library
  view parses one per book, so `meta` **SHOULD** stay within a few kilobytes.
  Bulk provenance (per-sentence scores, request logs, full prompt texts) belongs
  in a producer-defined archive entry of its own — [§2.1](#21-archive-layout)
  permits extra entries and requires consumers to ignore them — not here.
- Nothing in `meta` carries format semantics. In particular `prompts.align` is
  the producer's internal alignment cache-key version
  ([§8.2](#82-word-matching-the-chunk-intermediate-and-the-cache-informative)),
  which is unrelated to `formatVersion` ([§9](#9-versioning)) and moves
  independently of it.

---

## 4. Chapter files — `chapters/<id>.json`

Each chapter referenced by `manifest.chapters[].file` is a JSON object holding
the chapter's prose, structured as **paragraphs → sentences**.

```json
{
  "paragraphs": [
    [ { /* Sentence */ }, { /* Sentence */ } ],
    [],
    [ { /* Sentence */ } ]
  ],
  "paragraphStyles": ["subtitle", "sceneBreak", "body"],
  "figures": [ { "para": 4, "image": "images/img2.jpg" } ],
  "tables":  [ { "para": 9, "rows": [ /* … */ ] } ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `paragraphs` | array of array of `Sentence` | **Yes** | Ordered paragraphs; each paragraph is an ordered list of sentences. A paragraph **MAY** be empty (e.g. a scene break, [§4.1](#41-paragraphstyles--per-paragraph-roles)); the array **MAY** be empty. |
| `paragraphStyles` | array of string | No | Per-paragraph **roles**, index-aligned with `paragraphs`. Absent/short ⇒ missing entries are `"body"`. See [§4.1](#41-paragraphstyles--per-paragraph-roles). |
| `figures` | array of `Figure` | No | Images attached to `figure`-role paragraphs ([§4.2](#42-figures)). |
| `tables` | array of `Table` | No | Tables attached to `table`-role paragraphs ([§4.3](#43-tables)). |

Rendering: a consumer **SHOULD** render paragraphs as block-level units
(paragraph breaks between them) and sentences as inline, contiguous text within
a paragraph (typically joined with a single space). Only chapter-body prose
appears as sentences; the book title and epigraphs are **not** included. The
chapter heading lives in `ChapterRef.title`, and the reference producer
additionally re-emits it as the chapter's first `heading`-role paragraph so the
displayed title is translated and tappable ([§4.1](#41-paragraphstyles--per-paragraph-roles)).

### 4.1 `paragraphStyles` — per-paragraph roles

`paragraphStyles` is an **optional** array of role strings, **parallel** to
`paragraphs`: `paragraphStyles[i]` is the role of `paragraphs[i]`. It exists so a
consumer can reproduce the source book's block structure.

| Role | Meaning | Typical rendering |
|------|---------|-------------------|
| `body` | Ordinary prose paragraph (the **default**). | Normal body text. |
| `subtitle` | A chapter subtitle/dateline directly under the heading. | Centered, italic/secondary. |
| `heading` | An in-body heading (e.g. a section title). | Larger/bold, centered. |
| `sceneBreak` | A scene divider. The paragraph carries **no sentences** (`[]`). | A centered ornament (e.g. `* * *`) or vertical gap. |
| `figure` | A body image's **caption** paragraph ([§4.2](#42-figures)). **MAY** be empty (caption-less image). | The image (from `figures`), caption below it. |
| `table` | A table anchor. The paragraph carries **no sentences** (`[]`); content lives in `tables` ([§4.3](#43-tables)). | The rendered table. |

Rules:

- The array, when present, **SHOULD** be index-aligned with `paragraphs`.
  Producers **MAY** truncate trailing `body` entries; consumers **MUST** treat a
  missing or out-of-range entry — and any unknown role string — as `body`.
- A `sceneBreak` or `table` paragraph **MUST** be an empty sentence list (`[]`).
  Every other role carries its sentences normally.
- A producer **SHOULD** omit `paragraphStyles` entirely when every paragraph is
  `body`, so a chapter with no special formatting omits the field entirely.
- Roles are **presentational hints**. A consumer **MAY** render any role as plain
  body text; doing so never loses tap-to-translate behavior. This keeps the field
  fully backward-compatible. (A minimal consumer renders a `figure` paragraph as
  its caption text and a `table` paragraph as nothing.)

The chapter's own title always lives in `ChapterRef.title` ([§3.1](#31-fields))
— that is the TOC/navigation title. Since 2026-07-19 the reference producer
**also** re-inserts it as the chapter's first `heading`-role paragraph, carrying
ordinary sentences, so the rendered title is translated, aligned, and tappable;
files from older producers lack it. A consumer **MAY** display
`ChapterRef.title` as a heading above the paragraphs, but **SHOULD** skip that
copy when the chapter already opens with a `heading` paragraph, so the title is
not rendered twice.

### 4.2 `figures`

A **figure** is a body image anchored to a paragraph. The paragraph (role
`figure`) carries the **caption** as ordinary sentences — translated, aligned,
and tappable exactly like prose. The image itself is metadata:

```json
{ "para": 4, "image": "images/img2.jpg", "alt": "" }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `para` | integer | **Yes** | Index into `paragraphs` of the caption paragraph. **MUST** be in range, and that paragraph's role **MUST** be `figure`. |
| `image` | string | **Yes** | ZIP entry name of the image (e.g. `"images/img2.jpg"`). The entry **MUST** exist. |
| `alt` | string | No | Source alt text, if any. |

- As with the cover ([§3.3](#33-cover-image)), the extension is **nominal**:
  consumers **SHOULD** detect the format from the bytes.
- A consumer renders the image as a block at the paragraph's position, caption
  below. A consumer ignoring `figures` renders just the caption text (valid
  prose); an image with no caption degrades to nothing.
- The reference producer downscales oversized images (long edge > 1400 px or
  > 300 KB → JPEG, long edge 1200 px) to keep archives reader-sized.

### 4.3 `tables`

A **table** is anchored to an empty paragraph (role `table`); its cells are
mini-paragraphs of ordinary sentences — translated, aligned, and tappable:

```json
{
  "para": 9,
  "rows": [
    [ { "sentences": [ { /* Sentence */ } ], "header": true },
      { "sentences": [ { /* Sentence */ } ], "header": true } ],
    [ { "sentences": [ { /* Sentence */ }, { /* Sentence */ } ] },
      { "sentences": [] } ]
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `para` | integer | **Yes** | Index into `paragraphs` of the (empty) anchor paragraph. **MUST** be in range with role `table`. |
| `rows` | array of array of `TableCell` | **Yes** | Rows in source order. **MUST** be non-empty. |

`TableCell`:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sentences` | array of `Sentence` | **Yes** | The cell's content. **MAY** be empty. |
| `header` | boolean | No | `true` for a header cell (default `false`). |

Rows **MAY** have differing cell counts (column spans are not modeled; a
consumer **SHOULD** render by row). A consumer ignoring `tables` loses the table
content — exactly what a version-1-only consumer already did — so the field is
still additive.

### 4.4 Footnotes — `notes.json`

Footnotes/endnotes have two halves: an inline **marker** inside a sentence
([§5.6](#56-notes--footnote-markers)) and a **body** stored book-level in the
`notes.json` entry, named by `manifest.notes`. The body is paragraphs of
ordinary sentences — translated, aligned, and tappable, so the note popup gets
tap-to-translate for free.

`notes.json` is a single JSON object mapping **note id** → `Note`:

```json
{
  "n17": {
    "label": "1",
    "kind": "citation",
    "paragraphs": [ [ { /* Sentence */ } ] ]
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `label` | string | **Yes** | Display label of the marker ("1", "*", "†", …). |
| `kind` | string | No | `"note"` (default) — a content note; `"citation"` — a bibliographic reference. Unknown values **MUST** be treated as `"note"`. |
| `paragraphs` | array of array of `Sentence` | **Yes** | The note body. |

- Note ids are opaque strings, unique within the file (producer: `n1`, `n2`, …
  in first-reference order).
- **Citations MAY be left untranslated** (empty `tr` entries, [§5.3.1](#531-empty-and-missing-translations)):
  bibliographic references have near-zero learner value, and skipping them saves
  producer tokens. The reference producer's `--skip-citations` does exactly this.
- A consumer **SHOULD** render a marker as a superscript `label` and open the
  note body on tap (e.g. a bottom sheet). A consumer ignoring notes entirely
  renders clean prose — the marker text is **not** part of `src`.

---

## 5. The `Sentence` object

The core unit of the format.

```json
{
  "src": "I was in the middle of the whole thing with Stan and Julie.",
  "words": [[0,1],[2,5],[6,8],[9,12],[13,19],[20,22],[23,26],[27,32],[33,38],[39,43],[44,48],[49,52],[53,58]],
  "tr": {
    "ru": {
      "text": "Я был в самой середине всего этого с Стэном и Джули.",
      "align": [
        { "t": [0,1],   "w": [0] },
        { "t": [2,5],   "w": [1] },
        { "t": [6,7],   "w": [2] },
        { "t": [8,13],  "w": [7] }
      ]
    }
  },
  "spans": [ { "s": 33, "e": 38, "k": "i" } ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `src` | string | **Yes** | The original source-language sentence, exactly as displayed. |
| `words` | array of `[start, end]` | **Yes** | Character offsets of each **tappable source word** in `src`. **MAY** be empty. |
| `tr` | object (map) | **Yes** | Maps each target language code → `Translation` ([§5.3](#53-tr--translations)). **MAY** be empty. |
| `spans` | array of `Span` | No | Inline italic/bold runs over `src` ([§5.5](#55-spans--inline-emphasis)). Omitted when there is no emphasis. |
| `notes` | array of `NoteRef` | No | Inline footnote markers ([§5.6](#56-notes--footnote-markers)). Omitted when there are none. |

### 5.1 `src`

`src` is the verbatim source text of the sentence. It has already been
normalized by the producer: whitespace collapsed to single spaces, non-breaking
spaces converted, trimmed. Consumers **SHOULD** display it as-is and **MUST**
treat it as the coordinate space for `words` offsets ([§6](#6-character-offset-semantics)).

### 5.2 `words` — tappable source tokens

`words` is an ordered array. Each element is a two-element integer array
`[start, end]` giving the **half-open** character range `[start, end)` of one
tappable word within `src`. The word's text is `src.substring(start, end)`.

Invariants — for every `words[i] = [a, b]`:

- **MUST** have exactly two elements.
- **MUST** satisfy `0 ≤ a < b ≤ length(src)`.
- Ranges are in **left-to-right order** and **SHOULD NOT** overlap.
- They need not be contiguous: punctuation and whitespace between words are
  intentionally **not** covered (only letter-runs are tappable).

The index `i` of a word in this array is its **word index** — the identifier
used by alignment `w` values ([§5.4](#54-align--word-level-alignment)) and by
the tap interaction ([§7](#7-tap-to-translate-algorithm)).

A "word" in the reference producer is a run of letters allowing intra-word
apostrophes and hyphens (regex: `\p{L}[\p{L}\p{M}]*(?:['’\-]\p{L}[\p{L}\p{M}]*)*`).
Numbers and standalone punctuation are not tokenized as tappable words.

### 5.3 `tr` — translations

`tr` maps a target language code (a key from `manifest.targetLangs`) to a
`Translation` object:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `text` | string | **Yes** | The full translated sentence in the target language, as displayed. **MAY** be empty. |
| `align` | array of `AlignChunk` | **Yes** | Word-level alignment spans ([§5.4](#54-align--word-level-alignment)). **MAY** be empty. |
| `q` | number | No | **Optional, informative** alignment-coverage score in `[0, 1]` — the fraction of translated words that highlight a source word. A producer **MAY** emit it so tools can flag low-confidence sentences for re-alignment; a consumer **MUST** ignore it (it never affects rendering). The reference producer emits it (2 decimals) for every non-empty translation. |

#### 5.3.1 Empty and missing translations

A sentence that has not been translated for a given language **MUST** still carry
the key with an empty translation: `{"text": "", "align": []}`. Consumers
**MUST** treat an empty `text` as "no translation available" and render the
source sentence without a gloss. Producers **MUST NOT** omit a `targetLangs`
entry from `tr`; they **MAY**, however, leave it empty.

### 5.4 `align` — word-level alignment

`align` is the heart of tap-to-translate. Each `AlignChunk` maps a span of the
**translation** `text` to the **source word indices** it translates:

```json
{ "t": [8, 13], "w": [7] }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `t` | array `[start, end]` | **Yes** | Half-open char range `[start, end)` into `Translation.text`. |
| `w` | array of integer | **Yes** | Source **word indices** (into `Sentence.words`) that this span translates. **MAY** be empty (`[]`). |

Semantics and invariants:

- **Word granularity (the quality contract).** Each chunk **SHOULD** cover a
  single translated word (with its attached punctuation), aligned **by meaning**
  to the source word(s) it renders — *not* by position. Coarse phrase-level
  chunks that lump many words together are a known defect: tapping one source
  word then lights up an entire phrase. This is a quality rule, not a
  well-formedness rule — see [§8](#8-alignment-quality-rules-informative).
- `t` **MUST** have two elements with `0 ≤ start ≤ end ≤ length(text)`.
- Every index in `w` **MUST** satisfy `0 ≤ index < length(words)`.
- `w` **MAY** be empty — used for a target word inserted with no source
  counterpart (e.g. a required article or particle). Such a chunk participates
  in `text` but is never highlighted by any tap.
- A single source word index **MAY** appear in multiple chunks (one-to-many
  alignment across reordering). A chunk **MAY** map to multiple source indices
  (a multi-word source unit translated as one target word).
- The chunks' `t` spans **need not** cover all of `text` and **need not** be
  contiguous; gaps correspond to spacing/punctuation between aligned words —
  and, since contract v6, to any target words the align pass failed to place
  (`text` is the raw translation, canonical and complete regardless;
  [§8.1](#81-locating-fragments-in-the-canonical-text-informative)).
- Every `t` span lies inside `text` at the word it highlights. This is a
  producer guarantee, not something consumers must verify.

### 5.5 `spans` — inline emphasis

`spans` is an **optional** array marking inline italic/bold runs in the
**source** sentence `src`. It lets a consumer render `<em>`/`<strong>` styling
from the original book. Each `Span`:

```json
{ "s": 33, "e": 38, "k": "i" }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `s` | integer | **Yes** | Half-open start char offset `[s, e)` into `src` ([§6](#6-character-offset-semantics)). |
| `e` | integer | **Yes** | End char offset; `s < e`. |
| `k` | string | **Yes** | Emphasis kind: `"i"` (italic) or `"b"` (bold). |

Semantics and invariants:

- Offsets are **character offsets into `src`** — the same coordinate space as
  `words` ([§5.2](#52-words--tappable-source-tokens)) — half-open, with
  `0 ≤ s < e ≤ length(src)`. A span styles `src.substring(s, e)`.
- `k` **MUST** be `"i"` or `"b"`. A consumer **MUST** ignore a span with any
  other kind (forward-compatibility for future kinds).
- Spans **MAY** overlap. Bold-italic text is represented as two spans over the
  same range (one `"i"`, one `"b"`); a consumer **unions** the styles of all
  spans covering a character.
- Spans cover only emphasis; they need not (and usually do not) cover all of
  `src`. They are **independent of** `words` and `align`: emphasis and
  tap-to-translate are orthogonal, and a span boundary need not fall on a word
  boundary.
- Emphasis is a **presentational hint**. A consumer **MAY** ignore `spans` and
  render plain text without losing any tap-to-translate behavior.

A robust consumer **MUST** clamp/skip out-of-range or degenerate spans
([§6](#6-character-offset-semantics)), exactly as for `words` and `align`.

### 5.6 `notes` — footnote markers

`notes` is an **optional** array of inline footnote markers. Each `NoteRef`:

```json
{ "p": 44, "id": "n17", "label": "1" }
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `p` | integer | **Yes** | **Insertion point** in `src`: the character offset the superscript marker renders at, `0 ≤ p ≤ length(src)`. Unlike span offsets this is a point, not a range, and **MAY** equal `length(src)` (marker after the last character). |
| `id` | string | **Yes** | Note id — a key of the `notes.json` object ([§4.4](#44-footnotes--notesjson)). |
| `label` | string | **Yes** | The marker's display text ("1", "*", …). |

Semantics and invariants:

- The marker's label is **NOT part of `src`** — the producer strips it from the
  prose. This is deliberate: marker digits glued into sentence text pollute the
  translation and the alignment (and did, before this field existed). A consumer
  ignoring `notes` therefore renders clean prose.
- `p` shares the coordinate space of `words`/`spans` ([§6](#6-character-offset-semantics)).
  A robust consumer clamps `p` into `[0, length(src)]`.
- Markers are **independent of** `words` and `align`: they carry no
  tap-to-translate information, and tapping the marker opens the note instead.
- An `id` that does not resolve in `notes.json` **SHOULD** be ignored (render
  nothing).

---

## 6. Character offset semantics

**This section is normative and easy to get wrong.**

All offsets — `words[i]`, `align[].t`, `spans[].{s,e}`, and `notes[].p` — are
**character offsets** (half-open `[start, end)` for ranges, an insertion point
for `p`) into the relevant string (`src` for `words`/`spans`/`notes`,
`Translation.text` for `t`).

The reference **producer** is the Go CLI, which emits **rune** indices — i.e.
**Unicode code points**. The reference **consumer** renders in a WebView, where
JavaScript/TypeScript `String.length` and `String.slice` count **UTF-16 code
units**.

For every character in the **Basic Multilingual Plane** (which includes Latin,
Cyrillic, Greek, Hebrew, Arabic, most CJK, and all common European-language
text), one code point equals one UTF-16 code unit, so producer and consumer
offsets agree exactly.

They diverge **only** for characters **outside the BMP** (e.g. emoji, rare CJK
extension characters, some historic scripts), which code-point indexing counts
as 1 and UTF-16 counts as 2. Implementations:

- Producers **SHOULD** emit offsets as **Unicode code-point** indices and
  **SHOULD** avoid astral-plane characters in `src`/`text` where alignment
  precision matters.
- Consumers **MUST** be defensive: clamp every offset into the valid range of
  the target string before use, and skip degenerate (`end ≤ start`) spans. The
  reference consumer does exactly this (see [§7](#7-tap-to-translate-algorithm)).

A future format version **MAY** fix one encoding normatively; under version 1,
treat astral-plane misalignment as a latent edge case mitigated by consumer
clamping.

---

## 7. Tap-to-translate algorithm

This is the canonical consumer interaction the format exists to support.

**Given** a tapped position in the rendered source sentence:

1. **Resolve the word index.** Find the index `i` such that the tap falls within
   `words[i] = [a, b)` in `src`. If the tap is outside every word range, do
   nothing.
2. **Pick the gloss language** `L` (from `targetLangs`; the file may offer a
   picker when more than one target exists).
3. **Look up** `tr[L]`. If absent or `text` is empty, show "no translation".
4. **Render** `tr[L].text`, and **highlight** every chunk whose `w` contains `i`:
   for each `chunk` in `tr[L].align` with `i ∈ chunk.w` and `chunk.t.length ≥ 2`,
   style the substring `text[ clamp(t[0]) , clamp(t[1]) )`.

Reference consumer (TypeScript) — note the defensive clamping per [§6](#6-character-offset-semantics):

```typescript
/** [start,end) ranges in `tr.text` to highlight for the given source wordIndex. */
function translationHighlightRanges(tr: Translation, wordIndex: number): Array<[number, number]> {
    const n = tr.text.length;
    const out: Array<[number, number]> = [];
    for (const chunk of tr.align ?? []) {
        if (!(chunk.w ?? []).includes(wordIndex)) continue;
        if (!chunk.t || chunk.t.length < 2) continue;
        const a = clamp(chunk.t[0], 0, n);
        const b = clamp(chunk.t[1], a, n);
        if (b > a) out.push([a, b]);   // skip degenerate spans
    }
    return out;
}
```

A robust consumer **MUST** tolerate out-of-range offsets and indices (clamp/skip
rather than crash), because alignment is produced by a language model and may
contain rare imperfections.

---

## 8. Alignment quality rules (informative)

These rules govern *how good* a file is, not *whether it is well-formed*. They
are reproduced here because they are the format's most important quality
contract.

- **One chunk per target word.** The translator emits one `align` chunk per
  translated word, each mapped to the source word(s) it renders **by meaning**,
  even across reordering — never by contiguous positional ranges.
- **Why it matters.** If chunks are coarse (a few big phrase-level spans with
  contiguous source-index ranges), tapping a single source word highlights an
  entire phrase. Fine word-level chunks make a tap highlight exactly the
  corresponding target word(s).

Worked example. Source words `0=Stan 1=went 2=to 3=the 4=living 5=room`;
translation "Стэн прошёл в гостиную":

```json
[
  {"tgt":"Стэн","en":"Stan"},
  {"tgt":"прошёл","en":"went"},
  {"tgt":"в","en":"to"},
  {"tgt":"гостиную","en":["living","room"]}
]
```

This is the **model-emitted** chunk shape: each chunk carries the target fragment
`tgt` plus `en` — the **source word(s) as text**, *not* indices. Tapping "Stan"
highlights exactly «Стэн»; "living room" (`["living","room"]`) highlights
«гостиную». The producer matches that text back to source-word indices and
computes the final `{t, w}` offset form — see
[§8.2](#82-word-matching-the-chunk-intermediate-and-the-cache-informative).

### 8.1 Locating fragments in the canonical text (informative)

Since contract v6 the **raw pass-1 translation is the canonical `text`**,
stored verbatim; the align pass never constructs text. Each align chunk's
fragment is **located** inside the raw text instead: both are folded to
lowercase letters/digits (punctuation and case ignored), and the folded
fragment is searched for from a cursor that only moves forward — fragments
arrive in target order, so repeated words resolve to successive occurrences.
The `t` span covers the fragment's letters in the raw text (not surrounding
punctuation), so offsets always match the emitted `text`.

Consequences: an echo that drops commas or changes case still aligns; a
fragment with **no letters** (`").",` `"—"`) never yields a highlight; a
*mapped* fragment that cannot be located means the echo rewrote the
translation — the sentence is rejected and retried, and if it never aligns it
ships as raw text with an empty alignment (fallback, [§8.2](#82-word-matching-the-chunk-intermediate-and-the-cache-informative)).
(Pre-v6, `text` was rebuilt by concatenating the echoed fragments with spacing
rules — which let a sloppy echo strip punctuation from the shipped text.)

### 8.2 Word-matching: the chunk intermediate and the cache (informative)

The final `{t, w}` alignment is **derived deterministically by the producer**;
the translation model never emits character offsets and — **by design** — never
emits source-word **indices** either. Instead it copies source-word **text**,
which the producer matches back to indices. This **match-by-text** approach is
deliberately robust against the model losing count of indices: the model only
has to echo the word, and the producer locates it.

**Production is two-pass (recommended).** Translation and alignment are produced
in *separate* model passes: pass 1 translates (`{id, src}` → translation text),
pass 2 aligns given that finished translation (`{id, src, words, tr}` →
`{tgt, en}` chunks). Asking one pass to translate *and* emit the reverse
word-alignment makes a model fall back to a **positional** alignment (target
word *i* ↔ source word *i*) at batch scale — structurally valid yet semantically
wrong, book-wide. Decoupling the two greatly reduces this, but does **not**
eliminate it: a weak model still positionally-drifts a fraction of sentences and
mistranslates others, and these survive structural validation (see
[§10.4](#104-alignment-coverage-and-correctness-informative)).
A semantic verification pass — an independent LLM judge comparing source, target,
and mapping, re-doing what fails — is the only reliable quality gate. The
producer caches raw pass-1 translations under a **separate key namespace**
(`…|tr|…`) until pass 2 aligns them, and versions the two contracts
independently (`TrPromptVersion = "v4"` for pass 1; `PromptVersion = "v6"` for
pass 2 and the final entries), so an alignment-contract change re-aligns a book
without re-translating it.

The **numbered echo** align contract, introduced in v5 (chosen after A/B
measurement against
plain text echoing on a cheap model — it roughly **halves** wrong word-pairs and
also beat a stronger model running the older contract):

- The align input carries the sentence's tokenized words as a numbered list
  (`"words": "0:Los 1:hábitos 2:son …"`).
- The model returns, per target word, `{"tgt": "<target fragment>",
  "en": ["<index:text>", …]}` — echoing **both** the source word's index and its
  text (e.g. `"3:interés"`). The field is named `en` regardless of the actual
  source language. An empty `en` (`[]` or `""`) marks a target word inserted with
  no source counterpart — the prompt stresses this: an inserted word silently
  consuming a source word is what starts an off-by-one cascade.
- The align pass runs in **small batches** (producer default: a quarter of the
  translate batch): long alignment items at large batch sizes are where cheap
  models drift.

The pipeline, per sentence:

1. **Model output.** An ordered list of chunks as above. Tolerated variants: a
   plain-text token without the `index:` prefix (the v4 form); a string instead
   of an array; a multi-word *string* (`"4:living room"`), which is **split on
   whitespace** into tokens so it resolves word-by-word.
2. **Resolve tokens → indices** (`resolveEnToIndices`). An `index:text` token
   resolves to its **index** when the echoed text matches that source word
   (normalized: trimmed, surrounding punctuation stripped, lowercased) — the
   text acts as a checksum on the index. On any mismatch, out-of-range index, or
   plain-text token, it falls back to **match-by-text**: matched against the
   lowercased source words, consumed **left-to-right**, so repeated source words
   map to successive occurrences; a word reused beyond its occurrence count
   (one-to-many) falls back to its **last** occurrence. Tokens that match
   nothing are dropped. The result is the internal chunk shape
   `{tgt, src: [idx, …]}`.
3. **Locate** ([§8.1](#81-locating-fragments-in-the-canonical-text-informative))
   each `tgt` fragment inside the raw pass-1 translation (the canonical `text`),
   computing its `[start, end)` `t` span, and validate the `src` indices against
   the word count — yielding the final `{t, w}` chunks stored in the file. A
   leading list marker (`1.`, `2)`) present in `src` but dropped from the raw
   translation is restored deterministically before locating.

> **Contract note.** A chunk carrying only bare `src` **indices** (no `en`) is
> **not** aligned by the current producer: `resolveEnToIndices` reads `en` only,
> so such a chunk resolves to empty `w` and highlights nothing. Tools that emit
> chunks for this pipeline **MUST** emit `index:text` (or plain text) tokens
> under `en`.

The producer additionally rejects pass-1 output containing leaked special tokens
(`<bos>`, `<eos>`, `<|…|>`, U+FFFD) — a cheap-model failure mode that would
otherwise bake artifacts into `text` — and re-translates those sentences.

Per-sentence results are cached on disk (`converter/.tbook_cache/`) as
`{"text": "...", "align": [...]}`, keyed by
`promptVersion | model | sourceLang | targetLang | src` (SHA-256). The cache is
an **implementation detail** of the producer, **not** part of the `.tbook` file,
but it shares the `Translation` shape. The align prompt version (the
`PromptVersion` constant, currently `v6`) keys the final aligned entries; the
translate version (`TrPromptVersion`, currently `v4`) keys the raw pass-1
entries (`…|tr|…`): changing either contract's rules **MUST** bump its version
so the cache regenerates instead of serving stale (e.g. v3 combined-pass,
positionally-drifted) output.

Two optional producer passes extend this scheme without touching it:

- **Glossary** (`--glossary`): a one-call pre-pass extracts the book's recurring
  key terms + proper nouns with fixed target translations, appended to every
  translate prompt so terminology stays consistent across batches. Because the
  prompt changes, the cache's *model* component gains a glossary-hash suffix
  (`model+g:<hash12>`) — glossary translations never mix with plain ones, and
  the plain cache stays valid.
- **Judge** (`--judge`, [§10.4](#104-alignment-coverage-and-correctness-informative)):
  verdicts are cached per (judge model, src, translation, alignment) under
  `judge-*.json`, so re-runs only judge what changed.

Note that stripping footnote markers ([§5.6](#56-notes--footnote-markers))
changes `src` for the affected sentences, so those cache keys change naturally —
formerly marker-polluted sentences re-translate; everything else cache-hits.

---

## 9. Versioning

- `manifest.formatVersion` is an integer. This document specifies version **1**,
  the first published version of the format.
- A consumer **SHOULD** check `formatVersion` and **MAY** refuse or warn on a
  version it does not understand.
- Consumers **MUST** ignore unknown JSON fields and unknown ZIP entries, so that
  future additive, backward-compatible changes do not break older readers. (The
  reference consumer decodes with `ignoreUnknownKeys = true`.)
- The field **SHOULD** always be present; the reference consumer defaults a
  missing `formatVersion` to the current version (`1`) and parses best-effort.
- `formatVersion` (the **on-disk file format**) is independent of the producer's
  internal **alignment-prompt cache key** (`PROMPT_VERSION`,
  [§8.2](#82-word-matching-the-chunk-intermediate-and-the-cache-informative)),
  which may carry a different value; its only appearance in a `.tbook` is as an
  informative provenance string in `meta.runs[].prompts`
  ([§3.4](#34-meta--provenance-and-service-metadata)). They version different
  things and need not move together.

### 9.1 Optional fields and forward compatibility

These fields are **optional** and exist for presentation/content fidelity:

- `chapters[].paragraphStyles` — per-paragraph roles ([§4.1](#41-paragraphstyles--per-paragraph-roles)).
- `Sentence.spans` — inline emphasis ([§5.5](#55-spans--inline-emphasis)).
- `chapters[].figures` + `images/*` — body images ([§4.2](#42-figures)).
- `chapters[].tables` — tables ([§4.3](#43-tables)).
- `Sentence.notes` + `manifest.notes` + `notes.json` — footnotes ([§4.4](#44-footnotes--notesjson), [§5.6](#56-notes--footnote-markers)).
- `Translation.q` — alignment-coverage score ([§5.3](#53-tr--translations)).
- `manifest.meta` — provenance / service metadata ([§3.4](#34-meta--provenance-and-service-metadata)); itself an open-ended, namespaced extension point.

A **minimal consumer MAY ignore all of them** and still render correct, tappable
prose: omitted roles render as body paragraphs (an empty `sceneBreak`/`table`
paragraph as a blank gap, a `figure` paragraph as its caption text), omitted
spans render as plain text, and omitted notes render clean prose without
markers. None of them touch `src`, sentence segmentation, or alignment. Future
versions that add further optional fields **SHOULD** follow the same rule — a
consumer ignoring them must still render valid prose.

---

## 10. Conformance

### 10.1 A conformant `.tbook` file MUST

1. Be a ZIP archive containing a valid UTF-8 JSON `manifest.json` with
   `formatVersion = 1` and all required manifest fields.
2. Contain every chapter entry named by `manifest.chapters[].file`, each a valid
   chapter object with a `paragraphs` array. If `paragraphStyles` is present,
   every entry **MUST** be a string, and each `sceneBreak`/`table` entry's
   paragraph **MUST** be empty (`[]`).
3. For every sentence — in chapter prose, figure captions, table cells, and note
   bodies alike: include `src`; satisfy all `words` invariants
   ([§5.2](#52-words--tappable-source-tokens)); include a `tr` entry for each
   `targetLangs` code (possibly empty).
4. For every non-empty translation: satisfy all `align` invariants
   ([§5.4](#54-align--word-level-alignment)) — every `t` in range of `text`,
   every `w` index in range of `words`.
5. For every `spans` entry: satisfy the `Span` invariants
   ([§5.5](#55-spans--inline-emphasis)) — `0 ≤ s < e ≤ length(src)` and
   `k ∈ {"i", "b"}`.
6. Include `cover.jpg` (or the named cover entry) iff `manifest.cover` is
   non-null; include `notes.json` (or the named notes entry) iff
   `manifest.notes` is non-null.
7. For every `figures` entry: `para` in range with role `figure`, and `image`
   naming an existing archive entry. For every `tables` entry: `para` in range
   with role `table`, and non-empty `rows`. For every `notes` marker:
   `0 ≤ p ≤ length(src)` and `id` present in `notes.json`.
8. If `manifest.meta` is present: be a JSON object carrying no credentials
   ([§3.4](#34-meta--provenance-and-service-metadata)). Its keys are otherwise
   unconstrained — a validator **MUST NOT** reject unknown ones — and nothing
   else in the file may depend on it.

### 10.2 A conformant consumer MUST

1. Read `manifest.json` first and discover chapters only via `chapters[].file`.
2. Ignore unknown fields and unknown archive entries.
3. Treat an empty `tr[L].text` as "no translation."
4. Clamp/skip out-of-range offsets, word indices, emphasis spans, and note
   markers instead of failing ([§6](#6-character-offset-semantics), [§7](#7-tap-to-translate-algorithm)).
5. Treat a missing/out-of-range/unknown `paragraphStyles` entry as `body`, and
   ignore a `spans` entry whose `k` is neither `"i"` nor `"b"`.
6. If it renders notes: ignore a marker whose `id` is absent from `notes.json`,
   and treat an unknown `Note.kind` as `"note"`. If it renders figures/tables:
   skip an entry whose `para` is out of range and detect image formats from
   bytes, not extensions.
7. Treat `manifest.meta` as opaque, optional data: never require it, never fail
   on its contents, and never derive rendering or tap behavior from it
   ([§3.4](#34-meta--provenance-and-service-metadata)).

### 10.3 Validation snippet (reference)

The Go producer validates automatically after assembly
(`converter/internal/tbook/validate.go`). This standalone Python snippet asserts
the same structural invariants, checks for empty translations, and reports
**alignment coverage** ([§10.4](#104-alignment-coverage-and-correctness-informative)) — handy for
checking any `.tbook` independently:

```python
import zipfile, json, sys
z = zipfile.ZipFile(sys.argv[1])
man = json.loads(z.read('manifest.json'))
notes = json.loads(z.read(man['notes'])) if man.get('notes') else {}
entries = set(z.namelist())
total = empty = err = 0
if not isinstance(man.get('meta', {}), dict):      # §3.4 — object, else opaque
    err += 1
tgt_words = tgt_aligned = src_total = src_covered = 0

def check(s):
    global total, empty, err, tgt_words, tgt_aligned, src_total, src_covered
    total += 1
    L = len(s['src'])
    for w in s['words']:
        if not (len(w) == 2 and 0 <= w[0] < w[1] <= L):
            err += 1
    for tr in s['tr'].values():
        if not tr['text']:
            empty += 1
        tl = len(tr['text'])
        covered = set()
        for ch in tr['align']:
            t = ch['t']
            if not (0 <= t[0] <= t[1] <= tl):
                err += 1
            for x in ch['w']:
                if not (0 <= x < len(s['words'])):
                    err += 1
                else:
                    covered.add(x)
            a, b = max(0, min(t[0], tl)), max(0, min(t[1], tl))
            if any(c.isalnum() for c in tr['text'][a:b]):   # a rendered word
                tgt_words += 1
                if ch['w']:
                    tgt_aligned += 1
        src_total += len(s['words'])
        src_covered += len(covered)
    for sp in s.get('spans', []):                  # inline emphasis
        if not (0 <= sp['s'] < sp['e'] <= L) or sp['k'] not in ('i', 'b'):
            err += 1
    for nr in s.get('notes', []):                  # footnote markers
        if not (0 <= nr['p'] <= L) or nr['id'] not in notes:
            err += 1

for c in man['chapters']:
    ch = json.loads(z.read(c['file']))
    styles = ch.get('paragraphStyles', [])
    role = lambda i: styles[i] if i < len(styles) else 'body'
    for i, para in enumerate(ch['paragraphs']):
        if role(i) in ('sceneBreak', 'table') and para:
            err += 1
        for s in para:
            check(s)
    for f in ch.get('figures', []):                # figures
        if role(f['para']) != 'figure' or f['image'] not in entries:
            err += 1
    for t in ch.get('tables', []):                 # tables
        if role(t['para']) != 'table' or not t['rows']:
            err += 1
        for row in t['rows']:
            for cell in row:
                for s in cell['sentences']:
                    check(s)
for n in notes.values():                           # note bodies
    for para in n['paragraphs']:
        for s in para:
            check(s)

cov_t = tgt_aligned / tgt_words if tgt_words else 1.0
cov_s = src_covered / src_total if src_total else 1.0
print(f"chapters {len(man['chapters'])} sentences {total} notes {len(notes)} "
      f"empty {empty} errors {err}")
print(f"coverage: {cov_t:.0%} target words aligned, {cov_s:.0%} source words highlighted")
# NOTE: empty translations are LEGAL (§5.3.1) — reported, but not an error.
print("OK" if err == 0 else "PROBLEM",
      f"({empty} untranslated)" if empty else "",
      "" if cov_t >= 0.55 else "(LOW COVERAGE — alignment may be drifted/empty)")
```

### 10.4 Alignment coverage and correctness (informative)

Structural validity (§10.1) and alignment *quality* are independent — and there
are **two distinct** quality questions, only one of which any automatic check can
answer.

**Coverage** (mechanically checkable) — is each word *mapped to something*?

- **target coverage** — fraction of rendered translation words whose chunk
  highlights ≥1 source word;
- **source coverage** — fraction of source words highlighted by some target word.

A producer **SHOULD** report these and **SHOULD** warn below a threshold (the
reference producer warns under **55%** target coverage; a healthy literary
alignment lands ~85–95%). Coverage near zero is the signature of a *gross* failure
— alignment empty or absent — and means regenerate.

**Correctness** (NOT mechanically checkable) — is each word mapped to the *right*
thing? This is the failure that actually bites and that coverage is **blind to**.
A **partial** positional drift (target word *i* ↔ source word *i* through part of
a sentence) maps every word to a *real* source word, so **coverage stays ~100%**
while taps are wrong (e.g. tapping "Dursley" highlights "always"). Likewise a
fluent mistranslation has full, valid structure. Neither offset-errors, coverage,
nor any structural metric detects these. The **only** reliable detector is a
**semantic verification pass**: an independent LLM judge that reads `src`, `text`,
and the `align` mapping per sentence and flags wrong word-mappings and
mistranslations, after which the flagged sentences are regenerated (ideally by a
stronger model) and re-judged. A `.tbook` should not be considered correct on the
strength of structural validation + coverage alone.

The reference producer implements this as `--judge`: every translated sentence
is judged (verdicts cached), flagged sources are written to `<out>.flagged.json`
in the `--invalidate` format, and `--judge-invalidate` clears their cache
entries so the very next run regenerates exactly those. The judge model
defaults to the translate model (`--judge-model` overrides). Counter-intuitive
but measured: a *stronger* judge keeps full drift recall yet flags half the
book on mapping pedantry — including most of its **own** escalated output — so
it condemns the escalation loop to mass fallback; the calibrated same-model
judge plus the static gate is the economical standing configuration, with a
stronger judge reserved for one-off audits. Judge-prompt changes **MUST** be
re-calibrated against synthetic-drift twins (recall) before adoption. With
`--escalate-model <stronger>`, flagged sentences are redone immediately by the
stronger model inside the same cache namespace — the recommended cheap-model
loop.

Escalated output **MUST be re-verified** — a stronger model is not a verified
model (observed in practice: an escalation hallucinating a sentence's meaning
around a source typo, shipping with perfect coverage). The reference producer
re-runs the same gates (lexcheck + judge) over the escalated sentences, gives
whatever is still flagged one more escalation attempt, and strips persistent
failures rather than shipping them: a judged *mistranslation* loses its
translation entirely (re-translated on the next run), an alignment failure
keeps the raw translation with an empty alignment (no highlights). Persistent
failures are listed in `<out>.unverified.json`.

A free **static pre-gate** complements the judge: `--lexcheck` scores each
align pair against a bilingual dictionary (surface-form, multi-sense — the
producer ships a fetcher for every pair of de/en/es/fr/it/ru). Because words
are polysemous and dictionaries incomplete, single pairs are never judged in
isolation; a sentence is flagged only on aggregate low support or on the
*shift signature* — fragments implausible for their own source word but
plausible for its neighbor, the fingerprint of positional drift. Measured by
synthetic drift injection on a real book: ~87% recall of full cascades at
~2% false positives. Lexcheck costs nothing per run and its flags feed the
same escalation path; the LLM judge remains the only detector for
mistranslations and sense-level errors.

---

## 11. Reference implementations

The reference **producer** is [`tbook_converter`](https://github.com/adubovskoy/tbook_converter) (Go); the reference **consumer** is [`tbook_desktop`](https://github.com/adubovskoy/tbook_desktop) (Tauri: Rust core + Svelte WebView). Paths below are relative to each repo's root.

| Role | Location | Notes |
|------|----------|-------|
| Producer (CLI / parse / assemble) | `tbook_converter` → `cmd/convert`, `internal/{epub,segment,tbook}` | Go CLI: EPUB → `.tbook`; roles, emphasis, figures, tables, footnotes; deterministic fragment location + ZIP assembly. |
| Producer (translate / align / verify) | `tbook_converter` → `internal/{translate,align,cache}` | OpenRouter client + word-level prompt + match-by-text alignment + resumable cache; glossary + judge passes. |
| Producer (provenance `meta`) | `tbook_converter` → `internal/tbook/meta.go`, `internal/buildinfo`, `cmd/convert/meta.go` | Run record of [§3.4](#34-meta--provenance-and-service-metadata): producer version/commit, provider, per-pass models and prompt versions, run options; carries an existing `meta` (unknown keys included) across a re-assembly. |
| Consumer (data model) | `tbook_desktop` → `src-tauri/src/models.rs` | Rust `serde` structs that (de)serialize every object here. |
| Consumer (reader) | `tbook_desktop` → `src-tauri/src/tbook.rs` | Random-access ZIP reader; reads entries by name. |
| Consumer (render) | `tbook_desktop` → `src/components/Paragraph.svelte` (+ `src/lib/render.ts`) | Paragraph roles + inline emphasis ([§4.1](#41-paragraphstyles--per-paragraph-roles), [§5.5](#55-spans--inline-emphasis)). |
| Consumer (highlight) | `tbook_desktop` → `src/lib/align.ts`, `src/components/TranslationSheet.svelte` | `highlightedHTML()` — the tap algorithm of [§7](#7-tap-to-translate-algorithm). |
| Consumer (web demo) | `tbook_web/src/lib/demoData.js` | Hand-authored, spec-faithful sample for the marketing site's interactive demo. |

---

## Appendix A — Complete minimal example

`manifest.json`:

```json
{
  "formatVersion": 1,
  "title": "Demo",
  "author": "A. Author",
  "sourceLang": "en",
  "targetLangs": ["ru"],
  "cover": null,
  "chapters": [
    { "id": "ch1", "title": "Chapter 1", "file": "chapters/ch1.json" }
  ]
}
```

`chapters/ch1.json` — a subtitle, a scene break (empty paragraph), then a body
paragraph with one italic word:

```json
{
  "paragraphs": [
    [
      { "src": "A quiet evening.", "words": [[0,1],[2,7],[8,15]], "tr": { "ru": { "text": "Тихий вечер.", "align": [] } } }
    ],
    [],
    [
      {
        "src": "Stan went to the living room.",
        "words": [[0,4],[5,9],[10,12],[13,16],[17,23],[24,28]],
        "tr": {
          "ru": {
            "text": "Стэн прошёл в гостиную.",
            "align": [
              { "t": [0,4],   "w": [0] },
              { "t": [5,11],  "w": [1] },
              { "t": [12,13], "w": [2] },
              { "t": [14,22], "w": [4,5] }
            ]
          }
        },
        "spans": [ { "s": 17, "e": 23, "k": "i" } ]
      }
    ]
  ],
  "paragraphStyles": ["subtitle", "sceneBreak", "body"]
}
```

Tapping **"living"** (word index 4) or **"room"** (index 5) highlights
«гостиную» (`text[14,22)`); tapping **"Stan"** (index 0) highlights «Стэн»
(`text[0,4)`). Word index 3 (`"the"`) appears in no chunk's `w`, so tapping it
highlights nothing — correct, since Russian has no article. The first paragraph
renders as a centered **subtitle**, the second as a **scene break**, and
"living" renders **italic** (`spans[0]` covers `src[17,23)`) — independently of
the alignment.

---

## Appendix B — Figures, tables, and footnotes example

A chapter with a figure (caption is a normal tappable paragraph), a table, and
a footnote marker; plus the book-level `notes.json`. `manifest.json` adds
`"notes": "notes.json"`, and the archive carries `images/img1.jpg`.

`chapters/ch2.json`:

```json
{
  "paragraphs": [
    [ { "src": "The results were surprising.", "words": [[0,3],[4,11],[12,16],[17,27]],
        "tr": { "ru": { "text": "Результаты были удивительными.", "align": [], "q": 0 } },
        "notes": [ { "p": 28, "id": "n1", "label": "1" } ] } ],
    [ { "src": "FIGURE 1. Small habits compound over time.", "words": [[0,6],[10,15],[16,22],[23,31],[32,36],[37,41]],
        "tr": { "ru": { "text": "", "align": [] } } } ],
    []
  ],
  "paragraphStyles": ["body", "figure", "table"],
  "figures": [ { "para": 1, "image": "images/img1.jpg" } ],
  "tables": [ {
    "para": 2,
    "rows": [
      [ { "sentences": [ { "src": "Good habits", "words": [[0,4],[5,11]], "tr": { "ru": { "text": "", "align": [] } } } ], "header": true },
        { "sentences": [ { "src": "Bad habits",  "words": [[0,3],[4,10]], "tr": { "ru": { "text": "", "align": [] } } } ], "header": true } ],
      [ { "sentences": [ { "src": "Reading daily.", "words": [[0,7],[8,13]], "tr": { "ru": { "text": "", "align": [] } } } ] },
        { "sentences": [ { "src": "Smoking.", "words": [[0,7]], "tr": { "ru": { "text": "", "align": [] } } } ] } ]
    ]
  } ]
}
```

`notes.json`:

```json
{
  "n1": {
    "label": "1",
    "kind": "note",
    "paragraphs": [ [
      { "src": "You may wonder about luck. Luck matters too.",
        "words": [[0,3],[4,7],[8,14],[15,20],[21,25],[27,31],[32,39],[40,43]],
        "tr": { "ru": { "text": "", "align": [] } } }
    ] ]
  }
}
```

The marker renders as a superscript **¹** after "surprising." (insertion point
28 = end of `src`); tapping it opens note `n1`, whose body is ordinary tappable
sentences. The figure paragraph renders `images/img1.jpg` with its caption
below; a consumer that predates figures renders just the caption text. Empty
`"text"` values mean "not (yet) translated" and are legal per
[§5.3.1](#531-empty-and-missing-translations).

---

## License

This specification is licensed under the
[Creative Commons Attribution 4.0 International License (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/).
You are free to share and adapt it for any purpose, including commercially,
provided you give appropriate attribution.
