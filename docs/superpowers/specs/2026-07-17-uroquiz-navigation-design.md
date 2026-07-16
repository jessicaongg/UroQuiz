# UroQuiz — Navigation, Shuffling & Resume Design

**Date:** 2026-07-17
**Status:** Approved

## Purpose

Extend the existing UroQuiz app (single self-contained HTML/CSS/JS file, hosted
as a Claude Artifact — see `docs/superpowers/specs/2026-07-16-uroquiz-design.md`
for the original design) with:

1. Shuffling of answer options within each question (question-order shuffling
   already exists).
2. An explicit "All Topics" checkbox on the home screen.
3. Previous/Next navigation within a quiz session, replacing the current
   click-to-advance/click-to-reveal model.
4. A resumable in-progress session: leaving mid-quiz via a new "Home" button
   saves state; the home screen offers "Resume Quiz" (and "Restart" to
   discard the saved session).

This does not change the original app's core screens (Home, Quiz, Results,
Stats), question schema, or hosting model — it modifies quiz-taking
interaction and adds one new `localStorage` field.

## Architecture changes

### Per-session option shuffling

When `startQuiz()` (or the "Retry Missed Only" / "Quiz Missed Questions"
flows) builds `state.sessionQuestions`, each question is copied with its
`options` array shuffled and `correctIndex` remapped to match the new order.
The original `QUESTIONS` array is never mutated — only per-session copies are
shuffled. A shared helper produces this copy:

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

`startQuiz()` and any other place that builds `sessionQuestions` from
`QUESTIONS` maps each question through `shuffleQuestionOptions` before
shuffling question order (order of the two shuffles doesn't matter, but
option-shuffling must happen once, producing a fixed per-session copy — the
options must not re-shuffle on every render of the same question).

### Decoupling answering from navigation

`recordAnswer` no longer advances `state.currentIndex`. It becomes an upsert:

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

Navigation (`state.currentIndex++` / `state.currentIndex--`) is now handled
exclusively by explicit Previous/Next button click handlers in `renderQuiz`.

### Quiz screen behavior by mode

- **Study mode:** clicking an option (when not yet answered) records the
  answer and immediately shows correct/incorrect highlighting + explanation,
  same as today. Once answered, re-clicking options does nothing (existing
  guard). Previous/Next both work regardless of answered state; revisiting an
  answered question shows it in its answered (read-only, feedback-revealed)
  state.
- **Exam mode:** clicking an option records/updates the answer with no
  visual feedback (as today). Unlike today, it does NOT auto-advance.
  Revisiting an already-answered question allows clicking a different option,
  which updates the existing answer entry (upsert, not duplicate).
- Both modes: "Previous" is hidden/disabled on the first question. "Next"
  reads "Finish" on the last question and transitions to `results` screen
  (same transition path as the existing `if (!q)` guard, but now reachable
  via explicit button rather than only via running out of questions).
- A "Home" button is added to the quiz screen. Clicking it calls
  `saveInProgressSession()` (see below) then sets `state.screen = "home"`.

### Resumable in-progress session

New `localStorage` field alongside the existing `topicStats`/`missedIds`
(default `null` when absent — `loadProgress()`'s existing normalization
pattern is extended to default this field too):

```js
{
  topicStats: {...},
  missedIds: [...],
  inProgressSession: {
    mode: "study" | "exam",
    sessionQuestions: [...],  // per-session shuffled copies, as built by startQuiz()
    currentIndex: 0,
    answers: [...]
  } | null
}
```

- **Saved** when the quiz-screen "Home" button is clicked (`saveInProgressSession()`
  writes `state.mode`, `state.sessionQuestions`, `state.currentIndex`,
  `state.answers` into `inProgressSession` and persists via the existing
  `saveProgress()`).
- **Cleared** (`inProgressSession: null`, persisted) when a quiz reaches the
  Results screen normally, or when "Restart" is clicked on the home screen.
- **Consumed** when "Resume Quiz" is clicked on the home screen: loads
  `inProgressSession` into `state` (`mode`, `sessionQuestions`, `currentIndex`,
  `answers`) and transitions to the `quiz` screen. Does NOT clear it from
  storage at this point — it's only cleared on finish/restart, so if the user
  leaves again mid-resume, the "Home" button save flow overwrites it again
  with the latest progress.

## Features & UX flow (updated)

1. **Home screen** — topic checkboxes plus a new "All Topics" checkbox
   (checking it checks all three topic boxes; one-way convenience action, not
   two-way synced — unchecking individual boxes afterward does not affect
   the "All Topics" checkbox's own state). Mode selector and "Start Quiz"
   unchanged. If `inProgressSession` is non-null: a "Resume Quiz" button
   appears, with a smaller "Restart" action next to it that clears the saved
   session (so a subsequent "Start Quiz" begins fresh instead of leaving a
   stale resume target around).
2. **Quiz screen** — progress indicator, question card, Previous/Next buttons
   (Next → "Finish" on the last question), and a Home button. Behavior by
   mode as described above.
3. **Results / Stats screens** — unchanged from the existing implementation.

## Out of scope

- Two-way sync between "All Topics" and individual topic checkboxes (e.g.
  auto-checking "All Topics" if a user manually checks all three individually).
- Multiple concurrent in-progress sessions (only one is ever saved — starting
  a brand new quiz before resuming/finishing the saved one overwrites it).
- Cross-device resume (still per-browser `localStorage`, per the original
  design).

## Testing / verification approach

Manual, consistent with the original design (single-file, no build tooling):

- Start a quiz twice with the same topic selection; confirm a given
  question's option order differs between the two sessions (option
  shuffling working, not just a fixed order).
- Navigate Previous/Next through a full session in Study mode; confirm
  answered questions display their revealed state when revisited, and
  unanswered questions remain interactive.
- In Exam mode, answer a question, navigate away and back, choose a
  different option; confirm `state.answers` has one entry for that question
  (updated), not two.
- Click Home mid-quiz; confirm `localStorage`'s `inProgressSession` is
  populated; reload the page; click "Resume Quiz"; confirm the exact
  question/answers state is restored.
- Click "Restart" on the home screen when a resumable session exists;
  confirm `inProgressSession` becomes `null` and "Resume Quiz" no longer
  appears.
- Finish a quiz normally (reach Results); confirm `inProgressSession` is
  cleared.
