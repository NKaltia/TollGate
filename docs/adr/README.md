# Architecture Decision Records

An ADR is a short document that records one important decision: the context, the options considered, what was chosen, and **why**. ADRs are the project's answer to "why is it built this way?". Read them before the defense.

## Rules

- One decision per file: `NNNN-short-title.md` (e.g. `0001-database.md`).
- Write it when the decision is made, not at the end.
- Never rewrite history. If a decision changes, write a new ADR that **supersedes** the old one and update the old one's status.
- Keep it short: about one page.

## Index

| # | Decision | Status |
|---|----------|--------|
| 0001 | Database: PostgreSQL + pgvector | Planned (M0) |

## Template

```markdown
# NNNN. Title

- Status: Proposed | Accepted | Superseded by NNNN
- Date: YYYY-MM-DD

## Context
What problem or question forces a decision? What constraints matter?

## Options considered
1. Option A: pros / cons
2. Option B: pros / cons

## Decision
What we chose.

## Consequences
What becomes easier, what becomes harder, what we must watch out for.

## How we will know it was right (optional)
A measurement or signal that would make us revisit this.
```
