---
name: rebase
description: Rebase the current branch onto the upstream default branch and rework the series into functionally minimal commits, folding similar ones and enforcing the project's commit message rules. Use when asked to rebase, rework, tidy, squash, or fold a branch's commits.
---

# Rebase and rework a branch

Rebase onto the upstream default branch, then rework the series so each commit
is functionally minimal and similar commits are folded, with no known issue or
regression introduced.

The rebase is usually the small half. The rework is the work.

## 0. Refuse outside a git repository

This rewrites history, so there has to be history. Check first:

```sh
git rev-parse --is-inside-work-tree
```

If that fails, stop and say so; there is nothing here to rebase. Do not offer
to run `git init`, because an empty repository has no series to rework and
initialising one is not what was asked.

Two more things make the branch unreworkable, and both are worth catching
before any surgery rather than halfway through it:

- A detached HEAD. `git symbolic-ref -q HEAD` fails, and there is no branch to
  move. Ask which branch was meant.
- No commits of its own. `git rev-list --count origin/<default>..HEAD` is 0, so
  the rebase may still be real but there is nothing to rework.

An unrelated dirty tree is not a blocker by itself, but uncommitted work must be
backed up before the reconstruction in step 3 touches it. See Safety.

## 1. Establish the target and the baseline

Find the upstream default branch rather than assuming `origin/main`. Locally,
without touching the network:

```sh
git symbolic-ref --short refs/remotes/origin/HEAD    # e.g. origin/main
```

If that ref is missing, `git remote set-head origin --auto` restores it, or
`git remote show origin | sed -n 's/.*HEAD branch: //p'` asks the remote.

Then:

```sh
git fetch origin
git rev-parse origin/<default> HEAD
git merge-base origin/<default> HEAD
```

If merge-base equals the upstream tip, the branch is already on top of it and
the rebase is a no-op. Say so plainly and move to the rework; do not narrate a
rebase that did nothing.

If the branch is behind, rebase first and resolve conflicts before reworking.
Beware: a branch's tracking upstream may be its own remote branch, so "behind
4" in `git status` is often measured against `origin/<branch>`, not against the
default branch. Check with `git rev-parse --abbrev-ref @{upstream}`.

Before touching anything, record the baseline:

- Run the project's full gate (see AGENTS.md, CLAUDE.md, CONTRIBUTING.md, or a
  `scripts/test.sh`) and record that it passes at the current tip. A rework
  that "fixes" a pre-existing failure is a rework hiding a change.
- `git tag pre-rework HEAD` or note the SHA. The final tree must match it.

## 2. Survey the series for what to fold

Read every commit message and stat:

```sh
git log --format='%H%n%s%n%b%n---' --stat origin/<default>..HEAD
```

The strongest fold signal is churn: content one commit adds that a later
commit in the same series removes, replaces, or relocates. Measure it:

```sh
for f in $(git diff --name-only origin/<default>..HEAD); do
  echo "$f  added:$(git diff A B -- $f | grep -c '^+[^+]')  removed_later:$(git diff B C -- $f | grep -c '^-[^-]')"
done
```

Fold when:

- A commit only makes sense as an edit to an earlier one, and could not stand
  alone against the upstream branch. That is one change recorded as drafts.
- Two commits are the same kind of work, done for the same reason, verified the
  same way (for example two structural extractions with no behavior change).
- A commit writes prose into one file that a later commit moves to another.
  Reorder so the destination exists first and each commit writes where its text
  finally lives.

Do not fold:

- Unrelated subsystems, however similar the mechanics.
- A feature and a separate control it motivated. Each stays revertible.
- A code change and an unrelated convention it happened to prompt.

## 3. Reconstruct rather than edit in place

Interactive rebase is unavailable in this environment (`git rebase -i` and
`git add -i` are not supported). Reconstruct the series instead:

1. Snapshot every final file outside the repo, into a scratch directory that
   mirrors the tree. `cp --parents` is GNU-only, so create the parent directory
   yourself:

   ```sh
   for f in $(git diff --name-only origin/<default>..HEAD); do
     mkdir -p "$S/final/$(dirname "$f")" && cp "$f" "$S/final/$f"
   done
   ```

   Also save `git diff > backup.diff` when there is uncommitted work.
2. `git reset --mixed origin/<default>`. HEAD and index move; the working tree
   keeps the final content.
3. Build each commit by writing the exact content that commit should have, then
   `git add` only its files and commit.
4. For a file split across commits, construct the intermediate from the final
   version by reverting the later commit's hunks with exact string replacement.
   Assert each replacement matched exactly once, so a silent miss fails loudly.
5. Unchanged commits can simply be replayed with `git cherry-pick`.

Watch for `git mv` staging a rename early: it will leak into whichever commit
lands first. Check `git status --short` before every commit.

## 4. Per-commit quality pass

For each commit, before it lands:

- Run /simplify (the `code-simplifier` agent) over that commit's new code. Ask
  it to report rather than edit, so nothing is changed under you, and tell it
  the house comment style is deliberate so it does not propose deleting
  reasoning. Apply only genuine simplifications: duplicated logic, a needless
  binding, an argument that is always the same, a helper with one caller that
  adds nothing, dead code, a visibility wider than its callers.
- Run `commentflow --check` on every file it touches that `commentflow`
  handles. It covers C, C++, Objective-C, Rust, POSIX shell and assembly;
  `commentflow --help` prints the exact extension list, and JavaScript,
  Markdown and HTML are not among them, so a browser-only or docs-only commit
  has nothing to check. A formatter run after it (`cargo fmt`, `shfmt`) has the
  last word, so a blank line `commentflow` wants and the formatter removes is
  not a failure to chase. Check whether the upstream branch already fails it
  before treating a hit as yours.
- Run the project's formatter and linter gates.

## 5. Commit messages

Follow https://cbea.ms/git-commit/:

1. Separate subject from body with a blank line.
2. Subject at most 50 characters.
3. Capitalize the subject.
4. No trailing period on the subject.
5. Imperative mood in the subject.
6. Wrap the body at 72 characters.
7. Body explains what and why, not how.

Project rules on top of those:

- Never begin a subject with "Say".
- No backticks and no em dashes anywhere in the message. Use commas, colons,
  parentheses, or separate sentences.
- Strip every machine attribution marker. `Co-Authored-By:` naming an assistant
  or a bot, `Claude-Session:` and any other session or conversation link,
  "Generated with", "Created by", and tool footers all come out. The rework is
  rewriting these messages anyway, so a marker that survives it was left there
  deliberately, and none of them should be.
- Keep `Co-Authored-By:` that names a person. A human co-author is a real
  attribution and removing it takes credit away from somebody, which is a worse
  error than leaving a tool footer in. When a trailer is ambiguous, ask rather
  than guess: an address at a company domain is a person, `noreply@anthropic.com`
  and the like are not.
- Keep bodies short: subject plus a paragraph or two. Detail belongs in code
  comments, not in the log.

Verify mechanically rather than by eye:

```sh
git log origin/<default>..HEAD --format='%s' | awk '{print length($0)": "$0}'
git log origin/<default>..HEAD --format='%b' | awk 'length($0)>72 {print "LONG: "$0}'
git log origin/<default>..HEAD --format='%B' | grep -n '`\|—\|–'
git log origin/<default>..HEAD --format='%s' | grep -i '^say'

# Machine attribution, which must be empty. Human co-authors are excluded here
# because they are allowed to stay.
git log origin/<default>..HEAD --format='%B' \
  | grep -niE 'claude-session|generated with|created by .*(claude|copilot|cursor|codex)|co-authored-by:.*(noreply@anthropic|\[bot\]|copilot|cursor)'

# Every co-author trailer, to read once and confirm each names a person.
git log origin/<default>..HEAD --format='%B' | grep -i '^co-authored-by:' | sort -u
```

Do not restate the diff in the body. Say what changed and why it changed.

## 6. Never commit the working notes

`TODO.md` and `DONE.md` stay out of the repository. Confirm after building the
series:

```sh
git log origin/<default>..HEAD --name-only --format= | sort -u | grep -E 'TODO\.md|DONE\.md'
```

Keep them untracked by maintaining `.git/info/exclude`. Editing them is fine;
committing them is not.

## 7. Verify before reporting

Three checks, all mechanical:

```sh
# The rework changed history and nothing else.
git diff pre-rework HEAD

# Every commit builds, so the series bisects. Substitute the project's own
# cheapest build check for the cargo line.
for c in $(git log --format=%h --reverse origin/<default>..HEAD); do
  git checkout -q $c
  cargo check --all-targets >/dev/null 2>&1 && echo "builds  $c" || echo "BROKEN  $c"
done
git checkout -q <branch>
```

Then run the full gate at the tip and report its exit code and test counts.

`git diff pre-rework HEAD` must be empty. If it is not, either you lost work or
you smuggled a change into a rework, and both need saying out loud rather than
explaining away.

Where a commit's claim is checkable, check it. If a message says a test would
have caught something, break the thing and watch the test fail, then restore.

## Safety

- Take a backup before any surgery on uncommitted work, and diff against it
  afterwards. Splitting a working tree by hand loses files quietly.
- Never `git checkout -- <file>` while uncommitted work exists in that file: it
  reverts to the index, which discards the user's edits, not just yours. Restore
  from your own backup instead.
- Never commit, amend, rebase, reset, or push unless the task asked for it.
  A request to rebase and rework authorizes rewriting this branch's own history
  and nothing further.
- Do not squash away a commit whose content you cannot account for in the final
  tree. If a commit's changes are absent from the tip, find out why before
  dropping it.
