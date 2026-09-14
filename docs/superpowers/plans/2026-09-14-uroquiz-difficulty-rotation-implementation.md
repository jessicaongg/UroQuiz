# UroQuiz Harder/Deeper Questions + Rotation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rewrite all existing questions' distractors to be clinically plausible, roughly double the question bank (94 → ~200) with deeper, scenario-driven content, and add a rotation mechanism so a question can't reappear until every question in the current topic selection has had a turn.

**Architecture:** Rotation is a small, mechanical change to `loadProgress`, `startQuiz`, and `saveSessionToStorage` in the existing single-file `index.html` (new `lastSeenAt` map in the persisted `progress` object). The content work is a large but non-structural change to the `QUESTIONS` array only — no schema change (`{id, topic, subtopic, question, options, correctIndex, explanation}` stays as-is), split into three commits by top-level topic to keep each commit reviewable.

**Tech Stack:** Same as the existing app — vanilla HTML/CSS/JavaScript, no npm, no bundler, no test framework. Verification stays manual (visual check in a browser) plus a one-off Node integrity check (no dependencies) for the content pass, per `docs/superpowers/specs/2026-09-14-uroquiz-difficulty-rotation-design.md`.

**Note on content tasks (3-5):** Tasks 1-2 are ordinary code changes with exact diffs, per the usual plan format. Tasks 3-5 are large-scale medical content authoring (adding ~106 new MCQs and rewriting ~94 existing ones) — pre-writing all of that text into this plan document would just relocate the work rather than plan it. Each content task instead specifies: the exact rubric to apply (from the design spec), a concrete per-subtopic target count, one fully worked example to anchor format/quality, and the same schema fields as every other question. This mirrors how this project's own existing `uroquiz-eau-guideline-update` scheduled task already treats bank growth — bounded by rules, not pre-scripted.

---

## File Structure

- Modify: `index.html` only (no new files; project convention is single-file, no separate data files)

---

### Task 1: Sync with in-flight EAU guideline update

**Files:** none (git sync step)

- [ ] **Step 1: Confirm the `uroquiz-eau-guideline-update` scheduled-task run has finished**

That run and this plan both edit `index.html` and push to `main` in the same working directory — do not proceed to Task 2 until it has completed (check via `mcp__scheduled-tasks__list_task_runs` or `mcp__ccd_session_mgmt__get_session` on its session id).

- [ ] **Step 2: Pull latest**

```bash
git pull origin main
```

Expected: fast-forward merge (or "Already up to date" if the EAU run made no changes). Resolve before proceeding if it isn't a clean fast-forward — this plan assumes the EAU run's changes (if any) are only inside the `QUESTIONS` array.

---

### Task 2: Rotation mechanism

**Files:**
- Modify: `index.html:72-96` (`loadProgress`, `saveProgress` region)
- Modify: `index.html:1549-1557` (`startQuiz`)
- Modify: `index.html:1676-1697` (`saveSessionToStorage`)

- [ ] **Step 1: Add `lastSeenAt` to the progress data model and its migration**

Current code:
```js
function loadProgress() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return { topicStats: {}, missedIds: [], inProgressSession: null, attemptedIds: [] };
    const data = JSON.parse(raw);
    if (!data.topicStats) data.topicStats = {};
    if (!data.missedIds) data.missedIds = [];
    if (data.inProgressSession === undefined) data.inProgressSession = null;
    if (data.attemptedIds === undefined) {
      // Pre-existing installs (before per-question coverage tracking was added) only
      // have aggregate topicStats, not individual question ids. Approximate coverage
      // with placeholder ids so readiness doesn't wrongly drop to "0 seen" on upgrade.
      const legacyAttempted = Object.values(data.topicStats).reduce((sum, s) => sum + s.attempted, 0);
      const legacyCount = Math.min(legacyAttempted, QUESTIONS.length);
      data.attemptedIds = Array.from({ length: legacyCount }, (_, i) => `legacy-${i}`);
    }
    return data;
  } catch (e) {
    return { topicStats: {}, missedIds: [], inProgressSession: null, attemptedIds: [] };
  }
}
```

Replace with (adds `lastSeenAt: {}` to both empty-state returns, and a migration branch for existing saved data):
```js
function loadProgress() {
  try {
    const raw = localStorage.getItem(STORAGE_KEY);
    if (!raw) return { topicStats: {}, missedIds: [], inProgressSession: null, attemptedIds: [], lastSeenAt: {} };
    const data = JSON.parse(raw);
    if (!data.topicStats) data.topicStats = {};
    if (!data.missedIds) data.missedIds = [];
    if (data.inProgressSession === undefined) data.inProgressSession = null;
    if (data.attemptedIds === undefined) {
      // Pre-existing installs (before per-question coverage tracking was added) only
      // have aggregate topicStats, not individual question ids. Approximate coverage
      // with placeholder ids so readiness doesn't wrongly drop to "0 seen" on upgrade.
      const legacyAttempted = Object.values(data.topicStats).reduce((sum, s) => sum + s.attempted, 0);
      const legacyCount = Math.min(legacyAttempted, QUESTIONS.length);
      data.attemptedIds = Array.from({ length: legacyCount }, (_, i) => `legacy-${i}`);
    }
    if (data.lastSeenAt === undefined) {
      // Pre-rotation installs have no exposure history. Default to "never seen" for
      // every question rather than guessing, so the first post-upgrade session draws
      // from the full pool (least-surprising: nothing wrongly excluded as "recent").
      data.lastSeenAt = {};
    }
    return data;
  } catch (e) {
    return { topicStats: {}, missedIds: [], inProgressSession: null, attemptedIds: [], lastSeenAt: {} };
  }
}
```

- [ ] **Step 2: Use `lastSeenAt` to pick the session pool in `startQuiz`**

Current code:
```js
function startQuiz() {
  const topics = state.selectedTopics.length ? state.selectedTopics : getTopics();
  const pool = QUESTIONS.filter(q => topics.includes(q.topic));
  state.sessionQuestions = shuffle(pool.map(shuffleQuestionOptions));
  state.currentIndex = 0;
  state.answers = [];
  state.screen = "quiz";
  render();
}
```

Replace with:
```js
const SESSION_SIZE = 20;

function startQuiz() {
  const topics = state.selectedTopics.length ? state.selectedTopics : getTopics();
  const pool = QUESTIONS.filter(q => topics.includes(q.topic));
  const progress = loadProgress();
  const byRecency = pool.slice().sort((a, b) => {
    const aSeen = progress.lastSeenAt[a.id] ?? -Infinity;
    const bSeen = progress.lastSeenAt[b.id] ?? -Infinity;
    return aSeen - bSeen;
  });
  const batch = byRecency.slice(0, SESSION_SIZE);
  state.sessionQuestions = shuffle(batch.map(shuffleQuestionOptions));
  state.currentIndex = 0;
  state.answers = [];
  state.screen = "quiz";
  render();
}
```

(Never-seen questions have `lastSeenAt[id] === undefined`, which `?? -Infinity` sorts first; among already-seen questions, the least-recently-seen sorts first. `SESSION_SIZE` is declared once, above `startQuiz`, so it's available wherever it's needed later.)

- [ ] **Step 3: Record exposure in `saveSessionToStorage`**

Current code:
```js
function saveSessionToStorage() {
  const progress = loadProgress();

  state.answers.forEach(answer => {
    const q = state.sessionQuestions.find(q => q.id === answer.questionId);
    const topic = q.topic;
    if (!progress.topicStats[topic]) progress.topicStats[topic] = { attempted: 0, correct: 0 };
    progress.topicStats[topic].attempted++;
    if (answer.correct) progress.topicStats[topic].correct++;

    const missedSet = new Set(progress.missedIds);
    if (answer.correct) missedSet.delete(answer.questionId);
    else missedSet.add(answer.questionId);
    progress.missedIds = [...missedSet];

    const attemptedSet = new Set(progress.attemptedIds);
    attemptedSet.add(answer.questionId);
    progress.attemptedIds = [...attemptedSet];
  });

  saveProgress(progress);
}
```

Replace with (adds one line recording `lastSeenAt` per answered question):
```js
function saveSessionToStorage() {
  const progress = loadProgress();

  state.answers.forEach(answer => {
    const q = state.sessionQuestions.find(q => q.id === answer.questionId);
    const topic = q.topic;
    if (!progress.topicStats[topic]) progress.topicStats[topic] = { attempted: 0, correct: 0 };
    progress.topicStats[topic].attempted++;
    if (answer.correct) progress.topicStats[topic].correct++;

    const missedSet = new Set(progress.missedIds);
    if (answer.correct) missedSet.delete(answer.questionId);
    else missedSet.add(answer.questionId);
    progress.missedIds = [...missedSet];

    const attemptedSet = new Set(progress.attemptedIds);
    attemptedSet.add(answer.questionId);
    progress.attemptedIds = [...attemptedSet];

    progress.lastSeenAt[answer.questionId] = Date.now();
  });

  saveProgress(progress);
}
```

- [ ] **Step 4: Manual verification**

- Clear `localStorage` (fresh-user simulation). Select a single small subtopic (e.g. "penile", 3 questions pre-expansion) so a full round completes quickly. Start a Study session; confirm all questions in the subtopic appear (pool smaller than `SESSION_SIZE`). Finish it. Start another session with the same topic selection; confirm the same questions reappear in a *different* shuffled order (expected — the round has "completed," least-recently-seen ties are broken by shuffle) rather than an error or empty session.
- Select a larger topic (e.g. "oncology", pre-expansion 33 questions, more once Tasks 3-5 land). Start a session; confirm it contains exactly `min(33, 20) = 20` questions. Finish it. Start another session with the same topic; confirm it draws from the remaining ~13 never-seen questions first (all 13 should appear, plus 7 of the least-recently-seen from the first batch to fill to 20).
- Simulate a pre-upgrade install: in the browser console, `JSON.parse(localStorage.getItem('uroquiz-progress-v1'))`, delete the `lastSeenAt` key, save it back, reload, and start a quiz — confirm no error and a normal full-pool-first session.

- [ ] **Step 5: Commit**

```bash
git add index.html
git commit -m "Add lastSeenAt-based session rotation so questions don't repeat until every question in scope has had a round"
```

---

### Task 3: Content pass — Oncology

**Files:**
- Modify: `index.html` (the `QUESTIONS` array, oncology entries: `subtopic` values `prostate`, `bladder`, `rcc`, `utuc`, `testis`, `penile`)

- [ ] **Step 1: Read current state before editing**

Read the full oncology block of `QUESTIONS` in `index.html` and note, per subtopic, the current count and highest existing numeric suffix (e.g. `onco-prostate-008` → next is `onco-prostate-009`). Do not guess — the design spec's target counts below assume the *current* counts read from the file, which may already differ slightly if the Task 1 EAU-update run touched oncology questions.

- [ ] **Step 2: Rewrite every existing oncology question's distractors**

Apply the distractor-quality rubric from `docs/superpowers/specs/2026-09-14-uroquiz-difficulty-rotation-design.md` § "Content pass": each wrong option must be either (a) an adjacent risk tier/threshold, (b) a plausible prior-guideline-version or other-body (AUA/NCCN) recommendation, or (c) a half-right answer (right modality, wrong line/sequencing/threshold). Keep `id`, `topic`, `subtopic`, and `correctIndex`'s underlying fact unchanged; update `explanation` if needed so it still correctly justifies the answer against the new distractors (2-4 sentences, naming the rationale, not just restating the fact).

Worked example — `onco-bladder-001` before:
```js
{
  id: "onco-bladder-001",
  topic: "oncology", subtopic: "bladder",
  question: "For high-risk non-muscle-invasive bladder cancer (NMIBC), what is the recommended intravesical therapy per EAU guidelines?",
  options: [
    "Intravesical chemotherapy single instillation only",
    "BCG induction plus maintenance for 1-3 years",
    "Immediate radical cystectomy in all cases",
    "Surveillance only, no adjuvant therapy"
  ],
  correctIndex: 1,
  explanation: "High-risk NMIBC is treated with BCG induction (6 weekly instillations) followed by a maintenance schedule, typically for 1-3 years depending on risk substratification."
}
```

After (distractors now plausible near-misses instead of extremes):
```js
{
  id: "onco-bladder-001",
  topic: "oncology", subtopic: "bladder",
  question: "For high-risk non-muscle-invasive bladder cancer (NMIBC), what is the recommended intravesical therapy per EAU guidelines?",
  options: [
    "BCG induction only, with no maintenance phase",
    "BCG induction plus maintenance for 1-3 years",
    "Mitomycin C maintenance instillations for 1 year, reserving BCG for recurrence",
    "BCG induction plus maintenance for 1-3 years, but only after a normal cystoscopy at 3 months"
  ],
  correctIndex: 1,
  explanation: "High-risk NMIBC requires BCG induction (6 weekly instillations) followed by a maintenance schedule, typically 1-3 years depending on risk substratification — induction alone (option A) undertreats, and chemotherapy maintenance (option C) is inferior to BCG for high-risk disease per EAU comparative evidence. Maintenance is not gated on an interim cystoscopy (option D); it proceeds per schedule with surveillance cystoscopy alongside, not as a prerequisite."
}
```

- [ ] **Step 3: Expand each oncology subtopic to its target count**

Add new questions using the same schema and rubric, following the existing id convention. Prefer clinical-vignette stems where the guideline content supports it. Target counts (current → target, add the difference — adjust only if Step 1's actual current count differs from the pre-expansion baseline below, in which case keep the same target total):

| Subtopic | Baseline count | Target count | Add |
|---|---|---|---|
| prostate | 8 | 14 | 6 |
| bladder | 7 | 13 | 6 |
| rcc | 6 | 12 | 6 |
| utuc | 5 | 12 | 7 |
| testis | 4 | 11 | 7 |
| penile | 3 | 10 | 7 |

Oncology subtotal: 33 → 72 (add 39).

- [ ] **Step 4: Verify no duplicate ids introduced in this block**

```bash
grep -o 'id: "onco-[a-z-]*-[0-9]*"' index.html | sort | uniq -d
```
Expected: no output (empty = no duplicates).

- [ ] **Step 5: Manual spot-check**

Read back 5 rewritten and 5 newly added oncology questions; confirm each has exactly 4 options, `correctIndex` still points at the genuinely correct answer, and the explanation reads as clinically sound (not just plausible-sounding).

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Deepen and expand oncology question bank (33 -> 72 questions)"
```

---

### Task 4: Content pass — Non-oncology

**Files:**
- Modify: `index.html` (the `QUESTIONS` array, non-oncology entries: `subtopic` values `stones`, `bph`, `incontinence`, `neuro-urology`, `female-functional`)

- [ ] **Step 1: Read current state before editing**

Same as Task 3 Step 1, scoped to the non-oncology block.

- [ ] **Step 2: Rewrite every existing non-oncology question's distractors**

Same rubric and worked-example pattern as Task 3 Step 2, applied to this block.

- [ ] **Step 3: Expand each non-oncology subtopic to its target count**

| Subtopic | Baseline count | Target count | Add |
|---|---|---|---|
| stones | 9 | 15 | 6 |
| bph | 8 | 14 | 6 |
| incontinence | 6 | 13 | 7 |
| neuro-urology | 5 | 12 | 7 |
| female-functional | 5 | 12 | 7 |

Non-oncology subtotal: 33 → 66 (add 33).

- [ ] **Step 4: Verify no duplicate ids introduced in this block**

```bash
grep -oE 'id: "(stones|bph|incontinence|neuro|female)[a-z-]*-[0-9]*"' index.html | sort | uniq -d
```
Expected: no output.

- [ ] **Step 5: Manual spot-check**

Same as Task 3 Step 5, scoped to this block.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Deepen and expand non-oncology question bank (33 -> 66 questions)"
```

---

### Task 5: Content pass — Infections/other

**Files:**
- Modify: `index.html` (the `QUESTIONS` array, entries: `subtopic` values `uti-urosepsis`, `trauma-reconstruction`, `andrology`, `paediatric`, `transplantation`)

- [ ] **Step 1: Read current state before editing**

Same as Task 3 Step 1, scoped to this block.

- [ ] **Step 2: Rewrite every existing question's distractors in this block**

Same rubric and worked-example pattern as Task 3 Step 2.

- [ ] **Step 3: Expand each subtopic to its target count**

| Subtopic | Baseline count | Target count | Add |
|---|---|---|---|
| uti-urosepsis | 8 | 14 | 6 |
| trauma-reconstruction | 5 | 12 | 7 |
| andrology | 6 | 13 | 7 |
| paediatric | 6 | 13 | 7 |
| transplantation | 3 | 10 | 7 |

Subtotal: 28 → 62 (add 34). Full bank total after Tasks 3-5: 94 → 200.

- [ ] **Step 4: Verify no duplicate ids introduced in this block**

```bash
grep -oE 'id: "(infect|trauma|andro|paeds|transplant)[a-z-]*-[0-9]*"' index.html | sort | uniq -d
```
Expected: no output. (Prefixes confirmed from the current file: `infect*` for uti-urosepsis, `trauma*`, `andro*` for andrology, `paeds*` for paediatric, `transplant*` for transplantation.)

- [ ] **Step 5: Manual spot-check**

Same as Task 3 Step 5, scoped to this block.

- [ ] **Step 6: Commit**

```bash
git add index.html
git commit -m "Deepen and expand infections/other question bank (28 -> 62 questions)"
```

---

### Task 6: Full-bank integrity verification

**Files:** none (verification step)

- [ ] **Step 1: Run the schema/uniqueness check across the entire `QUESTIONS` array**

```bash
node -e "
const fs = require('fs');
const html = fs.readFileSync('index.html', 'utf8');
const startMarker = 'const QUESTIONS = [';
const start = html.indexOf(startMarker) + startMarker.length - 1;
let depth = 0, end = -1;
for (let i = start; i < html.length; i++) {
  if (html[i] === '[') depth++;
  else if (html[i] === ']') { depth--; if (depth === 0) { end = i; break; } }
}
const arrText = html.slice(start, end + 1);
const QUESTIONS = eval(arrText);
const ids = new Set();
const errors = [];
const fields = ['id','topic','subtopic','question','options','correctIndex','explanation'];
for (const q of QUESTIONS) {
  for (const f of fields) if (!(f in q)) errors.push((q.id||'?') + ' missing ' + f);
  if (ids.has(q.id)) errors.push('duplicate id ' + q.id);
  ids.add(q.id);
  if (!Array.isArray(q.options) || q.options.length !== 4) errors.push(q.id + ' options length != 4');
  if (typeof q.correctIndex !== 'number' || q.correctIndex < 0 || q.correctIndex >= q.options.length) errors.push(q.id + ' correctIndex out of range');
}
console.log('Total questions:', QUESTIONS.length);
console.log('Errors:', errors.length);
errors.forEach(e => console.log(' -', e));
"
```

Expected: `Total questions: 200` (or close to it, depending on exact adds), `Errors: 0`. If errors are listed, fix the specific question(s) named and re-run before proceeding.

- [ ] **Step 2: If errors were found, fix and re-commit**

```bash
git add index.html
git commit -m "Fix question bank schema issues found by integrity check"
```

(Skip this step if Step 1 reported zero errors.)

---

### Task 7: Full walkthrough, redeploy, and push

**Files:** none (verification + deployment step)

- [ ] **Step 1: Full manual walkthrough**

- Start a Study session on "All Topics"; confirm it contains 20 questions (rotation cap) and none feel trivially easy (spot-check 5 for plausible distractors).
- Finish it, start another "All Topics" session; confirm none of the 20 just-seen questions reappear (180 remain unseen, well above the 20-question cap).
- Start an Exam-mode session; confirm the same rotation/cap behavior applies (per the "apply to both modes" decision).
- Confirm Missed Question Log (Stats screen) and "Retry Missed" still work unchanged.
- Confirm Exam Readiness percentage on Home still reflects `attemptedIds.length / QUESTIONS.length` correctly against the new total (200).

- [ ] **Step 2: Redeploy the Claude Artifact**

Use the Artifact tool with `file_path` set to `/Users/jessica/Desktop/UroQuiz/index.html` and `url` set to the existing published Artifact URL (`https://claude.ai/code/artifact/0d869db7-55ef-4d35-a68c-fab9f5126e8a`), keeping the same favicon (🩺) and a similar description.

- [ ] **Step 3: Push to GitHub**

```bash
git push origin main
```
