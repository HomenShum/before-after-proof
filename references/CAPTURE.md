# Capture methods

The command for each change type, and the traps. Run the before-capture and the after-capture with the identical command.

---

## Set up once per change

```bash
SLUG=fix-reward-crash          # short, kebab case
mkdir -p "evidence/$SLUG"
```

Everything below writes into that folder. Keep the folder in the repo, or attach it to the pull request. Evidence that lives only in a chat window is not evidence.

---

## Command output, the most common case

```bash
# BEFORE, run this while the old behaviour still exists
<your command> 2>&1 | tee "evidence/$SLUG/before.txt"

# ... make the change ...

# AFTER, the identical command
<your command> 2>&1 | tee "evidence/$SLUG/after.txt"

# The difference, which is the artifact you show
diff -u "evidence/$SLUG/before.txt" "evidence/$SLUG/after.txt" \
  > "evidence/$SLUG/diff.txt"
```

**Trap.** Output that contains a timestamp, a path, a process id, or a random seed produces a noisy diff. Strip those first:

```bash
<your command> 2>&1 | sed -E 's/[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9:.]+//g' | tee ...
```

---

## A number: timing, accuracy, size, or cost

One run is not a measurement. Run it at least three times and record all of them.

```bash
for i in 1 2 3; do
  /usr/bin/time -f "%e" <your command> 2>> "evidence/$SLUG/before_timing.txt"
done
```

Report the mean and the spread. **A change inside the spread is not a change.** This repository has already published a result that was one lucky seed, and had to withdraw it.

For accuracy or a score, use the same seed on both sides:

```bash
python bench.py --seed 0 > "evidence/$SLUG/before_seed0.txt"
```

---

## A user interface

```bash
# with a headless browser
npx playwright screenshot --viewport-size=1280,800 \
  http://localhost:3000/the-page "evidence/$SLUG/before.png"
```

Or use the browser tools if the agent has them, and save the image to the same folder.

**Traps.**
- Use the identical viewport for both captures. A different width is a different layout.
- Wait for the page to settle. A screenshot of a loading spinner proves nothing.
- Capture the specific element when the page is long. A full-page image hides a small change.
- Log in first if the screen needs it. A login wall is not a before.

---

## A bug fix

The before-capture **must show the failure**. This is not optional.

```bash
# BEFORE: reproduce the bug and capture it failing
pytest tests/test_thing.py::test_the_bug -x 2>&1 | tee "evidence/$SLUG/before.txt"
# the file must contain the failure
```

If the before-capture passes, you have not reproduced the bug. Stop. A fix for an unreproduced bug is a guess, and the test you add will pass whether or not the fix works.

Then fix, then run the identical command. The after-capture shows the pass.

---

## A refactor

The evidence for a refactor is that **nothing changed**.

```bash
# BEFORE
pytest -q 2>&1 | tee "evidence/$SLUG/before_tests.txt"
/usr/bin/time -f "%e" python bench.py 2>> "evidence/$SLUG/before_timing.txt"

# AFTER, identical commands
```

Both captures should match. A refactor that changes the test output is not a refactor. Say so and treat it as a behaviour change.

---

## An API or a service

```bash
# BEFORE
curl -s -X POST localhost:8000/endpoint -d @fixture.json \
  | python -m json.tool > "evidence/$SLUG/before.json"
```

Use a fixed fixture file, not a hand-typed body. A different request body makes a different response for a reason unrelated to the change.

For a log diff, capture a fixed window:

```bash
docker logs --since 60s the-service > "evidence/$SLUG/before.log" 2>&1
```

---

## A database or schema change

```bash
# BEFORE
pg_dump --schema-only mydb > "evidence/$SLUG/before_schema.sql"
psql -c "select count(*) from the_table" > "evidence/$SLUG/before_count.txt"
```

Record the row count with the schema. A migration that silently drops rows passes a schema check.

---

## A prompt or a model change

Treat it like accuracy. Fix the seed and the input set.

```bash
python eval.py --seed 0 --n 50 > "evidence/$SLUG/before.txt"
```

Report the mean over the set. **Never report the best single output.** A cherry-picked generation is the same error as a lucky seed.

---

## A dependency bump

```bash
pip freeze | grep -i the-package > "evidence/$SLUG/before_version.txt"
pytest -q 2>&1 | tee "evidence/$SLUG/before_tests.txt"
```

---

## When there is genuinely nothing to observe

Write the reason in `evidence/$SLUG/no-observable.md`. It must name the change class:

- a comment or a docstring only
- a rename with no call-site behaviour change, and the test output proves it
- dead code removal, and the evidence is that the code was unreachable

**"It is only a refactor" is not on the list.** A refactor has observable evidence: the tests and the timing, both unchanged.

---

## The side-by-side block

This is what goes in the pull request, the changelog, or the message to the user.

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

The `UNCHANGED` block is the part that catches regressions. Do not drop it.

---

## Recovery: you already edited, and there is no before

The rule is capture first. Sometimes you did not. The before is not gone — it is in git — but
reconstructing it is strictly worse than capturing it, so treat this as repair, not as a workflow.

**Never write the before from memory.** The guess is usually right, which is the problem: the one
time it is wrong, nothing in the artifact reveals it. Run the old code.

Three mechanisms, in order of preference.

### 1. Run the old file in place (single-file behaviour change)

```bash
git show <sha>:src/cli.mjs > src/cli.before.tmp.mjs
node src/cli.before.tmp.mjs <same args>   # capture this
rm src/cli.before.tmp.mjs
```

Relative imports resolve against the *current* tree, so this is only valid when the behaviour you
are demonstrating lives in that one file. If the change spans modules, the "before" is
contaminated by new code and the comparison is a lie. Use a worktree instead.

Two traps, both of which have produced false artifacts:

- The temp file appears in tooling that scans the tree — generated maps, indexes, coverage. Delete
  it before regenerating anything, or it lands in a committed artifact.
- If the module re-exports from a sibling you also changed, you are running a hybrid. Check the
  import graph before trusting the output.

### 2. Detached worktree (multi-file change)

```bash
git worktree add --detach ../before-capture <sha>
```

Correct for anything spanning more than one file. The catch is dependencies: do **not** reinstall,
and do not copy `node_modules` — a large copy will blow a command timeout partway and leave a
half-populated directory. Symlink it, or run the capture with the parent's resolution.

Remove it when done: `git worktree remove --force ../before-capture`.

### 3. Stash, capture, restore (last resort)

Only when the change is uncommitted and the other two do not apply.

**Do not chain stash + checkout + a long command in one invocation.** If the timeout fires between
the checkout and the restore, your work is sitting in a stash while the branch has moved, and the
recovery is manual. Run each step separately and verify between them.

### When the before genuinely cannot be reproduced

Production data is gone, the bug needed a state you no longer have, the environment is destroyed.
Then say exactly that and publish the after alone, labelled one-sided.

> Before could not be reproduced: the failure required the pre-migration table, which no longer
> exists. After-capture only.

An honest one-sided artifact is worth more than a fabricated pair. What you may not do is present a
reconstructed before as if it were captured.

---

## Deployment is not a capture

A build log, a green pipeline, a `git push` that succeeded, and a CLI exit code are all claims that
a deploy was *attempted*. None of them observes the running system.

Before writing "deployed", "live", or "the site now shows X":

1. Confirm a new deployment exists and is newer than the commit.
2. Fetch the live URL.
3. Grep the response for a **concrete signal from this change** — a testid, an exact string, a
   count, a version.

If the signal is not in the response, it is not shipped, whatever the build log said.

Three failure modes this catches, all of which produce a green build and a stale site:

- The repository-to-host webhook is silently disconnected. Pushes land; nothing rebuilds.
- Server-rendered output differs from what a browser shows. A client-only component can render a
  fallback in the fetched HTML while looking correct in your browser.
- A CDN serves cached HTML for the old build.

The capture is the fetched response containing the signal — not the deploy log.
