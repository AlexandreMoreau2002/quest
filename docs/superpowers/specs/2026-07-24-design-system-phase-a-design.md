# Phase A — Tokens sémantiques & refonte visuelle de la carte

Source visuelle : `design_handoff_quest/Quest Design System.dc.html` (Claude Design), thème "Atlas Nocturne" uniquement.

## Objectif

Remplacer les couleurs/tailles/polices codées en dur de `web/src/app/globals.css` et `web/src/components/quest-map.tsx` par un système de tokens sémantiques `--q-*`, et rapprocher l'anatomie visuelle des cartes, connexions, panneaux et fond du design de référence — sans changer les interactions existantes (pan, zoom, sélection, création via formulaire).

## Tokens

Fichier `web/src/app/tokens.css` (importé dans `globals.css`), variables scopées sur `.qt-nocturne` posée sur `.quest-shell` :

- Surface : `--q-bg`, `--q-bg-elevated`, `--q-surface`, `--q-surface-2`
- Texte : `--q-text`, `--q-text-muted`, `--q-text-subtle`
- Bordure : `--q-border`, `--q-border-strong`
- Accent/CTA : `--q-accent`, `--q-accent-contrast`, `--q-cta-bg` (gradient), `--q-cta-text`
- Statuts : `--q-done`, `--q-progress`, `--q-locked`, `--q-error`
- Connexion : `--q-conn`
- Ombres/glow : `--q-glow`, `--q-shadow-1`, `--q-shadow-2`, `--q-shadow-3`
- Fond : `--q-texture`, classe `.atlas-calm`
- Typo : `--q-font-display` (Source Serif 4, via `next/font/google`), `--q-font-body` (DM Sans, conservé), `--q-font-mono` (DM Mono, conservé)
- Espacement/rayons : `--q-radius-card`, `--q-radius-panel`, `--q-space-1` à `--q-space-6`

Aucune couleur brute ne doit rester dans `globals.css` ou `quest-map.tsx` après la migration — toute valeur visuelle dupliquée devient un token.

Valeurs de palette Atlas Nocturne (cf. handoff) :
`#071329`, `#111C41`, `#22194A`, `#294F99`, `#533096`, `#DCE7FF`, `#A99AFB`, `#6DB7E8`.

## Composant `QuestNode`

- Statut communiqué par la forme, pas seulement la couleur :
  - `done` : bordure pleine `--q-done`, icône check, eyebrow mono "ACCOMPLI".
  - `active`/`progress` : bordure pleine `--q-progress`, point pulsant (`q-pulse` keyframe, 1.8s ease-in-out infinite) + glow, eyebrow "EN COURS".
  - `locked` : bordure pointillée, pas de fond, opacité 0.78, icône cadenas, eyebrow "À VENIR", texte estompé.
  - `loading` (classe CSS prête, pas de logique async branchée) : shimmer horizontal (`q-shimmer` keyframe).
  - `error` (classe CSS prête) : bordure/texte corail `#C4665C` / `#D98A82`.
- Taille fixe : nœud Objectif 272×132px, nœud Projet/Étape 224×100px (indépendant de la profondeur).
- Police titre : `--q-font-display` (Source Serif 4).
- `:focus-visible` : anneau `--q-accent`, 2px offset, visible clavier uniquement (pas au clic souris).

## Connexions

- Nouveau composant d'edge React Flow enregistré dans `edgeTypes` (`QuestEdge`), remplaçant le type `'straight'` générique.
- Tracé bezier quadratique avec offset perpendiculaire dérivé de l'index du edge parmi ses frères (variation organique, jamais deux lignes identiques).
- Points de départ/arrivée calculés par intersection avec le rectangle du nœud (fonction pure isolée, testable), pas par le centre — utilise les tailles fixes définies ci-dessus.
- Style : trait 1.3–1.5px, couleur `--q-conn`, pointillé, `stroke-linecap: round`.

## Panneaux

`creation-panel` et `inspector` migrés vers les tokens (`--q-surface`, `--q-border`, `--q-shadow-2`). Structure existante conservée (eyebrow + titre serif + corps + CTA bas). Vérifier `box-sizing: border-box` sur les largeurs fixes.

## Fond étoilé

Réécriture de `.quest-shell::before`/`::after` selon la recette du handoff : base → 3-4 halos asymétriques (radial-gradient, opacité .2-.5) → champ d'étoiles (petits radial-gradients + quelques `<span class="q-star">` animées `q-twinkle` avec bloom) → vignette (radial-gradient transparent→sombre, opacité .5-.6). Tokenisé via `--q-texture`. Une seule variante : `.atlas-calm`.

## État vide

Si `objectives.length === 0`, afficher un message centré invitant à créer le premier objectif, dans le style du panneau de création (pas de nouveau composant lourd — état conditionnel dans `quest-map.tsx`).

## Hors scope Phase A

Ancrages hover, drag-to-create-branche, re-route de lien, placement automatique par anneau, thèmes Pirate/Futuriste, variante mobile bottom-sheet. Voir Phase B et Phase C.

## Tests

- Tests existants (`graph.test.ts`, `momentum.test.ts`, `client.test.ts`) inchangés.
- Nouveau test unitaire pour la fonction de calcul d'intersection bord-de-rectangle utilisée par `QuestEdge` (fonction pure dans `src/lib/map/`).

## Livrables associés (selon CLAUDE.md)

- `docs/design-system-phase-a/fonctionnement.md`
- `docs/design-system-phase-a/guide-test.md`
- `http/design-system-phase-a.http` (si des endpoints sont concernés — a priori aucun en Phase A, à confirmer au plan)
