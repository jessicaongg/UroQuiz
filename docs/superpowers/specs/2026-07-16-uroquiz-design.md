# UroQuiz — EAU Guideline MCQ Quiz Bank

**Date:** 2026-07-16
**Status:** Approved

## Purpose

A personal study tool for a urology resident preparing for exams on EAU
(European Association of Urology) guideline content. Serves multiple-choice
questions, tracks progress, and can be updated with new question batches over
time.

## Content sourcing & accuracy caveat

Questions are written by Claude from training knowledge of EAU guideline
content (training cutoff: January 2026), not sourced from a live fetch of the
current EAU guideline documents. EAU guidelines are revised annually, so some
details (exact staging/grading thresholds, specific numeric cutoffs, most
recent edition's changes) may drift from the current published edition. All
questions should be spot-checked by the user against the current EAU
guideline PDFs before being relied on for final exam prep. Each question's
explanation should make the underlying guideline concept clear enough to
verify against the source.

## Architecture

Single self-contained HTML file, hosted as a Claude Artifact (private URL,
no backend, no build step, works on any device via browser — phone and
desktop). No React/build tooling: Artifacts must be dependency-free and
self-contained (inline CSS/JS, no external requests).

Internal structure, kept modular despite being framework-free:

- **`QUESTIONS`** — a JS array/object near the top of the script, grouped by
  topic. This is the section edited to add new question batches; app logic
  is never touched to add content.
  Each question shape:
  ```js
  {
    id: "onco-prostate-014",
    topic: "oncology",          // oncology | non-oncology | infections-other
    subtopic: "prostate",       // e.g. prostate, bladder, stones, bph, uti...
    question: "...",
    options: ["...", "...", "...", "..."],
    correctIndex: 2,
    explanation: "..."
  }
  ```
- **Render functions** act as "components," each owning one screen, swapped
  via a single state variable + re-render call (no routing library):
  - `renderTopicPicker()`
  - `renderModeSelector()`
  - `renderQuestion()`
  - `renderResults()`
  - `renderStats()`
- **`localStorage`** holds cross-session state directly (no framework store):
  per-topic accuracy, a missed-question log, last-used mode. This is
  per-browser/per-device — phone and desktop track stats independently.
- CSS: inline `<style>`, theme-aware (light/dark via
  `prefers-color-scheme` + `data-theme` override per Artifact conventions),
  class names scoped simply (BEM-ish).

Updating later: new question objects are appended to `QUESTIONS` and the
Artifact is redeployed to the same URL — the bookmarked link keeps working,
no rebuild step visible to the user.

## Features & UX flow

1. **Home screen** — checkboxes to pick topic(s), or "All topics":
   - *Oncology*: prostate, bladder (NMIBC/MIBC), RCC, UTUC, testicular,
     penile
   - *Non-oncology / functional*: urolithiasis, BPH/male LUTS, incontinence,
     neuro-urology, female/functional urology
   - *Infections & other*: UTI/urosepsis/Fournier's, trauma/reconstruction,
     andrology/male infertility, paediatric urology, transplantation

   Then choose mode:
   - **Study Mode** — immediate feedback + explanation after each answer
   - **Exam Mode** — a block of N questions, no feedback until the block ends

2. **Quiz screen** — one question at a time, 4-5 options, progress indicator
   ("Q7/20"). Study mode reveals correctness + explanation on selection;
   Exam mode advances silently with no reveal.

3. **Results screen** — score, list of missed questions with explanations
   (shown here regardless of mode), and a "retry missed only" action.

4. **Stats dashboard** — reachable from the home screen: per-topic accuracy
   over time, total questions attempted, and a running missed-question list
   that can be re-quizzed specifically.

## Question bank plan

- **First build**: ~100-120 questions, roughly evenly split across the three
  domain groups above, each with plausible distractors and a 1-3 sentence
  explanation tied to the guideline concept.
- **Growth**: future sessions add batches of ~50-100 questions per topic,
  targeting an eventual total of ~300-400 questions. Lower volume than an
  initial ask of 1000, chosen deliberately to keep per-question accuracy
  higher given no automated verification against source guideline text.

## Out of scope (for this build)

- Backend/accounts/sync across devices — stats are local to each
  browser/device by design.
- Verbatim sourcing from live EAU guideline documents — content is
  generated from training knowledge, not fetched or scraped.
- Non-MCQ question types (e.g. free text, image-based questions).

## Testing / verification approach

Since this is a single-file, no-build app, verification is manual:
- Load the Artifact URL and click through: topic selection → both modes →
  results → stats dashboard, on both a desktop-sized and mobile-sized
  viewport.
- Confirm `localStorage` persistence survives a page reload (stats and
  missed-question log still present).
- Spot-check a sample of questions per topic for internal consistency
  (correct answer index matches the explanation's reasoning).
