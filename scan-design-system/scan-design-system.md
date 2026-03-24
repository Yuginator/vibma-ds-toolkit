---
name: scan-design-system
description: Audit and generate a component registry from any Figma design system file. Produces a fiber-ds-style directory with component JSONs, tokens, and text styles — ready for Vibma cross-file workflows. Run this while connected to the DS source file.
allowed-tools: Read, Write, Bash, mcp__Vibma__connection, mcp__Vibma__components, mcp__Vibma__styles, mcp__Vibma__variables, mcp__Vibma__variable_collections, mcp__Vibma__instances, mcp__Vibma__frames, mcp__Vibma__selection
---

# Design System Scanner

Generate a component registry from a Figma design system file. The registry
enables Vibma to place, swap, and style components from this DS in any consumer file.

## Prerequisites

The `VIBMA-CROSS-FILE-PATCH.md` changes must be applied to Vibma. Specifically:

1. **Part A (key exposure)** — so `components.list`, `variables.list`, and
   `styles.list` return the `key` field
2. **Bonus: description support** (Steps 13–14) — so you can set component
   descriptions programmatically via `frames.update`

Without Part A, the scan cannot extract key hashes. Without the description
support, all descriptions must be pasted manually in Figma.

### Build order

After applying the patch, build in this order:
```sh
npx tsx schema/compiler/index.ts          # 1. recompile schema
npm run build                              # 2. rebuild core (MCP server)
cd packages/adapter-figma && npx tsup      # 3. rebuild plugin (depends on core)
```

Order matters — the plugin bundles code from core, so core must be built first.

### Figma plugin caching

Figma aggressively caches plugin code. After rebuilding, a simple "rerun" may
not pick up changes. If a feature you just added doesn't work:
1. Close the plugin in Figma
2. Go to Plugins → Development → remove the plugin from development
3. Re-import from `plugin/manifest.json`
4. Run the plugin

This forces Figma to read the new bundle from disk.

---

## Phase 0 — Connect and verify

1. Verify Vibma is connected: `connection(get)`
2. Confirm you're on the **DS source file** (not a consumer file)
3. If not connected, guide the user through `connection(create)`

---

## Phase 1 — Audit (health check)

Before generating the registry, assess the DS file's readiness. Do NOT skip
this phase — scanning a messy DS produces a bad registry.

### Step 1.1 — Inventory

Gather counts:
```
components.list(limit: 1)          → totalCount
variable_collections.list          → collections + variable counts
styles.list(type: "text", limit:1) → totalCount
styles.list(type: "paint", limit:1)→ totalCount
```

Present a summary:
```
DS File: [documentName]
Components: [N] component sets
Variables: [N] across [M] collections
Text styles: [N]
Paint styles: [N]
```

### Step 1.2 — Component health scan

List all components (paginate at 100):
```
components.list(limit: 100)
components.list(limit: 100, offset: 100)
...
```

**File-level checks** (from the list data):

| Severity | Check | Issue |
|---|---|---|
| Critical | Missing `key` field | Component is **unpublished** — cannot be imported cross-file |
| Warning | Duplicate names | Ambiguous cross-file lookup by name |
| Warning | Generic name (Frame, Group, Rectangle) | Layer naming needs cleanup |
| Warning | Special characters in name (*, etc.) | May cause matching issues |
| Suggestion | Inconsistent naming conventions (mixed case, prefixes) | Hard for AI to predict names |

**Per-component checks** — for main UI components (skip icons and sub-components),
fetch properties and description:
```
components.get(id: "[componentSetId]", fields: ["propertyDefinitions", "description"])
```

Do NOT use `depth` on this call — large components (500+ variants) will exceed
response limits. The `fields` filter keeps the response small.

| Severity | Check | Issue |
|---|---|---|
| Warning | No description | AI won't know when or how to use this component |
| Warning | Description is just the component name (e.g., "accordion") | Useless — needs rewriting |
| Warning | Description has incomplete notes (e.g., "medium -\nlarge -") | Unfinished documentation |
| Suggestion | Generic property names ("Property 1", "Slot", "Slot 2") | Confusing for AI |

### Step 1.3 — Fix descriptions (interactive)

For components with missing or poor descriptions, follow this loop:

1. **Read** the component's properties to understand what it does
2. **Draft** a description following the Description Writing Guide (below)
3. **Present** the draft to the user and ask for approval/corrections
4. **Apply corrections** from the user — they know their DS better than you
5. **Write** the approved description:
   ```
   frames(update, { id: "[componentSetId]", description: "..." })
   ```

**Never write a description without user approval.** Every description we tested
needed at least a small correction — the user knows nuances about component
behavior, intended use cases, and naming conventions that can't be inferred
from properties alone.

### Step 1.4 — Token health scan

```
variable_collections.list
```

For each collection:
```
variables.list(collectionId: "[name]", limit: 100)
```

Flag:
| Severity | Check | Issue |
|---|---|---|
| Critical | Missing `key` field on variables | Variables are **unpublished** |
| Warning | No scopes set (scopes = ALL_SCOPES) | Variable may bind to unintended properties |
| Suggestion | No description | AI won't understand the token's purpose |

### Step 1.5 — Text style health scan

```
styles.list(type: "text")
```

Flag:
| Severity | Check | Issue |
|---|---|---|
| Critical | Missing `key` field | Style is **unpublished** |
| Suggestion | Inconsistent naming (mixed separators, no hierarchy) | Hard for AI to pick the right style |

### Step 1.6 — Present audit results

Group findings by severity. Show counts:
```
Audit Results
─────────────
Critical: [N] (must fix before scanning)
Warning:  [N] (recommended to fix)
Suggestion: [N] (nice to have)

Critical issues:
- [N] unpublished components (no key hash)
- [N] unpublished variables
...
```

**If critical issues exist:** Tell the user what to fix in Figma before proceeding.
Unpublished components/variables/styles MUST be published for cross-file import.

**If no critical issues:** Proceed to Phase 2.

Ask: "Ready to generate the registry? I found [N] components. Want to scan all
of them, or pick specific ones?"

**Tip:** Most DS files have many icons and sub-components. Suggest grouping icons
into a single `Icons.json` and scanning only the main UI components individually.

---

## Phase 2 — Scan (registry generation)

### Step 2.0 — Create output directory

Ask the user for a directory name (default: `design-system/`):
```sh
mkdir -p [directory-name]
```

### Step 2.1 — Generate tokens.json

For each variable collection:
```
variables.list(collectionId: "[name]", limit: 500)
```

Write `tokens.json`:
```json
{
  "_meta": {
    "description": "Token registry. key = 40-char hex for cross-file import via variableKey.",
    "source": "[documentName] via Vibma scan",
    "totalTokens": N,
    "updatedAt": "YYYY-MM-DD"
  },
  "tokens": {
    "color/background/neutral-default/enabled": {
      "variableId": "VariableID:808:963",
      "key": "765e814c12fcc9bc295978c9daca5bb2b951450f"
    }
  }
}
```

Only include variables that have a `key` field (published).

Read `examples/tokens.example.json` for the exact format.

### Step 2.2 — Generate text-styles.json

```
styles.list(type: "text", limit: 500, fields: ["*"])
```

Write `text-styles.json`:
```json
{
  "_meta": {
    "description": "Text style registry. key = 40-char hex for cross-file import via textStyleKey.",
    "source": "[documentName] via Vibma scan",
    "totalStyles": N,
    "font": "[primary font family]",
    "updatedAt": "YYYY-MM-DD"
  },
  "styles": {
    "Heading/24/Semibold": {
      "key": "abc123...",
      "fontSize": 24,
      "fontWeight": "Semibold",
      "lineHeight": 32
    }
  }
}
```

Read `examples/text-styles.example.json` for the exact format.

### Step 2.3 — Generate component files

For each component the user confirmed:

#### 2.3a — Get component detail

**For components with many variants (100+):** Use fields filtering, no depth:
```
components.get(id: "[componentSetId]", fields: ["propertyDefinitions", "description"])
```
Then get variant keys separately by fetching child stubs.

**For smaller components:** Depth=0 or depth=1 is safe:
```
components.get(id: "[componentSetId]", depth: 0)
```

**Never use depth=1 on large component sets** (500+ variants) — it will exceed
response limits and timeout. Button (850 variants) is a common example.

#### 2.3b — Build the component JSON

```json
{
  "componentSetId": "617:3946",
  "variantKeys": {
    "[variantNodeId]": {
      "key": "[40-char hex]",
      "name": "color=brand, type=filled, size=medium, ..."
    }
  },
  "textNodeIds": {
    "formula": "I{instanceId};{nodeId}",
    "note": "Use component properties (instances.update) when available. Fall back to direct text node IDs using the formula.",
    "byVariant": {}
  },
  "defaultVariant": {},
  "description": "[from Figma component description — already set in Phase 1]",
  "variantProperties": {
    "color": ["brand", "neutral", ...],
    "size": ["small", "medium", ...]
  },
  "anatomy": {
    "Label": "primary text (TEXT)",
    "Icon": "icon instance (INSTANCE_SWAP)"
  },
  "componentProperties": {
    "Label#1:0": "TEXT — the label text"
  },
  "whenToUse": "[from description or auto-drafted]",
  "notes": []
}
```

Read `examples/Button.example.json` for the exact format with all fields.

**Extracting variantKeys:** From `components.get`, each child of a COMPONENT_SET
is a variant. Map `child.id` → `{ key: child.key, name: child.name }`.

**Extracting textNodeIds:** For the default variant, use `components.get` with
depth=2 on that specific variant (not the whole set) to find TEXT children.

**Extracting anatomy:** From the default variant's children, map each child name
to its type and role (TEXT, INSTANCE, FRAME, etc.).

**Extracting componentProperties:** From `propertyDefinitions` on the component set.
Already fetched in Phase 1.

### Step 2.4 — Generate _index.json

After all component files are written, generate the master index:

```json
{
  "designSystem": "[DS name from documentName]",
  "figmaFileKey": "[ask user or extract from Figma URL]",
  "lastUpdated": "YYYY-MM-DD",
  "note": "Read _index.json to find the right component, then read its file for full details.",
  "components": {
    "Button": {
      "componentSetId": "617:3946",
      "key": "[component set key hash]",
      "file": "Button.json",
      "description": "[same as in the component file]",
      "variantCount": 850,
      "variantProperties": ["color", "type", "size", "layout", "state"]
    }
  }
}
```

Read `examples/_index.example.json` for the exact format.

### Step 2.5 — User review

Present a summary:
```
Registry generated in [directory]/
- _index.json          — [N] components indexed
- tokens.json          — [N] tokens with keys
- text-styles.json     — [N] text styles with keys
- [Component].json     — one file per component

Next steps:
1. Review descriptions in each component file
2. Add notes for any quirks or constraints you know about
3. Test cross-file import from a consumer file
```

---

## Description writing guide

When drafting or reviewing component descriptions, follow these principles.

**Important:** Always get user approval before writing a description. Every
description needs at least small corrections — the user knows their DS
better than what can be inferred from properties alone.

### The description should answer:
1. **What is this?** — One-sentence identity (e.g., "A compact status label")
2. **When do I pick which variant?** — Decision guidance for key properties
3. **What's the default?** — So the AI knows what comes out of the box
4. **How do I set text?** — Component property name or direct text node

### Tone:
- Write for an AI agent that needs to make design decisions
- Be prescriptive: "Use X for Y" not "X can be used for Y"
- Focus on WHEN to use each variant, not WHAT variants exist
- Include the user's domain language (they may correct generic terms)

### Examples of good descriptions:

**Button:**
> Primary action trigger. Use filled+brand for main CTAs, outline+neutral for
> secondary actions. Default size is small. Set label via the buttonLabel
> component property.

**Badge:**
> Compact status or category label. Use color=critical for errors, success for
> positive states, neutral for default. style=light on colored backgrounds.
> borderRadius=round for pill shape.

**SingleValueInput:**
> Free-text input for name, email, password, number fields. Default state is
> enabled; switches to filled when user enters data. Set label and placeholder
> via component properties. size=small is default.

**Multi-Value Input:**
> Tag input that converts typed text into removable badges on Enter. Use for
> fields that accept multiple free-form values (emails, tags, recipients).
> Users type a value, press Enter to create a tag, and repeat.

### Examples of bad descriptions:

> "A button component" — Too vague, no decision guidance.

> "Has 5 colors, 3 types, 5 sizes" — Just restates variantProperties.

> "Use this component to create buttons in the UI" — Circular, tells the AI nothing.

> "accordion" — Just the component name. Useless.

> "small - Only used for table cell\nmedium - Used in forms" — Size notes without
> explaining what the component does.

---

## Reference: Output file formats

Read the example files in the `examples/` folder alongside this skill:

```
.claude/commands/scan-design-system/
├── scan-design-system.md        ← this file
└── examples/
    ├── _index.example.json      ← master index format
    ├── Button.example.json      ← component file format (with all fields)
    ├── tokens.example.json      ← token registry format
    └── text-styles.example.json ← text style registry format
```

Before generating any file, read the corresponding example to match the exact structure.
All example data is redacted/placeholder — replace with real values from the scan.
