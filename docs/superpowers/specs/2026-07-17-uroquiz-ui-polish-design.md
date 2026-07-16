# UroQuiz — Exam Selection Highlight, All-Topics Exclusivity, Button Layout Design

**Date:** 2026-07-17
**Status:** Approved

## Purpose

Four small UX refinements to the already-complete UroQuiz app, confirmed via
mockups during brainstorming:

1. In Exam mode, clicking an option now visibly highlights it as "selected"
   (distinct from Study mode's correct/incorrect reveal colors).
2. The "All Topics" checkbox becomes mutually exclusive with the individual
   topic checkboxes, rather than checking all three simultaneously.
3. The quiz screen's Previous/Next/Home buttons are rearranged: Previous
   pinned far-left, Next/Finish pinned far-right (one row), Home centered on
   its own row below.
4. The home screen's action buttons become mutually exclusive based on
   whether a resumable session exists: "Start Quiz" alone (no saved session)
   vs. "Resume Quiz" + "Restart" together (saved session present) — replacing
   the current split where Resume/Restart appear at the top of the page and
   Start Quiz/View Stats appear at the bottom. "View Stats" always appears
   centered on its own row below the action row.

## Design decisions

### 1. Exam mode selection highlight

A new `.option.selected` CSS class, using the existing `--accent` color
(already used for buttons) — visually distinct from `--correct`/`--incorrect`
so it can never be mistaken for a correctness reveal:
```css
.option.selected { border-color: var(--accent); background: color-mix(in srgb, var(--accent) 15%, var(--card-bg)); }
```
In `renderQuiz`'s option-rendering loop, when `state.mode === "exam"` and the
question has been answered, the option matching `existingAnswer.chosenIndex`
gets `.selected`. This persists correctly when navigating away and back,
since it reads from the already-recorded answer, not transient click state.

### 2. All Topics mutual exclusivity

No new state field is needed — the app already treats an empty
`state.selectedTopics` as "use every topic" (see `startQuiz`'s
`state.selectedTopics.length ? state.selectedTopics : getTopics()`
fallback). The "All Topics" checkbox is changed to lean on this existing
fallback instead of explicitly selecting all three topics:

- Checking "All Topics" sets `state.selectedTopics = []` (not
  `getTopics()`), so on re-render every individual topic checkbox
  (`checkbox.checked = state.selectedTopics.includes(topic)`) naturally
  shows unchecked.
- The "All Topics" checkbox itself is now rendered with
  `checked = state.selectedTopics.length === 0`, so it stays visually
  checked across re-renders (previously it always rendered unchecked,
  a pre-existing minor quirk flagged in a prior review) and un-checks
  itself the moment an individual topic is chosen (since that pushes into
  `selectedTopics`, making its length > 0).
- No change needed to the individual checkboxes' own logic.

### 3. Quiz screen button layout

Two rows instead of one:
- **Top row** (`.nav-row-split`, `display:flex; justify-content:space-between`):
  Previous (or an empty placeholder div, keeping Next pinned right when
  Previous is absent on question 1) on the left; Next Question/Finish (or an
  empty placeholder) on the right.
- **Bottom row** (`.home-row`, `display:flex; justify-content:center`): Home,
  alone, centered.

Using empty placeholder `<div>`s in the unused slot (rather than
conditionally switching `justify-content`) keeps the layout logic simple and
consistent regardless of which buttons are present.

### 4. Home screen action buttons

The existing Resume/Restart block (currently rendered near the top of
`renderHome`, right after the readiness card) is removed from that position.
In its place, the existing bottom "Start Quiz" / "View Stats" block is
replaced with:

- **Action row**: if `progress.inProgressSession` is truthy, render "Resume
  Quiz" + "Restart" (same click handlers as before); otherwise render "Start
  Quiz" alone.
- **Stats row** (`.home-row-center`, centered): "View Stats", always shown,
  on its own row below the action row.

This consolidates what were previously two separate button groups (one at
the top of the page, one at the bottom) into a single bottom action area,
with Start-Quiz-vs-Resume-Quiz+Restart as mutually exclusive alternatives
rather than Resume/Restart being an *addition* above the normal flow.

## Architecture

All four changes are edits to the existing `index.html` — no new files, no
new state fields, no new storage schema. Affected functions: `renderHome`,
`renderQuiz`. Affected CSS: three new rules (`.option.selected`,
`.nav-row-split`, `.home-row-center` — `.home-row` already exists as a name
pattern but needs to be added since the quiz screen's current single
`navRow` div has no class).

## Out of scope

- Changing what "All Topics" or individual topic selection actually does
  functionally (`startQuiz`'s filtering logic is unchanged — this is purely
  about which checkboxes visually appear ticked).
- Any change to Study mode's existing correct/incorrect highlight behavior.
- Persisting which home-screen layout state (action-row vs resume-row) was
  last shown — it's always derived live from `loadProgress().inProgressSession`.

## Testing / verification approach

Manual, consistent with prior specs:
- In Exam mode, click an option; confirm it shows the blue `.selected`
  highlight (not green/red) and no explanation is shown. Navigate away and
  back to that question; confirm the same option still shows selected.
- Check "All Topics"; confirm all three individual topic checkboxes appear
  unchecked and "All Topics" itself stays checked across a re-render (e.g.
  after toggling Study/Exam mode, which re-renders the Home screen). Then
  check one individual topic; confirm "All Topics" becomes unchecked.
- On the quiz screen, confirm question 1 shows Next pinned right with no
  Previous button visible (not shifted left awkwardly), and later questions
  show Previous pinned left / Next pinned right with Home centered below.
- With no saved session, confirm Home shows "Start Quiz" alone plus "View
  Stats" centered below. Start a quiz, click Home mid-quiz, confirm Home now
  shows "Resume Quiz" + "Restart" together (no "Start Quiz") plus "View
  Stats" still centered below.
