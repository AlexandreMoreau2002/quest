# Design — Space en graphe unifié (Nœuds Objectif/Étape)

Date : 2026-07-26
Statut : validé par l'utilisateur, en attente de plan d'implémentation

## Contexte

Le modèle actuel (`Space` → `Quest` → `Step`, cf. `api/prisma/schema.prisma`) force chaque Quest à être un arbre isolé : un Step appartient à une seule Quest, via un unique `parentStepId`. En pratique, l'utilisateur vit son objectif principal ("Gagner beaucoup d'argent") comme un flux d'étapes qui irrigue plusieurs objectifs concrets à la fois (Freelance, Cloudbreak, achat-revente). Le modèle en arbres isolés ne permet ni de partager une étape entre objectifs, ni de lier des objectifs entre eux, et ne correspond pas à l'usage réel (mind map façon Obsidian Canvas).

Ce document remplace la structure Quest/Step par un graphe unifié à un seul type de Nœud, et définit le lexique associé (voir [`LEXIQUE.md`](../../../LEXIQUE.md) à la racine du projet, tenu à jour en continu).

## Décisions structurantes

1. **Nœud unique** : `Quest` et `Step` fusionnent en une seule entité `Node`, différenciée par un champ `type` (`OBJECTIF` | `ETAPE`). Un Objectif est un Nœud comme un autre — il peut donc être ancêtre ou descendant d'un autre Objectif, sans mécanisme séparé.
2. **Graphe libre (DAG)**, pas un arbre : un Nœud peut avoir plusieurs parents et plusieurs enfants via des `Edge` (liens orientés "mène à"). Cela permet à une même Étape d'alimenter plusieurs Objectifs.
3. **Space léger et singleton** : conservé comme conteneur du graphe (porte ouverte à en avoir plusieurs un jour), mais un seul existe pour le MVP.
4. **Pas de verrouillage automatique** : les statuts `locked` disparaissent. Un Nœud (Objectif ou Étape) n'a que deux statuts : `active` et `completed`. Aucune règle de dépendance ne bloque un Nœud — l'utilisateur coche librement.
5. **Progression d'un Objectif calculée automatiquement** : `nombre d'ancêtres complétés / nombre total d'ancêtres` (recherche par remontée du graphe depuis le Nœud Objectif). Pas de pondération manuelle pour le MVP.
6. **Validation manuelle distincte de la progression** : un Objectif peut afficher 100% de progression sans être `completed`. Le passage à `completed` nécessite une action explicite de l'utilisateur (`validatedAt` renseigné).
7. **Suppression d'un Nœud = suppression de ses Edges uniquement**, jamais de cascade sur ses descendants/ascendants. Ce point modifie la règle actuelle documentée dans `api/CLAUDE.md` ("une suppression de Step supprime explicitement ses descendants dans une transaction"), qui ne tient plus dans un graphe partagé. Les Nœuds orphelins restent visibles sur le Space ; l'utilisateur les reconnecte manuellement s'il le souhaite.
8. **Position persistée** : chaque Nœud stocke `positionX`/`positionY` pour restituer sa Card au même endroit sur le Space entre deux sessions.

## Modèle de données (cible)

```prisma
enum NodeType {
  OBJECTIF
  ETAPE
}

enum NodeStatus {
  active
  completed
}

model Space {
  id        String   @id @default(uuid()) @db.Uuid
  name      String   @unique
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  nodes     Node[]
}

model Node {
  id          String     @id @default(uuid()) @db.Uuid
  spaceId     String     @db.Uuid
  type        NodeType
  title       String
  description String?
  status      NodeStatus @default(active)
  validatedAt DateTime?
  positionX   Float      @default(0)
  positionY   Float      @default(0)
  createdAt   DateTime   @default(now())
  updatedAt   DateTime   @updatedAt

  space         Space  @relation(fields: [spaceId], references: [id], onDelete: Cascade)
  outgoingEdges Edge[] @relation("EdgeSource")
  incomingEdges Edge[] @relation("EdgeTarget")

  @@index([spaceId])
  @@unique([spaceId, title])
}

model Edge {
  id           String   @id @default(uuid()) @db.Uuid
  sourceNodeId String   @db.Uuid
  targetNodeId String   @db.Uuid
  createdAt    DateTime @default(now())

  sourceNode Node @relation("EdgeSource", fields: [sourceNodeId], references: [id], onDelete: Cascade)
  targetNode Node @relation("EdgeTarget", fields: [targetNodeId], references: [id], onDelete: Cascade)

  @@unique([sourceNodeId, targetNodeId])
  @@index([sourceNodeId])
  @@index([targetNodeId])
}
```

Notes :
- `onDelete: Cascade` sur `Edge` est correct ici : supprimer un `Node` doit supprimer les `Edge` qui le référencent (règle 7), pas les `Node` de l'autre côté du lien.
- Le calcul de progression d'un Objectif (règle 5) est une lecture dérivée (remontée récursive des `Edge` par `targetNodeId`), pas une colonne stockée — à réévaluer si le graphe devient trop grand pour un calcul à la volée.

## Comportement attendu

- Créer un Nœud : choix du `type` (Objectif ou Étape) à la création, position par défaut sur le Space.
- Cocher une Étape la passe en `completed` ; ça se répercute automatiquement sur le pourcentage de progression de tous les Objectifs dont elle est ancêtre.
- Un Objectif à 100% de progression affiche cet état visuellement, mais reste au statut précédent tant que l'utilisateur ne clique pas explicitement sur "Valider".
- Supprimer un Nœud : supprime uniquement ses `Edge` entrants/sortants. Les Nœuds qui étaient reliés restent sur le Space, non connectés, et sont reconnectables manuellement.
- Aucune règle de verrouillage : tous les Nœuds sont toujours actionnables.

## Vue front (Space)

- Un seul canvas React Flow par Space (un seul Space existe pour le MVP), zoomable et déplaçable.
- Chaque Nœud est rendu comme une **Card** ; distinction visuelle Objectif vs Étape (couleur/forme, à affiner avec le design system existant du projet).
- Drag & drop libre d'une Card : la position se persiste (`positionX`/`positionY`), les Edges connectés suivent visuellement pendant le drag (comportement natif React Flow, façon Obsidian Canvas).
- Créer un Edge : glisser depuis le bord d'une Card vers une autre.
- Supprimer un Edge : sélection + suppression, ou bouton au survol du lien.
- Le **HUD** (barre d'outils, panneau de détail du Nœud sélectionné, recherche) se superpose au Space sans jamais bloquer son interaction (pan/zoom/drag restent actifs sous le HUD).

## Hors scope (pour ce design)

- Pondération manuelle de la progression (option B écartée, cf. discussion) — à réévaluer si le ratio simple s'avère insuffisant à l'usage.
- Authentification / multi-utilisateur.
- Choix visuel définitif des couleurs/formes de Card (dépend du design system, traité séparément).
- Migration des données existantes (Space/Quest/Step actuels) vers le nouveau modèle — à couvrir dans le plan d'implémentation si des données de seed doivent être conservées.

## Impact sur les règles existantes

- `api/CLAUDE.md` doit être mis à jour : la règle "suppression de Step supprime ses descendants en cascade" est remplacée par la règle 7 ci-dessus.
- Le module NestJS `quests`/`steps`/`spaces` (règle "un module par domaine") doit être reconsidéré : probablement `spaces` et `nodes` (incluant les `edges`), à trancher dans le plan d'implémentation.

## Lexique

Voir [`LEXIQUE.md`](../../../LEXIQUE.md) à la racine du projet — tenu à jour en continu, dernière stabilisation le 2026-07-26.
