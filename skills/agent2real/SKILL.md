---
name: agent2real
description: Validation method for AI-written code that runs on real hardware (robots, drones, vehicles, any real-time edge system). Stops a coding agent from "solving the test" instead of the problem, and decides which changes can be trusted from the bench and which must earn a field test. Use before calling a change done, before a field test, when tuning parameters, and whenever an agent reports "all tests pass" on code that touches sensors, timing, estimation or control.
---

# agent2real — tests pass is where validation starts

A coding agent optimises for the signal you give it. If the signal is "tests green", it will find
the cheapest way to green, and sometimes that means changing the test instead of fixing the code. This is
overfitting, and on hardware it shows up the same way sim-to-real does: fine on the bench, broken
once real sensors and real timing are involved.

The fix is not more tests. It is a **ladder of evidence**: each rung is harder to game than the
one below, and a change climbs only as high as its risk requires.

```
 unit tests ─► real-log replay ─► faults on real logs ─► held-out + placebo ─► target HW replay ─► field
   (cheap, gameable)                                                                    (expensive, honest)
```

## Rule 0 — the agent does not grade its own homework

- Tests, scorer, thresholds and the regression set are **read-only for the agent**. If a diff
  touches them, it goes to separate human review and is never part of the same change as the fix.
- Keep a **held-out set** of logs that the agent never sees while iterating. It is used for the
  verdict only. If a change is tuned against the held-out set, that set is burned: replace it.
- "Agent reports success" is a claim, not evidence. The gate runs the commands itself and reads
  the output.

## Rung 1 — replay real field logs on every change

Synthetic tests check what you thought of. Recorded field data checks what actually happened.

- Replay recorded sensor logs through the **production code path** (same binary, same config, same
  launch), not a test harness that skips the hard parts.
- Replay at **real time with original timestamps**. Running faster than real time hides queue overflow,
  starvation, late messages and races, which are the bugs that reach the field.
- Fix the log set **before** you change the code, and run all of it every time. If a change
  improves one log and was not run on the others, it has not been tested.
- **Every field failure becomes a permanent log in the set.** The set should grow from incidents,
  not from what is convenient to record.
- Report per log: median and **worst case**. A mean across logs hides the one log that broke.

## Rung 2 — inject faults into real logs

Clean logs show the code works on a good day. Inject the faults the field produces into
**real** recordings, so everything except the fault stays real:

| Fault | What it exposes |
|---|---|
| dropped / duplicated / reordered messages | buffering and ordering assumptions |
| timestamp jitter, clock offset, clock jump | two clocks compared as one, `dt <= 0` |
| sensor saturation, stuck values, NaN | missing input validation on the hot path |
| image blur, exposure change, low texture | front-end robustness |
| CPU contention (pin and load cores) | deadline misses that only appear under load |

Place each fault at several points in the log (early, middle, late) and run several seeds.
Results from one fault placement are anecdotes.

## Rung 3 — measure the noise floor and test out of sample

This rung keeps you from fooling yourself.

- **Placebo arm.** Run a change that should do nothing (for example, a parameter scaled by 1.001).
  The spread it produces is your noise band. If an improvement falls inside that band, nothing
  improved.
- **Out of sample.** Tune on one set of logs, seeds and fault placements, then give the verdict
  on a different set. If the result collapses when a threshold moves by 10%, report that it is
  fragile.
- **Mechanism before merge.** Write one falsifiable sentence before the run: "X improves because
  counter Y drops when Z happens." Then log counter Y. If the metric improves but the counter did
  not move, the improvement came from luck or from the test, so reject it.

## Rung 4 — replay on the target hardware

Desktop x86 and the vehicle's ARM board differ in floating point, scheduling, memory, thermals and
library versions. Before the field, replay the same logs **on the target board, or the same SoC in
CI**. A pass on the desktop is required before this rung, but it does not replace it.

## Verdicts — use these words, not "looks good"

| Verdict | Meaning |
|---|---|
| PASS | No new failures on any log, improvement above the placebo band, holds on held-out data |
| MIXED | Helps some logs and hurts others, or flips with config. Name the trade |
| FAIL | Loses logs the baseline keeps, or the worst case gets worse |
| INERT | Identical to baseline. Say whether the code path even triggered |
| VOID | The test could not measure it (wrong build, scorer bug, path not exercised). Re-run it; it is not a result |

**Max 3 iterations per idea**, each with a new written hypothesis. If an idea has no PASS after
three, it is a final FAIL/MIXED. Do not tweak knobs without a hypothesis, because that is how an
agent overfits a test set.

## Which changes go to the field?

Field time is the scarcest resource. Spend it where the bench cannot answer:

**Bench is enough** (rungs 1–4 PASS):
- refactors with byte-identical replay output
- logging, tooling, docs, offline analysis
- algorithm changes whose mechanism the replay fully exercises

**Field required**, even after the bench passes:
- anything on a control loop, failsafe, arming or mode logic
- timing, threading, scheduling, driver or sensor-ingest changes (replay cannot reproduce the real
  interrupt and bus timing)
- anything whose mechanism needs conditions **not in your log set** (new environment, altitude,
  speed, temperature, sensor). Write down the gap, because that gap is the field test's question
- first run on new hardware or a new firmware

Every field test has one written question it answers. After the test, its logs join the replay set
in Rung 1, so the next change is checked against that data on the bench.

## Gate checklist (paste into the PR)

```
[ ] Tests / scorer / log set untouched by this change (or reviewed separately)
[ ] Full log set replayed at real time — per-log worst case reported
[ ] Fault set run at ≥3 placements × ≥3 seeds
[ ] Placebo band measured; improvement exceeds it
[ ] Held-out logs confirm the verdict
[ ] Mechanism hypothesis written before the run; its counter moved
[ ] Target-hardware replay passed
[ ] Verdict: PASS / MIXED / FAIL / INERT / VOID
[ ] Field needed? yes/no — if yes, the one question it answers
```
