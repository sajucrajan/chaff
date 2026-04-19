# Chaff

> Real questions, hidden in decoys.

A local-only interviewer portal that makes AI interview-cheating tools (Cluely, Interview Coder, Final Round AI, Natively, LockedIn AI, etc.) work harder than the candidate. Chaff doesn't try to *detect* AI assistants — it assumes they're watching and makes them solve the wrong problems.

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Status: v0.1](https://img.shields.io/badge/Status-v0.1-orange.svg)]()
[![Single file](https://img.shields.io/badge/Architecture-Single_HTML_File-green.svg)]()

---

## Why this exists

AI interview cheating tools are a $20-150/month subscription industry now. They use OCR (typically Tesseract.js) to read questions from the candidate's screen and feed answers through GPT-4 / Claude / Gemini in 3-5 seconds. They run as invisible overlays that screen-sharing software cannot capture.

Three classes of defense exist:

1. **Detection** — proctoring tools that try to identify AI usage from behavioral signals. Best-in-class detection tops out around 85% accuracy and requires installing software on the candidate's machine.
2. **Prevention** — locked-down assessment platforms that block all background processes. High friction; doesn't work for live video interviews.
3. **Deflection** — assume the AI is watching, but make it produce wrong/garbled output. **This is what Chaff does.**

Chaff sits on the interviewer's screen. The interviewer shares it via Zoom/Meet/Teams. Candidates see a grid of questions. Some are real, some are decoys built from the same vocabulary, some shimmer at the edges, words may be temporally masked. The interviewer **verbally** points to the real one. The AI watching the candidate's screen sees noise.

## How it works

Chaff currently ships four modes of increasing AI friction:

| Mode | Name | AI defense |
|------|------|------------|
| 1 | **Pick N of 10** | 10 real questions on screen. AI must solve all 10 in parallel to be ready for whichever you pick. Latency budget triples. |
| 2 | **Shuffle & Reveal** | Screen starts with 10 nonsense questions built from real vocabulary. You press Reveal — words physically fly across the screen to form the real questions. AI captures gibberish before reveal; after reveal, candidate is already answering. |
| 3 | **Flicker periphery** | 10 real questions with cells around the edges flickering 3-20Hz with random fragments. Every AI screenshot captures a different mix of unrelated words. |
| 4 | **Full chaff** | Modes 2 + 3 combined. Maximum adversarial pressure. |

A 5th mode — **OCR poison** (per-word temporal masking) — is on the roadmap. See `ROADMAP.md`.

## Quick start

Chaff is a single HTML file. No build, no server, no install.

```bash
git clone https://github.com/YOUR_ORG/chaff.git
cd chaff
open index.html        # macOS
xdg-open index.html    # Linux
start index.html       # Windows
```

Or just download `index.html` and double-click it. Works offline. No telemetry. No data leaves your machine.

## Interview workflow

1. Open `index.html` before your call starts.
2. Click **Load sample questions** to try it, or paste your own 8-15 questions.
3. Pick a mode. **Mode 2 (Shuffle & reveal)** is the recommended starting point — most theatrical, medium friction.
4. Click **Start session**.
5. Join your video call, share the Chaff browser tab.
6. **Brief the candidate verbally:** *"I'm using a tool that displays decoy questions alongside real ones. When I point to one, that's real — read silently, then answer."*
7. Use the interviewer-only control panel on the right to point to real questions.

## Showing it to your team

Click **▶ Demo mode** on the setup screen. Runs a 60-second self-playing walkthrough of all four modes with a simulated "what the AI sees" panel showing why each defense works. Designed to be shareable on a screen-share without narration.

## Project structure

```
chaff/
├── index.html              # The whole app (single file, ~100KB, ~3000 lines)
├── README.md               # This file
├── ROADMAP.md              # Detailed roadmap with the OCR-poison feature spec
├── LICENSE                 # Apache 2.0
└── (no other files yet)
```

The single-file architecture is intentional — Chaff is meant to be downloadable, runnable offline, and inspectable in 30 seconds. No npm, no bundle step, no asset pipeline.

### Inside `index.html`

The file is organized in three large blocks:

1. **`<style>` block** (lines ~10-1400) — full design system, theme tokens, all component CSS
2. **HTML body** (lines ~1400-1800) — two screens (`#screen-setup`, `#screen-interview`) plus demo mode UI elements
3. **`<script>` block** (lines ~1800-3000) — single self-contained script, no modules, no globals beyond the `state` object

Section markers in the code use `// ═══...═══` headers so you can `grep` for landmarks:

```bash
grep -n "═══" index.html        # all section boundaries
grep -n "function " index.html  # all function declarations
```

### Design system

- **Tactical dark palette** by default (distinct from the warm-stone palette used by the sibling Dayside project) — deep blacks, muted gold accent (`#c9b284`), forest green for "real" markers (`#4a9160`), rust for "decoy" markers (`#8b4543`)
- **Light theme** with warm stone off-white background, deeper gold accent (`#8a6e2e`)
- **Auto theme** follows `prefers-color-scheme`
- Theme persists via `localStorage.chaffTheme`
- **Typography:** Fraunces (display serif), Inter Tight (body), JetBrains Mono (technical labels)

All colors live in CSS custom properties under `:root` / `[data-theme="light"]` / `[data-theme="dark"]` / `[data-theme="auto"]`. **Never hardcode hex values in component CSS** — use the tokens.

### State

Single `state` object near the top of the script. No external state management library. Mutations are synchronous; render functions read from `state` and write to the DOM.

```js
const state = {
  realQuestions: [],          // user's pasted questions
  decoys: [],                 // generated nonsense (mode 2/4 only)
  displayOrder: [],           // shuffled array of slot objects
  mode: 1,                    // 1-4
  flickerRate: 8,             // Hz
  decoyCount: 10,
  activeQuestionIndex: null,  // currently highlighted slot
  flickerEnabled: false,
  cheatMode: false,           // interviewer-only R/D label flood
  flickerTimer: null,
  sessionKey: null,           // human-readable real-slot positions
  shufflePhase: 'real',       // 'nonsense' | 'revealing' | 'real'
  nonsenseQuestions: [],      // displayed during 'nonsense' phase
};
```

### Key functions

| Function | What it does |
|----------|-------------|
| `parseQuestions()` | Reads the textarea, strips numbering, returns array of strings |
| `generateDecoys(reals, count)` | Grammar-preserving decoy generator. Tokenizes all real questions, pools tokens by POS class (noun/verb/func/num via heuristic sets), then samples from pools following original sentence templates. |
| `startSession()` | Builds `state.displayOrder`, switches to interview screen, kicks off flicker if needed |
| `renderInterview()` | Re-renders the question grid based on current `state`. Computes adaptive font-size per card from word count. Rebuilds active-picker and reveal-button visibility. |
| `renderWords(text, slotIdx)` | Wraps each word in `<span class="q-word" data-slot data-wordIdx>` so reveal animation can target individual words |
| `performReveal()` | The visual centerpiece. Captures every nonsense word's screen position, switches grid to real text, clones nonsense words as `position: fixed` flying elements, animates left/top to destination positions over 1.4s. |
| `rescramble()` | Regenerates nonsense and switches back to nonsense phase. Lets you replay the reveal effect. |
| `setActive(index)` | Highlights a question (gold border) and scrolls it into view |
| `startFlicker()` / `tickFlicker()` | Manages the peripheral flicker grid (8×12 cells, central reading zone hidden via JS-computed `.hidden` class) |
| `applyTheme(theme)` | Switches `<body data-theme>`, updates active button states, persists to localStorage |
| `startDemo()` / `playStage(i)` / `scheduleStageActions(stage)` | The 4-stage demo mode with simulated cursor movements, stage transitions, and AI-view panel updates |

### Decoy generator (`generateDecoys`)

Worth understanding because it's the engine for both Mode 2's nonsense questions and the eventual OCR-poison feature.

**Strategy:** preserve grammar structure of real questions but substitute content words. Result is grammatical-looking but semantically meaningless, e.g. *"Implement a mutex that sorts a URL shortener"* — reads like a real question, isn't.

**Steps:**

1. Tokenize each real question, classify each token as `noun` / `verb` / `func` / `num` via heuristic sets (`COMMON_FUNC_WORDS`, `COMMON_VERBS` in the script). Anything not in those sets defaults to `noun`.
2. Build pools per class from all real questions combined.
3. To generate a decoy: pick a random real question as a template. Walk its tokens; for `func` words, keep them (grammatical scaffolding); for content words, sample from the matching pool.
4. Capitalize and apply punctuation to match original positions.

This is good enough for v0.1 but is the most-likely-to-be-replaced subsystem. See ROADMAP.md for upgrade plans (better POS tagger like `compromise` library, domain-vocabulary seeding, Hungarian-algorithm word matching for the reveal animation).

### Theme system

Three states: `light`, `dark`, `auto`. Driven by `data-theme` attribute on `<body>`.

```css
:root, [data-theme="dark"] { /* dark tokens */ }
[data-theme="light"]       { /* light tokens */ }
@media (prefers-color-scheme: light) {
  [data-theme="auto"]      { /* light tokens */ }
}
```

Theme toggle pill (Light/Auto/Dark) lives in the chrome strip on both screens. Selection persists in `localStorage.chaffTheme`. JS in `applyTheme()` reads on boot and writes on toggle.

## Development

### Running locally

Just open `index.html` in a browser. No tooling needed.

```bash
# Quick reload during dev
python3 -m http.server 8000
# Then visit http://localhost:8000
```

### Validating changes

The whole project has no build, but two checks are worth running before any commit:

```bash
# Check JS parses (extract <script> contents and node-check)
python3 -c "import re; \
  c = open('index.html').read(); \
  s = re.search(r'<script>(.*?)</script>', c, re.DOTALL).group(1); \
  open('/tmp/chaff.js', 'w').write(s)" && node --check /tmp/chaff.js

# Check CSS brace balance (catches the duplicate-selector bug that bit us in v0.1 dev)
python3 -c "import re; \
  c = open('index.html').read(); \
  s = re.search(r'<style>(.*?)</style>', c, re.DOTALL).group(1); \
  print('balanced:', s.count('{') == s.count('}'))"
```

### Visual regression

Use Playwright to screenshot all major states and inspect manually. Sample script in `dev/snap.py` (TODO — not yet committed).

### Anti-patterns to avoid

- **No external dependencies** unless absolutely required. The single-file constraint is load-bearing for trust (people inspect the source) and portability.
- **No telemetry, no analytics, no network calls.** Chaff must work fully offline. The threat model includes "candidate's company watches network traffic from interviewer's machine."
- **No persistence of question content.** Questions live only in browser memory. `localStorage` is used only for theme preference. This is intentional — interviewer's question banks should never be written to disk by us.
- **Don't break the single-file architecture.** If a feature would require splitting files, write it as a self-contained section inside `index.html` first; we'll consider modularization later.

## Threat model

**What Chaff defends against:**

- Real-time AI-assistance overlays running on the candidate's machine (Cluely, Interview Coder, Final Round AI, Natively, LockedIn AI, and similar)
- Tools that screenshot and OCR the candidate's screen at intervals of 500ms–3s
- Tools that maintain a `~3-5 second "lag loop"` (capture → OCR → LLM → display)
- AI assistants that filter by topic keywords or expect a single visible question

**What Chaff does NOT defend against:**

- A candidate photographing your shared screen with their phone and querying an LLM separately
- Pre-leaked question banks (a separate problem)
- Audio-based attacks (AI listening to your spoken instructions and inferring "answer 7" — currently a real gap)
- Sophisticated AIs that can wait, accumulate multiple screenshots, and reconstruct text across time (much harder against OCR-poison mode)

Chaff is one layer in a defense-in-depth strategy. Pair it with proper interviewing technique (follow-up questions, asking the candidate to walk through *why* not just *what*).

## Sibling projects

- **Dayside** — Detection-side defense. Desktop tool that scans the candidate's machine for AI-assistant processes during interviews. Not yet public; in design-finalized state. Chaff and Dayside are designed to be used together.
- **INTERVU** — Parent platform (in development). Long-term, Chaff will be embedded as an interview module inside INTERVU. The single-file architecture and clean state model are designed to make this future integration straightforward.

## Contributing

This is v0.1. The codebase is small enough that any contribution is essentially "edit `index.html`, validate, PR." Specific things we'd love help with:

- **Real-world testing in mock interviews** — the highest-value contribution right now is feedback from actual mock interviews. What confused candidates? What broke under screen-share compression?
- **Better decoy generation** — see `generateDecoys()` notes above
- **OCR-poison mode** (Mode 5) — this is the next major feature; full spec in ROADMAP.md
- **Audio defense** — currently a gap (see threat model)
- **Detection of in-the-wild attacks** — if you encounter an AI assistant that defeats Chaff, we want to know

Open an issue first for anything bigger than a small fix.

## License

Apache 2.0. See `LICENSE`.

## Status

**v0.1 — early prototype, working but not polished.** Single-file HTML, tested in Chrome and Safari. Not yet tested at scale in production interviews. Use with awareness that the codebase will change as we learn from real-world usage.
