# Figma Design System AI Readiness Audit

A Claude skill that audits your Figma design system and tells you exactly what to fix so AI agents can use it effectively — then optionally generates a local component registry that dramatically reduces token usage and eliminates lookup ambiguity.

## The problem

Figma's native MCP lets Claude query your design system in real time. But raw Figma data — component names, variant lists, style names — doesn't tell Claude *when* to use a component, *which* color token for a CTA, or *what* text style for a section header. That's design intent, and it lives in descriptions that most DS files don't have.

On top of that, live queries search across all enabled libraries — Claude has to guess which result is yours, burn multiple searches to find the right component, and still lacks decision guidance.

This skill fixes both problems.

## What it does

**Phase 1: Audit & Fix** — enumerates all assets via Figma Plugin API (not search — full inventory) and works through them in order of impact:

1. **Components** — missing descriptions, unpublished, generic property names
2. **Color tokens** — missing descriptions, wrong scopes, semantic vs. raw naming
3. **Typography** — checks if naming is semantic (fine) or structural (needs descriptions)
4. **Spacing & sizing tokens** — scope misconfigurations, ambiguous names
5. **Effect styles** — unpublished, non-obvious names

For each issue, Claude drafts a fix and asks for your approval before writing anything back to Figma. You know your DS — Claude just does the legwork.

**Phase 2: Local Registry (optional but recommended)** — generates compact JSON files scoped to your DS. Tested to significantly reduce token usage, number of MCP calls, and lookup ambiguity compared to live queries.

## Why the registry matters

| | Live Figma queries | Local registry |
|---|---|---|
| **Token cost** | High — `search_design_system` returns 250+ tokens per result, Claude needs multiple searches per component | Low — `_index.json` is 50-100 tokens per component, component file is 500-1000 tokens with full guidance |
| **Accuracy** | Lower — search spans all enabled libraries, Claude must disambiguate | High — registry is scoped to your DS only, no ambiguity |
| **Decision guidance** | None — just names and keys | Included — `whenToUse`, default variants, variant properties |
| **Call count** | Multiple searches + follow-up calls per component | One file read per component |
| **Maintenance** | None | Manual — regenerate when DS changes significantly |

A single design screen can burn 10,000–20,000 tokens on component lookups alone with live queries. With a local registry, the same screen uses a fraction of that — and Claude picks the right component on the first try.

**Recommendation:** Always generate the component registry. For tokens and text styles, live queries are usually sufficient after a good audit — descriptions are already in Figma and you just need a name and key.

## Prerequisites

- **Figma** with your design system in a dedicated library file
- **Figma MCP** connected to Claude Code
- Connected to your **DS source file** (not a consumer file) when running the audit

## Usage

Copy `figma-ds-audit/figma-ds-audit.md` to `.claude/commands/` in your project, then:

```
/figma-ds-audit
```

## Files

| File | What it is |
|---|---|
| `figma-ds-audit/figma-ds-audit.md` | The Claude skill |
| `figma-ds-audit/examples/` | Example registry JSON files showing output format |
