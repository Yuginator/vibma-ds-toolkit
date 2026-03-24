# Vibma Cross-File Library Patch

Complete patch to add cross-file design system library support to Vibma. Enables
reading key hashes from a DS source file and using them to import components,
variables, and text styles into consumer files.

## What this patch does

**Part A — Read keys:** Exposes the published 40-char hex key hash on
`components.list`, `variables.list`, and `styles.list` output. Connect Vibma
to your DS source file to discover all keys.

**Part B — Use keys:** Adds `componentKey`, `variableKey`, and `textStyleKey`
params that route through Figma's cross-file import APIs. Use these from
consumer files to import assets from the DS library.

```
DS source file (Figma)          Consumer file (Figma)
─────────────────────           ─────────────────────
components.list                 instances(create, { componentKey: "abc..." })
  → { name, id, key }          instances(swap,   { componentKey: "abc..." })

variables.list                  frames(update, { bindings: [
  → { name, key }                { field: "fills/0/color", variableKey: "def..." }
                                ]})

styles.list                     text(create, { textStyleKey: "ghi..." })
  → { name, id, key }          text(update, { textStyleKey: "ghi..." })
```

The existing `*Id` and `*Name` params continue to work for local assets.

---

## Order of operations

1. Apply Part A: key hash exposure (steps 1–3)
2. Apply Part B: schema changes (steps 4–7)
3. Run schema compiler: `npx tsx schema/compiler/index.ts`
4. Apply Part B: handler changes (steps 8–12)
5. Rebuild plugin: `cd packages/adapter-figma && npx tsup`
6. Rebuild MCP server: `npm run build`
7. Reload Figma plugin

---

# Part A — Expose Key Hashes

These changes make `components.list`, `variables.list`, and `styles.list`
include the published `key` field in their output.

Each file needs two fixes:
1. **Serialize** the key in the serializer function
2. **Preserve** it past `pickFields` (which strips non-identity fields on list stubs)

---

## Step 1: `packages/adapter-figma/src/handlers/components.ts`

### 1a. Expose key in `serializeComponentSummary`

Find:
```ts
function serializeComponentSummary(node: any): any {
  const out: any = { id: node.id, name: node.name };
  if (node.description) out.description = node.description;
```
Replace with:
```ts
function serializeComponentSummary(node: any): any {
  const out: any = { id: node.id, name: node.name };
  // Expose published key hash (40-char hex) for cross-file importComponentByKeyAsync
  if ((node as any).key) out.key = (node as any).key;
  if (node.description) out.description = node.description;
```

### 1b. Return objects (not bare strings) from `components.list`

Find (in `listComponentsFigma`):
```ts
  const items = paged.items.map((c: any) => c.name);
```
Replace with:
```ts
  const items = paged.items.map((c: any) => {
    const stub: any = { name: c.name, id: c.id };
    if (c.key) stub.key = c.key;
    return stub;
  });
```

---

## Step 2: `packages/adapter-figma/src/handlers/styles.ts`

### 2a. Expose key in `serializeStyle`

Find:
```ts
  const r: any = { id: style.id, name: style.name, type: style.type };
  if (style.description) r.description = style.description;
```
Replace with:
```ts
  const r: any = { id: style.id, name: style.name, type: style.type };
  // Expose published key hash for cross-file importStyleByKeyAsync
  // Plugin API may not expose .key directly — extract from style ID format "S:<hex>,"
  try {
    if ((style as any).key) { r.key = (style as any).key; }
    else if (style.id) {
      const m = style.id.match(/^S:([0-9a-f]{40}),/i);
      if (m) r.key = m[1];
    }
  } catch (_) {}
  if (style.description) r.description = style.description;
```

### 2b. Preserve key past `pickFields` in `listStylesFigma`

Find (in `listStylesFigma`):
```ts
  const items = paged.items.map(s => {
    const full = serializeStyle(s);
    // Stubs by default; fields to request more; ["*"] for everything
    if (!fields?.length) return pickFields(full, []);
    return pickFields(full, fields);
```
Replace with:
```ts
  const items = paged.items.map(s => {
    const full = serializeStyle(s);
    // Stubs by default; fields to request more; ["*"] for everything
    // Always include key if available (needed for cross-file imports)
    const result = !fields?.length ? pickFields(full, []) : pickFields(full, fields);
    if (full.key) result.key = full.key;
    return result;
```

---

## Step 3: `packages/adapter-figma/src/handlers/variables.ts`

### 3a. Expose key in `serializeVariable`

Find:
```ts
  const result: Record<string, any> = {
    name: v.name, type: v.resolvedType,
    valuesByMode, scopes: v.scopes,
  };
  if (v.description) result.description = v.description;
```
Replace with:
```ts
  const result: Record<string, any> = {
    name: v.name, type: v.resolvedType,
    valuesByMode, scopes: v.scopes,
  };
  // Expose published key hash for cross-file importVariableByKeyAsync
  try { if (v.key) result.key = v.key; } catch (_) {}
  if (v.description) result.description = v.description;
```

### 3b. Preserve key past `pickFields` in `listVariablesFigma`

Find (in `listVariablesFigma`):
```ts
  for (const v of paged.items) {
    const full = await serializeVariable(v);
    items.push(!fields?.length ? pickFields(full, ["valuesByMode", "scopes", "description"]) : pickFields(full, fields));
  }
```
Replace with:
```ts
  for (const v of paged.items) {
    const full = await serializeVariable(v);
    const result = !fields?.length ? pickFields(full, ["valuesByMode", "scopes", "description"]) : pickFields(full, fields);
    // Always include key if available (needed for cross-file imports)
    if (full.key) result.key = full.key;
    items.push(result);
  }
```

---

# Part B — Cross-File `*Key` Params

These changes add `componentKey`, `variableKey`, and `textStyleKey` parameters
to the schema and handlers.

---

## Step 4: `schema/tools/instances.yaml`

### 4a. Update `create` method — add `componentKey`

Find:
```yaml
    example: 'instances(method:"create", items:[{componentId:"1:23", variantProperties:{"Size":"Large"}, properties:{"Label":"Click me"}, parentId:"2:45", layoutSizingHorizontal:"FILL"}])'
```
Replace with:
```yaml
    example: 'instances(method:"create", items:[{componentKey:"abcdef1234567890abcdef1234567890abcdef12", variantProperties:{"Size":"Large"}, properties:{"Label":"Click me"}, parentId:"2:45", layoutSizingHorizontal:"FILL"}])'
```

Find:
```yaml
        description: "Array of {componentId, variantProperties?, x?, y?, parentId?, layoutSizingHorizontal?, ...}"
```
Replace with:
```yaml
        description: "Array of {componentKey or componentId, variantProperties?, x?, y?, parentId?, layoutSizingHorizontal?, ...}"
```

Find (inside `create.items.properties`):
```yaml
            componentId: { type: string, optional: true, description: "Local component or component set node ID (e.g. '1:23')", aliases: [id] }
```
Add `componentKey` **above** it and update `componentId` description:
```yaml
            componentKey: { type: string, optional: true, description: "Published component key hash (40-char hex) for cross-file library import. Preferred over componentId for library components." }
            componentId: { type: string, optional: true, description: "Local component or component set node ID (e.g. '1:23'). Use componentKey instead for cross-file library components.", aliases: [id] }
```

### 4b. Update `update` method — add `variableKey` to bindings

Find (in `update.items.properties`):
```yaml
            bindings: { type: array, optional: true, description: "Bind variables to properties. field path examples: 'fills/0/color', 'strokes/0/color', 'opacity', 'width', 'cornerRadius', 'itemSpacing'.", items: { type: object, properties: { field: { type: string, required: true }, variableName: { type: string }, variableId: { type: string } } } }
```
Replace with:
```yaml
            bindings: { type: array, optional: true, description: "Bind variables to properties. field path examples: 'fills/0/color', 'strokes/0/color', 'opacity', 'width', 'cornerRadius', 'itemSpacing'.", items: { type: object, properties: { field: { type: string, required: true }, variableName: { type: string }, variableKey: { type: string, description: "Published variable key hash (40-char hex) for cross-file library token. Preferred over variableId." }, variableId: { type: string } } } }
```

### 4c. Update `swap` method — add `componentKey`

Find:
```yaml
    description: "Swap instance component (preserves overrides)"
```
Replace with:
```yaml
    description: "Swap instance component (preserves overrides). Use componentKey for cross-file library components, componentId for local."
```

Find:
```yaml
        description: "Array of {id, componentId}"
```
Replace with:
```yaml
        description: "Array of {id, componentKey or componentId, variantProperties?}"
```

Find (in `swap.items.properties`):
```yaml
            id: { type: string, required: true, description: "Instance node ID" }
            componentId: { type: string, optional: true, description: "Local component or component set node ID" }
```
Replace with:
```yaml
            id: { type: string, required: true, description: "Instance node ID" }
            componentKey: { type: string, optional: true, description: "Published component key hash (40-char hex) for cross-file library import. Preferred over componentId for library components." }
            componentId: { type: string, optional: true, description: "Local component or component set node ID" }
            variantProperties: { type: object, optional: true, description: 'Pick variant e.g. {"size":"24","style":"regular"}' }
```

---

## Step 5: `schema/tools/text.yaml`

Find (in `create.items.properties`, near `textStyleId`):
```yaml
            textStyleId:
              type: string
              description: "Text style ID or name (case-insensitive) — overrides fontSize/fontWeight"
```
Add `textStyleKey` **above** it and update `textStyleId` description:
```yaml
            textStyleKey:
              type: string
              optional: true
              description: "Published text style key hash (40-char hex) for cross-file library import. Preferred over textStyleId for library styles."
            textStyleId:
              type: string
              description: "Local text style ID or name (case-insensitive) — overrides fontSize/fontWeight. Use textStyleKey instead for cross-file library styles."
```

---

## Step 6: `schema/base/node.yaml`

Find (in the shared PatchItem `bindings` property):
```yaml
            bindings: { type: array, optional: true, description: "Bind variables to properties. field path examples: 'fills/0/color', 'strokes/0/color', 'opacity', 'width', 'cornerRadius', 'itemSpacing'.", items: { type: object, properties: { field: { type: string, required: true }, variableName: { type: string }, variableId: { type: string } } } }
```
Replace with:
```yaml
            bindings: { type: array, optional: true, description: "Bind variables to properties. field path examples: 'fills/0/color', 'strokes/0/color', 'opacity', 'width', 'cornerRadius', 'itemSpacing'.", items: { type: object, properties: { field: { type: string, required: true }, variableName: { type: string }, variableKey: { type: string, description: "Published variable key hash (40-char hex) for cross-file library token. Preferred over variableId." }, variableId: { type: string } } } }
```

---

## Step 7: `schema/mixins/text.yaml`

Find:
```yaml
  textStyleId: { type: string, optional: true, description: "Apply text style by ID — overrides fontSize/fontWeight" }
  textStyleName: { type: string, optional: true, description: "Text style by name (case-insensitive)" }
```
Replace with:
```yaml
  textStyleKey: { type: string, optional: true, description: "Published text style key hash (40-char hex) for cross-file library import. Preferred over textStyleId for library styles." }
  textStyleId: { type: string, optional: true, description: "Apply text style by ID — overrides fontSize/fontWeight. Use textStyleKey instead for cross-file library styles." }
  textStyleName: { type: string, optional: true, description: "Text style by name (case-insensitive)" }
```

---

## ⚙️ Run schema compiler

After steps 4–7:
```sh
npx tsx schema/compiler/index.ts
```
This regenerates `packages/core/src/tools/generated/defs.ts`, `guards.ts`, and `help.ts`.

---

## Step 8: `packages/adapter-figma/src/handlers/components.ts`

### 8a. Rewrite `instanceCreateSingle` resolver

Find:
```ts
async function instanceCreateSingle(p: any) {
  let node: any = await figma.getNodeByIdAsync(p.componentId);
  if (!node) {
    await figma.loadAllPagesAsync();
    node = await figma.getNodeByIdAsync(p.componentId);
  }
  if (!node) throw new Error(`Component not found: ${p.componentId}`);
```
Replace with:
```ts
async function instanceCreateSingle(p: any) {
  let node: any;

  if (p.componentKey) {
    // Cross-file library import via published key hash (preferred path)
    try { node = await (figma as any).importComponentSetByKeyAsync(p.componentKey); } catch (_) {}
    if (!node) {
      try { node = await figma.importComponentByKeyAsync(p.componentKey); } catch (_) {}
    }
    if (!node) throw new Error(`Component not found by key: ${p.componentKey}`);
  } else if (p.componentId) {
    // Local node ID lookup
    node = await figma.getNodeByIdAsync(p.componentId);
    if (!node) {
      await figma.loadAllPagesAsync();
      node = await figma.getNodeByIdAsync(p.componentId);
    }
    if (!node) throw new Error(`Component not found: ${p.componentId}`);
  } else {
    throw new Error("Either componentKey or componentId is required");
  }
```

### 8b. Rewrite `instanceSwapSingle` resolver

> **Note:** This also adds `variantProperties` matching to swap. Upstream only
> does `defaultVariant || children[0]` for component sets, giving no way to pick
> a specific variant. This replacement enables e.g. `{"size":"large","color":"brand"}`.

Find:
```ts
async function instanceSwapSingle(p: any) {
  const node = await figma.getNodeByIdAsync(p.id);
  if (!node) throw new Error(`Node not found: ${p.id}`);
  if (node.type !== "INSTANCE") throw new Error(`Node ${p.id} is ${node.type}, not an INSTANCE`);
  let comp: any = await figma.getNodeByIdAsync(p.componentId);
  if (!comp) throw new Error(`Component not found: ${p.componentId}`);
  if (comp.type === "COMPONENT_SET") comp = comp.defaultVariant || comp.children?.[0];
```
Replace with:
```ts
async function instanceSwapSingle(p: any) {
  const node = await figma.getNodeByIdAsync(p.id);
  if (!node) throw new Error(`Node not found: ${p.id}`);
  if (node.type !== "INSTANCE") throw new Error(`Node ${p.id} is ${node.type}, not an INSTANCE`);

  // Resolve component: componentKey (cross-file) takes priority over componentId (local)
  let comp: any;

  if (p.componentKey) {
    // Cross-file library import via published key hash (preferred path)
    try { comp = await (figma as any).importComponentSetByKeyAsync(p.componentKey); } catch (_) {}
    if (!comp) {
      try { comp = await figma.importComponentByKeyAsync(p.componentKey); } catch (_) {}
    }
    if (!comp) throw new Error(`Component not found by key: ${p.componentKey}`);
  } else if (p.componentId) {
    // Local node ID lookup
    comp = await figma.getNodeByIdAsync(p.componentId);
    if (!comp) {
      await figma.loadAllPagesAsync();
      comp = await figma.getNodeByIdAsync(p.componentId);
    }
    if (!comp) throw new Error(`Component not found: ${p.componentId}`);
  } else {
    throw new Error("Either componentKey or componentId is required");
  }

  // Resolve component set → pick matching variant or default
  if (comp.type === "COMPONENT_SET") {
    if (!comp.children?.length) throw new Error("Component set has no variants");
    if (p.variantProperties && typeof p.variantProperties === "object") {
      const match = comp.children.find((child: any) => {
        if (child.type !== "COMPONENT" || !child.variantProperties) return false;
        return Object.entries(p.variantProperties).every(
          ([k, v]) => {
            if (child.variantProperties[k] === v) return true;
            const prefixedKey = `${comp.name}/${k}`;
            return child.variantProperties[prefixedKey] === v;
          }
        );
      });
      if (match) comp = match;
      else {
        const prefix = `${comp.name}/`;
        const available = comp.children
          .filter((c: any) => c.type === "COMPONENT")
          .map((c: any) => {
            const props: Record<string, string> = {};
            for (const [k, v] of Object.entries(c.variantProperties || {})) {
              props[k.startsWith(prefix) ? k.slice(prefix.length) : k] = v as string;
            }
            return props;
          });
        throw new Error(`No variant matching ${JSON.stringify(p.variantProperties)} in ${comp.name}. Available: ${JSON.stringify(available)}`);
      }
    } else {
      comp = comp.defaultVariant || comp.children[0];
    }
  }

  if (comp.type !== "COMPONENT") throw new Error(`Resolved node is ${comp.type}, not a COMPONENT`);
  (node as InstanceNode).swapComponent(comp as ComponentNode);
  return {};
}
```

### 8c. Pass `textStyleKey` through in instance update text handler

> **Known issue:** This is in `instanceUpdateCombined` which is defined but
> **not wired into the `instances.update` dispatcher** (upstream bug). Apply
> it anyway so it's ready when upstream fixes the wiring.

Find (inside the `syntheticItems` map in `instanceUpdateCombined` — search
for `syntheticItems`):
```ts
      fontWeight: item.fontWeight,
      textStyleId: item.textStyleId,
```
Replace with:
```ts
      fontWeight: item.fontWeight,
      textStyleKey: item.textStyleKey,
      textStyleId: item.textStyleId,
```

### 8d. Fix `normalizeInlineChildTypes` to detect `componentKey`

Find (in `normalizeInlineChildTypes`, near the top of the file):
```ts
    } else if (child.componentId || child.id) {
```
Replace with:
```ts
    } else if (child.componentKey || child.componentId || child.id) {
```

---

## Step 9: `packages/adapter-figma/src/handlers/text.ts`

### 9a. Rewrite text style resolver in `setTextPropertiesSingle`

Find:
```ts
  // Text style takes priority
  let resolvedStyleId = p.textStyleId;
  if (!resolvedStyleId && p.textStyleName && ctx.textStyles) {
    const exact = ctx.textStyles.find((s: any) => s.name === p.textStyleName);
    if (exact) resolvedStyleId = exact.id;
    else {
      const fuzzy = ctx.textStyles.find((s: any) => s.name.toLowerCase().includes(p.textStyleName.toLowerCase()));
      if (fuzzy) resolvedStyleId = fuzzy.id;
    }
  }
  if (resolvedStyleId) {
    const s = await figma.getStyleByIdAsync(resolvedStyleId);
    if (s?.type === "TEXT") await (node as any).setTextStyleIdAsync(s.id);
  } else {
    if (p.fontWeight !== undefined) {
      const family = (node.fontName !== figma.mixed && node.fontName) ? node.fontName.family : "Inter";
      node.fontName = { family, style: getFontStyle(p.fontWeight) };
    }
    if (p.fontSize !== undefined) node.fontSize = p.fontSize;
  }
```
Replace with:
```ts
  // Text style: textStyleKey (cross-file) > textStyleId > textStyleName
  let resolvedStyleId: string | undefined;
  if (p.textStyleKey) {
    // Cross-file library import via published key hash (preferred path)
    const s = await figma.importStyleByKeyAsync(p.textStyleKey);
    if (s?.type === "TEXT") await (node as any).setTextStyleIdAsync(s.id);
  } else {
    resolvedStyleId = p.textStyleId;
    if (!resolvedStyleId && p.textStyleName && ctx.textStyles) {
      const exact = ctx.textStyles.find((s: any) => s.name === p.textStyleName);
      if (exact) resolvedStyleId = exact.id;
      else {
        const fuzzy = ctx.textStyles.find((s: any) => s.name.toLowerCase().includes(p.textStyleName.toLowerCase()));
        if (fuzzy) resolvedStyleId = fuzzy.id;
      }
    }
    if (resolvedStyleId) {
      const s = await figma.getStyleByIdAsync(resolvedStyleId);
      if (s?.type === "TEXT") await (node as any).setTextStyleIdAsync(s.id);
    } else {
      if (p.fontWeight !== undefined) {
        const family = (node.fontName !== figma.mixed && node.fontName) ? node.fontName.family : "Inter";
        node.fontName = { family, style: getFontStyle(p.fontWeight) };
      }
      if (p.fontSize !== undefined) node.fontSize = p.fontSize;
    }
  }
```

### 9b. Guard the style suggestion warning

Find (later in the same function):
```ts
  if (!resolvedStyleId && !p.textStyleName && !p.textStyleId &&
      (p.fontSize !== undefined || p.fontWeight !== undefined)) {
```
Replace with:
```ts
  if (!p.textStyleKey && !resolvedStyleId && !p.textStyleName && !p.textStyleId &&
      (p.fontSize !== undefined || p.fontWeight !== undefined)) {
```

---

## Step 10: `packages/adapter-figma/src/handlers/create-text.ts`

### 10a. Add `textStyleKey` to the prep phase (font preloading)

Find (in `prepSetTextProperties`, the text style resolution loop):
```ts
  const resolvedTextStyleMap = new Map<string, any>();
  for (const p of items) {
    let sid = p.textStyleId;
    let foundStyle: any = null;
    if (!sid && p.textStyleName && textStyles) {
      const exact = textStyles.find((s: any) => s.name === p.textStyleName);
      if (exact) { sid = exact.id; foundStyle = exact; }
      else {
        const fuzzy = textStyles.find((s: any) => s.name.toLowerCase().includes(p.textStyleName.toLowerCase()));
        if (fuzzy) { sid = fuzzy.id; foundStyle = fuzzy; }
      }
    }
    if (sid && !resolvedTextStyleMap.has(sid)) {
      // Use the style object found by name directly (avoids re-fetch failures for recently-created styles)
      const s = foundStyle ?? await figma.getStyleByIdAsync(sid);
      if (s?.type === "TEXT") {
        resolvedTextStyleMap.set(sid, s);
        const fn = (s as TextStyle).fontName;
        if (fn) fontRequests.push({ family: fn.family, style: fn.style });
      }
    }
  }
```
Replace with:
```ts
  // Resolve text style IDs and collect their fonts
  // Priority: textStyleKey (cross-file) > textStyleId > textStyleName
  const resolvedTextStyleMap = new Map<string, any>();
  for (const p of items) {
    // Cross-file library import via published key hash (preferred path)
    if (p.textStyleKey && !resolvedTextStyleMap.has(p.textStyleKey)) {
      const s = await figma.importStyleByKeyAsync(p.textStyleKey);
      if (s?.type === "TEXT") {
        resolvedTextStyleMap.set(p.textStyleKey, s);
        const fn = (s as TextStyle).fontName;
        if (fn) fontRequests.push({ family: fn.family, style: fn.style });
      }
    }
    if (!p.textStyleKey) {
      let sid = p.textStyleId;
      let foundStyle: any = null;
      if (!sid && p.textStyleName && textStyles) {
        const exact = textStyles.find((s: any) => s.name === p.textStyleName);
        if (exact) { sid = exact.id; foundStyle = exact; }
        else {
          const fuzzy = textStyles.find((s: any) => s.name.toLowerCase().includes(p.textStyleName.toLowerCase()));
          if (fuzzy) { sid = fuzzy.id; foundStyle = fuzzy; }
        }
      }
      if (sid && !resolvedTextStyleMap.has(sid)) {
        const s = foundStyle ?? await figma.getStyleByIdAsync(sid);
        if (s?.type === "TEXT") {
          resolvedTextStyleMap.set(sid, s);
          const fn = (s as TextStyle).fontName;
          if (fn) fontRequests.push({ family: fn.family, style: fn.style });
        }
      }
    }
  }
```

### 10b. Add `textStyleKey` to destructuring in `createTextSingle`

Find:
```ts
    parentId, textStyleId, textStyleName,
```
Replace with:
```ts
    parentId, textStyleKey, textStyleId, textStyleName,
```

### 10c. Add `textStyleKey` branch in the apply phase

Find (in `createTextSingle`, the text style application block):
```ts
    // Text style: by name > by ID (fonts already preloaded)
    let resolvedStyleId = textStyleId;
    if (!resolvedStyleId && textStyleName && ctx.textStyles) {
      const exact = ctx.textStyles.find((s: any) => s.name === textStyleName);
      if (exact) resolvedStyleId = exact.id;
      else {
        const fuzzy = ctx.textStyles.find((s: any) => s.name.toLowerCase().includes(textStyleName.toLowerCase()));
        if (fuzzy) resolvedStyleId = fuzzy.id;
      }
    }
    if (resolvedStyleId) {
      const cached = ctx.resolvedTextStyleMap.get(resolvedStyleId);
      if (cached) {
        try {
          await (textNode as any).setTextStyleIdAsync(cached.id);
        } catch (e: any) {
          hints.push({ type: "error", message: `textStyleName '${textStyleName || resolvedStyleId}' matched but failed to apply: ${e.message}` });
        }
      } else {
        hints.push({ type: "error", message: `textStyleName '${textStyleName || resolvedStyleId}' matched style ID '${resolvedStyleId}' but the style could not be loaded. It may be from a remote library or deleted.` });
      }
    } else if (textStyleName) {
      hints.push(styleNotFoundHint("textStyleName", textStyleName, ctx.textStyles!.map((s: any) => s.name)));
    } else {
      hints.push(await suggestTextStyle(fontSize, fontWeight));
    }
```
Replace with:
```ts
    // Text style: textStyleKey (cross-file) > textStyleId > textStyleName (fonts already preloaded)
    if (textStyleKey) {
      // Cross-file library import — already resolved in prep phase
      const cached = ctx.resolvedTextStyleMap.get(textStyleKey);
      if (cached) {
        try {
          await (textNode as any).setTextStyleIdAsync(cached.id);
        } catch (e: any) {
          hints.push({ type: "error", message: `textStyleKey '${textStyleKey}' resolved but failed to apply: ${e.message}` });
        }
      } else {
        hints.push({ type: "error", message: `textStyleKey '${textStyleKey}' could not be imported. Check the key hash is correct and the library is published.` });
      }
    } else {
      let resolvedStyleId = textStyleId;
      if (!resolvedStyleId && textStyleName && ctx.textStyles) {
        const exact = ctx.textStyles.find((s: any) => s.name === textStyleName);
        if (exact) resolvedStyleId = exact.id;
        else {
          const fuzzy = ctx.textStyles.find((s: any) => s.name.toLowerCase().includes(textStyleName.toLowerCase()));
          if (fuzzy) resolvedStyleId = fuzzy.id;
        }
      }
      if (resolvedStyleId) {
        const cached = ctx.resolvedTextStyleMap.get(resolvedStyleId);
        if (cached) {
          try {
            await (textNode as any).setTextStyleIdAsync(cached.id);
          } catch (e: any) {
            hints.push({ type: "error", message: `textStyleName '${textStyleName || resolvedStyleId}' matched but failed to apply: ${e.message}` });
          }
        } else {
          hints.push({ type: "error", message: `textStyleName '${textStyleName || resolvedStyleId}' matched style ID '${resolvedStyleId}' but the style could not be loaded. It may be from a remote library or deleted.` });
        }
      } else if (textStyleName) {
        hints.push(styleNotFoundHint("textStyleName", textStyleName, ctx.textStyles!.map((s: any) => s.name)));
      } else {
        hints.push(await suggestTextStyle(fontSize, fontWeight));
      }
    }
```

---

## Step 11: `packages/adapter-figma/src/handlers/patch-nodes.ts`

### 11a. Pass `textStyleKey` through in text update path (two locations)

**Occurrence 1** — in `patchSingleNode`, passing to `setTextPropertiesSingle` (around line 215):

Find:
```ts
      fills: item.fills,
      textStyleId: item.textStyleId,
      textStyleName: item.textStyleName,
```
Replace with:
```ts
      fills: item.fills,
      textStyleKey: item.textStyleKey,
      textStyleId: item.textStyleId,
      textStyleName: item.textStyleName,
```

**Occurrence 2** — in the `syntheticItems` batch map (around line 332):

Find:
```ts
      fontWeight: item.fontWeight,
      textStyleId: item.textStyleId,
      textStyleName: item.textStyleName,
```
Replace with:
```ts
      fontWeight: item.fontWeight,
      textStyleKey: item.textStyleKey,
      textStyleId: item.textStyleId,
      textStyleName: item.textStyleName,
```

### 11b. Add `variableKey` branch in variable bindings resolver

Find (in the bindings loop):
```ts
      const variable = b.variableName
        ? await findVariableByName(b.variableName)
        : await findVariableById(b.variableId);
      if (!variable) { hints.push({ type: "error", message: `Variable not found: ${b.variableName || b.variableId}` }); continue; }
```
Replace with:
```ts
      // Priority: variableName > variableKey (cross-file) > variableId (local)
      const variable = b.variableName
        ? await findVariableByName(b.variableName)
        : b.variableKey
          ? await figma.variables.importVariableByKeyAsync(b.variableKey)
          : b.variableId
            ? await findVariableById(b.variableId)
            : null;
      if (!variable) { hints.push({ type: "error", message: `Variable not found: ${b.variableName || b.variableKey || b.variableId}` }); continue; }
```

---

## Step 12: `packages/adapter-figma/src/handlers/helpers.ts`

### 12a. Update `findVariableById` JSDoc

Find:
```ts
/**
 * Resolve a variable by ID with scan fallback.
 * Direct lookup can fail for recently-created variables.
 */
export async function findVariableById(id: string): Promise<any> {
```
Replace with:
```ts
/**
 * Resolve a local variable by ID (e.g. VariableID:123).
 * For cross-file library variables, use variableKey param instead.
 * Tries direct async lookup first, then falls back to scanning all
 * local variables (direct lookup can fail for recently-created variables).
 */
export async function findVariableById(id: string): Promise<any> {
```

---

## ⚙️ Build

After all changes:
```sh
# Rebuild Figma plugin bundle
cd packages/adapter-figma && npx tsup

# Rebuild MCP server
cd ../.. && npm run build
```

Then reload the Figma plugin.

---

# Bonus — Enable `description` on node updates

**Unrelated to cross-file imports.** This is a general Vibma improvement that
lets you set component descriptions programmatically via `frames.update`. Useful
for the DS audit/scan workflow where you need to write descriptions on components.

Without this change, descriptions can only be set manually in Figma's right panel.

## Step 13: `packages/adapter-figma/src/handlers/patch-nodes.ts`

Find (near the top of the file):
```ts
const SIMPLE_PROPS = ["name", "visible", "locked", "rotation", "blendMode", "layoutPositioning", "overflowDirection"] as const;
```
Replace with:
```ts
const SIMPLE_PROPS = ["name", "description", "visible", "locked", "rotation", "blendMode", "layoutPositioning", "overflowDirection"] as const;
```

## Step 14: `schema/base/node.yaml`

Find (in the PatchItem properties):
```yaml
            name: { type: string, optional: true, description: "Rename node" }
            visible: { type: boolean, optional: true, description: "Show/hide (default true)" }
```
Replace with:
```yaml
            name: { type: string, optional: true, description: "Rename node" }
            description: { type: string, optional: true, description: "Set node description (visible in Figma's inspect panel). Useful for component documentation." }
            visible: { type: boolean, optional: true, description: "Show/hide (default true)" }
```

Then recompile schema (`npx tsx schema/compiler/index.ts`) and rebuild plugin.

**Usage:**
```
frames(update, { id: "<component-set-id>", description: "Your component description here" })
```

---

## Verification

### Part A — Key hash exposure

Connect to your DS source file and verify keys appear:
```
components.list              → each item has "key" field
styles.list(type: "text")    → each item has "key" field
variables.list(collectionId) → each item has "key" field
```

### Part B — Cross-file import

From a consumer file, test all paths:

| Test | Tool call |
|---|---|
| Create via `componentKey` | `instances(create, { componentKey: "<40-char-hex>" })` |
| Swap via `componentKey` | `instances(swap, { id: "...", componentKey: "<40-char-hex>" })` |
| Bind via `variableKey` | `frames(update, { bindings: [{ field: "fills/0/color", variableKey: "<40-char-hex>" }] })` |
| Text style create | `text(create, { text: "...", textStyleKey: "<40-char-hex>" })` |
| Text style update | `text(update, { id: "...", textStyleKey: "<40-char-hex>" })` |
| Local `componentId` | `instances(create, { componentId: "1:23" })` |
| Local `variableName` | `frames(update, { bindings: [{ field: "fills/0/color", variableName: "color/bg" }] })` |
| Local `textStyleName` | `text(update, { id: "...", textStyleName: "Body/M" })` |
