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
