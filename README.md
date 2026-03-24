# Teaching Claude to Design With Your Design System

A guide for designers who want Claude to use their Figma design system when
building screens, forms, and flows.

## What you'll end up with

After following this guide, Claude will be able to:
- **Place components** from your DS library into any Figma file
- **Bind tokens** (colors, spacing, radius) from your DS to custom elements
- **Apply text styles** from your DS to any text node
- **Follow your design patterns** — knowing which component to use, when, and how

## Prerequisites

**This toolkit builds on top of [Vibma](https://github.com/ufira-ai/Vibma) — you need it set up and working before starting.**

Vibma is a Figma MCP plugin that lets AI agents read and write to Figma via the Claude Code CLI. If you haven't set it up yet, follow the [Vibma setup guide](https://github.com/ufira-ai/Vibma) first and confirm that Claude can create a frame in Figma before continuing here.

Once Vibma is working, you'll also need:
- **Figma** with your design system in a dedicated library file
- **Claude Code** (the CLI) connected to Vibma

## The big picture

```
Your DS file (Figma)
        │
        │  Step 1: Patch Vibma (one-time setup)
        │  Step 2: Scan your DS (generates registry)
        ▼
Component Registry (JSON files)
  _index.json ─── "what components exist"
  Button.json ─── "how to use Button"
  tokens.json ─── "what design tokens exist"
  text-styles.json ── "what text styles exist"
        │
        │  Step 3: Write your design skill
        ▼
Design Skill (.md file)
  Decision tree ─── "when to use which component"
  Layout rules ──── "how to structure screens"
  Token guidance ── "which token for which element"
        │
        │  Step 4: Design with Claude
        ▼
Claude builds screens using your DS
```

---

## Step 1: Patch Vibma

> **Requires Vibma already installed.** This step modifies your local Vibma clone — not a separate install.

Vibma out of the box can only work with components in the current file. Your DS
lives in a separate library file, so Claude needs cross-file import support.

**What this does:**
- Lets Claude **read** key hashes from your DS file (`components.list`, `variables.list`, `styles.list` now include a `key` field)
- Lets Claude **use** those keys to import components, tokens, and text styles into any consumer file via new `componentKey`, `variableKey`, and `textStyleKey` params
- Lets Claude **set descriptions** on components programmatically

**How to apply:**

Open `VIBMA-CROSS-FILE-PATCH.md` and ask Claude to apply it. The patch has
step-by-step find-and-replace instructions for all files. Claude can do the
work — you just review and approve.

```
You: "Apply the changes in VIBMA-CROSS-FILE-PATCH.md to this repo"
Claude: [applies changes, rebuilds]
You: [reload Figma plugin]
```

After applying, test that it works:
1. Connect Vibma to your DS source file
2. Run `components.list` — each item should have a `key` field
3. Switch to a consumer file
4. Create an instance using `componentKey` from step 2

---

## Step 2: Scan your DS

This generates the component registry — JSON files that tell Claude everything
about your DS: what components exist, their keys, variants, anatomy, tokens,
and text styles.

**How to run:**

```
You: "/scan-design-system"
```

The skill walks you through two phases:

### Phase 1: Audit

Claude connects to your DS file and checks its health:
- Are all components published? (needed for cross-file import)
- Do components have descriptions? (needed for Claude to know when to use them)
- Any naming issues?

If descriptions are missing, Claude will draft them one at a time and ask for
your approval before writing them. **Always review these** — Claude's drafts
need your domain knowledge to be accurate.

Example:
```
Claude: "Avatar has no description. Proposed: 'User identifier displayed as a
         circular thumbnail. type=image for photos, letter for initials...' Approve?"
You: "Empty is not for anonymous, it's for unselected/no user assigned"
Claude: "Updated. Approve?"
You: "Yes"
Claude: [writes description to Figma]
```

### Phase 2: Generate registry

After the audit is clean, Claude generates:
- **`_index.json`** — master index of all components
- **`[Component].json`** — one file per component with keys, variants, anatomy
- **`tokens.json`** — all design tokens with cross-file keys
- **`text-styles.json`** — all text styles with cross-file keys

Claude asks which components to scan. For a first pass, pick your main UI
components (buttons, inputs, selects, modals) and skip icons/sub-components.

**Output structure:**
```
your-design-system/
├── _index.json
├── Button.json
├── Input.json
├── Select.json
├── Modal.json
├── tokens.json
└── text-styles.json
```

### What's in a component file

Each component JSON contains:

```json
{
  "componentSetId": "100:200",      ← Figma node ID
  "variantKeys": {                  ← keys for direct variant import
    "100:201": { "key": "abc...", "name": "size=medium, state=enabled" }
  },
  "defaultVariant": { "size": "medium", "state": "enabled" },
  "description": "What this component is and when to use it",
  "variantProperties": { "size": ["small", "medium"], "state": ["enabled", "disabled"] },
  "anatomy": { "Label": "text node (TEXT)", "Icon": "icon instance (INSTANCE_SWAP)" },
  "componentProperties": { "Label#1:0": "TEXT — the label text" },
  "whenToUse": "Decision guidance for the AI",
  "notes": ["Quirks and constraints"]
}
```

The description is the most important part — it tells Claude **when** to use
this component and **how** to pick variants. See the skill file's "Description
Writing Guide" for good vs bad examples.

---

## Step 3: Write your design skill

The registry tells Claude **what** your components are. The design skill tells
Claude **how to think** when designing with them.

Create a skill file at `.claude/commands/your-ds-name.md`. This is the "brain"
that Claude loads when building screens.

### What to include

**1. Registry pointers** — where to find the component files
```markdown
- Registry: `your-design-system/` directory
- Token registry: `your-design-system/tokens.json`
- Text styles: `your-design-system/text-styles.json`
```

**2. Step 0: Discuss before building** — for open-ended requests, Claude should
propose a plan in chat before touching Figma. This is where iteration happens.
```markdown
For open-ended requests ("design a settings page"), share a text plan:
1. Content outline — what sections exist
2. Component mapping — which DS component for each element
3. Layout sketch — frame tree with nesting
4. Token intent — which token for each custom primitive
Then wait for approval before building.
```

**3. Decision tree** — maps user needs to specific components and variants
```markdown
| User needs to... | Component |
|---|---|
| Trigger a primary action | Button (filled, brand) |
| Enter free text | Input (medium, enabled) |
| Pick from a list | Select (singleSelect) |
| Show a form in an overlay | Modal (complex) |
```

**4. Layout rules** — how to structure screens
```markdown
- Always use auto-layout
- Plan the frame tree before creating
- FILL for stretchy children, HUG for content-sized
```

**5. Typography guidance** — which text styles for which context
```markdown
| Tier | Style | When to use |
|---|---|---|
| Heading | Heading/18/Semibold | Section titles |
| Body | Body/14/Regular | Content text |
| Caption | Body/12/Regular | Small metadata |
```

**6. Token binding rules** — how to apply design tokens
```markdown
- Never bind tokens to DS component instances (they carry their own)
- Decide tokens at planning time, not build time
- Batch all bindings in one call after creating nodes
```

**7. Common tokens inline** — the most-used tokens so Claude doesn't need to
read the full file every time
```markdown
| Token | Key |
|---|---|
| Page background | abc123... |
| Primary text | def456... |
| Border default | ghi789... |
```

### Example structure

Look at the `scan-design-system/examples/` folder for component file formats.
For the design skill structure, here's a minimal skeleton:

```markdown
---
name: my-design-system
description: Workflow skill for [Your DS]. Use when creating or editing designs.
allowed-tools: Read, Bash
---

# [Your Design System Name]

- **Registry:** `your-ds/` directory
- **Tokens:** `your-ds/tokens.json`

## Step 0: Discuss before building
[Plan template]

## Decision tree
[Component mapping table]

## Layout rules
[Auto-layout guidelines]

## Typography
[Text style tiers + keys]

## Token binding
[Rules + common token table]
```

---

## Step 4: (Optional) Write pattern templates

If your product has recurring screen patterns (list pages, create forms, detail
views, settings panels), you can document these as a separate skill. This answers
"how should this screen be structured?" at a higher level than the design skill.

Example patterns:
- **List view:** sidebar + filter bar + data table + pagination
- **Create form:** modal with header + single-column form + footer buttons
- **Detail view:** header with summary + tabbed content sections
- **Confirmation:** modal with icon + message + cancel/confirm buttons

This is optional — start with the design skill alone and add patterns as you
notice Claude making the same structural mistakes repeatedly.

---

## Summary

| Step | What | Time | Frequency |
|---|---|---|---|
| 1. Patch Vibma | Apply cross-file changes | ~30 min | Once |
| 2. Scan DS | Audit + generate registry | ~1-2 hours | Once per DS update |
| 3. Write design skill | Create the "brain" file | ~1-2 hours | Iterate over time |
| 4. Pattern templates | Document screen patterns | Optional | As needed |

After setup, your daily workflow is just:
```
You: "Design an Add Customer form"
Claude: [reads your skill + registry, proposes a plan]
You: "Looks good, but use a Drawer not a Modal"
Claude: [builds it in Figma using your DS components]
```

---

## Files included

| File | What it is |
|---|---|
| `VIBMA-CROSS-FILE-PATCH.md` | Step-by-step patch for Vibma (Part A: key exposure, Part B: key params) |
| `.claude/commands/scan-design-system/` | Skill folder: audit + registry generation |
| `.claude/commands/scan-design-system/examples/` | Redacted example JSON files showing output format |
| This guide | The document you're reading now |
