# Architecture Decision Records

One file per significant decision: `NNNN-short-title.md`.

A decision belongs here if reversing it later would be expensive — stack choices,
custody model, the LLM/execution boundary, data retention. Strategy parameters do not
belong here; they live in versioned config.

**Template**

```markdown
# ADR-NNNN — Title

- **Status:** proposed | accepted | superseded by ADR-XXXX
- **Date:** YYYY-MM-DD

## Context
What forced a decision. Constraints, deadlines, what we knew and didn't.

## Decision
What we chose, stated plainly.

## Alternatives considered
What else was on the table and the specific reason it lost.

## Consequences
What this makes easy, what it makes hard, what it commits us to, and what would
cause us to revisit it.
```

## Index

| ADR | Title | Status |
|---|---|---|
| [0001](0001-llm-out-of-execution-path.md) | LLM stays out of the execution path | accepted |
