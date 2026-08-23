---
name: gh-pull-request-fix
description: Reads a GitHub pull request review comment, implements the findings the user selected as one commit per finding, and offers to post a follow-up comment with a status table linking each fix to its commit. Use whenever the user gives a review comment link or reference and says which findings to fix, asks to address review feedback, wants to act on a code review, or says something like "fix 1, 3 and 5 from that review".
argument-hint: "[comment URL] [which findings to fix or skip]"
disable-model-invocation: true
allowed-tools: Bash(gh pr view *) Bash(gh pr diff *) Bash(gh api repos/*/issues/comments/*) Bash(gh api repos/*/pulls/comments/*) Bash(git status *) Bash(git log *) Bash(git diff *) Bash(git show *) Bash(git add *) Bash(git commit *) Bash(rg *) Bash(gh api user --jq .login)
---
 
Reviewer GitHub login: !`gh api user --jq .login`
 
# Inputs
 
Arguments: $ARGUMENTS
 
Expect two things in the arguments: a reference to a review comment, and a selection of
which findings to act on. Either may be missing; ask for what is missing rather than
guessing.
 
**Resolving the comment.** A comment URL ends in `#issuecomment-<id>` or
`#discussion_r<id>`. Fetch the body directly:
 
```
gh api repos/<owner>/<repo>/issues/comments/<id> --jq .body      # issuecomment-<id>
gh api repos/<owner>/<repo>/pulls/comments/<id> --jq .body       # discussion_r<id>
```
 
If the user gives only a PR number or says "the last review", use
`gh pr view <n> --comments` and pick the most recent review comment, then state which
comment you selected before doing anything else.
 
**Resolving the selection.** Findings in these reviews are numbered continuously. The
user's selection may be numbers ("fix 1, 3, 7"), numbers plus exclusions ("all the
blockers, skip 4"), or severity bands ("everything at High and above"). Resolve it to an
explicit list before starting.
 
Then restate the plan and get confirmation before writing any code:
 
- the findings you will fix, by number and one-line description
- the findings the user explicitly declined, by number
- anything in the selection you could not map to a finding
# Scope discipline
 
Act only on findings the user named. This is the strictest rule in this skill.
 
- Do not fix a finding the user did not select, even a Blocker, even one you agree with.
  If you think an unselected finding is serious, say so in chat once and move on.
- Do not fix problems you notice yourself that were not in the review at all.
- Do not opportunistically reformat, rename, or tidy code you are touching for another
  reason. Each commit contains the fix and nothing else.
- If a selected finding cannot be fixed without also changing something the user did not
  select, stop and ask before proceeding.
# Before starting
 
- Confirm the working tree is clean with `git status --short`. If it is not, stop and ask
  — do not fold the user's uncommitted work into a fix commit.
- Confirm you are on the PR's head branch. If not, stop and say so rather than switching
  branches yourself.
- Read each selected finding fully, including the suggested fix. The suggestion is a
  proposal, not an instruction: if it is wrong or would break something, say so and
  propose an alternative instead of implementing it.
# Implementing a fix
 
Work through the selected findings one at a time, in the review's order unless a
dependency between them forces a different order.
 
For each finding:
 
1. Read the code at the referenced location and enough surrounding context to understand
   the change. The review's permalinks point at a specific commit — the code may have
   moved since, so locate it by content rather than trusting the line number.
2. Make the smallest change that actually resolves the finding. Do not extend it into a
   refactor.
3. If the fix changes behavior, update or add the test that covers it, and update any
   docstring or documentation the change invalidates. These belong in the same commit.
4. Verify before committing: run the relevant tests, linter, or type-checker if the repo
   has them. If verification fails, fix it or stop — do not commit a broken state and
   plan to fix it in a later commit.
5. Stage only the files this fix touched, then commit.
If a finding turns out to be a false positive, do not invent a change to satisfy it.
Stop, explain why the finding does not hold, and let the user decide whether to skip it.
A finding rejected this way is reported as not fixed, with that reason.
 
# Commits
 
One commit per finding, no exceptions. Never squash two findings into one commit, never
amend a previous commit, never rebase.
 
Message format:
 
```
<type>: <what changed, imperative, one line>
 
Addresses review finding #<n>: <the finding's one-line description>
<comment URL>
```
 
Use the repo's existing commit conventions for `<type>` and the subject line — check
`git log --oneline -30` for whether it uses Conventional Commits, a ticket prefix, or
plain sentences, and match it rather than imposing a format.
 
Capture each commit's SHA as you go; you need them for the table.
 
Do not push. When every fix is committed, report the commit list and tell the user the
commits are local, so they can review before pushing. If they ask you to push, that is a
separate confirmed action.
 
# Follow-up comment
 
After the commits are made, offer to post a follow-up comment on the PR. Offer once —
do not post unprompted, and do not re-offer if the user declines.
 
Commit links only resolve on GitHub once the commits are pushed. If the branch has not
been pushed, say so and let the user push first; do not post a comment full of dead
links.
 
## Format
 
First line, exactly:
 
```
🤖 Posted by Claude Opus on behalf of user @<login>.
```
 
Then one sentence naming the review comment being responded to, as a link.
 
Then the table:
 
```
| # | Finding | Status | Commit |
|---|---------|--------|--------|
| 1 | Unbounded result set loaded into memory in the scorer | ✅ Fixed | [`a3f9c1e`](https://github.com/<owner>/<repo>/commit/<full-sha>) |
| 4 | Docstring restates the signature | ❌ Not fixed | — |
```
 
Rules for the table:
 
- **It contains only findings the user explicitly asked to fix or explicitly declined.**
  A finding the user never mentioned does not appear in the table at all — not as a row,
  not as a footnote, not as "not addressed". Silence about it is the correct output.
- Keep the original finding numbers from the review. Do not renumber.
- `✅ Fixed` for implemented findings, `❌ Not fixed` for ones the user chose to skip or
  that were rejected as false positives.
- Finding column: one short line, reworded to fit the table. Do not paste the full
  finding text.
- Commit column: the short SHA as link text, linking to
  `https://github.com/<owner>/<repo>/commit/<full-sha>`. Use `—` for not-fixed rows.
- One row per finding. If a fix genuinely required two commits, link the primary one and
  mention the second in a note below the table rather than adding a row.
Below the table, one line per `❌` row explaining the decision:
 
```
**4.** Not fixed — repo convention requires docstrings on all public methods.
```
 
Nothing else. No summary paragraph, no restatement of what was fixed, no thanks.
 
## Posting
 
Print the full comment body and ask for confirmation. Post with
`gh pr comment <number> --body-file <file>` only after the user approves. Never edit or
delete existing comments, never resolve review threads, and never approve the PR.
 
# Discipline
 
- Report honestly. If a fix is partial, say which part is unaddressed rather than marking
  the row `✅ Fixed`.
- If you could not fix something you agreed to fix, do not quietly drop it — report it
  and ask how to proceed before producing the table.
- Do not summarize the review back at the user. They wrote the selection; they know it.