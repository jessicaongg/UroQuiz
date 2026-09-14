# UroQuiz: Harder/Deeper Questions + Spaced Rotation

## Context

User feedback on the current app (94 questions across oncology / non-oncology / infections-other):

1. Questions are too easy — distractors are often implausible extremes, so the correct answer stands out without real recall or reasoning.
2. No repeat protection — `startQuiz()` pulls the *entire* pool matching the selected topics every session (no subset, no cap), so the same full set can reappear immediately in back-to-back sessions.
3. Unclear whether the question bank reflects current EAU guidance.
4. (Added after initial design approval) Grow the bank substantially and make every question — old and new — test deeper understanding, not just single-fact recall.

Item 3 is handled separately: the existing monthly scheduled task `uroquiz-eau-guideline-update` was triggered manually for this request and runs independently (accuracy fixes + modest bank growth). This spec covers items 1, 2, and 4.

**Sequencing constraint:** the EAU-update run and this work both edit `index.html` and push to the same `main` branch in the same working directory (not a worktree). The content/rotation work in this spec must not start editing `index.html` until the EAU-update run has finished and pushed; pull latest before starting.

## Goals

- Rewrite all existing questions' distractors to be clinically plausible rather than obviously wrong.
- Substantially expand the bank (target: roughly double, ~94 → ~190-200 questions), prioritizing thin subtopics (penile: 3, transplantation: 3, testis: 4, neuro-urology: 5, female-functional: 5, trauma-reconstruction: 5, utuc: 5) while also adding depth to well-covered ones.
- Shift question style toward depth: clinical-scenario stems and multi-step reasoning (e.g. "which factor changes management from X to Y") rather than bare fact lookup, with explanations that state the underlying rationale, not just restate the fact.
- Add a spaced-rotation session-selection mechanism so a question doesn't reappear until every other question in the current topic scope has appeared at least once, applied uniformly to Study and Exam mode.

## Non-goals

- No session-size configuration UI — batch size is a hardcoded constant for this iteration.
- No changes to scoring, stats screens, topic-selection UI, or the missed-question retry flow beyond what's needed for rotation bookkeeping to stay consistent.
- No live EAU guideline fetching in this spec's scope (handled by the separate scheduled task).

## Design

### 1. Content pass: harder distractors + depth + expansion

Applies to all existing questions and all newly added ones.

**Distractor quality.** Replace implausible-extreme wrong options with distractors drawn from one of:
- An adjacent category/threshold (e.g. intermediate-risk criteria as a wrong answer on a low-risk question, instead of an unrelated extreme).
- A recommendation that was correct under a prior guideline version or a different body (AUA, NCCN), plausible unless you know the current EAU position.
- A "half-right" answer: correct drug class/modality, wrong line of therapy, wrong threshold, or wrong sequencing.

**Depth.** Prefer clinical-vignette stems (brief patient scenario + a decision point) over bare "what does EAU recommend for X" recall, where the underlying guideline content supports it. Explanations should name the mechanism or guideline rationale (why the distractors are wrong, not just that the correct answer is right), in ~2-4 sentences.

**Expansion.** Add new questions per subtopic using the existing schema exactly: `{id, topic, subtopic, question, options, correctIndex, explanation}`, 4 options, unique sequential ids per subtopic (e.g. `onco-penile-004` following existing `onco-penile-003`). Same accuracy caveat as the existing bank applies: content is generated from training knowledge, not live-verified against EAU, and worth spot-checking — the monthly automation partially mitigates drift going forward.

**Verification before commit:** every question has all 7 fields, `correctIndex` is a valid index into its own `options`, and there are no duplicate ids anywhere in the array (grep/count the full array, not just new additions) — same check the monthly automation already performs.

### 2. Spaced rotation

**Data model** — add to the `progress` object (`STORAGE_KEY = "uroquiz-progress-v1"`):

```js
lastSeenAt: { [questionId]: number }  // epoch ms; absent = never seen
```

Migrated in `loadProgress()` the same way `attemptedIds` is today: if `data.lastSeenAt` is undefined, default to `{}` (nothing is treated as "recently seen," so existing users' first post-upgrade session draws from their full pool, least-surprising default).

**Session building** — in `startQuiz()`, replace the current `shuffle(pool)` with:

1. `pool = QUESTIONS.filter(topics.includes(q.topic))` (unchanged).
2. Sort `pool` by `progress.lastSeenAt[q.id] ?? -Infinity` ascending — never-seen questions sort first; among seen questions, the least-recently-seen sorts first.
3. Take the first `SESSION_SIZE` (constant, default `20`) — or the whole sorted pool if smaller.
4. Shuffle only *within* that selected batch (so ordering inside a session stays randomized, but selection itself is recency-driven) and proceed as today (`shuffleQuestionOptions`, etc).

**Recording exposure** — in `saveSessionToStorage()` (already loops over `state.answers` per question), add `progress.lastSeenAt[answer.questionId] = Date.now()` alongside the existing `topicStats`/`missedIds`/`attemptedIds` updates. Only *answered* questions update `lastSeenAt`, consistent with how `attemptedIds`/`missedIds` already work — a question left unanswered (e.g. user exits via Home mid-session) is not considered "seen."

**Why this satisfies "no repeat until everyone's had a round":** because selection always takes the globally least-recently-seen questions first within whatever topic scope is active, a question cannot be picked again until every other question in that scope has a more recent (or equal, for never-seen) `lastSeenAt` — i.e. has had its turn. This holds automatically across arbitrary/changing topic selections with no separate cycle-reset bookkeeping.

**Mode scope:** applies identically to Study and Exam mode (per explicit choice — no exam-mode exemption). Missed-question retry sessions and in-progress-session resume are unaffected (they already bypass `startQuiz()`'s pool selection).

## Testing / Verification

- Manual, in-browser (Artifact preview): start a Study session on a single small-subtopic selection, complete it, start another — verify no repeats until the subtopic's full pool has cycled once.
- Verify Exam mode shows the same rotation behavior.
- Verify a fresh `localStorage` (new user) and an existing pre-upgrade `localStorage` (missing `lastSeenAt`) both produce a working first session.
- Script/grep check: all question objects have 7 fields, valid `correctIndex`, no duplicate ids across the full array.
- Spot-check a sample of rewritten questions/distractors for clinical plausibility and that `correctIndex` still points at the genuinely correct option after rewriting.

## Rollout

1. Wait for the in-flight `uroquiz-eau-guideline-update` run to finish, push, and redeploy.
2. `git pull` to get its changes before editing.
3. Implement rotation mechanism (small, mechanical code change).
4. Implement content pass (large content change: rewrite existing + add new questions).
5. Verify per above, commit, push, redeploy the Artifact (same URL).
