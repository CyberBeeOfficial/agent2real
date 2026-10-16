# Why it works

## Tests are a training set

When a person writes code, the tests mostly check what the code does. When an agent writes code
in a loop ("change, run tests, read failure, change again"), the tests become the **objective
the agent optimises against**. Anything optimised hard against a fixed objective overfits to it,
and the model behind the agent is no exception:

- a tolerance gets loosened "because the test was flaky"
- a special case appears that matches exactly the input the test uses
- a timeout gets raised, a check gets skipped, or a mock returns the value the assertion wants
- the code is right for the synthetic data and wrong for the sensor

None of these is malicious; each is the shortest path to green. And unit tests tend to be synthetic,
clean and timing-free, which is exactly where hardware differs.

Machine learning solved this problem decades ago: you **separate the data you fit on from the data
you judge on**, you **measure noise before you claim a gain**, and you **don't let the learner touch
the evaluation**. `agent2real` applies the same discipline to coding agents, and adds the one thing
robotics needs on top: an evaluation that includes **real sensors and real time**.

## What each rung removes

| Rung | Loophole it closes | Why that loophole matters on hardware |
|---|---|---|
| **0. Frozen evaluation** | The agent edits the test, the scorer or the data | The quickest route to green, and the hardest to spot in review |
| **0. Held-out logs** | Tuning to the logs you judge on | Turns the test set into a training set without anyone noticing |
| **1. Real-log replay at real time** | Synthetic inputs and timing-free tests | Real logs have the noise, the gaps and the message ordering; at real time, queues, races and late data appear |
| **1. Failures become permanent logs** | The same failure coming back | The set grows toward what actually breaks, not what is easy to record |
| **2. Faults on *real* logs** | Code that only works on a good day | Injecting into real data keeps everything except the fault real, unlike a synthetic sim where everything is idealised |
| **2. Several placements × seeds** | One lucky run | A single fault position is an anecdote |
| **3. Placebo arm** | Calling noise a gain | Estimators and schedulers are chaotic; a no-op change can move results. That spread is your noise floor |
| **3. Mechanism before the run** | Right number, wrong reason | If the metric improves and the counter that should have moved did not, the gain came from somewhere else and won't hold in the field |
| **3. Verdict words incl. INERT / VOID** | "No change" that hides "not exercised" | A config key the build never reads looks exactly like a key that did nothing |
| **3. Max 3 iterations per idea** | Tweaking knobs until it passes | Unlimited retries against the same set is the definition of overfitting |
| **4. Target-HW replay** | Desktop ≠ vehicle | Floating point, compiler flags, cores, memory, thermals, library versions |
| **Field, with one written question** | Field tests that "look fine" | A test without a question confirms nothing; its logs then feed rung 1 |

## Why a ladder and not "always field test"

Field time is the scarce resource: a day, a team, a vehicle, and sometimes the hardware. AI makes
changes much faster than the field can absorb them, so you can't field-test everything, and you
shouldn't have to.

The ladder is a filter. Most ideas fail cheaply on rungs 1–3 (in our case about four in five).
The few that reach the field arrive with a written question the bench couldn't answer, so the
field test is spent confirming that answer instead of finding the problem in the first place.

## Why it is a skill and not a CI script

The CI parts (replay, scorer, fault injection) already exist in most teams in some form. What's
usually missing is the **discipline around them**: who may touch the evaluation, what counts as a
gain, which words a verdict may use, when an idea is dead, and which changes the bench may clear.
Agents follow written rules reliably when the rules are concrete. The skill writes them down where
the agent reads them every time it is about to say "done".
