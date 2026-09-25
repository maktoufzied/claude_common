---
name: fast-review
description: Code review of the current diff (or a PR/branch/path target) at max-effort recall for a fraction of the tokens — one shared diff bundle, 3 parallel finders, inline verify, no verifier fleet. Covers correctness bugs, /simplify-style cleanup (whether the change needs to exist at all, reuse, simplification, efficiency, altitude), and — when the PR closes a Linear ticket — whether the diff actually delivers what the ticket asked for. Use instead of /code-review when the review should be thorough but cheap. Pass --comment to post findings as one PR review — short plain-English inline comments prefixed blocker:/nit:, with suggested changes where the fix is known, no summary, Request Changes only when there is a blocker. Pass --fix to apply the cleanups.
argument-hint: "[--comment] [--fix] [<pr#>|<branch>|<path>]"
---

# fast-review

Same angles and verdict ladder as `/code-review max`, ~3 subagents instead of
~90. The saving is not fewer checks — it is that the diff is computed **once**
and read from a file, instead of ~90 agents each running `git diff` and
re-reading the same enclosing functions.

## Phase 0 — Build the shared bundle (one Bash call)

```bash
R=$(git rev-parse --show-toplevel) && D="${TMPDIR:-/tmp}/review-bundle-$$.md" && {
  echo "# Review standards"; cat "$R/REVIEW.md" 2>/dev/null
  echo; echo "# Conventions"; cat "$R/CLAUDE.md" "$R/AGENTS.md" 2>/dev/null
  echo; echo "# Diff (whole enclosing functions)"
  git diff -W @{upstream}...HEAD 2>/dev/null || git diff -W main...HEAD 2>/dev/null || git diff -W HEAD~1
  git diff -W HEAD
} > "$D"; wc -l "$D"; echo "BUNDLE=$D"
```

`-W` (`--function-context`) emits the **whole enclosing function** for every
hunk. That is the trick: it removes the "now Read the enclosing function" step
from every finder, which is where most of the per-agent tool cost went.

If a PR number, branch, or path was passed as an argument, swap the diff
command for that target. If the bundle is empty (0 lines of diff), stop and
say there is nothing to review — do not spawn finders.

Do **not** print the bundle. Note its path; it is the shared memory.

### Add the Linear ticket, when there is one

```bash
gh pr view "$PR" --json title,headRefName -q '.title + " " + .headRefName' \
  | grep -oE '[A-Z][A-Z0-9]+-[0-9]+' | sort -u
```

Title and branch only — the PR body cites other tickets and would produce false
matches. If that yields exactly one ID, read it with
`mcp__claude_ai_Linear__get_issue` and append the **title and full description**
to the bundle:

```bash
cat >> "$D" <<'TICKET'

# Ticket AGE-548 — <title>
<description>
TICKET
```

Skip silently when there is no PR, no ID, more than one ID, or Linear is not
connected. Never infer the intent from the PR title instead — a PR title
describes the diff, so checking the diff against it proves nothing.

## Phase 1 — Three parallel finders

Two hunt correctness (cap **8** each), one hunts cleanup (cap **20** — a merged
finder needs the summed budget of the lenses it absorbed, or cleanup silently
loses to correctness before the report even starts).

Launch all three in a **single message**. Each prompt starts with:

> Read `$BUNDLE` — it contains the review standards, the conventions, and the
> full diff with whole enclosing functions. Do NOT run git; the diff is already
> there. Read or Grep other files only when your angle genuinely needs code the
> bundle does not contain.
>
> Return **at most `<cap>`** candidates, one per line, exactly:
> `path/to/file.ext:123 | one-line summary | concrete cost or failure scenario`
> For correctness, the failure scenario is the user-visible consequence (error,
> wrong output, data loss), not an intermediate state. For cleanup, it is the
> concrete cost — what is duplicated, wasted, harder to maintain, or which rule
> is broken. Pass through every candidate you can name a consequence for — do
> not silently drop half-believed ones. No prose, no preamble. `(none)` if
> nothing qualifies.

Then one angle cluster each:

**Finder 1 — `line`** (correctness, cap 8)
> Read every hunk line by line. For each line ask: what input, state, timing, or
> platform makes this wrong? Inverted/wrong conditions, off-by-one,
> null/undefined deref, missing `await`, falsy-zero checks, wrong-variable
> copy-paste, error swallowed in catch, unescaped regex metachars. Plus the
> classic pitfalls of this language/framework: JS `==` coercion and
> closure-captured loop vars; Python mutable default args and late-binding
> closures; Go nil-map writes and range-var capture; SQL injection; timezone/DST
> drift; float equality. Bugs in unchanged lines of a touched function are in
> scope — the diff re-exposes or fails to fix them.

**Finder 2 — `delta`** (correctness, cap 8)
> For every line the diff DELETES or replaces, name the invariant it enforced,
> then find where the new code re-establishes it. If you cannot, that is a
> candidate: a removed guard, a dropped error path, a narrowed validation, a
> deleted test that covered a real case. Also: when the diff adds or modifies a
> type that wraps another (cache, proxy, decorator, adapter), check every method
> routes to the wrapped instance and not back through a registry/session/global,
> and that it forwards the methods callers actually use. Finally, for each
> function the diff changes, Grep for its callers and check whether the change
> breaks any call site: a new precondition, a changed return shape, a new
> exception, a timing/ordering dependency. Check callees too — does a parallel
> change in the same diff make a call unsafe?

**Finder 3 — `clean`** (cleanup, cap 20 — the `/simplify` pass)
> You are improving the quality of the changed code, not hunting for bugs. Do
> not report correctness defects; other finders cover those. Review the changed
> code through **each** of the six lenses below. Cover whichever apply — you do
> not need findings from every lens; prioritize the highest-cost issues across
> all of them.
>
> **Necessity** — the change, or a piece of it, should not exist at all. Ask
> this lens first, because the cheapest fix for the other five is deleting the
> code they would improve. Is there an existing solution that makes the new code
> redundant — a dependency already in the manifest, a stdlib or platform
> feature, a config option, a code path that already does this job? Does the
> diff add a seam for a case that does not exist yet: an abstraction with one
> implementation, a factory for one product, a parameter every caller passes the
> same value for, a hook or plugin layer with one user? Name what to delete and
> what covers it instead. When the whole change is unnecessary, file it once on
> the entry point, not on ten lines.
>
> Do not flag: anything the ticket or PR description asks for; validation at a
> trust boundary, error handling, accessibility, or a test; scaffolding this diff
> already consumes. The bar is a named replacement or an unused seam — "I would
> not have built this" is not a finding.
>
> **Reuse** — new code that re-implements something the codebase already has.
> Grep shared/utility modules and files adjacent to the change, and name the
> existing helper to call instead.
>
> **Simplification** — unnecessary complexity the diff adds: redundant or
> derivable state, copy-paste with slight variation, deep nesting, dead code
> left behind. Name the simpler form that does the same job.
>
> **Efficiency** — wasted work the diff introduces: redundant computation or
> repeated I/O, independent operations run sequentially, blocking work added to
> startup or hot paths. Also long-lived objects built from closures or captured
> environments — they keep the entire enclosing scope alive for the object's
> lifetime, a leak when that scope holds large values; prefer a struct that
> copies only the fields it needs. Name the cheaper alternative.
>
> **Altitude** — changes implemented at the wrong depth, as a fragile bandaid.
> Special cases layered on shared infrastructure are a sign the fix isn't deep
> enough; prefer generalizing the underlying mechanism.
>
> **Conventions** — check the diff against the review standards and conventions
> at the top of the bundle, plus any CLAUDE.md in a directory above a changed
> file. Flag a violation only when you can quote the exact rule and the exact
> line that breaks it — no style preferences, no "spirit of the doc" inferences.
> Name the file and quote the rule.

## Phase 2 — Verify inline (no subagents)

Read the bundle yourself. Dedup candidates pointing at the same line and
mechanism, keeping the one with the most concrete consequence.

**Fact-check every candidate — no exceptions, cleanup included.** A finder
reports what it believes; nothing reaches the report until you have read the
line it names. For each candidate, quote that line out of the bundle and check
three things: the `file:line` exists and says what the candidate claims, the
surrounding code does not already handle it, and the named fix actually applies
there. A candidate the quoted line contradicts is dropped — that is the whole
check. When the bundle cannot settle it, Read or Grep the file; never pass one
through unread, and never fact-check by re-reading the candidate's own summary.

**Cleanup candidates stop after the fact-check.** The verdict ladder below asks
"does this bug actually trigger", which is meaningless for a duplicated helper
or a wasted round trip; running cleanup through it only invents doubt. They
still have to clear the value bar in Phase 3.

Then judge each remaining **correctness** candidate against the bundle, quoting
the line:

- **CONFIRMED** — you can name the inputs/state that trigger it and the wrong
  output or crash.
- **PLAUSIBLE** — mechanism is real, trigger is uncertain (timing, env, config).
- **REFUTED** — factually wrong (quote the actual line), provably impossible
  (show the type/constant/invariant), already handled in this diff (cite the
  guard), or pure style with no observable effect.

PLAUSIBLE by default. Do not refute for being "speculative" when the state is
realistic: concurrency races, nil on a rare-but-reachable path (error handler,
cold cache, missing optional field), falsy-zero treated as missing, off-by-one
on a boundary the code does not exclude, retry storms, a regex that lost an
anchor. Keep CONFIRMED and PLAUSIBLE, drop REFUTED.

Read or Grep a file only for candidates the bundle cannot settle. That is the
second big saving — the old design spent one subagent per candidate to do this.

### Does the diff do what the ticket asked?

Only when the bundle has a `# Ticket` section. Take each requirement the ticket
states and find where the diff delivers it. This is the one check no line-level
angle can make: a diff can be flawless and still not fix the thing.

- A requirement the ticket states plainly, not implemented → **`blocker:`**.
- Changes well outside what the ticket asked for → **`nit:`**.

Quote the ticket sentence you are judging against, so the author can disagree
with the reading rather than with you.

Do not flag: something the ticket lists as out of scope or follow-up; partial
delivery when the PR says it is one of several; or your own view of what the
ticket should have asked for. If the diff delivers the ticket, say nothing —
this check exists to catch a miss, not to confirm a hit.

The finding has no natural line. Put it on the changed line closest to the gap
— the function that should have handled the missing requirement — never as a
review body.

## Phase 3 — Report

Two **separate** caps: at most 10 correctness findings and at most 5 cleanup
findings. They do not compete — a bug-heavy diff does not swallow the cleanup
budget, and a clean diff does not get its cleanup list padded to the cap.
Correctness first, CONFIRMED before PLAUSIBLE; cleanup after, highest-value
first — a necessity finding leads the cleanup list, since deleting code beats
improving it, and it must not be crowded out of the cap by five smaller nits.

### The nit value bar

A nit is a request for someone's afternoon. Keep a cleanup finding only when
one of these is true, and let that be the consequence clause in the comment:

- **Performance** — a measurable win on a real path: an N+1 removed, an
  allocation out of a hot loop, blocking work off startup. Not "could be faster".
- **Maintainability** — the next change here goes wrong without it: duplicated
  logic that will drift, state that can contradict itself, a special case bolted
  onto shared infrastructure.
- **Simplification** — the same behavior with materially less code: a helper
  that already exists, a dead branch, twenty lines that are five.
- **Necessity** — the code does not need to exist: something already shipped
  does the job, or the seam it adds has one caller and no second case in sight.
  The fix is deleting it, and you can name what covers it instead.

Drop everything else — naming, ordering, formatting, equivalent idioms,
one-line restructures, "consider extracting", anything whose only argument is
taste. If the win needs a hedge to state, there is no win.

Ten thin nits are worse than two real ones: the author stops reading at the
third. Under-filling the cap is the correct outcome, not a gap to fill.

### Label each finding

- **`blocker:`** — must be fixed before merge. Every CONFIRMED correctness
  finding; a PLAUSIBLE one only when the consequence is a crash, data loss, or
  wrong output reaching a user; a conventions finding that quotes an explicit
  must/never rule.
- **`nit:`** — everything else. All cleanup findings, and PLAUSIBLE bugs with a
  mild consequence.

Drop anything that is neither. Code that is already acceptable gets no comment —
no praise, no "consider whether…", no observations. A comment means someone has
to do something.

### Last gate — the labeled list

Now walk the final list, every `blocker:` and every `nit:`, and put each one
back against the bundle. Say out loud, per finding: the quoted line, and the
one thing that makes it true — the input that breaks it, the helper it
duplicates, the work it wastes. No quote, or a quote that does not carry the
claim, means it does not go out. This is the pass that catches the finding the
finders were confident about and the code does not support; skipping it because
Phase 2 already looked is how a wrong comment reaches the author.

### Write each comment

**Two sentences, 40 words, hard cap** — prose only, the suggestion block does
not count. One sentence for what is wrong, one for what happens. If it will not
fit, the comment is carrying reasoning that belongs in your head, not on the PR.

**Open on the defect.** The first clause names the thing that is on the line —
the word, the value, the missing call. A rule citation (`REVIEW.md: "…"`) is
evidence for the finding, not the finding: it comes second, never first.

**When the finding is a behavior difference, show one instance.** An input and
what happens to it — `"Starts at thirty-eight thousand" on a $47,020 line should
be WRONG_FIGURE` — beats any amount of describing it. Writing the instance is
also how you find out the finding is real; one you cannot instantiate usually
is not.

Cut all of this, every time:

- **Pointers of any kind.** Not only "at line 2508" — "gate rule 1 below", "the
  paragraph above", "protocol step 3", "Mode 2's body" all send the reader
  hunting through the file to work out what you meant. Quote the four words you
  mean instead.
- **Test, eval, or symbol names as evidence.** Naming the eval that fails is
  showing your work, not telling them what to change.
- **Why you rejected the other fix.** You picked one. Give it.
- **Restating the code back at them**, and hedging stacks ("it may potentially
  be possible that").
- **Coined terms and nominalizations.** "the absence register", "both
  fabrication patterns", "reads as the regating having landed". If an ordinary
  word works, it wins.
- **A description of the bug where the bug itself would fit.** "states the test
  one-sidedly" describes it from a distance; "`above` fails only a figure that
  is too high" points straight at it.

Be direct and kind — the finding is about the code, never the author.

Do not write (real comment, 1061 characters):

> blocker: `carfax_1_owner="no"` means the record affirmatively reports more
> than one owner, but "the record does NOT show a single owner" is the absence
> register — the same phrasing `_carfax_echo` uses at line 2508 for the unknown
> state … `eval_sales_answers_one_owner_and_clean_title` requires the answer
> "stated as a fact about that vehicle", which a hedge isn't, and it defeats
> what this line is for. State what the record holds instead of what it lacks.
> This wording is clear of both fabrication patterns …

Write (23 words):

> blocker: `"no"` means the car had several owners, but this says we don't know.
> The caller hears "I don't have that one" instead.
>
> ```suggestion
> _ONE_OWNER_NEGATIVE_GLOSS: Final[str] = " - the record lists multiple owners"
> ```

The suggestion block carries the fix, so the prose never has to describe it.
When you have a suggestion, the sentence about what to do is redundant — delete
it.

If you know the fix **and** can write the exact replacement lines, add a GitHub
suggestion block:

````
```suggestion
  const value = await ensureInit().then(get)
```
````

Only when it fully fixes the issue on its own and the replacement lines line up
exactly with the lines you are commenting on — a suggestion that does not apply
cleanly is worse than none. If you know the fix but cannot express it as exact
lines, say it in one sentence instead ("move this below the `await` on line 40").

### Output

Without `--comment`: print one line per finding, `file:line — blocker/nit: text`.
If the `ReportFindings` tool is available, call it once with
`{level: "high", findings}` instead — each entry with `file`, `line`, `summary`,
`short_summary` (≤60 chars, the claim alone), `failure_scenario`, `category`
(`correctness`, `necessity`, `reuse`, `simplification`, `efficiency`,
`altitude`, `conventions`), and `verdict` for correctness entries — and do not also print
them as text.

### Posting to a PR (`--comment`)

Post **one review** carrying every comment, not N loose comments. Write the
payload to a file (comment bodies contain backticks and newlines that do not
survive shell quoting) and send it in a single call:

```bash
gh api repos/{owner}/{repo}/pulls/{pr}/reviews --input "$PAYLOAD"
```

```json
{
  "event": "REQUEST_CHANGES",
  "body": "",
  "comments": [
    { "path": "src/thing.ts", "line": 42, "side": "RIGHT", "body": "blocker: …" }
  ]
}
```

Before sending, count the words in every comment body. Anything over 40 words of
prose goes back and gets cut — do not post it and note that it is long.

Then read each comment as the author will: with only the line it is attached to,
and none of your context. Rewrite it if a pointer sends them elsewhere in the
file, if a noun in it is one you coined this session, if it opens on a rule
citation instead of the defect, or if the subject of the example sentence is not
the thing that acts. Fixing this after posting costs an edit per comment.

The verdict follows the blockers, nothing else:

- One or more `blocker:` → `REQUEST_CHANGES`.
- No blockers → `APPROVE`, however many nits there are. **Never request changes
  for nits alone.**

No review summary and no overall PR comment — the inline comments are the
review. `body` stays `""` on both events. (The API docs claim `body` is required
for `REQUEST_CHANGES`; that only applies to a review with no inline comments. A
review carrying a `comments` array posts fine with an empty body.)

Every `line` must fall inside the diff, or the whole call 422s. Context lines
shown by `-W` are fine; a finding anchored outside the diff is not — move it to
the nearest changed line in the same hunk, or leave it out of the payload and
print it locally instead.

Do not merge the PR, and do not push. Report the review URL when done.

## Phase 4 — Apply (`--fix` only)

Skip this phase unless `--fix` was passed.

Fix each **cleanup** finding directly, then each **CONFIRMED** correctness
finding. Leave PLAUSIBLE correctness findings alone — an unconfirmed bug fix is
a guess at the working tree, and the report already names them for the user.

Skip any finding whose fix would change intended behavior, require changes well
outside the reviewed diff, or that you judge to be a false positive — note the
skip rather than arguing with it.

If `ReportFindings` was called, call it again with the same findings, each
carrying an `outcome`: `fixed`, `no_change_needed`, or `skipped`, and do not
repeat the findings as text. Otherwise give one line per skipped finding saying
why. Do not commit.

Finally: `rm -f "$BUNDLE"`.

## Deliberate omissions

- **No sweep round.** An extra find+verify cycle for the weakest 8 candidates.
  Add a 4th finder ("find only what is not on this list") for a release blocker.
- **No verifier subagents.** Inline verify means the context that found a
  candidate also judges it. Spawn one judge for a single load-bearing finding.
- **Root-level CLAUDE.md/AGENTS.md only**; per-directory ones are left to
  Finder 3's Grep.
- **Verdicts are not severities.** CONFIRMED/PLAUSIBLE is confidence;
  blocker/nit is "must act before merge". Consequence decides, not verdict.
- **No separate necessity finder.** It is a lens on Finder 3, sharing the cap
  20 — the angle needs the same diff and the same "what already exists" Grep as
  Reuse, so a fourth agent would buy nothing.
- **One cleanup finder, not four.** Same diff, same finding shape — merging
  saves 3 agents. Only safe because the cap is summed (20, not 8).
