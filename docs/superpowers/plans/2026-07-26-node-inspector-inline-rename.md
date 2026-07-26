# Node Inspector Inline Rename Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Align the selected-node inspector with the Quest design system and make drag-created child nodes immediately renameable.

**Architecture:** Add title mutation to `useSpaceMap`, keeping optimistic local graph state and API persistence in the hook. Keep the inspector inside `SpaceCanvas`, but give it a controlled title editor with explicit save/cancel/focus state; the drag-to-create flow activates that editor for the new node.

**Tech Stack:** Next.js 16, React 19, TypeScript, React Flow, Vitest, Testing Library, NestJS PATCH node API.

## Global Constraints

- Use the existing `PATCH /nodes/:nodeId` API contract with `{ title }`.
- Keep overlays marked with `data-map-overlay`.
- Use existing Quest design tokens and CSS classes; do not introduce a second visual system.
- Keep imports grouped and ordered from smallest to largest visually.
- Preserve local fallback mode and the current drag/link behavior.
- Run web tests, lint, and build before claiming completion.

---

### Task 1: Add title mutation to the map hook

**Files:**
- Modify: `web/src/hooks/use-space-map.ts`
- Modify: `web/src/hooks/use-space-map.test.ts`

**Interfaces:**
- Consumes: `QuestApiClient.updateNode`, `SpaceNode`, and the existing graph state.
- Produces: `updateNodeTitle(nodeId: string, title: string): Promise<boolean>` that trims, updates local graph state, persists for API mode, and returns success.

- [ ] **Step 1: Write the failing hook tests**

Add tests for a non-empty title update in local mode and an API-mode PATCH payload. Assert that an empty trimmed title returns `false` and does not update the graph. Reuse the existing fetch/client mocks in the file.

- [ ] **Step 2: Run focused tests and verify the new expectations fail**

Run: `npm test -- --run src/hooks/use-space-map.test.ts`

Expected: FAIL because `updateNodeTitle` is not exposed yet.

- [ ] **Step 3: Implement the minimal mutation**

Add `updateNodeTitle` beside `updateNodeStatus`: trim and reject empty input; optimistically map the matching node’s title in `graph`; call `QuestApiClient.updateNode(nodeId, { title })` in API mode; replace the local node with the returned node on success; on failure restore the previous node and set the existing error.

```typescript
const updateNodeTitle = useCallback(async (nodeId: string, nextTitle: string) => {
  const title = nextTitle.trim();
  if (!title) return false;
  const previous = graph.nodes.find((node) => node.id === nodeId);
  if (!previous) return false;
  setGraph((current) => ({ ...current, nodes: current.nodes.map((node) => node.id === nodeId ? { ...node, title } : node) }));
  if (source !== 'api') return true;
  try {
    const updated = await new QuestApiClient(API_URL).updateNode(nodeId, { title });
    setGraph((current) => ({ ...current, nodes: current.nodes.map((node) => node.id === nodeId ? updated : node) }));
    setError(null);
    return true;
  } catch {
    setGraph((current) => ({ ...current, nodes: current.nodes.map((node) => node.id === nodeId ? previous : node) }));
    setError('Le titre n\'a pas pu être enregistré.');
    return false;
  }
}, [graph.nodes, source]);
```

- [ ] **Step 4: Run focused tests and lint**

Run: `npm test -- --run src/hooks/use-space-map.test.ts && npm run lint`

Expected: focused tests and ESLint pass.

- [ ] **Step 5: Commit the hook change**

```bash
git add src/hooks/use-space-map.ts src/hooks/use-space-map.test.ts
git commit -m "feat(web): support node title updates"
```

### Task 2: Redesign inspector and inline rename flow

**Files:**
- Modify: `web/src/components/space-canvas.tsx`
- Modify: `web/src/app/globals.css`
- Modify: `web/src/i18n/locales/fr.json`
- Modify: `web/src/i18n/locales/en.json`
- Create or modify: `web/src/components/space-canvas.test.tsx`

**Interfaces:**
- Consumes: `updateNodeTitle` from Task 1 and the existing node selection/actions.
- Produces: selected-node inspector with controlled title editing; a `startRename(nodeId, initialDraft?)` flow that focuses the title input for drag-created nodes.

- [ ] **Step 1: Write failing component tests**

Cover these behaviors with Testing Library:

```typescript
it('saves an edited inspector title on Enter', async () => {
  // select a node, edit [aria-label="Titre du node"], press Enter
  // expect updateNodeTitle(nodeId, 'Nouveau titre')
});

it('focuses the title field after creating a child by drag', async () => {
  // simulate the create-child completion
  // expect the title input toHaveFocus() and haveValue('')
});
```

Also cover Escape restoring the previous title and blur saving a non-empty title.

- [ ] **Step 2: Run focused component tests and verify they fail**

Run: `npm test -- --run src/components/space-canvas.test.tsx`

Expected: FAIL because the inspector currently renders a heading and no title editor/focus state.

- [ ] **Step 3: Implement controlled title editing**

Add `editingNodeId`, `titleDraft`, and a ref for the title input. The inspector renders a labelled input with the selected title; Enter/blur calls `updateNodeTitle`, Escape restores the original title. Only the drag-created node starts with an empty draft and `inputRef.current?.focus()` in an effect.

- [ ] **Step 4: Implement the design-system inspector markup and styles**

Use the existing `eyebrow`, `state-badge`, `secondary-button`, `danger`, `glass-panel`, and token variables. Add a close button, node-type eyebrow, helper copy, and consistent button grouping. Add responsive rules for the existing bottom-sheet mode.

- [ ] **Step 5: Add translation keys**

Add matching FR/EN keys for title label, placeholder, save/error copy, close inspector, and the temporary child title. No user-visible hardcoded strings may remain in the component.

- [ ] **Step 6: Run focused tests and lint**

Run: `npm test -- --run src/components/space-canvas.test.tsx && npm run lint`

Expected: component tests and ESLint pass.

- [ ] **Step 7: Commit the inspector change**

```bash
git add src/components/space-canvas.tsx src/app/globals.css src/i18n/locales/fr.json src/i18n/locales/en.json src/components/space-canvas.test.tsx
git commit -m "feat(web): redesign node inspector and inline rename"
```

### Task 3: Delivery documentation and validation

**Files:**
- Create: `docs/node-inspector-rename/fonctionnement.md`
- Create: `docs/node-inspector-rename/guide-test.md`
- Create: `http/node-inspector-rename.http`

- [ ] **Step 1: Write the feature explanation**

Document the inspector structure, rename state machine, drag-to-create flow, PATCH request, and error behavior with an ASCII flow.

- [ ] **Step 2: Write the manual guide and HTTP artifact**

Cover existing-node rename, Enter/blur/Escape, drag-created child focus, empty-title behavior, API-offline behavior, responsive layout, and the optional API request needed to inspect the PATCH contract.

- [ ] **Step 3: Run complete validation**

Run from `web/`: `npm run validate`.

Expected: all tests, ESLint, TypeScript, and Next build pass.

- [ ] **Step 4: Verify scope and whitespace**

Run from root: `git diff --check`, `git status --short`, and inspect the web submodule pointer. Existing unrelated changes in `api`, `.claude`, and prior docs must remain untouched.
