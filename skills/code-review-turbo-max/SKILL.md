---
name: code-review-turbo-max
description: Run a five-reviewer code review on the current branch's PR. Waits for Cursor Bugbot, then runs four reviewers in parallel — a Claude sub-agent and Codex (both on the generated prompt), Claude via the pr-review-toolkit specialist agents, and Codex via its native requesting-code-review skill — then cross-references all findings to filter out hallucinations. Use when you want a thorough, multi-perspective code review before merging.
metadata:
  disable-model-invocation: 'true'
  argument-hint: '[pr-number]'
allowed-tools: Bash(gh:*) Bash(codex:*) Bash(cat:*) Bash(tee:*) Bash(sleep:*) Agent Read Grep Glob Write(/tmp/*)
---

# Code Review Turbo

Five-reviewer code review with cross-referencing to separate real bugs from hallucinations. The five independent reviewers are:

1. **Cursor Bugbot** (Step 1)
2. **Claude sub-agent** on the generated `REVIEW_PROMPT` (Step 3a)
3. **Codex** on the same generated `REVIEW_PROMPT` (Step 3b)
4. **Claude via the pr-review-toolkit specialist agents** (Step 3c)
5. **Codex via its native `requesting-code-review` skill** (Step 3d)

Reviewers 2 and 3 share the bug-focused `REVIEW_PROMPT`. Reviewers 4 and 5 deliberately use *different* lenses (specialist quality dimensions, and plan-alignment / architecture / production-readiness) so the perspectives don't overlap.

## Step 1: Ensure PR Exists and Trigger Bugbot

Determine the PR number. If `$ARGUMENTS` is provided, use that as the PR number. Otherwise, try to detect it from the current branch:

```
gh pr view --json number,isDraft -q '{number: .number, isDraft: .isDraft}'
```

### If no PR exists

Create a **draft** PR for the current branch and comment to trigger Bugbot:

```
gh pr create --draft --fill
gh pr comment <number> --body "@cursor review"
```

Tell the user you created a draft PR and triggered Bugbot.

### If the PR exists and is a draft

Check whether a `@cursor review` or `@bugbot review` trigger comment already exists:

```
gh pr view <number> --json comments --jq '.comments[].body'
```

If no trigger comment is found, add one:

```
gh pr comment <number> --body "@cursor review"
```

### If the PR exists and is NOT a draft

Bugbot runs automatically on non-draft PRs, so no trigger comment is needed — unless the review is stale (see below).

### Check for stale Bugbot review

Bugbot posts **one comment per issue** it finds, and resolves individual comments when the issue is fixed. So "Bugbot has reviewed" means there are Bugbot comments on the PR, and "stale" means commits were pushed after Bugbot's review pass.

To detect staleness:

1. Get the timestamps of ALL Bugbot comments (it posts multiple — one per issue):
   ```
   gh pr view <number> --json comments --jq '[.comments[] | select(.author.login | test("bugbot|cursor"; "i")) | .createdAt]'
   ```
   Also check review comments (inline on the diff):
   ```
   gh api repos/{owner}/{repo}/pulls/<number>/comments --jq '[.[] | select(.user.login | test("bugbot|cursor"; "i")) | .created_at]'
   ```

2. Get the timestamp of the most recent commit on the PR:
   ```
   gh pr view <number> --json commits --jq '.commits | last | .committedDate'
   ```

3. If the latest commit is **newer** than ALL of Bugbot's comments (or if Bugbot has never commented), the review is stale. Post a new trigger comment:
   ```
   gh pr comment <number> --body "@cursor review"
   ```
   Tell the user: "Bugbot's review is stale (commits landed after its last review). Triggered a fresh review."

4. If Bugbot has comments that are **newer** than the latest commit, the review is current. Proceed with the existing comments.

### Poll for Bugbot's review

If you triggered a fresh review (due to staleness, draft PR, or new PR), poll for Bugbot comments to appear. Run individual commands — do NOT write a bash loop:

1. Run `gh pr view <number> --json comments --jq '[.comments[] | select(.author.login | test("bugbot|cursor"; "i"))]'`
2. If empty, run `sleep 30`
3. Repeat up to 30 times (15 minutes total)

If Bugbot never shows up, warn the user and ask whether to proceed anyway or keep waiting.

### Collect Bugbot's findings

Bugbot posts inline review comments (one per issue) and resolves them when the issue is fixed. You must use the **GraphQL API** to check resolution status, because the REST API does not expose it.

First, get the repo owner and name:
```
gh repo view --json owner,name --jq '.owner.login + "/" + .name'
```

Then fetch all review threads with their resolution status and filter to Bugbot/Cursor comments:
```
gh api graphql -f query='
  query {
    repository(owner: "<OWNER>", name: "<REPO>") {
      pullRequest(number: <NUMBER>) {
        reviewThreads(first: 100) {
          nodes {
            isResolved
            comments(first: 10) {
              nodes {
                author { login }
                body
                path
                line
              }
            }
          }
        }
      }
    }
  }
'
```

From the result, keep only threads where:
1. `isResolved` is `false`, AND
2. At least one comment has an `author.login` matching `bugbot` or `cursor` (case-insensitive)

**Ignore all resolved threads** — these are issues Bugbot already confirmed as fixed.

Also check top-level PR comments (these are rare for Bugbot but possible):
```
gh pr view <number> --json comments --jq '[.comments[] | select(.author.login | test("bugbot|cursor"; "i")) | select(.isMinimized | not)]'
```

Save all active (non-resolved) Bugbot findings for later comparison.

## Step 2: Build the Review Prompt

Gather the PR context by running these commands:

```
gh pr diff <number>
gh pr view <number> --json title,body,baseRefName,headRefName
```

Then construct the following review prompt (referred to as `REVIEW_PROMPT` below). This EXACT prompt must be used for BOTH the sub-agent and Codex — do not alter it between the two:

---

**START OF REVIEW_PROMPT**

You are reviewing a pull request. Here is the diff:

<INSERT PR DIFF HERE>

PR title: <INSERT>
PR description: <INSERT>
Base branch: <INSERT>
Head branch: <INSERT>

Review this PR thoroughly. Focus on these categories IN ORDER OF IMPORTANCE:

### 1. Functional Bugs (MOST IMPORTANT)
Look for logic errors, off-by-one errors, null/undefined issues, race conditions, incorrect conditionals, missing edge cases, wrong variable usage, broken control flow, and any code that simply won't work as intended. This is BY FAR the most important category.

### 2. KISS Violations
Overly complex solutions where simpler ones exist. Unnecessary abstractions, premature generalizations, or convoluted logic.

### 3. DRY Violations
Duplicated logic that should be extracted. Copy-pasted code with minor variations.

### 4. Missing Tests
New functionality or bug fixes lacking appropriate test coverage.

### 5. Performance Issues
- For SQL queries: DO NOT GUESS what the query planner will do. Instead, run `EXPLAIN ANALYZE` on the actual local database to verify.
- For migrations: Will they lock tables for too long? Are they safe for large tables?
- For application code: N+1 queries, unnecessary allocations, missing batching, O(n^2) loops on large datasets.

### 6. Accessibility Issues
For any TSX/JSX files: missing aria labels, improper heading hierarchy, missing alt text, keyboard navigation issues, color contrast concerns.

DO NOT report:
- Code formatting or style issues (these are linted automatically)
- Minor TypeScript type issues (also linted)
- Nitpicks that don't affect correctness or maintainability

For each issue found, report:
- **File and line number** (from the diff)
- **Severity**: critical / high / medium / low
- **Category**: which of the above categories
- **Description**: what the issue is and why it matters
- **Suggestion**: how to fix it

Return a structured list grouped by severity (critical first, then high, medium, low).

**END OF REVIEW_PROMPT**

---

## Step 3: Run the Four Parallel Reviewers

Launch ALL FOUR of these at the same time (in parallel). 3a/3b share the generated `REVIEW_PROMPT`; 3c/3d use different lenses on purpose.

### 3a. Claude Sub-Agent (generated prompt)
Use the Agent tool to spawn a sub-agent with the full `REVIEW_PROMPT`. This agent should have access to Bash, Read, Grep, and Glob tools so it can run EXPLAIN queries and inspect code.

### 3b. Codex (generated prompt)
Run the EXACT SAME `REVIEW_PROMPT` through Codex. First write the prompt to a randomly-named temp file using the Write tool (e.g., `/tmp/review-prompt-<random-8-chars>.txt` — generate a unique random suffix to avoid collisions with concurrent agents), then pipe it via stdin:

```
codex exec --full-auto - < /tmp/review-prompt-<random>.txt
```

The `--full-auto` flag prevents Codex from prompting for approval on shell commands (e.g., EXPLAIN queries). The `-` tells it to read the prompt from stdin.

### 3c. Claude via pr-review-toolkit specialist agents

This reviewer reproduces the `/pr-review-toolkit:review-pr` workflow. **Do NOT spawn a sub-agent that itself runs the `/pr-review-toolkit:review-pr` command** — that command is an orchestrator that fans out to its own sub-agents via the Task tool, and a sub-agent cannot spawn further sub-agents (the nesting fails). Instead, **you (the top-level orchestrator) launch the specialist agents directly** via the Agent tool, in parallel.

Decide which specialists apply based on the diff (from `gh pr diff <number> --name-only` and the diff content):

- **Always:** `pr-review-toolkit:code-reviewer` (general quality + bug detection)
- **If test files changed:** `pr-review-toolkit:pr-test-analyzer`
- **If error handling / catch blocks / fallbacks changed:** `pr-review-toolkit:silent-failure-hunter`
- **If new or modified types:** `pr-review-toolkit:type-design-analyzer`
- **If comments/docs added or changed:** `pr-review-toolkit:comment-analyzer`

Give each specialist the PR number and base/head branch so it can run its own `git diff` / `gh pr diff`. Collect all of their reports — together they form the single reviewer-#4 finding set (tag each finding with the specialist that raised it).

### 3d. Codex via its native requesting-code-review skill

This reviewer uses Codex running its OWN `requesting-code-review` skill (the Codex-side copy), which applies a different lens than the bug-focused `REVIEW_PROMPT`: plan alignment, architecture, and production-readiness.

The skill's `SKILL.md` is just an orchestration wrapper ("dispatch a reviewer subagent"); the actual reviewer prompt lives in its bundled `code-reviewer.md`. So point Codex at its own skill directory and pre-fill the placeholders.

1. Get the base/head SHAs from the PR (no raw `git` needed — stay within `gh`):
   ```
   gh pr view <number> --json baseRefOid,headRefOid --jq '"BASE=\(.baseRefOid) HEAD=\(.headRefOid)"'
   ```
2. Write a temp file (random suffix, e.g. `/tmp/review-rcr-<random-8-chars>.txt`) instructing Codex to follow its own skill. Include:
   - An instruction to read and follow the reviewer template at
     `/Users/taishi/.codex/plugins/cache/openai-curated/superpowers/<hash>/skills/requesting-code-review/code-reviewer.md`
     (resolve `<hash>` at runtime with a glob — the directory name is a content hash and can change).
   - The pre-filled placeholders so Codex doesn't have to guess:
     - `{DESCRIPTION}` ← PR title + body
     - `{PLAN_OR_REQUIREMENTS}` ← PR description (or a referenced plan file, if any)
     - `{BASE_SHA}` ← from above
     - `{HEAD_SHA}` ← from above
   - A reminder to emit the skill's output format (Strengths / Critical / Important / Minor / Recommendations / Assessment).
3. Run it:
   ```
   codex exec --full-auto - < /tmp/review-rcr-<random>.txt
   ```
   `--full-auto` lets Codex run `git diff {BASE_SHA}..{HEAD_SHA}` itself to read the changes.

> If the Codex-side `requesting-code-review` skill directory does not exist on this machine, fall back to embedding the `code-reviewer.md` template contents (with placeholders filled) directly in the temp file.

## Step 4: Cross-Reference and Validate

**CRITICAL: DO NOT do any of your own code research, file reading, or EXPLAIN queries until ALL FIVE reviewers (Bugbot, Claude sub-agent, Codex, pr-review-toolkit specialists, Codex requesting-code-review) have returned their results.** If you investigate the code first, you will form your own opinions and become a 6th reviewer with a veto over the other 5 — biased toward confirming your own findings and dismissing theirs. The whole point of this step is to be an OBJECTIVE judge of five independent reviewers.

### Step 4a: Compile findings FIRST (no research yet)

Collect and deduplicate all findings from the five reviewers into a single list. For each unique issue, note which reviewer(s) reported it (treat #4's specialist agents as a single "pr-review-toolkit" source, but keep the specialist name in the note). Do NOT yet judge whether the issues are real — just organize them.

### Step 4b: NOW do your own research to validate

Only after compiling the full list, go through each finding and verify it:

- **Read the actual source code** around each reported issue (not just the diff)
- **Run EXPLAIN ANALYZE** on any flagged SQL queries against local database
- **Check test files** to see if flagged "missing tests" actually exist
- **Trace the logic** for any reported functional bugs — actually verify the bug is real

For each unique issue, determine:
- Is it a **real issue** (confirmed by your investigation)?
- Is it a **hallucination** (the code doesn't actually have this problem)?
- Which reviewers found it and which missed it?

Be especially careful not to dismiss a finding just because only one reviewer reported it — sometimes the lone dissenter found the most critical bug. Note that #4 and #5 review different dimensions (quality/architecture/plan-alignment) than #1–#3 (bugs), so a finding unique to #4 or #5 is expected, not suspicious.

## Step 5: Final Report

Present the validated findings in this format:

### Critical Issues
(issues you confirmed are real and need fixing before merge)

### High Issues
(real issues that should be fixed)

### Medium Issues
(real but lower-risk issues)

### Low Issues
(minor improvements, optional)

### Dismissed Findings
(issues reported by reviewers that turned out to be hallucinations or false positives — briefly explain why each was dismissed)

### Reviewer Agreement Summary
A table showing which reviewer found which real issue. Columns:
- **Bugbot** — #1
- **Claude (prompt)** — #2, generated `REVIEW_PROMPT` sub-agent
- **Codex (prompt)** — #3, generated `REVIEW_PROMPT`
- **pr-review-toolkit** — #4, the specialist agents
- **Codex (RCR)** — #5, Codex requesting-code-review skill

| Issue | Bugbot | Claude (prompt) | Codex (prompt) | pr-review-toolkit | Codex (RCR) | Verdict |
|-------|--------|-----------------|----------------|-------------------|-------------|---------|
| ...   | ...    | ...             | ...            | ...               | ...         | ...     |

End with a clear **merge recommendation**: ready to merge, merge after fixes, or needs significant rework.