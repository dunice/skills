---
name: writing-pull-request-descriptions
description: Use when opening a pull request, running `gh pr create`, filling in a PR template, or rewriting the description on an existing PR — including when the branch has picked up unrelated changes, the diff has grown past a few hundred lines, or the work changes something a user can see.
---

# Writing Pull Request Descriptions

## Overview

The description has one job: **the reviewer can predict what the diff looks like before they open it.** Open the Files-changed tab and be surprised, and the description failed.

Two gates come before you write a word. A description cannot rescue a branch that fails either one.

## Gate 0 — Pick the Base Branch

Every command below diffs against a base. Resolve it once, first, and pin it.

**A branch the user named wins.** "against `develop`", "base it on `release/24.4`" — use that, skip the rest.

Otherwise, in this order:

1. `main` exists → base is `main`
2. else `master` exists → base is `master`
3. neither exists → **ask the user which branch to review against.** Do not pick one.

```bash
git fetch origin
if   git show-ref -q refs/remotes/origin/main   || git show-ref -q refs/heads/main;   then BASE=main
elif git show-ref -q refs/remotes/origin/master || git show-ref -q refs/heads/master; then BASE=master
else BASE=; fi
echo "base=${BASE:-UNRESOLVED - ask the user}"
```

Every diff in this skill is then `git diff origin/$BASE...HEAD`.

**Step 3 is a question, not a puzzle to solve.** `origin/HEAD`, the protected branch, the longest-lived branch, the one most commits land on, `develop` because it is conventional — none of these are the rule. A repo with `develop` and `trunk` and no `main`/`master` is exactly where guessing means the diff you reviewed is not the diff the reviewer sees.

Three-dot `...` is the merge base, which is what the PR will show. Two-dot pulls in everything that landed on the base since you branched.

## Gate 1 — Review Your Own Diff First

Read the whole diff the way the reviewer will: `git diff origin/$BASE...HEAD`, or the Files-changed tab in GitHub/GitLab. Not the commits — the diff.

**Everything you find, you fix on the branch. You do not describe it in the PR body.**

| Look for | Do |
|---|---|
| `console.log`, `printf`, debug prints on a hot path | Delete |
| Commented-out code "kept for reference" | Delete — git remembers it |
| Empty or placeholder test bodies | Write them, or drop the file |
| Whitespace/format-only hunks in files you didn't otherwise touch | Revert |
| Renamed variable or import that drifted in from another task | Revert |
| A file you can't explain in one sentence | Understand it or remove it |

A PR body containing a section titled "not ready to merge as-is" is a branch that was not self-reviewed. Fix, then describe.

## Gate 2 — Check the Scope

| Property | Threshold |
|---|---|
| Diff size | Aim under 200–300 changed lines. Review quality collapses past 400 — bugs stop being found. |
| Logical changes | Exactly one. One bug, or one feature. |

Generated files, lockfiles and pure moves don't count against the line budget — say so in the description when they inflate the number.

**Unrelated changes leave the branch before the PR opens.** A typo fix, a drive-by refactor, a style tweak, an extra helper nobody calls — each is its own PR, or is reverted.

```bash
git restore --source=origin/$BASE -- README.md src/format.js   # drop it
git switch -c chore/typo origin/$BASE && git cherry-pick -n <sha>  # or rehome it
```

Too big and genuinely one thing? Split it into a stack of PRs, each reviewable on its own.

## The Description

Exactly this shape, in this order. Repo template wins on section names; the four jobs stay the same.

**Title:** `TICKET-123: imperative summary` when the branch carries a tracker key (`TICKET-412`). A GitHub issue number never goes in the title — it belongs in the body as `Closes #412` — so those, and branches with no ticket, get the imperative summary alone. What the change does, not what you did.

### What changed

Two or three sentences of behavior. The new rule, the new endpoint, the thing that used to happen and now doesn't. Numbers where they matter (limits, timeouts, sizes).

Bullets only for what the file list does **not** already tell the reviewer. Never walk the diff file by file — they have the diff.

### Why

The reason, in one or two sentences.

**The ticket comes from the branch name.** `feature/TICKET-412-rate-limit` → `TICKET-412`; `fix/1847-stale-cache` → `#1847`. Read it off the branch rather than asking for it. A ticket the user names outright wins over the branch.

```bash
B=$(git rev-parse --abbrev-ref HEAD)
echo "$B" | grep -oE '(^|/)[A-Z][A-Z0-9]+-[0-9]+' | tr -d '/' | head -1   # tracker key
echo "$B" | grep -oE '(^|/)[0-9]+-'              | tr -d '/-' | head -1   # issue number
```

Both are anchored to the start of a path segment, so `fix/ipv6-2001-db8` yields nothing. **A number still has to be confirmed before you use it** — `release/2024-11-cutover` matches, and `#2024` is not a ticket:

```bash
gh issue view 1847 --json number,title   # resolves → use it. doesn't → no ticket line.
```

| Branch | Put in the body |
|---|---|
| Key + you know the tracker URL | The linked key — `[TICKET-412](https://jira.acme.com/browse/TICKET-412)`. No `Closes` — the keyword only closes issues in the same GitHub repo. |
| Key, tracker URL unknown | The bare key — `TICKET-412`. Most trackers autolink it. |
| GitHub issue number | `Closes #412` — add the `#`, no host needed |
| No key in the branch | Nothing. The reason alone is the whole section. |

**Never invent the host.** `jira.example.com`, `your-org.atlassian.net`, a guessed subdomain — a link that 404s is worse than the bare key. Take the URL from a merged PR body, the repo docs, or the user; otherwise write the key alone.

**No key in the branch means no ticket line at all.** Not "No ticket.", not "N/A", not `Closes #TODO`, not a note about why you didn't file one. Give the reason the work happened and stop — an unreferenced PR is a normal PR.

### How to review

Where to start and what to focus on:

- the file to open first, and the order after it
- the decision you actually want an opinion on
- what can be skimmed (generated, mechanical, test fixtures)
- how you verified it — the command, the environment

### Screenshots

**Include one whenever the change is visible to a user** — screenshot for static UI, GIF or short recording for anything with states or motion. Before/after side by side when you changed something that already existed.

Nothing user-visible changed? Omit this section entirely. Do not write "N/A — no UI changes".

## Contract

| Property | How to check |
|---|---|
| Base branch resolved | User's branch, else `main`, else `master`, else you asked |
| Reviewer can predict the diff | Read your own description, then the file list. Any surprise? |
| One logical change | Can you name it without "and"? |
| Under ~300 lines | `git diff origin/$BASE...HEAD --stat \| tail -1` |
| Title prefix | Tracker key → `KEY-123: ` prefix. Issue number or no ticket → summary alone |
| Ticket handled | Key in the branch → it appears in the body. No key → no ticket line, no placeholder, no invented host |
| Starting point named | "How to review" names a specific file |
| Visuals present | Any user-visible change has an image, GIF or recording |
| No self-review debt | The body contains no known defect, TODO, or apology |

## Example

Branch `feature/TICKET-412-rate-limit` — the key comes off the branch, and the
tracker host is unknown here, so the body carries the bare key. After both gates: unrelated typo/helper/formatting changes moved off the branch, `console.log` and commented-out `naiveLimit` deleted, stub tests filled in. 48 lines across 3 files.

```markdown
TICKET-412: rate limit requests per IP

## What changed

Every request now counts against a per-IP budget of 100 requests per 60-second
fixed window. Over budget returns `429` with `Retry-After`; every response
carries `X-RateLimit-Limit` and `X-RateLimit-Remaining`. Counters are in-process
— per instance, not global.

## Why

TICKET-412. A single misbehaving webhook caller saturated the service twice
last week.

## How to review

Start at `src/rateLimit.js` — the whole policy is the two constants at the top
and the window reset. `src/server.js` is a one-line `app.use`. Tests cover under
limit, over limit, and reset after the window.

The decision worth your opinion: buckets live in a process-local `Map` and are
pruned on access only. Behind more than one instance the effective limit is
100 × instances. Ship it that way, or block on Redis?
```

## Rationalizations (verbatim from baseline testing)

| Excuse | Reality |
|---|---|
| "Happy to split them out if reviewers would rather" | Splitting is your job and it takes a minute. Offering it makes the reviewer decide, and they'll say yes. |
| "Flagging these rather than letting them slip through review" | Flagging a `console.log` is not removing a `console.log`. Delete it and delete the flag. |
| "Commented-out, kept for reference" | Git is the reference. |
| "They're tiny unrelated changes, splitting is more overhead than it's worth" | Size isn't the cost. Each unrelated change adds a reason the whole PR can be blocked. |
| "The typo fix is already in the diff, taking it out is churn" | `git restore` is not churn. Shipping two things as one is. |
| "The reviewer should know the limitations" | A limitation needing a decision goes in *How to review*. A known defect goes in a commit, before you open the PR. |
| "Listing every file makes it easier to follow" | It duplicates the diff and hides the two lines that matter. Give them a map, not a transcript. |
| "It's one feature, it's just a big one" | Then it's a stack of PRs. Past 400 lines the review stops finding bugs. |
| "`origin/HEAD` points at `trunk`, so that's the base" | The rule is the user's branch, then `main`, then `master`, then ask. `origin/HEAD` is not step 3. |
| "No `main`, but `develop` is obviously the integration branch" | Obvious to you. Ask — it costs one sentence and a wrong base invalidates the whole review. |
| "I'll just diff against `main` and see if it errors" | Resolve the base before the first diff, not by trial and error. |
| "`jira.example.com/browse/TICKET-412` — the user can swap in the real host" | You shipped a dead link and made it their job. Write the bare key. |
| "Every PR should reference something, so I'll note there is no ticket" | Announcing an absence is the placeholder. Delete the line. |
| "I'll ask which ticket this is" | The branch says. Read it before asking. (Asking for the tracker *host* is fine — that is not in the branch.) |
| "The branch key is in the title, so the body doesn't need it" | The title is not the body. Put the key in Why. |
| "Tests are stubs but they pass" | A green suite that asserts nothing is worse than no suite. Not ready to open. |

## Red Flags — STOP

- You are running `git diff main...HEAD` without having checked that `main` exists
- No `main` and no `master`, and you picked a base instead of asking
- Your body contains `example.com`, `TODO`, `XXX`, or a tracker host you did not verify
- You are writing "No ticket" or "N/A" in the Why section
- You are about to ask which ticket this is without having read the branch name
- You are writing a section called "Known limitations", "Not ready to merge", or "TODO before merge"
- You are typing "happy to split this out if you'd rather"
- Your body lists every changed file with a description of each
- The diff includes a file you'd struggle to justify if asked
- `--stat` says 400+ lines and you are writing the description anyway
- The only mention of the ticket is in the title
- You changed the UI and are about to open the PR without a screenshot

**All of these mean: go back to the gates.**

## Not Covered

Commit messages — see `writing-commit-messages`. Bodies and trailers are fine in a PR description; they are not fine in a commit.
