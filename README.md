# before-after-proof

An agent skill that forces before-and-after evidence for **every** code change, including pure backend changes.

Works with Claude Code, Codex, Cursor, and any runtime that reads the Agent Skills format.

---

## The rule

> **Capture the before FIRST. You cannot make a before after you have edited the file.**

That is the whole skill. Everything else is detail.

An agent that edits first and then describes the previous behaviour is guessing. The guess is usually right, which is worse, because the one time it is wrong nothing in the artifact reveals it.

## Why this is not visual regression testing

Visual regression testing already exists and it is good. This is a different thing.

| Visual regression testing | This skill |
| --- | --- |
| Runs in CI, after the change | Runs before the edit |
| Compares against a stored baseline | Captures the baseline now, for this change |
| Covers screens | Covers screens, terminals, numbers, logs, and schemas |
| Answers "did the UI drift" | Answers "did my change do what I said" |

## The gap it fills

A change without a before-and-after has three failure modes, and all three are invisible in a diff:

1. **The change did nothing.** The code looks different. The behaviour is identical.
2. **The change did something else.** The stated fix and the real effect do not match.
3. **The change made it worse.** Nobody measured, so nobody knows.

## Pure backend is not exempt

This is the case people skip. Evidence is ranked; use the highest one the change permits.

| Rank | Evidence | Use when |
| --- | --- | --- |
| 1 | A screenshot or recording of the real screen | the change touches a UI |
| 2 | Captured command output, saved to a file | a CLI, a build, a test, a script |
| 3 | A number produced by the same command three times | performance, accuracy, cost, size |
| 4 | A diff of two saved logs | a service or a background job |
| 5 | A schema dump and a row count | a database |
| 6 | A written statement that no observable exists | only when 1 to 5 are truly impossible |

**Rank 6 requires a reason, and "it is just a refactor" is not one.** A refactor's evidence is that the tests and the timing are unchanged. That is an observable.

## Two rules that came from real failures

**A bug fix's before-capture must show the failure.** If the before passes, you have not reproduced the bug. The test you add will pass whether or not your fix works.

**Every report needs an UNCHANGED block.** An after-capture that shows only the improvement is an advertisement. The things that stayed the same are where regressions hide.

## What a report looks like

```
CHANGE   fix the crash in the second lookahead loop
FILE     data/reward_functions/v4d.py
COMMAND  python scorer/run.py --all data/reward_functions

BEFORE                              AFTER
106 of 214 steps raised             0 steps raised
mean reward on optimal 4.1125       mean reward on optimal 7.6767
reversed lap 208% of optimal        reversed lap 0% of optimal

UNCHANGED
every other trajectory scored identically
offtrack still 0% of optimal

EVIDENCE  evidence/fix-reward-crash/{before,after,diff}.txt
```

## Install

Clone into your skills directory:

```bash
git clone https://github.com/HomenShum/before-after-proof \
  ~/.claude/skills/before-after-proof
```

The skill triggers before any edit, and on the words fix, change, refactor, optimise, improve, add, remove, update, migrate, bump, and tune.

## Files

| File | Contents |
| --- | --- |
| `SKILL.md` | the protocol, the six steps, what breaks the proof |
| `references/CAPTURE.md` | the capture command for each change type, the traps, and how to recover a before from git when you already edited |

## Origin

This came out of a session that produced three false positives in one day: a result measured with the wrong statistic, a simulator with no physical limit that made impossible results look good, and a single lucky seed reported as a win. All three were caught, and all three would have been caught sooner by a before-and-after.

The companion skill [easier-to-read-submissions](https://github.com/HomenShum/easier-to-read-submissions) runs at commit and hand-off time. This one runs before the edit. They compose: capture here, cite the evidence path there.

## Licence

MIT.
