# Requirements Document

## Introduction

A single-page web application that lets users build and track daily habits using a Duolingo-style streak system. Users can add named habits, mark them complete each day, and watch their streak count grow. A mercy window allows up to 2 consecutive missed days before a streak is broken, so a user who misses day 1 and day 2 can still resume on day 3 without losing progress — mirroring Duolingo's streak-freeze concept.

## Glossary

- **Habit**: A named recurring activity the user wants to perform daily.
- **Completion**: A record that a specific Habit was marked done on a specific calendar date.
- **Streak**: The count of the longest unbroken sequence of days on which a Habit was completed, subject to the Mercy Window.
- **Mercy Window**: A grace period of up to 2 consecutive missed days that does not break the Streak. If a Habit is not completed for 3 or more consecutive days, the Streak resets to 0.
- **Active Streak**: A Streak that is still eligible to continue; the most recent Completion falls within the Mercy Window of today.
- **Broken Streak**: A Streak where the most recent Completion is more than 2 days before today, causing the count to reset to 0.
- **Streak_Engine**: The subsystem responsible for computing and updating streak values.
- **Habit_Store**: The subsystem responsible for persisting habit and completion data in browser localStorage.
- **UI**: The visual interface rendered in the browser.

---

## Requirements

### Requirement 1: Add a Habit

**User Story:** As a user, I want to add a new habit by name, so that I can start tracking it daily.

#### Acceptance Criteria

1. THE UI SHALL display an input field and a submit button for entering a new habit name.
2. WHEN the user submits a habit name that is non-empty after trimming leading and trailing whitespace, THE UI SHALL store the trimmed name, add the habit to the habit list, and clear the input field.
3. IF the user submits an input that is empty or contains only whitespace, THEN THE UI SHALL display an inline validation message indicating that the habit name cannot be blank and SHALL NOT add a habit.
4. IF a habit with the same trimmed name (case-insensitive) already exists, THEN THE UI SHALL display an inline validation message indicating the name is already in use and SHALL NOT add a second habit.
5. WHEN a new habit is added, THE Habit_Store SHALL persist the habit so it survives a page reload.
6. IF the user submits a habit name whose trimmed length exceeds 100 characters, THEN THE UI SHALL display an inline validation message indicating the name is too long and SHALL NOT add a habit.

---

### Requirement 2: Display Habit List

**User Story:** As a user, I want to see all my habits in a list, so that I can review and act on them.

#### Acceptance Criteria

1. THE UI SHALL display each saved habit as a distinct card showing the habit name, the current streak count, and a completion checkbox for today.
2. WHILE a habit has an Active Streak, THE UI SHALL display the streak count with a visual indicator (such as a flame icon or highlighted color) that is absent on habits without an Active Streak.
3. WHILE a habit has a Broken Streak, THE UI SHALL display the streak count as 0.
4. WHEN the page loads, THE UI SHALL restore and display all habits and their streak states from the Habit_Store.
5. IF the Habit_Store contains no habits, THEN THE UI SHALL display an empty-state message indicating that no habits have been added yet.

---

### Requirement 3: Complete a Habit

**User Story:** As a user, I want to mark a habit as done for today, so that my streak advances.

#### Acceptance Criteria

1. WHEN the user checks the completion checkbox for a habit, THE Habit_Store SHALL record a Completion for that habit using the user's local calendar date as "today".
2. WHEN a Completion is recorded for today, THE Streak_Engine SHALL recalculate the habit's Streak count by incrementing it by 1 if the previous Completion was recorded on the immediately preceding calendar day, or resetting it to 1 if no Completion exists for the preceding calendar day.
3. WHILE a habit has already been completed today, THE UI SHALL show the completion checkbox in a checked and disabled state to prevent duplicate completions.
4. WHEN the user checks the completion checkbox, THE UI SHALL update the displayed streak count within 500 milliseconds without a page reload.
5. IF the Habit_Store fails to record the Completion, THEN THE UI SHALL display an error message indicating the save failed and leave the checkbox in its previous unchecked state.

---

### Requirement 4: Streak Calculation with Mercy Window

**User Story:** As a user, I want my streak to survive up to 2 consecutive missed days, so that a short lapse does not erase my progress.

#### Acceptance Criteria

1. WHEN calculating a Streak, THE Streak_Engine SHALL count each calendar day, evaluated in the user's local timezone, on which at least one Completion exists as contributing +1 to the streak count.
2. WHEN calculating a Streak, THE Streak_Engine SHALL treat a gap of 1 or 2 consecutive days with no Completion as a continuation of the streak (the gap days do not add to the count but do not break it).
3. WHEN calculating a Streak, IF the gap between two adjacent Completions is 3 or more calendar days, THEN THE Streak_Engine SHALL treat all Completions before that gap as belonging to a prior broken streak and SHALL begin a new streak count of 0 starting from the first Completion after the gap.
4. IF the most recent Completion date is 0, 1, or 2 calendar days before today's date, THEN THE Streak_Engine SHALL classify the Streak as Active.
5. IF the most recent Completion date is 3 or more calendar days before today's date, THEN THE Streak_Engine SHALL classify the Streak as Broken and SHALL return a streak count of 0.
6. THE Streak_Engine SHALL produce a streak count equal to the number of distinct Completion days in the current unbroken run as defined by criteria 2 and 3, where "current" means the run whose most recent Completion day is within 2 calendar days of today.
7. IF no Completions exist for a habit, THEN THE Streak_Engine SHALL return a streak count of 0 and SHALL classify the Streak as Broken.

---

### Requirement 5: Persist Data Across Sessions

**User Story:** As a user, I want my habits and streaks to be saved in the browser, so that I do not lose my progress when I close and reopen the page.

#### Acceptance Criteria

1. THE Habit_Store SHALL save all habit names and their Completion records to browser localStorage under a key prefixed with "habit_tracker_".
2. WHEN the page loads, THE Habit_Store SHALL read all habit and Completion data from localStorage and provide it to the UI before the first render completes.
3. IF localStorage is unavailable or contains data that cannot be parsed as valid habit and Completion records, THEN THE Habit_Store SHALL initialize an empty habit list and SHALL display a non-blocking warning message to the user indicating that saved data could not be loaded.
4. WHEN any habit name or Completion record is created, updated, or deleted, THE Habit_Store SHALL write the full updated habit list to localStorage within 500 milliseconds of the change.
5. IF the data stored in localStorage exceeds the browser's localStorage quota, THEN THE Habit_Store SHALL display an error message to the user indicating that the save failed and SHALL retain the previous successfully saved state in localStorage.

---

### Requirement 6: Delete a Habit

**User Story:** As a user, I want to remove a habit I no longer track, so that my list stays relevant.

#### Acceptance Criteria

1. THE UI SHALL display a delete button on each habit card.
2. WHEN the user activates the delete button, THE UI SHALL display a confirmation prompt before removing the habit.
3. WHEN the user cancels the confirmation prompt, THE UI SHALL dismiss the prompt and leave the habit and its Completion records unchanged.
4. WHEN the user confirms deletion, THE Habit_Store SHALL remove the habit and all Completion records associated with that habit's ID from localStorage.
5. WHEN the user confirms deletion, THE UI SHALL remove the habit card from the list within 300 milliseconds.
6. IF the localStorage removal operation fails, THEN THE UI SHALL display an error message indicating the deletion could not be completed and retain the habit card in the list.

---

### Requirement 7: Daily Reset of Completion State

**User Story:** As a user, I want each new calendar day to show habits as uncompleted, so that I can mark them done again each day.

#### Acceptance Criteria

1. WHEN the page loads and the current local calendar date differs from the date of the user's last recorded session, THE UI SHALL display all habit checkboxes in the unchecked state.
2. WHILE the page is open, WHEN the user's local calendar date advances past midnight, THE UI SHALL update all habit checkboxes to the unchecked state without requiring a page reload.
3. WHEN the daily reset occurs, THE UI SHALL preserve all completion records from prior dates so that historical completion data remains unaffected.
