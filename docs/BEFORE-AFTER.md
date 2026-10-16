# Before / after

These are real cases from one release cycle of CyberBee's GNSS-denied navigation stack (visual-inertial
estimation for drones). Coding agents proposed, implemented and ran most of the experiments. Flights
are anonymised, and the only figures given are counts of ideas.

"Before" is what the cheap signal said: tests green, plus at most one replay of a clean log.
"After" is what the gate found.

## The aggregate

| | Ideas |
|---|---|
| Went through the gate | **106** |
| PASS | 20 |
| FAIL | 62 |
| Changed nothing (INERT / null) | 11 |
| MIXED (helped some cases, hurt others) | 7 |
| VOID: the test couldn't measure them | 5 |
| Still iterating | 1 |

**About one idea in five survived.** Each of the other 86 could have been called "tests pass". The gate
kept them from reaching the field.

## The cases

### 1. Held-out logs caught what the tuning set hid (Rung 0, 3)

- **Before:** a bundle of estimator fixes recovered many of the baseline's failures on the tuning
  flights and added no new ones.
- **After:** on held-out flights the bundle produced **new** failures the baseline didn't have.
  Leave-one-out showed that one member caused them. It was removed and the rest of the bundle shipped.
  A later candidate showed the same pattern and was also kept out.
- **Lesson:** a set you tuned on can't also give the verdict.

### 2. "It did nothing" really meant "it never ran" (Rung 3: VOID vs INERT)

- **Before:** a replay on the production node reported that a set of config changes had no effect.
  The obvious conclusion was that the change was useless.
- **After:** the branch under test didn't contain the code that reads those keys, so they were
  silently ignored. The verdict was VOID, not INERT. The same changes were the biggest gain in the
  validated stack.
- **Lesson:** before you call a result "no effect", check that the code path actually fired.

### 3. The replay harness itself was wrong (Rung 1)

- **Before:** a series of replays looked consistent and gave stable conclusions.
- **After:** a topic-name mismatch meant those replays had been running **without the barometer**. With
  the barometer restored, the baseline changed, and the earlier conclusions were marked stale.
- **Lesson:** the evaluation needs checks too. Identity checks (the same input should give the same
  output, and the expected sensors should be present) run before anyone trusts a number.

### 4. Faults injected into real logs found crashes that clean logs never hit (Rung 2)

- **Before:** clean replays passed.
- **After:** frame-drop injection crashed the estimator when the tracked features fell to almost
  zero. Separately, existing recovery code crashed on an empty matrix in the solver, and only under
  injected faults. Both were fixed before any field exposure.
- **Lesson:** a clean log only tests the good-day case.

### 5. Passing simulation is not the same as being real-time safe (Rung 4 + review)

- **Before:** an agent-written estimator feature ran in simulation.
- **After:** review found **8 blockers**, including a heap allocation on the hot path and missing
  unit tests. Its experiments were put on hold until the fixes were rebuilt and re-verified.
- **Lesson:** simulation can't see your deadline. The hot path needs its own review.

### 6. The desktop couldn't answer a timing question (Rung 4)

- **Before:** a CPU-limited desktop replay was expected to reproduce the device's timing behaviour.
- **After:** it didn't match the device. The question was moved to a target-board run, which
  became a release blocker.
- **Lesson:** only the target hardware can answer timing questions.

### 7. A "regression" turned out to be noise (Rung 3: placebo)

- **Before:** one cell looked worse after a change.
- **After:** a significance test against the placebo arm didn't support it (it was a knife-edge
  case, where tiny no-op perturbations flip the outcome). It wasn't counted as a loss or as a gain.
- **Lesson:** measure the noise floor before you call anything a change.

### 8. The fix for the faulty case broke the clean case (Rung 2 + 1)

- **Before:** a feature-loss trip helped on fault scenarios.
- **After:** with the trip on, it fired on a **clean** flight, and results under faults got worse,
  not better. It was left off until a proper recovery path existed.
- **Lesson:** run the whole set, clean logs and faulty logs, on every change.

### 9. The same mechanism showed up in both sim and real node (what PASS looks like)

- One initialisation fix (restore the bias estimate after a rejected attempt) passed in simulation.
  Separately, a replay on the production node with real field logs showed the same improvement,
  with the same mechanism, and it was **never worse** on any flight.
- That's the bar: independent evidence on separate rungs, the mechanism confirmed, and no new
  failures. It went to the top of the ship list.

## What changed for the team

| | Without the gate | With the gate |
|---|---|---|
| Signal at "done" | Tests green | A verdict word plus a written mechanism, on real logs |
| Ideas reaching the field | Everything that compiled and passed | The ones that earned it, each with one written question |
| Field test outcome | Where problems were discovered | Where one bench answer was confirmed |
| Field failures | Fixed once | Turned into a permanent replay log |
| Arguments about results | "It looks better" | Placebo band, held-out set, the counter moved or it didn't |
