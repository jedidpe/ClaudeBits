# Branching Strategy — Ship / Show / Ask

Source: https://martinfowler.com/articles/ship-show-ask.html

Every code change falls into one of three categories. Choose consciously — don't default
to always opening a PR just because it's the safest habit.

---

## The Three Categories

### Ship — merge directly to main, no PR

Use when the change is routine and its correctness is obvious:

- Applying an established pattern already used elsewhere in the codebase
- Fixing a simple, well-understood bug
- Updating documentation, comments, or config
- Implementing feedback that was already agreed in a review
- Ticking a ToDo item with no design ambiguity
- Typo / naming fix

CI still runs. If it goes red, fix it immediately — Ship doesn't mean skip quality.

### Show — open a PR, merge after CI passes, don't wait for human approval

Use when the change is worth making visible but doesn't need a gate:

- A non-trivial refactor that others might learn from
- A novel approach to a recurring problem
- A bug fix with an interesting root cause
- Adding a new shared utility or pattern to `pmdx_common/`
- Work you'd welcome async comments on, but feedback is not blocking

Merge once the automated checks are green. Reviewers can comment after the fact.
Leave a short note in the PR description explaining *why* it's interesting.

### Ask — open a PR, wait for feedback before merging

Use when uncertainty or risk justifies a gate:

- Unsure about the right architectural approach
- Cross-cutting change that touches many files or multiple stacks
- New external dependency being introduced
- Security-sensitive code (auth, IAM, tenant isolation)
- Significant CDK infrastructure change (new resources, permission changes)
- Work that is half-finished at end of day and needs eyes before proceeding
- Anything that breaks existing API contracts or DB schemas

---

## Decision Guide

```
Is the approach already agreed / obvious?
  YES → Ship

  NO → Is feedback blocking, or is there meaningful uncertainty?
         YES → Ask
         NO  → Show
```

When in doubt between Show and Ask, Ask. When in doubt between Ship and Show, Show.

---

## Prerequisites — what makes this work

- **Mainline is always deployable.** Feature toggles (`FEATURE_X` env vars), dark
  launches, and schema-compatible migrations keep `main` safe to deploy at any time.
  Never merge a half-built feature without a toggle.

- **CI is the merge gate for Ship/Show.** pytest, npm run build, and cdk synth must
  be green before anything lands on main. These are non-negotiable.

- **Short-lived branches.** Branches older than a day or two create merge debt.
  Rebase frequently; if a branch is getting long, split it or convert it to Show/Ask
  so it can land incrementally.

- **Talk before coding, not just after.** For Ask changes, discuss the approach
  *before* writing code — not in the PR after the fact. A five-minute conversation
  prevents hours of rework.

---

## PMDX-Specific Guidance

| Change type | Category |
|---|---|
| Ticking a completed ToDo item (docs, comments) | Ship |
| New Lambda handler following existing pattern | Show |
| New `pmdx_common/` shared utility | Show |
| New CDK stack or major resource addition | Ask |
| IAM policy change | Ask |
| New DynamoDB access pattern / GSI | Ask |
| Frontend component (self-contained page) | Show |
| Changes to authorizer or tenant context logic | Ask |
| Fixing a failing test | Ship |
| Adding new test coverage | Show |
| New external npm or pip dependency | Ask |
| CI/CD pipeline change | Ask |

---

## What This Strategy Does NOT Change

- All merges to `main` trigger the full CI pipeline (Trivy → pytest → Vite build → CDK synth/deploy).
- Secrets, credentials, and `.env` files are never committed regardless of category.
- Commit message format (`type(scope): description`) applies to all three categories.
- PRs for Ask changes should include a clear description of the decision being made
  and the options considered — not just "here is my code."
