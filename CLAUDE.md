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
