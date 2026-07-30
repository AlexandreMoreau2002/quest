# Phase B — Ancrages, drag-to-create-branch, placement automatique

Dépend de la Phase A (tokens, tailles fixes de cartes, fonction d'intersection bord-de-rectangle réutilisée pour positionner les ancrages).

## Ancrages contextuels

- État `hoveredId` (déjà partiellement disponible via `onMouseEnter`/`onMouseLeave` sur les nœuds React Flow) déclenche le rendu d'un unique ancrage par nœud survolé.
- Position : côté opposé au parent, calculée depuis le vecteur dx/dy vers le nœud parent (connu via `graph.edges`).
- Style : cercle `--q-surface` opaque + anneau `--q-border-strong` (le bord de la carte ne doit jamais transparaître) ; badge "+" distinct en `--q-cta-bg` pour l'ancrage de croissance de branche, centré exactement sur le bord.
- À la sélection d'un nœud, seuls les ancrages avec un lien réellement attaché restent visibles ; les côtés inutilisés restent très estompés/pointillés. Jamais plus d'un ancrage proéminent à la fois.

## Drag-to-create-branch

- Nouveau hook `use-branch-drag.ts` : mousedown sur l'ancrage "+" → suivi de la position souris → aperçu de ligne fluide vers le curseur → mouseup sur zone vide ouvre une micro-interface positionnée au point de drop.
- Micro-interface (popover) : 3 choix — "Ajouter un objectif lié", "Ajouter une étape", "Lier un élément existant" (recherche inline parmi les nœuds existants du graphe).
- Le nouvel élément est positionné via `placeNode` (voir ci-dessous), jamais au point de drop brut — impression de "pousser une branche" à un endroit cohérent.
- États de compatibilité pendant le drag : cible valide = anneau vert magnétique + snap ; cible invalide = anneau pointillé corail ; relâcher hors zone = snap-back élastique (transition CSS retour à la position d'origine de l'ancrage).

## Placement automatique

- Fonction pure `placeNode(parent, siblingIndex, siblingCount)` dans `src/lib/map/placement.ts`.
- Règles : rayon minimum ~180px du bord du parent ; secteur angulaire ~45° réservé par branche existante ; résolution douce par glissement le long de l'anneau en cas de collision (détection AABB entre rectangles de cartes, tailles fixes de la Phase A).
- Remplace le calcul de position actuel (absent/naïf) dans `use-quest-map.ts` → `createStep`.

## Re-route d'un ancrage existant

- Sélection d'un lien : ses deux ancrages (côté parent, côté enfant) deviennent draggables individuellement.
- Drag d'un ancrage vers un nouveau nœud cible re-parente le lien : met à jour l'état local + persiste via l'API (`parentId`/ordre).
- Aperçu magnétique identique au drag-to-create (anneau vert/corail selon validité de la cible).

## Alternative clavier (obligatoire, pas de fallback optionnel)

- Tab parcourt les nœuds du plus proche au plus loin du centre (ordre dérivé du rayon calculé par `placeNode`).
- Entrée/Espace sur un ancrage focusé ouvre la même micro-interface que le drop en souris.
- Flèches directionnelles re-ciblent un ancrage en mode re-route clavier.

## API / persistance

- Vérifier au moment du plan si l'API NestJS (`api/`) expose déjà un endpoint de mise à jour du parent/ordre d'une étape. Si absent, l'ajouter fait partie du scope de cette phase (impacte `api/`, pas seulement `web/`).
- `QuestApiClient` (`web/src/lib/api/client.ts`) étendu avec les méthodes nécessaires (create linked goal, link existing, re-parent).

## Hors scope Phase B

Thèmes de démo, variante mobile complète (voir Phase C) — le comportement de drag doit néanmoins rester utilisable au doigt sur tactile a minima (pas d'optimisation mobile dédiée).

## Livrables associés

- `docs/design-system-phase-b/fonctionnement.md`
- `docs/design-system-phase-b/guide-test.md`
- `http/design-system-phase-b.http` (endpoints de re-parentage/liaison si ajoutés côté API)
