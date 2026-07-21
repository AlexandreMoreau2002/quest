# Design — Sprint : CRUD persistant et carte d’objectifs

## Statut

Ce document définit le sprint testable du 21 juillet 2026. Il étend le vertical slice frontend avec une API persistante et remplace temporairement la limite « données locales uniquement » de la spec frontend. Les décisions de rendu de [la carte interactive](2026-07-11-carte-interactive-design.md) restent valides.

## Objectif du sprint

Permettre de tester ce soir une expérience complète : créer un objectif et ses étapes, les enregistrer dans PostgreSQL, les revoir après rechargement, puis les explorer sur la carte interactive.

## Décisions de périmètre

- Base de données : PostgreSQL local via Docker Compose, avec Prisma comme couche de schéma et migration.
- API : NestJS séparé dans le submodule `api/`.
- Frontend : Next.js dans le submodule `web/`, qui consomme l’API HTTP locale.
- Première organisation : un seul Space seedé, `Revenus`, sans authentification. L’utilisateur technique de développement est implicite ; aucun `user_id` n’est demandé par le client dans ce sprint.
- Les projets liés sont représentés par des branches d’étapes de l’objectif racine. Ils ne sont pas encore des Quest/Space séparés ni des CrossLinks.
- L’UI expose le CRUD complet pour l’objectif et les étapes : création, lecture, renommage et suppression. La suppression d’une étape supprime récursivement ses descendants.
- Le desktop reste la cible ; le mobile n’est pas une exigence du sprint.

## Parcours de test

1. Ouvrir la carte : le seed `Gagner beaucoup d’argent` apparaît avec ses projets et leurs jalons.
2. Déplacer la carte par clic-glisser sur tout fond libre, zoomer et sélectionner un nœud.
3. Lire le détail de l’étape dans le panneau à droite.
4. Créer un nouvel objectif et ses étapes depuis le panneau gauche ; il apparaît immédiatement sur la carte.
5. Renommer une étape depuis l’inspecteur ; son libellé est mis à jour sur la carte.
6. Supprimer une étape puis recharger la page : elle n’est plus présente.

## Données seedées

### Space

`Revenus`

### Objectif racine

`Gagner beaucoup d’argent`

### Branches et jalons

```text
Gagner beaucoup d’argent
├── Développer mon activité freelance
│   ├── Clarifier mes offres
│   ├── Signer 3 clients récurrents
│   └── Atteindre 5 000 € / mois
├── Lancer Cloudbreak
│   ├── Lancer l’application
│   ├── Obtenir mes premiers utilisateurs
│   ├── Obtenir mes premiers utilisateurs premium
│   ├── Atteindre 1 000 € MRR
│   └── Scale Cloudbreak
└── Démarrer l’achat-revente de voitures
    ├── Acheter un lecteur OBD
    ├── Acheter le matériel de carrosserie
    ├── Épargner 3 000 € pour le premier achat
    ├── Acheter le premier véhicule
    └── Réaliser la première revente
```

Le seed marque `Gagner beaucoup d’argent` et `Développer mon activité freelance` comme `active`; les autres étapes démarrent `locked`, sauf les prérequis matériels de l’achat-revente, qui démarrent `active` afin de rendre les états de la carte visibles.

## Modèle persistant

Le schéma du MVP conserve `Space`, `Quest` et `Step`, mais limite la première migration aux champs requis pour la carte.

```text
Space
  id, name, theme, createdAt

Quest
  id, spaceId, title, description?, status, createdAt

Step
  id, questId, parentStepId?, title, order,
  progressPercent, status, createdAt, updatedAt
```

Pour ce sprint, un nouvel objectif correspond à une `Quest`; ses étapes sont des `Step`. Les liens entre étapes utilisent `parentStepId`. `SubGoal`, `CrossLink` et `ProgressEvent` restent hors migration jusqu’à leur story dédiée.

## Contrat API

### Lecture

- `GET /spaces` — liste les Spaces disponibles.
- `GET /spaces/:spaceId/map` — retourne les Quests du Space avec leurs Steps hiérarchisés.
- `GET /quests/:questId` — retourne une Quest et ses Steps.

### CRUD objectifs

- `POST /spaces/:spaceId/quests` — crée `{ title, description?, steps: string[] }`.
- `PATCH /quests/:questId` — renomme ou modifie la description.
- `DELETE /quests/:questId` — supprime l’objectif et ses étapes.

### CRUD étapes

- `POST /quests/:questId/steps` — crée `{ title, parentStepId?, order? }`.
- `PATCH /steps/:stepId` — modifie `{ title?, status?, progressPercent?, order? }`.
- `DELETE /steps/:stepId` — supprime récursivement l’étape et ses descendants.

Chaque route retourne les ressources utiles à l’interface avec des erreurs HTTP explicites : `400` pour payload invalide, `404` pour ressource inconnue et `409` pour une position ou une relation invalide.

## Architecture

```text
Next.js web
  ├── API client typé
  ├── carte React Flow + contrôleur d’inertie
  ├── panneau création / édition / suppression
  └── cache client synchronisé après mutation
              │ HTTP
NestJS API
  ├── SpaceModule
  ├── QuestModule
  ├── StepModule
  ├── DTOs + validation
  └── PrismaService
              │
        PostgreSQL local
```

Le frontend récupère la carte au chargement. Après chaque création, modification ou suppression, il remplace le cache de carte avec la réponse de l’API ou relit `GET /spaces/:spaceId/map`; aucune mutation optimiste n’est requise ce soir.

## Comportement de carte

- Le rendu complet utilise le viewport du navigateur.
- Les nœuds et arêtes proviennent uniquement des données API ; chaque arête relie `parentStepId` à l’ID de l’enfant.
- La surface vide déclenche pan et inertie ; les éléments marqués `data-map-overlay` ne le font jamais.
- React Flow gère zoom molette, pinch et commandes de zoom ; le contrôleur d’inertie amortit la dernière vitesse de pan par facteur `0.91` jusqu’au seuil `0.18 px/frame`.
- Le clic nœud sélectionne l’étape sans démarrer de pan.

## Tests et validation

- API : tests e2e des routes de création, lecture, mise à jour, suppression récursive et données seedées.
- API : test du refus d’un parent inexistant et de champs invalides.
- Web : tests du client API, du formulaire de création, de l’édition de titre et du panneau inspecteur.
- Web : test de conversion arbre API → nœuds/arêtes.
- Manuel : créer, renommer et supprimer dans l’UI, recharger, puis vérifier la persistance et les gestes de carte.

## Definition of done ce soir

- `docker compose up -d` démarre PostgreSQL.
- Une migration et un seed reproduisent l’arbre ci-dessus.
- L’API NestJS passe ses tests et expose le CRUD décrit.
- Le web permet de manipuler les données seedées sans rechargement manuel.
- Après rechargement, la carte reflète la base de données.
- Tests, lint et builds frontend/backend passent.
