# UroQuiz Navigation, Shuffling & Resume Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-question option shuffling, Previous/Next/Finish navigation (decoupled from answering), a Home button that saves an in-progress session, a Resume/Restart flow on the home screen, and an "All Topics" checkbox — all within the existing single-file `index.html`.

**Architecture:** All changes are localized edits to existing functions in `index.html` (no new files, no build tooling, no framework — same constraints as the original app). New: `shuffleQuestionOptions()`, `saveInProgressSession()`, `clearInProgressSession()`. Modified: `loadProgress()`, `startQuiz()`, `recordAnswer()`, `renderQuiz()`, `renderResults()`, `renderHome()`, plus the "Retry Missed Only" and "Quiz Missed Questions" button handlers inside `renderResults()`/`renderStats()`.

**Tech Stack:** Same as the existing app — vanilla HTML/CSS/JavaScript, `localStorage`. No npm, no bundler.

Per both design specs (`docs/superpowers/specs/2026-07-16-uroquiz-design.md`,
`docs/superpowers/specs/2026-07-17-uroquiz-navigation-design.md`), verification
stays manual (click-through in a browser) since this remains a single static
file with no test framework.

---

## File Structure

- Modify: `index.html` only (all changes are within the existing `<script>` block)

---

### Task 1: Storage schema for in-progress session

**Files:**
- Modify: `index.html` (the `loadProgress` function, currently at line 68, and the area after `saveProgress`, currently at line 81-83)

- [ ] **Step 1: Replace `loadProgress` to normalize a new `inProgressSession` field**

Current code (lines 68-79):
```js
function loadProgress() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return { topicStats: {}, missedIds: [] };
    const data = JSON.parse(raw);
    if (!data.topicStats) data.topicStats = {};
    if (!data.missedIds) data.missedIds = [];
    return data;
  } catch (e) {
    return { topicStats: {}, missedIds: [] };
  }
}
```

Replace with:
```js
function loadProgress() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return { topicStats: {}, missedIds: [], inProgressSession: null };
    const data = JSON.parse(raw);
    if (!data.topicStats) data.topicStats = {};
    if (!data.missedIds) data.missedIds = [];
    if (data.inProgressSession === undefined) data.inProgressSession = null;
    return data;
  } catch (e) {
    return { topicStats: {}, missedIds: [], inProgressSession: null };
  }
}
```

- [ ] **Step 2: Add `saveInProgressSession` and `clearInProgressSession` functions**

Add immediately after the existing `saveProgress` function (lines 81-83):
```js
function saveInProgressSession() {
  const progress = loadProgress();
  progress.inProgressSession = {
    mode: state.mode,
    sessionQuestions: state.sessionQuestions,
    currentIndex: state.currentIndex,
    answers: state.answers
  };
  saveProgress(progress);
}

function clearInProgressSession() {
  const progress = loadProgress();
  progress.inProgressSession = null;
  saveProgress(progress);
}
```

- [ ] **Step 3: Manual verification**

Open `index.html` in a browser console and run:
```js
loadProgress() // expect { topicStats: {}, missedIds: [], inProgressSession: null } on a fresh browser profile (or existing data with inProgressSession: null appended)
saveInProgressSession() // should not throw even before state.sessionQuestions is populated
loadProgress().inProgressSession // should now be a { mode, sessionQuestions, currentIndex, answers } object
clearInProgressSession()
loadProgress().inProgressSession // should be null again
```

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add localStorage schema and helpers for in-progress quiz sessions"
```

---

### Task 2: Per-question option shuffling

**Files:**
- Modify: `index.html` (add a helper near `shuffle`, currently at line 1391; update `startQuiz`, currently at line 1400; update the "Retry Missed Only" handler inside `renderResults`, currently around line 1539-1549; update the "Quiz Missed Questions" handler inside `renderStats`, currently around line 1593-1601)

- [ ] **Step 1: Add `shuffleQuestionOptions` helper**

Add immediately after the existing `shuffle` function (lines 1391-1398):
```js
function shuffleQuestionOptions(question) {
  const indices = shuffle(question.options.map((_, i) => i));
  return {
    ...question,
    options: indices.map(i => question.options[i]),
    correctIndex: indices.indexOf(question.correctIndex)
  };
}
```

- [ ] **Step 2: Apply it in `startQuiz`**

Current code (lines 1400-1408):
```js
function startQuiz() {
  const topics = state.selectedTopics.length ? state.selectedTopics : getTopics();
  const pool = QUESTIONS.filter(q => topics.includes(q.topic));
  state.sessionQuestions = shuffle(pool);
  state.currentIndex = 0;
  state.answers = [];
  state.screen = "quiz";
  render();
}
```

Replace the `state.sessionQuestions = shuffle(pool);` line with:
```js
  state.sessionQuestions = shuffle(pool.map(shuffleQuestionOptions));
```

- [ ] **Step 3: Apply it in the "Retry Missed Only" handler (inside `renderResults`)**

Current code:
```js
    retryBtn.addEventListener("click", () => {
      state.sessionQuestions = shuffle(
        missed
          .map(a => state.sessionQuestions.find(q => q.id === a.questionId))
          .filter(Boolean)
      );
      state.currentIndex = 0;
      state.answers = [];
      state.screen = "quiz";
      render();
    });
```

Replace with:
```js
    retryBtn.addEventListener("click", () => {
      state.sessionQuestions = shuffle(
        missed
          .map(a => state.sessionQuestions.find(q => q.id === a.questionId))
          .filter(Boolean)
          .map(shuffleQuestionOptions)
      );
      state.currentIndex = 0;
      state.answers = [];
      state.screen = "quiz";
      render();
    });
```

- [ ] **Step 4: Apply it in the "Quiz Missed Questions" handler (inside `renderStats`)**

Current code:
```js
    retryBtn.addEventListener("click", () => {
      const missedQuestions = QUESTIONS.filter(q => progress.missedIds.includes(q.id));
      state.sessionQuestions = shuffle(missedQuestions);
      state.currentIndex = 0;
      state.answers = [];
      state.screen = "quiz";
      render();
    });
```

Replace with:
```js
    retryBtn.addEventListener("click", () => {
      const missedQuestions = QUESTIONS.filter(q => progress.missedIds.includes(q.id));
      state.sessionQuestions = shuffle(missedQuestions.map(shuffleQuestionOptions));
      state.currentIndex = 0;
      state.answers = [];
      state.screen = "quiz";
      render();
    });
```

- [ ] **Step 5: Manual verification**

In the browser console:
```js
const q = QUESTIONS[0];
const a = shuffleQuestionOptions(q);
const b = shuffleQuestionOptions(q);
a.options[a.correctIndex] === q.options[q.correctIndex] // expect true — the correct answer TEXT is preserved
a.options.length === q.options.length // expect true
```
Also manually: start a quiz with the same topic selection twice in a row; confirm at least one question's option order visibly differs between the two runs (acceptable to be probabilistically true — re-run a few times if needed).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Add per-session answer-option shuffling"
```

---

### Task 3: recordAnswer upsert (remove auto-advance)

**Files:**
- Modify: `index.html` (the `recordAnswer` function, currently at line 1472-1480)

- [ ] **Step 1: Replace `recordAnswer`**

Current code:
```js
function recordAnswer(question, chosenIndex) {
  const correct = chosenIndex === question.correctIndex;
  state.answers.push({ questionId: question.id, chosenIndex, correct });

  // Exam mode has no review-then-advance step, so advance the index here immediately.
  if (state.mode === "exam") {
    state.currentIndex++;
  }
}
```

Replace with:
```js
function recordAnswer(question, chosenIndex) {
  const correct = chosenIndex === question.correctIndex;
  const existing = state.answers.find(a => a.questionId === question.id);
  if (existing) {
    existing.chosenIndex = chosenIndex;
    existing.correct = correct;
  } else {
    state.answers.push({ questionId: question.id, chosenIndex, correct });
  }
}
```

Note: navigation (advancing `state.currentIndex`) moves entirely to explicit Previous/Next/Finish button handlers, implemented in Task 4. Do not implement Task 4's `renderQuiz` changes as part of this task — this task only changes `recordAnswer` itself. (The app will be in a temporarily inconsistent state — Exam mode will stop auto-advancing until Task 4 lands — but each task must still leave the file in valid, running JavaScript. This is acceptable since these two tasks land back-to-back in the same execution session.)

- [ ] **Step 2: Manual verification**

In the browser console (after loading a fresh page):
```js
state.sessionQuestions = QUESTIONS.slice(0, 2);
state.answers = [];
recordAnswer(state.sessionQuestions[0], 1);
state.answers.length // expect 1
recordAnswer(state.sessionQuestions[0], 2); // re-answer the same question
state.answers.length // expect 1 (still — upserted, not duplicated)
state.answers[0].chosenIndex // expect 2 (updated to the latest choice)
```

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Change recordAnswer to upsert instead of push-and-advance"
```

---

### Task 4: Previous/Next/Finish/Home navigation in renderQuiz

**Files:**
- Modify: `index.html` (the `renderQuiz` function, currently at line 1409-1470)

- [ ] **Step 1: Replace `renderQuiz`**

Current code:
```js
function renderQuiz() {
  const container = document.createElement("div");
  const q = state.sessionQuestions[state.currentIndex];

  if (!q) {
    state.screen = "results";
    render();
    return container;
  }

  const progress = document.createElement("div");
  progress.className = "progress";
  progress.textContent = `Question ${state.currentIndex + 1} of ${state.sessionQuestions.length} — ${q.subtopic}`;
  container.appendChild(progress);

  const card = document.createElement("div");
  card.className = "card";
  const questionText = document.createElement("p");
  questionText.textContent = q.question;
  card.appendChild(questionText);

  const existingAnswer = state.answers.find(a => a.questionId === q.id);
  const alreadyAnswered = !!existingAnswer;

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

  if (state.mode === "study" && alreadyAnswered) {
    const explanation = document.createElement("p");
    explanation.className = "explanation";
    explanation.textContent = existingAnswer.correct
      ? "Correct. " + q.explanation
      : "Incorrect. " + q.explanation;
    card.appendChild(explanation);

    const nextBtn = document.createElement("button");
    nextBtn.textContent = state.currentIndex + 1 < state.sessionQuestions.length ? "Next Question" : "See Results";
    nextBtn.addEventListener("click", () => {
      state.currentIndex++;
      render();
    });
    card.appendChild(nextBtn);
  }

  container.appendChild(card);
  return container;
}
```

Replace with:
```js
function renderQuiz() {
  const container = document.createElement("div");
  const q = state.sessionQuestions[state.currentIndex];

  if (!q) {
    state.screen = "results";
    render();
    return container;
  }

  const progress = document.createElement("div");
  progress.className = "progress";
  progress.textContent = `Question ${state.currentIndex + 1} of ${state.sessionQuestions.length} — ${q.subtopic}`;
  container.appendChild(progress);

  const card = document.createElement("div");
  card.className = "card";
  const questionText = document.createElement("p");
  questionText.textContent = q.question;
  card.appendChild(questionText);

  const existingAnswer = state.answers.find(a => a.questionId === q.id);
  const alreadyAnswered = !!existingAnswer;
  const isLastQuestion = state.currentIndex + 1 >= state.sessionQuestions.length;

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

  if (state.mode === "study" && alreadyAnswered) {
    const explanation = document.createElement("p");
    explanation.className = "explanation";
    explanation.textContent = existingAnswer.correct
      ? "Correct. " + q.explanation
      : "Incorrect. " + q.explanation;
    card.appendChild(explanation);
  }

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

- [ ] **Step 2: Manual verification**

Open `index.html`, start a quiz in Study mode:
- Confirm question 1 has no "Previous" button (first question).
- Answer question 1; confirm "Next Question" appears alongside "Previous" (still absent) and "Home".
- Click "Next Question"; confirm "Previous" now appears on question 2.
- Click "Previous"; confirm question 1 re-displays in its answered (read-only, feedback-shown) state, and clicking its options does nothing.
- Navigate to the last question and answer it; confirm the advance button reads "Finish" and clicking it goes to the Results screen.
- Start a fresh quiz in Exam mode: confirm "Next Question"/"Previous" are available immediately without answering, confirm clicking a different option on a previously-answered question updates the answer (no visual feedback shown, per Exam mode design), and confirm no duplicate entries pile up in `state.answers` (check via console: `state.answers.length` should never exceed `state.sessionQuestions.length`).
- Mid-quiz (either mode), click "Home"; confirm you land on the Home screen. In the browser console, run `loadProgress().inProgressSession` and confirm it reflects the current mode/index/answers.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add Previous/Next/Finish/Home navigation to quiz screen"
```

---

### Task 5: Clear in-progress session on quiz completion

**Files:**
- Modify: `index.html` (the `renderResults` function, currently starting at line 1500)

- [ ] **Step 1: Add `clearInProgressSession()` call alongside the existing `saveSessionToStorage()` call**

Current code (top of `renderResults`):
```js
function renderResults() {
  const container = document.createElement("div");
  const total = state.answers.length;
  const correctCount = state.answers.filter(a => a.correct).length;

  saveSessionToStorage();
```

Replace the `saveSessionToStorage();` line with:
```js
  saveSessionToStorage();
  clearInProgressSession();
```

- [ ] **Step 2: Manual verification**

Start a quiz, click "Home" mid-quiz (confirm `loadProgress().inProgressSession` is non-null per Task 4's verification), then resume it (once Task 6 adds the Resume button — if Task 6 hasn't landed yet, you can simulate resuming by manually restoring `state` from `loadProgress().inProgressSession` in the console and calling `render()`), finish the quiz normally by reaching Results, and confirm `loadProgress().inProgressSession` is now `null`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Clear in-progress session when a quiz reaches the results screen"
```

---

### Task 6: "All Topics" checkbox + Resume/Restart on home screen

**Files:**
- Modify: `index.html` (the `renderHome` function, currently at line 1319-1389)

- [ ] **Step 1: Insert an "All Topics" checkbox before the per-topic checkbox loop**

Current code (relevant section, inside `renderHome`):
```js
  const topicCard = document.createElement("div");
  topicCard.className = "card";
  const topicTitle = document.createElement("h3");
  topicTitle.textContent = "Topics";
  topicCard.appendChild(topicTitle);

  getTopics().forEach(topic => {
```

Insert this block between `topicCard.appendChild(topicTitle);` and `getTopics().forEach(topic => {`:
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
(So the full sequence becomes: `topicCard.appendChild(topicTitle);`, then the new "All Topics" block above, then the existing `getTopics().forEach(...)` loop unchanged.)

- [ ] **Step 2: Add Resume/Restart buttons near the top of `renderHome`**

Current code (start of `renderHome`):
```js
function renderHome() {
  const container = document.createElement("div");

  const title = document.createElement("h1");
  title.textContent = "UroQuiz";
  container.appendChild(title);

  const topicCard = document.createElement("div");
```

Insert this block between `container.appendChild(title);` and `const topicCard = document.createElement("div");`:
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

```

- [ ] **Step 3: Manual verification**

Open `index.html` with no prior `localStorage` data: confirm no "Resume Quiz"/"Restart" buttons appear, and an "All Topics" checkbox appears above the three topic checkboxes. Check "All Topics"; confirm all three topic checkboxes visually become checked. Start a quiz, click "Home" mid-quiz, and confirm the Home screen now shows "Resume Quiz" and "Restart" buttons. Click "Resume Quiz"; confirm you land back on the exact question/state you left. Return home again, click "Restart" this time; confirm the buttons disappear and `loadProgress().inProgressSession` is `null`.

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "Add All Topics checkbox and Resume/Restart quiz session controls"
```

---

### Task 7: Full walkthrough and redeploy

**Files:** none (verification + deployment step)

- [ ] **Step 1: Full manual walkthrough**

On a desktop-sized browser window, walk through:
- Home: check "All Topics", start a quiz in Study mode. Answer a few questions, use Previous to go back and confirm answered questions show read-only feedback, use Next to move forward, confirm option order for a repeated question differs across separate quiz starts.
- Mid-quiz, click Home; confirm Resume/Restart appear on the home screen; click Resume and confirm exact state restoration.
- Finish a quiz (reach Results via "Finish"); confirm results/missed-question review works as before, and confirm `loadProgress().inProgressSession` is `null` afterward.
- Repeat the same flow in Exam mode: confirm free Previous/Next navigation without needing to answer first, and confirm re-answering a previously-answered question updates rather than duplicates.
- Check the Stats dashboard still reflects accuracy correctly after these sessions.

- [ ] **Step 2: Redeploy the Claude Artifact**

Use the Artifact tool with `file_path` set to `/Users/jessica/Desktop/UroQuiz/index.html` and `url` set to the existing published Artifact URL (`https://claude.ai/code/artifact/0d869db7-55ef-4d35-a68c-fab9f5126e8a`) so the update lands on the same link, keeping the same favicon (🩺) and description.

- [ ] **Step 3: Push to GitHub**

```bash
git push origin main
```
