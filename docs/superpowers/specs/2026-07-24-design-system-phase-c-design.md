# Phase C — Thèmes de démonstration & variante mobile

Dépend de la Phase A (architecture de tokens `--q-*`) et de la Phase B (`placeNode` paramétrable, ancrages pour le FAB mobile).

## Thèmes de démonstration

- Deux nouvelles classes de conteneur `.qt-pirate` et `.qt-futurist`, chacune ne redéfinissant que les valeurs des tokens `--q-*` définis en Phase A — aucune nouvelle propriété CSS, aucun changement d'anatomie de composant. Sert de preuve que l'architecture de tokens tient sans toucher à la logique métier.
- `--q-font-display` varie par thème (serif pour Pirate, sans géométrique pour Futuriste), chargé via `next/font` avec sélection par variable CSS selon la classe active.
- Sélecteur de thème : 3 points dans la topbar (repris du prototype de référence), préférence stockée en `localStorage` (pas de persistance API — l'authentification utilisateur, epic 2, est encore en backlog).

## Variante mobile

- Extension de la media query `max-width: 760px` déjà amorcée dans `globals.css`.
- Panneau détail (`inspector`) devient un bottom-sheet (ouverture par le bas, fermeture par bouton ou geste de glissement).
- Panneau de création (`creation-panel`) devient un FAB "+" en bas à droite ouvrant le formulaire en overlay plein écran.
- Spread angulaire des branches réduit sous le breakpoint mobile — paramètre additionnel de `placeNode` (Phase B) : rayon/secteur réduits.
- Contrôles de zoom repositionnés pour ne jamais chevaucher le FAB.

## Hors scope Phase C

Thème utilisateur persistant côté API, thème additionnel au-delà de Pirate/Futuriste, mode clair.

## Livrables associés

- `docs/design-system-phase-c/fonctionnement.md`
- `docs/design-system-phase-c/guide-test.md`
- `http/design-system-phase-c.http` (si la préférence de thème finit par être persistée côté API — a priori non, localStorage seul)
