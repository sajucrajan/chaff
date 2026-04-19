# Chaff Roadmap

This document describes what Chaff currently does, what it will do soon, and where it's going long-term. It is the source of truth for "what's next" — anyone (human or agent) picking up the codebase should start here.

Last updated: April 2026

---

## Current shipped features (v0.1)

### Setup screen
- Full-width 2-column layout (questions left, mode picker + sliders right)
- Sticky bottom launch bar with primary "Start session" + secondary "Demo mode" buttons
- Theme toggle (Light / Auto / Dark) in top-right chrome strip, persists via localStorage
- "Load sample questions" link populates 10 realistic interview questions
- Question count validation (8-15) with live counter
- Live decoy preview (only shown when Mode 2 or 4 selected)

### Interview screen
- Full-viewport question grid, no inner scrollbars
- Adaptive per-card font sizing based on word count (14-26px range, scaled by formula)
- Interviewer-only control panel (sticky right side, ~280px wide) with: reveal button, active-question picker, flicker controls, private-reveal toggle, reshuffle, pause
- Real/decoy markers on each card (subtle gray dots by default; flooded with R/D labels in private-reveal mode)
- Active-question highlight: gold outline + glow + smooth scroll-into-view

### Four chaff modes
- **Mode 1 — Pick N of 10:** real questions only, AI must solve all in parallel
- **Mode 2 — Shuffle & Reveal:** nonsense questions visible, words physically fly into place when you press Reveal
- **Mode 3 — Flicker periphery:** real questions + 8×12 grid of peripheral cells flickering 3-20Hz with random word fragments
- **Mode 4 — Full chaff:** Modes 2 + 3 combined

### Reveal animation
- Each word rendered as individual `<span class="q-word">` for animation targeting
- Word-matching: nonsense words match destination words by normalized text first (so "mutex" flies to "mutex"); falls back to any unused word
- 1.4-second cubic-bezier flight with gold glow during transit
- Cards fade to 15% opacity during shuffle, real text fades in on landing
- "Scramble again" button lets you replay the effect

### Demo mode
- 4-stage auto-playing walkthrough (one stage per Chaff mode)
- Floating "AI's view" panel showing simulated screenshot fragments per mode
- Fake gold cursor that points to questions and clicks the reveal button
- Playback controls (play/pause, prev/next stage, exit) and arrow-key navigation
- Stage banners with mode names and explanatory narration

### Theme system
- Three states: light, dark, auto (auto follows `prefers-color-scheme`)
- Token-based CSS — no hardcoded colors anywhere in component CSS
- Switches both setup and interview chrome simultaneously
- Persists via `localStorage.chaffTheme`

---

## In-flight / next up (v0.2)

### Mode 5 — OCR poison (per-word temporal masking)

**This is the next major feature and the highest-priority roadmap item.**

The existing four modes defeat AI in different ways but they all assume the AI can read complete questions when it captures the screen. Mode 5 attacks the OCR step itself: ensure the screen never displays a complete, readable question at any single moment in time.

#### Threat we're defeating

Real-world AI cheating tools use Tesseract.js or similar in-browser OCR libraries. OCR is CPU-expensive and rate-limited — typical capture cadence is **500ms to 3 seconds per screenshot**. The full pipeline ("Lag Loop") is **3-5 seconds end-to-end**: capture → OCR → LLM → display. Even high-end commercial tools cap below 2Hz in practice.

If we cycle word visibility faster than the OCR pipeline can capture it, we ensure no single screenshot contains all the words of any one question. The AI has to either:

1. Run OCR much faster (CPU/GPU prohibitive on candidate's machine)
2. Stitch together multiple screenshots over time (introduces 5-15 second delay before AI has a complete question — by then the candidate is already answering)
3. Use a more sophisticated computer-vision pipeline (raises the cost dramatically)

#### Mechanics

- **Granularity: per-word.** Each word in each question is a candidate for masking on each frame. (Per-character was considered and rejected — too disruptive for human readers.)
- **Mask treatment:** invisible-but-reserves-space. Hidden words become `visibility: hidden` (or `opacity: 0`), so the layout doesn't reflow. To a human, the gap reads as "a word that's momentarily not there"; the eye/brain integrates across frames and the question reads continuously. To OCR, the gap is just whitespace.
- **Refresh cadence:** target ~10Hz (every 100ms). Browsers handle this comfortably via `setInterval`. The cadence is configurable via slider. Range 5-20Hz.
- **Mask density:** 15-25% of words hidden per frame. Sweet spot — enough to reliably break OCR, low enough that humans read at normal speed.
- **Mask pattern rotation:** cycle through 4-6 distinct patterns. Random masks are too erratic and cause flickering perceptual artifacts; rotating predetermined patterns gives a smoother visual and ensures every word is visible at least 70% of frames.

#### UI integration

Two ways to enable:

1. **Mode 5 — OCR poison** as a fifth standalone mode (10 real questions + temporal masking, no decoys, no flicker)
2. **Sub-toggle "OCR poison"** in the interviewer control panel that can layer onto Modes 1-4

The sub-toggle is the more useful pattern — it lets the interviewer combine OCR poison with shuffle reveal, with peripheral flicker, etc. The standalone Mode 5 is the simplest defaults.

#### Implementation plan

```js
// New state fields
ocrPoisonEnabled: false,
ocrPoisonRate: 10,           // Hz, configurable
ocrPoisonDensity: 0.20,      // fraction of words hidden per frame
ocrPoisonMaskPatterns: [],   // computed once per render
ocrPoisonTimer: null,
ocrPoisonPatternIdx: 0,

// New functions
function buildMaskPatterns(qWords) {
  // Generate 4-6 distinct mask patterns covering all word indices
  // Each pattern hides ~density% of words; together patterns ensure
  // every word is visible in at least 70% of patterns
  const N = qWords.length;
  const hideCount = Math.round(N * state.ocrPoisonDensity);
  const PATTERN_COUNT = 5;
  const patterns = [];
  // Use a Latin-square-like distribution to ensure coverage
  for (let p = 0; p < PATTERN_COUNT; p++) {
    const hidden = new Set();
    for (let i = 0; i < hideCount; i++) {
      hidden.add((p * hideCount + i) % N);
    }
    patterns.push(hidden);
  }
  return patterns;
}

function tickOcrPoison() {
  const pattern = state.ocrPoisonMaskPatterns[state.ocrPoisonPatternIdx];
  document.querySelectorAll('.q-word').forEach((w, i) => {
    w.style.visibility = pattern.has(i) ? 'hidden' : 'visible';
  });
  state.ocrPoisonPatternIdx = (state.ocrPoisonPatternIdx + 1) %
                               state.ocrPoisonMaskPatterns.length;
}

function startOcrPoison() {
  stopOcrPoison();
  const allWords = Array.from(document.querySelectorAll('.q-word'));
  state.ocrPoisonMaskPatterns = buildMaskPatterns(allWords);
  state.ocrPoisonTimer = setInterval(tickOcrPoison, 1000 / state.ocrPoisonRate);
  tickOcrPoison(); // paint first frame immediately
}

function stopOcrPoison() {
  if (state.ocrPoisonTimer) clearInterval(state.ocrPoisonTimer);
  state.ocrPoisonTimer = null;
  document.querySelectorAll('.q-word').forEach(w => {
    w.style.visibility = '';
  });
}
```

The pattern indexing uses a simple offset to ensure word `i` is hidden in pattern `p` only when `(i - p * hideCount) mod N < hideCount`. This guarantees every word is hidden in exactly `hideCount / N` patterns and visible in the rest. Across 5 patterns at 20% density, every word is visible in 4/5 = 80% of frames.

#### Open implementation questions

- **What about the active-question highlight?** When a card is `.active`, should its words be exempt from masking so the candidate can read it cleanly? Probably yes — the interviewer's verbal pointing already commits to that one question, so masking it post-pointing only hurts the candidate.
- **Interaction with Mode 2/4 nonsense phase?** Yes, mask the nonsense too — it's even more important to break OCR while the screen shows nonsense, because that's the long-duration state.
- **Interaction with reveal animation?** Pause OCR poison during the 1.4s reveal flight — the animation already creates plenty of mid-flight chaos for OCR.
- **Visual fairness for the candidate:** at 10Hz / 20% density, words should "shimmer in and out" but read continuously. If real-world testing shows this is too distracting, we may need to drop to 5Hz or 15% density. Make these defaults user-tunable.

#### Demo mode integration

Add a 5th demo stage:
- Banner: "Mode 5 · OCR poison"
- Narration: "Words flicker in and out 10 times a second. A human reads the question normally; an OCR tool gets a different incomplete fragment every screenshot."
- AI-view panel: show simulated OCR output as a progression — t=0s: *"...mutex...sorts...semaphore..."* / t=1s: *"Implement...that...a..."* / t=2s: *"...locks...binary..."* / footer: "**OCR coverage incomplete after 5 seconds. AI cannot reconstruct full question.**"

#### Acceptance criteria

- 10Hz cycling on a typical mid-2020s laptop with no observable performance issues (CPU < 5%)
- Human readability test: 5 testers can read every question in under 3 seconds (control: same questions without masking, baseline ~1-2 seconds)
- OCR test: run Tesseract.js against screenshots taken at 1Hz over 10 seconds; verify <50% word recovery for any single question

---

### Other v0.2 items

- **Popout interviewer control panel** — open the right panel as a separate browser window, eliminates the screen-share risk of accidentally sharing the panel
- **Audio defense (research)** — currently AI assistants can transcribe interviewer audio and infer "answer 7" → highlight question 7 in their context. Possible defenses: have the interviewer point at questions via the UI rather than verbally announcing the number; or generate fake "answer 3, no wait answer 9" decoy audio cues
- **Better decoy generation** — swap heuristic POS classifier for the `compromise` library; add domain-vocabulary seeding (different word pools for crypto vs web-dev vs system design)
- **Hungarian-algorithm word matching for reveal** — currently word-matching is greedy first-match; an optimal matching would make the animation more semantically satisfying (every word flies to its true twin where possible)
- **Visual regression test suite** — Playwright-based screenshot tests for major states across themes and viewport sizes
- **Per-candidate session keys** — pre-generated encoded position strings the interviewer can paste in to randomize without seeing the question

---

## Medium-term (v0.3)

### Multi-screen / external display support
- First-class support for putting the candidate-facing grid on one monitor and the control panel on another
- Detection of multi-monitor setups, suggested config

### Adaptive difficulty
- Track which questions the candidate answers strongly vs weakly, surface follow-ups
- Note: this requires interviewer input during the session, not just observation

### Question bank management
- Save question sets to localStorage with named labels (e.g., "Senior Backend", "Frontend Lead")
- Import/export question banks as JSON
- Tag questions by topic for targeted decoy generation (better than current "all questions go in one pool")

### Internationalization
- Initially English-only. UI strings extracted to a single object for easier translation
- Decoy generator may need rework for non-English (the POS heuristics are English-specific)

### Print/export session record
- After session ends, optionally generate a printable record of which questions were asked, in what order, with notes the interviewer typed during the session
- Useful for debrief and for compliance trails

### Interview mode for whiteboard/coding
- Currently questions are text-only. For coding interviews, support a question with attached starter code or a reference image
- Decoy generation gets weirder here — code decoys are harder to make plausible

---

## Long-term / research

### Adversarial co-evolution tracking
- Maintain a public log of which AI assistants Chaff defeats and which it doesn't
- Run an internal red-team test against new Cluely / Interview Coder / Natively releases
- Publish defense effectiveness ratings — be transparent about where Chaff is and isn't holding up

### Computer-vision-resistant rendering
- OCR poison defeats text recognition. The next step in the AI-cheat arms race is full computer-vision models (CLIP-style) that can read partially-occluded text by learning context
- Counter-defenses: anti-CV adversarial perturbations on the rendered text, font-mixing within words, near-invisible UI distortions
- This is research-grade work; not for v1

### Audio-based defenses
- Generate ultrasonic interference patterns on the interviewer's audio to defeat speech-to-text on the candidate's machine (technical and legal concerns; research only)
- Work with conferencing platforms to add interviewer-side audio markers that AI assistants can't process

### Standalone INTERVU integration
- Chaff becomes an embeddable interview module inside the INTERVU parent platform
- Question banks, candidate records, scheduling all live in INTERVU; Chaff is the in-call defense layer
- API surface: `init({questions, mode, callbacks})` returning an `iframe`-mountable URL
- Single-file architecture should make this straightforward — Chaff already takes its inputs entirely from local state, just need to expose hooks for parent-platform telemetry

### Open standards
- Propose an "interviewer-side AI defense" capability on a few major proctoring/interview platforms (HackerRank, CoderPad, Karat) — pitch Chaff as the reference implementation
- Coordinate with detection-side projects (Fabric, Talview, etc.) — defense-in-depth pitch where detection + deflection complement each other

---

## Out of scope

These are decisions, not deferrals — Chaff explicitly does NOT do these things:

- **Recording the candidate.** Chaff is interviewer-side only. We don't record audio or video, we don't analyze the candidate's behavior, we don't try to "catch" them. The candidate's experience should be unaffected if they're not using AI.
- **Network telemetry.** Chaff must work fully offline. Question content never leaves the local machine.
- **Subscription / SaaS.** Chaff is and will remain free, open-source, single-file, downloadable. The product strategy is the opposite of the cheating tools' subscription model.
- **AI-generated questions.** Chaff defends questions you bring; it does not generate them. Question quality is the interviewer's job.
- **Catching past cheaters.** Chaff is real-time defense. Forensic detection is the detection-side project's job (Dayside).

---

## Versioning

- **v0.1** (April 2026) — current. Four modes, theme system, demo mode, single-file architecture
- **v0.2** — OCR poison (Mode 5), popout control panel, audio defense research
- **v0.3** — Question banks, multi-screen, internationalization
- **v1.0** — First version recommended for production interview use without disclaimers; requires real-world test results from at least 50 mock interviews and effectiveness ratings against the top 5 commercial cheating tools

Versioning follows [SemVer](https://semver.org/) once we hit v1.0.

---

## Open questions for the maintainers

If you're an agent picking up this project, here are decisions that haven't been made yet:

1. **License of decoys** — if Chaff generates decoy questions from interviewer-provided real questions, does the interviewer own the decoys? (Default: yes, since they're derivative of interviewer input. But if Chaff seeds with built-in vocabulary later, this gets murkier.)
2. **Fairness disclosure** — current default is "tell the candidate Chaff is in use." Is there a mode where this is not required? Some interviewers may want adversarial-only mode against confirmed-cheating roles. Ethical implications of removing the disclosure default need careful thought.
3. **Reporting back to the cheating-tool ecosystem** — when Chaff defeats a tool, should we report it to the tool's vendor? Doing so accelerates the arms race; not doing so means defenses become public knowledge slower. No clear right answer.
4. **Demo mode as standalone artifact** — should "Chaff demo" be its own standalone HTML file (just the demo, no setup screen) for sharing as a marketing asset? Current demo mode is reachable inside the main app; could be a 30KB shareable.
