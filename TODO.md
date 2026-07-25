# TODO — Quest

> Carnet de bord du développement. À consulter au début de chaque session.

## En cours

- Story 1.4 — CRUD depuis les panneaux de la carte (prochaine story prête).

## Prochaines priorités

1. [ ] Story 1.4 — CRUD depuis les panneaux de la carte.
2. [ ] Déployer le projet conformément à la spécification déjà préparée avec Claude.

## Sprint actif — CRUD et carte

| Story | Contenu | Statut |
|---|---|---|
| 1.1 | PostgreSQL, Prisma et seed des objectifs | `done` |
| 1.2 | CRUD NestJS objectifs et étapes | `done` |
| 1.3 | Carte Next.js reliée à l’API | `done` |
| Design system A/B/C | Tokens, ancrages/drag-to-create, thèmes + mobile | `done` |
| i18n | react-i18next, FR par défaut, toggle FR/EN fonctionnel | `done` |
| 1.4 | CRUD depuis les panneaux de la carte | `ready-for-dev` |

## Dette technique

- [ ] Définir le modèle utilisateur et l’authentification après validation de la carte MVP.
- [ ] Ré-étudier l'heuristique clavier vs souris de `NodeAnchor` (event.detail === 0) pour un mécanisme plus standard (pointerType) avant un rollout mobile/AT plus large.
- [ ] Ajouter la détection de cycle sur la chaîne d'ancêtres lors du re-parentage d'étape (actuellement seule l'auto-référence directe est bloquée).

## Mergé sur develop ✅

| Story | PR | Date |
|---|---|---|
| Story 0.1 — Workflow projet et documentation | [root PR #1](https://github.com/AlexandreMoreau2002/quest/pull/1), API PR #1, web PR #1 | 2026-07-21 |
| Story 1.1/1.2/1.3 + Design system (A/B/C) + i18n | [API PR #3](https://github.com/AlexandreMoreau2002/quest-api/pull/3), [Web PR #2](https://github.com/AlexandreMoreau2002/quest-web/pull/2) | 2026-07-26 |
