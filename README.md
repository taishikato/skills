# skills

Personal collection of [Claude Code](https://docs.claude.com/en/docs/claude-code) skills.

## `/code-review-turbo-max`

An enhanced version of [`code-review-turbo`](https://gist.github.com/nolanlawson/4150b0ca9640654c256b324fac0d5253) by [@nolanlawson](https://github.com/nolanlawson).

The original runs a multi-reviewer code review and cross-references the findings to separate real bugs from hallucinations. `code-review-turbo-max` extends it from a smaller reviewer pool to a **five-reviewer panel**, then has Claude act as an objective judge over all of their findings.

### The five reviewers

| # | Reviewer | Lens |
|---|----------|------|
| 1 | **Cursor Bugbot** | Automated PR review via `@cursor review` |
| 2 | **Claude sub-agent** (generated prompt) | Bug-focused `REVIEW_PROMPT` |
| 3 | **Codex** (generated prompt) | Same bug-focused `REVIEW_PROMPT` |
| 4 | **Claude pr-review-toolkit specialists** | Quality dimensions (tests, silent failures, type design, comments) |
| 5 | **Codex `requesting-code-review` skill** | Plan alignment, architecture, production-readiness |

Reviewers 2 and 3 share the exact same bug-focused prompt so their results can be directly compared. Reviewers 4 and 5 deliberately use *different* lenses so the perspectives don't overlap.

### How it works

1. **Ensure a PR exists and trigger Bugbot** — creates a draft PR if needed, posts a `@cursor review` comment, detects stale reviews, and polls for Bugbot's findings (filtering out already-resolved threads via the GraphQL API).
2. **Build a shared review prompt** from the PR diff, title, description, and base/head branches. The prompt prioritizes functional bugs first, then KISS / DRY / missing tests / performance / accessibility.
3. **Run all four AI reviewers in parallel** (reviewers 2–5).
4. **Cross-reference and validate** — Claude compiles every finding *before* doing any of its own research, so it stays an objective judge rather than becoming a biased sixth reviewer. It then verifies each finding against the actual source (reading code, running `EXPLAIN ANALYZE`, checking test files) to flag real issues vs. hallucinations.
5. **Final report** — issues grouped by severity, a dismissed-findings section, a reviewer-agreement table showing who caught what, and a clear merge recommendation.

### Usage

```
/code-review-turbo-max [pr-number]
```

If no PR number is given, it detects the PR from the current branch.

### Requirements

- [`gh`](https://cli.github.com/) CLI authenticated against the repo
- [Cursor Bugbot](https://cursor.com/bugbot) enabled on the repo
- [`codex`](https://github.com/openai/codex) CLI available locally
- The `pr-review-toolkit` plugin (for reviewer #4) and Codex's `requesting-code-review` skill (for reviewer #5)

### Credit

Original concept and `code-review-turbo` skill by [@nolanlawson](https://github.com/nolanlawson) — see the [gist](https://gist.github.com/nolanlawson/4150b0ca9640654c256b324fac0d5253). This `-max` variant adds the two extra reviewers (pr-review-toolkit specialists and Codex's native review skill) and the objective-judge cross-referencing step.
