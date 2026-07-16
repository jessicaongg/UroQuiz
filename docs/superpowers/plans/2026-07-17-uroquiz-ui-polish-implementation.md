# UroQuiz UI Polish Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an Exam-mode selection highlight, make "All Topics" mutually exclusive with individual topic checkboxes, rearrange the quiz screen's navigation buttons (Previous far-left / Next far-right / Home centered below), and rearrange the home screen's action buttons (Start Quiz vs. Resume+Restart as mutually exclusive alternatives, View Stats always centered below).

**Architecture:** All changes are localized edits to `renderHome` and `renderQuiz` in the existing single-file `index.html`, plus three new CSS rules. No new state fields, no new storage schema.

**Tech Stack:** Same as the existing app — vanilla HTML/CSS/JavaScript. No npm, no bundler.

Per the design spec (`docs/superpowers/specs/2026-07-17-uroquiz-ui-polish-design.md`), verification stays manual (visual check in a browser).

---

## File Structure

- Modify: `index.html` only

---

### Task 1: Add new CSS rules

**Files:**
- Modify: `index.html` (inside the existing `<style>` block)

- [ ] **Step 1: Add three CSS rules**

Find this existing rule:
```css
  label.option.incorrect { border-color: var(--incorrect); background: color-mix(in srgb, var(--incorrect) 15%, var(--card-bg)); }
```
Add immediately after it:
```css
  label.option.selected { border-color: var(--accent); background: color-mix(in srgb, var(--accent) 15%, var(--card-bg)); }
```

Find this existing rule:
```css
  .donut-wrapper { display: flex; justify-content: center; margin: 0.5rem 0; }
```
Add immediately after it:
```css
  .nav-row-split { display: flex; justify-content: space-between; align-items: center; }
  .center-row { display: flex; justify-content: center; margin-top: 0.5rem; }
```

- [ ] **Step 2: Manual verification**

Read the updated `<style>` block and confirm all three new rules are present with correct syntax (no missing semicolons/braces).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add CSS rules for exam selection highlight and split/centered button rows"
```

---

### Task 2: Exam mode selection highlight

**Files:**
- Modify: `index.html` (the option-rendering loop inside `renderQuiz`)

- [ ] **Step 1: Add the `.selected` class for Exam mode**

Current code inside `renderQuiz`:
```js
  q.options.forEach((optionText, idx) => {
    const label = document.createElement("label");
    label.className = "option";
    label.textContent = optionText;

    if (state.mode === "study" && alreadyAnswered) {
      if (idx === q.correctIndex) label.classList.add("correct");
      else if (idx === existingAnswer.chosenIndex) label.classList.add("incorrect");
    }

    label.addEventListener("click", () => {
      if (state.mode === "study" && alreadyAnswered) return;
      recordAnswer(q, idx);
      render();
    });
    card.appendChild(label);
  });
```

Replace the `if (state.mode === "study" && alreadyAnswered) { ... }` block with:
```js
    if (state.mode === "study" && alreadyAnswered) {
      if (idx === q.correctIndex) label.classList.add("correct");
      else if (idx === existingAnswer.chosenIndex) label.classList.add("incorrect");
    } else if (state.mode === "exam" && alreadyAnswered && idx === existingAnswer.chosenIndex) {
      label.classList.add("selected");
    }
```

(Everything else in this loop — the click handler, `card.appendChild(label)` — stays unchanged.)

- [ ] **Step 2: Manual verification**

Start a quiz in Exam mode. Click an option; confirm it shows the blue `.selected` highlight (border/background using `--accent`) and no explanation text appears (Exam mode still reveals nothing else). Navigate to another question and back (once Task 4 adds Previous/Next — if this task runs before Task 4 in the same session, verify via the current single-direction "Next"/"Finish" flow instead); confirm the previously-selected option still shows `.selected` when revisited.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Highlight the selected answer in Exam mode"
```

---

### Task 3: All Topics mutual exclusivity

**Files:**
- Modify: `index.html` (the "All Topics" checkbox inside `renderHome`)

- [ ] **Step 1: Change the All Topics checkbox to use the empty-selection fallback**

Current code:
```js
  const allTopicsRow = document.createElement("div");
  allTopicsRow.className = "topic-row";
  const allCheckbox = document.createElement("input");
  allCheckbox.type = "checkbox";
  allCheckbox.id = "topic-all";
  allCheckbox.addEventListener("change", () => {
    if (allCheckbox.checked) {
      state.selectedTopics = getTopics();
      render();
    }
  });
  const allLabel = document.createElement("label");
  allLabel.htmlFor = allCheckbox.id;
  allLabel.textContent = "All Topics";
  allTopicsRow.appendChild(allCheckbox);
  allTopicsRow.appendChild(allLabel);
  topicCard.appendChild(allTopicsRow);
```

Replace with:
```js
  const allTopicsRow = document.createElement("div");
  allTopicsRow.className = "topic-row";
  const allCheckbox = document.createElement("input");
  allCheckbox.type = "checkbox";
  allCheckbox.id = "topic-all";
  allCheckbox.checked = state.selectedTopics.length === 0;
  allCheckbox.addEventListener("change", () => {
    if (allCheckbox.checked) {
      state.selectedTopics = [];
      render();
    }
  });
  const allLabel = document.createElement("label");
  allLabel.htmlFor = allCheckbox.id;
  allLabel.textContent = "All Topics";
  allTopicsRow.appendChild(allCheckbox);
  allTopicsRow.appendChild(allLabel);
  topicCard.appendChild(allTopicsRow);
```

(Only two changes: added `allCheckbox.checked = state.selectedTopics.length === 0;`, and changed `state.selectedTopics = getTopics();` to `state.selectedTopics = [];`. The individual per-topic checkboxes below this, and `startQuiz`'s existing empty-means-all fallback, need no changes.)

- [ ] **Step 2: Manual verification**

On the Home screen, check "All Topics"; confirm all three individual topic checkboxes appear unchecked. Toggle Study/Exam mode (which re-renders Home); confirm "All Topics" remains checked across that re-render. Then check one individual topic checkbox; confirm "All Topics" becomes unchecked. Start a quiz with "All Topics" checked and confirm questions from all topics appear (unchanged filtering behavior, since `state.selectedTopics = []` already meant "all" via the pre-existing `startQuiz` fallback).

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Make All Topics checkbox mutually exclusive with individual topic selection"
```

---

### Task 4: Quiz screen button layout (Previous/Next split, Home centered below)

**Files:**
- Modify: `index.html` (the navigation section at the end of `renderQuiz`)

- [ ] **Step 1: Replace the single `navRow` with a split top row and a centered Home row**

Current code:
```js
  container.appendChild(card);

  const navRow = document.createElement("div");

  if (state.currentIndex > 0) {
    const prevBtn = document.createElement("button");
    prevBtn.className = "secondary";
    prevBtn.textContent = "Previous";
    prevBtn.addEventListener("click", () => {
      state.currentIndex--;
      render();
    });
    navRow.appendChild(prevBtn);
  }

  // Study mode requires answering before advancing (pedagogical gate, unchanged
  // from the original design); Exam mode allows free navigation at any time.
  const canAdvance = state.mode === "exam" || alreadyAnswered;
  if (canAdvance) {
    const nextBtn = document.createElement("button");
    nextBtn.style.marginLeft = "0.5rem";
    nextBtn.textContent = isLastQuestion ? "Finish" : "Next Question";
    nextBtn.addEventListener("click", () => {
      if (isLastQuestion) {
        state.screen = "results";
      } else {
        state.currentIndex++;
      }
      render();
    });
    navRow.appendChild(nextBtn);
  }

  const homeBtn = document.createElement("button");
  homeBtn.className = "secondary";
  homeBtn.textContent = "Home";
  homeBtn.style.marginLeft = "0.5rem";
  homeBtn.addEventListener("click", () => {
    saveInProgressSession();
    state.screen = "home";
    render();
  });
  navRow.appendChild(homeBtn);

  container.appendChild(navRow);
  return container;
}
```

Replace the section from `const navRow = document.createElement("div");` through `container.appendChild(navRow);` (keep `container.appendChild(card);` before it and `return container;` after it unchanged) with:

```js
  const navTopRow = document.createElement("div");
  navTopRow.className = "nav-row-split";

  if (state.currentIndex > 0) {
    const prevBtn = document.createElement("button");
    prevBtn.className = "secondary";
    prevBtn.textContent = "Previous";
    prevBtn.addEventListener("click", () => {
      state.currentIndex--;
      render();
    });
    navTopRow.appendChild(prevBtn);
  } else {
    navTopRow.appendChild(document.createElement("div"));
  }

  // Study mode requires answering before advancing (pedagogical gate, unchanged
  // from the original design); Exam mode allows free navigation at any time.
  const canAdvance = state.mode === "exam" || alreadyAnswered;
  if (canAdvance) {
    const nextBtn = document.createElement("button");
    nextBtn.textContent = isLastQuestion ? "Finish" : "Next Question";
    nextBtn.addEventListener("click", () => {
      if (isLastQuestion) {
        state.screen = "results";
      } else {
        state.currentIndex++;
      }
      render();
    });
    navTopRow.appendChild(nextBtn);
  } else {
    navTopRow.appendChild(document.createElement("div"));
  }

  container.appendChild(navTopRow);

  const homeRow = document.createElement("div");
  homeRow.className = "center-row";
  const homeBtn = document.createElement("button");
  homeBtn.className = "secondary";
  homeBtn.textContent = "Home";
  homeBtn.addEventListener("click", () => {
    saveInProgressSession();
    state.screen = "home";
    render();
  });
  homeRow.appendChild(homeBtn);
  container.appendChild(homeRow);

  return container;
}
```

Note: the empty placeholder `<div>`s (appended when Previous or Next/Finish isn't shown) keep the *other* button pinned to its side via `justify-content: space-between` on `.nav-row-split` — without a placeholder, a lone remaining flex child would sit at the start (left) rather than staying pinned to its intended side.

- [ ] **Step 2: Manual verification**

On question 1 (Previous absent), confirm "Next Question" appears pinned to the far right (not shifted left). Advance to a later question; confirm "Previous" is pinned far-left and "Next Question"/"Finish" is pinned far-right, on the same row. Confirm "Home" appears centered on its own row below that.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Split quiz nav into Previous/Next row and centered Home row"
```

---

### Task 5: Home screen action buttons (Start Quiz vs Resume+Restart, View Stats always centered below)

**Files:**
- Modify: `index.html` (`renderHome`)

- [ ] **Step 1: Remove the existing top-of-page Resume/Restart block**

Current code (right after the readiness card):
```js
  const progress = loadProgress();
  if (progress.inProgressSession) {
    const resumeBtn = document.createElement("button");
    resumeBtn.textContent = "Resume Quiz";
    resumeBtn.addEventListener("click", () => {
      const session = progress.inProgressSession;
      state.mode = session.mode;
      state.sessionQuestions = session.sessionQuestions;
      state.currentIndex = session.currentIndex;
      state.answers = session.answers;
      state.screen = "quiz";
      render();
    });
    container.appendChild(resumeBtn);

    const restartBtn = document.createElement("button");
    restartBtn.className = "secondary";
    restartBtn.textContent = "Restart";
    restartBtn.style.marginLeft = "0.5rem";
    restartBtn.addEventListener("click", () => {
      clearInProgressSession();
      render();
    });
    container.appendChild(restartBtn);
  }

  const topicCard = document.createElement("div");
```

Replace with (keep the `loadProgress()` call — it's still needed later in this task — but remove the `if (progress.inProgressSession) { ... }` button-creation block entirely):
```js
  const progress = loadProgress();

  const topicCard = document.createElement("div");
```

- [ ] **Step 2: Replace the bottom Start Quiz / View Stats block**

Current code (near the end of `renderHome`):
```js
  const startBtn = document.createElement("button");
  startBtn.textContent = "Start Quiz";
  startBtn.addEventListener("click", startQuiz);
  container.appendChild(startBtn);

  const statsBtn = document.createElement("button");
  statsBtn.className = "secondary";
  statsBtn.textContent = "View Stats";
  statsBtn.style.marginLeft = "0.5rem";
  statsBtn.addEventListener("click", () => { state.screen = "stats"; render(); });
  container.appendChild(statsBtn);

  return container;
}
```

Replace with:
```js
  const actionRow = document.createElement("div");
  if (progress.inProgressSession) {
    const resumeBtn = document.createElement("button");
    resumeBtn.textContent = "Resume Quiz";
    resumeBtn.addEventListener("click", () => {
      const session = progress.inProgressSession;
      state.mode = session.mode;
      state.sessionQuestions = session.sessionQuestions;
      state.currentIndex = session.currentIndex;
      state.answers = session.answers;
      state.screen = "quiz";
      render();
    });
    actionRow.appendChild(resumeBtn);

    const restartBtn = document.createElement("button");
    restartBtn.className = "secondary";
    restartBtn.textContent = "Restart";
    restartBtn.style.marginLeft = "0.5rem";
    restartBtn.addEventListener("click", () => {
      clearInProgressSession();
      render();
    });
    actionRow.appendChild(restartBtn);
  } else {
    const startBtn = document.createElement("button");
    startBtn.textContent = "Start Quiz";
    startBtn.addEventListener("click", startQuiz);
    actionRow.appendChild(startBtn);
  }
  container.appendChild(actionRow);

  const statsRow = document.createElement("div");
  statsRow.className = "center-row";
  const statsBtn = document.createElement("button");
  statsBtn.className = "secondary";
  statsBtn.textContent = "View Stats";
  statsBtn.addEventListener("click", () => { state.screen = "stats"; render(); });
  statsRow.appendChild(statsBtn);
  container.appendChild(statsRow);

  return container;
}
```

- [ ] **Step 3: Manual verification**

With no saved session, confirm Home shows "Start Quiz" alone (no Resume/Restart), plus "View Stats" centered on its own row below. Start a quiz, click Home mid-quiz; confirm Home now shows "Resume Quiz" + "Restart" together (no "Start Quiz" anywhere on the page), plus "View Stats" still centered below. Click "Restart"; confirm it reverts to showing "Start Quiz" alone again. Click "Resume Quiz" after starting a session and clicking Home; confirm it correctly returns to the exact quiz state.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Consolidate home screen actions: Start Quiz vs Resume+Restart, View Stats centered"
```

---

### Task 6: Full walkthrough, redeploy, and push

**Files:** none (verification + deployment step)

- [ ] **Step 1: Full manual walkthrough**

- Exam mode: click an option, confirm blue `.selected` highlight, no explanation shown; use Previous/Next to revisit and confirm the highlight persists.
- Study mode: confirm correct/incorrect highlighting is unaffected by this change.
- Home: check "All Topics", confirm individual boxes uncheck and "All Topics" stays checked across a re-render (e.g. toggle mode radios); check one individual topic, confirm "All Topics" unchecks.
- Quiz screen: confirm Previous/Next split layout and centered Home row on question 1 (no Previous) and a later question (both present).
- Home: confirm "Start Quiz" alone with no saved session, and "Resume Quiz"/"Restart" together (no "Start Quiz") with a saved session, with "View Stats" always centered below in both cases.
- Confirm nothing from earlier features (donut charts, exam readiness, missed-question review, stats dashboard) regressed.

- [ ] **Step 2: Redeploy the Claude Artifact**

Use the Artifact tool with `file_path` set to `/Users/jessica/Desktop/UroQuiz/index.html` and `url` set to the existing published Artifact URL (`https://claude.ai/code/artifact/0d869db7-55ef-4d35-a68c-fab9f5126e8a`), keeping the same favicon (🩺) and a similar description.

- [ ] **Step 3: Push to GitHub**

```bash
git push origin main
```
