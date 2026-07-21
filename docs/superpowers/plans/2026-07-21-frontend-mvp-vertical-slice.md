# Frontend MVP Vertical Slice Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the Quest desktop web MVP where a user can create a local quest, navigate its full-screen interactive map, and inspect its steps.

**Architecture:** The `web/` submodule becomes a Next.js + TypeScript app. Local quest state lives in a focused React context; presentation components receive typed data and callbacks. React Flow owns graph rendering and camera state, while `useMomentumPan` decorates camera movement with inertia without coupling it to quest data.

**Tech Stack:** Next.js App Router, React, TypeScript, `@xyflow/react`, Vitest, React Testing Library, ESLint.

## Global Constraints

- Implement desktop-first; do not add a mobile layout in this slice.
- Keep all state local to the browser; do not call the API or introduce persistence.
- Use gradients and CSS stars only; do not add icon packs, image assets, WebGL, or a final themed background.
- The map route must occupy the complete browser viewport.
- Do not make `.superpowers/` tracked content.
- Use conventional commits and keep frontend commits inside the `web/` submodule; update the root submodule pointer only after an accepted frontend commit.

---

## File Structure

```
web/
├── app/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── map/
│   │   ├── page.test.tsx
│   │   └── page.tsx
│   └── globals.css
├── features/
│   ├── map/
│   │   ├── QuestMap.tsx
│   │   ├── QuestMap.test.tsx
│   │   ├── QuestNode.tsx
│   │   └── useMomentumPan.ts
│   └── quest/
│       ├── fixtures.ts
│       ├── QuestComposer.tsx
│       ├── QuestComposer.test.tsx
│       ├── QuestProvider.tsx
│       ├── StepInspector.tsx
│       ├── types.ts
│       └── questModel.test.ts
├── test/setup.ts
├── vitest.config.ts
└── package.json
```

## Task 1: Initialize the frontend foundation

**Files:**
- Create: `web/package.json`
- Create: `web/tsconfig.json`
- Create: `web/next.config.ts`
- Create: `web/eslint.config.mjs`
- Create: `web/vitest.config.ts`
- Create: `web/test/setup.ts`
- Create: `web/app/layout.tsx`
- Create: `web/app/page.tsx`
- Create: `web/app/map/page.tsx`
- Create: `web/app/globals.css`
- Modify: `web/README.md`

**Interfaces:**
- Produces the `/map` route and the `npm run dev`, `npm run lint`, `npm run test`, and `npm run build` commands used by every later task.

- [ ] **Step 1: Create the Next.js project configuration and install dependencies**

  In `web/`, add scripts for `dev`, `build`, `start`, `lint`, and `test`. Install runtime dependencies `next`, `react`, `react-dom`, and `@xyflow/react`; install development dependencies `typescript`, `@types/node`, `@types/react`, `@types/react-dom`, `eslint`, `eslint-config-next`, `vitest`, `jsdom`, `@testing-library/react`, and `@testing-library/jest-dom`.

- [ ] **Step 2: Add the first route test and verify it fails**

  Create `web/app/map/page.test.tsx` with:

  ```tsx
  import { render, screen } from '@testing-library/react';
  import MapPage from './page';

  test('renders the interactive map landmark', () => {
    render(<MapPage />);
    expect(screen.getByRole('main', { name: 'Carte de quête' })).toBeInTheDocument();
  });
  ```

  Run: `npm run test -- app/map/page.test.tsx`

  Expected: FAIL because the route component does not exist yet.

- [ ] **Step 3: Implement the minimal app shell**

  - `app/layout.tsx` exports root metadata titled `Quest` and imports `globals.css`.
  - `app/page.tsx` redirects to `/map` using `redirect('/map')`.
  - `app/map/page.tsx` renders `<main aria-label="Carte de quête" />`.
  - `globals.css` resets the page margin, makes `html`, `body`, and `main` viewport-sized, and defines the existing deep-blue gradient palette as CSS custom properties.

- [ ] **Step 4: Verify the foundation**

  Run: `npm run test -- app/map/page.test.tsx && npm run lint && npm run build`

  Expected: all commands exit with code `0`.

- [ ] **Step 5: Commit the frontend foundation**

  ```bash
  git add package.json package-lock.json tsconfig.json next.config.ts eslint.config.mjs vitest.config.ts test app README.md
  git commit -m "feat: initialize quest map frontend"
  ```

## Task 2: Add the local quest model and map fixtures

**Files:**
- Create: `web/features/quest/types.ts`
- Create: `web/features/quest/fixtures.ts`
- Create: `web/features/quest/QuestProvider.tsx`
- Create: `web/features/quest/questModel.test.ts`

**Interfaces:**
- Produces `QuestProvider`, `useQuest`, `createQuest(input)`, `selectStep(stepId)`, and typed `Quest`/`Step` values.
- Consumes the app shell from Task 1.

- [ ] **Step 1: Write the failing model tests**

  Test that `createQuest({ title: 'Lancer mon SaaS', stepTitles: ['Valider l’idée', 'Construire le MVP'] })` returns two steps, with the first `active`, the second `locked`, and distinct IDs. Test that an empty title or an empty `stepTitles` array returns a validation error.

  Run: `npm run test -- features/quest/questModel.test.ts`

  Expected: FAIL because the model creator is missing.

- [ ] **Step 2: Implement the types and pure quest creator**

  In `types.ts`, define `StepStatus`, `Step`, `Quest`, and `CreateQuestInput`. Export `createQuest(input)` from `fixtures.ts`; it trims values, rejects invalid input, and creates `parentStepId` as the prior step ID for a linear first quest. Export one four-step demonstration quest matching the approved mock states.

- [ ] **Step 3: Add the local provider**

  `QuestProvider` stores `quests`, `activeQuestId`, and `selectedStepId` with `useState`. `useQuest` throws a clear error when used outside the provider. `createQuest` adds a validated quest, marks it active, and selects its first step.

- [ ] **Step 4: Verify the model and provider**

  Run: `npm run test -- features/quest/questModel.test.ts && npm run lint`

  Expected: all assertions and lint checks pass.

- [ ] **Step 5: Commit the quest model**

  ```bash
  git add features/quest
  git commit -m "feat: add local quest state"
  ```

## Task 3: Render the navigable graph

**Files:**
- Create: `web/features/map/QuestMap.tsx`
- Create: `web/features/map/QuestMap.test.tsx`
- Create: `web/features/map/QuestNode.tsx`
- Create: `web/features/map/useMomentumPan.ts`
- Modify: `web/app/map/page.tsx`
- Modify: `web/app/globals.css`

**Interfaces:**
- Consumes `Quest`, `Step`, `useQuest`, and `@xyflow/react`.
- Produces `QuestMap({ quest: Quest })`, which calls `selectStep` when a node is clicked.

- [ ] **Step 1: Write the failing graph conversion tests**

  Test a linear three-step quest with `toFlowElements(quest)` returns three nodes and two edges. Assert the source and target IDs of each edge correspond to `parentStepId` values.

  Run: `npm run test -- features/map/QuestMap.test.tsx`

  Expected: FAIL because `toFlowElements` is missing.

- [ ] **Step 2: Implement graph conversion and custom nodes**

  Export `toFlowElements(quest)` from `QuestMap.tsx`. Each node data value contains `id`, `title`, `status`, and `progressPercent`; each edge uses the `step.id` and `parentStepId` pair. `QuestNode` renders the title, visible status text, and a status-specific CSS class. Use React Flow custom node types, a dashed edge style, `fitView`, and only the supplied quest data.

- [ ] **Step 3: Implement panning, zoom and inertia**

  `useMomentumPan` receives the React Flow instance and a map surface ref. It records pointer displacement and timestamp only when the target is not inside `[data-map-overlay]`. On pointer release, it calls `setViewport` in `requestAnimationFrame` while multiplying velocity by `0.91`; it stops below `0.18` pixels per frame. Wheel, pinch and zoom controls remain handled by React Flow. The React Flow wrapper covers the viewport; panels carry `data-map-overlay`.

- [ ] **Step 4: Compose the map route and verify behavior**

  Wrap `/map` with `QuestProvider`. Render `QuestMap` with the active quest. Add an accessible map label and a manual test checklist in `web/README.md` covering pan on the empty background, overlay exclusion, wheel zoom, pinch zoom, node selection, and a fast drag with inertia.

  Run: `npm run test -- features/map/QuestMap.test.tsx && npm run lint && npm run build`

  Expected: all commands exit with code `0`.

- [ ] **Step 5: Commit the map foundation**

  ```bash
  git add app/map features/map app/globals.css README.md
  git commit -m "feat: add interactive quest map"
  ```

## Task 4: Add quest creation and step inspection overlays

**Files:**
- Create: `web/features/quest/QuestComposer.tsx`
- Create: `web/features/quest/QuestComposer.test.tsx`
- Create: `web/features/quest/StepInspector.tsx`
- Modify: `web/app/map/page.tsx`
- Modify: `web/app/globals.css`

**Interfaces:**
- Consumes `useQuest`, `createQuest`, and the selected step ID from Task 2.
- Produces overlays marked `data-map-overlay`, so they never start a map pan.

- [ ] **Step 1: Write the failing composer tests**

  Test that submitting a blank goal displays `L’objectif est requis`. Test that a valid goal with two step inputs invokes `createQuest` with the trimmed goal and the two non-empty step titles.

  Run: `npm run test -- features/quest/QuestComposer.test.tsx`

  Expected: FAIL because `QuestComposer` does not exist.

- [ ] **Step 2: Implement the composer**

  Render a left floating panel with a title input, editable initial-step inputs, an `Ajouter une étape` button, and a `Créer la quête` submit button. Start with three step inputs. Reject an empty title or a submission with no non-empty step. Mark the panel and its controls with `data-map-overlay`.

- [ ] **Step 3: Implement the inspector**

  Render nothing when no step is selected. Otherwise show a right floating panel with the selected title, human-readable status, progress percentage, and a progress bar. Mark it `data-map-overlay`.

- [ ] **Step 4: Integrate and verify the full slice**

  Render both overlays in `app/map/page.tsx`. Use gradient surfaces, compact typography, and the approved status colors. Verify that a created quest becomes the active graph and its first step is visible in the inspector.

  Run: `npm run test && npm run lint && npm run build`

  Expected: all commands exit with code `0`.

- [ ] **Step 5: Commit the complete slice**

  ```bash
  git add app/map features/quest app/globals.css README.md
  git commit -m "feat: create and inspect local quests"
  ```

## Task 5: Root repository handoff

**Files:**
- Modify: root `web` gitlink
- Modify: `README.md` only if the public project overview changes

**Interfaces:**
- Consumes accepted commits from Tasks 1–4 in the `web/` submodule.
- Produces an updated root pointer to the tested frontend revision.

- [ ] **Step 1: Verify the web submodule is clean and tested**

  Run from `web/`: `git status --short && npm run test && npm run lint && npm run build`

  Expected: no uncommitted files and all commands exit with code `0`.

- [ ] **Step 2: Update the root submodule pointer**

  Run from the root: `git add web && git diff --cached --submodule=log`

  Expected: the diff contains only the accepted frontend commits.

- [ ] **Step 3: Commit the root handoff**

  ```bash
  git commit -m "chore: update quest web frontend"
  ```

## Plan Self-Review

- Spec coverage: Tasks 1–4 cover full viewport, local creation, graph nodes/edges, pan, inertia, zoom, selection, detail panel, gradients, and the desktop-only boundary. Task 5 covers the submodule handoff.
- Placeholder scan: no unresolved markers or implicit implementation steps remain.
- Interface consistency: `QuestProvider` is introduced before all consumers; `toFlowElements`, `QuestMap`, and `useMomentumPan` are produced before route composition; all overlay components use the single `data-map-overlay` contract.
