# Build prompt — SkillForge DSA training camp

> **How to use this file.** Paste everything below the line into Claude Code (or another coding assistant) in an empty folder on the new machine. It describes the whole SkillForge **DSA** app: what it does, how it looks (every colour and font), how it behaves, and all of its content. Build it in the order given in §14. If you still have the original project, copying the folder is faster; this prompt is for rebuilding it from scratch.

---

## 0. The request

Build **SkillForge — DSA training camp**: a local, beginner-friendly website for learning **Data Structures & Algorithms in Java** for service-company interviews (TCS, Infosys, Wipro, Accenture, Cognizant, Capgemini, HCL, Tech Mahindra and similar). The learner is a working developer who wants **plain, simple English**, **Java 8+ code**, **visuals**, and a **daily 30-minute routine** they can follow like an Olympic training camp.

Every time the learner opens the site, **Today** tells them exactly what to train. The sessions follow a **94-day plan** that covers every pattern, every question type, every practice problem, every Java block and every edge-case trap. It has **competition days** and a **mock-interview week**. Code runs locally: a small Node server compiles and runs Java against hidden tests.

**Non-negotiables**
- Runs fully locally: `npm start` (or `start.bat` on Windows) → http://localhost:7070.
- Plain HTML + CSS + vanilla JavaScript ES modules. **No framework and no build step.**
- Node.js 20.12+ server with no dependencies except the optional `@anthropic-ai/sdk`. JDK 17+ must be on PATH (`java`, `javac`).
- All progress lives in the browser's `localStorage`, under keys prefixed `sf.`. Images live on disk in `data/uploads/`.
- Writing style: short sentences, everyday analogies, no jargon without an explanation, and a worked example for every idea.
- Every code sample must compile. Every practice solution must pass its tests. Tools in §13 verify this.

---

## 1. Tech stack

| Part | Choice |
|---|---|
| Server | `server.mjs` — Node `http` module, static files from `public/`, JSON API (§12) |
| Java runner | `lib/javaRunner.mjs` — write `Main.java` to a temp dir, `javac` once, run `java` once per test input on stdin. JVM options: `-XX:+UseSerialGC -XX:TieredStopAtLevel=1 -Xss64m -Dfile.encoding=UTF-8 -Dstdout.encoding=UTF-8 -Dstderr.encoding=UTF-8`. 5 s timeout per run, output capped at 64 KB, verdicts: Accepted / Wrong Answer / Runtime Error / Time Limit / Compile Error, plus time in ms per test |
| Optional AI | `lib/ai.mjs` — Anthropic SDK (use the latest Claude model), enabled only when `.env` has `ANTHROPIC_API_KEY`. Streaming chat over Server-Sent Events. System prompts per mode: `tutor`, `hint` (one progressive hint that moves the learner forward and never gives the full answer), `review` (a senior Java interviewer reviews correctness, complexity, edge cases and style), `drill`. Also a problem generator that returns JSON (title, statement, input/output format, constraints, hints, approach, complexity, solution, tests); the server compiles and tests it before saving |
| Editor | **CodeMirror 5.65.16** from cdnjs: `codemirror.min.js`, `mode/clike`, `addon/edit/matchbrackets`, `addon/edit/closebrackets`, `addon/selection/active-line`, `addon/hint/show-hint` (+ its CSS) |
| Markdown for AI output | `marked@12.0.2` + `dompurify@3.1.6` from jsDelivr (fall back to a tiny built-in markdown) |
| Fonts | Google Fonts **Inter** (400/500/600/700/800) for the UI, **JetBrains Mono** (400/600) for code |
| Icons | Inline SVG stroke icons (24×24 viewBox, stroke 2, round caps and joins), drawn at 16 px: menu, search, code, flame, layers, puzzle, target, scale, sparkles, home, check, x, format, chevronLeft/Right/Up/Down, maximize, minimize, play, pause, send, clock, arrow, reset, eye, book, trophy, alert, bulb, bot, flag, shuffle, briefcase, thumb, trash, share, dots, heart, bookmark, pen, image, message |

Suggested folders:
```
server.mjs  start.bat  package.json  .env (optional)  README.md
lib/javaRunner.mjs  lib/ai.mjs
data/uploads/
public/index.html
public/css/app.css  camp.css  themes.css  notes.css
public/js/main.js ui.js store.js srs.js shell.js api.js learn.js progress.js program.js
          mistakes.js notes.js feed.js feed-view.js focus-read.js attachments.js visual.js
          java-format.js java-hint.js practice-starter.js recall-view.js practice-ui.js
public/js/views/today.js program.js learn.js coding.js java.js java-practice.js review.js notes.js
public/js/data/library.js problems.js warmup.js patterns-*.js service-*.js service-bank.js
          java-streams.js java-streams-output.js trap-examples.js learn/*.js
tools/verify.mjs check-snippets.mjs check-streams.mjs check-format.mjs check-traps.mjs
      import-service-csv.mjs
```

---

## 2. Design system

### 2.1 Colour tokens

Every colour is a CSS variable on `:root`. The theme is set with `<html data-theme="…" data-tone="light|dark">`. The **default theme is `dark`**. The same tokens are used everywhere, so a theme switch recolours the whole app, including code windows and the editor.

**Dark (default, GitHub-dimmed style)**
```css
--bg:#1b1f24; --panel:#22272e; --panel-2:#2a3038; --panel-3:#313842; --border:#353c45;
--text:#e6edf3; --muted:#9aa4af; --faint:#6e7781;
--accent:#3aa0ff; --accent-strong:#1f6feb; --accent-soft:rgba(58,160,255,.14);
--green:#2ea043; --green-bright:#3fb950; --green-soft:rgba(63,185,80,.14);
--red:#f85149; --red-soft:rgba(248,81,73,.14); --yellow:#d29922; --yellow-soft:rgba(210,153,34,.16);
--orange:#f47c6c; --purple:#a371f7; --purple-soft:rgba(163,113,247,.15);
--heat-0:#2b3139; --heat-1:#0e4429; --heat-2:#006d32; --heat-3:#26a641; --heat-4:#39d353;
--code-bg:#1e2228; --shadow:0 8px 30px rgba(0,0,0,.35); --radius:10px;
--font:'Inter',system-ui,-apple-system,'Segoe UI',sans-serif;
--mono:'JetBrains Mono',ui-monospace,'Cascadia Code',Consolas,monospace;
```

**Paper (light)**
```css
--bg:#f3f5f8; --panel:#fff; --panel-2:#f5f7fa; --panel-3:#eaeef2; --border:#d8dee4; --text:#1f2328; --muted:#59636e; --faint:#8c959f;
--accent:#0969da; --accent-strong:#0969da; --accent-soft:rgba(9,105,218,.1); --green:#1f883d; --green-bright:#1a7f37; --green-soft:rgba(31,136,61,.1);
--red:#cf222e; --red-soft:rgba(207,34,46,.1); --yellow:#9a6700; --yellow-soft:rgba(154,103,0,.12); --orange:#d1573f; --purple:#8250df; --purple-soft:rgba(130,80,223,.1);
--heat-0:#ebedf0; --heat-1:#9be9a8; --heat-2:#40c463; --heat-3:#30a14e; --heat-4:#216e39; --code-bg:#f6f8fa; --shadow:0 8px 30px rgba(31,35,40,.12);
```

**Sepia (light)**
```css
--bg:#f4ecd8; --panel:#fbf5e6; --panel-2:#f3e9d2; --panel-3:#e9dcbf; --border:#dccdab; --text:#3b2f22; --muted:#74624d; --faint:#a08b6f;
--accent:#8b2e2e; --accent-strong:#8b2e2e; --accent-soft:rgba(139,46,46,.1); --green:#4f7a28; --green-bright:#3f6b1c; --green-soft:rgba(79,122,40,.12);
--red:#b3261e; --red-soft:rgba(179,38,30,.1); --yellow:#946200; --yellow-soft:rgba(148,98,0,.12); --orange:#c0653a; --purple:#7b4f9a; --purple-soft:rgba(123,79,154,.1);
--heat-0:#e6d9bc; --heat-1:#c9d69a; --heat-2:#9bb862; --heat-3:#6f9a3a; --heat-4:#4f7a28; --code-bg:#f3e9d2; --shadow:0 8px 30px rgba(80,60,30,.14);
```

**Mist (light)**
```css
--bg:#e9eef3; --panel:#f7f9fb; --panel-2:#edf1f5; --panel-3:#e0e7ee; --border:#cfd8e2; --text:#1c2833; --muted:#566676; --faint:#8795a3;
--accent:#2b6cb0; --accent-strong:#2b6cb0; --accent-soft:rgba(43,108,176,.1); --green:#2f855a; --green-bright:#276749; --green-soft:rgba(47,133,90,.1);
--red:#c53030; --red-soft:rgba(197,48,48,.1); --yellow:#b7791f; --yellow-soft:rgba(183,121,31,.12); --orange:#dd6b20; --purple:#6b46c1; --purple-soft:rgba(107,70,193,.1);
--heat-0:#dde4ec; --heat-1:#a3d9c0; --heat-2:#63b98f; --heat-3:#38996b; --heat-4:#276749; --code-bg:#edf1f5; --shadow:0 8px 30px rgba(28,40,51,.12);
```

**Nord (dark)**
```css
--bg:#2e3440; --panel:#3b4252; --panel-2:#434c5e; --panel-3:#4c566a; --border:#4c566a; --text:#eceff4; --muted:#b8c1d1; --faint:#8390a8;
--accent:#88c0d0; --accent-strong:#5e81ac; --accent-soft:rgba(136,192,208,.15); --green:#8fbc6f; --green-bright:#a3be8c; --green-soft:rgba(163,190,140,.16);
--red:#bf616a; --red-soft:rgba(191,97,106,.16); --yellow:#ebcb8b; --yellow-soft:rgba(235,203,139,.16); --orange:#d08770; --purple:#b48ead; --purple-soft:rgba(180,142,173,.16);
--heat-0:#434c5e; --heat-1:#4f6b58; --heat-2:#6a8f5e; --heat-3:#8fbc6f; --heat-4:#a3be8c; --code-bg:#3b4252; --shadow:0 8px 30px rgba(0,0,0,.3);
```

**Night (dark, near-black)**
```css
--bg:#0a0d12; --panel:#10141b; --panel-2:#161b24; --panel-3:#1d2330; --border:#232a36; --text:#e4e8ee; --muted:#8a95a5; --faint:#5d6778;
--accent:#6cb6ff; --accent-strong:#316dca; --accent-soft:rgba(108,182,255,.12); --green:#2ea043; --green-bright:#57ab5a; --green-soft:rgba(87,171,90,.14);
--red:#e5534b; --red-soft:rgba(229,83,75,.14); --yellow:#c69026; --yellow-soft:rgba(198,144,38,.15); --orange:#f0806b; --purple:#b083f0; --purple-soft:rgba(176,131,240,.14);
--heat-0:#1a202a; --heat-1:#0e4429; --heat-2:#006d32; --heat-3:#26a641; --heat-4:#39d353; --code-bg:#0d1117; --shadow:0 8px 30px rgba(0,0,0,.5);
```
Set `color-scheme: light` on Paper, Sepia and Mist, and `dark` on the others. Treat the legacy value `light` as `paper`.

**Code-window palettes** (`--cw-*`, one set per theme; used by code windows and the CodeMirror editor)

| theme | bg | head | border | text | muted | line-no | keyword | string | number | comment | function | type | var |
|---|---|---|---|---|---|---|---|---|---|---|---|---|---|
| dark | #1c2128 | #22272e | #373e47 | #cdd9e5 | #768390 | #636e7b | #f47067 | #96d0ff | #6cb6ff | #768390 | #dcbdfb | #cdd9e5 | #f69d50 |
| paper | #f6f8fa | #eef1f4 | #d0d7de | #1f2328 | #59636e | #8c959f | #cf222e | #0a3069 | #0550ae | #6e7781 | #8250df | #1f2328 | #953800 |
| sepia | #f8f0dd | #efe3c8 | #dccdab | #3b2f22 | #74624d | #a89373 | #a3322b | #4f7a28 | #9a5b13 | #93836a | #6a4c93 | #3b2f22 | #9a5b13 |
| mist | #f3f6f9 | #e5ebf1 | #cfd8e2 | #1c2833 | #566676 | #8795a3 | #c53030 | #276749 | #2b6cb0 | #718096 | #6b46c1 | #1c2833 | #c05621 |
| nord | #2e3440 | #353c4a | #4c566a | #d8dee9 | #9aa5b8 | #616e88 | #81a1c1 | #a3be8c | #b48ead | #7b88a1 | #88c0d0 | #8fbcbb | #d08770 |
| night | #0d1117 | #0d1117 | #30363d | #e6edf3 | #8b949e | #6e7681 | #ff7b72 | #a5d6ff | #79c0ff | #8b949e | #d2a8ff | #e6edf3 | #ffa657 |

Difficulty badges: Easy = green-soft background with green-bright text; Medium = yellow-soft with yellow; Hard = red-soft with red. Other badges: `accent` and `purple`.

### 2.2 Typography and size

- Body text is `14px × var(--fs)`, line-height 1.5, with antialiasing. Every font size in the CSS is written as `calc(Npx * var(--fs, 1))` so the **text size control** scales text only, never the layout.
- Text size steps: 0.85, 0.9, **1**, 1.1, 1.2, 1.35, 1.5. The controls are **A−** / **A+** in the Theme menu (clicking the percentage resets it). Shortcuts: **Ctrl+Alt+=** bigger, **Ctrl+Alt+−** smaller, **Ctrl+Alt+0** reset. Plain Ctrl+/− stays the browser zoom. Saved as `sf.fontScale`.
- Turn off font ligatures in code (`font-variant-ligatures: none`).
- `:focus-visible` shows a 2px accent outline with a 2px offset.

### 2.3 Page chrome

- **Top bar** (sticky, 60px tall, `--panel` background, bottom border), left to right:
  - ☰ menu button, which toggles the left panel.
  - Logo: **skill**`forge`, 22px, weight 800, letter-spacing −0.5px, with "forge" in `--accent`.
  - Rounded search box: 40px tall, 20px radius, `--panel-2` background, search icon inside. Placeholder "Search patterns, recipes, problems, Java blocks… (press /)". Results drop down in a panel (arrow keys to move, Enter to open, Esc to close).
  - A spacer.
  - **Mistake** button (logs a mistake).
  - Server status pill: a dot plus "Java 17 · AI on/off".
  - Streak: 🔥 icon and number in `--orange`, bold.
  - **Theme** picker.
  - Round avatar with the initial "L" (34px, `--accent-strong` background).
- **Navbar** (sticky under the top bar, centred): a pill container with `--panel` background, 1px border, 14px radius and 5px padding. It holds links with an icon and label (9px × 14px padding, 10px radius, muted text, weight 600). The active link has accent text on `--accent-soft` with a small triangle pointer under it. Sections: **Today (flag) · Plan (layers) · Learn (book) · Warmup (flame) · Patterns (puzzle) · Service DSA (briefcase) · Java (code) · Review (reset) · Notes (pen)**. Review shows a red count of due recall cards.
- **Theme menu** (280px popover):
  - "Theme" label and 6 chips (Paper, Sepia, Mist, Dark, Nord, Night). Each chip is uppercase monospace, drawn in its own theme's colours, and the active chip has an accent outline.
  - "Text size" row: A− · 100% · A+.
  - Theme is saved as `sf.theme`. An inline `<script>` in `<head>` applies the saved theme and text size **before first paint**, so there is no flash.
- **Workspaces** (coding and library pages): a 3-column grid `360px | 1fr | 290px`, height `calc(100vh − 124px)`. Left panel has a right border; right panel has a left border and 16px padding. The centre uses `--bg`.
  - **Side-panel handles:** a small ‹/› button sits on the edge of the left panel (and of the right panel on coding pages) to hide or show it. **Ctrl+B** toggles the left panel. The state is remembered. Collapsing the left panel removes its column.
  - When the right panel is hidden on coding pages, **Run** and **Submit** move into the editor bar.
  - On narrow screens (< 900px) the layout stacks, and ☰ opens the left panel.
- **List items** (`.qitem`): a round check circle (turns green with a white tick when done), a title with ellipsis, and a difficulty badge. The active item uses accent-soft. Group headers are 11px uppercase muted, letter-spacing .06em.
- **Buttons:** `.btn` (panel background with border), `.primary` (accent-strong), `.success` (green), `.success-outline`, `.ghost`, `.danger`, plus sizes `.sm`, `.lg` and `.block`. **Segmented controls** (`.seg`): the active segment is accent-strong with white text. **Cards:** `--panel` background, 1px border, 10px radius. **Toasts** appear bottom-right. **Modals** close on Esc or a backdrop click.
- **Code window** (used for every code sample in the app):
  - Head: three macOS dots (#ff5f57, #febc2e, #28c840), the language label, a **copy** button and a **⤢** full-screen button.
  - Body: line numbers and Java syntax colours from the `--cw-*` palette. 12px radius and a soft shadow.
  - ⤢ opens the code full screen with bigger text; Esc, **exit** or a click outside closes it.
  - A tiny built-in Java highlighter handles keywords, types, strings, numbers, comments and method names.
- **Heatmap**: a GitHub-style activity grid (weeks × 7 days) using `--heat-0…4`.

---

## 3. Data model and storage

`store.js` wraps localStorage with prefix `sf.`, and every read is wrapped in try/catch. It provides `load/save/update/onChange`, plus:
- Local dates (`YYYY-MM-DD`), so the streak rolls over at the learner's own midnight.
- `recordActivity()` (per-day counts, which feed the heatmap and streak) and `streak()`.
- Solved problems, attempts per problem, "done" sets (`learned`, `javaStudied`, `trapsStudied`) and quiz answers.

`srs.js` handles spaced repetition (SM-2 style). A card looks like `{id, type, ref, front, back, code?, codeLang?, images?, link?, due, interval, ease=2.5, reps, lapses, created}`. Grades:
- **again** → interval 0 (due again today), ease −0.2 (minimum 1.3), lapses +1, reps reset.
- **hard** → interval ×1.2 (minimum 1), ease −0.15.
- **good** → 1 day on the first rep, 3 days on the second, then interval × ease.
- **easy** → 3 days on the first rep, otherwise interval × ease × 1.3 (minimum 3), ease +0.15.

Each grade button shows the interval it would give. Keys: **1–4** grade, **Space** shows the answer. There is a per-day review log.

Recall cards are created by:
- Marking a question type learned (`recipe:` — front: pattern → the question; back: recipe + memory move + explanation + code).
- **I can write this** on a Java block (`java:`).
- **Got it** on a trap (`trap:`).
- An **Accepted** problem (`problem:` — a re-solve card, pre-graded "good" so its first review is in a few days).
- Every logged mistake (`mistake:`).
- A note turned into a card (`note:`).

---

## 4. Today

This page runs today's session from the 94-day plan.
- **Hero:** "Day N of 94", the phase name, today's title, a **session clock** (30:00 countdown with start/pause), streak, and schedule status (on track / N days behind / ahead).
- **Four blocks**, each with a tick-off and a direct link:
  - **A Warm-up:** recall reps. Shows due cards up to a cap and opens the Review recall room.
  - **B Technique:** learn 1–2 question types, or edge-case traps on foundation days.
  - **C Main set:** solve in Java (opens the problem in the right track), or "rebuild the recipe from memory" when the pattern has no problem left.
  - **D Cool-down:** a Java block, plus a one-line reflection box.
- **Complete day** marks the day done; the camp can also be started, restarted, or moved to any day.
- **Coming up:** the next few days. The side column shows the streak and heatmap, due cards, solved counts per track, and learned counts.

## 5. Plan (the 94-day camp)

Build the plan **deterministically** from the content (no hand-written list), so every item appears exactly once:

1. **Phase 0 — Foundations.** Take the foundation problems two per day: `w-array-sum, w-max-min, w-second-largest, w-fizzbuzz, s-reverse-integer, s-palindrome-number, s-plus-one, s-max-consecutive-ones`.
   - Blocks: A Orientation (day 1) or Recall (5 min, cap 12) · B Edge-case traps ×4 (8 min) · C Warm-up problems (15 min) · D Java blocks ×4 (2 min).
2. **Phases 1–6.** Go through the patterns in phase order (§7.1), two question types per day.
   - Each day: A Recall (5 min, cap 12) · B Technique: pattern (8 min, 2 question types) · C Solve in Java (15 min, the next unassigned practice problem of that pattern, else "Rebuild the recipe from memory") · D Java block + reflection (2 min).
   - A long pattern is split into "part 1/3" and so on.
3. **Competition day after every 6 training days:** A Recall (7 min, cap 25) · B two timed problems in **Interview mode** (18 min, leftovers first, else re-solve earlier ones) · C Film review of your mistakes (3 min) · D two edge-case traps.
4. **Phase 7 — Service-company mock week.** At least 5 days of three problems each, taken from the mock pool: `s-roman-to-integer, s-integer-to-roman, s-longest-common-prefix, s-reverse-words, s-add-binary, s-pascals-triangle, s-rotate-image, s-spiral-matrix, s-next-permutation, s-count-primes, s-atoi, p-max-sum-k, s-three-sum, s-trapping-rain-water, s-median-two-sorted` plus any leftovers. The last day is the **Graduation mock interview**. Blocks: Recall · Interview-mode problems (20 min) · Film review · Java blocks.
5. Any Java blocks or traps still unplaced go on the last days, so nothing is skipped. With the content in §7 this comes to **94 days**.

The Plan page shows:
- A progress bar and completed count.
- Days grouped by phase, with a kind badge (Foundations / Training / Competition / Mock).
- Each day's title and blocks, plus open, jump-to and mark-done actions.
Camp state is saved in `sf.camp` as `{startedOn, days:{n:{done, blocks}}}`.

---

## 6. Learn

- **Left panel:** the 6 phases → patterns → question types, each with a learned tick and progress, plus a filter box. There is also a **Recipe map** tab: a searchable table of pattern → question → recipe → key move.
- **Pattern page:**
  - The idea in plain English, an everyday **analogy**, and "How to spot it" bullets.
  - A **step-by-step visual**: an array of boxes with pointer labels (i, j, L, R…), done/bad colouring and a note per frame, with Prev/Next/Play controls.
  - The Java **template**, and the list of question types with learned state.
  - Practice problems linked to the Warmup/Patterns tracks.
- **Question-type (lesson) page:**
  - The **question** it answers (e.g. "Have I seen this value before?") and the **recipe** (e.g. HashSet).
  - A **memory sentence** (pattern → question → move → tool).
  - Plain-English explanation ("say"), numbered **steps**, a tiny worked **trace**, and the **Java code** in a code window.
  - Time and space complexity, the keywords that give it away, and the **trap**.
  - Practice problem links.
  - **Rebuild from memory:** an editor with the code hidden. The learner writes it, then compares.
  - **Mark learned**, which creates a recall card.
  - Previous/next navigation.

## 7. Content

### 7.1 The 26 patterns and 132 question types (question type [recipe])

Phases, in study order:
- **1 Arrays, Strings & Hashing:** hashing, frequency-counting, prefix-sum, kadane, two-pointers, sliding-window-fixed, sliding-window-dynamic
- **2 Stacks, Queues & Heaps:** stack, monotonic-stack, queue, monotonic-queue, heap, top-k
- **3 Sorting, Searching & Greedy:** intervals, binary-search, greedy
- **4 Linked Lists, Trees & Tries:** linked-list, trees, trie
- **5 Graphs:** bfs-dfs, graphs, topological-sort, union-find
- **6 Recursion, DP & Bits:** backtracking, dynamic-programming, bit-manipulation

| Pattern | Question types [recipe] |
|---|---|
| Hashing | Seen before / duplicate detection [HashSet] · Complement lookup [HashMap] · Fast lookup [HashMap / HashSet] · Grouping [HashMap<Key, List<...>>] |
| Frequency Counting | Count occurrences [HashMap<T,Integer>] · Small bounded values [Frequency array] · Most / least frequent [Frequency map + scan/heap] · Anagram equality [Compare frequencies] |
| Prefix Sum | Range sum query [prefix[r+1] − prefix[l]] · Subarray sum equals K [Prefix sum + HashMap] · Zero-sum subarray [Repeated prefix sum] · 2D rectangle sum [2D prefix matrix] |
| Kadane | Maximum subarray [Running best] · Minimum subarray [Running minimum] · Track actual range [Kadane + indices] · Circular maximum subarray [total − minimum subarray] |
| Two Pointers | Pair in sorted array [Left + right pointers] · Palindrome [Compare both ends] · Remove duplicates in-place [Read + write pointer] · Container / max area [Move limiting side] |
| Sliding Window Fixed | Window sum / average [Add right, remove left] · Count condition in every K window [Running count] · Find anagrams [Window frequency map] · Window max/min [Monotonic deque] |
| Sliding Window Dynamic | Longest valid window [Expand right, shrink when invalid] · Smallest valid window [Expand until valid, shrink aggressively] · At most K distinct [Frequency map + left pointer] · No repeats [Set / last-seen map] |
| Stack | Balanced brackets [Push opens, match closes] · Expression evaluation [Operand/operator stack] · Undo history [Push actions] · Iterative DFS [Explicit stack] |
| Monotonic Stack | Next greater element [Decreasing stack] · Next smaller element [Increasing stack] · Previous greater/smaller [Clean stack then peek] · Daily Temperatures [Decreasing stack of indices] · Largest Rectangle [Increasing stack + widths] |
| Queue | FIFO processing [offer → poll] · Tree level order [Queue by level] · BFS frontier [Queue] · Work buffer [Producer/consumer queue] |
| Monotonic Queue | Sliding window maximum [Decreasing deque] · Sliding window minimum [Increasing deque] · Expire old values [Store indices] · DP window optimization [Deque of best states] |
| Heap | Repeated smallest [Min Heap] · Repeated largest [Max Heap] · Kth largest [Min heap of size K] · Kth smallest [Max heap of size K] · Merge K sorted streams [Min heap of heads] · Scheduling by end time [Min heap of end times] |
| Top K | Top K largest [Min heap size K] · Top K smallest [Max heap size K] · Top K frequent [Frequency + heap/bucket] · K closest points [Max heap size K] · Streaming Top K [Bounded heap] |
| Intervals | Detect any overlap [Sort by start → check adjacent] · Merge overlaps [Sort by start → compare last merged] · Insert interval [Before → merge → after] · Minimum meeting rooms [Sort + min heap of end times] · Maximum simultaneous intervals [Sweep line] · Remove minimum overlaps [Sort by end → greedy] · Interval intersection [Two pointers] |
| Binary Search | Exact search [Classic] · First / last occurrence [Biased binary search] · Lower / upper bound [Boundary binary search] · Rotated array [Identify sorted half] · Binary search on answer [Monotonic feasibility] · Peak finding [Compare mid with neighbor] |
| Linked List | Reverse list [prev / curr / next] · Middle node [Slow + fast] · Cycle detection [Floyd slow/fast] · Merge sorted lists [Dummy head + tail] · Remove nth from end [Two pointers with gap] · Reorder list [Middle → reverse → merge] |
| Trees | Preorder [Node → left → right] · Inorder [Left → node → right] · Postorder [Left → right → node] · Height / depth [DFS returns child result] · Path problems [DFS + path state] · Lowest common ancestor [Recursive split / BST ordering] |
| BFS/DFS | Reachability [DFS or BFS + visited] · Shortest unweighted path [BFS] · Connected components [Traversal from every unvisited node] · Flood fill / islands [Grid DFS/BFS] · Tree levels [BFS by queue size] |
| Graphs | Adjacency list [List of neighbors] · Undirected connectivity [DFS/BFS or Union Find] · Weighted shortest path [Dijkstra] · Negative edges [Bellman-Ford] · All-pairs shortest path [Floyd-Warshall] · Cycle detection [DFS colors / Union Find] |
| Topological Sort | Course schedule possible? [Kahn / DFS cycle detection] · Produce dependency order [Indegree queue] · Build order [Topological ordering] · Cycle in directed graph [Processed count / DFS colors] |
| Union Find | Connectivity query [find(a) == find(b)] · Merge groups [union(a,b)] · Path compression [Flatten find path] · Union by rank/size [Attach smaller under bigger] · Redundant edge [Already connected?] · Count components [Start n, decrement on union] |
| Backtracking | Subsets [Choose / skip] · Permutations [Choose unused item] · Combination Sum [Choose candidate repeatedly] · N-Queens [Place → validate → recurse → remove] · Sudoku [Fill → recurse → undo] · Word Search [Grid DFS + temporary visited] |
| Greedy | Interval scheduling [Sort by end → take earliest finish] · Jump Game [Track farthest reachable] · Gas station [Reset after failed prefix] · Minimum arrows / overlap removal [Sort intervals strategically] · Activity selection [Earliest compatible finish] |
| Dynamic Programming | 1D choose / skip [dp[i] from earlier states] · Knapsack [Item × capacity state] · Grid DP [Reuse top/left/neighbors] · LCS [2D prefix DP] · LIS [DP or tails + binary search] · Coin Change [Amount state] · Memoization [Recursive state + cache] |
| Trie | Insert word [Create child per character] · Exact search [Walk + end marker] · Prefix search [Walk prefix only] · Autocomplete [Prefix node + DFS descendants] · Word Search II [Trie + grid DFS] |
| Bit Manipulation | Check bit [x & (1<<k)] · Set bit [x \| (1<<k)] · Clear bit [x & ~(1<<k)] · Toggle bit [x ^ (1<<k)] · Single Number [XOR everything] · Power of two [n & (n−1)] · Enumerate subsets [Bitmask 0..2^n−1] |

Every question type needs these fields:
- `slug`, `name`, `question`, `recipe`, `move` and `memory`.
- `keywords`: the words that give the pattern away.
- `example`, e.g. `[4, 2, 7, 2] → when the second 2 arrives, the set already contains 2.`
- `say` (plain English), `steps[]` and `trace[]` (a tiny worked example).
- `code`: correct, compilable Java for this exact question type.
- `time`, `space` and `trap`.
- `practice[]`: named LeetCode problems.

Every pattern needs `idea`, `analogy`, `spot[]`, `frames[]` (the step-by-step visual) and `template`. Example of the tone:
> **Hashing** — idea: *A HashSet or HashMap is a notebook that answers "have I seen this?" or "what did I store for this?" in almost no time — O(1) on average.* Analogy: *Like a school register: instead of searching every desk for a student, you look up the name and instantly know if they are present.*

There are also **80 recipe cards** (one per core question type), each with pattern, name, move, memory sentence, example problem, tool and Java. The **Recipe map** has 26 rows: pattern → [question, recipe, key move].

### 7.2 Practice problems (104 runnable in the app + 384 in the bank)

**Problem format:** `{id, title, difficulty, topic, statement, inputFormat, outputFormat, constraints[], hints[], approach, complexity{time,space}, tests[{in,out}] (≥5, including edge cases), solution, track, patternId?}`.
- The solution is a full `public class Main` that reads **stdin with Scanner** and prints to stdout.
- The solution method's body sits between the markers `// >>> solution` and `// <<< <fallback line, e.g. return 0;>`. The **starter code** is generated by replacing that part with `// TODO: write your solution here` plus the fallback line.
- The first **2 tests are samples** (visible); the rest are hidden.
- Compare output with trimmed lines and normalised whitespace.

- **Warmup (15, ids `w-*`):** Sum of an Array · Reverse a String · Maximum and Minimum · Count Vowels · Valid Palindrome · FizzBuzz · Second Largest Distinct Element · Character Frequency · Two Sum · Running (Prefix) Sums · Move Zeroes · Valid Parentheses · Binary Search · N-th Fibonacci (mod 1e9+7) · Valid Anagram
- **Patterns (45, ids `p-*`)**, grouped by 19 problem patterns: Two Pointers, Sliding Window, Fast & Slow Pointers, Merge Intervals, Cyclic Sort, Modified Binary Search, Prefix Sum + HashMap, Monotonic Stack, Heap / Top-K, BFS / DFS on Grids, Topological Sort, Union-Find, Shortest Path (Dijkstra), Backtracking, DP (1D), DP (2D), Greedy, Bit Manipulation, Trie. Each problem pattern has a summary and "recognize it when" bullets. The problems: Pair With Target Sum (Sorted) · Container With Most Water · Maximum Sum Subarray of Size K · Longest Substring Without Repeating Characters · Minimum Size Subarray Sum · Find the Duplicate Number · Happy Number · Merge Overlapping Intervals · Meeting Rooms II · Missing Number · First Missing Positive · First and Last Position in a Sorted Array · Search in Rotated Sorted Array · Koko Eating Bananas · Subarray Sum Equals K · Range Sum Queries · Next Greater Element · Daily Temperatures · Largest Rectangle in Histogram · K-th Largest Element · Top K Frequent Elements · Number of Islands · Rotting Oranges (multi-source BFS) · Course Schedule · Course Schedule II (smallest order) · Number of Connected Components · Redundant Connection · Network Delay Time · Generate Parentheses · Subsets · N-Queens (count) · Climbing Stairs · House Robber · Coin Change · Longest Increasing Subsequence · Longest Common Subsequence · Edit Distance · Unique Paths · 0/1 Knapsack · Jump Game · Maximum Subarray (Kadane) · Non-overlapping Intervals · Single Number · Counting Bits · Count Words With Prefix
- **Service (44 local Java versions, ids `s-*`):** Longest Palindromic Substring · Reverse Integer · Best Time to Buy and Sell Stock (I & II) · Palindrome Number · Group Anagrams · Merge Sorted Array · Longest Common Prefix · Rotate Array · Add Two Numbers · Roman to Integer · Integer to Roman · Remove Duplicates from Sorted Array · Majority Element · Median of Two Sorted Arrays · 3Sum · Next Permutation · Pascal's Triangle · Reverse Words in a String · Contains Duplicate · Trapping Rain Water · Rotate Image · Sort Colors · Longest Consecutive Sequence · Product of Array Except Self · Spiral Matrix · Reverse Linked List · Merge Two Sorted Lists · Linked List Cycle · Min Stack · Plus One · Add Binary · Peak Index in a Mountain Array · Find the Winner of the Circular Game · Search Insert Position · Largest Number · Max Consecutive Ones · Squares of a Sorted Array · Letter Combinations of a Phone Number · Permutations · Count Primes · Sqrt(x) · First Unique Character in a String · String to Integer (atoi)
- **Service DSA bank (384 questions)**, imported from a company-wise CSV with `tools/import-service-csv.mjs <csv>`. Each entry is `{id (LeetCode number), title, url, difficulty, acceptance, frequency, companies[] (accenture, capgemini, infosys, wipro, hcl, tech-mahindra, cognizant, tcs, accolite, persistent-systems, altimetrik, mindtree, mphasis…), type}`. If the CSV is not available on the new machine, build the bank from well-known service-company LeetCode questions in this format.

### 7.3 Java (core blocks, traps and Streams)

**92 Java 8 blocks.** Each has `slug, group, name, move, memory, when, java, trap` and shows as a code window with an "I can write this" button. The groups:
- **Basics:** Iterate int[] · Iterate 2D int[][] · Sort int[] ascending/descending · Sort 2D by first/second column · Multiple keys · Sort List<Integer> · HashMap put/get · Frequency with getOrDefault · Frequency with merge · Grouping with computeIfAbsent · Iterate entries · HashSet · ArrayList · List<Integer> → int[] · List<int[]> → int[][] · String → char[] · charAt · StringBuilder · Deque as stack / queue · PriorityQueue min / max / custom comparator · Comparator ascending/descending · Math min/max · Integer extremes · long for sums/products · 4- and 8-direction grids · Stream filter / sort · method reference · computeIfAbsent graph · clone int[] · deep copy int[][].
- **Characters & ASCII (15):** char ↔ int, digit ↔ char, letter → index, isLetter/isDigit/isLetterOrDigit, manual ASCII checks, upper/lower case, filter special characters (two pointers / regex / builder), ASCII frequency array, sort characters, reverse string.
- **HashMap & Collections (15):** iterate via entrySet / keySet / values / forEach, reverse a map (unique and duplicate values), entry with max value, sort entries by value asc/desc or by key, putIfAbsent, computeIfAbsent, merge for counting, getOrDefault, HashMap → TreeMap.
- **Sorting & Reversing (11):** reverse int[] in place, sort int[] descending without boxing, Integer[] descending, reverse a List, List descending, sort strings by length (asc/desc), 2D first asc / second desc, sort then iterate descending, sort char[], reverse a StringBuilder.
- **Parsing & Conversion (6):** String ↔ int, char[] → String, String → char[], split on whitespace, substring.
- **Arrays & Utilities (7):** Arrays.fill, fill a 2D array, copy a range, swap helper, array max/min, sum safely, matrix bounds check.

**28 edge-case traps.** Each has a one-line explanation plus a **breaks vs safe** code pair; both sides must compile. The traps: Empty Input · Single Element · Off-by-One · Touching Intervals · All Negative · Duplicate Values · Integer Overflow · Null / Empty Collections · Modify While Iterating · Backtracking Aliasing · 2D Arrays · Binary Search Boundaries · Graph Visited Timing · Linked List Pointer Loss · Recursive Stack Depth · String vs Array Length · Character Index · Heap Empty · HashMap Null Semantics · Equality · char is not "just ASCII" · c − '0' · c − 'a' · Reverse Map Collision · Primitive Reverse Sort · Strings Are Immutable · Arrays.asList(int[]) · Regex vs Pointer Filtering.

**Java Streams: 46 interview questions (Q1–Q46) in 7 patterns.** All run on one shared sample dataset:
- `class Employee(name, dept, salary, age, rank, List<String> skills)` with 8 employees:
  - Alice (Engineering, 95,000, 29, rank 1, Java/Spring/SQL)
  - Bob (Engineering, 95,000, 34, rank 2, Java/Kafka)
  - Carol (Engineering, 78,000, 26, rank 3, Python/SQL)
  - Dave (Finance, 88,000, 41, rank 1, Excel/SQL)
  - Eve (Finance, 72,000, 28, rank 2, Excel/Tableau)
  - Frank (HR, 60,000, 45, rank 1, Recruiting)
  - Grace (HR, 65,000, 24, rank 2, Recruiting/Excel)
  - Heidi (Marketing, 70,000, 31, rank 1, SEO/Excel)
- `class Order(id, status, List<String> items, amount)` with 5 orders (SETTLED / PENDING / CANCELLED).
- `Map<String,String> managerOf`: an employee → manager hierarchy whose root has "-" as its manager.

The seven patterns:
1. **Filter + Collect:** filter on two conditions → names; distinct + sorted; immutable result; anyMatch; count.
2. **Group + Aggregate:** downstream collectors; count/sum/max per group; unwrap the Optional with collectingAndThen; group into lists; mapping; filtering (keeps empty keys); two-level grouping; summarizing stats; teeing; LinkedHashMap to keep order.
3. **Sort + Slice:** 2nd and Nth highest distinct salary; top K objects; multi-key sort; best rank per group with minBy; takeWhile.
4. **Flatten:** unique values from inner lists; filter then flatten; sum a field; flatMapping inside groupingBy; flatten then count per group.
5. **Partition:** split into two lists; partition + count; partition + names; partition then group; both keys always present.
6. **Strings:** joining with prefix/suffix; joining distinct values; Unicode-safe character frequency; first non-repeating character; group words by first letter.
7. **Employee → Manager hierarchy:** find the root; manager → direct reports (flip the map); direct report count; all reports (direct + indirect, BFS); chain of command; depth of every employee; people who manage nobody; print the org chart; lowest common manager.

Each Streams block has:
- `q` (Q-number), `when` (the task), the Java answer and a `memory` line (e.g. `filter → filter → map(getName) → collect(toList())`).
- A trap, and the variable to print.
- Its **real output**: `tools/check-streams.mjs` runs every block and writes `java-streams-output.js`, which the page shows.
- A **Java 8 version** when the answer uses a newer API (`toList()`, `teeing`, `takeWhile`…).

**Java page UI.** Left: groups and search. Centre: blocks. Traps tab.
- **Cards / Feed toggle.** Feed is a Twitter-style timeline with like, save, **I can write this** and **Cover the code** (hide the code until you tap, so it becomes a quiz).
- Saved posts also appear in Notes → Saved.
- Every block has a **Practice** button that opens an editor:
  - For Streams, the sample data is pre-loaded and a TODO sits at the top. **Run & check** compares your output with the expected output; a match adds the block to recall.
  - For core blocks, it opens a scratch `Main` program with **Show answer** to compare.

---

## 8. Coding pages (Warmup, Patterns, Service DSA)

A 3-column workspace:

- **Left panel:** the problem list. It has a filter/search box and difficulty and status filters (solved, attempted, new), and is grouped by topic or pattern. Service DSA's left panel is the **All Questions** bank, filterable by **company**, difficulty, type, status and "has Java tests". Choosing a question opens it in the editor.
- **Centre (top), tabs Description / Solution / AI:**
  - **Description:** statement, input/output format, constraints, examples (the first two tests), hints revealed one at a time, and a pattern callout (hidden in Interview mode until solved).
  - **Solution:** approach, complexity and the reference code. It unlocks according to the mode.
- **Centre (bottom):** the CodeMirror Java editor.
  - Editor bar: language label, **Format**, **Reset to starter**, **Full screen**.
  - Drafts autosave per problem.
  - Keys: Ctrl+Enter runs, Ctrl+Shift+Enter submits.
- **Test console** (collapsible, 42px when collapsed), tabs Testcase / Result / Custom input:
  - **Run** executes the sample tests.
  - **Submit** executes all tests. Each shows pass/fail and time per case; hidden tests show only their verdict in Interview mode.
  - The result shows expected vs your output, stderr, compile errors (with line numbers) and the verdict.
- **Right panel:**
  - **Mode:** Learn (hints and solution any time) / Practice (hints one by one; solution unlocks after a submission) / Interview (timed — Easy 20, Medium 35, Hard 45 minutes — with no hints and no solution until solved).
  - A timer, big **Run** and **Submit** buttons, and solved/attempt stats.
  - **AI Coach** buttons: Get a hint (disabled in Interview), Review my code (in Interview, only after solving) and, on the Patterns track, Generate a new problem. AI answers stream into the AI tab.
  - LeetCode-only bank questions also get **Generate a Java version with tests (AI)**.
  - A **Log this mistake** shortcut after a failed submission.
- **LeetCode-only bank questions** have no local tests. They get a generated starter template, custom-input runs, a link to LeetCode and **Mark as solved**.
- **Accepted** → confetti, the problem is marked solved, and a re-solve recall card is created.
- **Full-screen coding:** hides everything except the editor and the console. Run and Submit move into the editor bar; Esc exits.
- **Narrow editor:** secondary editor buttons shrink to icons with tooltips.

**Java autocomplete** (`java-hint.js`, through CodeMirror show-hint):
- Suggestions appear as you type; Enter or Tab accepts; Ctrl+Space opens the list.
- It knows the declared variables and their types (for example `map.get(k).` → List methods, when the map is `Map<K, List<V>>`), static members (`Math.`, `Arrays.`, `Collections.`, `Integer.`, `Character.`…) and the learner's own classes and methods.
- Snippets: `sout`, `souf`, `fori`, `forj`, `forr` (reverse loop), `fore` (for-each), `while`, `ifs`, `ifelse`, `psvm`, `scanner`, `readarr` (read n and int[] from stdin), `hashmap`, `arraylist`, `hashset`, `minheap`, `maxheap`, `stack`, `queue`, `sb` (StringBuilder), `dirs` (4-direction array), `binsearch`, `bfs`.

**Formatter** (`java-format.js`; **Format** button, Shift+Alt+F or Ctrl+Alt+L):
- Re-indents with 4 spaces, using 8 for continuation lines such as `.filter(...)` chains; switch/case bodies are indented too.
- Fixes spacing: `if(x){` → `if (x) {`, `a=b+c` → `a = b + c`, `f( a ,b )` → `f(a, b)`, `}else{` → `} else {`. Generics stay tight (`Map<String, List<Integer>>`) and unary minus stays attached (`-1`).
- Keeps the learner's line breaks, allows at most one blank line in a row, and never touches strings or comments.
- Is a single undo step, and formatting twice changes nothing.

## 9. Review

- **Recall room:** due cards one at a time. Front first; **Space** reveals the back, code window and images, plus a link to the source. Grade with Again/Hard/Good/Easy (keys 1–4), each showing its next interval. Session counter, "all done" state, and stats by card type.
- **Mistake journal:** a feed of post cards. Each card shows category, pattern, where it happened, what went wrong, the fix/lesson hidden behind **…more** (so reading it is recall practice), code blocks and images.
  - **Got it** / **Still tricky** reschedule the card.
  - Filter by *Due for recall* or by category. A bar chart shows mistakes per category.
- **All cards:** a browser for every recall card. Search, filter by type, see the due date, and delete or suspend.
- **Logging a mistake** (top-bar **Mistake** button, or after a failed submission) opens a modal with:
  - **Category:** Picked the wrong pattern · Off-by-one / boundary · Missed an edge case · Java syntax / API · Integer overflow · Too slow (TLE) · Forgot the recipe · Misread the problem · Logic bug · Other.
  - Pattern, where (problem/topic), what happened and the lesson.
  - **Add code block** (any number, each with a language: java, python, javascript, sql or text).
  - **Add image** (button, Ctrl+V paste, or drag and drop). Images upload to the server (downscaled above 1800px) and are deleted with the mistake.

## 10. Notes (Twitter-style feed)

- **Tabs:** For you / My notes / Saved, plus a **Hide panel** toggle (Ctrl+B) that widens the feed to one column.
- **Composer** (avatar "L"): text with **#hashtags** and `inline code`, code blocks, and images (button, paste, drop). **Post** or Ctrl+Enter.
- **For you** is an endless feed (infinite scroll plus a **Show more posts** button). The order:
  1. Due recall cards (notes, Java blocks, mistakes), each with a reveal and grade buttons.
  2. Posts from the pattern the learner is on in the camp.
  3. A daily seeded shuffle of recipe cards, "do you remember?" quizzes (tap to reveal, then grade yourself), Java blocks, edge-case traps and problem challenges.
  4. The learner's own notes every third post.
- **Post actions:** ♥ like, bookmark → Saved, **Recall** (a note becomes a card), **Learned** (recipes, blocks, traps), ⋮ edit/delete your own notes, and share (copies the text). Long notes clamp with **…more**.
- **My notes:** everything you posted; click a hashtag to filter.
- **Sidebar:** **Productive scroll** (posts read, recalls tried and things learned today, against a goal of 30 posts), cards due, your tags.
- **Focus reading:** double-click any post (Notes feed, Java feed or mistake journal) to read it alone, large and centred, with **← / →** or Prev/Next to move through the feed and Esc or a click outside to close. Buttons still work there.

## 11. Global behaviour

- Hash router (`#/section/id?query`). Each view clears the `#view` click and change handlers on every route change, so nothing leaks between pages. The last route is remembered.
- **Search index:** patterns, question types, recipes, problems (all tracks and the bank), Java blocks, Streams questions and traps. **/** focuses the search box.
- The **server status pill** polls `/api/health` (Java version, AI on/off).
- All user-facing text is simple English, and every piece of code uses the code-window style.

## 12. Server API

| Method and path | Purpose |
|---|---|
| `GET /api/health` | `{java: "17.0.x" or null, ai: bool, model}` |
| `POST /api/run` | `{code, inputs[]}` → compile once, run per input → `{compile:{ok, error}, results:[{stdout, stderr, timeMs, exitCode, timedOut}]}`. Limits: 100 KB of code, 40 inputs |
| `POST /api/upload` | `{dataUrl}` (PNG/JPEG/GIF/WebP, ≤ 8 MB) → saves `data/uploads/<time36>-<rand>.ext` → `{url:"/uploads/…"}` |
| `DELETE /api/upload/:name` | deletes an uploaded image |
| `POST /api/ai/chat` | Server-Sent Events stream `{text}` … `{done, stop}`. 503 if no API key |
| `POST /api/ai/generate` | generates a problem, compiles it, tests the solution, returns it |
| `GET /uploads/*`, `GET /*` | static files (directory → index.html), with path-traversal protection, `cache-control: no-cache` |

The server listens on `127.0.0.1:7070` (the `PORT` and `HOST` environment variables override it), loads `.env` if present, and on start prints the URL, the Java version and the AI status.

## 13. Quality tools (all must pass)

- `npm run verify` (`tools/verify.mjs`): every practice solution passes all its tests, and every generated starter compiles.
- `tools/check-snippets.mjs`: every Java snippet in Learn compiles, each wrapped in a harness class.
- `tools/check-streams.mjs`: runs all 46 Streams blocks on the sample data and writes their real outputs. Every Practice starter compiles, and the reference answer placed into the starter passes the check.
- `tools/check-traps.mjs`: both the breaks and the safe version of every trap compile.
- `tools/check-format.mjs`: after formatting, every solution still passes its tests, starters and Streams programs still compile, and formatting twice changes nothing.

## 14. Build order

1. Server, Java runner and the health/run/upload API. Then `index.html`, the design system (all 6 themes and the code palettes), the top bar, navbar, theme menu and text size.
2. `store.js`, `srs.js`, the router, search, side-panel handles and focus reader.
3. Content data: library (patterns, question types, recipes, recipe map, Java blocks, traps), authored lessons for all 26 patterns, and problems (warmup, patterns, service, with tests). Then the Streams bank and the service bank.
4. Coding pages: editor, console, modes, autocomplete and formatter.
5. Learn, then Java (cards, feed, traps, practice), then Review, mistakes and Notes.
6. The camp plan builder, then Today and Plan.
7. Optional AI features.
8. The check tools, then a README (sections, how to run, checks, where data is stored).

**Acceptance:**
- `npm start` opens Today with Day 1 ready.
- Every section works in all 6 themes and at 85–150% text size.
- All tools in §13 pass.
- No console errors.
- Every recall source creates a card that comes back on schedule.
- Images survive a reload and are deleted with their owner.
