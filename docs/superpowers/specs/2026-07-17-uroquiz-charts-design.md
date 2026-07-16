# UroQuiz — Score Donut Chart & Exam Readiness Design

**Date:** 2026-07-17
**Status:** Approved

## Purpose

Extend UroQuiz (see prior specs in this directory) with:

1. A correct-vs-wrong donut chart on the Results screen (per quiz session).
2. A correct-vs-wrong donut chart on the Stats screen (all-time cumulative).
3. An "Exam Readiness" percentage on the Home screen (all-time overall accuracy).

## Design decisions

- **Chart type:** a two-slice donut chart (not a chart library — inline SVG,
  consistent with the single-file, dependency-free Artifact constraint).
- **Color:** reuses the app's existing `--correct`/`--incorrect` CSS variables
  (already used for option highlighting in the quiz screen), rather than
  introducing a new palette. This keeps "correct = green, incorrect = red"
  meaning consistent everywhere in the app.
- **Readiness metric:** overall accuracy only — `sum(correct) / sum(attempted)`
  across every topic in `topicStats`, expressed as a percentage. Chosen for
  simplicity over coverage-weighted or weakest-topic alternatives.
- **No-data state:** before any question has ever been answered
  (`topicStats` empty / all attempted counts are 0), the Home screen shows
  "Not enough data yet" instead of a percentage, and the Stats screen donut is
  simply omitted (the existing "No quiz attempts recorded yet." message
  already covers this case).

## Architecture

### Reusable donut chart function

A single helper builds the donut SVG, used by both the Results screen (per
session) and Stats screen (all-time):

```js
function renderScoreDonut(correct, total) {
  const wrapper = document.createElement("div");
  wrapper.className = "donut-wrapper";
  if (total === 0) return wrapper; // caller is responsible for not invoking this with total 0

  const pct = Math.round((correct / total) * 100);
  const radius = 40;
  const circumference = 2 * Math.PI * radius;
  const correctLength = (correct / total) * circumference;

  const svg = document.createElementNS("http://www.w3.org/2000/svg", "svg");
  svg.setAttribute("viewBox", "0 0 100 100");
  svg.setAttribute("width", "120");
  svg.setAttribute("height", "120");

  const bg = document.createElementNS("http://www.w3.org/2000/svg", "circle");
  bg.setAttribute("cx", "50"); bg.setAttribute("cy", "50"); bg.setAttribute("r", String(radius));
  bg.setAttribute("fill", "none");
  bg.setAttribute("stroke", "var(--incorrect)");
  bg.setAttribute("stroke-width", "16");

  const fg = document.createElementNS("http://www.w3.org/2000/svg", "circle");
  fg.setAttribute("cx", "50"); fg.setAttribute("cy", "50"); fg.setAttribute("r", String(radius));
  fg.setAttribute("fill", "none");
  fg.setAttribute("stroke", "var(--correct)");
  fg.setAttribute("stroke-width", "16");
  fg.setAttribute("stroke-dasharray", `${correctLength} ${circumference - correctLength}`);
  fg.setAttribute("stroke-dashoffset", String(circumference / 4));
  fg.setAttribute("transform", "rotate(-90 50 50)");

  const label = document.createElementNS("http://www.w3.org/2000/svg", "text");
  label.setAttribute("x", "50"); label.setAttribute("y", "55");
  label.setAttribute("text-anchor", "middle");
  label.setAttribute("font-size", "20");
  label.setAttribute("fill", "var(--fg)");
  label.textContent = `${pct}%`;

  svg.appendChild(bg);
  svg.appendChild(fg);
  svg.appendChild(label);
  wrapper.appendChild(svg);
  return wrapper;
}
```

Notes:
- `stroke-dashoffset` of a quarter-circumference rotates the start point to
  12 o'clock (combined with the `rotate(-90 ...)` transform for consistent
  starting orientation across browsers).
- Single series pair (correct/incorrect) with the percentage as a direct
  label in the center — per the dataviz guidance, a two-slice status
  chart needs no separate legend box since both meanings are already named
  by the existing "Score: X/Y" text beside it (Results) or "X/Y correct"
  text beside it (Stats).
- `total === 0` guard: caller must not invoke this when there's nothing to
  show (Results always has `total > 0` since a quiz always has ≥1 question;
  Stats must check before calling, per the no-data state above).

### Results screen integration

`renderResults()` already computes `correctCount` and `total`. Immediately
after the existing "Score: X / Y" heading, append:
```js
summary.appendChild(renderScoreDonut(correctCount, total));
```

### Stats screen integration

`renderStats()` already loads `progress.topicStats`. Add a computation of
all-time totals:
```js
const allTimeAttempted = Object.values(progress.topicStats).reduce((sum, s) => sum + s.attempted, 0);
const allTimeCorrect = Object.values(progress.topicStats).reduce((sum, s) => sum + s.correct, 0);
```
If `allTimeAttempted > 0`, render the donut (via `renderScoreDonut(allTimeCorrect, allTimeAttempted)`) near the top of the Stats screen, above the per-topic breakdown. If `allTimeAttempted === 0`, the existing "No quiz attempts recorded yet." message already handles this — no donut is rendered.

### Home screen integration

Add an "Exam Readiness" line near the top of `renderHome()` (below the
title, above the Resume/Restart block from the prior navigation feature).
Computed the same way as the Stats all-time totals:
```js
const readinessProgress = loadProgress();
const readinessAttempted = Object.values(readinessProgress.topicStats).reduce((sum, s) => sum + s.attempted, 0);
const readinessCorrect = Object.values(readinessProgress.topicStats).reduce((sum, s) => sum + s.correct, 0);
```
If `readinessAttempted > 0`: display `Exam Readiness: {pct}%` as a text
line/card (no chart — a single headline number, per dataviz guidance that a
lone number doesn't need a chart form).
If `readinessAttempted === 0`: display `Exam Readiness: Not enough data yet`.

Note: `renderHome()` will now call `loadProgress()` twice per render (once
for the existing Resume/Restart check, once for readiness) — this is
acceptable (cheap localStorage read, not a hot loop) rather than
restructuring the existing Resume/Restart code to share the call, keeping
this change minimal and independent of the navigation feature's code.

## Out of scope

- Per-topic donut charts (only one aggregate correct/wrong split per screen).
- Coverage-weighted or weakest-topic readiness formulas (explicitly deferred
  per the "overall accuracy only" decision above).
- Historical trend charts (accuracy over time) — only current cumulative
  state is shown.
- A colorblind-safe palette re-validation — this reuses the app's existing
  `--correct`/`--incorrect` tokens rather than introducing a new palette.

## Testing / verification approach

Manual, consistent with prior specs:
- Complete a quiz with a mix of correct/incorrect answers; confirm the
  Results screen donut visually reflects the split and its center percentage
  matches `Math.round(correctCount/total*100)`.
- View Stats with prior quiz history; confirm the all-time donut matches the
  sum of all topics' correct/attempted, and the percentage matches what
  Home's "Exam Readiness" shows.
- Clear all `localStorage` (fresh browser profile) and confirm Home shows
  "Not enough data yet" and Stats shows no donut (just the existing
  "No quiz attempts recorded yet." message).
- Toggle light/dark mode (or `data-theme`) and confirm the donut's colors
  switch correctly via the existing CSS variables.
