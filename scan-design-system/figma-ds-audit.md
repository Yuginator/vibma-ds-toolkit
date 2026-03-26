---
name: figma-ds-audit
description: Audit your Figma design system for AI readiness. Enumerates components, color tokens, typography, spacing tokens, and other styles via Figma Plugin API — fixes descriptions interactively and optionally generates a local registry for cheaper, more accurate AI-driven design.
allowed-tools: Read, Write, Bash
---

# Figma Design System AI Readiness Audit

Systematically checks your design system and fixes what prevents AI agents from using it effectively. Works through assets in order of impact on design output — you can stop after any phase.

## How this works

All data is read and written via the **Figma Plugin API**, invoked through the `use_figma` skill. Claude writes JavaScript code and passes it as a string to `use_figma` — Figma executes it in the plugin context and returns whatever the `return` statement produces. Write operations (setting descriptions) don't need a `return`.

This gives full, direct access to the file's contents — no guessing search terms, no cross-library noise. Everything is scoped to the DS file you're currently connected to.

> **Before starting:** Make sure you're connected to your DS source file (not a consumer file). Invoke `use_figma` for all Plugin API calls below.

---

## Severity levels

| Level | Meaning |
|---|---|
| **Critical** | Blocks AI use — must fix before Claude can use this asset |
| **Warning** | Degrades quality — Claude will make worse decisions without it |
| **Suggestion** | Nice to have — improves consistency and predictability |

---

## Phase 1: Audit & Fix

Work through each section in order. Each section follows the same pattern:
1. **Enumerate** — pass the JS snippet to `use_figma`; it runs in the Figma plugin context and returns the data
2. **Flag** — identify issues by severity
3. **Fix loop** — Claude drafts a fix, you approve or correct, Claude passes a write snippet to `use_figma`

You can stop after any section if you're satisfied with coverage.

---

### Section 1: Components

**Highest impact.** If Claude picks the wrong component or can't find one, the whole design is wrong.

#### Enumerate

```javascript
// Get all component sets across all pages
const sets = figma.root.findAllWithCriteria({ types: ['COMPONENT_SET'] })
return sets.map(c => ({
  id: c.id,
  name: c.name,
  key: c.key,
  description: c.description,
  variantCount: c.children?.length ?? 0
}))
```

Also get standalone components (not part of a set):
```javascript
const components = figma.root.findAllWithCriteria({ types: ['COMPONENT'] })
  .filter(c => c.parent?.type !== 'COMPONENT_SET')
return components.map(c => ({ id: c.id, name: c.name, key: c.key, description: c.description }))
```

#### Flag

| Severity | Check | Issue |
|---|---|---|
| Critical | `key` is empty | Component is **unpublished** — cannot be imported cross-file |
| Critical | No description | Claude has no idea when or how to use this component |
| Warning | Description is just the component name (e.g., "accordion") | Useless — needs rewriting |
| Warning | Description has incomplete notes (e.g., "medium -\nlarge -") | Unfinished |
| Warning | Generic property names ("Property 1", "Slot 2") | Confusing when setting values |
| Warning | Duplicate names | Ambiguous lookup |
| Suggestion | Inconsistent naming conventions (mixed case, no prefixes) | Hard for AI to predict names |

Skip icons and sub-components (components whose name starts with a `.` or `_`, or whose name contains `/icon/`, `/internal/`). Focus on main UI components.

#### Fix loop: descriptions

For each component with a missing or poor description:

1. Read its properties to understand what it does:
```javascript
const node = figma.getNodeById('[id]')
return {
  description: node.description,
  variantProperties: node.variantGroupProperties,
  children: node.children?.slice(0,3).map(c => ({ name: c.name, type: c.type }))
}
```

2. Draft a description following these principles:

**A good description answers:**
- What is this component? (one sentence)
- Which variant for which situation? (decision guidance for key properties)
- What is the default? (so Claude knows what it gets without specifying)
- How do I set the label/text? (component property name)

**Tone:** Write for an AI making design decisions. Be prescriptive ("Use X for Y"), not descriptive ("X can be used for Y"). Focus on WHEN to use each variant, not WHAT variants exist.

**Good examples:**
> **Button:** Primary action trigger. Use filled+brand for main CTAs, outline+neutral for secondary actions. Default size is small. Set label via the `buttonLabel` component property.

> **Badge:** Compact status label. Use color=critical for errors, success for positive states, neutral for default. style=light on colored backgrounds. borderRadius=round for pill shape.

> **SingleValueInput:** Free-text input for name, email, password, number fields. Default state is enabled; switches to filled when user enters data. Set label and placeholder via component properties.

**Bad examples:**
> "A button component" — too vague, no decision guidance
> "Has 5 colors, 3 types, 5 sizes" — just restates variant properties
> "accordion" — just the component name, useless

3. Present the draft to the user. **Always get approval before writing.** Every description needs at least a small correction — the user knows their DS better than what can be inferred from properties alone.

4. Write the approved description:
```javascript
const node = figma.getNodeById('[id]')
node.description = '[approved description]'
```

---

### Section 2: Color tokens

**Second highest impact.** Colors appear everywhere — backgrounds, text, borders, interactive states. Wrong color token = off-brand design.

#### Enumerate

```javascript
const variables = figma.getLocalVariables('COLOR')
return variables.map(v => ({
  id: v.id,
  name: v.name,
  key: v.key,
  description: v.description,
  scopes: v.scopes,
  valuesByMode: v.valuesByMode
}))
```

#### Flag

| Severity | Check | Issue |
|---|---|---|
| Critical | `key` is empty | Variable is **unpublished** |
| Warning | No description | Claude can't infer usage intent from the name alone |
| Warning | `scopes` is `['ALL_SCOPES']` | Variable may bind to unintended properties |
| Suggestion | Name doesn't encode semantic role (e.g., `blue-500` vs `color/background/primary`) | Hard for Claude to pick the right token |

#### Fix loop: descriptions

Token descriptions are different from component descriptions — the user's brand intent drives them, not just the token's value. Before drafting:

- Ask: *"Do you have a brand guide or color usage doc I can reference?"*
- If yes, use it to connect token → brand intent → usage rule
- If no, draft from the token name and ask the user to correct

**A good token description answers:**
- What UI element or state is this token for?
- When should you NOT use it? (boundaries are as important as intended use)

**Good examples:**
> `color/background/primary` → "Primary brand surface. Use for CTAs and key interactive elements. Do not use for page backgrounds or cards."

> `color/text/subtle` → "Secondary text. Use for supporting labels, captions, metadata. Not for body copy or headings."

**Bad examples:**
> "Blue color" — just describes the value
> "Primary" — no usage context

Write approved descriptions:
```javascript
const variable = figma.getVariableById('[id]')
variable.description = '[approved description]'
```

---

### Section 3: Typography / text styles

**Third highest impact.** Affects readability and visual hierarchy.

#### Enumerate

```javascript
const styles = figma.getLocalTextStyles()
return styles.map(s => ({
  id: s.id,
  name: s.name,
  key: s.key,
  description: s.description,
  fontSize: s.fontSize,
  fontWeight: s.fontWeight,
  lineHeight: s.lineHeight
}))
```

#### Flag

| Severity | Check | Issue |
|---|---|---|
| Critical | `key` is empty | Style is **unpublished** |
| Warning | Structural naming + no description | Claude can't infer usage context from "Heading 2" |
| Suggestion | Inconsistent naming separators (mixed `/` and `-`) | Harder to predict style names |

#### Naming convention check

First, determine whether the naming convention is **semantic** or **structural**:

- **Semantic** (encodes context) → descriptions optional
  - Examples: `Caption`, `Body/Regular`, `Subheading`, `Display/Large`, `Label/Small`
- **Structural** (encodes scale only) → descriptions needed
  - Examples: `Heading 1`, `Heading 2`, `H3`, `Text 1`, `24px Bold`

If naming is structural, run the fix loop for each style.

#### Fix loop: descriptions (structural naming only)

**A good text style description answers:**
- What content tier is this for? (page title, section header, body, caption)
- When does a designer reach for this over a similar style?
- Any constraints? (e.g., "one per screen", "never inside cards")

**Good examples:**
> `Heading 1` → "Page-level titles only. One per screen. Never inside cards or modals."
> `Heading 2` → "Section headers within a page. Use when content has multiple distinct sections."
> `Body/Regular` → "Default body copy. Use for content paragraphs, descriptions, form labels."

Write approved descriptions:
```javascript
const style = figma.getLocalTextStyleById('[id]')
style.description = '[approved description]'
```

---

### Section 4: Spacing & sizing tokens

**Fourth priority.** Affects layout consistency — padding, gap, border radius.

#### Enumerate

```javascript
const variables = figma.getLocalVariables('FLOAT')
return variables.map(v => ({
  id: v.id,
  name: v.name,
  key: v.key,
  description: v.description,
  scopes: v.scopes
}))
```

#### Flag

| Severity | Check | Issue |
|---|---|---|
| Critical | `key` is empty | Variable is unpublished |
| Warning | `scopes` is `['ALL_SCOPES']` | May bind to unintended properties (e.g., a gap token binding to opacity) |
| Suggestion | No description on ambiguous names | `spacing/4` is clear, `size/md` is not |

Descriptions are usually only needed when the name alone is ambiguous. Most well-named spacing tokens (`spacing/4`, `radius/sm`) don't need descriptions.

---

### Section 5: Other styles

**Lowest priority.** Effect styles (shadows, blurs) and grid styles. Rarely the cause of a wrong design decision — safe to skip on a first pass.

#### Enumerate

```javascript
const effectStyles = figma.getLocalEffectStyles()
return effectStyles.map(s => ({ id: s.id, name: s.name, key: s.key, description: s.description }))
```

#### Flag

| Severity | Check | Issue |
|---|---|---|
| Critical | `key` is empty | Style is unpublished |
| Suggestion | No description on non-obvious names | "Elevation/4" needs context; "Card shadow" doesn't |

---

### Audit summary

After completing all sections, present a summary:

```
AI Readiness Audit — [DS name]
──────────────────────────────
Components:    [N] audited, [N] fixed, [N] remaining issues
Color tokens:  [N] audited, [N] fixed, [N] remaining issues
Text styles:   [N] audited, [N] fixed, [N] remaining issues
Spacing tokens:[N] audited, [N] fixed, [N] remaining issues
Other styles:  [N] audited, [N] fixed, [N] remaining issues

Critical issues remaining: [N] (must fix before AI can use these assets)
Warnings remaining:        [N]
```

---

## Phase 2: Local Registry (optional)

With a well-audited DS, Claude can query Figma in real time. But a local registry is worth generating if you do frequent AI-driven design work.

### Trade-offs

| | Live Figma queries | Local registry |
|---|---|---|
| **Token cost** | High — `search_design_system` returns 250+ tokens per result, may need multiple searches | Low — index is 50-100 tokens per component, component file is 500-1000 tokens |
| **Accuracy** | Lower — search returns results across all enabled libraries, Claude must guess which is correct | High — registry is scoped to your DS only |
| **Decision guidance** | None — just names and keys | Included — `whenToUse`, default variants, variant properties |
| **Maintenance** | None | Manual — must regenerate when DS changes |

**Recommendation:** Generate a component registry. For tokens and text styles, live queries are usually fine — you just need a name and key, and descriptions are in Figma after the audit.

### Component registry

After the audit, generate `_index.json` and one file per component.

**`_index.json`** — master index, read this first to find the right component:
```json
{
  "designSystem": "[DS name]",
  "lastUpdated": "YYYY-MM-DD",
  "note": "Read _index.json first, then open the component file for full details.",
  "components": {
    "Button": {
      "componentSetId": "617:3946",
      "file": "Button.json",
      "description": "[one-line summary]",
      "variantCount": 850,
      "variantProperties": ["color", "type", "size", "state"]
    }
  }
}
```

**`[Component].json`** — full detail per component:
```json
{
  "componentSetId": "617:3946",
  "description": "[from Figma — already set in Phase 1]",
  "defaultVariant": { "color": "brand", "type": "filled", "size": "medium" },
  "variantProperties": {
    "color": ["brand", "neutral", "critical"],
    "type": ["filled", "outline", "ghost"],
    "size": ["small", "medium", "large"]
  },
  "variantKeys": {
    "[variantNodeId]": { "key": "[40-char hex]", "name": "color=brand, type=filled, size=medium" }
  },
  "componentProperties": {
    "buttonLabel#1:0": "TEXT — the button label"
  },
  "anatomy": {
    "Label": "primary text (TEXT)",
    "Icon": "icon slot (INSTANCE_SWAP)"
  }
}
```

Read `examples/Button.example.json` for the exact format.

### Generating variant keys

Pass to `use_figma`:
```javascript
const set = figma.getNodeById('[componentSetId]')
return set.children.map(v => ({
  id: v.id,
  key: v.key,
  name: v.name
}))
```

### Optional: token and text style registries

Only generate these if you want consistency or work in environments where Figma MCP is unavailable. See `examples/tokens.example.json` and `examples/text-styles.example.json` for format.
