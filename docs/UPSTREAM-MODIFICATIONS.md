# Upstream Modifications Registry

> Machine-readable + human-readable record of every upstream file modified by this custom layer.
> Required by PROJECT DIRECTION.md §53.5 and the fork-safe architecture rules in `CLAUDE.md` / `AGENTS.md`.

Each entry documents: what the file is upstream, why we touched it, what we changed,
how hard it will be to rebase onto a future upstream revision, and whether the change
could eventually be extracted into a standalone extension.

## Format

For every modified upstream file record:

| Field | Description |
|---|---|
| **File** | Path relative to repo root |
| **Upstream responsibility** | What this file does in upstream DeskcommCRM |
| **Why we modified it** | The exact requirement that forced a change |
| **Our modification** | What we actually changed |
| **Dependency on custom functionality** | Whether custom code depends on this change |
| **How difficult to rebase** | Low / Medium / High — estimated churn when upstream changes this file |
| **Possible future extraction** | Can this be moved out of the upstream file into an extension/adapter? |

---

## Entries

### AGENTS.md

| Field | Value |
|---|---|
| **File** | `AGENTS.md` |
| **Upstream responsibility** | Portable agent contract for non-Claude code agents (Codex, Cursor, Copilot, etc.). Derived from `CLAUDE.md`. |
| **Why we modified it** | PROJECT DIRECTION.md introduces project-specific constraints (WasenderAPI, AI-optional V1, fork-safe architecture, English-only output) not covered by upstream. |
| **Our modification** | Added `## Restrições do PROJECT DIRECTION.md` section: 21 Proibido rules (including WAHA→WasenderAPI replacement, Coolify-from-GitHub deployment, three-way AI distinction, silent-failure prevention) and 18 Obrigatório mandates (including full audit, contract-first Wasender provider, kill switches, observability, public domain, git checkpoints, golden baseline, impact analysis, English-only product output). |
| **Dependency on custom functionality** | Yes — engineering skills (`to-issues`, `triage`, `diagnose`, `tdd`, etc.) read this section. |
| **How difficult to rebase** | Medium — upstream may add its own sections; content-based merge needed. |
| **Possible future extraction** | No — AGENTS.md is the root-level contract file and must stay at repo root. |

### CLAUDE.md

| Field | Value |
|---|---|
| **File** | `CLAUDE.md` |
| **Upstream responsibility** | Complete non-negotiable doctrine for Claude Code sessions. Read before any code work. |
| **Why we modified it** | PROJECT DIRECTION.md constraints must be available to Claude Code sessions as first-class doctrine. |
| **Our modification** | Added `## Restrições do PROJECT DIRECTION.md` section: 21 Proibido rules (including WAHA→WasenderAPI replacement, Coolify-from-GitHub deployment, three-way AI distinction, silent-failure prevention) and 18 Obrigatório mandates (including full audit, contract-first Wasender provider, kill switches, observability, public domain, git checkpoints, golden baseline, impact analysis, English-only product output). Updated English-language constraint to reflect verified i18n reality (pt-BR default + es partial, en non-functional). Updated `docs/UPSTREAM-MODIFICATIONS.md` fields list to include "Dependency on custom functionality" and reformatted "How difficult to rebase" entries. |
| **Dependency on custom functionality** | Yes — Claude Code reads this as the primary doctrine file. |
| **How difficult to rebase** | Medium — upstream may reorganize sections; anchor-based merge needed. |
| **Possible future extraction** | No — CLAUDE.md is the root-level doctrine file. |

---

## How to add entries

1. Identify the upstream file being modified.
2. Fill in all seven fields above.
3. If the file did not exist upstream (purely custom), note that in "Upstream responsibility."
4. Re-run the rebase test (PROJECT DIRECTION.md §53.15) to confirm custom functionality survives an upstream update.
