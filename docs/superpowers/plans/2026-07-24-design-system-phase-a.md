# Design System Phase A — Tokens sémantiques & refonte visuelle Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Remplacer toutes les valeurs visuelles codées en dur de la carte Quest (`web/`) par un système de tokens CSS sémantiques `--q-*`, et rapprocher l'anatomie des cartes/connexions/panneaux/fond du design "Atlas Nocturne" livré par Claude Design — sans changer les interactions existantes (pan, zoom, sélection, création via formulaire).

**Architecture:** Un fichier `tokens.css` définit les variables sémantiques scopées sur `.qt-nocturne`, appliqué au conteneur racine `.quest-shell`. `globals.css` et `quest-map.tsx` consomment ces tokens au lieu de valeurs brutes. Un nouveau composant d'edge React Flow (`QuestEdge`) remplace l'edge `'straight'` générique, en s'appuyant sur une fonction pure de calcul d'intersection bord-de-rectangle réutilisée pour le tracé.

**Tech Stack:** Next.js 16, React 19, TypeScript, `@xyflow/react` (React Flow) 12, Vitest, `next/font/google` pour Source Serif 4.

---

## Fichiers concernés

- Créer `web/src/app/tokens.css` — variables `--q-*`, classe `.qt-nocturne`.
- Modifier `web/src/app/globals.css` — migration des couleurs/tailles vers les tokens, refonte fond étoilé, keyframes `q-pulse`/`q-shimmer`/`q-twinkle`, focus ring.
- Modifier `web/src/app/layout.tsx` — chargement de la police Source Serif 4 via `next/font/google`, import de `tokens.css`.
- Créer `web/src/lib/map/edge-geometry.ts` — fonction pure `intersectRectangle` (calcul du point d'intersection bord-de-rectangle).
- Créer `web/src/lib/map/edge-geometry.test.ts` — tests unitaires de la fonction pure.
- Créer `web/src/components/quest-edge.tsx` — composant d'edge React Flow custom (bezier organique).
- Modifier `web/src/components/quest-map.tsx` — enregistrement de `QuestEdge` dans `edgeTypes`, ajout classes de statut (`loading`/`error` prêtes), état vide, structure `isObjective` pour tailles fixes.
- Modifier `web/src/lib/map/graph.ts` — tailles de cartes différenciées (objectif vs étape) pour le placement/anti-chevauchement existant.

Aucun fichier `api/` n'est concerné par cette phase.

---

### Task 1: Fondations — fichier de tokens

**Files:**
- Create: `web/src/app/tokens.css`
- Modify: `web/src/app/globals.css:1` (ajouter l'import en tête de fichier)

- [ ] **Step 1: Créer le fichier de tokens**

```css
/* web/src/app/tokens.css */
.qt-nocturne {
  /* Surface */
  --q-bg: #071329;
  --q-bg-elevated: #111c41;
  --q-surface: rgba(45, 37, 79, 0.88);
  --q-surface-2: rgba(30, 24, 57, 0.77);

  /* Texte */
  --q-text: #f8f6ff;
  --q-text-muted: #c4bddb;
  --q-text-subtle: #a79fbe;

  /* Bordure */
  --q-border: rgba(238, 232, 255, 0.14);
  --q-border-strong: rgba(228, 216, 255, 0.28);

  /* Accent / CTA */
  --q-accent: #ad96ff;
  --q-accent-contrast: #21193d;
  --q-cta-bg: linear-gradient(135deg, #d8cdfc, #b89fff);
  --q-cta-text: #21193d;

  /* Statuts */
  --q-done: #75d7bf;
  --q-progress: #c4a8ff;
  --q-locked: #a7a0bb;
  --q-error: #c4665c;
  --q-error-text: #d98a82;

  /* Connexion */
  --q-conn: #9b89ec;

  /* Ombres / glow */
  --q-glow: rgba(155, 126, 225, 0.47);
  --q-shadow-1: 0 12px 34px rgba(9, 5, 24, 0.51);
  --q-shadow-2: 0 22px 60px rgba(5, 3, 19, 0.35);
  --q-shadow-3: 0 5px 24px rgba(155, 126, 225, 0.47);

  /* Fond */
  --q-texture:
    radial-gradient(ellipse 48rem 35rem at 76% 12%, rgba(83, 101, 211, 0.25), transparent 68%),
    radial-gradient(ellipse 41rem 32rem at 16% 86%, rgba(83, 48, 150, 0.30), transparent 72%),
    radial-gradient(ellipse 38rem 25rem at 54% 54%, rgba(41, 83, 153, 0.17), transparent 72%),
    linear-gradient(135deg, #071329 0%, #111c41 43%, #22194a 100%);

  /* Typo */
  --q-font-display: var(--font-source-serif, 'Source Serif 4', serif);
  --q-font-body: 'DM Sans', Arial, sans-serif;
  --q-font-mono: 'DM Mono', monospace;

  /* Espacement / rayons */
  --q-radius-card: 15px;
  --q-radius-panel: 18px;
  --q-space-1: 4px;
  --q-space-2: 8px;
  --q-space-3: 12px;
  --q-space-4: 16px;
  --q-space-5: 22px;
  --q-space-6: 32px;
}

.atlas-calm {
  --q-star-density: 0.72;
}
```

- [ ] **Step 2: Importer le fichier dans `globals.css`**

Ajouter en toute première ligne de `web/src/app/globals.css` (avant l'import Google Fonts existant) :

```css
@import './tokens.css';
```

- [ ] **Step 3: Vérifier que le build ne casse pas**

Run: `cd web && npm run build`
Expected: build réussit (le fichier de tokens n'est pas encore consommé mais doit être syntaxiquement valide).

- [ ] **Step 4: Commit**

```bash
cd web && git add src/app/tokens.css src/app/globals.css
git commit -m "feat: add semantic q-* design tokens for Atlas Nocturne theme"
```

---

### Task 2: Police Source Serif 4 via next/font

**Files:**
- Modify: `web/src/app/layout.tsx`

- [ ] **Step 1: Lire le layout actuel pour connaître sa structure exacte**

Run: `cat web/src/app/layout.tsx`

- [ ] **Step 2: Ajouter le chargement de la police**

Dans `web/src/app/layout.tsx`, ajouter l'import et l'application de la police sur l'élément `<html>` ou `<body>` (adapter selon la structure lue à l'étape 1, en conservant le contenu existant) :

```tsx
import { Source_Serif_4 } from 'next/font/google';

const sourceSerif = Source_Serif_4({
  subsets: ['latin'],
  weight: ['600', '700'],
  variable: '--font-source-serif',
});
```

Puis ajouter `sourceSerif.variable` à la liste de classes du `<body>` (ou `<html>`), aux côtés des classes déjà présentes, sans en supprimer aucune.

- [ ] **Step 3: Vérifier le build**

Run: `cd web && npm run build`
Expected: build réussit, aucune erreur de police manquante.

- [ ] **Step 4: Commit**

```bash
cd web && git add src/app/layout.tsx
git commit -m "feat: load Source Serif 4 display font via next/font"
```

---

### Task 3: Migration des couleurs de `globals.css` vers les tokens

**Files:**
- Modify: `web/src/app/globals.css`

- [ ] **Step 1: Ajouter la classe de thème sur `.quest-shell` et migrer le fond**

Remplacer le bloc `.quest-shell { ... background: ... }` (lignes ~10-17 actuelles) par :

```css
.quest-shell {
  position: relative; width: 100vw; height: 100vh; overflow: hidden;
  background: var(--q-texture);
  color: var(--q-text);
  font-family: var(--q-font-body);
}
```

Le composant `quest-map.tsx` devra porter la classe `qt-nocturne` en plus de `quest-shell` (traité en Task 6) pour que les tokens s'appliquent — jusque-là, les valeurs par défaut définies sur `:root` serviront de repli. Ajouter donc aussi un fallback `:root` minimal en tête de `globals.css` :

```css
:root {
  --q-bg: #071329; --q-text: #f8f6ff; --q-font-body: 'DM Sans', Arial, sans-serif;
}
```

- [ ] **Step 2: Migrer topbar, brand-mark, eyebrow, connection-pill**

Remplacer les couleurs brutes de ces sélecteurs par les tokens équivalents :

```css
.brand-mark { display: grid; place-items: center; width: 40px; height: 40px; border-radius: 13px; color: var(--q-accent-contrast); font: 700 22px var(--q-font-display); background: var(--q-cta-bg); box-shadow: var(--q-shadow-3); }
.topbar h1 { margin: 1px 0 0; font: 600 19px var(--q-font-display); letter-spacing: -.02em; color: var(--q-text); }
.eyebrow { margin: 0; color: var(--q-text-muted); font: 500 10px var(--q-font-mono); letter-spacing: .13em; }
.connection-pill { margin-left: auto; padding: 8px 11px; border: 1px solid var(--q-border); border-radius: 999px; color: var(--q-text-muted); background: var(--q-surface-2); backdrop-filter: blur(12px); font-size: 11px; }
.connection-pill i { display: inline-block; width: 6px; height: 6px; margin-right: 5px; border-radius: 50%; background: #f5b464; box-shadow: 0 0 8px #f5b464; }
.connection-pill.api i { background: var(--q-done); box-shadow: 0 0 8px var(--q-done); }
```

(La couleur `#f5b464` pour l'état "non connecté" n'a pas d'équivalent sémantique dans le design — elle reste une couleur d'avertissement ponctuelle, conservée en brut.)

- [ ] **Step 3: Migrer les panneaux (`glass-panel`, formulaire, boutons)**

```css
.glass-panel { position: absolute; z-index: 5; border: 1px solid var(--q-border); border-radius: var(--q-radius-panel); background: linear-gradient(145deg, var(--q-surface), var(--q-surface-2)); box-shadow: var(--q-shadow-2); backdrop-filter: blur(18px); pointer-events: auto; box-sizing: border-box; }
.glass-panel h2 { margin: 9px 0 7px; color: var(--q-text); font: 600 25px/1.1 var(--q-font-display); letter-spacing: -.035em; }
.panel-copy { margin: 0; color: var(--q-text-muted); font-size: 13px; line-height: 1.5; }
label { color: var(--q-text-muted); font-size: 11px; font-weight: 600; }
input { width: 100%; padding: 11px 12px; outline: none; border: 1px solid var(--q-border); border-radius: 9px; color: var(--q-text); background: rgba(11,8,28,.45); transition: border-color .18s, box-shadow .18s; }
select { width: 100%; margin-top: 8px; padding: 10px 12px; outline: none; border: 1px solid var(--q-border); border-radius: 9px; color: var(--q-text); background: rgba(11,8,28,.45); }
input::placeholder { color: var(--q-text-subtle); }
input:focus { border-color: var(--q-accent); box-shadow: 0 0 0 3px rgba(157, 130, 245, 0.18); }
form button, .secondary-button { border: 0; border-radius: 9px; color: var(--q-cta-text); background: var(--q-cta-bg); font-size: 12px; font-weight: 700; transition: transform .18s, filter .18s; }
form button { padding: 11px 13px; margin-top: 4px; }
button:hover { transform: translateY(-1px); filter: brightness(1.06); }
button:disabled { cursor: wait; opacity: .65; transform: none; }
button span { float: right; font-size: 16px; line-height: 11px; }
.form-error { margin: 12px 0 0; color: var(--q-error-text); font-size: 11px; line-height: 1.45; }
.tip { margin: 20px 0 0; color: var(--q-text-subtle); font-size: 11px; line-height: 1.5; }
.tip b { color: var(--q-text-muted); }
.secondary-button { width: 100%; padding: 11px 13px; margin-top: 20px; color: var(--q-text-muted); border: 1px solid var(--q-border-strong); background: rgba(255,255,255,.06); }
```

- [ ] **Step 4: Migrer `state-badge` et `map-hint`**

```css
.state-badge { display: inline-block; margin: 12px 0 16px; padding: 5px 9px; border: 1px solid; border-radius: 999px; font: 500 10px var(--q-font-mono); }
.state-badge.done { color: var(--q-done); border-color: color-mix(in srgb, var(--q-done), transparent 66%); background: color-mix(in srgb, var(--q-done), transparent 92%); }
.state-badge.active { color: var(--q-progress); border-color: color-mix(in srgb, var(--q-progress), transparent 64%); background: color-mix(in srgb, var(--q-progress), transparent 93%); }
.state-badge.locked { color: var(--q-locked); border-color: color-mix(in srgb, var(--q-locked), transparent 76%); background: color-mix(in srgb, var(--q-locked), transparent 94%); }
.map-hint { position: absolute; z-index: 5; bottom: 28px; right: 33px; margin: 0; color: var(--q-text-subtle); font: 10px var(--q-font-mono); pointer-events: auto; }
```

- [ ] **Step 5: Vérifier visuellement en local**

Run: `cd web && npm run dev` puis ouvrir `http://localhost:3000`.
Expected: la carte se charge sans erreur console, panneaux et topbar visibles avec les mêmes couleurs qu'avant (migration 1:1, aucun changement visuel à ce stade).

- [ ] **Step 6: Commit**

```bash
cd web && git add src/app/globals.css
git commit -m "refactor: migrate panel and topbar colors to q-* tokens"
```

---

### Task 4: Anatomie des cartes — statut par la forme, tailles fixes, focus ring

**Files:**
- Modify: `web/src/app/globals.css`
- Modify: `web/src/components/quest-map.tsx:21-33` (composant `QuestNode`)

- [ ] **Step 1: Réécrire le CSS des nœuds**

Remplacer le bloc `.quest-node { ... }` jusqu'à `.status-locked { ... }` (lignes ~70-78 actuelles) par :

```css
.quest-node {
  position: relative; width: 224px; min-height: 100px; padding: 16px 18px 15px;
  border: 1.5px solid color-mix(in srgb, var(--quest-color), transparent 40%);
  border-radius: var(--q-radius-card); color: var(--q-text);
  background: linear-gradient(145deg, rgba(58,48,99,.97), rgba(34,27,67,.98));
  box-shadow: var(--q-shadow-1), 0 0 0 1px rgba(255,255,255,.03) inset;
  transition: box-shadow .2s, filter .2s;
  font-family: var(--q-font-body);
}
.quest-node.is-objective {
  width: 272px; min-height: 132px;
  border-color: color-mix(in srgb, var(--quest-color), #f4efff 30%);
  background: linear-gradient(145deg, color-mix(in srgb, var(--quest-color), #35285f 68%), rgba(28,21,62,.99));
  box-shadow: var(--q-shadow-1), 0 0 34px color-mix(in srgb, var(--quest-color), transparent 62%);
}
.quest-node:hover { filter: brightness(1.1); box-shadow: 0 18px 38px rgba(9,5,24,.63), 0 0 22px color-mix(in srgb, var(--quest-color), transparent 72%); }
.quest-node.is-selected { box-shadow: 0 0 0 2px var(--quest-color), var(--q-shadow-1), 0 0 30px color-mix(in srgb, var(--quest-color), transparent 62%); }
.quest-node:focus-visible { outline: 2px solid var(--q-accent); outline-offset: 3px; }

.node-status { display: block; margin-bottom: 12px; color: color-mix(in srgb, var(--quest-color), #e8edff 54%); font: 500 9px var(--q-font-mono); letter-spacing: .06em; text-transform: uppercase; }
.quest-node strong { display: block; font-family: var(--q-font-display); font-size: 15px; line-height: 1.25; }
.node-description { display: block; margin-top: 7px; color: var(--q-text-muted); font-size: 10px; }
.node-handle { top: 50% !important; width: 8px !important; height: 8px !important; border: 2px solid #302651 !important; background: var(--quest-color) !important; opacity: .94; }

.status-done { border-style: solid; }
.status-active { border-style: solid; }
.status-active .node-status::before { content: ''; display: inline-block; width: 6px; height: 6px; margin-right: 6px; border-radius: 50%; background: var(--q-progress); animation: q-pulse 1.8s ease-in-out infinite; vertical-align: middle; }
.status-locked { border-style: dashed !important; opacity: .78; filter: saturate(.6); }
.status-locked strong, .status-locked .node-description { color: var(--q-text-subtle); }
.status-loading { position: relative; overflow: hidden; }
.status-loading::after {
  content: ''; position: absolute; inset: 0; pointer-events: none;
  background: linear-gradient(100deg, transparent 30%, rgba(255,255,255,.08) 50%, transparent 70%);
  background-size: 200% 100%; animation: q-shimmer 1.6s linear infinite;
}
.status-error { border-color: var(--q-error) !important; }
.status-error .node-status, .status-error strong { color: var(--q-error-text); }

@keyframes q-pulse {
  0%, 100% { opacity: 1; box-shadow: 0 0 0 0 rgba(196, 168, 255, .55); }
  50% { opacity: .6; box-shadow: 0 0 0 5px rgba(196, 168, 255, 0); }
}
@keyframes q-shimmer {
  0% { background-position: -200% 0; }
  100% { background-position: 200% 0; }
}
```

- [ ] **Step 2: Rendre les nœuds focusables et appliquer la classe de statut**

Dans `web/src/components/quest-map.tsx`, modifier le composant `QuestNode` (lignes 21-33) :

```tsx
function QuestNode({ data, selected }: NodeProps<QuestFlowNode>) {
  return (
    <div
      className={`quest-node status-${data.status} ${data.isObjective ? 'is-objective' : ''} ${selected ? 'is-selected' : ''}`}
      style={{ '--quest-color': data.color } as React.CSSProperties}
      tabIndex={0}
      role="button"
      aria-label={data.title}
    >
      <Handle id="target-left" type="target" position={Position.Left} className="node-handle" />
      <Handle id="source-left" type="source" position={Position.Left} className="node-handle" />
      <span className="node-status">{data.isObjective ? 'Objectif' : data.status === 'done' ? 'Accompli' : data.status === 'active' ? 'En cours' : 'À venir'}</span>
      <strong>{data.title}</strong>
      <small className="node-description">{data.isObjective ? `Espace · ${data.questTitle}` : data.questTitle}</small>
      <Handle id="target-right" type="target" position={Position.Right} className="node-handle" />
      <Handle id="source-right" type="source" position={Position.Right} className="node-handle" />
    </div>
  );
}
```

- [ ] **Step 3: Vérifier visuellement**

Run: `cd web && npm run dev`
Expected: les nœuds "locked" ont une bordure pointillée et sont estompés ; les nœuds "active" ont un point pulsant devant l'eyebrow ; Tab fait apparaître un anneau de focus violet autour du nœud actif.

- [ ] **Step 4: Commit**

```bash
cd web && git add src/app/globals.css src/components/quest-map.tsx
git commit -m "feat: shape-by-status card anatomy, fixed sizes, keyboard focus ring"
```

---

### Task 5: Tailles de cartes différenciées dans le calcul de placement

**Files:**
- Modify: `web/src/lib/map/graph.ts:47-50`
- Test: `web/src/lib/map/graph.test.ts`

- [ ] **Step 1: Lire le test existant pour connaître les attentes actuelles**

Run: `cat web/src/lib/map/graph.test.ts`

- [ ] **Step 2: Mettre à jour les constantes de taille**

Dans `web/src/lib/map/graph.ts`, remplacer les lignes 47-50 :

```ts
const ATLAS_FOCUS: Point = { x: 760, y: 480 };
const OBJECTIVE_WIDTH = 272;
const OBJECTIVE_HEIGHT = 132;
const STEP_WIDTH = 224;
const STEP_HEIGHT = 100;
const CARD_GAP = 64;
const CARD_CLEARANCE = { x: OBJECTIVE_WIDTH + CARD_GAP, y: OBJECTIVE_HEIGHT + CARD_GAP };
```

`CARD_CLEARANCE` reste basé sur la taille du plus grand nœud (objectif) : c'est un majorant sûr qui garantit l'absence de chevauchement quel que soit le type de nœud comparé, sans avoir à ré-écrire l'algorithme `findOpenPosition`/`cardsOverlap` existant (YAGNI — ces fonctions n'ont pas besoin de connaître le type de nœud).

- [ ] **Step 3: Lancer les tests existants**

Run: `cd web && npm test -- graph.test`
Expected: PASS (les tests actuels vérifient des positions relatives, pas les constantes de taille elles-mêmes — si un test échoue en comparant une valeur littérale à `210`/`104`, l'ajuster à `272`/`132`).

- [ ] **Step 4: Commit**

```bash
cd web && git add src/lib/map/graph.ts
git commit -m "refactor: differentiate objective and step card sizes in placement clearance"
```

---

### Task 6: Fonction pure d'intersection bord-de-rectangle

**Files:**
- Create: `web/src/lib/map/edge-geometry.ts`
- Create: `web/src/lib/map/edge-geometry.test.ts`

- [ ] **Step 1: Écrire le test qui échoue**

```ts
// web/src/lib/map/edge-geometry.test.ts
import { describe, expect, it } from 'vitest';

import { intersectRectangle } from './edge-geometry';

describe('intersectRectangle', () => {
  it('returns the point on the right edge when the target is directly to the right', () => {
    const rect = { x: 0, y: 0, width: 100, height: 50 }; // centre (50, 25)
    const target = { x: 500, y: 25 };
    const point = intersectRectangle(rect, target);
    expect(point.x).toBeCloseTo(100);
    expect(point.y).toBeCloseTo(25);
  });

  it('returns the point on the bottom edge when the target is directly below', () => {
    const rect = { x: 0, y: 0, width: 100, height: 50 };
    const target = { x: 50, y: 500 };
    const point = intersectRectangle(rect, target);
    expect(point.x).toBeCloseTo(50);
    expect(point.y).toBeCloseTo(50);
  });

  it('returns the point on the correct corner-adjacent edge for a diagonal target', () => {
    const rect = { x: 0, y: 0, width: 100, height: 50 }; // centre (50, 25), half-width 50, half-height 25
    const target = { x: 200, y: 125 }; // dx=150 (>0), dy=100 (>0) from centre; dx/hw=3, dy/hh=4 -> steeper in y -> bottom edge
    const point = intersectRectangle(rect, target);
    expect(point.y).toBeCloseTo(50);
    expect(point.x).toBeGreaterThan(50);
    expect(point.x).toBeLessThan(100);
  });
});
```

- [ ] **Step 2: Lancer le test pour vérifier qu'il échoue**

Run: `cd web && npm test -- edge-geometry`
Expected: FAIL avec une erreur "Cannot find module './edge-geometry'" ou équivalente.

- [ ] **Step 3: Implémenter la fonction**

```ts
// web/src/lib/map/edge-geometry.ts
export interface Rect {
  x: number;
  y: number;
  width: number;
  height: number;
}

export interface Point {
  x: number;
  y: number;
}

/**
 * Point où le segment [centre du rectangle -> target] croise le bord du rectangle.
 */
export function intersectRectangle(rect: Rect, target: Point): Point {
  const centerX = rect.x + rect.width / 2;
  const centerY = rect.y + rect.height / 2;
  const halfWidth = rect.width / 2;
  const halfHeight = rect.height / 2;

  const dx = target.x - centerX;
  const dy = target.y - centerY;

  if (dx === 0 && dy === 0) {
    return { x: centerX, y: centerY };
  }

  const scaleX = dx !== 0 ? halfWidth / Math.abs(dx) : Infinity;
  const scaleY = dy !== 0 ? halfHeight / Math.abs(dy) : Infinity;
  const scale = Math.min(scaleX, scaleY);

  return {
    x: centerX + dx * scale,
    y: centerY + dy * scale,
  };
}
```

- [ ] **Step 4: Lancer le test pour vérifier qu'il passe**

Run: `cd web && npm test -- edge-geometry`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
cd web && git add src/lib/map/edge-geometry.ts src/lib/map/edge-geometry.test.ts
git commit -m "feat: add rectangle-edge intersection geometry for organic connections"
```

---

### Task 7: Composant d'edge custom `QuestEdge`

**Files:**
- Create: `web/src/components/quest-edge.tsx`
- Modify: `web/src/components/quest-map.tsx`

- [ ] **Step 1: Écrire le composant**

```tsx
// web/src/components/quest-edge.tsx
'use client';

import { BaseEdge, useInternalNode, type EdgeProps } from '@xyflow/react';

import { intersectRectangle, type Rect } from '@/lib/map/edge-geometry';

const OBJECTIVE_SIZE = { width: 272, height: 132 };
const STEP_SIZE = { width: 224, height: 100 };

function nodeRect(node: ReturnType<typeof useInternalNode>): Rect | null {
  if (!node) return null;
  const isObjective = Boolean((node.data as { isObjective?: boolean }).isObjective);
  const size = isObjective ? OBJECTIVE_SIZE : STEP_SIZE;
  return {
    x: node.internals.positionAbsolute.x,
    y: node.internals.positionAbsolute.y,
    width: size.width,
    height: size.height,
  };
}

function rectCenter(rect: Rect) {
  return { x: rect.x + rect.width / 2, y: rect.y + rect.height / 2 };
}

export function QuestEdge({ id, source, target, style, markerEnd }: EdgeProps) {
  const sourceNode = useInternalNode(source);
  const targetNode = useInternalNode(target);

  const sourceRect = nodeRect(sourceNode);
  const targetRect = nodeRect(targetNode);
  if (!sourceRect || !targetRect) return null;

  const sourceCenter = rectCenter(sourceRect);
  const targetCenter = rectCenter(targetRect);

  const start = intersectRectangle(sourceRect, targetCenter);
  const end = intersectRectangle(targetRect, sourceCenter);

  // Offset perpendiculaire dérivé de l'id (variation organique déterministe et stable entre rendus).
  const seed = Array.from(id).reduce((acc, char) => acc + char.charCodeAt(0), 0);
  const direction = seed % 2 === 0 ? 1 : -1;
  const magnitude = 18 + (seed % 24);

  const midX = (start.x + end.x) / 2;
  const midY = (start.y + end.y) / 2;
  const dx = end.x - start.x;
  const dy = end.y - start.y;
  const length = Math.hypot(dx, dy) || 1;
  const normalX = (-dy / length) * magnitude * direction;
  const normalY = (dx / length) * magnitude * direction;

  const controlX = midX + normalX;
  const controlY = midY + normalY;

  const path = `M ${start.x} ${start.y} Q ${controlX} ${controlY} ${end.x} ${end.y}`;

  return (
    <BaseEdge
      id={id}
      path={path}
      markerEnd={markerEnd}
      style={{ ...style, strokeLinecap: 'round' }}
    />
  );
}
```

- [ ] **Step 2: Enregistrer `QuestEdge` dans `quest-map.tsx`**

Ajouter l'import et le mapping `edgeTypes`, remplacer le type `'straight'` par `'quest'` :

```tsx
import { QuestEdge } from '@/components/quest-edge';
```

```tsx
const edgeTypes = { quest: QuestEdge };
```

Modifier la construction des edges :

```tsx
const edges: Edge[] = graph.edges.map((edge) => ({
  ...edge,
  type: 'quest',
  className: 'quest-edge',
  style: { stroke: 'var(--q-conn)', strokeWidth: 1.4, strokeDasharray: '7 9', opacity: 0.8 },
}));
```

Passer `edgeTypes={edgeTypes}` à `<ReactFlow>`.

- [ ] **Step 3: Vérifier visuellement**

Run: `cd web && npm run dev`
Expected: les lignes entre nœuds sont légèrement courbes (pas de rendu identique entre chaque paire), toujours pointillées, et touchent visuellement le bord des cartes plutôt que leur centre.

- [ ] **Step 4: Vérifier que le build passe**

Run: `cd web && npm run build`
Expected: build réussit (TypeScript strict sur `useInternalNode`/`EdgeProps`).

- [ ] **Step 5: Commit**

```bash
cd web && git add src/components/quest-edge.tsx src/components/quest-map.tsx
git commit -m "feat: replace generic straight edges with organic edge-to-edge connections"
```

---

### Task 8: Fond étoilé selon la recette du design

**Files:**
- Modify: `web/src/app/globals.css`

- [ ] **Step 1: Remplacer le bloc `.quest-shell::before`/`::after`**

Remplacer les lignes actuelles (`.quest-shell::before, .quest-shell::after { ... }` jusqu'à `.quest-shell::after { inset: 0; box-shadow: ...}`) par :

```css
.quest-shell::before, .quest-shell::after { content: ''; position: absolute; pointer-events: none; z-index: 2; }
.quest-shell::before {
  inset: 0; opacity: calc(0.9 * var(--q-star-density, 0.72)); background-repeat: no-repeat; background-size: 100% 100%;
  background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 1600 1000'%3E%3Cg fill='%23dce7ff'%3E%3Ccircle cx='71' cy='92' r='1.2'/%3E%3Ccircle cx='164' cy='273' r='.9'/%3E%3Ccircle cx='291' cy='108' r='1.6'/%3E%3Ccircle cx='380' cy='402' r='1'/%3E%3Ccircle cx='487' cy='171' r='.8'/%3E%3Ccircle cx='562' cy='710' r='1.5'/%3E%3Ccircle cx='667' cy='231' r='1.1'/%3E%3Ccircle cx='753' cy='793' r='.8'/%3E%3Ccircle cx='840' cy='117' r='1.7'/%3E%3Ccircle cx='916' cy='612' r='.9'/%3E%3Ccircle cx='1017' cy='302' r='1.3'/%3E%3Ccircle cx='1092' cy='854' r='1'/%3E%3Ccircle cx='1218' cy='202' r='.9'/%3E%3Ccircle cx='1311' cy='522' r='1.7'/%3E%3Ccircle cx='1454' cy='148' r='1.1'/%3E%3Ccircle cx='1527' cy='731' r='1.4'/%3E%3C/g%3E%3Cg fill='%23a99afb'%3E%3Ccircle cx='121' cy='671' r='1'/%3E%3Ccircle cx='236' cy='509' r='1.5'/%3E%3Ccircle cx='349' cy='845' r='.8'/%3E%3Ccircle cx='449' cy='64' r='1.2'/%3E%3Ccircle cx='624' cy='471' r='.8'/%3E%3Ccircle cx='714' cy='332' r='1.4'/%3E%3Ccircle cx='802' cy='925' r='1'/%3E%3Ccircle cx='967' cy='63' r='.9'/%3E%3Ccircle cx='1059' cy='741' r='1.5'/%3E%3Ccircle cx='1173' cy='435' r='.8'/%3E%3Ccircle cx='1398' cy='904' r='1.1'/%3E%3Ccircle cx='1504' cy='405' r='.9'/%3E%3C/g%3E%3C/svg%3E");
}
.quest-shell::after {
  inset: 0; background: radial-gradient(ellipse 70% 65% at 50% 45%, transparent 0%, transparent 45%, rgba(4,2,16,.55) 100%);
}
```

- [ ] **Step 2: Ajouter les étoiles scintillantes animées**

Ajouter à la fin de `globals.css` :

```css
.q-star { position: absolute; z-index: 3; width: 3px; height: 3px; border-radius: 50%; background: #eef3ff; box-shadow: 0 0 10px 3px rgba(220, 231, 255, .55); animation: q-twinkle 3.4s ease-in-out infinite; pointer-events: none; }
@keyframes q-twinkle {
  0%, 100% { opacity: .35; transform: scale(1); }
  50% { opacity: 1; transform: scale(1.4); }
}
```

- [ ] **Step 3: Ajouter 4 étoiles scintillantes au DOM**

Dans `web/src/components/quest-map.tsx`, à l'intérieur du `<main className="quest-shell" ...>`, avant `<ReactFlow>`, ajouter :

```tsx
<span className="q-star" style={{ top: '14%', left: '22%', animationDelay: '0s' }} />
<span className="q-star" style={{ top: '68%', left: '78%', animationDelay: '.8s' }} />
<span className="q-star" style={{ top: '32%', left: '86%', animationDelay: '1.6s' }} />
<span className="q-star" style={{ top: '82%', left: '12%', animationDelay: '2.3s' }} />
```

- [ ] **Step 4: Ajouter la classe de thème sur le conteneur**

Modifier la balise `<main>` de `quest-map.tsx` pour porter les classes de thème :

```tsx
<main className="quest-shell qt-nocturne atlas-calm" ref={surface} aria-label="Carte de quête">
```

- [ ] **Step 5: Vérifier visuellement**

Run: `cd web && npm run dev`
Expected: 4 points lumineux pulsent doucement à des positions fixes sur la carte, en plus du champ d'étoiles statique existant ; aucune régression de lisibilité des panneaux/nœuds.

- [ ] **Step 6: Commit**

```bash
cd web && git add src/app/globals.css src/components/quest-map.tsx
git commit -m "feat: rebuild starfield background per Atlas Nocturne recipe with twinkling stars"
```

---

### Task 9: État vide

**Files:**
- Modify: `web/src/components/quest-map.tsx`
- Modify: `web/src/app/globals.css`

- [ ] **Step 1: Ajouter le style de l'état vide**

Ajouter à la fin de `globals.css` :

```css
.empty-state { position: absolute; z-index: 5; top: 50%; left: 50%; transform: translate(-50%, -50%); width: 320px; padding: 26px; text-align: center; }
.empty-state h2 { margin: 10px 0 8px; }
```

- [ ] **Step 2: Ajouter le rendu conditionnel**

Dans `web/src/components/quest-map.tsx`, avant `<ReactFlow>`, ajouter :

```tsx
{objectives.length === 0 && (
  <div className="empty-state glass-panel" data-map-overlay>
    <p className="eyebrow">CARTE VIDE</p>
    <h2>Aucun objectif pour l&rsquo;instant.</h2>
    <p className="panel-copy">Crée ton premier objectif dans le panneau à gauche pour commencer à explorer.</p>
  </div>
)}
```

- [ ] **Step 3: Vérifier manuellement**

Vider temporairement `fallbackSpace.quests` dans `web/src/lib/map/fallback.ts` (localement, ne pas commit ce changement) et relancer `npm run dev` pour observer le message centré, puis restaurer le fichier.

Run: `cd web && git diff src/lib/map/fallback.ts`
Expected: aucune différence après restauration.

- [ ] **Step 4: Commit**

```bash
cd web && git add src/components/quest-map.tsx src/app/globals.css
git commit -m "feat: add empty state when no objective exists"
```

---

### Task 10: Validation finale

**Files:** aucun (vérification uniquement)

- [ ] **Step 1: Lancer la suite complète**

Run: `cd web && npm run validate`
Expected: `test`, `lint` et `build` passent tous les trois sans erreur.

- [ ] **Step 2: Vérification visuelle manuelle**

Run: `cd web && npm run dev`, ouvrir `http://localhost:3000`. Vérifier :
- Pan (glisser la carte), zoom (molette) fonctionnent comme avant.
- Sélection d'un nœud met à jour le panneau de droite.
- Création d'objectif/étape via les formulaires fonctionne toujours.
- Nœuds "locked" en pointillé/estompé, "active" avec point pulsant, tailles 272×132 (objectif) / 224×100 (étape).
- Connexions légèrement courbes, touchant le bord des cartes.
- Focus clavier (Tab) visible sur les nœuds.
- Fond avec étoiles scintillantes discrètes.

- [ ] **Step 3: Commit final si des ajustements ont été faits pendant la vérification**

```bash
cd web && git add -A
git commit -m "fix: visual QA adjustments for design system phase A"
```
(Ne committer que si des changements ont effectivement été faits à cette étape.)

---

## Livrables documentaires (obligatoires selon CLAUDE.md)

Après l'implémentation de toutes les tasks ci-dessus, produire :

- `docs/design-system-phase-a/fonctionnement.md` — expliquer le système de tokens, l'anatomie des cartes par statut, le calcul des connexions organiques, avec schémas ASCII (ex. rectangle + point d'intersection).
- `docs/design-system-phase-a/guide-test.md` — recette manuelle : prérequis (`npm run dev` sur `web/`), scénarios (créer un objectif, ajouter une étape, observer chaque statut de nœud, vérifier le focus clavier, redimensionner en mobile), cas limites (carte vide, plusieurs objectifs qui se chevauchent potentiellement), checklist finale.
- Pas de fichier `http/design-system-phase-a.http` : cette phase ne touche à aucun endpoint API (confirmé — uniquement `web/`).

Ces deux fichiers doivent être écrits par l'agent d'implémentation à la fin de la Task 10, avant la mise à jour de `TODO.md`.
