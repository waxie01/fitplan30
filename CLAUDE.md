# FitPlan 30 — Dev Context

## Project structure

- `index.html` — David's app (served at root `/`)
- `krissy/index.html` — Krissy's app (served at `/krissy`)
- `fitplan30.html` / `fitplan-krissy.html` — older legacy versions, not deployed
- `vercel.json` — static hosting config; pushes to `main` on GitHub auto-deploy via Vercel
- `.claude/launch.json` — preview server config (serves project root on port 4321 via `npx serve`)

## Deployment

Repo: https://github.com/waxie01/fitplan30  
Auth: `gh auth setup-git` wires the `gh` CLI token for HTTPS pushes (account: waxie01).  
Vercel auto-deploys on every push to `main` — no manual step needed.

---

## Session log — June 4, 2026

### Bug fix: weight input tap resetting last exercise (`index.html` + `krissy/index.html`)

**Problem:** After logging the last set of a weighted exercise, the weight entry row appeared. If the user tapped anywhere on the card body (not the Save button), `blur` fired first (hiding the weight row), then `onclick` fired on the card — but the weight row was already hidden so the existing `.weight-row` guard missed it, and `tapLog` reset the exercise to 0 sets.

**Fix:** Added `const _wRowHiddenAt = {}` (module-level timestamp map). `hideWeightRow` now records `_wRowHiddenAt[exId] = Date.now()`. The card `onclick` skips `tapLog` if the weight row was hidden within the last 300ms.

---

### Bug fix: header date not matching title when workout is "Tomorrow" (`index.html` + `krissy/index.html`)

**Problem:** The date line under the title always showed today's date (`new Date()`), even when the title read "Tomorrow" or a future date.

**Fix:** Moved the `_pDt` / `_diff` calculation before the `hd-date` assignment. Date now uses `formatDateFull(_diff <= 0 ? new Date() : _pDt)` — shows the plan day's actual date when it's in the future.

---

### Travel workouts: bodyweight-only Days 11–15 (`index.html` only — David's app)

David is away from the gym June 4–8 (Days 11–15 of his plan). Day 14 is already a rest day.

Replaced gym exercises with bodyweight-only, max 2 sets:

| Day | Original | Travel version |
|-----|----------|----------------|
| 11 | DB Bench Press, DB Flyes, Bicep Curls, Tricep Dips | Push-ups 2×15, Diamond Push-ups 2×12, Tricep Dips 2×12, Incline Push-ups 2×12 |
| 12 | Plank, Bicycle Crunches, Leg Raises, Russian Twists (3 sets each) | Same exercises, reduced to 2 sets each |
| 13 | Elliptical, Concentration Curls, Shoulder Press | Mountain Climbers 2×30, Incline Push-ups 2×15, Plank 2×45s |
| 14 | Rest Day | (unchanged) |
| 15 | DB Bench Press, DB Flyes, Incline Push-ups, Bicep Curls, Tricep Extensions | Push-ups 2×15, Incline Push-ups 2×15, Diamond Push-ups 2×12, Tricep Dips 2×15 |

---

### Fix: weight entry prompt only for dumbbell exercises (`index.html` + `krissy/index.html`)

**Problem:** `isWeightedEx` used a heuristic (`reps` doesn't end in `'s'`) that incorrectly triggered the custom weight entry row for bodyweight exercises like push-ups, tricep dips, leg raises, mountain climbers, etc.

**Fix:** Replaced with an explicit allowlist:
```js
const WEIGHTED_EXS = new Set([
  'db-bench-press', 'db-flyes', 'bicep-curls', 'hammer-curls',
  'concentration-curls', 'tricep-extensions', 'shoulder-press', 'lateral-raises'
]);
function isWeightedEx(exId) { return WEIGHTED_EXS.has(exId); }
```

---

### Fix: suppress rest timer when weight entry opens on last set (`index.html` + `krissy/index.html`)

**Problem:** When the final set of a weighted exercise was logged, the rest timer started simultaneously with the weight entry row appearing. Tapping Skip on the timer closed the weight entry with no way to reopen it.

**Fix:** In `tapLog`, compute `_lastWeightedSet = p.sets[id] >= ex.sets && isWeightedEx(id)` and only call `startRestTimer()` when that's false. Timer still fires normally for intermediate sets and all bodyweight exercises.

---

### Design: exercise card icon changed from kebab (⋮) to circled ⓘ (`index.html` + `krissy/index.html`)

The three-dot menu icon implied actions (edit/delete/share). The button actually opens an info sheet (exercise description, how-to, video) with a secondary "Log Set" capability. Circled ⓘ better represents the primary intent. `aria-label` updated to "Exercise info".

---

## UX backlog — June 5, 2026

Full review of the app experience. Items below are prioritized roughly by impact. Each is a self-contained fix or feature to be built in a future session.

---

### Bug fix: modal "Log Set" still fires rest timer on last weighted set

**Problem:** The fix that suppresses the rest timer on the last set of a weighted exercise only applies to the card-tap path (`tapLog`). `logSetFromModal()` always calls `startRestTimer()`, so the weight entry row + rest timer conflict can still be triggered via the exercise modal.

**Fix:** Apply the same `_lastWeightedSet` guard in `logSetFromModal()` — check if `p.sets[modalEx] >= ex.sets && isWeightedEx(modalEx)` after incrementing, and skip `startRestTimer()` / show weight row accordingly.

**Scope:** `index.html` + `krissy/index.html`

---

### Bug fix / UX: accidental tap-to-reset on completed exercise cards

**Problem:** Tapping a completed card (all sets done, green border, "✓ Done") silently resets it to 0 sets with no warning, no undo, and no visual affordance that this is even possible. A user who finishes their last set and taps the card out of habit loses all their logged work.

**Fix options (pick one):**
- Require a **long-press** to reset; a normal tap on a complete card does nothing (or shows a brief "hold to undo" hint).
- Show a **confirmation sheet**: "Reset this exercise?" with Cancel / Reset.
- Implement **swipe-left to undo last set** as the reset gesture, making it intentional and reversible one-set-at-a-time.

**Scope:** `index.html` + `krissy/index.html`

---

### Feature: built-in countdown for timed exercises

**Problem:** Exercises with time-based reps (Plank × 45s, Mountain Climbers × 30s) show the duration as static text. The user has to self-time, guess, or stare at the phone clock.

**Feature:** When a timed exercise card is tapped to log a set, start a countdown timer matching the exercise's duration (parse `reps` for a trailing `s` — e.g. `'45s'` → 45 seconds). Reuse or extend the existing rest timer pill UI. Timer fires, counts down, then transitions to the rest timer (or ends). The log is recorded when the countdown starts (or when it ends — decide which feels better).

**Scope:** `index.html` + `krissy/index.html`

---

### Feature: adjustable rest timer duration

**Problem:** The rest timer is fixed at 30 seconds. That's appropriate for conditioning circuits but too short for strength sets (conventional rest is 60–90 seconds). David's plan has both types.

**Feature:** Add a rest duration setting in the Settings sheet — e.g. a segmented control: 30s / 60s / 90s / 2 min. Persist in `localStorage`. Optionally, default weighted exercises to 60s and bodyweight to 30s automatically.

**Scope:** `index.html` + `krissy/index.html`

---

### Feature: "Day Complete" as a bottom sheet, not a scroll-to banner

**Problem:** The completion banner ("Day Complete! 🎉") appears below the exercise list, requiring the user to scroll down after finishing the last set. The payoff is buried.

**Feature:** Replace the inline banner with a bottom sheet modal (same pattern as exercise modal / day drawer) that slides up automatically when all sets are logged. Confetti still fires. "Start Day X" button is in the sheet. Dismissing the sheet returns to the (now all-green) exercise list.

**Scope:** `index.html` + `krissy/index.html`

---

### Feature: "Tomorrow's Workout" preview on Today screen

**Feature:** A small preview card at the bottom of the Today view (below the exercise list, above the tab bar padding) showing the next plan day's workout type and exercise list. Label: "Tomorrow · Core & Abs" with the exercises listed. On rest days, say "Tomorrow · Rest Day 😌". Tapping it could navigate to a read-only day drawer for that day.

Helps users pace today's effort, builds anticipation, and makes the plan feel like a program rather than a one-day-at-a-time checklist.

**Scope:** `index.html` + `krissy/index.html`

---

### Feature: personal records (PRs) for weighted exercises

**Problem:** There is already a `.rest-pill.pr` CSS class (gold background, amber glow box-shadow) in the stylesheet — PR detection was anticipated but never wired up.

**Feature:** After saving a weight for a weighted exercise, compare it against the highest weight ever logged for that exercise across all completed days (`S.progress`). If it's a new high, show the PR pill: "New PR! 🏆 You lifted X lbs." Persist per-exercise PR history in state.

**Scope:** `index.html` + `krissy/index.html` (Krissy's version also has weighted exercises)

---

### Feature: week-complete summary on rest days

**Feature:** When the current day is a rest day (Days 7, 14, 21), replace the static "Rest & Recover" card with a week-in-review summary: total sets logged that week, days completed, current streak, and a motivating message. Turns a passive rest day into a milestone moment.

**Scope:** `index.html` + `krissy/index.html`

---

### Feature: per-day notes field

**Feature:** A simple single-line text input at the bottom of each workout day (below the exercise list, above the completion banner) — "Add a note…" placeholder. Saved to `S.progress[day].note`. Surfaced in the calendar day-detail drawer when reviewing past days. Over 30 days this becomes a personal workout journal.

**Scope:** `index.html` + `krissy/index.html`
