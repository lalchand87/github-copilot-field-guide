# Study Library — Rebuild Specification

A complete, implementation-independent description of the system, sufficient to
rebuild it from scratch on a clean machine. Written as a spec, not a tutorial:
it states *what must be true*, and explains *why* wherever a decision is not
obvious.

**Last verified:** 2026-08-27, against the running system.

---

## 1. What this is

A personal reading system for long technical documents. Two halves:

| Half | What it does |
|---|---|
| **The reader** | Static HTML pages sharing one CSS file and one JS file. Focus mode, themes, read-aloud, progress memory. |
| **The pipeline** | Node scripts that turn Markdown, DOCX or EPUB into those pages. |

### Design constraints — these drive everything else

| # | Constraint | Consequence |
|---|---|---|
| 1 | **Published pages are zero-dependency static HTML** | No framework, no bundler, no build step to *read* a page |
| 2 | **Must work from `file://`** | No `fetch()`, no ES modules, no external font URLs |
| 3 | **ES5 syntax in `study.js`** | Opens in any browser, with no transpiler |
| 4 | **Node is build-time only** | The reader never needs Node; only conversion and importing do |
| 5 | **Everything stays hand-editable** | `topics.js` is a plain array; pages are readable HTML |

Constraint 2 shapes the most code. A page opened by double-clicking is an
*opaque origin* in Chrome: it cannot fetch its own sibling files, and
`@font-face` obeys CORS. That is why fonts are inlined as base64 `data:` URIs,
and why the topic list is a `<script>` assigning a global rather than a JSON
file that gets fetched.

---

## 2. Prerequisites

| Requirement | Version | Needed for |
|---|---|---|
| **Node.js** | 18 or newer (built on v24) | build scripts, importer |
| npm | ships with Node | three importer packages |
| A browser | any modern one | reading |
| Microsoft Edge | optional | by far the best read-aloud voices on Windows |

No global npm packages. No Python. No build toolchain.

---

## 3. Directory layout

```text
study/
├── fonts.css                 generated — base64 @font-face, ~251 KB
├── study.css                 the shared stylesheet
├── study.js                  the shared engine (ES5, IIFE-wrapped)
├── topics.js                 the topic list — hand-editable
├── template.html             skeleton for a hand-written page
├── start-study.bat           Windows launcher
├── README.md                 human notes
├── SPEC.md                   this file
│
├── <topic>.html              one file per topic, all siblings
│
├── assets/
│   └── <topic-slug>/         images extracted from imported EPUBs
│
├── build/
│   ├── md2study.js           Markdown/blocks -> study page  (the core)
│   ├── fonts.js              downloads fonts, writes fonts.css
│   ├── <topic>.json          one build config per generated page
│   ├── assemble-*.js         concatenate multi-file sources
│   ├── mkpat.js / mkcfg.js   config generators
│   └── sources/              the Markdown sources
│
└── importer/
    ├── package.json          three dependencies
    ├── server.js             plain node:http, port 8901
    ├── import.js             orchestration
    ├── docx.js               DOCX  -> blocks
    ├── epub.js               EPUB  -> blocks
    ├── html2blocks.js        HTML  -> blocks
    ├── library.js            safe read/write of ../topics.js
    └── public/
        ├── index.html        drop zone
        └── library.html      drag-and-drop topic organiser
```

---

## 4. The page contract

Every topic page is a standalone HTML file satisfying exactly this:

```html
<!DOCTYPE html>
<html lang="en"><head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>...</title>
<link rel="stylesheet" href="fonts.css">
<link rel="stylesheet" href="study.css">
</head>
<body data-topic="UNIQUE-ID" data-toc-title="Interview Study">
  <div class="shell">
    <main id="top">
      <div class="col">
        ... content ...
      </div>
    </main>
  </div>
  <script src="topics.js"></script>
  <script src="study.js"></script>
</body></html>
```

| Requirement | Why |
|---|---|
| `data-topic` unique across all pages | keys the saved reading position |
| `topics.js` loaded **before** `study.js` | the rail is built at load time from `window.STUDY_TOPICS` |
| Content nested `.shell > main > .col` | the CSS grid and the dim veil both target this |
| No inline `<style>` or `<script>` in the body | everything shared lives in the two shared files |

**Everything else — top bar, topics rail, contents list, control panel, highlight
layer — is constructed by `study.js` at load time.** A page author never writes
any of it.

### Semantic content classes

| Class | Meaning |
|---|---|
| `.masthead` | page header: kicker, title, standfirst, byline |
| `.part-open` | Part divider: `.part-tag`, `.part-title`, `.part-intro` |
| `section.sec` | one numbered section: `.sec-head` > `.sec-num` + `.sec-title` |
| `.lede` | section summary line |
| `.scene` | narrative example, with `.lab` and `.beat` |
| `.model` | mental model: `.mm` claim + `.why` reason |
| `.ask` | interview question: `.lab` + `.q` + `.gen` |
| `.carry` | the line to remember: `.bb` |
| `.note` | warning / danger box |
| `.reveal` | fade-in on scroll (see §5.6) |

---

## 5. The reader — `study.js`

One IIFE, ES5, no dependencies. Internally organised in six numbered sections.

### 5.1 Chrome construction

Builds and injects the top bar (progress + location), topics rail, contents
list, highlight layer, zen hint and control panel. The rail groups topics by
their `group` field in array order; group collapse state lives in
`localStorage['study-rail-shut']`.

The control panel is an **accordion**: four constantly-used controls sit in an
always-visible quick strip (focus toggle, prev, next, play), with four
collapsible groups below — `read`, `look`, `voice`, `keys`. The open group is
persisted as `state.grp`.

### 5.2 Progress and location

The top bar shows a progress bar and the current section title, derived from
scroll position against the collected block list.

### 5.3 Focus reading — the highlighter

The most delicate part of the system.

```text
collect()   walk `main p,li,tr,caption,h1,h2,h3,h4,pre,.beat`
            skip anything inside <figure> and anything with no text
            split each block into visual ROWS via rowsOf()
            -> a flat array of {el, li, b} line units

rowsOf(el)  TR:    one rect from getBoundingClientRect()
            other: Range.selectNodeContents(el).getClientRects(),
                   filtered to width > 2 && height > 2,
                   grouped by top with tolerance
                   Math.max(5, rect.height * 0.45)

paint()     READ EVERYTHING FIRST, then write.
            Measure band rects, the .col box and the spoken-word rect;
            only then touch #hlLayer.
```

**Two invariants that must not be broken:**

| Invariant | Why |
|---|---|
| Geometry is measured at paint time, never cached across paints | fonts load late, themes change metrics, the window resizes |
| All reads happen before any write to `#hlLayer` | writing then reading forces a full re-layout per keypress — measured at 24.2 ms, reduced to 10.7 ms purely by ordering |

The painted band uses deliberate insets:

```text
top    = rect.top  - 2
left   = rect.left - 4
width  = rect.w    + 8
height = rect.h    + 4
```

Any audit comparing painted geometry against text geometry **must** account for
these, or every band appears 2 px misaligned.

The highlight uses `mix-blend-mode`. **No ancestor of `#hlLayer` may create a
stacking context** — no `transform`, `filter`, `opacity < 1` or `will-change` —
or the blend silently stops working.

### 5.4 Controls

Themes, band colour, band alpha, text strength, font size, step size, line/para
mode, rail and contents toggles, zen mode.

**Text strength** is a contrast control, added because the default ink was too
sharp at night. Base tokens are stored as `--ink0 / --soft0 / --muted0 /
--pre-ink0`, and the live values derive from them:

```css
--ink: color-mix(in srgb, var(--ink0) var(--tw), var(--paper));
```

`--tw` is clamped 45–100 %. Font size `--fs` is clamped 80–170 in steps of 5.

### 5.5 Read aloud

Web Speech API, with these behaviours required:

| Requirement | Reason |
|---|---|
| `getVoices()` may return empty on the first call | voices arrive asynchronously via `voiceschanged` |
| **Never overwrite the saved voice with a default** | otherwise the choice resets on every reload |
| Fall back to the browser default if the saved voice is absent | the same page opens in Edge and Chrome, which have different voice lists |
| Never call `speak()` synchronously inside an `end` handler | Chrome silently drops it |
| A 1–2 s stall watchdog | Chrome sometimes drops an utterance without ever firing `end` |
| Word highlighting via `boundary` events | supplies `charIndex` / `charLength` |

`textFor(el)` returns `{text, base, synthetic}`. `synthetic` is true for
generated marker text — for example the announcement replacing a skipped code
block — and in that case character offsets must **not** be mapped back into the
element's real characters. Whitespace is deliberately **not** normalised; `base`
carries the leading whitespace so offsets stay aligned.

Play starts from what is on screen, not from wherever the band was last left.
Clicking any line sets the position, and this works whether or not focus mode is
on; a click made during a text selection is ignored.

### 5.6 Reveal animation

`.reveal` fades in via `IntersectionObserver`. **Focus mode clears the transform
entirely:**

```css
body.focus-on .reveal { opacity:1 !important; transform:none !important; }
```

Any geometry measurement must therefore be taken with focus mode **on**, or
unrevealed blocks sit 16 px low.

---

## 6. State and persistence

```js
state = { on:false, color:1, theme:'paper', step:2, stepPara:1, idx:0,
          alpha:35, mode:'lines', zen:false, dim:false, rail:true, toc:true,
          tw:100, voice:null, rate:100, skipCode:true, skipTables:false,
          stopAtSection:false, grp:'', fs:100 }
```

| localStorage key | Holds |
|---|---|
| `study-prefs` | everything above **except** `idx` — shared across all topics |
| `study-pos-<topic>` | reading position (`idx`), **per topic** |
| `study-rail-shut` | collapsed rail groups |

Preferences are global on purpose; position is per-topic on purpose.

---

## 7. Keyboard map

| Key | Action |
|---|---|
| `f` | focus mode on/off |
| `z` | zen mode |
| `d` | dim (veil the rest of the page) |
| `s` / `S` | speak play-pause / stop |
| `b` | topics rail |
| `c` | contents |
| `Esc` | stop speech → exit zen → exit focus, in that order |
| `↑ ↓` `Space` `PgUp` `PgDn` | move the band — **or skip blocks while speaking** |
| `1`–`5` | band colour |
| `t` | cycle theme: paper → sepia → dark |
| `p` | line mode / paragraph mode |
| `[` `]` | band step size |
| `{` `}` | font size |
| `-` `=` | text strength |
| `,` `.` | band alpha |

---

## 8. Theming

Three themes on `html[data-theme]`: `paper` (default), `sepia`, `dark`.

Token groups: `--paper --ink0 --soft0 --muted0 --pre-ink0 --accent --rule
--rule-strong --surface --surface2 --surface3 --card --codebg --codebg2 --toc
--topbar --dim --hl-rgb --hl-alpha --hl-blend --svg-*`, plus the layout tokens
`--rail --maxread --fs --tw`.

The rail widens only where there is slack:

```css
:root { --rail: 210px }
@media (min-width:1440px){ :root{--rail:270px} .shell{max-width:1500px} }
```

**Zen mode must override the collapse rules explicitly.** `body.no-rail .shell`
and `body.no-toc .shell` appear later in the file and would otherwise win,
leaving `main` stuck in a 210 px sidebar track:

```css
body.zen .shell,
body.zen.no-rail .shell,
body.zen.no-toc .shell,
body.zen.no-rail.no-toc .shell { grid-template-columns:minmax(0,1fr); max-width:920px }
```

---

## 9. Fonts

Two families, eight faces, ~251 KB, all inlined into `fonts.css`.

| Family | Use | Weights |
|---|---|---|
| **Newsreader** | body text | 400, 500, 600 — variable, one file serves all three |
| **IBM Plex Mono** | code, labels, small caps | 400 |

Rebuild with `node build/fonts.js`. It downloads the woff2 files, **groups by
URL** — Newsreader is a variable font served identically for several weights, so
naive per-weight embedding wastes about 250 KB — and writes base64 `data:` URIs.

Always declare real fallbacks: `"Newsreader",Georgia,"Times New Roman",serif`
and `"IBM Plex Mono",monospace`.

---

## 10. The pipeline — `build/md2study.js`

The core converter.

```js
module.exports = { build, buildFromBlocks, parse, inline, renderSimple, slug };
```

| Function | Role |
|---|---|
| `parse(md)` | Markdown → blocks (the Markdown front-end) |
| `build(cfg)` | thin wrapper: read file, parse, hand off |
| **`buildFromBlocks(blocks, cfg)`** | **everything downstream** — the shared core |

Splitting at `buildFromBlocks` is what lets Markdown, DOCX and EPUB share all
the section logic, id de-duplication and rendering, so every format inherits the
same output and the same fixes.

### 10.1 The block model

```js
{t:'h',       lvl, text}
{t:'p',       text, html?}
{t:'list',    ordered, items, html?}
{t:'table',   head, body, html?}
{t:'code',    text}
{t:'quote',   text}
{t:'callout', kind, text}
{t:'fig',     src, alt}
{t:'hr'}
```

`html?` means the block carries pre-rendered inline HTML; a `txt(s, raw)` helper
honours it.

### 10.2 Config schema

```json
{
  "src":        "sources/x.md",
  "out":        "../x.html",
  "topic":      "x",
  "tocTitle":   "Interview Study",
  "title":      "text for <title>",
  "kicker":     "small line above the h1",
  "h1":         "page title",
  "standfirst": "one paragraph under the title",
  "byline":     ["short", "facts", "in", "a", "row"],
  "abstract":   "**How to read this.** ...",

  "rules": [
    { "lvl": 1, "match": "^Part (\\d+) — (.*)$",
      "as": "part",    "tag": "Part $1", "title": "$2" },
    { "lvl": 2, "match": "^(\\d+) — (.*)$",
      "as": "section", "num": "$1",      "title": "$2" }
  ],

  "auto":        { "partLvl": 1, "secLvl": 2 },
  "skip":        [],
  "dropHeading": [],
  "injectParts": [],
  "keepUpper":   []
}
```

`rules` map headings to parts and sections by regex against the heading text.
`auto` is the fallback for imported documents with no naming convention — it maps
by heading **level** instead. `partLvl: -1` means "no parts at all".

Run it:

```bash
node build/md2study.js build/<topic>.json
```

Relative paths inside a config resolve against the config's own folder.

### 10.3 Heading conventions in Markdown sources

| Source | Becomes |
|---|---|
| `### 🎯 Interview Question` followed by a blockquote | `.ask` |
| `### 🎓 Staff-Level Thinking` | `.carry` |
| `> [!danger]` / `> [!warning]` | `.note` |
| a blockquote immediately after a Part heading | `.part-intro` |
| any other `###` | `h4`, with the emoji lifted into `.hmark` |

### 10.4 Rules the converter must enforce

| Rule | Failure it prevents |
|---|---|
| A heading only opens a section if it **outranks** the open one — guard with `!sec \|\| b.lvl <= secLvl` | a nested `##` inside a chapter being promoted to a top-level section, which drags Part dividers into the middle of a chapter |
| List rendering keeps the parent `<li>` open while descending, closing `</ul></li>` on the way back up | invalid `</li><ul>` nesting |
| Section ids de-duplicated through a `usedIds` set with a `-secN` suffix | two chapters sharing a title producing duplicate DOM ids |

---

## 11. The importer

`start-study.bat` launches `importer/server.js` on **port 8901**
(`process.env.PORT` overrides).

| Route | Purpose |
|---|---|
| `/` | drop page |
| `/library` | drag-and-drop topic organiser |
| `POST /api/import` | convert an uploaded file, write the page, register the topic |
| `/api/topics` | read and write the topic list |
| everything else | served statically from the study folder |

Plain `node:http` with hand-rolled multipart parsing — no Express.

Serving the pages over `http://localhost` is a free side benefit: it removes
Safari's `file://` localStorage restriction.

### 11.1 Dependencies

```json
{ "mammoth": "^1.8.0", "jszip": "^3.10.1", "node-html-parser": "^6.1.13" }
```

All pure JS, all build-time only. Published pages stay dependency-free.

### 11.2 DOCX

`mammoth` with a style map converting `Heading1`–`Heading4`, `ListBullet`,
`ListNumber` and tables to semantic HTML, then HTML → blocks. Levels adapt: a
document with no `Heading1` uses `Heading2` as its section level and has no
parts.

### 11.3 EPUB — two-tier, and this matters

Read `META-INF/container.xml` → `content.opf` → spine order. Then count real
`<h1>`–`<h6>` tags across the whole book and choose a tier:

| Tier | When | Behaviour |
|---|---|---|
| **semantic** | the book has real heading tags | headings drive parts and sections normally |
| **flat** | zero or near-zero heading tags — Calibre conversions use `<p class="h">` | one section per spine document; force `{partLvl:-1, secLvl:2}`; class-detected headings drop to level 4 |

Without the flat tier every spine document becomes a Part, so "Copyright" and
"Dedication" show up as empty parts. In flat mode, titles come from `toc.ncx`
when it has more than a couple of entries, else from the first short paragraph
whose class is rare within that document, else from the filename.

Images extract to `assets/<topic-slug>/` and are referenced relatively. Unlike
fonts, `<img>` is not CORS-restricted, so a folder is safe from `file://`.

### 11.4 Naming

Uploads land in `os.tmpdir()`, so `path.basename()` yields a temp name such as
`study-import-1787494169591`. A `displayName` / `opts.name` parameter must be
threaded through `docx.read` and `epub.read`, or that leaks into the page title.

### 11.5 Writing topics.js

`library.js` reads `../topics.js` in a **vm sandbox** — not `eval`, not
`require` — mutates the array, and writes it back **formatted and commented**,
leaving a `.bak`. The file must remain hand-editable; that is the whole point.

---

## 12. topics.js contract

```js
window.STUDY_TOPICS = [
  { id:'iam', label:'IAM', note:'Identity & Access',
    file:'iam.html', group:'Domain' },
  ...
];
```

| Field | Rule |
|---|---|
| `id` | must equal that page's `<body data-topic>` |
| `label` | rail text |
| `note` | small grey subtitle |
| `file` | path relative to the study folder |
| `group` | rail heading |
| `soon` | optional; renders greyed out and unclickable |

**Order matters.** Groups appear in the order they first occur; topics appear in
the order listed.

---

## 13. Rebuild procedure

```bash
# 1. create the tree
mkdir -p study/build/sources study/importer/public study/assets

# 2. author the four shared files
#    study.css, study.js, topics.js, template.html

# 3. generate the fonts (needs internet, once)
node build/fonts.js            # -> fonts.css, 8 faces, ~251 KB

# 4. install the importer
cd importer && npm install --no-audit --no-fund && cd ..

# 5. build a page from Markdown
node build/md2study.js build/<topic>.json

# 6. run it
start-study.bat                # or:  cd importer && npm start
# opens http://localhost:8901/
```

Pages also open by double-clicking. The server is needed only for importing, and
for Safari.

---

## 14. Verification

A rebuild is correct when all of the following pass.

### 14.1 Structural audit — run on every page

| Check | Required |
|---|---|
| duplicate `id` attributes | 0 |
| broken internal `href="#..."` anchors | 0 |
| `</li><ul>` or `</li><ol>` nesting | 0 |
| balanced tags: `section table ul ol li pre h2 h3 h4 div p tr blockquote` | all balanced |
| tables with ragged column counts | 0 |
| unconverted `**bold**` or raw pipe-table rows outside `<pre>` | 0 |

**Strip `<pre>` contents before scanning for raw Markdown**, or ASCII diagrams
containing `|` produce false positives.

### 14.2 Highlighter audit — in a browser, on the longest page

Serve over `http://localhost:8901`. Enable focus mode **first**, then measure.

| Check | Required |
|---|---|
| backward steps in reading order | 0 |
| zero-sized or oversized (> 200 px) bands | 0 |
| painted band vs computed rect, ≥ 100 sampled lines | < 1 px, after allowing for the §5.3 insets |
| forward navigation, 600 consecutive steps | 0 stalled, 0 backward, 0 unpainted |

Clear `study-pos-<topic>` afterwards — the audit moves the saved position.

**Two false-failure modes, both previously hit:**

1. Measuring **before** enabling focus mode → every `.reveal` sits 16 px low, so
   the audit reports hundreds of phantom misalignments.
2. Comparing against the raw text rect instead of the inset band rect → a
   constant 2 px "misalignment" on every single line.

In both cases the page was correct and the audit was wrong. Check the source
before believing a failure.

### 14.3 Refactor safety

After changing `md2study.js`, regenerate an existing page and confirm it is
**byte-identical** to the previous version. That is the only reliable proof that
a refactor changed nothing.

---

## 15. Known environment traps

Recorded because each one cost real time.

| Trap | Detail |
|---|---|
| **Heredoc backslash collapse** | Writing files through shell heredocs collapses `\` to `\`. A JSON config needing `"\d+"` ends up as `"\d+"` and fails to parse. Generate configs with a Node script using `String.fromCharCode(92)`, or write them with a file tool. Single backslashes — ASCII art — survive fine. |
| **Long heredocs fail** | Past roughly 90–100 lines a heredoc dies with `unexpected EOF`. Write long files in chunks and append. |
| **`/tmp` on Windows** | Node resolves `/tmp` to `F:\tmp` when the cwd is on `F:`. Use absolute scratch paths. |
| **Robocopy exit code 1** | Means "files copied successfully", not failure. |
| **Java `%` on negatives** | `-3 % 5` is `-2`. Prefix-remainder code must normalise with `((x % k) + k) % k`. |
| **`java.util.Stack`** | Extends `Vector`, so every operation is synchronised. Use `ArrayDeque`. |

---

## 16. Content conventions

Not enforced by code, but what the existing pages follow.

Every chapter opens with the question the topic exists to answer, explains the
idea plainly, shows the mechanics on concrete numbers, then names the mistakes
that are actually made and the remark that reads as senior.

| Deck | Shape |
|---|---|
| **DSA Notes** | 36 chapters across 2 parts; each chapter 7–12 sub-topics with worked trace tables. Chapters 18 and 36 are deliberately short revision sheets. |
| **DSA Patterns** | Two Pointers uses a deep 27-section master-guide format; the other nine use an 8-section card — Recognize, Example, How it works, Visual, Code, Common problems, Complexity, Tips — plus 15 problems at 5 easy, 5 medium, 5 hard. |
| **Coding Muscle** | 11 parts, 107 sections. Deliberately withholds finished solutions, because the source programme mandates no AI, no Google and no copied solution before your own attempt. |

Code is Java throughout. Diagrams are plain ASCII inside `text` fenced blocks.
Prose uses British spelling.

---

# Part II — Functional Requirements

Everything above describes *structure*. This part describes *behaviour*: what the
system must do, stated so each item can be tested. `MUST` is mandatory; `SHOULD`
is a strong default that may be traded away with a reason.

---

## 17. Screen layout

Three columns plus a fixed top bar.

```text
+--------------------------------------------------------------------+
|  top bar:  progress ......  Part II / 12. Binary Search      12/36  |
+------------+--------------------------------------+----------------+
|            |                                      |                |
|  TOPICS    |            READING COLUMN            |   CONTENTS     |
|  rail      |            (.shell > main > .col)    |   of this page |
|            |                                      |                |
|  grouped   |   max-width --maxread                |   sections,    |
|  folders   |   centred                            |   scroll-spy   |
|            |                                      |                |
+------------+--------------------------------------+----------------+
                                          [ control panel, bottom-right ]
```

| Requirement | |
|---|---|
| **R17.1** | The layout MUST be a CSS grid of `--rail` / `1fr` / `--toc`. |
| **R17.2** | The reading column MUST be capped at `--maxread` and centred, regardless of window width. |
| **R17.3** | Either side column MUST be collapsible independently, and the reading column MUST expand to fill the freed space. |
| **R17.4** | At viewport widths below 1100 px both side columns SHOULD hide automatically. |
| **R17.5** | The rail width MUST widen to 270 px only at ≥ 1440 px, where there is slack. |

### Top bar

| Requirement | |
|---|---|
| **R17.6** | MUST show a progress bar reflecting position through the document. |
| **R17.7** | MUST show the current Part and Section title, updating on scroll. |
| **R17.8** | MUST show position as `n / total` sections. |
| **R17.9** | MUST be hidden in zen mode. |

### Topics rail

| Requirement | |
|---|---|
| **R17.10** | MUST be built at load time from `window.STUDY_TOPICS`; the page MUST NOT hard-code it. |
| **R17.11** | MUST group topics by `group`, in first-occurrence order. |
| **R17.12** | Each group MUST be collapsible, and the collapsed set MUST persist across sessions and across pages. |
| **R17.13** | The current topic MUST be visually marked. |
| **R17.14** | A topic with `soon: true` MUST render greyed and MUST NOT be clickable. |
| **R17.15** | MUST end with a link to the library organiser. |

### Contents

| Requirement | |
|---|---|
| **R17.16** | MUST be generated from the page's own `section.sec` elements — never hand-authored. |
| **R17.17** | MUST highlight the section currently in view (scroll-spy). |
| **R17.18** | Clicking an entry MUST scroll to that section. |

---

## 18. Reading modes

Four independent toggles that compose freely.

| Mode | Key | What it does |
|---|---|---|
| **Focus** | `f` | Paints a highlight band over the current line or paragraph. Arrow keys move it. |
| **Dim** | `d` | Veils everything above and below the band, within the text column only. |
| **Zen** | `z` | Hides top bar and both side columns; the reading column takes the full width. |
| **Line / Paragraph** | `p` | The band covers *n* visual lines, or *n* whole blocks. |

| Requirement | |
|---|---|
| **R18.1** | Focus mode MUST survive reload, per the shared preference store. |
| **R18.2** | The band MUST cover `step` units, adjustable 1–5, with a separate value per mode. |
| **R18.3** | Clicking any text line MUST move the band there — **whether or not focus mode is currently on**. |
| **R18.4** | A click that ends a text selection MUST be ignored, so selecting text does not move the band. |
| **R18.5** | Moving the band past the viewport MUST scroll it into view. |
| **R18.6** | Dim MUST veil only the text column plus a small horizontal pad, never the side columns. |
| **R18.7** | Zen MUST restore the previous rail and contents state on exit. |
| **R18.8** | `Esc` MUST unwind one layer at a time: stop speech, then exit zen, then exit focus. |

---

## 19. The highlight band

| Requirement | |
|---|---|
| **R19.1** | A "line" MUST be a *visual* row, not a DOM element — a wrapped paragraph of four rows is four units. |
| **R19.2** | Table rows, list items, headings, captions and `<pre>` blocks MUST each be addressable units. |
| **R19.3** | Content inside `<figure>` MUST never be banded. |
| **R19.4** | Empty or whitespace-only blocks MUST be skipped. |
| **R19.5** | The band MUST realign after font-size change, theme change, window resize and web-font load. |
| **R19.6** | Band colour MUST offer 5 presets and an adjustable opacity. |
| **R19.7** | The band MUST NOT obscure the text beneath it — hence `mix-blend-mode`. |
| **R19.8** | Reading order MUST be monotonic: advancing MUST never move the band up the page. |

---

## 20. Reading position

| Requirement | |
|---|---|
| **R20.1** | Position MUST be stored **per topic**, so each document remembers its own place. |
| **R20.2** | Preferences — theme, colours, sizes, voice — MUST be stored **globally**, shared by every topic. |
| **R20.3** | Reopening a page MUST restore its position. |
| **R20.4** | Position MUST be an index into the collected unit list, not a pixel offset, so it survives reflow at a different window size. |
| **R20.5** | All persistence MUST be wrapped so that a failure — private browsing, disabled storage — degrades to defaults rather than throwing. |

---

## 21. The control panel

Bottom-right, collapsible. A permanently visible **quick strip**, then four
accordion groups of which at most one is open. The open group persists.

### Quick strip — always visible

`Focus` toggle · `⟨` previous · `⟩` next · `Play`

### Group 1 — Reading

| Control | Range / options |
|---|---|
| Text size | `A−` `A+` `Reset`, 80–170 % in steps of 5 |
| Mode | Zen mode · Focus line |
| Side columns | Topics · Contents |
| Step by | Lines · Paragraph |
| Lines at a time | 1–5 |

### Group 2 — Appearance

| Control | Range / options |
|---|---|
| Colour | 5 band presets |
| Theme | Paper · Sepia · Dark |
| Opacity | 0–100 % |
| Text strength | 45–100 % |

### Group 3 — Read aloud

| Control | Range / options |
|---|---|
| Transport | Play/Pause · Stop · Prev · Next |
| Voice | every voice the browser exposes |
| Speed | 0.6× – 2.0× |
| Skip while reading | Code blocks · Tables |
| Stop reading at | Section end |

### Group 4 — Keyboard

A reference list of every shortcut in §7.

| Requirement | |
|---|---|
| **R21.1** | At most one group MAY be open at a time; opening one MUST close the others. |
| **R21.2** | The open group MUST persist across sessions. |
| **R21.3** | Every control MUST have a keyboard equivalent — the panel is a convenience, never the only route. |
| **R21.4** | In zen mode the panel MUST fade to near-invisible and return on hover. |
| **R21.5** | Clicks inside the panel MUST NOT propagate to the document click handler, or adjusting a control would move the band. |

---

## 22. Read aloud

Built on the Web Speech API, which is inconsistent across browsers. The
requirements exist mostly to absorb that.

### Queue

| Requirement | |
|---|---|
| **R22.1** | The speech queue MUST be the same block list the highlighter collected — one source of truth. |
| **R22.2** | Reading MUST continue block to block automatically, not stop after each paragraph. |
| **R22.3** | Pressing Play MUST start from **what is on screen**, not from wherever the band was last left. |
| **R22.4** | If the band is already visible, Play MUST resume from it; if the reader has scrolled away, Play MUST start from the first readable line near the top of the viewport. |

### What gets spoken

| Requirement | |
|---|---|
| **R22.5** | Code blocks MUST be skippable, and when skipped MUST be announced briefly rather than silently dropped. |
| **R22.6** | Tables MUST be independently skippable. |
| **R22.7** | Announcement text MUST be marked *synthetic*, and character offsets from a synthetic utterance MUST NOT be mapped back into the element's real text. |
| **R22.8** | Whitespace MUST NOT be normalised before speaking; offsets must stay aligned with the DOM text. |
| **R22.9** | An optional "stop at section end" MUST halt reading at the section boundary. |

### Word tracking

| Requirement | |
|---|---|
| **R22.10** | The currently spoken word SHOULD be highlighted more brightly than the band, using `boundary` events. |
| **R22.11** | Where a voice emits no `boundary` events, the system MUST degrade to block-level highlighting rather than failing. |
| **R22.12** | The band MUST follow the voice, scrolling as reading advances. |

### Voice selection

| Requirement | |
|---|---|
| **R22.13** | The chosen voice MUST persist across reloads. |
| **R22.14** | `getVoices()` returning empty MUST NOT overwrite the stored choice — voices arrive asynchronously via `voiceschanged`. |
| **R22.15** | If the stored voice is unavailable in the current browser, the system MUST fall back to the default **without discarding the stored preference**, so the same page works in Edge and Chrome. |
| **R22.16** | Where speech is unsupported entirely, the controls MUST be disabled with an explanation, and the rest of the page MUST work. |

### Robustness

| Requirement | |
|---|---|
| **R22.17** | The system MUST detect an utterance that goes silent without firing `end` and resume from where the words stopped. |
| **R22.18** | `speak()` MUST NOT be called synchronously inside an `end` handler. |
| **R22.19** | Speech MUST be cancelled on page unload. |
| **R22.20** | Changing voice or speed MUST take effect on the next utterance; both are fixed for the life of one utterance. |

---

## 23. Non-functional requirements

| Requirement | |
|---|---|
| **R23.1 Performance** | A band move MUST complete in under ~12 ms on a 400 KB page. Achieved by reading all geometry before writing to the highlight layer; the read-then-write ordering took one measured case from 24.2 ms to 10.7 ms. |
| **R23.2 Page weight** | A generated page SHOULD stay under ~1 MB. Beyond that, split the source. |
| **R23.3 Offline** | Every page MUST render fully with no network, including fonts. |
| **R23.4 Browser support** | Any browser with `IntersectionObserver`, `Range.getClientRects` and CSS custom properties. `study.js` MUST remain ES5 — no arrow functions, `let`, template literals or modules. |
| **R23.5 Graceful degradation** | Missing speech support, missing fonts, or disabled storage MUST each degrade to a working page. |
| **R23.6 No build step to read** | Opening a `.html` file directly MUST work. Node is required only to *generate* pages. |
| **R23.7 Accessibility** | Text MUST remain selectable and copyable with focus mode on; the highlight is decorative and MUST NOT intercept pointer events. |

---

## 24. Error and empty states

| Situation | Required behaviour |
|---|---|
| `topics.js` missing or malformed | Page still reads; the rail is empty or absent. Never a blank page. |
| A topic's `file` does not exist | The rail entry still renders; the browser reports the 404 on click. |
| `data-topic` missing | Fall back to a default key, so position still saves — just not per topic. |
| No `section.sec` elements | Contents column hides; progress reports a single unit. |
| Speech unsupported | Voice controls disabled with a short explanatory hint. |
| Stored voice unavailable | Silent fallback to the browser default; the stored name is kept. |
| `localStorage` unavailable | All defaults; no crash. |
| Empty page body | Focus mode is a no-op rather than an exception. |

---

## 25. Pipeline requirements

| Requirement | |
|---|---|
| **R25.1** | One converter core MUST serve every input format. Markdown, DOCX and EPUB MUST all reduce to the same block array and share everything downstream, so a fix in one benefits all. |
| **R25.2** | Conversion MUST be deterministic — the same input and config MUST produce a byte-identical page. |
| **R25.3** | Section ids MUST be unique within a page, de-duplicated by suffix when titles collide. |
| **R25.4** | Generated HTML MUST be valid: correct list nesting, balanced tags, no stray Markdown. |
| **R25.5** | A heading MUST only open a new section if it outranks the currently open one, so nested headings do not get promoted. |
| **R25.6** | Configs MUST be plain JSON, hand-editable, with no code in them. |
| **R25.7** | The converter MUST report a one-line summary — parts, sections, tables, code blocks, size — so a bad build is obvious immediately. |

## 26. Importer requirements

| Requirement | |
|---|---|
| **R26.1** | Dropping a `.docx` or `.epub` MUST produce a finished topic page and register it, with no hand-editing. |
| **R26.2** | Structure MUST be inferred from the document, not guessed from filenames. |
| **R26.3** | EPUBs without real heading tags MUST still produce sensible chapters — Calibre conversions use styled paragraphs, and treating each spine file as a Part yields empty "Copyright" and "Dedication" parts. |
| **R26.4** | The page title MUST come from document metadata or the uploaded filename — never from the server's temp filename. |
| **R26.5** | Images MUST extract to `assets/<topic>/` and be referenced relatively. |
| **R26.6** | Writing `topics.js` MUST preserve its formatting and comments, and MUST leave a `.bak`. |
| **R26.7** | The topic list MUST be reorderable by drag and drop, and MUST remain hand-editable afterwards. |
| **R26.8** | Slug collisions MUST get a numeric suffix rather than overwriting an existing page. |
| **R26.9** | PDF is **out of scope** — it carries no reliable structure, so it would be the only format needing guesswork. |

---

## 27. Acceptance tests

The system is complete when a fresh build passes all of these by hand.

### Reader

| # | Test | Pass |
|---|---|---|
| 1 | Open a page by double-clicking the file | renders fully, fonts included, no console errors |
| 2 | Press `f` | a band appears on the first line |
| 3 | Press `↓` twenty times | band advances monotonically, scrolling as needed |
| 4 | Click a line halfway down | band jumps there |
| 5 | Select a sentence with the mouse | band does **not** move |
| 6 | Press `t` three times | paper → sepia → dark → paper |
| 7 | Press `{` and `}` | text resizes, band stays aligned |
| 8 | Press `-` five times | text softens, no layout shift |
| 9 | Press `z` | side columns and top bar vanish; column is full width |
| 10 | Reload | theme, sizes and position are restored |
| 11 | Open a different topic, then return | each remembers its own position |
| 12 | Resize the window narrow, then wide | band remains correctly aligned |
| 13 | Collapse a rail group, open another page | it is still collapsed |

### Read aloud

| # | Test | Pass |
|---|---|---|
| 14 | Scroll to mid-document, press `s` | reading starts from what is on screen, not the top |
| 15 | Let it run past a paragraph end | continues into the next block automatically |
| 16 | Let it reach a code block with skip on | announces briefly, does not read the code |
| 17 | Pick a voice, reload | the same voice is still selected |
| 18 | Open the same page in another browser | falls back cleanly if that voice is absent |
| 19 | Press `↓` while reading | skips a block rather than nudging the band |
| 20 | Press `Esc` | speech stops immediately |

### Pipeline

| # | Test | Pass |
|---|---|---|
| 21 | Build a page from Markdown | summary line reports the expected counts |
| 22 | Rebuild the same source | byte-identical output |
| 23 | Run the structural audit (§14.1) | zero findings |
| 24 | Run the highlighter audit (§14.2) | zero misaligned, zero backward |

### Importer

| # | Test | Pass |
|---|---|---|
| 25 | Drop a `.docx` | page created, topic registered, appears in the rail |
| 26 | Drop a semantic `.epub` | real chapter titles, no empty parts |
| 27 | Drop a Calibre-converted `.epub` | still real titles, no `part0002` |
| 28 | Reorder topics in the library | `topics.js` rewritten, still readable and commented |

---

## 28. Build order

If starting from nothing, this order keeps the system testable at every stage.

| Stage | Deliverable | Verified by |
|---|---|---|
| 1 | `study.css` — layout, three themes, tokens | a hand-written `template.html` renders correctly |
| 2 | `topics.js` + rail construction | rail lists topics, groups collapse |
| 3 | Contents generation + scroll-spy | contents tracks scroll |
| 4 | `collect()` and `rowsOf()` | logs the right number of line units |
| 5 | `paint()` + focus mode | band appears and moves — **the hard part** |
| 6 | Control panel | every control works and persists |
| 7 | `md2study.js` — Markdown front-end | a generated page matches a hand-written one |
| 8 | `fonts.js` | pages render offline with the right faces |
| 9 | Read aloud | tests 14–20 pass |
| 10 | Importer | tests 25–28 pass |

**Stage 5 is where the difficulty is concentrated.** Everything before it is
ordinary DOM work; everything after it is additive. Budget accordingly, and get
the geometry right before building anything on top of it.
