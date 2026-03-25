# Figma Design System AI Readiness Audit

A Claude skill that audits your Figma design system and tells you exactly what to fix so AI agents can use it effectively.

## The problem

Figma's native MCP lets Claude query your design system in real time. But raw Figma data — component names, variant lists, style names — doesn't tell Claude *when* to use a component, *which* color token for a CTA, or *what* text style for a section header. That's design intent, and it lives in descriptions that most DS files don't have.

This skill finds the gaps and helps you fill them.

## What it does

**Phase 1: Audit & Fix** — enumerates all assets via Figma Plugin API (not search — full inventory) and works through them in order of impact:

1. **Components** — missing descriptions, unpublished, generic property names
2. **Color tokens** — missing descriptions, wrong scopes, semantic vs. raw naming
3. **Typography** — checks if naming is semantic (fine) or structural (needs descriptions)
4. **Spacing & sizing tokens** — scope misconfigurations, ambiguous names
5. **Effect styles** — unpublished, non-obvious names

For each issue, Claude drafts a fix and asks for your approval before writing anything back to Figma. You know your DS — Claude just does the legwork.

**Phase 2: Local Registry (optional)** — generates compact JSON files for cheaper, more accurate AI-driven design work. See [trade-offs](#registry-trade-offs) below.

## Prerequisites

- **Figma** with your design system in a dedicated library file
- **Figma MCP** connected to Claude Code
- Connected to your **DS source file** (not a consumer file) when running the audit

## Usage

```
/figma-ds-audit
```

or:

```
You: "Run the design system audit"
```

## Registry trade-offs

| | Live Figma queries | Local registry |
|---|---|---|
| **Token cost** | High — search returns 250+ tokens per result, multiple searches needed | Low — index is 50-100 tokens per component |
| **Accuracy** | Lower — search spans all enabled libraries | High — scoped to your DS only |
| **Decision guidance** | None — just names and keys | Included — `whenToUse`, defaults, variant properties |
| **Maintenance** | None | Manual — regenerate when DS changes |

**Recommendation:** Generate a component registry. For tokens and text styles, live queries are usually sufficient after a good audit — descriptions are already in Figma.

## Files

| File | What it is |
|---|---|
| `scan-design-system/figma-ds-audit.md` | The Claude skill — copy to `.claude/commands/` in your project |
| `scan-design-system/examples/` | Example registry JSON files showing output format |
