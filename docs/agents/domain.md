# Domain Docs

How the engineering skills consume this repo's domain documentation.

## Before exploring, read these

- `CONTEXT.md` at the repo root — does not exist yet; created lazily as terms get resolved
- `docs/adr/` — read ADRs that touch the area you're about to work in (exists: `0001-packaging-e-distribuicao.md`)

If any of these files don't exist, **proceed silently**. The producer skill (`/grill-with-docs`) creates them lazily.

## File structure

Single-context repo:

```
/
├── CONTEXT.md
├── docs/
│   └── adr/
│       └── 0001-packaging-e-distribuicao.md
└── src/
```

## Use the glossary's vocabulary

When your output names a domain concept (issue title, refactor proposal, hypothesis, test name), use the term as defined in `CONTEXT.md`.

## Flag ADR conflicts

If your output contradicts an existing ADR, surface it explicitly rather than silently overriding.
