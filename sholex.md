# Building a "Sholex Review" Pre-PR Skill

A complete, self-contained guide for capturing Sholex's PR review standards from
past comments and turning them into a Claude Code skill you can run against your
own diff **before** opening a pull request.

The goal is not to imitate Sholex as a person, but to extract the review
standards and reasoning that Sholex consistently applies — including the things
Sholex catches that other reviewers let through — and encode them as a reusable
rubric.

---

## Overview of the workflow

1. **Collect** Sholex's review comments from past PRs into a corpus.
2. **Distill** that corpus into patterns and the reasoning behind them.
3. **Encode** the patterns as a `SKILL.md` reviewer rubric.
4. **Calibrate** the skill against PRs Sholex already reviewed.
5. **Run** it on your diff before every PR.

---

## Prerequisites

On the controlled Mac you'll need:

- **GitHub CLI (`gh`)** — for pulling comments via the API.
- **Claude Code** — to run the distillation and, later, the skill itself.
- **Read access** to the repository whose PRs you want to mine.
- Sholex's **GitHub login** (the `@username`, not the display name). You can
  find it on any of Sholex's past comments in the GitHub UI.

### Install and authenticate `gh` on macOS

```bash
# Install (Homebrew)
brew install gh

# Authenticate — choose HTTPS and follow the browser flow
gh auth login

# Verify you're logged in and can see the repo
gh auth status
gh repo view OWNER/REPO
```

Throughout this guide, replace:

- `OWNER/REPO` with your repository (e.g. `plainset/numkeep`)
- `SHOLEX_LOGIN` with Sholex's GitHub username

---

## Phase 1 — Collect Sholex's comments as a corpus

There are two kinds of review feedback worth capturing, and you want both:

- **Inline code comments** — line-level nitpicks and concrete objections.
- **Review summaries** — the approve / request-changes bodies, which often
  hold the higher-level architectural objections.

### 1a. Pull all inline comments, filtered to Sholex

```bash
gh api --paginate "repos/OWNER/REPO/pulls/comments" \
  --jq '.[] | select(.user.login=="SHOLEX_LOGIN") | {pr: .pull_request_url, path, line, body}' \
  > sholex-inline-comments.json
```

This grabs every inline review comment Sholex has made across all PRs in the
repo, with the file path and line for context.

### 1b. Pull review summaries

Review summaries are per-PR, so loop over recent PRs. This pulls the last 50
PRs (any state) and collects Sholex's review bodies:

```bash
for pr in $(gh pr list --repo OWNER/REPO --state all --limit 50 --json number --jq '.[].number'); do
  gh api "repos/OWNER/REPO/pulls/$pr/reviews" \
    --jq '.[] | select(.user.login=="SHOLEX_LOGIN") | {pr: '"$pr"', state, body}'
done > sholex-review-summaries.json
```

> **Tip:** `state` will be values like `APPROVED`, `CHANGES_REQUESTED`, or
> `COMMENTED`. The `CHANGES_REQUESTED` ones are gold — they're where Sholex
> drew a hard line.

### 1c. (Optional) Grab the surrounding diff context

A raw comment like "this should use the central handler" is more useful with
the code it was attached to. The inline-comments endpoint includes a
`diff_hunk` field — add it to your jq selection if you want richer context:

```bash
gh api --paginate "repos/OWNER/REPO/pulls/comments" \
  --jq '.[] | select(.user.login=="SHOLEX_LOGIN") | {path, line, diff_hunk, body}' \
  > sholex-inline-with-context.json
```

### How much is enough?

You don't need a huge sample. **30–50 comments usually exposes the patterns
clearly.** If Sholex reviews a narrow area (e.g. only backend PRs), bias your
PR list toward those so the corpus is representative.

---

## Phase 2 — Distill the patterns (not just a list)

Feed the corpus to Claude (in Claude Code, or by pasting the JSON) and ask it
to extract the _reasoning_, grouped into themes — not just a flat list of
comments. The reasoning is what generalizes to new code.

A weak extraction: _"Sholex rejected this string concatenation."_
A strong extraction: _"Sholex wants all user-facing errors routed through the
central handler so they're logged to Sentry."_ The second one applies to code
Sholex has never seen.

### Distillation prompt to use

Paste your corpus and this prompt:

```
Here are PR review comments and review summaries from a single reviewer
(Sholex), collected from past pull requests in our repo.

Analyze them and produce a distilled review rubric. Specifically:

1. Group the feedback into themes (e.g. error handling, naming, test
   coverage, edge cases, API/contract design, security, performance,
   logging/observability, documentation).

2. For each theme, state the RULE and the underlying REASONING — why
   Sholex cares, not just what was said.

3. Call out anything Sholex consistently flags that OTHER reviewers seem
   to let through. This is the highest-value signal.

4. For each rule, include 1–2 representative real comments as examples,
   paired with what the comment actually wanted (the generalizable intent).

5. Rank the themes by how reliably Sholex raises them
   (would-block vs. nit-level).

Output as structured markdown I can paste into a SKILL.md.
```

Review the output and correct anything that looks like an over-generalization
before moving on.

---

## Phase 3 — Encode it as a `SKILL.md`

Structure the skill as a **reviewer rubric** that runs against your diff and
predicts the comments Sholex would make. Below is a complete template — fill in
the bracketed sections with the distilled output from Phase 2.

### Where it goes

```
.claude/skills/pr-review-sholex/SKILL.md
```

(Same convention as your other Claude Code skills — one folder per skill, with
a `SKILL.md` inside.)

### Template

```markdown
---
name: pr-review-sholex
description: >
  Self-review a diff against Sholex's review standards before opening a PR.
  Use before issuing any pull request, or when asked to "review like Sholex"
  or "do a Sholex pass". Predicts the comments Sholex would leave and suggests
  fixes so they can be addressed pre-emptively.
---

# Pre-PR review — Sholex's standards

## When to use

Run before opening a PR, or on request ("review like Sholex"). The goal is to
surface the comments Sholex would leave so they can be fixed before review.

## How to run

1. Determine the diff under review. Default to:
   `git diff origin/main...HEAD`
   (or against whatever the PR's base branch is).
2. Review every changed hunk against the standards below.
3. Output predicted comments in the format described at the end.
4. Do not invent nits Sholex would not raise — precision matters more than
   volume. When unsure, mark it low-confidence rather than omitting it.

## What Sholex consistently flags (ranked by how reliably it blocks)

### Would-block issues

- **[Theme 1, e.g. Error handling]** — [the rule]. Reasoning: [why Sholex
  cares — e.g. user-facing errors must route through `handleError()` so they
  reach Sentry].
- **[Theme 2, e.g. Test coverage]** — [the rule + reasoning].
- **[Theme 3, e.g. API/contract design]** — [the rule + reasoning].

### Nit-level issues

- **[Theme 4, e.g. Naming]** — [the rule + reasoning].
- **[Theme 5]** — [the rule + reasoning].

## Things Sholex catches that others miss

> The highest-value section — these are the differentiators.

- [pattern other reviewers let through, + why it matters]
- [pattern ...]

## Examples (real comment → what it wanted)

- "[actual Sholex comment]" → [the generalizable intent]
- "[actual Sholex comment]" → [the generalizable intent]

## Output format

For each finding, produce:

| Severity          | Location             | Issue                         | Suggested fix  |
| ----------------- | -------------------- | ----------------------------- | -------------- |
| would-block / nit | `path/to/file.ts:42` | [what Sholex would object to] | [concrete fix] |

End with a one-line verdict: would Sholex approve, comment, or request changes?
```

---

## Phase 4 — Calibrate against a known PR

Before trusting the skill, test it:

1. Pick a PR that Sholex **already reviewed** (ideally one with several
   comments).
2. Check that branch out locally, or point the skill at that PR's diff.
3. Run the skill and compare its predicted comments to Sholex's **actual**
   comments on that PR.
4. Score it:
   - **Misses** — real comments the skill didn't predict → strengthen that
     rule, or add a missing theme.
   - **False positives** — nits the skill raised that Sholex would never make
     → tighten or remove that rule.
5. Repeat against a second PR. Two or three calibration rounds usually gets it
   surprisingly close.

Keep a short note at the bottom of the SKILL.md (in a comment) recording which
PRs you calibrated against, so you can re-test after edits.

---

## Phase 5 — Run it before every PR

Once calibrated, the routine is:

```bash
# stage your branch, then in Claude Code:
# "Do a Sholex pass on this diff"   (or invoke the skill directly)
```

The skill reviews `git diff origin/main...HEAD`, lists the comments Sholex
would likely raise with file:line and fixes, and gives a verdict. Address the
would-block items, decide on the nits, then open the PR with much higher odds
of a clean first review.

---

## Maintenance

- **Refresh the corpus periodically.** Sholex's standards evolve; re-run
  Phase 1 every few months and fold new patterns into the rubric.
- **Add new themes as they emerge.** If Sholex starts flagging a new class of
  issue, add it rather than rewriting from scratch.
- **Keep the "catches that others miss" section curated** — it's the part that
  delivers the most value and the easiest to let go stale.

---

## Quick reference — all commands in one place

```bash
# Prereqs
brew install gh
gh auth login
gh auth status

# Phase 1a — inline comments
gh api --paginate "repos/OWNER/REPO/pulls/comments" \
  --jq '.[] | select(.user.login=="SHOLEX_LOGIN") | {pr: .pull_request_url, path, line, body}' \
  > sholex-inline-comments.json

# Phase 1b — review summaries
for pr in $(gh pr list --repo OWNER/REPO --state all --limit 50 --json number --jq '.[].number'); do
  gh api "repos/OWNER/REPO/pulls/$pr/reviews" \
    --jq '.[] | select(.user.login=="SHOLEX_LOGIN") | {pr: '"$pr"', state, body}'
done > sholex-review-summaries.json

# Phase 1c — inline with diff context (optional)
gh api --paginate "repos/OWNER/REPO/pulls/comments" \
  --jq '.[] | select(.user.login=="SHOLEX_LOGIN") | {path, line, diff_hunk, body}' \
  > sholex-inline-with-context.json

# Phase 5 — review your diff locally
git diff origin/main...HEAD
```
