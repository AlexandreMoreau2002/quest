# Quest Web i18n Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Externalize every user-facing string in the Quest map UI (`web/`) into i18n keys via `react-i18next`, ship French as the only active locale, and add a real (non-cosmetic) FR/EN toggle in the topbar with EN disabled — proving the plumbing works without shipping a second language yet.

**Architecture:** A client-side `I18nProvider` wraps the app in `layout.tsx` and initializes a shared `i18next` instance from two resource files (`fr.json`, `en.json`, kept in full key-parity). A `useLocale` hook (mirroring the existing `useTheme` hook's `useSyncExternalStore` pattern) tracks the active locale, persisted to `localStorage`, and switches `i18next`'s language when changed. Components call `useTranslation()` and replace every hardcoded string with a `t('namespace.key')` call; interpolated strings (e.g. a person's card title inside a longer sentence) use `t('key', { variable })`.

**Tech Stack:** `react-i18next` + `i18next` (new dependencies), Next.js 16 App Router (client components), Vitest.

---

## Fichiers concernés

- Modify `web/package.json` — add `react-i18next`, `i18next` dependencies.
- Create `web/src/i18n/locales/fr.json` — French strings (default, complete).
- Create `web/src/i18n/locales/en.json` — English strings (full key parity, unused by the UI toggle today but ready).
- Create `web/src/i18n/config.ts` — `i18next` instance setup.
- Create `web/src/i18n/i18n-provider.tsx` — client component wiring the instance to React via `I18nextProvider`, applying the persisted locale before first paint.
- Create `web/src/i18n/locales.test.ts` — key-parity test between `fr.json`/`en.json`.
- Create `web/src/hooks/use-locale.ts` + test — persisted locale state (mirrors `use-theme.ts`).
- Create `web/src/components/language-toggle.tsx` — FR/EN pill, EN disabled.
- Modify `web/src/app/layout.tsx` — wrap `children` in `I18nProvider`.
- Modify `web/src/components/quest-map.tsx` — replace every hardcoded string with `t(...)`.
- Modify `web/src/components/branch-menu.tsx` — replace every hardcoded string with `t(...)`.
- Modify `web/src/components/theme-switcher.tsx` — replace hardcoded labels with `t(...)`, wire in `LanguageToggle`.
- Modify `web/src/hooks/use-quest-map.ts` — replace hardcoded error strings with `t(...)`.

---

### Task 1: Install dependencies and i18next configuration

**Files:**
- Modify: `web/package.json`
- Create: `web/src/i18n/locales/fr.json`
- Create: `web/src/i18n/locales/en.json`
- Create: `web/src/i18n/config.ts`

- [ ] **Step 1: Install packages**

Run: `cd web && npm install react-i18next i18next`
Expected: `package.json`/`package-lock.json` updated, no peer-dependency errors (both packages support React 19).

- [ ] **Step 2: Create the French resource file**

```json
// web/src/i18n/locales/fr.json
{
  "topbar": {
    "eyebrow": "EXPÉDITION ACTIVE",
    "apiConnected": "API connectée",
    "localMode": "Mode exploration"
  },
  "theme": {
    "groupLabel": "Choix du thème",
    "nocturne": "Atlas Nocturne",
    "pirate": "Pirate",
    "futurist": "Ville futuriste"
  },
  "language": {
    "groupLabel": "Langue",
    "fr": "FR",
    "en": "EN",
    "comingSoon": "Bientôt disponible"
  },
  "creationPanel": {
    "eyebrow": "OBJECTIF AFFICHÉ",
    "title": "Choisis un cap.",
    "description": "Explore un objectif à la fois, puis ajoute ses étapes sur la carte.",
    "objectiveLabel": "Objectif",
    "newObjectiveLabel": "Nouvel objectif",
    "newObjectivePlaceholder": "Ex. Apprendre l’italien",
    "firstStepLabel": "Premier projet ou étape",
    "firstStepPlaceholder": "Ex. Choisir une méthode",
    "submit": "Créer l’objectif",
    "submitPending": "Création…",
    "tipLabel": "Astuce",
    "tipText": "fais glisser la carte, pince pour zoomer."
  },
  "inspector": {
    "description": "Cette étape donne une direction concrète à ta progression. Choisis le prochain geste, puis avance.",
    "addStepLabel": "Ajouter une étape à « {{questTitle}} »",
    "addStepPlaceholder": "Ex. Préparer le premier test",
    "submit": "Ajouter l’étape",
    "submitPending": "Création…",
    "statusDone": "Accompli",
    "statusActive": "En cours",
    "statusLocked": "Verrouillé"
  },
  "emptyState": {
    "eyebrow": "CARTE VIDE",
    "title": "Aucun objectif pour l’instant.",
    "description": "Crée ton premier objectif dans le panneau à gauche pour commencer à explorer."
  },
  "node": {
    "objectiveEyebrow": "Objectif",
    "statusDone": "Accompli",
    "statusActive": "En cours",
    "statusLocked": "À venir",
    "objectiveDescription": "Espace · {{questTitle}}"
  },
  "anchor": {
    "addBranchLabel": "Ajouter une branche depuis {{title}}"
  },
  "branchMenu": {
    "addLinkedGoal": "Ajouter un objectif lié",
    "addStep": "Ajouter une étape",
    "linkExisting": "Lier un élément existant",
    "linkedGoalTitleLabel": "Titre de l’objectif lié",
    "stepTitleLabel": "Titre de l’étape",
    "create": "Créer",
    "searchLabel": "Rechercher un élément"
  },
  "map": {
    "hint": "Carte vivante · clique une étape pour l’explorer",
    "ariaLabel": "Carte de quête",
    "createObjectiveFab": "Créer un objectif"
  },
  "errors": {
    "questNotSaved": "La quête n’a pas été enregistrée. Vérifie que l’API est bien lancée.",
    "stepNotSaved": "L’étape n’a pas été enregistrée. Réessaie dans un instant.",
    "reparentRequiresApi": "Le re-parentage nécessite l’API. Lance l’API pour utiliser cette fonctionnalité.",
    "reparentFailed": "Le re-parentage a échoué. Réessaie dans un instant."
  }
}
```

- [ ] **Step 3: Create the English resource file (same keys, English copy)**

```json
// web/src/i18n/locales/en.json
{
  "topbar": {
    "eyebrow": "ACTIVE EXPEDITION",
    "apiConnected": "API connected",
    "localMode": "Exploration mode"
  },
  "theme": {
    "groupLabel": "Theme",
    "nocturne": "Atlas Nocturne",
    "pirate": "Pirate",
    "futurist": "Futuristic City"
  },
  "language": {
    "groupLabel": "Language",
    "fr": "FR",
    "en": "EN",
    "comingSoon": "Coming soon"
  },
  "creationPanel": {
    "eyebrow": "DISPLAYED OBJECTIVE",
    "title": "Choose a direction.",
    "description": "Explore one objective at a time, then add its steps on the map.",
    "objectiveLabel": "Objective",
    "newObjectiveLabel": "New objective",
    "newObjectivePlaceholder": "E.g. Learn Italian",
    "firstStepLabel": "First project or step",
    "firstStepPlaceholder": "E.g. Choose a method",
    "submit": "Create objective",
    "submitPending": "Creating…",
    "tipLabel": "Tip",
    "tipText": "drag the map, pinch to zoom."
  },
  "inspector": {
    "description": "This step gives concrete direction to your progress. Choose your next move, then go.",
    "addStepLabel": "Add a step to \"{{questTitle}}\"",
    "addStepPlaceholder": "E.g. Prepare the first test",
    "submit": "Add step",
    "submitPending": "Creating…",
    "statusDone": "Done",
    "statusActive": "In progress",
    "statusLocked": "Locked"
  },
  "emptyState": {
    "eyebrow": "EMPTY MAP",
    "title": "No objective yet.",
    "description": "Create your first objective in the left panel to start exploring."
  },
  "node": {
    "objectiveEyebrow": "Objective",
    "statusDone": "Done",
    "statusActive": "In progress",
    "statusLocked": "Upcoming",
    "objectiveDescription": "Space · {{questTitle}}"
  },
  "anchor": {
    "addBranchLabel": "Add a branch from {{title}}"
  },
  "branchMenu": {
    "addLinkedGoal": "Add a linked goal",
    "addStep": "Add a step",
    "linkExisting": "Link an existing item",
    "linkedGoalTitleLabel": "Linked goal title",
    "stepTitleLabel": "Step title",
    "create": "Create",
    "searchLabel": "Search an item"
  },
  "map": {
    "hint": "Living map · click a step to explore it",
    "ariaLabel": "Quest map",
    "createObjectiveFab": "Create an objective"
  },
  "errors": {
    "questNotSaved": "The quest wasn't saved. Check that the API is running.",
    "stepNotSaved": "The step wasn't saved. Try again in a moment.",
    "reparentRequiresApi": "Re-parenting requires the API. Start the API to use this feature.",
    "reparentFailed": "Re-parenting failed. Try again in a moment."
  }
}
```

- [ ] **Step 4: Create the i18next instance**

```ts
// web/src/i18n/config.ts
import i18next from 'i18next';
import { initReactI18next } from 'react-i18next';

import en from './locales/en.json';
import fr from './locales/fr.json';

export type SupportedLocale = 'fr' | 'en';
export const DEFAULT_LOCALE: SupportedLocale = 'fr';

if (!i18next.isInitialized) {
  i18next.use(initReactI18next).init({
    resources: { fr: { translation: fr }, en: { translation: en } },
    lng: DEFAULT_LOCALE,
    fallbackLng: DEFAULT_LOCALE,
    interpolation: { escapeValue: false },
  });
}

export default i18next;
```

`escapeValue: false` is correct here (not a XSS risk) because React already escapes all rendered text by default — `i18next`'s own HTML-escaping is redundant for plain string interpolation rendered via JSX text nodes, and would otherwise double-escape characters like `’`.

- [ ] **Step 5: Verify build**

Run: `cd web && npm run build`
Expected: passes (nothing consumes this module yet).

- [ ] **Step 6: Commit**

```bash
cd web && git add package.json package-lock.json src/i18n/locales/fr.json src/i18n/locales/en.json src/i18n/config.ts
git commit -m "feat: add react-i18next configuration and fr/en resource files"
```

---

### Task 2: Key-parity test between fr.json and en.json

**Files:**
- Create: `web/src/i18n/locales.test.ts`

- [ ] **Step 1: Write the test**

```ts
// web/src/i18n/locales.test.ts
import { describe, expect, it } from 'vitest';

import en from './locales/en.json';
import fr from './locales/fr.json';

function collectKeys(value: unknown, prefix = ''): string[] {
  if (typeof value !== 'object' || value === null) return [prefix];
  return Object.entries(value).flatMap(([key, nested]) => collectKeys(nested, prefix ? `${prefix}.${key}` : key));
}

describe('locale key parity', () => {
  it('has the exact same key set in fr.json and en.json', () => {
    const frKeys = collectKeys(fr).sort();
    const enKeys = collectKeys(en).sort();
    expect(enKeys).toEqual(frKeys);
  });

  it('has no empty string values in either locale', () => {
    const emptyIn = (value: unknown, path = ''): string[] => {
      if (typeof value === 'string') return value.trim() === '' ? [path] : [];
      if (typeof value !== 'object' || value === null) return [];
      return Object.entries(value).flatMap(([key, nested]) => emptyIn(nested, path ? `${path}.${key}` : key));
    };
    expect(emptyIn(fr)).toEqual([]);
    expect(emptyIn(en)).toEqual([]);
  });
});
```

- [ ] **Step 2: Run**

Run: `cd web && npm test -- locales.test`
Expected: PASS (2/2) — Task 1's two resource files already have matching keys, this test locks that invariant in for future edits.

- [ ] **Step 3: Commit**

```bash
cd web && git add src/i18n/locales.test.ts
git commit -m "test: lock fr/en locale key parity"
```

---

### Task 3: `useLocale` hook

**Files:**
- Create: `web/src/hooks/use-locale.ts`
- Test: `web/src/hooks/use-locale.test.ts`

- [ ] **Step 1: Read `web/src/hooks/use-theme.ts` first** to match its exact `useSyncExternalStore` pattern (don't invent a different one):

Run: `cat web/src/hooks/use-theme.ts`

- [ ] **Step 2: Write the failing test**

```ts
// web/src/hooks/use-locale.test.ts
import { act, renderHook } from '@testing-library/react';
import { beforeEach, describe, expect, it } from 'vitest';

import { useLocale } from './use-locale';

describe('useLocale', () => {
  beforeEach(() => {
    localStorage.clear();
  });

  it('defaults to fr when nothing is stored', () => {
    const { result } = renderHook(() => useLocale());
    expect(result.current.locale).toBe('fr');
  });

  it('persists the chosen locale to localStorage and reflects it on next mount', () => {
    const { result } = renderHook(() => useLocale());
    act(() => result.current.setLocale('en'));
    expect(result.current.locale).toBe('en');
    expect(localStorage.getItem('quest-locale')).toBe('en');

    const { result: secondMount } = renderHook(() => useLocale());
    expect(secondMount.current.locale).toBe('en');
  });

  it('ignores a corrupted stored value and falls back to fr', () => {
    localStorage.setItem('quest-locale', 'not-a-real-locale');
    const { result } = renderHook(() => useLocale());
    expect(result.current.locale).toBe('fr');
  });
});
```

- [ ] **Step 3: Run to verify it fails**

Run: `cd web && npm test -- use-locale`
Expected: FAIL — module not found.

- [ ] **Step 4: Implement, mirroring `use-theme.ts`'s `useSyncExternalStore` shape exactly**

```ts
// web/src/hooks/use-locale.ts
'use client';

import { useCallback, useSyncExternalStore } from 'react';

import i18n, { DEFAULT_LOCALE, type SupportedLocale } from '@/i18n/config';

const STORAGE_KEY = 'quest-locale';
const VALID_LOCALES: SupportedLocale[] = ['fr', 'en'];
const listeners = new Set<() => void>();

function readStoredLocale(): SupportedLocale {
  if (typeof window === 'undefined') return DEFAULT_LOCALE;
  const stored = window.localStorage.getItem(STORAGE_KEY);
  return (VALID_LOCALES as string[]).includes(stored ?? '') ? (stored as SupportedLocale) : DEFAULT_LOCALE;
}

function subscribe(onStoreChange: () => void): () => void {
  listeners.add(onStoreChange);
  return () => listeners.delete(onStoreChange);
}

export function useLocale() {
  // Server snapshot always returns the default so the server-rendered markup
  // and the first client render match; the persisted locale (if any) is read
  // from localStorage on the client via getSnapshot, avoiding a hydration
  // mismatch (same pattern as useTheme).
  const locale = useSyncExternalStore(subscribe, readStoredLocale, () => DEFAULT_LOCALE);

  const setLocale = useCallback((next: SupportedLocale) => {
    if (typeof window !== 'undefined') {
      window.localStorage.setItem(STORAGE_KEY, next);
    }
    void i18n.changeLanguage(next);
    listeners.forEach((listener) => listener());
  }, []);

  return { locale, setLocale };
}
```

- [ ] **Step 5: Run to verify it passes**

Run: `cd web && npm test -- use-locale`
Expected: PASS (3/3).

- [ ] **Step 6: Commit**

```bash
cd web && git add src/hooks/use-locale.ts src/hooks/use-locale.test.ts
git commit -m "feat: add useLocale hook with localStorage persistence"
```

---

### Task 4: `I18nProvider` and wiring into `layout.tsx`

**Files:**
- Create: `web/src/i18n/i18n-provider.tsx`
- Modify: `web/src/app/layout.tsx`

- [ ] **Step 1: Create the provider**

```tsx
// web/src/i18n/i18n-provider.tsx
'use client';

import { useEffect, type ReactNode } from 'react';
import { I18nextProvider } from 'react-i18next';

import i18n from './config';
import { useLocale } from '@/hooks/use-locale';

export function I18nProvider({ children }: { children: ReactNode }) {
  const { locale } = useLocale();

  useEffect(() => {
    if (i18n.language !== locale) void i18n.changeLanguage(locale);
  }, [locale]);

  return <I18nextProvider i18n={i18n}>{children}</I18nextProvider>;
}
```

- [ ] **Step 2: Wire it into the root layout**

Modify `web/src/app/layout.tsx` — import `I18nProvider` from `@/i18n/i18n-provider` and wrap `{children}`:

```tsx
import type { Metadata } from 'next';
import { Chakra_Petch, Pirata_One, Source_Serif_4 } from 'next/font/google';
import type { ReactNode } from 'react';

import { I18nProvider } from '@/i18n/i18n-provider';

import './globals.css';

const sourceSerif = Source_Serif_4({
  subsets: ['latin'],
  weight: ['600', '700'],
  variable: '--font-source-serif',
});

const pirateDisplay = Pirata_One({
  subsets: ['latin'],
  weight: ['400'],
  variable: '--font-pirate-display',
});

const futuristDisplay = Chakra_Petch({
  subsets: ['latin'],
  weight: ['600', '700'],
  variable: '--font-futurist-display',
});

export const metadata: Metadata = {
  title: 'Quest · Ton atlas de progression',
  description: 'Transforme tes objectifs en quêtes explorables.',
};

export default function RootLayout({ children }: Readonly<{ children: ReactNode }>) {
  return (
    <html lang="fr">
      <body
        className={`${sourceSerif.variable} ${pirateDisplay.variable} ${futuristDisplay.variable}`}
      >
        <I18nProvider>{children}</I18nProvider>
      </body>
    </html>
  );
}
```

Note: keep `<html lang="fr">` static for now — French is the only shipped locale and the `<html>` tag lives in a server component that doesn't know the client's persisted locale choice at request time; this is an acceptable, documented limitation (see Task 8's docs).

- [ ] **Step 3: Verify build**

Run: `cd web && npm run build`
Expected: passes.

- [ ] **Step 4: Commit**

```bash
cd web && git add src/i18n/i18n-provider.tsx src/app/layout.tsx
git commit -m "feat: wire I18nProvider into the root layout"
```

---

### Task 5: `LanguageToggle` component

**Files:**
- Create: `web/src/components/language-toggle.tsx`
- Modify: `web/src/app/globals.css`

- [ ] **Step 1: Add CSS**

Append to `web/src/app/globals.css`:

```css
.language-toggle { display: flex; gap: 4px; margin-left: 10px; padding: 3px; border-radius: 999px; background: var(--q-surface-2); border: 1px solid var(--q-border); }
.language-toggle button { padding: 4px 10px; border-radius: 999px; font-size: 11px; font-weight: 700; letter-spacing: .04em; background: transparent; color: var(--q-text-muted); }
.language-toggle button[aria-pressed='true'] { background: var(--q-cta-bg); color: var(--q-cta-text); }
.language-toggle button:disabled { opacity: .45; cursor: not-allowed; }
```

- [ ] **Step 2: Implement the component**

```tsx
// web/src/components/language-toggle.tsx
'use client';

import { useTranslation } from 'react-i18next';

import type { SupportedLocale } from '@/i18n/config';

interface LanguageToggleProps {
  activeLocale: SupportedLocale;
  onSelect: (locale: SupportedLocale) => void;
}

export function LanguageToggle({ activeLocale, onSelect }: LanguageToggleProps) {
  const { t } = useTranslation();

  return (
    <div className="language-toggle" role="group" aria-label={t('language.groupLabel')}>
      <button type="button" aria-pressed={activeLocale === 'fr'} onClick={() => onSelect('fr')}>
        {t('language.fr')}
      </button>
      <button type="button" aria-pressed={activeLocale === 'en'} disabled title={t('language.comingSoon')} onClick={() => {}}>
        {t('language.en')}
      </button>
    </div>
  );
}
```

The EN button stays `disabled` unconditionally — per the brief, EN is present-but-inactive to prove the toggle exists, not to actually switch language yet. Its `onClick` is a no-op (unreachable while `disabled`, kept only so the button's type checks cleanly as an interactive element).

- [ ] **Step 3: Verify build**

Run: `cd web && npm run build`
Expected: passes (not wired into `quest-map.tsx` yet — that's Task 7).

- [ ] **Step 4: Commit**

```bash
cd web && git add src/components/language-toggle.tsx src/app/globals.css
git commit -m "feat: add LanguageToggle component (FR active, EN disabled)"
```

---

### Task 6: Migrate `quest-map.tsx` to i18n keys

**Files:**
- Modify: `web/src/components/quest-map.tsx`

- [ ] **Step 1: Read the current file in full** (it has evolved across Phases A/B/C and recent bug fixes — don't work from an old snapshot):

Run: `cat web/src/components/quest-map.tsx`

- [ ] **Step 2: Add the import and hook call**

Add near the top imports:
```tsx
import { useTranslation } from 'react-i18next';
```

Inside `QuestMapInner`, near the other hooks:
```tsx
const { t } = useTranslation();
```

- [ ] **Step 3: Replace every hardcoded string with the matching key from `fr.json`/`en.json` (Task 1)**

Apply these exact replacements (match by the literal string currently in the file, not by line number, since line numbers have shifted since this plan was written):

| Current literal | Replacement |
|---|---|
| `'Objectif'` (in `QuestNode`'s status span, the `data.isObjective ? 'Objectif' : ...` ternary) | `t('node.objectiveEyebrow')` |
| `'Accompli'` (same ternary, `status === 'done'`) | `t('node.statusDone')` |
| `'En cours'` (same ternary, `status === 'active'`) | `t('node.statusActive')` |
| `'À venir'` (same ternary, else branch) | `t('node.statusLocked')` |
| `` `Espace · ${data.questTitle}` `` | `t('node.objectiveDescription', { questTitle: data.questTitle })` |
| `` `Ajouter une branche depuis ${hoveredNode.data.title}` `` | `t('anchor.addBranchLabel', { title: hoveredNode.data.title })` |
| `'CARTE VIDE'` | `t('emptyState.eyebrow')` |
| `'Aucun objectif pour l’instant.'` (currently written as `Aucun objectif pour l&rsquo;instant.` in JSX) | `t('emptyState.title')` |
| `'Crée ton premier objectif dans le panneau à gauche pour commencer à explorer.'` | `t('emptyState.description')` |
| `'EXPÉDITION ACTIVE'` | `t('topbar.eyebrow')` |
| `'API connectée'` | `t('topbar.apiConnected')` |
| `'Mode exploration'` | `t('topbar.localMode')` |
| `'Créer un objectif'` (the mobile FAB's `aria-label`) | `t('map.createObjectiveFab')` |
| `'OBJECTIF AFFICHÉ'` | `t('creationPanel.eyebrow')` |
| `'Choisis un cap.'` | `t('creationPanel.title')` |
| `'Explore un objectif à la fois, puis ajoute ses étapes sur la carte.'` | `t('creationPanel.description')` |
| `'Objectif'` (the `<label htmlFor="objective-selector">`) | `t('creationPanel.objectiveLabel')` |
| `'Nouvel objectif'` | `t('creationPanel.newObjectiveLabel')` |
| `'Ex. Apprendre l’italien'` (placeholder) | `t('creationPanel.newObjectivePlaceholder')` |
| `'Premier projet ou étape'` | `t('creationPanel.firstStepLabel')` |
| `'Ex. Choisir une méthode'` | `t('creationPanel.firstStepPlaceholder')` |
| `` isCreatingQuest ? 'Création…' : 'Créer l’objectif' `` | `` isCreatingQuest ? t('creationPanel.submitPending') : t('creationPanel.submit') `` |
| `<b>Astuce</b>` text | `<b>{t('creationPanel.tipLabel')}</b>` |
| `' — fais glisser la carte, pince pour zoomer.'` (the text right after `</b>`) | `{` — ${t('creationPanel.tipText')}`}` — keep the leading em-dash as a literal separator character, only the translatable phrase after it goes through `t()` |
| `'Carte de quête'` (the `<main>` element's `aria-label`) | `t('map.ariaLabel')` |
| `` node.data.status === 'done' ? 'Accompli' : node.data.status === 'active' ? 'En cours' : 'Verrouillé' `` (inspector status badge) | `` node.data.status === 'done' ? t('inspector.statusDone') : node.data.status === 'active' ? t('inspector.statusActive') : t('inspector.statusLocked') `` |
| `'Cette étape donne une direction concrète à ta progression. Choisis le prochain geste, puis avance.'` | `t('inspector.description')` |
| `` `Ajouter une étape à « ${node.data.questTitle} »` `` (the `<label htmlFor="step-title">` text, currently written with literal `«`/`»` around a JSX expression) | `t('inspector.addStepLabel', { questTitle: node.data.questTitle })` |
| `'Ex. Préparer le premier test'` | `t('inspector.addStepPlaceholder')` |
| `` isCreatingStep ? 'Création…' : 'Ajouter l’étape' `` | `` isCreatingStep ? t('inspector.submitPending') : t('inspector.submit') `` |
| `'Carte vivante · clique une étape pour l’explorer'` | `t('map.hint')` |

Leave `data.title`, `spaceName`, `objective.title`, and any other value coming from actual quest/step data untouched — only static UI copy goes through `t()`.

- [ ] **Step 4: Verify build**

Run: `cd web && npm run build`
Expected: passes.

- [ ] **Step 5: Verify visually**

Run: `cd web && npm run dev`. Confirm the map renders identically to before (all text still in French, since `fr` is the default locale) — this is a pure refactor, no visible change expected.

- [ ] **Step 6: Commit**

```bash
cd web && git add src/components/quest-map.tsx
git commit -m "refactor: migrate quest-map.tsx copy to i18n keys"
```

---

### Task 7: Migrate `branch-menu.tsx`, wire `LanguageToggle` into the topbar

**Files:**
- Modify: `web/src/components/branch-menu.tsx`
- Modify: `web/src/components/quest-map.tsx`

- [ ] **Step 1: Migrate `branch-menu.tsx`**

Add `import { useTranslation } from 'react-i18next';` and `const { t } = useTranslation();` inside `BranchMenu`. Replace:

| Current literal | Replacement |
|---|---|
| `'Ajouter un objectif lié'` | `t('branchMenu.addLinkedGoal')` |
| `'Ajouter une étape'` | `t('branchMenu.addStep')` |
| `'Lier un élément existant'` | `t('branchMenu.linkExisting')` |
| `` mode === 'linked-goal' ? 'Titre de l’objectif lié' : 'Titre de l’étape' `` | `` mode === 'linked-goal' ? t('branchMenu.linkedGoalTitleLabel') : t('branchMenu.stepTitleLabel') `` |
| `'Créer'` (submit button) | `t('branchMenu.create')` |
| `'Rechercher un élément'` | `t('branchMenu.searchLabel')` |

- [ ] **Step 2: Wire `LanguageToggle` into `quest-map.tsx`'s topbar**

Import `LanguageToggle` and `useLocale`:
```tsx
import { LanguageToggle } from '@/components/language-toggle';
import { useLocale } from '@/hooks/use-locale';
```

Inside `QuestMapInner`:
```tsx
const { locale, setLocale } = useLocale();
```

In the `<header className="topbar" data-map-overlay>` block, right after the existing `<ThemeSwitcher .../>`:
```tsx
<LanguageToggle activeLocale={locale} onSelect={setLocale} />
```

- [ ] **Step 3: Verify build**

Run: `cd web && npm run build`
Expected: passes.

- [ ] **Step 4: Verify visually**

Run: `cd web && npm run dev`. Confirm the FR/EN pill appears in the topbar next to the theme dots, FR highlighted as active, EN visibly disabled (dimmed, not clickable) with a "Bientôt disponible" tooltip on hover.

- [ ] **Step 5: Commit**

```bash
cd web && git add src/components/branch-menu.tsx src/components/quest-map.tsx
git commit -m "refactor: migrate branch-menu.tsx copy to i18n keys, wire LanguageToggle into topbar"
```

---

### Task 8: Migrate `use-quest-map.ts` error strings

**Files:**
- Modify: `web/src/hooks/use-quest-map.ts`

- [ ] **Step 1: Read the current file in full**

Run: `cat web/src/hooks/use-quest-map.ts`

- [ ] **Step 2: Add the translation hook and replace the 4 hardcoded error strings**

Add `import { useTranslation } from 'react-i18next';` at the top, and inside `useQuestMap()`:
```ts
const { t } = useTranslation();
```

Replace each `setError('...')` call's literal string:

| Current literal | Replacement |
|---|---|
| `'La quête n’a pas été enregistrée. Vérifie que l’API est bien lancée.'` | `t('errors.questNotSaved')` |
| `'L’étape n’a pas été enregistrée. Réessaie dans un instant.'` | `t('errors.stepNotSaved')` |
| `'Le re-parentage nécessite l’API. Lance l’API pour utiliser cette fonctionnalité.'` | `t('errors.reparentRequiresApi')` |
| `'Le re-parentage a échoué. Réessaie dans un instant.'` | `t('errors.reparentFailed')` |

`useQuestMap` is a custom hook always called from within a client component already wrapped by `I18nProvider` (Task 4), so calling `useTranslation()` inside it is valid — hooks can call other hooks.

- [ ] **Step 3: Verify build and existing tests**

Run: `cd web && npm run build`
Expected: passes. This file has no dedicated unit test today (verified by `ls web/src/hooks/use-quest-map.test.ts 2>&1` returning "No such file" before this task) — don't add one now, that's a pre-existing gap outside this plan's scope.

- [ ] **Step 4: Commit**

```bash
cd web && git add src/hooks/use-quest-map.ts
git commit -m "refactor: migrate use-quest-map.ts error strings to i18n keys"
```

---

### Task 9: Final validation, hardcoded-string regression guard, and docs

**Files:**
- Create: `web/src/i18n/no-hardcoded-strings.test.ts`
- Modify: none else (verification + docs only)

- [ ] **Step 1: Write a lightweight regression guard**

This test doesn't try to parse JSX/AST (overkill for this codebase's size) — it greps the migrated files for a short list of French words/phrases that should no longer appear as raw literals now that Task 6-8 migrated them, catching an accidental regression (e.g. someone hardcoding a new string later) without needing a full static-analysis pass.

```ts
// web/src/i18n/no-hardcoded-strings.test.ts
import { readFileSync } from 'node:fs';
import { join } from 'node:path';
import { describe, expect, it } from 'vitest';

const MIGRATED_FILES = [
  'src/components/quest-map.tsx',
  'src/components/branch-menu.tsx',
  'src/hooks/use-quest-map.ts',
];

// A handful of literal French phrases that were hardcoded before this i18n
// migration — if any of these ever appear again as a raw string in these
// files, it's a strong signal a new hardcoded copy slipped in unmigrated.
const FORBIDDEN_LITERALS = [
  'EXPÉDITION ACTIVE',
  'Créer l’objectif',
  'Ajouter l’étape',
  'CARTE VIDE',
  'Ajouter un objectif lié',
  'La quête n’a pas été enregistrée',
];

describe('no hardcoded strings regression guard', () => {
  it.each(MIGRATED_FILES)('%s does not contain any known-migrated literal string', (relativePath) => {
    const content = readFileSync(join(process.cwd(), relativePath), 'utf-8');
    for (const literal of FORBIDDEN_LITERALS) {
      expect(content).not.toContain(literal);
    }
  });
});
```

- [ ] **Step 2: Run it**

Run: `cd web && npm test -- no-hardcoded-strings`
Expected: PASS (3/3) — proves Tasks 6-8 actually replaced these strings rather than leaving a duplicate literal alongside the `t()` call.

- [ ] **Step 3: Full validation**

Run: `cd web && npm run validate`
Expected: `test`, `lint`, `build` all pass.

- [ ] **Step 4: Manual QA**

Run `cd web && npm run dev`. Verify:
- The map renders with all French copy exactly as before (topbar, panels, node statuses, empty state, branch menu, error messages if you trigger one by stopping the API) — this is a refactor, not a visual change.
- The FR/EN toggle is visible in the topbar: FR highlighted, EN visibly disabled with a native tooltip "Bientôt disponible" on hover, and clicking EN does nothing.
- Switch themes (Nocturne/Pirate/Futurist) — confirm the language toggle's own styling re-skins correctly per theme (it uses `--q-*` tokens, same as everything else).

- [ ] **Step 5: Commit any QA fixes (only if changes were made)**

```bash
cd web && git add -A && git commit -m "fix: QA adjustments for i18n migration"
```

- [ ] **Step 6: Write the deliverable docs**

Create `docs/i18n/fonctionnement.md` (repo root `/Users/alex/Desktop/dev/quest`, NOT inside `web/`) in French, explain-like-a-child style:
- Le principe : chaque texte affiché passe par une clé (ex. `t('topbar.eyebrow')`) au lieu d'être écrit en dur dans le composant. Deux fichiers (`fr.json`, `en.json`) contiennent la traduction de chaque clé.
- Le toggle FR/EN dans la barre du haut : FR actif, EN visible mais désactivé (`disabled`, infobulle "Bientôt disponible") — ça prouve que la mécanique de traduction fonctionne réellement (les deux fichiers existent et sont synchronisés), sans encore livrer une vraie expérience en anglais.
- Le test de parité (`locales.test.ts`) qui empêche `fr.json` et `en.json` de diverger dans le futur (une clé ajoutée dans l'un doit exister dans l'autre).
- Comment ajouter une nouvelle clé : ajouter la ligne dans `fr.json` ET `en.json`, puis utiliser `t('...')` dans le composant.
- Liste des fichiers impactés : `web/src/i18n/config.ts`, `web/src/i18n/i18n-provider.tsx`, `web/src/i18n/locales/fr.json`, `web/src/i18n/locales/en.json`, `web/src/hooks/use-locale.ts`, `web/src/components/language-toggle.tsx`, `web/src/components/quest-map.tsx`, `web/src/components/branch-menu.tsx`, `web/src/hooks/use-quest-map.ts`, `web/src/app/layout.tsx`.
- Limitation connue : `<html lang="fr">` reste statique (fixé côté serveur) puisque le choix de langue est une préférence stockée côté client — un changement de langue ne changera donc pas cet attribut HTML tant qu'une vraie deuxième langue n'est pas livrée.

Create `docs/i18n/guide-test.md` (repo root, French) as a manual test recipe:
- Prérequis : `cd web && npm run dev`, ouvrir `http://localhost:3000`.
- Scénarios : vérifier que tout le texte de l'interface est en français (topbar, panneaux, cartes, menu de branche, messages d'erreur), vérifier que le bouton EN est visible mais grisé/inactif avec une infobulle, vérifier que cliquer sur FR (déjà actif) ne casse rien.
- Cas limite : couper l'API (`Ctrl+C` sur `npm run start:dev`) puis tenter de créer un objectif — le message d'erreur affiché doit être le texte français attendu (`t('errors.questNotSaved')`), pas une clé brute du type `errors.questNotSaved` affichée telle quelle (ce qui indiquerait un problème de configuration i18next).
- Checklist finale cochable.

- [ ] **Step 7: Commit the docs**

```bash
git add docs/i18n/
git commit -m "docs: add fonctionnement and guide-test for i18n"
```
Run `git branch --show-current` in the root repo first — should be `feature/1-1-postgresql-prisma-seed-objectifs`, consistent with every other doc commit this session. Commit there (root repo, not a submodule).
