# Design system — Phase B : fonctionnement

Explication façon « comme si tu avais 8 ans », pour comprendre ce que la Phase B a ajouté à la carte de quêtes.

## C'est quoi une « ancre de branche » ?

Imagine que chaque carte (objectif ou étape) sur la carte est une planète. Quand tu poses ta souris dessus, ou que tu l'atteins au clavier, un petit bouton rond avec un `+` apparaît collé sur le bord de la carte, du côté opposé à son parent (comme un petit port d'attache d'où partirait un nouveau vaisseau).

Ce bouton, c'est l'**ancre de branche** (`NodeAnchor`). Elle apparaît dans deux cas :

- **au survol** (hover) d'une carte avec la souris ;
- **au focus clavier** (Tab) sur une carte.

Elle ne s'affiche jamais pour deux cartes à la fois : une seule ancre à la fois, sur la carte actuellement survolée ou focus.

```
        ┌─────────────┐
        │   Ma carte  │●  <- ancre (bouton +)
        └─────────────┘
```

## Le flux « glisser pour créer une branche »

C'est comme tirer un fil depuis la carte pour faire naître une nouvelle carte reliée.

1. Tu survoles une carte → l'ancre apparaît sur son bord.
2. Tu cliques sur l'ancre (mousedown+mouseup) → le mode "glisser" démarre (`useBranchDrag.startDrag`).
3. Tu déplaces la souris → un point suit ton curseur.
4. Tu cliques à nouveau, là où tu veux lâcher → le petit menu (`BranchMenu`) s'ouvre à cet endroit.
5. Tu choisis une action dans le menu :
   - **Ajouter une étape** : crée une nouvelle étape, enfant de la carte d'où tu as tiré (pas de la carte sélectionnée avant !).
   - **Ajouter un objectif lié** : crée un nouvel objectif indépendant.
   - **Lier un élément existant** : rattache une carte déjà existante comme enfant de la carte source (via `PATCH /steps/:id`).
6. La nouvelle carte apparaît sur la carte, attachée au bon parent.

Schéma en séquence ASCII :

```
Utilisateur        NodeAnchor         useBranchDrag        BranchMenu        API
    |                   |                    |                  |             |
    |--survole carte--->|                    |                  |             |
    |                   |--(ancre visible)-->|                  |             |
    |--clic ancre------>|--startDrag-------->|                  |             |
    |                   |                    |--status=dragging |             |
    |--déplace souris-->|                    |--updateDrag------|             |
    |--clic ailleurs--->|                    |--endDrag--------->|            |
    |                   |                    |--status=menu-open->(ouvre menu)|
    |--choisit "Étape"->|                    |                  |--createStep(sourceNodeId)->|
    |                   |                    |                  |             |--POST /quests/:id/steps
    |                   |                    |                  |             |  {parentStepId: sourceNodeId}
    |<---------------- nouvelle carte affichée, reliée au bon parent ---------|
```

**Point important corrigé pendant la QA finale** : la nouvelle étape est bien attachée à la carte *d'où tu as tiré le fil* (le `sourceNodeId` du drag), même si une autre carte était sélectionnée avant. Le bug initial (Task 7) faisait que `createStep` regardait toujours la carte *sélectionnée en mémoire* plutôt que la carte de départ du glisser — corrigé en passant explicitement `sourceNodeId` à `createStep`.

## L'algorithme de placement en anneau (`placeNode`)

Pas encore branché dans l'interface, mais prêt pour plus tard. Son idée : quand une carte a plusieurs enfants, on ne les empile pas n'importe comment — on leur donne chacun un petit secteur (comme une part de gâteau) autour du parent, sur un anneau (un cercle invisible à une certaine distance), puis on les pousse un peu le long de l'anneau pour éviter qu'ils se chevauchent.

```
                  enfant 2
                     •
                  ⟋     ⟍
     enfant 1  •         •  enfant 3
                  ⟍     ⟋
                     •
                  (parent)
```

Chaque enfant reçoit un secteur angulaire (`360° / nombre d'enfants`), puis une petite "nudge" (ajustement) le long de l'anneau pour garder un espacement propre même si le nombre d'enfants change. Aujourd'hui, la disposition réelle des cartes vient encore de l'arbre calculé par `buildQuestGraph` (Phase A) ; `placeNode` est une brique testée et prête, en attente d'un futur brancement dans le rendu.

## Le re-parentage : « Lier un élément existant »

Quand tu choisis **Lier un élément existant** dans le menu, tu cherches une carte déjà présente sur la carte et tu cliques dessus. Ça appelle `linkExisting(existingNodeId, sourceNodeId)`, qui appelle `reparentStep(stepId, newParentStepId)`, qui envoie :

```
PATCH /steps/:stepId
{ "parentStepId": "<newParentStepId>" }
```

Côté API (`api/src/steps/steps.service.ts`), avant d'accepter le changement, le service vérifie :

- que la nouvelle carte parent existe et appartient **à la même quête** que l'étape qu'on déplace (sinon `409 Conflict` : « The parent step must belong to the same quest. ») ;
- qu'une étape ne devienne pas son propre parent (sinon `409 Conflict` : « A step cannot be its own parent. ») ;
- qu'on ne crée pas de cycle (une étape ne peut pas devenir le parent de l'un de ses propres ancêtres).

**Pourquoi interdire le re-parentage entre quêtes ?** Parce que chaque étape appartient à une seule quête (`questId` fixe) : une quête, c'est un peu comme un dossier — on peut réorganiser les pages à l'intérieur du dossier, mais on ne peut pas faire glisser une page d'un dossier dans un autre sans casser la logique de rangement (`buildQuestGraph` regroupe les cartes par quête).

Enfin, si tu essaies de lier une étape sous un **objectif** (pas une vraie étape), ça échouera aussi : l'API attend un `parentStepId` qui pointe vers une vraie ligne `Step`, pas vers un objectif/quête.

## Fichiers impactés

- `api/src/steps/dto/update-step.dto.ts` — DTO de validation du `PATCH /steps/:id` (accepte `parentStepId`).
- `api/src/steps/steps.service.ts` — logique de re-parentage et ses gardes-fous (même quête, pas de cycle, pas d'auto-parent).
- `web/src/lib/api/client.ts` — `QuestApiClient.updateStep` (PATCH) et `createStep` (POST, accepte maintenant `parentStepId`).
- `web/src/lib/map/placement.ts` — algorithme `placeNode` (anneau + nudge), pas encore branché.
- `web/src/components/node-anchor.tsx` — le bouton d'ancre (`+`), avec gestion distincte clic souris / activation clavier.
- `web/src/components/branch-menu.tsx` — le petit menu contextuel (étape / objectif lié / élément existant).
- `web/src/hooks/use-branch-drag.ts` — machine à états du glisser (`idle` → `dragging` → `menu-open`).
- `web/src/components/quest-map.tsx` — orchestration : hover/focus, calcul de la position d'ancre en coordonnées écran, drag, ouverture du menu.
- `web/src/hooks/use-quest-map.ts` — `createStep`, `reparentStep`, `linkExisting`, avec la correction du parent explicite.

## Limites connues

- **Clavier — Shift+Tab, pas Tab** : à cause de l'ordre du DOM (l'ancre est rendue avant la carte React Flow dans le HTML), atteindre l'ancre au clavier fonctionne de façon fiable en **Shift+Tab** (en arrière) depuis un élément qui suit, plutôt qu'en Tab classique vers l'avant.
- **`placeNode` pas branché** : le placement automatique des cartes utilise toujours l'arbre de `buildQuestGraph` (Phase A). L'algorithme en anneau est testé et prêt, mais son intégration dans le rendu réel est un choix de scope explicitement reporté.
- **Re-parenter sous un objectif** : l'API refuse un `parentStepId` qui ne correspond pas à une vraie étape (`Step`) ; on ne peut pas rattacher une étape directement à un objectif via ce mécanisme.
- **Correction QA additionnelle** : pendant la validation finale, deux bugs ont été trouvés et corrigés : (1) l'ancre utilisait des coordonnées de graphe brutes au lieu de coordonnées écran (elle apparaissait au mauvais endroit dès que la carte était pan/zoom) ; (2) l'activation clavier de l'ancre restait bloquée en état « glissement » faute d'un `pointerup` pour le terminer — elle ouvre maintenant le menu directement.
