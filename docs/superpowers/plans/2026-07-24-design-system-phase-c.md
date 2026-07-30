# Design System Phase C — Thèmes de démonstration & variante mobile Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Prove the Phase A token architecture is truly themable by adding two demo themes (Pirate, Futuriste) that only redefine `--q-*` values, add a theme switcher persisted in `localStorage`, and deliver a mobile variant (bottom-sheet detail panel, FAB creation button, reduced branch spread) — without touching component anatomy or business logic.

**Architecture:** Two new CSS classes (`.qt-pirate`, `.qt-futurist`) mirror `.qt-nocturne`'s token set with different values. A `useTheme` hook manages the active theme id, persisted to `localStorage`, applied by swapping the container class. Mobile layout is handled entirely via an existing/extended `max-width: 760px` media query plus a `useMediaQuery` hook driving conditional rendering (bottom-sheet vs. side panel, FAB vs. inline form).

**Tech Stack:** Next.js/React/TypeScript, `next/font/google` for per-theme display fonts, CSS custom properties, Vitest.

---

## Fichiers concernés

- Create `web/src/app/theme-tokens.css` — `.qt-pirate`, `.qt-futurist` token sets.
- Modify `web/src/app/layout.tsx` — load additional display fonts (Pirata One or similar for Pirate, a geometric sans for Futurist).
- Create `web/src/hooks/use-theme.ts` + test — theme id state, localStorage persistence.
- Create `web/src/components/theme-switcher.tsx` — 3-dot switcher UI.
- Create `web/src/hooks/use-media-query.ts` + test — reusable breakpoint hook.
- Modify `web/src/components/quest-map.tsx` — wire theme class, mobile conditional rendering (bottom-sheet, FAB).
- Modify `web/src/app/globals.css` — bottom-sheet/FAB styles, mobile spread hook into `placeNode` callers (none currently call `placeNode` from Phase B — this phase doesn't change that; mobile spread reduction applies only if/when `placeNode` is wired, so this task documents the parameter but doesn't force a wiring that doesn't exist yet — see Task 6 for the honest scope call).

---

### Task 1: Pirate and Futurist token sets

**Files:**
- Create: `web/src/app/theme-tokens.css`
- Modify: `web/src/app/globals.css` (add one `@import`)

- [ ] **Step 1: Create the token file**

```css
/* web/src/app/theme-tokens.css */
.qt-pirate {
  --q-bg: #2b1d0f;
  --q-bg-elevated: #3a2712;
  --q-surface: rgba(74, 52, 26, .88);
  --q-surface-2: rgba(54, 38, 18, .8);
  --q-text: #f3e6c8;
  --q-text-muted: #d8c193;
  --q-text-subtle: #a98f5f;
  --q-border: rgba(230, 200, 140, .18);
  --q-border-strong: rgba(230, 200, 140, .32);
  --q-accent: #d99b3f;
  --q-accent-contrast: #2b1d0f;
  --q-cta-bg: linear-gradient(135deg, #e8b563, #c47f2a);
  --q-cta-text: #2b1d0f;
  --q-done: #7fae5e;
  --q-progress: #d99b3f;
  --q-locked: #8a765a;
  --q-error: #b5502f;
  --q-error-text: #d98a6a;
  --q-conn: #b5813c;
  --q-glow: rgba(217, 155, 63, .4);
  --q-shadow-1: 0 12px 34px rgba(20, 12, 4, .55);
  --q-shadow-2: 0 22px 60px rgba(20, 12, 4, .4);
  --q-shadow-3: 0 5px 24px rgba(217, 155, 63, .4);
  --q-texture:
    radial-gradient(ellipse 50rem 36rem at 20% 15%, rgba(180, 130, 60, .12), transparent 70%),
    radial-gradient(ellipse 40rem 30rem at 80% 85%, rgba(120, 80, 30, .16), transparent 72%),
    linear-gradient(135deg, #241708 0%, #2b1d0f 45%, #3a2712 100%);
  --q-font-display: var(--font-pirate-display, 'Pirata One', serif);
  --q-font-body: 'DM Sans', Arial, sans-serif;
  --q-font-mono: 'DM Mono', monospace;
  --q-radius-card: 10px;
  --q-radius-panel: 12px;
}

.qt-futurist {
  --q-bg: #050912;
  --q-bg-elevated: #0a1220;
  --q-surface: rgba(10, 22, 38, .88);
  --q-surface-2: rgba(6, 14, 26, .8);
  --q-text: #e4faff;
  --q-text-muted: #93d8e8;
  --q-text-subtle: #4f8496;
  --q-border: rgba(80, 220, 255, .18);
  --q-border-strong: rgba(80, 220, 255, .34);
  --q-accent: #38e0ff;
  --q-accent-contrast: #011018;
  --q-cta-bg: linear-gradient(135deg, #38e0ff, #ff3ec9);
  --q-cta-text: #011018;
  --q-done: #4fffb0;
  --q-progress: #38e0ff;
  --q-locked: #3d5866;
  --q-error: #ff4d6d;
  --q-error-text: #ff8fa3;
  --q-conn: #38e0ff;
  --q-glow: rgba(56, 224, 255, .45);
  --q-shadow-1: 0 12px 34px rgba(1, 4, 10, .6);
  --q-shadow-2: 0 22px 60px rgba(1, 4, 10, .5);
  --q-shadow-3: 0 5px 24px rgba(56, 224, 255, .45);
  --q-texture:
    radial-gradient(ellipse 46rem 34rem at 85% 10%, rgba(56, 224, 255, .12), transparent 68%),
    radial-gradient(ellipse 42rem 30rem at 10% 90%, rgba(255, 62, 201, .1), transparent 72%),
    linear-gradient(135deg, #020409 0%, #050912 45%, #0a1220 100%);
  --q-font-display: var(--font-futurist-display, 'Chakra Petch', sans-serif);
  --q-font-body: 'DM Sans', Arial, sans-serif;
  --q-font-mono: 'DM Mono', monospace;
  --q-radius-card: 4px;
  --q-radius-panel: 6px;
}
```

- [ ] **Step 2: Import in `globals.css`**

Add, right after the existing `@import './tokens.css';` line:

```css
@import './theme-tokens.css';
```

- [ ] **Step 3: Verify build**

Run: `cd web && npm run build`
Expected: passes (classes not yet applied anywhere — this task only defines them).

- [ ] **Step 4: Commit**

```bash
cd web && git add src/app/theme-tokens.css src/app/globals.css
git commit -m "feat: add Pirate and Futurist demo theme token sets"
```

---

### Task 2: Load Pirate and Futurist display fonts

**Files:**
- Modify: `web/src/app/layout.tsx`

- [ ] **Step 1: Read current `layout.tsx`**

Run: `cat web/src/app/layout.tsx`

- [ ] **Step 2: Add the two additional fonts alongside the existing `Source_Serif_4`**

```tsx
import { Chakra_Petch, Pirata_One, Source_Serif_4 } from 'next/font/google';

const sourceSerif = Source_Serif_4({ subsets: ['latin'], weight: ['600', '700'], variable: '--font-source-serif' });
const pirateDisplay = Pirata_One({ subsets: ['latin'], weight: ['400'], variable: '--font-pirate-display' });
const futuristDisplay = Chakra_Petch({ subsets: ['latin'], weight: ['600', '700'], variable: '--font-futurist-display' });
```

Add both new `.variable` classes to the same `<body>` className list alongside `sourceSerif.variable` (all three fonts load once; only the active theme's `--q-font-display` reference actually renders one of them — this is the standard `next/font` CSS-variable pattern, no per-theme conditional loading needed).

- [ ] **Step 3: Verify build**

Run: `cd web && npm run build`
Expected: passes.

- [ ] **Step 4: Commit**

```bash
cd web && git add src/app/layout.tsx
git commit -m "feat: load Pirate and Futurist display fonts via next/font"
```

---

### Task 3: `useTheme` hook

**Files:**
- Create: `web/src/hooks/use-theme.ts`
- Test: `web/src/hooks/use-theme.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// web/src/hooks/use-theme.test.ts
import { act, renderHook } from '@testing-library/react';
import { beforeEach, describe, expect, it } from 'vitest';

import { useTheme } from './use-theme';

describe('useTheme', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  it('defaults to nocturne when nothing is stored', () => {
    const { result } = renderHook(() => useTheme());
    expect(result.current.themeId).toBe('nocturne');
  });

  it('persists the chosen theme to localStorage and reflects it on next mount', () => {
    const { result } = renderHook(() => useTheme());
    act(() => result.current.setThemeId('pirate'));
    expect(result.current.themeId).toBe('pirate');
    expect(localStorage.getItem('quest-theme')).toBe('pirate');

    const { result: secondMount } = renderHook(() => useTheme());
    expect(secondMount.current.themeId).toBe('pirate');
  });

  it('ignores a corrupted stored value and falls back to nocturne', () => {
    localStorage.setItem('quest-theme', 'not-a-real-theme');
    const { result } = renderHook(() => useTheme());
    expect(result.current.themeId).toBe('nocturne');
  });
});
```

- [ ] **Step 2:** Run `cd web && npm test -- use-theme` — expect FAIL.

- [ ] **Step 3: Implement**

```ts
// web/src/hooks/use-theme.ts
'use client';

import { useCallback, useState } from 'react';

export type ThemeId = 'nocturne' | 'pirate' | 'futurist';

const STORAGE_KEY = 'quest-theme';
const VALID_THEMES: ThemeId[] = ['nocturne', 'pirate', 'futurist'];

function readStoredTheme(): ThemeId {
  if (typeof window === 'undefined') return 'nocturne';
  const stored = window.localStorage.getItem(STORAGE_KEY);
  return (VALID_THEMES as string[]).includes(stored ?? '') ? (stored as ThemeId) : 'nocturne';
}

export function useTheme() {
  const [themeId, setThemeIdState] = useState<ThemeId>(readStoredTheme);

  const setThemeId = useCallback((next: ThemeId) => {
    setThemeIdState(next);
    if (typeof window !== 'undefined') {
      window.localStorage.setItem(STORAGE_KEY, next);
    }
  }, []);

  return { themeId, setThemeId };
}

export const THEME_CLASS: Record<ThemeId, string> = {
  nocturne: 'qt-nocturne',
  pirate: 'qt-pirate',
  futurist: 'qt-futurist',
};
```

- [ ] **Step 4:** Run `cd web && npm test -- use-theme` — expect PASS (3/3).

- [ ] **Step 5: Commit**

```bash
cd web && git add src/hooks/use-theme.ts src/hooks/use-theme.test.ts
git commit -m "feat: add useTheme hook with localStorage persistence"
```

---

### Task 4: `ThemeSwitcher` component + wiring

**Files:**
- Create: `web/src/components/theme-switcher.tsx`
- Modify: `web/src/components/quest-map.tsx`
- Modify: `web/src/app/globals.css`

- [ ] **Step 1: Add switcher CSS**

Append to `web/src/app/globals.css`:

```css
.theme-switcher { display: flex; gap: 6px; margin-left: 10px; }
.theme-dot { width: 14px; height: 14px; border-radius: 50%; border: 1.5px solid var(--q-border-strong); cursor: pointer; padding: 0; }
.theme-dot[data-theme='nocturne'] { background: #22194a; }
.theme-dot[data-theme='pirate'] { background: #d99b3f; }
.theme-dot[data-theme='futurist'] { background: #38e0ff; }
.theme-dot[aria-pressed='true'] { box-shadow: 0 0 0 2px var(--q-accent); }
```

- [ ] **Step 2: Implement `ThemeSwitcher`**

```tsx
// web/src/components/theme-switcher.tsx
'use client';

import type { ThemeId } from '@/hooks/use-theme';

const THEMES: { id: ThemeId; label: string }[] = [
  { id: 'nocturne', label: 'Atlas Nocturne' },
  { id: 'pirate', label: 'Pirate' },
  { id: 'futurist', label: 'Ville futuriste' },
];

interface ThemeSwitcherProps {
  activeTheme: ThemeId;
  onSelect: (theme: ThemeId) => void;
}

export function ThemeSwitcher({ activeTheme, onSelect }: ThemeSwitcherProps) {
  return (
    <div className="theme-switcher" role="group" aria-label="Choix du thème">
      {THEMES.map((theme) => (
        <button
          key={theme.id}
          type="button"
          className="theme-dot"
          data-theme={theme.id}
          aria-label={theme.label}
          aria-pressed={activeTheme === theme.id}
          onClick={() => onSelect(theme.id)}
        />
      ))}
    </div>
  );
}
```

- [ ] **Step 3: Wire into `quest-map.tsx`**

Import `useTheme`, `THEME_CLASS` from `@/hooks/use-theme` and `ThemeSwitcher` from `@/components/theme-switcher`. In the component body:

```tsx
const { themeId, setThemeId } = useTheme();
```

Change the root `<main>` className from the hardcoded `"quest-shell qt-nocturne atlas-calm"` to:

```tsx
<main className={`quest-shell ${THEME_CLASS[themeId]} atlas-calm`} ref={surface} aria-label="Carte de quête">
```

Add `<ThemeSwitcher activeTheme={themeId} onSelect={setThemeId} />` inside the existing `<header className="topbar" data-map-overlay>`, right after the `connection-pill` span.

- [ ] **Step 4: Verify visually**

Run: `cd web && npm run dev`. Click each of the 3 theme dots — confirm the whole map (background, cards, panels) re-skins live, and the choice survives a page reload (localStorage).

- [ ] **Step 5: Verify build**

Run: `cd web && npm run build` — must pass.

- [ ] **Step 6: Commit**

```bash
cd web && git add src/components/theme-switcher.tsx src/components/quest-map.tsx src/app/globals.css
git commit -m "feat: add live theme switcher with localStorage persistence"
```

---

### Task 5: `useMediaQuery` hook

**Files:**
- Create: `web/src/hooks/use-media-query.ts`
- Test: `web/src/hooks/use-media-query.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// web/src/hooks/use-media-query.test.ts
import { act, renderHook } from '@testing-library/react';
import { afterEach, describe, expect, it, vi } from 'vitest';

import { useMediaQuery } from './use-media-query';

function mockMatchMedia(matches: boolean) {
  const listeners: Array<(event: MediaQueryListEvent) => void> = [];
  const mql = {
    matches,
    media: '',
    addEventListener: (_: string, listener: (event: MediaQueryListEvent) => void) => listeners.push(listener),
    removeEventListener: vi.fn(),
  };
  vi.stubGlobal('matchMedia', vi.fn().mockReturnValue(mql));
  return { mql, listeners };
}

describe('useMediaQuery', () => {
  afterEach(() => vi.unstubAllGlobals());

  it('returns the current match state', () => {
    mockMatchMedia(true);
    const { result } = renderHook(() => useMediaQuery('(max-width: 760px)'));
    expect(result.current).toBe(true);
  });

  it('updates when the media query change event fires', () => {
    const { listeners } = mockMatchMedia(false);
    const { result } = renderHook(() => useMediaQuery('(max-width: 760px)'));
    expect(result.current).toBe(false);

    act(() => listeners[0]?.({ matches: true } as MediaQueryListEvent));
    expect(result.current).toBe(true);
  });
});
```

- [ ] **Step 2:** Run `cd web && npm test -- use-media-query` — expect FAIL.

- [ ] **Step 3: Implement**

```ts
// web/src/hooks/use-media-query.ts
'use client';

import { useEffect, useState } from 'react';

export function useMediaQuery(query: string): boolean {
  const [matches, setMatches] = useState(() => (typeof window !== 'undefined' ? window.matchMedia(query).matches : false));

  useEffect(() => {
    const mql = window.matchMedia(query);
    setMatches(mql.matches);
    const handleChange = (event: MediaQueryListEvent) => setMatches(event.matches);
    mql.addEventListener('change', handleChange);
    return () => mql.removeEventListener('change', handleChange);
  }, [query]);

  return matches;
}
```

- [ ] **Step 4:** Run `cd web && npm test -- use-media-query` — expect PASS (2/2).

- [ ] **Step 5: Commit**

```bash
cd web && git add src/hooks/use-media-query.ts src/hooks/use-media-query.test.ts
git commit -m "feat: add useMediaQuery hook for mobile layout detection"
```

---

### Task 6: Mobile bottom-sheet inspector + FAB creation panel

**Files:**
- Modify: `web/src/components/quest-map.tsx`
- Modify: `web/src/app/globals.css`

- [ ] **Step 1: Add mobile CSS**

Append to `web/src/app/globals.css` (inside/extending the existing `@media (max-width: 760px)` block — read the current block first with `grep -n "max-width: 760px" -A 20 web/src/app/globals.css` and add these rules alongside the existing ones rather than duplicating the media query):

```css
.bottom-sheet { position: fixed; z-index: 8; left: 0; right: 0; bottom: 0; border-radius: 18px 18px 0 0; max-height: 70vh; overflow-y: auto; padding: 20px; }
.bottom-sheet-handle { width: 36px; height: 4px; margin: 0 auto 14px; border-radius: 999px; background: var(--q-border-strong); }
.fab { position: fixed; z-index: 8; right: 20px; bottom: 24px; width: 56px; height: 56px; border-radius: 50%; border: none; background: var(--q-cta-bg); color: var(--q-cta-text); font-size: 28px; line-height: 1; display: grid; place-items: center; box-shadow: var(--q-shadow-3); }
.fab-sheet { position: fixed; inset: 0; z-index: 9; background: rgba(0,0,0,.5); display: flex; align-items: flex-end; }
.fab-sheet-panel { width: 100%; }
```

- [ ] **Step 2: Read the current mobile media query block and `quest-map.tsx` panels**

Run: `grep -n "max-width: 760px" -A 20 web/src/app/globals.css` and re-read the current `creation-panel`/`inspector` JSX in `quest-map.tsx` (structure has evolved across Phase A/B — read it fresh, don't assume the Phase A snapshot is still accurate).

- [ ] **Step 3: Add mobile-conditional rendering**

In `quest-map.tsx`, add:

```tsx
const isMobile = useMediaQuery('(max-width: 760px)');
const [isCreatePanelOpen, setIsCreatePanelOpen] = useState(false);
```

Wrap the existing `creation-panel` `<aside>` so that on mobile it only renders inside a FAB-triggered overlay, and on desktop it renders inline as before:

```tsx
{isMobile ? (
  <>
    <button type="button" className="fab" aria-label="Créer un objectif" onClick={() => setIsCreatePanelOpen(true)}>+</button>
    {isCreatePanelOpen && (
      <div className="fab-sheet" onClick={() => setIsCreatePanelOpen(false)}>
        <div className="fab-sheet-panel" onClick={(event) => event.stopPropagation()}>
          <aside className="creation-panel glass-panel bottom-sheet" data-map-overlay>
            <div className="bottom-sheet-handle" />
            {/* same inner content as the existing creation-panel aside — eyebrow, h2, select, form, error, tip */}
          </aside>
        </div>
      </div>
    )}
  </>
) : (
  <aside className="creation-panel glass-panel" data-map-overlay>
    {/* existing inline content, unchanged */}
  </aside>
)}
```

Do the same conditional wrapping for the `inspector` `<aside>` (rendered when `selectedNode` is set): on mobile, render it as `<aside className="inspector glass-panel bottom-sheet" data-map-overlay>` with the `bottom-sheet-handle` div prepended; on desktop, keep the existing `inspector glass-panel` classes without `bottom-sheet`.

Since both panels' *inner content* (labels, inputs, buttons, the existing `submit`/`submitStep` handlers) stays identical between mobile and desktop — only the outer wrapper/classes/trigger differ — extract each panel's inner JSX into a small local variable or function within the component (e.g. `const creationPanelContent = (<>...</>)`) to avoid duplicating the form markup twice. Keep this extraction local to the file, not a new component, since the plan's file-structure section didn't call for one and the content is tightly coupled to this component's local state (`title`, `firstStepTitle`, `isCreatingQuest`, etc.).

Import `useMediaQuery` from `@/hooks/use-media-query`.

- [ ] **Step 4: Verify visually**

Run: `cd web && npm run dev`. Resize the browser (or use the browser tool's `resize_window` with a mobile preset) to under 760px width. Confirm: creation panel is hidden, a "+" FAB appears bottom-right; tapping it opens a bottom-sheet with the same form; selecting a node opens the inspector as a bottom-sheet instead of a fixed side panel. Resize back above 760px — confirm both panels return to their desktop inline layout.

- [ ] **Step 5: Verify build**

Run: `cd web && npm run build` — must pass.

- [ ] **Step 6: Commit**

```bash
cd web && git add src/components/quest-map.tsx src/app/globals.css
git commit -m "feat: add mobile bottom-sheet inspector and FAB creation panel"
```

---

### Task 7: Final validation

**Files:** none (verification only)

- [ ] **Step 1: Web validation**

Run: `cd web && npm run validate`
Expected: `test`, `lint`, `build` all pass.

- [ ] **Step 2: Manual end-to-end QA**

Run `cd web && npm run dev` (and `cd api && npm run start:dev` if available, to exercise the live-data path). Verify:
- All 3 themes render distinctly and correctly re-skin cards, panels, connections, and background — no anatomy/layout breakage in any theme.
- Theme choice persists across a page reload.
- Mobile viewport (< 760px): FAB + bottom-sheet creation flow works end to end (create an objective); selecting a node opens the inspector as a bottom-sheet; desktop viewport (≥ 760px) is unaffected.
- All Phase A/B behavior still works: pan, zoom, organic edges, hover/keyboard anchors, drag-to-create-branch, focus rings — across at least 2 of the 3 themes (to catch any theme-specific regression).

- [ ] **Step 3: Commit any fixes found during QA**

```bash
cd web && git add -A && git commit -m "fix: QA adjustments for design system phase C"
```
(Only if changes were made.)

---

## Livrables documentaires

- `docs/design-system-phase-c/fonctionnement.md` — expliquer le principe de thémabilité (même anatomie, tokens différents), comment le switcher fonctionne, comment la variante mobile bascule les panneaux.
- `docs/design-system-phase-c/guide-test.md` — recette manuelle : basculer entre les 3 thèmes, vérifier la persistance, tester en viewport mobile, vérifier la non-régression Phase A/B dans chaque thème.
- Pas de fichier `.http` : cette phase ne touche à aucun endpoint API.
