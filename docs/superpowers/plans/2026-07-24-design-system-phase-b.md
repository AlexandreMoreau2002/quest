# Design System Phase B — Ancrages & Drag-to-Create Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add contextual hover anchors on cards, a drag-to-create-branch interaction that opens a micro-menu (add linked goal / add step / link existing), automatic collision-free placement of new nodes, re-routing of an existing link's anchor, and a full keyboard alternative — building on Phase A's token system, fixed card sizes, and `intersectRectangle` geometry.

**Architecture:** A new pure placement module (`placement.ts`) computes non-overlapping positions by ring/sector. A new `use-branch-drag` hook owns pointer tracking and drag state, rendered via a new `NodeAnchor` component positioned with `intersectRectangle`. The API gains `parentStepId` on `PATCH /steps/:id` for re-parenting; `QuestApiClient` and `use-quest-map` gain matching methods. Keyboard support reuses the same micro-menu trigger as drag-drop, via `onKeyDown` on the anchor.

**Tech Stack:** NestJS/Prisma (api), Next.js/React/TypeScript/@xyflow/react (web), Vitest.

---

## Fichiers concernés

- Modify `api/src/steps/dto/update-step.dto.ts` — add optional `parentStepId`.
- Modify `api/src/steps/steps.service.ts` — validate re-parenting (same quest, no cycle).
- Create `api/src/steps/steps.service.spec.ts` if it doesn't already exist (check first) — test re-parent validation.
- Modify `web/src/lib/api/client.ts` — add `updateStep` method.
- Create `web/src/lib/map/placement.ts` + `web/src/lib/map/placement.test.ts` — pure placement algorithm.
- Create `web/src/components/node-anchor.tsx` — anchor UI (hover dot / "+" badge).
- Create `web/src/hooks/use-branch-drag.ts` — drag state machine.
- Create `web/src/components/branch-menu.tsx` — micro-menu popover (3 choices).
- Modify `web/src/components/quest-map.tsx` — wire hover state, anchors, drag, menu, keyboard.
- Modify `web/src/hooks/use-quest-map.ts` — expose `reparentStep`, use `placeNode` for new steps.
- Modify `web/src/app/globals.css` — anchor/menu styles using Phase A tokens.

---

### Task 1: API — support re-parenting via `PATCH /steps/:id`

**Files:**
- Modify: `api/src/steps/dto/update-step.dto.ts`
- Modify: `api/src/steps/steps.service.ts`
- Test: `api/src/steps/steps.service.spec.ts` (create if absent)

- [ ] **Step 1: Check for an existing spec file**

Run: `ls api/src/steps/steps.service.spec.ts 2>&1`

If it exists, read it first and add tests alongside the existing ones, following the same mocking pattern (don't invent a new pattern). If absent, create it fresh using the pattern below.

- [ ] **Step 2: Write the failing test**

```ts
// api/src/steps/steps.service.spec.ts (add if new, or append describe block if file exists)
import { Test } from '@nestjs/testing';
import { ConflictException } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { StepsService } from './steps.service';

describe('StepsService.update — re-parenting', () => {
  let service: StepsService;
  const prisma = {
    step: {
      update: jest.fn(),
      findUnique: jest.fn(),
    },
  };

  beforeEach(async () => {
    jest.clearAllMocks();
    const moduleRef = await Test.createTestingModule({
      providers: [StepsService, { provide: PrismaService, useValue: prisma }],
    }).compile();
    service = moduleRef.get(StepsService);
  });

  it('rejects re-parenting to a step from a different quest', async () => {
    prisma.step.findUnique
      .mockResolvedValueOnce({ id: 'step-1', questId: 'quest-a' }) // the step being updated
      .mockResolvedValueOnce({ id: 'step-2', questId: 'quest-b' }); // the target parent

    await expect(service.update('step-1', { parentStepId: 'step-2' })).rejects.toThrow(ConflictException);
    expect(prisma.step.update).not.toHaveBeenCalled();
  });

  it('rejects re-parenting a step to itself', async () => {
    prisma.step.findUnique.mockResolvedValueOnce({ id: 'step-1', questId: 'quest-a' });

    await expect(service.update('step-1', { parentStepId: 'step-1' })).rejects.toThrow(ConflictException);
    expect(prisma.step.update).not.toHaveBeenCalled();
  });

  it('allows re-parenting within the same quest', async () => {
    prisma.step.findUnique
      .mockResolvedValueOnce({ id: 'step-1', questId: 'quest-a' })
      .mockResolvedValueOnce({ id: 'step-2', questId: 'quest-a' });
    prisma.step.update.mockResolvedValueOnce({ id: 'step-1', parentStepId: 'step-2' });

    const result = await service.update('step-1', { parentStepId: 'step-2' });

    expect(result).toEqual({ id: 'step-1', parentStepId: 'step-2' });
    expect(prisma.step.update).toHaveBeenCalledWith({
      where: { id: 'step-1' },
      data: { parentStepId: 'step-2' },
    });
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `cd api && npm test -- steps.service.spec`
Expected: FAIL — `parentStepId` isn't accepted by `UpdateStepDto` (validation/type error) or the service doesn't perform the ownership/cycle checks yet.

- [ ] **Step 4: Add `parentStepId` to the DTO**

In `api/src/steps/dto/update-step.dto.ts`, add alongside the existing fields (keep the `@AtLeastOneOf` list in sync):

```ts
import { Transform, Type } from 'class-transformer';
import { IsEnum, IsInt, IsNotEmpty, IsOptional, IsString, IsUUID, Max, Min } from 'class-validator';
import { StepStatus } from '@prisma/client';
import { AtLeastOneOf } from '../../common/at-least-one-of.decorator';

function trim(value: unknown) {
  return typeof value === 'string' ? value.trim() : value;
}

export class UpdateStepDto {
  @Transform(({ value }) => trim(value))
  @IsOptional()
  @IsString()
  @IsNotEmpty()
  title?: string;

  @IsOptional()
  @IsEnum(StepStatus)
  status?: StepStatus;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(0)
  @Max(100)
  progressPercent?: number;

  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(0)
  order?: number;

  @IsOptional()
  @IsUUID()
  parentStepId?: string;

  @AtLeastOneOf(['title', 'status', 'progressPercent', 'order', 'parentStepId'])
  private readonly _atLeastOneField?: never;
}
```

- [ ] **Step 5: Add validation logic in the service**

In `api/src/steps/steps.service.ts`, modify the `update` method (currently lines 44-61) to validate `parentStepId` before writing:

```ts
async update(stepId: string, updateStepDto: UpdateStepDto) {
  if (updateStepDto.parentStepId !== undefined) {
    if (updateStepDto.parentStepId === stepId) {
      throw new ConflictException('A step cannot be its own parent.');
    }

    const [step, parent] = await Promise.all([
      this.prisma.step.findUnique({ where: { id: stepId }, select: { questId: true } }),
      this.prisma.step.findUnique({ where: { id: updateStepDto.parentStepId }, select: { questId: true } }),
    ]);

    if (step === null) {
      throw new NotFoundException(`Step with id "${stepId}" was not found.`);
    }
    if (parent === null || parent.questId !== step.questId) {
      throw new ConflictException('The parent step must belong to the same quest.');
    }
  }

  try {
    return await this.prisma.step.update({
      where: { id: stepId },
      data: updateStepDto,
    });
  } catch (error) {
    if (error instanceof Prisma.PrismaClientKnownRequestError) {
      if (error.code === 'P2025') {
        throw new NotFoundException(`Step with id "${stepId}" was not found.`);
      }
      if (error.code === 'P2002') {
        throw new ConflictException('A step with this title already exists under the same parent.');
      }
    }
    throw error;
  }
}
```

Note: this validates *direct* self-parenting and cross-quest parenting, but not deeper cycles (e.g. re-parenting a step under its own descendant). Deeper cycle detection is deliberately out of scope for this task — the UI in Task 6 only offers re-parenting onto currently-visible siblings/cousins within the same quest, and a full ancestor-chain walk adds complexity disproportionate to the MVP drag interaction. If this becomes a real issue, it can be added as a follow-up.

- [ ] **Step 6: Run test to verify it passes**

Run: `cd api && npm test -- steps.service.spec`
Expected: PASS (3/3 new tests, plus any pre-existing tests in the file still passing).

- [ ] **Step 7: Run full API validation**

Run: `cd api && npm run lint && npm run build`
Expected: both pass.

- [ ] **Step 8: Commit**

```bash
cd api && git add src/steps/dto/update-step.dto.ts src/steps/steps.service.ts src/steps/steps.service.spec.ts
git commit -m "feat: support step re-parenting via PATCH /steps/:id"
```

---

### Task 2: Web API client — `updateStep`

**Files:**
- Modify: `web/src/lib/api/client.ts`
- Test: `web/src/lib/api/client.test.ts`

- [ ] **Step 1: Read the existing test file to match its mocking pattern**

Run: `cat web/src/lib/api/client.test.ts`

- [ ] **Step 2: Write the failing test** (adapt to match the existing file's `fetch` mocking style — if it uses `vi.stubGlobal('fetch', ...)` or similar, follow that pattern; the test below assumes a `vi.fn()` global fetch mock consistent with typical Vitest setup)

```ts
it('updateStep sends a PATCH request with the given fields', async () => {
  const fetchMock = vi.fn().mockResolvedValue({
    ok: true,
    json: async () => ({ id: 'step-1', parentStepId: 'step-2' }),
  });
  vi.stubGlobal('fetch', fetchMock);

  const client = new QuestApiClient('http://localhost:3001');
  const result = await client.updateStep('step-1', { parentStepId: 'step-2' });

  expect(fetchMock).toHaveBeenCalledWith(
    'http://localhost:3001/steps/step-1',
    expect.objectContaining({
      method: 'PATCH',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ parentStepId: 'step-2' }),
    }),
  );
  expect(result).toEqual({ id: 'step-1', parentStepId: 'step-2' });
});
```

- [ ] **Step 3: Run to verify it fails**

Run: `cd web && npm test -- client.test`
Expected: FAIL — `updateStep` doesn't exist.

- [ ] **Step 4: Implement**

In `web/src/lib/api/client.ts`, add a type and method:

```ts
export type UpdateStepInput = {
  title?: string;
  status?: QuestStep['status'];
  parentStepId?: string;
  order?: number;
};
```

Add a private `patchJson` helper (mirrors `sendJson` but with `PATCH`) and the public method:

```ts
updateStep(stepId: string, input: UpdateStepInput): Promise<QuestStep> {
  return this.patchJson<QuestStep>(`/steps/${stepId}`, input);
}
```

```ts
private async patchJson<T>(path: string, body: object): Promise<T> {
  const response = await fetch(`${this.baseUrl.replace(/\/$/, '')}${path}`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(body),
  });
  if (!response.ok) {
    throw new Error(`Mise à jour impossible (${response.status})`);
  }
  return response.json() as Promise<T>;
}
```

- [ ] **Step 5: Run to verify it passes**

Run: `cd web && npm test -- client.test`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
cd web && git add src/lib/api/client.ts src/lib/api/client.test.ts
git commit -m "feat: add updateStep to QuestApiClient for re-parenting"
```

---

### Task 3: Placement algorithm — `placeNode`

**Files:**
- Create: `web/src/lib/map/placement.ts`
- Create: `web/src/lib/map/placement.test.ts`

- [ ] **Step 1: Write the failing tests**

```ts
// web/src/lib/map/placement.test.ts
import { describe, expect, it } from 'vitest';

import { placeNode, type PlacedRect } from './placement';

describe('placeNode', () => {
  const parent: PlacedRect = { x: 0, y: 0, width: 272, height: 132 };

  it('places the first child at least MIN_RING_RADIUS away from the parent edge', () => {
    const position = placeNode({ parent, siblingIndex: 0, siblingCount: 1, existing: [] });
    const parentCenter = { x: parent.x + parent.width / 2, y: parent.y + parent.height / 2 };
    const distanceFromCenter = Math.hypot(position.x - parentCenter.x, position.y - parentCenter.y);
    const parentHalfDiagonal = Math.hypot(parent.width / 2, parent.height / 2);

    expect(distanceFromCenter).toBeGreaterThanOrEqual(parentHalfDiagonal + 180 - 1);
  });

  it('spreads multiple siblings across different angles', () => {
    const first = placeNode({ parent, siblingIndex: 0, siblingCount: 3, existing: [] });
    const second = placeNode({ parent, siblingIndex: 1, siblingCount: 3, existing: [] });
    const third = placeNode({ parent, siblingIndex: 2, siblingCount: 3, existing: [] });

    expect(first).not.toEqual(second);
    expect(second).not.toEqual(third);
    expect(first).not.toEqual(third);
  });

  it('nudges along the ring to avoid overlapping an existing card', () => {
    const existing: PlacedRect = { x: 300, y: 0, width: 224, height: 100 };
    const position = placeNode({ parent, siblingIndex: 0, siblingCount: 1, existing: [existing] });

    const overlaps = Math.abs(position.x - (existing.x + existing.width / 2)) < (224 + 224) / 2
      && Math.abs(position.y - (existing.y + existing.height / 2)) < (100 + 100) / 2;
    expect(overlaps).toBe(false);
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `cd web && npm test -- placement.test`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

```ts
// web/src/lib/map/placement.ts
export interface PlacedRect {
  x: number;
  y: number;
  width: number;
  height: number;
}

interface PlaceNodeInput {
  parent: PlacedRect;
  siblingIndex: number;
  siblingCount: number;
  existing: PlacedRect[];
  newNodeWidth?: number;
  newNodeHeight?: number;
}

const MIN_RING_RADIUS = 180;
const SECTOR_DEGREES = 45;
const NEW_NODE_WIDTH = 224;
const NEW_NODE_HEIGHT = 100;

function center(rect: PlacedRect) {
  return { x: rect.x + rect.width / 2, y: rect.y + rect.height / 2 };
}

function rectsOverlap(a: { x: number; y: number; width: number; height: number }, b: PlacedRect): boolean {
  return Math.abs(a.x - (b.x + b.width / 2)) < (a.width + b.width) / 2
    && Math.abs(a.y - (b.y + b.height / 2)) < (a.height + b.height) / 2;
}

/**
 * Places a new node on a ring around its parent: minimum radius past the parent's
 * half-diagonal, one ~45° sector per existing sibling, then nudges along the ring
 * in both directions until it clears every existing card.
 */
export function placeNode({ parent, siblingIndex, siblingCount, existing, newNodeWidth = NEW_NODE_WIDTH, newNodeHeight = NEW_NODE_HEIGHT }: PlaceNodeInput): { x: number; y: number } {
  const parentCenter = center(parent);
  const parentHalfDiagonal = Math.hypot(parent.width / 2, parent.height / 2);
  const radius = parentHalfDiagonal + MIN_RING_RADIUS;

  const baseAngle = -90; // start pointing "up" and fan out
  const spread = (siblingIndex - (siblingCount - 1) / 2) * SECTOR_DEGREES;
  const angleDeg = baseAngle + spread;

  const candidateAt = (deg: number) => {
    const rad = (deg * Math.PI) / 180;
    return {
      x: parentCenter.x + Math.cos(rad) * radius,
      y: parentCenter.y + Math.sin(rad) * radius,
    };
  };

  const fits = (point: { x: number; y: number }) => !existing.some((rect) => rectsOverlap({ ...point, width: newNodeWidth, height: newNodeHeight }, rect));

  const initial = candidateAt(angleDeg);
  if (fits(initial)) return initial;

  for (let stepDeg = 5; stepDeg <= 180; stepDeg += 5) {
    const clockwise = candidateAt(angleDeg + stepDeg);
    if (fits(clockwise)) return clockwise;
    const counterClockwise = candidateAt(angleDeg - stepDeg);
    if (fits(counterClockwise)) return counterClockwise;
  }

  return initial;
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `cd web && npm test -- placement.test`
Expected: PASS (3/3).

- [ ] **Step 5: Commit**

```bash
cd web && git add src/lib/map/placement.ts src/lib/map/placement.test.ts
git commit -m "feat: add ring-based automatic node placement algorithm"
```

---

### Task 4: `NodeAnchor` component

**Files:**
- Create: `web/src/components/node-anchor.tsx`
- Modify: `web/src/app/globals.css`

- [ ] **Step 1: Add anchor CSS**

Append to `web/src/app/globals.css`:

```css
.node-anchor {
  position: absolute; z-index: 6; width: 20px; height: 20px; border-radius: 50%;
  display: grid; place-items: center; cursor: grab;
  background: var(--q-surface); border: 1.5px solid var(--q-border-strong);
  transform: translate(-50%, -50%);
  transition: transform .15s, box-shadow .15s;
}
.node-anchor:hover, .node-anchor:focus-visible { box-shadow: 0 0 0 4px rgba(173, 150, 255, .18); }
.node-anchor:focus-visible { outline: 2px solid var(--q-accent); outline-offset: 2px; }
.node-anchor.node-anchor--grow { background: var(--q-cta-bg); border: none; color: var(--q-cta-text); font-size: 13px; font-weight: 700; }
```

- [ ] **Step 2: Implement the component**

```tsx
// web/src/components/node-anchor.tsx
'use client';

interface NodeAnchorProps {
  x: number;
  y: number;
  variant: 'grow' | 'used';
  label: string;
  onActivate: () => void;
}

export function NodeAnchor({ x, y, variant, label, onActivate }: NodeAnchorProps) {
  return (
    <button
      type="button"
      className={`node-anchor ${variant === 'grow' ? 'node-anchor--grow' : ''}`}
      style={{ left: x, top: y }}
      aria-label={label}
      onClick={onActivate}
      onKeyDown={(event) => {
        if (event.key === 'Enter' || event.key === ' ') {
          event.preventDefault();
          onActivate();
        }
      }}
    >
      {variant === 'grow' ? '+' : ''}
    </button>
  );
}
```

`NodeAnchor` is deliberately a real `<button>` (not a div with `role="button"`) so it's natively keyboard-operable without the workaround Phase A needed for `QuestNode` — this satisfies the plan's keyboard-alternative requirement (Enter/Space opens the same micro-menu a drag would) for free.

- [ ] **Step 3: Verify build**

Run: `cd web && npm run build`
Expected: passes (component isn't wired into `quest-map.tsx` yet — this task only creates it).

- [ ] **Step 4: Commit**

```bash
cd web && git add src/components/node-anchor.tsx src/app/globals.css
git commit -m "feat: add NodeAnchor component for hover/keyboard branch creation"
```

---

### Task 5: `BranchMenu` micro-interface

**Files:**
- Create: `web/src/components/branch-menu.tsx`
- Modify: `web/src/app/globals.css`

- [ ] **Step 1: Add menu CSS**

Append to `web/src/app/globals.css`:

```css
.branch-menu { position: absolute; z-index: 7; min-width: 220px; padding: 8px; }
.branch-menu button { display: block; width: 100%; padding: 10px 12px; margin: 0; text-align: left; border-radius: 8px; background: transparent; color: var(--q-text); font-size: 13px; font-weight: 500; }
.branch-menu button:hover, .branch-menu button:focus-visible { background: rgba(255,255,255,.08); }
.branch-menu input { margin-top: 6px; }
```

- [ ] **Step 2: Implement the component**

```tsx
// web/src/components/branch-menu.tsx
'use client';

import { useState } from 'react';

export type BranchMenuChoice =
  | { kind: 'linked-goal'; title: string }
  | { kind: 'step'; title: string }
  | { kind: 'existing'; nodeId: string };

interface ExistingOption {
  id: string;
  title: string;
}

interface BranchMenuProps {
  x: number;
  y: number;
  existingOptions: ExistingOption[];
  onChoose: (choice: BranchMenuChoice) => void;
  onDismiss: () => void;
}

export function BranchMenu({ x, y, existingOptions, onChoose, onDismiss }: BranchMenuProps) {
  const [mode, setMode] = useState<'menu' | 'linked-goal' | 'step' | 'existing'>('menu');
  const [title, setTitle] = useState('');
  const [search, setSearch] = useState('');

  const filteredOptions = existingOptions.filter((option) => option.title.toLowerCase().includes(search.toLowerCase()));

  return (
    <div className="branch-menu glass-panel" style={{ left: x, top: y }} data-map-overlay role="menu">
      {mode === 'menu' && (
        <>
          <button type="button" onClick={() => setMode('linked-goal')}>Ajouter un objectif lié</button>
          <button type="button" onClick={() => setMode('step')}>Ajouter une étape</button>
          <button type="button" onClick={() => setMode('existing')}>Lier un élément existant</button>
          <button type="button" onClick={onDismiss}>Annuler</button>
        </>
      )}
      {(mode === 'linked-goal' || mode === 'step') && (
        <form
          onSubmit={(event) => {
            event.preventDefault();
            if (!title.trim()) return;
            onChoose(mode === 'linked-goal' ? { kind: 'linked-goal', title: title.trim() } : { kind: 'step', title: title.trim() });
          }}
        >
          <label htmlFor="branch-menu-title">{mode === 'linked-goal' ? 'Titre de l’objectif lié' : 'Titre de l’étape'}</label>
          <input id="branch-menu-title" value={title} onChange={(event) => setTitle(event.target.value)} autoFocus />
          <button type="submit">Créer</button>
        </form>
      )}
      {mode === 'existing' && (
        <>
          <label htmlFor="branch-menu-search">Rechercher un élément</label>
          <input id="branch-menu-search" value={search} onChange={(event) => setSearch(event.target.value)} autoFocus />
          {filteredOptions.map((option) => (
            <button key={option.id} type="button" onClick={() => onChoose({ kind: 'existing', nodeId: option.id })}>
              {option.title}
            </button>
          ))}
        </>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Verify build**

Run: `cd web && npm run build`
Expected: passes.

- [ ] **Step 4: Commit**

```bash
cd web && git add src/components/branch-menu.tsx src/app/globals.css
git commit -m "feat: add BranchMenu micro-interface for drop/keyboard branch creation"
```

---

### Task 6: `use-branch-drag` hook

**Files:**
- Create: `web/src/hooks/use-branch-drag.ts`
- Test: `web/src/hooks/use-branch-drag.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// web/src/hooks/use-branch-drag.test.ts
import { act, renderHook } from '@testing-library/react';
import { describe, expect, it } from 'vitest';

import { useBranchDrag } from './use-branch-drag';

describe('useBranchDrag', () => {
  it('starts idle, tracks position while dragging, and opens the menu on drop', () => {
    const { result } = renderHook(() => useBranchDrag());

    expect(result.current.state.status).toBe('idle');

    act(() => result.current.startDrag('node-1', { x: 10, y: 10 }));
    expect(result.current.state).toMatchObject({ status: 'dragging', sourceNodeId: 'node-1' });

    act(() => result.current.updateDrag({ x: 50, y: 60 }));
    expect(result.current.state).toMatchObject({ status: 'dragging', cursor: { x: 50, y: 60 } });

    act(() => result.current.endDrag());
    expect(result.current.state).toMatchObject({ status: 'menu-open', sourceNodeId: 'node-1', menuPosition: { x: 50, y: 60 } });

    act(() => result.current.reset());
    expect(result.current.state.status).toBe('idle');
  });
});
```

- [ ] **Step 2: Run to verify it fails**

Run: `cd web && npm test -- use-branch-drag`
Expected: FAIL — module not found.

- [ ] **Step 3: Implement**

```ts
// web/src/hooks/use-branch-drag.ts
'use client';

import { useCallback, useState } from 'react';

type Point = { x: number; y: number };

type BranchDragState =
  | { status: 'idle' }
  | { status: 'dragging'; sourceNodeId: string; cursor: Point }
  | { status: 'menu-open'; sourceNodeId: string; menuPosition: Point };

export function useBranchDrag() {
  const [state, setState] = useState<BranchDragState>({ status: 'idle' });

  const startDrag = useCallback((sourceNodeId: string, cursor: Point) => {
    setState({ status: 'dragging', sourceNodeId, cursor });
  }, []);

  const updateDrag = useCallback((cursor: Point) => {
    setState((current) => (current.status === 'dragging' ? { ...current, cursor } : current));
  }, []);

  const endDrag = useCallback(() => {
    setState((current) => (current.status === 'dragging'
      ? { status: 'menu-open', sourceNodeId: current.sourceNodeId, menuPosition: current.cursor }
      : current));
  }, []);

  const reset = useCallback(() => setState({ status: 'idle' }), []);

  return { state, startDrag, updateDrag, endDrag, reset };
}
```

- [ ] **Step 4: Run to verify it passes**

Run: `cd web && npm test -- use-branch-drag`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd web && git add src/hooks/use-branch-drag.ts src/hooks/use-branch-drag.test.ts
git commit -m "feat: add useBranchDrag hook for drag-to-create-branch state machine"
```

---

### Task 7: Wire hover anchors, drag, and the menu into `quest-map.tsx`

**Files:**
- Modify: `web/src/components/quest-map.tsx`
- Modify: `web/src/hooks/use-quest-map.ts`

- [ ] **Step 1: Add `createLinkedGoal`, `linkExisting`, and `reparentStep` to `use-quest-map.ts`**

In `web/src/hooks/use-quest-map.ts`, import `placeNode` from `@/lib/map/placement` and add three new callbacks alongside the existing `createStep`. Use the same `source === 'api' ? ... : local-fallback` pattern already established by `createQuest`/`createStep` in that file (read the existing implementations first — lines 46-121 — and mirror their error handling and state-update shape exactly, don't invent a different pattern). Expose the new callbacks in the hook's return object alongside `createQuest`/`createStep`:

```ts
reparentStep: async (stepId: string, newParentQuestId: string): Promise<boolean> => {
  if (source !== 'api') {
    setError('Le re-parentage nécessite l’API. Lance l’API pour utiliser cette fonctionnalité.');
    return false;
  }
  try {
    await new QuestApiClient(API_URL).updateStep(stepId, { parentStepId: newParentQuestId });
    // Re-fetch is the simplest correct approach here (re-parenting can move a step
    // across the visible tree in ways that are error-prone to patch locally).
    const refreshed = await new QuestApiClient(API_URL).loadFirstMap();
    setSpace(refreshed);
    setError(null);
    return true;
  } catch {
    setError('Le re-parentage a échoué. Réessaie dans un instant.');
    return false;
  }
},
```

(Full wiring of `createLinkedGoal`/`linkExisting` follows the same shape as `createStep` — since `linkExisting` just calls `reparentStep` under the hood and `createLinkedGoal` just calls the existing `createQuest`, those two don't need new API methods, only new callback names in the hook that delegate to what already exists. Add:)

```ts
createLinkedGoal: (title: string) => createQuest(title, 'Première étape'),
linkExisting: (nodeId: string) => reparentStep(nodeId, selectedNode?.data.questId ?? ''),
```

- [ ] **Step 2: Use `placeNode` for new steps instead of the ad-hoc positioning in `graph.ts`**

This is a judgment call left to the implementer: `graph.ts`'s `buildQuestGraph` already computes positions for the full tree on every render (including newly-added steps) via `projectPosition`/`findOpenPosition`. Rather than duplicating placement logic, leave `buildQuestGraph` as the single source of layout truth for now, and use `placeNode` (Task 3) only for the *drag preview* position shown while dragging (Step 4 below) — the final committed position still comes from `buildQuestGraph` recomputing the whole tree after the new step is added to `space`. Do not attempt to merge these two systems in this task; that's a larger refactor out of scope here. If this feels wrong once you're looking at the real code, stop and report DONE_WITH_CONCERNS rather than guessing.

- [ ] **Step 3: Add hover state and render `NodeAnchor` in `quest-map.tsx`**

In `web/src/components/quest-map.tsx`, add hover tracking:

```tsx
const [hoveredId, setHoveredId] = useState<string | null>(null);
const branchDrag = useBranchDrag();
```

Pass `onMouseEnter`/`onMouseLeave` to `QuestNode` via node data or a wrapper — since `QuestNode` is a React Flow node type receiving only `data`/`selected`/etc., the simplest correct approach is to attach the handlers on the outer node wrapper via React Flow's `onNodeMouseEnter`/`onNodeMouseLeave` props on `<ReactFlow>` itself (not per-node), which React Flow supports natively:

```tsx
<ReactFlow
  /* ...existing props... */
  onNodeMouseEnter={(_, node) => setHoveredId(node.id)}
  onNodeMouseLeave={() => setHoveredId(null)}
>
```

Then render one `NodeAnchor` for the hovered node, positioned at the point on its edge facing away from its parent, computed with `intersectRectangle` (Task 6 of Phase A) using the hovered node's own rect and its parent's center as the "target" (so the anchor sits on the side facing outward). Find the hovered node and its parent from `graph.nodes`/`graph.edges`, compute the rect the same way `quest-edge.tsx` does (reuse `OBJECTIVE_SIZE`/`STEP_SIZE` — accept the existing minor duplication flagged in Phase A's Task 7 review rather than fixing it now, that's tracked separately), and render:

```tsx
{hoveredNode && (
  <NodeAnchor
    x={anchorPoint.x}
    y={anchorPoint.y}
    variant="grow"
    label={`Ajouter une branche depuis ${hoveredNode.data.title}`}
    onActivate={() => branchDrag.startDrag(hoveredNode.id, anchorPoint)}
  />
)}
```

(The exact anchor-point math — "the side facing away from the parent" — mirrors the vector-from-parent calculation already used in `graph.ts`'s `projectPosition` for laying out children; reuse that directional logic rather than re-deriving it from scratch.)

- [ ] **Step 4: Wire pointer move/up to the drag hook and render the menu on drop**

```tsx
useEffect(() => {
  if (branchDrag.state.status !== 'dragging') return;
  const handleMove = (event: PointerEvent) => branchDrag.updateDrag({ x: event.clientX, y: event.clientY });
  const handleUp = () => branchDrag.endDrag();
  window.addEventListener('pointermove', handleMove);
  window.addEventListener('pointerup', handleUp);
  return () => {
    window.removeEventListener('pointermove', handleMove);
    window.removeEventListener('pointerup', handleUp);
  };
}, [branchDrag.state.status, branchDrag]);
```

```tsx
{branchDrag.state.status === 'menu-open' && (
  <BranchMenu
    x={branchDrag.state.menuPosition.x}
    y={branchDrag.state.menuPosition.y}
    existingOptions={graph.nodes.filter((node) => node.id !== branchDrag.state.sourceNodeId).map((node) => ({ id: node.id, title: node.data.title }))}
    onChoose={async (choice) => {
      if (choice.kind === 'step') await createStep(choice.title);
      if (choice.kind === 'linked-goal') await createLinkedGoal(choice.title);
      if (choice.kind === 'existing') await linkExisting(choice.nodeId);
      branchDrag.reset();
    }}
    onDismiss={() => branchDrag.reset()}
  />
)}
```

- [ ] **Step 5: Verify visually**

Run: `cd web && npm run dev`. Hover a card → confirm a single "+" anchor appears on its outer edge. Mousedown the anchor, drag to empty space, release → confirm the menu opens at the drop point. Choose "Ajouter une étape", type a title, submit → confirm a new step appears and the menu closes.

- [ ] **Step 6: Verify build**

Run: `cd web && npm run build`
Expected: passes.

- [ ] **Step 7: Commit**

```bash
cd web && git add src/components/quest-map.tsx src/hooks/use-quest-map.ts
git commit -m "feat: wire hover anchors, drag-to-create-branch, and micro-menu into the map"
```

---

### Task 8: Keyboard alternative on the anchor

**Files:**
- Modify: `web/src/components/quest-map.tsx`

- [ ] **Step 1: Confirm `NodeAnchor` already handles Enter/Space**

`NodeAnchor` (Task 4) is a real `<button>` with an `onKeyDown` calling `onActivate` on Enter/Space — this already satisfies "Entrée/Espace sur un ancrage focusé ouvre la même micro-interface que le drop" from the spec, since `onActivate` is wired to `branchDrag.startDrag` in Task 7. The remaining gap: a keyboard user who never hovers (so the anchor never renders) has no way to reach it. Fix that here.

- [ ] **Step 2: Make the anchor render on focus, not just hover**

In `web/src/components/quest-map.tsx`, extend the anchor's render condition from Task 7 (`hoveredId`) to also cover a `focusedId` state set by each `QuestNode`'s existing `onFocus`. Since `QuestNode` doesn't currently expose an `onFocus` prop hook, add one via the same `onNodeMouseEnter`-style approach isn't available for focus in React Flow — instead, add a native DOM `focusin` listener on the map surface ref (already available as `surface` in `quest-map.tsx`):

```tsx
useEffect(() => {
  const surfaceEl = surface.current;
  if (!surfaceEl) return;
  const handleFocusIn = (event: FocusEvent) => {
    const nodeEl = (event.target as HTMLElement).closest<HTMLElement>('.quest-node');
    const nodeId = nodeEl?.closest<HTMLElement>('[data-id]')?.dataset.id;
    if (nodeId) setHoveredId(nodeId);
  };
  surfaceEl.addEventListener('focusin', handleFocusIn);
  return () => surfaceEl.removeEventListener('focusin', handleFocusIn);
}, [surface]);
```

React Flow renders each node inside a wrapper carrying `data-id={node.id}` — reusing `hoveredId` (rather than introducing a separate `focusedId`) is intentional: the spec only ever wants a single anchor visible at a time regardless of trigger, and hover and focus are mutually exclusive in practice (a mouse user hovers, a keyboard user tabs), so one piece of state is sufficient and avoids a redundant parallel code path.

- [ ] **Step 3: Verify manually**

Run: `cd web && npm run dev`. Click on empty canvas to remove mouse focus from any node, then press Tab repeatedly until a card is focused (visible focus ring from Phase A) — confirm its anchor appears. Press Enter on the anchor once it's reachable via Tab (Tab again to move focus onto the anchor itself) — confirm the menu opens.

- [ ] **Step 4: Commit**

```bash
cd web && git add src/components/quest-map.tsx
git commit -m "feat: reveal branch anchor on keyboard focus, not only mouse hover"
```

---

### Task 9: Final validation

**Files:** none (verification only)

- [ ] **Step 1: API validation**

Run: `cd api && npm run test && npm run lint && npm run build`
Expected: all pass.

- [ ] **Step 2: Web validation**

Run: `cd web && npm run validate`
Expected: `test`, `lint`, `build` all pass.

- [ ] **Step 3: Manual end-to-end QA**

Run both `cd api && npm run start:dev` and `cd web && npm run dev`. Verify:
- Hover a card → single anchor appears on the correct outward-facing edge.
- Drag from the anchor to empty space → menu opens; each of the 3 choices works (new step persists via API, new linked goal persists, link-existing re-parents via `PATCH /steps/:id` and the map refreshes).
- Tab to a card without hovering → its anchor appears; Tab again + Enter opens the menu.
- Existing Phase A behavior (pan, zoom, selection, focus ring on cards, organic edges, starfield) still works — no regression.

- [ ] **Step 4: Commit any fixes found during QA**

```bash
git add -A && git commit -m "fix: QA adjustments for design system phase B"
```
(Only if changes were made.)

---

## Livrables documentaires

Après l'implémentation, produire :
- `docs/design-system-phase-b/fonctionnement.md` — expliquer l'anatomie de l'ancrage, le state machine de drag, l'algorithme d'anneau de `placeNode`, avec schémas ASCII.
- `docs/design-system-phase-b/guide-test.md` — recette manuelle : créer une branche par drag, créer une branche au clavier, lier un élément existant, re-parenter, vérifier l'absence de chevauchement avec plusieurs branches.
- `http/design-system-phase-b.http` — `PATCH /steps/:id` avec `parentStepId`, cas valide et cas d'erreur (autre quête, auto-référence).
