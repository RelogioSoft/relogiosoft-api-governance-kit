# Contributing to the Governance Kit

## Principles

- Standards are the source of truth — keep them in `docs/standards/`
- All rules MUST use RFC 2119 language (MUST / SHOULD / MAY)
- Every rule MUST include rationale ("why") and at least one example
- Opinionated choices MUST be explained — do not hide the reasoning
- Cover the "not covered" case explicitly if a topic is deliberately out of scope

## Proposing a new standard

1. Open an issue describing the gap, including real-world examples where the gap caused problems
2. Draft the new document in `docs/standards/NN-topic-name.md` using the standard template (see below)
3. Add the document to `mkdocs.yml` nav
4. Update `docs/index.md` standards table
5. Add Spectral rules for any machine-checkable requirements
6. Open a pull request — governance changes require at least two approvals

## Updating an existing standard

- Minor clarifications (typos, wording): single approval
- New MUST rules: two approvals + rationale update
- Changing a MUST to SHOULD or removing a rule: two approvals + migration guidance

## Document template

```markdown
# NN — Standard Title

> One-sentence summary of what this standard covers.

## Overview

Why this standard exists. What problems it solves.

## Rules

### Topic Name

**[MUST / SHOULD / MAY]** rule statement.

Rationale: why this rule exists.

#### Example

\`\`\`http
# ✅ Compliant
...

# ❌ Non-compliant
...
\`\`\`

## Summary table

| Rule | Requirement | Ref |
|------|-------------|-----|
| ... | MUST | section link |
```

## Running docs locally

```bash
pip install -r requirements-docs.txt
mkdocs serve
# Open http://127.0.0.1:8000
```

## Testing Spectral rules

```bash
npm install -g @stoplight/spectral-cli
spectral lint examples/openapi-example.yaml --ruleset .spectral.yml
```
