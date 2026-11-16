# Implementation Plan: Habit Streak Tracker

## Overview

Deliver a zero-dependency, three-file SPA (`index.html`, `style.css`, `app.js`) placed directly in the workspace root. Implementation follows the layered architecture from the design: `Habit_Store` (localStorage adapter) → `Streak_Engine` (pure functions) → `UI` (DOM + event handlers). Tasks are ordered so each layer is built, wired, and tested before the next layer depends on it.

## Tasks

- [x] 1. Scaffold static files and shared utilities
  - [x] 1.1 Create `index.html` with page skeleton
    - Add `<!DOCTYPE html>`, `<head>` (charset, viewport, title), and `<body>` with the top-level landmark regions: header with the add-habit form, main habit list container, and an empty-state placeholder
    - Link `style.css` in `<head>` and `app.js` with `defer` at the bottom of `<body>`
    - Add the toast container element (`<div id="toast-container">`) used by `UI.showToast`
    - _Requirements: 1.1, 2.4, 2.5_

  - [x] 1.2 Create `style.css` with layout and component styles
    - CSS reset and base typography
    - Layout: centered single-column, max-width container
    - Habit card styles (name, streak count, checkbox, delete button)
    - Active-streak indicator (flame icon or highlight color) and broken/zero-streak visual state
    - Toast notification styles for `info`, `warn`, and `error` levels with a fade-out animation
    - Inline validation error styles for the add-habit form
    - _Requirements: 2.1, 2.2, 2.3_

  - [x] 1.3 Create `app.js` with the `todayString` utility and module stubs
    - Write the `todayString()` function using local timezone arithmetic (`getFullYear`, `getMonth`, `getDate`)
    - Stub out the three IIFE modules — `Habit_Store`, `Streak_Engine`, `UI` — as empty objects so later tasks fill them in without touching file structure
    - Write a simple `uuidv4()` helper (crypto.randomUUID with Math.random fallback)
    - _Requirements: 4.1 (timezone correctness), 5.1_

- [ ] 2. Implement `Streak_Engine`
  - [-] 2.1 Implement `Streak_Engine.compute` and `Streak_Engine.isActive`
    - Implement `compute(completionDates, today)` following the reference algorithm in the design: deduplicate and sort ascending, check if `mostRecent` is ≥ 3 days before `today` (return `{count:0, isActive:false}`), then walk backward counting runs with gap < 3
    - Implement `isActive(completionDates, today)` as a thin wrapper around `compute`
    - Implement the `daysBetween(a, b)` helper (parse `"YYYY-MM-DD"` strings to midnight-local `Date` objects, return absolute diff in whole days)
    - _Requirements: 4.1, 4.2, 4.3, 4.4, 4.5, 4.6, 4.7_

  - [ ]* 2.2 Write property tests for `Streak_Engine` (Properties 4–7)
    - Create `tests/streak-engine.test.html` with fast-check loaded via `<script>` CDN tag
    - **Property 4: Mercy window — gaps ≤ 2 do not break the streak** — generator: custom arbitrary building date arrays where every consecutive gap is 1 or 2 days; assert `compute(dates, today).count === dates.length`
    - **Validates: Requirements 4.1, 4.2**
    - **Property 5: Gap ≥ 3 resets the streak** — generator: custom arbitrary injecting a gap of 3–7 days; assert count equals only the completions after the last such gap
    - **Validates: Requirements 4.3**
    - **Property 6: Active/Broken classification** — generator: `fc.tuple(fc.array(...), fc.date())`; assert `isActive` ↔ `daysBetween(mostRecent, today) ≤ 2`
    - **Validates: Requirements 4.4, 4.5**
    - **Property 7: Empty list returns zero** — generator: `fc.constant([])`; assert `compute([], today)` returns `{count:0, isActive:false}`
    - **Validates: Requirements 4.7**
    - Each property runs ≥ 100 iterations; tag each with `// Feature: habit-streak-tracker, Property N: ...`

- [ ] 3. Implement `Habit_Store`
  - [-] 3.1 Implement `Habit_Store.load` and `Habit_Store.save`
    - `load()`: read `localStorage.getItem("habit_tracker_data")`, `JSON.parse` inside try/catch — return `null` on any failure
    - `save(habitList)`: `JSON.stringify` and `localStorage.setItem` inside try/catch — return `{ok:true}` on success, `{ok:false, error}` on failure (quota exceeded, SecurityError, etc.)
    - _Requirements: 5.1, 5.2, 5.3, 5.4, 5.5_

  - [~] 3.2 Implement `Habit_Store.addHabit`, `Habit_Store.deleteHabit`, and `Habit_Store.addCompletion`
    - `addHabit(name)`: generate UUID, build `Habit` object with `createdAt: todayString()` and empty `completions`, push to in-memory list, call `save()`
    - `deleteHabit(id)`: filter the habit out of the in-memory list, call `save()`; return `{ok:false}` and do not mutate memory if `save()` fails
    - `addCompletion(habitId, dateString)`: find the habit, skip if `dateString` already in `completions` (idempotency), otherwise push and sort, call `save()`; return `{ok:false}` without mutating if `save()` fails
    - _Requirements: 1.5, 3.1, 3.3, 6.4, 8 (Property 8), 9 (Property 9)_

  - [ ]* 3.3 Write property tests for `Habit_Store` (Properties 3, 8, 9)
    - Create `tests/habit-store.test.html` with fast-check CDN
    - **Property 3: Persistence round-trip** — generator: `fc.array(habitArbitrary)`; call `save(list)` then `load()`, assert deep equality
    - **Validates: Requirements 5.1, 5.2**
    - **Property 8: Adding a completion grows the array by exactly 1** — generator: habit + date not already in completions; assert array length increases by 1 and date appears exactly once
    - **Validates: Requirements 3.1**
    - **Property 9: Duplicate completion is idempotent** — generator: habit with ≥ 1 existing completion; call `addCompletion` with an already-present date; assert array unchanged
    - **Validates: Requirements 3.3**

- [~] 4. Checkpoint — `Streak_Engine` and `Habit_Store` verified
  - Ensure all property tests pass and manually verify `todayString()` output in browser console. Ask the user if any questions arise.

- [ ] 5. Implement `UI` — rendering and add-habit flow
  - [~] 5.1 Implement `UI.init`, `UI.renderHabitList`, and `UI.renderHabitCard`
    - `UI.init()`: call `Habit_Store.load()`; if `null`, initialize empty list and call `showToast("Could not load saved data", "warn")`; set `lastSessionDate`; call `renderHabitList()`; wire the add-habit form submit handler; start the midnight-rollover `setInterval` (1-minute polling)
    - `UI.renderHabitList()`: clear the list container, iterate habits, append `renderHabitCard(habit)` for each, or show the empty-state element when the list is empty
    - `UI.renderHabitCard(habit)`: build an `<article>` card with habit name, streak count (with active-streak class when `StreakResult.isActive`), a checkbox (checked+disabled if already completed today), and a delete button; return the element
    - Bind `DOMContentLoaded` → `UI.init()`
    - _Requirements: 2.1, 2.2, 2.3, 2.4, 2.5_

  - [~] 5.2 Implement add-habit form validation and submission
    - On form submit: trim the input value; show `showValidationError` and abort if empty/whitespace (Req 1.3), if length > 100 chars (Req 1.6), or if name exists case-insensitively (Req 1.4)
    - On successful validation: call `Habit_Store.addHabit(trimmedName)`, call `renderHabitList()`, clear and re-focus the input
    - Implement `UI.showValidationError(field, message)` and `UI.clearValidationError(field)` to toggle an error `<span>` adjacent to the named field
    - _Requirements: 1.1, 1.2, 1.3, 1.4, 1.5, 1.6_

  - [ ]* 5.3 Write property tests for input validation (Properties 1, 2)
    - Add to `tests/ui-validation.test.html` (fast-check CDN)
    - **Property 1: Whitespace-only names are rejected** — generator: `fc.stringOf(fc.constantFrom(' ','\t','\n'))`; call the validation logic, assert habit list unchanged
    - **Validates: Requirements 1.3**
    - **Property 2: Duplicate names are rejected (case-insensitive)** — generator: `fc.string()` mapped to both lower and upper case variants; assert validation rejects the second addition
    - **Validates: Requirements 1.4**

- [ ] 6. Implement `UI` — completion, deletion, and daily reset
  - [~] 6.1 Implement completion checkbox handler
    - On checkbox change: call `Habit_Store.addCompletion(habitId, todayString())`; on success re-render the card's streak count and set checkbox to checked+disabled within 500 ms; on failure call `showToast("Save failed", "error")` and revert checkbox to unchecked
    - _Requirements: 3.1, 3.2, 3.4, 3.5_

  - [~] 6.2 Implement delete button handler with confirmation
    - On delete button click: call `window.confirm("Delete this habit and all its data?")` (or a custom inline modal); on cancel do nothing; on confirm call `Habit_Store.deleteHabit(id)`; on success remove the card from the DOM within 300 ms; on failure call `showToast("Deletion failed", "error")` and leave card intact
    - _Requirements: 6.1, 6.2, 6.3, 6.4, 6.5, 6.6_

  - [~] 6.3 Implement midnight-rollover detection
    - In the `setInterval` callback: call `todayString()`, compare to stored `lastSessionDate`; if different, update `lastSessionDate`, call `renderHabitList()` to redraw all cards with unchecked checkboxes (completion history untouched)
    - Also trigger the same logic on `visibilitychange` (tab regains focus) for reliability
    - _Requirements: 7.1, 7.2, 7.3_

  - [~] 6.4 Implement `UI.showToast`
    - Append a `<div class="toast toast--{level}">` to `#toast-container`, auto-remove after 4 seconds via `setTimeout`; support `info`, `warn`, and `error` levels
    - _Requirements: 5.3, 5.5, 3.5, 6.6_

  - [ ]* 6.5 Write unit tests for UI behaviors
    - Create `tests/ui.test.html` with a minimal DOM fixture and a simple `assert` wrapper
    - Test: empty-state message shown when habit list is empty
    - Test: active-streak class present when `isActive === true`, absent when false
    - Test: checkbox is checked+disabled for today's completion, unchecked for other habits
    - Test: validation error element appears/disappears correctly for each rule
    - _Requirements: 2.1, 2.2, 2.3, 2.5, 1.3, 1.4, 1.6_

- [~] 7. Final checkpoint — full integration verified
  - Ensure all property tests and unit tests pass. Open `index.html` directly in a browser (no server needed) and confirm: add habit → streak appears → check completion → streak increments → delete habit with confirmation. Ask the user if any questions arise.

## Notes

- Tasks marked with `*` are optional and can be skipped for a faster MVP
- Each task references specific requirements for traceability
- Property tests require fast-check loaded via CDN `<script>` tag in each test HTML file — no npm or build step needed
- All date arithmetic uses local timezone (`getFullYear/getMonth/getDate`), never `toISOString()`
- The `setInterval` polling period is 60 000 ms (1 minute); the `visibilitychange` listener provides tighter rollover detection when the user returns to the tab
- Error-handling paths (localStorage quota, parse failure, save failure) must show toasts per the design's error-handling table

## Task Dependency Graph

```json
{
  "waves": [
    { "id": 0, "tasks": ["1.1", "1.2", "1.3"] },
    { "id": 1, "tasks": ["2.1", "3.1"] },
    { "id": 2, "tasks": ["2.2", "3.2"] },
    { "id": 3, "tasks": ["3.3", "5.1"] },
    { "id": 4, "tasks": ["5.2", "6.1", "6.2", "6.3", "6.4"] },
    { "id": 5, "tasks": ["5.3", "6.5"] }
  ]
}
```
