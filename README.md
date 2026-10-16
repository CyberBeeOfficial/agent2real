# agent2real

**sim2real for coding agents.** AI-written code passes every test. Then it runs on real hardware and breaks.
`agent2real` makes that code earn the real world, one rung at a time.

`agent2real` is a skill for coding agents (Claude Code, or any agent that reads a markdown skill)
working on robots, drones and other real-time edge systems. It stops the agent from making the test
pass instead of fixing the code, and it decides which changes the bench can clear and which need a
field test.

It comes out of how [CyberBee](https://cyberbee.ai) validates the GNSS-denied navigation software
that its coding agents help write. We use it every day.

## The problem

A coding agent optimises for the signal you give it. If that signal is "tests green", it will find
the cheapest way to green, and sometimes that means changing the test instead of fixing the code.

This is overfitting, and it is the same gap robotics already knows as **sim2real**. Here the gap is
between the agent and the real world, so we call it **agent2real**. On hardware it plays out the same way: everything works on the bench until
real sensors, real timing and real faults are involved. And on a drone, a bad change usually shows
up as a crash rather than on a dashboard.

## The method: a ladder of evidence

```
 unit tests ─► real-log replay ─► faults on real logs ─► held-out + placebo ─► target HW replay ─► field
   (cheap, gameable)                                                                    (expensive, honest)
```

Each rung is harder to game than the one below it, and a change goes only as high as its risk
requires.

| Rung | What it does | What it catches |
|---|---|---|
| 0. Frozen evaluation | Tests, scorer and log set are read-only for the agent; the final verdict comes from logs it never saw | Making the test pass without fixing the code |
| 1. Real-log replay | Recorded field data, run through the production code path at real time | What nobody thought to write a test for |
| 2. Faults on real logs | Drops, clock jumps, blur, saturation, all injected into *real* recordings | Code that only survives a good day |
| 3. Placebo + held-out | A no-op change measures the noise floor, and a written reason for each change says which counter should move | "Improvements" that are noise or come from tuning on the test set |
| 4. Target-HW replay | The same logs on the vehicle's board | Floating-point, scheduling and timing differences from the desktop |
| Field | Only for questions the bench can't answer; each test answers one written question | What the log set doesn't contain yet |

Every field test's logs go back into rung 1, so the next change is checked against that data on the bench.

- **Why it works:** [docs/WHY-IT-WORKS.md](docs/WHY-IT-WORKS.md)
- **How it fits into an agent loop:** [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)
- **What it caught for us, before vs after:** [docs/BEFORE-AFTER.md](docs/BEFORE-AFTER.md)

## Before / after

These are real cases from one release cycle of our navigation stack. They're anonymised, and the
full list is in [BEFORE-AFTER.md](docs/BEFORE-AFTER.md).

| What the cheap check said | What the gate found |
|---|---|
| A bundle of fixes: many failures recovered, no new ones on the tuning flights | New failures on **held-out** flights; one member caused them and was removed |
| A config change on the real node "did nothing" | The build never read those keys, so the result was **VOID**, not INERT |
| A new feature passed simulation | Code review found **heap allocation on the hot path**, among 8 blockers |
| Clean replays passed | **Frame-drop injection** crashed the estimator, and the crash never appeared on a clean log |
| A difference that looked like a regression | Inside the **placebo** noise band, so it wasn't counted either way |
| Desktop replay settled a timing question | It didn't match the device; only the **target board** could answer it |

Across the cycle, **106 ideas** went through the gate. **About 1 in 5 passed.** The rest failed,
changed nothing, traded one case for another, or couldn't be measured. Without the gate, every one
of those would have arrived at the field labelled "tests pass".

## Install

Claude Code:

```bash
git clone https://github.com/CyberBeeOfficial/agent2real.git
cp -r agent2real/skills/agent2real ~/.claude/skills/
```

The agent then applies the skill whenever it's about to call hardware-facing work done. Other agents:
put `skills/agent2real/SKILL.md` in the agent's instructions or rules file.

The skill describes a method, so there's nothing to run. It works with whatever replay tool, scorer
and log format you already have.

## What you need to bring

- A set of recorded field logs (rosbags, MCAP, ULog, CSV) and a way to replay them through your real code
- A scorer that the agent can't edit
- A short list of fault injectors for your sensors
- For rung 4: one target board, or the same SoC in CI

## Contributing

Issues and PRs are welcome, especially other domains' fault tables (marine, automotive,
industrial arms) and the "field required" rules that worked for you.

## License

Apache-2.0. See [LICENSE](LICENSE). © 2026 Cyberbee Ltd.
