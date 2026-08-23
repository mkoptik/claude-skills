---
name: gh-pull-request-review
description: Reviews a GitHub pull request for correctness, security, data-loss risk, readability, and drift from repo conventions, then reports findings by severity and optionally posts them as a PR comment. Use whenever the user asks to review a PR, look over a pull request, check a branch before merging, asks whether a PR is ready, or gives a PR number or URL with no other instruction.
argument-hint: "[PR number/URL] [focus area]"
disable-model-invocation: true
allowed-tools: Bash(gh pr view *) Bash(gh pr diff *) Bash(gh pr checks *) Bash(gh issue view *) Bash(git log *) Bash(git show *) Bash(git diff *) Bash(git status *) Bash(rg *) Bash(gh api user --jq .login)
---

Reviewer GitHub login: !`gh api user --jq .login`

# Scope

Arguments: $ARGUMENTS

Interpret the arguments as a PR reference, a focus area, or both. If no PR is named,
review the PR for the current branch. If a focus area is given, weight it heavily but
still report anything Blocker- or High-severity found outside it.

Establish the target before reviewing:

```
gh pr view --json number,title,body,url,headRefName,baseRefName,files,additions,deletions
gh pr diff
```

Review the diff, not the files. Read surrounding context with `git show` or by reading
the file when a hunk is unclear, but only report on lines this PR touches — except where
a change breaks untouched code, which is in scope.

Skip generated files, lockfiles, and vendored directories unless the change to them is
itself suspicious (an unexplained lockfile bump, a hand-edited generated file).

If the diff is too large to review carefully in one pass, say so up front, then review
in order of risk: migrations and schema first, then security-relevant paths, then the
rest. Do not silently review a subset.

# Intent verification

Before reading code, establish what the PR claims to do:

- Does the description link a GitHub issue? If yes, read it with `gh issue view` and use
  it as the specification. If no, flag it as Medium and note what context is missing.
- Do the code changes actually resolve the linked issue, including the cases the issue
  describes? A PR that fixes the happy path of a bug that only manifests under
  concurrency has not fixed the bug.
- Is the description current with the code? Flag description drift: features described
  but not implemented, behavior changed but not documented, a stale "TODO in follow-up"
  that the diff already does, a title that no longer matches the contents.
- Does the diff contain changes unrelated to the stated purpose? Unexplained scope creep
  is a review finding, not a courtesy.

# Data and persistence

Treat this as the highest-stakes area. Any DDL or DML in migrations, seed scripts, or
one-off data scripts gets scrutiny before anything else in the PR.

- Destructive DDL: `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, type narrowing, or renames
  that discard data. Ask what happens to existing rows and whether the data is
  recoverable after deploy.
- `UPDATE` or `DELETE` without a bounded `WHERE`, or with a predicate that could match
  more than intended.
- Adding a `NOT NULL` column with no default or backfill, on a table that already has rows.
- Backfills that run inline in the migration on a large table — long transactions, lock
  escalation, statement timeouts leaving a partially applied migration.
- Rolling-deploy compatibility: during deploy, old code runs against the new schema and
  new code runs against the old schema. A migration that breaks either direction needs to
  be split across releases. Flag single-PR expand-and-contract as a Blocker.
- Is the migration reversible? Is there a down path, and if the change is genuinely
  irreversible, is that stated deliberately rather than by omission?
- Idempotency: can the migration be safely re-run after a partial failure?
- Ordering: does application code deploy before or after the migration, and does the PR
  say which?

# Correctness and robustness

Actively hunt for the cases the author did not consider. Do not just confirm the happy
path reads correctly.

- Edge cases: empty collections, zero, negative, null/None, single-element, off-by-one
  at boundaries, unicode and encoding, timezone and DST, integer overflow, floating-point
  comparison.
- Error handling gaps: swallowed exceptions, bare `except`/`catch` with no re-raise,
  errors logged and then execution continues on invalid state, error paths that leave
  partial writes behind.
- Improper error propagation: converting a real failure into a default return value,
  losing the original cause when re-wrapping, retrying a non-retryable error.
- Unhandled promises and un-awaited futures, fire-and-forget tasks whose failures nobody
  observes, missing `await` in a path with side effects.
- Resource leaks: file handles, sockets, DB connections and cursors, subprocesses,
  temp files, GPU memory, goroutines or threads, unclosed context managers, listeners
  never removed.
- Race conditions: check-then-act on shared state, non-atomic read-modify-write, missing
  locks or wrong lock granularity, TOCTOU on the filesystem, unsafe cache invalidation,
  assumptions that a single process holds a resource when it runs replicated.
- Retry and idempotency: is this operation safe to run twice? Network calls, message
  handlers, and webhook endpoints usually need to be.
- Concurrency limits and unbounded growth: queues without backpressure, unbounded caches,
  loading an entire result set into memory, N+1 query patterns.

# Security

- Injection paths: SQL string interpolation, shell invocation with user input,
  `shell=True`, path traversal in file operations, template injection, unsafe
  deserialization (`pickle`, `yaml.load`, `eval`).
- Authentication and authorization: new endpoints or handlers missing an authz check,
  authorization decided on client-supplied identity, object-level permission checks
  skipped on the read path.
- Secrets: credentials, tokens, keys, or internal hostnames in code, config, tests, or
  fixtures. Also flag secrets in logs and error messages.
- Input validation and output encoding at trust boundaries, including anything crossing
  the network or entering a template.
- Dependency changes: new or upgraded dependencies, what they pull in transitively,
  whether the addition is warranted. Flag a new dependency that replaces a handful of
  lines of standard library.
- Sensitive data handling: PII in logs, over-broad log levels on request bodies, data
  retained longer than the change implies.

# Readability and conventions

- Does the code follow the conventions already established in this repository, not
  general best practice in the abstract? Check neighboring files for naming, module
  layout, error-handling idiom, logging style, test structure, and typing discipline
  before calling something non-idiomatic. Use `rg` to confirm a pattern is actually the
  house style rather than one file's habit.
- If the repo has a linter, formatter, or type-checker config, judge style against that
  config and do not relitigate what it already enforces.
- Can a reader who did not write this understand it without reconstructing the author's
  reasoning? Flag deeply nested conditionals, functions doing several unrelated things,
  names that describe implementation instead of intent, and boolean parameters that make
  call sites unreadable.
- Flag abstraction that is not paying for itself: single-implementation interfaces,
  wrapper layers that only forward, configuration for something that will never vary.

# AI slop and comment hygiene

Flag generated-looking filler. Specifically:

- Docstrings that restate the signature: "Takes a user_id and returns a User." If the
  docstring adds nothing the signature already says, it should be deleted or rewritten to
  cover what is *not* obvious — invariants, units, ownership, failure modes, side effects.
- Comments that narrate the next line: `# increment the counter` above `counter += 1`.
  Comments should explain why, not what.
- Section-header comments over self-evident blocks, banner comments, decorative dividers.
- Inflated prose: "comprehensive", "robust", "seamlessly", "powerful", "leverages",
  emoji headers in code comments, marketing tone in technical documentation.
- Docstrings describing behavior the function does not have — parameters that no longer
  exist, raised exceptions it never raises, examples that would not run.
- Defensive scaffolding with no cause: try/except around code that cannot fail,
  null checks on values that are structurally non-null, redundant validation already
  done by the caller.
- Boilerplate tests that assert the mock was called rather than that the behavior holds.

State the deletion directly: name the comment or docstring and say to remove it. Do not
soften it into a suggestion to "consider" removing it.

# Tests and observability

- Do the changes have tests, and do the tests exercise the failure paths rather than
  only the happy path?
- Do existing tests still make sense, or were assertions weakened to make them pass?
- Are new failure modes observable — is there a log or metric on the paths that can now
  fail, at a level someone will actually see?

# Severity

Assign one level per finding:

- **Blocker** — data loss, security vulnerability, breaks production on deploy, or the PR
  does not do what it claims. Must be fixed before merge.
- **High** — a bug that will surface in normal use, a resource leak, an unhandled error
  path on a real code path, a missing authz check on a low-traffic route.
- **Medium** — correctness risk under uncommon conditions, missing tests on new logic,
  missing issue link, description drift, convention violations that will confuse maintainers.
- **Low** — readability, naming, structure, redundant code.
- **Nit** — cosmetic, non-blocking, explicitly optional.

Be strict about severity inflation. A style preference is not High. If a section of the
diff is clean, say so plainly instead of manufacturing findings to fill it out.

# Output

Produce two things.

Number every finding. Numbering runs continuously from 1 across the whole review, in
output order, and does not restart at each severity heading — so the author can say
"disagree with 4" and mean one specific thing. Use the same number for the same finding
in the console and in the PR comment.

Give every finding a headline: a short phrase, ten words maximum, that names the defect.
The headline is what someone quotes when discussing the finding, and what a follow-up
comment reuses to identify it, so it has to stand on its own without the body text.

- Name the defect, not the fix. "Connection leaked on retry path", not "Close the
  connection".
- Be specific enough to distinguish it from every other finding in the review. "Missing
  error handling" is useless when three findings could carry that label; "Migration
  failure leaves table half-backfilled" is not.
- No severity words. The severity label already says it, and "Critical security issue"
  spends the whole budget saying nothing.
- Sentence case, no trailing period.

Use the identical headline text in the console and in the PR comment.

**Console.** Findings grouped by severity, Blocker first, then in file order within each
group. One finding per issue, headline on the numbered line and the location beneath it:

```
[Blocker]
  1. Unbounded result set loaded into memory
     path/to/file.py:142
     Problem:  <what is wrong, in one or two sentences>
     Why:      <the concrete consequence — the failure that occurs, not "this is bad practice">
     Fix:      <specific suggested change>

  2. Migration drops column before backfill completes
     path/to/other.py:17
     ...
```

Keep the console form as bare `path:line`, with no markdown link and no URL. Terminals
and editors detect that pattern and make it clickable to the local file, which is what
you want while working in the repo; a GitHub URL there would send you to the browser
for code you already have checked out.

Open with a two-to-four line verdict: what the PR does, whether it matches its stated
intent and linked issue, and a merge recommendation of approve / approve with changes /
request changes. State the total count of findings and the count at each severity.

**PR comment.** The first line of the comment body, before anything else, must be
exactly:

```
🤖 Posted by Claude Opus on behalf of user @<login>.
```

Substitute the GitHub login resolved at the top of this skill, keeping the leading `@`
so it renders as a user link. Never post the comment with `<login>` unresolved.

Follow it with the verdict paragraph, then an `###` heading per severity level present,
then the findings as a numbered list. Each item is the headline in bold, then an em dash,
then the body on the same or next line:

```markdown
4. **Unbounded result set loaded into memory** —
   <problem, consequence, and suggested fix in two or three sentences>
```

Continue the numbering across headings rather than letting each list restart — write the
list items with explicit numbers (`4.`, `5.`) so GitHub renders the intended sequence.
Omit severity levels with no findings. Keep it scannable — a reviewer reads this in a
browser, not a terminal, and the bold headlines are what they skim.

Do not attach a location to a finding. The headline line is the headline and nothing else
— no `path:line`, no filename, no permalink after the em dash. Write the body so the
author knows which code it is about by naming the function, migration, or branch of the
conditional; the console output carries the exact locations for whoever is working in the
repo.

Links are still fine where one genuinely helps — the linked issue, a doc or spec, a prior
PR, or a permalink to code *elsewhere* in the repo that the finding depends on. What is
being dropped is the routine per-finding location stamp, not links in general.

Print the comment body and ask for confirmation before posting. Post with
`gh pr comment <number> --body-file <file>` only after the user approves. Never post
automatically, never edit or resolve existing review comments, and never approve or
request changes through the GitHub review API.

# Discipline

- Report only what you verified in the diff. If you suspect an issue but cannot confirm
  it without running the code, say so and mark it as needing verification rather than
  asserting it.
- Do not edit files, stage, commit, or push. If the user wants a fix applied, they will ask.
- Do not summarize the diff back at length. The author knows what they wrote.
- No praise padding. "Looks good" on a clean section is enough.