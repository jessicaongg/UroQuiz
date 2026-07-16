# UroQuiz Score Donut Charts & Exam Readiness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a reusable correct-vs-wrong donut chart (inline SVG) used on both the Results screen (per session) and Stats screen (all-time), plus an "Exam Readiness" percentage on the Home screen.

**Architecture:** One new function `renderScoreDonut(correct, total)` plus one new CSS rule, added to the existing single-file `index.html`. Three call sites: `renderResults()`, `renderStats()`, `renderHome()`. No new files, no libraries — inline SVG using the existing `--correct`/`--incorrect` CSS variables.

**Tech Stack:** Same as the existing app — vanilla HTML/CSS/JavaScript, inline SVG. No npm, no bundler, no charting library.

Per the design spec (`docs/superpowers/specs/2026-07-17-uroquiz-charts-design.md`), verification stays manual (visual check in a browser).

---

## File Structure

- Modify: `index.html` only

---

### Task 1: Add `renderScoreDonut` helper and CSS

**Files:**
- Modify: `index.html` (add a CSS rule inside the existing `<style>` block; add a new function after `getTopics`, currently at lines 1333-1335)

- [ ] **Step 1: Add a `.donut-wrapper` CSS rule**

Find this existing CSS rule:
```css
  .topic-row { display: flex; align-items: center; gap: 0.5rem; padding: 0.3rem 0; }
```

Add immediately after it:
```css
  .donut-wrapper { display: flex; justify-content: center; margin: 0.5rem 0; }
```

- [ ] **Step 2: Add `renderScoreDonut` function**

Add immediately after the existing `getTopics` function:
```js
function getTopics() {
  return [...new Set(QUESTIONS.map(q => q.topic))];
}

function renderScoreDonut(correct, total) {
  const wrapper = document.createElement("div");
  wrapper.className = "donut-wrapper";
  if (total === 0) return wrapper;

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
(The existing `getTopics` function stays exactly as-is above this addition — only add the new function after it.)

- [ ] **Step 3: Manual verification**

In a browser console after loading the page:
```js
document.body.appendChild(renderScoreDonut(7, 10)); // expect a donut showing "70%" with a green arc covering 70% and red covering 30%
document.body.appendChild(renderScoreDonut(0, 0)); // expect an empty wrapper (no SVG), no errors thrown
```
Remove these test elements afterward (refresh the page) since this is just a manual check, not a permanent change.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add reusable score donut chart helper and CSS"
```

---

### Task 2: Add donut chart to Results screen

**Files:**
- Modify: `index.html` (the `renderResults` function)

- [ ] **Step 1: Append the donut chart after the score heading**

Current code:
```js
  const summary = document.createElement("div");
  summary.className = "card";
  const scoreText = document.createElement("h2");
  scoreText.textContent = `Score: ${correctCount} / ${total}`;
  summary.appendChild(scoreText);
  container.appendChild(summary);
```

Replace with:
```js
  const summary = document.createElement("div");
  summary.className = "card";
  const scoreText = document.createElement("h2");
  scoreText.textContent = `Score: ${correctCount} / ${total}`;
  summary.appendChild(scoreText);
  summary.appendChild(renderScoreDonut(correctCount, total));
  container.appendChild(summary);
```

- [ ] **Step 2: Manual verification**

Complete a quiz with a mix of correct/incorrect/unanswered questions (e.g. 3 correct, 2 incorrect out of 5). On the Results screen, confirm a donut appears below "Score: 3 / 5" showing a 60% green arc, and the center label reads "60%".

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Show score donut chart on the results screen"
```

---

### Task 3: Add all-time donut chart to Stats screen

**Files:**
- Modify: `index.html` (the `renderStats` function)

- [ ] **Step 1: Compute all-time totals and render the donut before the per-topic breakdown**

Current code (start of `renderStats`):
```js
function renderStats() {
  const container = document.createElement("div");
  const title = document.createElement("h1");
  title.textContent = "Stats";
  container.appendChild(title);

  const progress = loadProgress();

  const topics = Object.keys(progress.topicStats);
  if (!topics.length) {
    const empty = document.createElement("p");
    empty.textContent = "No quiz attempts recorded yet.";
    container.appendChild(empty);
  } else {
    topics.forEach(topic => {
```

Replace the `const topics = Object.keys(progress.topicStats);` line and the `if (!topics.length) { ... } else {` block's opening with the following (this inserts the all-time donut inside the existing `else` branch, right before the per-topic `forEach`):

```js
  const topics = Object.keys(progress.topicStats);
  if (!topics.length) {
    const empty = document.createElement("p");
    empty.textContent = "No quiz attempts recorded yet.";
    container.appendChild(empty);
  } else {
    const allTimeAttempted = Object.values(progress.topicStats).reduce((sum, s) => sum + s.attempted, 0);
    const allTimeCorrect = Object.values(progress.topicStats).reduce((sum, s) => sum + s.correct, 0);
    container.appendChild(renderScoreDonut(allTimeCorrect, allTimeAttempted));

    topics.forEach(topic => {
```

(Everything after this — the existing `forEach` body building per-topic cards — stays exactly as-is; only the lines shown above change, and only by inserting the two `const` lines and the `renderScoreDonut` call right after the `else {` and right before `topics.forEach(topic => {`.)

- [ ] **Step 2: Manual verification**

With prior quiz history (from Task 2's verification or earlier), navigate Home → View Stats. Confirm a donut appears above the per-topic breakdown, and its center percentage matches `Math.round(allTimeCorrect/allTimeAttempted*100)` computed from the per-topic numbers shown below it (sum the correct/attempted across all topic cards to check by hand). On a completely fresh browser profile (clear localStorage or use a private window), confirm Stats shows only "No quiz attempts recorded yet." with no donut.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Show all-time score donut chart on the stats screen"
```

---

### Task 4: Add Exam Readiness percentage to Home screen

**Files:**
- Modify: `index.html` (the `renderHome` function)

- [ ] **Step 1: Compute and display readiness near the top of `renderHome`**

Current code (start of `renderHome`):
```js
function renderHome() {
  const container = document.createElement("div");

  const title = document.createElement("h1");
  title.textContent = "UroQuiz";
  container.appendChild(title);

  const progress = loadProgress();
  if (progress.inProgressSession) {
```

Insert this block between `container.appendChild(title);` and `const progress = loadProgress();` (so readiness is computed and displayed before the existing Resume/Restart check):

```js
  const readinessProgress = loadProgress();
  const readinessAttempted = Object.values(readinessProgress.topicStats).reduce((sum, s) => sum + s.attempted, 0);
  const readinessCorrect = Object.values(readinessProgress.topicStats).reduce((sum, s) => sum + s.correct, 0);
  const readinessCard = document.createElement("div");
  readinessCard.className = "card";
  if (readinessAttempted > 0) {
    const pct = Math.round((readinessCorrect / readinessAttempted) * 100);
    readinessCard.textContent = `Exam Readiness: ${pct}%`;
  } else {
    readinessCard.textContent = "Exam Readiness: Not enough data yet";
  }
  container.appendChild(readinessCard);

```

So the full sequence becomes: `container.appendChild(title);`, then the new readiness block above, then the existing `const progress = loadProgress();` / Resume-Restart block unchanged.

(Note: `renderHome` now calls `loadProgress()` twice — once as `readinessProgress` for the readiness card, once as the existing `progress` for the Resume/Restart check. This is an intentional, minimal choice per the design spec — not worth merging into one call for this change.)

- [ ] **Step 2: Manual verification**

On a fresh browser profile (no prior localStorage), load the Home screen and confirm it shows "Exam Readiness: Not enough data yet". Complete at least one quiz, return Home, and confirm it now shows "Exam Readiness: {pct}%" matching the same percentage shown on the Stats screen's all-time donut (from Task 3).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add Exam Readiness percentage to home screen"
```

---

### Task 5: Full walkthrough, redeploy, and push

**Files:** none (verification + deployment step)

- [ ] **Step 1: Full manual walkthrough**

- Fresh profile: Home shows "Exam Readiness: Not enough data yet"; Stats shows "No quiz attempts recorded yet." with no donut.
- Complete a quiz with a mix of correct/incorrect answers (Study or Exam mode): Results screen shows the score donut matching the score text.
- Return Home: readiness percentage now shown, matching Stats' all-time donut percentage.
- Toggle light/dark mode (or check both via system settings / `data-theme` if testable) and confirm donut colors switch via the existing CSS variables (green/red stay visually distinct in both themes).
- Complete a second quiz in a different topic; confirm Stats' all-time donut and Home's readiness percentage update to reflect the combined total across both sessions.

- [ ] **Step 2: Redeploy the Claude Artifact**

Use the Artifact tool with `file_path` set to `/Users/jessica/Desktop/UroQuiz/index.html` and `url` set to the existing published Artifact URL (`https://claude.ai/code/artifact/0d869db7-55ef-4d35-a68c-fab9f5126e8a`), keeping the same favicon (🩺) and a similar description.

- [ ] **Step 3: Push to GitHub**

```bash
git push origin main
```
