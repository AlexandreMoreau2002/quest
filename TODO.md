# TODO — Quest

> Carnet de bord du développement. À consulter au début de chaque session.

## En cours

- Refonte du modèle métier : Space/Quest/Step (arbres isolés) → graphe unifié Space/Node/Edge (un Node Objectif ou Étape, liens multi-parents). Voir [spec](docs/superpowers/specs/2026-07-26-space-graphe-unifie-design.md), [lexique](LEXIQUE.md) et [plan](docs/superpowers/plans/2026-07-26-space-graphe-unifie.md).
- Story 1.5 — API du graphe unifié (Node/Edge, progression, validation) — prochaine story prête.

## Prochaines priorités

1. [ ] Story 1.5 — API du graphe unifié Node/Edge.
2. [ ] Story 1.6 — Canvas Space unifié (drag & drop libre, connexions natives, HUD).
3. [ ] Epic 2 — Mise en production (déploiement 100% F2P), une fois le graphe unifié livré.
4. [ ] Epic 3 — Authentification et espaces utilisateur (les crosslinks entre projets sont déjà couverts nativement par le DAG, plus besoin de story dédiée).

## Sprint actif — Graphe unifié Space/Node/Edge

| Story | Contenu | Statut |
|---|---|---|
| 1.1 | PostgreSQL, Prisma et seed des objectifs | `done` — modèle remplacé par 1.5 |
| 1.2 | CRUD NestJS objectifs et étapes | `done` — modèle remplacé par 1.5 |
| 1.3 | Carte Next.js reliée à l’API | `done` — modèle remplacé par 1.6 |
| Design system A/B/C | Tokens, ancrages/drag-to-create, thèmes + mobile | `done` |
| i18n | react-i18next, FR par défaut, toggle FR/EN fonctionnel | `done` |
| 1.5 | API du graphe unifié (Node/Edge, progression, validation) | `ready-for-dev` |
| 1.6 | Canvas Space unifié (drag & drop, connexions, HUD) | `backlog` — dépend de 1.5 |

## Dette technique

- [ ] Définir le modèle utilisateur et l’authentification après validation de la carte MVP (epic 3).
- [ ] Vérifier qu'aucun reliquat de l'heuristique clavier vs souris de l'ancien `NodeAnchor` (event.detail === 0) ne survit après la Task 8 du plan graphe unifié (le composant est supprimé, remplacé par le drag & connect natif React Flow).

## Mergé sur develop ✅

| Story | PR | Date |
|---|---|---|
| Story 0.1 — Workflow projet et documentation | [root PR #1](https://github.com/AlexandreMoreau2002/quest/pull/1), API PR #1, web PR #1 | 2026-07-21 |
| Story 1.1/1.2/1.3 + Design system (A/B/C) + i18n | [API PR #3](https://github.com/AlexandreMoreau2002/quest-api/pull/3), [Web PR #2](https://github.com/AlexandreMoreau2002/quest-web/pull/2) | 2026-07-26 |
