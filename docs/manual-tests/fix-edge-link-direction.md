# Recette manuelle — Fix : direction d'arête lors du "grow a branch"

## Contexte

Dragguer un lien depuis la poignée d'une Card jusqu'au canvas vide crée une
nouvelle Card liée. L'arête créée pointait dans le mauvais sens (parent →
nouvelle Card) alors que la convention du graphe est enfant → parent (voir
`api/src/nodes/nodes.service.ts#progress`) : la nouvelle Card devenait
"ancêtre" du nœud dont elle est censée dépendre, cassant le calcul de
progression.

## Prérequis

- Docker Desktop lancé ;
- dans `api/`, `docker compose up --build -d` terminé, API sur `http://localhost:3001` ;
- dans `web/`, `npm run dev`, front sur `http://localhost:3000`.

## Parcours

1. **Given** le canvas chargé avec au moins un nœud OBJECTIF existant,
   dragguer un lien depuis la poignée droite (source) de l'OBJECTIF vers un
   point vide du canvas.
   **Then** une nouvelle Card ÉTAPE apparaît au point de dépôt, reliée par une
   arête à l'OBJECTIF, et passe immédiatement en mode renommage.
2. **Given** cette nouvelle arête, appeler `GET /nodes/:objectifId/progress`
   sur l'API (ou consulter la progression affichée sur l'OBJECTIF si validée).
   **Then** la nouvelle Étape est comptée comme contributrice de l'OBJECTIF
   (elle apparaît dans le calcul de progression), pas l'inverse.
3. **When** on répète le geste depuis la poignée gauche (target) d'un nœud.
   **Then** le comportement est identique : la nouvelle Card créée est
   toujours l'enfant (edge source), le nœud d'origine reste le parent (edge
   target).
4. **When** on relie manuellement deux Cards existantes en dragguant d'une
   poignée à l'autre.
   **Then** l'arête se crée normalement dans les deux sens de glisser-déposer
   (poignée source → poignée target, ou l'inverse).

## Résultat attendu

Toute arête créée via le geste "grow a branch" respecte la convention
enfant → parent du graphe, quelle que soit la poignée utilisée pour démarrer
le glisser-déposer.
