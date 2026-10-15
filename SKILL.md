---
name: before-after-proof
description: Capture before-and-after evidence for every code change, including pure backend changes. A UI change gets screenshots. A backend change gets captured terminal output, a metric, or a log diff. The capture must happen BEFORE the edit, because a "before" cannot be reconstructed afterwards. Use at the start of any change to code, config, schema, prompt, or infrastructure.
trigger: before you edit any file; when the user says "fix", "change", "refactor", "optimise", "improve", "make it faster", "add", "remove", "update", "migrate", "bump", "tune"; before any commit; when someone claims a change worked; when a result is reported without evidence
version: 1.0.0
---

# Before-and-after proof

Every change gets two captures: the state before, and the state after. This applies to a screen, a terminal, a number, a log, or a schema. Nothing is exempt because it "has no UI".

## The rule that makes this work

**Capture the before FIRST. You cannot make a before after you have edited the file.**

This is the whole skill. Everything else is detail.

An agent that edits first and then tries to describe the previous behaviour is guessing. The guess is usually right, which is worse, because the one time it is wrong nobody can tell.

## Why this exists

A change without a before-and-after has three failure modes:

1. **The change did nothing.** The code looks different. The behaviour is identical.
2. **The change did something else.** The stated fix and the real effect do not match.
3. **The change made it worse.** Nobody measured, so nobody knows.

All three are invisible in a diff. All three are obvious in a before-and-after.

## The protocol

### Step 1. Name the observable

Before you touch anything, answer one question:

> **What will look different, and where will I see it?**

If you cannot answer, stop. A change with no observable difference is either dead code or an unmeasured claim. Say which.

Write the answer down. It becomes the caption on the evidence.

### Step 2. Capture the before

Capture the observable now, while the old behaviour still exists. See `references/CAPTURE.md` for the method for each change type.

Save it to `evidence/<slug>/before.<ext>`.

### Step 3. Make the change

Normal work. No special rules.

### Step 4. Capture the after

**Use the identical command, the identical inputs, and the identical seed.** A different command produces a different-looking result for a reason that has nothing to do with the change.

Save it to `evidence/<slug>/after.<ext>`.

### Step 5. Put them side by side

Show both. Do not describe them. A reader must be able to see the difference without trusting the description.

```
BEFORE                          AFTER
106 of 214 steps raised         0 steps raised
mean reward 4.1125              mean reward 7.6767
```

### Step 6. State what did NOT change

This is the step everyone skips, and it is where regressions hide.

> Reversed lap fell from 208 percent to 0 percent. Every other trajectory scored identically.

An after-capture that only shows the improvement is an advertisement.

## What counts as evidence

Ranked. Use the highest one the change permits.

| Rank | Evidence | Use when |
| --- | --- | --- |
| 1 | A screenshot or recording of the real screen | the change touches a UI |
| 2 | Captured command output, saved to a file | the change touches a CLI, a build, a test, a script |
| 3 | A number produced by the same command before and after | the change touches performance, accuracy, cost, or size |
| 4 | A diff of two saved logs | the change touches a service or a background job |
| 5 | A schema dump or a row count, before and after | the change touches a database |
| 6 | A statement that no observable exists | only when ranks 1 to 5 are truly impossible |

**Rank 6 requires a reason.** Write it. "This is a comment-only change" is a reason. "It is just a refactor" is not, because a refactor that changes behaviour is exactly the case this catches.

## For a pure backend change

This is the case people skip. Do not skip it.

A backend change still has an observable. Find it:

- **An API change**: the response body before, and after. Same request.
- **A performance change**: the timing before, and after. Same input, three runs each.
- **A bug fix**: the failing output before, and the passing output after. The *before capture must show the failure*. A fix with no captured failure is a fix for a bug nobody proved existed.
- **A refactor**: the test output and the timing before, and after. Both must be unchanged. That is the evidence.
- **A schema change**: the schema before, and after, plus a row count.
- **A dependency bump**: the version and the test output before, and after.

**A terminal screenshot counts.** Save the text, not a picture of the text, unless colour or layout carries the meaning.

## What breaks the proof

- **Capturing the before after editing.** Then it is not a before.
- **Different commands for the two captures.** Then the difference is the command.
- **A stale after.** Re-run it after the final edit, not after the first one.
- **Showing only the good number.** Show the ones that stayed the same too.
- **Describing instead of capturing.** "Errors went away" is a claim. The captured output is evidence.
- **A before that already passes.** If the before-capture shows success, you have not reproduced the problem yet. Stop and reproduce it first.

## Working with the other skills

- `easier-to-read-submissions` runs at commit, push, and hand-off. **This skill runs before the edit.** They compose: capture here, and the changelog entry there cites the evidence path.
- `agentic-ui-qa` covers persona journeys and artifact-only completion claims for UI work. Use it when the change is a UI feature. Use this skill for every change, including the ones agentic-ui-qa does not cover.

## The one-line test

Before you say a change worked, ask:

> **Could someone else look at what I captured and disagree with me?**

If no, you have not captured evidence. You have written a summary.

## Reference

- `references/CAPTURE.md` — the capture command for each change type, and the traps
