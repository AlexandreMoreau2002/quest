# CLAUDE.md — Quest

## Suivi de chantier

- Lire `TODO.md` au début de chaque session et le mettre à jour après chaque story.
- `_bmad-output/implementation-artifacts/sprint-status.yaml` est la source de vérité des epics et stories.
- Toute story terminée fournit `docs/manual-tests/story-<id>-<slug>.md` : prérequis, commandes, parcours Given/When/Then et résultat attendu.

## Projet

Quest est un SaaS de progression personnelle : les objectifs deviennent des quêtes et les étapes un monde interactif à explorer.

Stack MVP :

- `web/` — Next.js, React, TypeScript, React Flow.
- `api/` — NestJS, Prisma, PostgreSQL.
- `docs/` — specs, plans, décisions et recettes manuelles.

## Structure

```text
quest/
├── api/                         # submodule GitHub quest-api
├── web/                         # submodule GitHub quest-web
├── docs/
│   ├── manual-tests/             # une recette par story livrée
│   └── superpowers/              # specs et plans
├── rules/                        # règles opérationnelles de projet
├── _bmad-output/
│   └── implementation-artifacts/ # stories et sprint-status.yaml
├── TODO.md
└── CLAUDE.md
```

## Git Flow obligatoire

- `main` est protégée et ne reçoit que les README initiaux des services, explicitement autorisés par le propriétaire.
- `develop` est la branche d’intégration quotidienne dans chaque submodule.
- Chaque story part de `develop` sur `feature/<id>-<slug>`.
- Chaque story passe par une PR GitHub ciblant `develop`; ne pas merger sans validation humaine de la recette manuelle.
- La PR est mergée en squash. Ensuite, mettre à jour le pointeur du submodule dans le repo racine et le committer sur une branche dédiée avant sa propre PR vers `develop`.
- Aucun force-push, reset destructif ou commit direct sur `develop` sans instruction explicite.

Lire avant de commencer :

- `rules/git-flow.md`
- `rules/story-delivery.md`
- `rules/quality.md`
- le `CLAUDE.md` du service concerné.

## Process obligatoire par story

1. Lire la story, les critères d’acceptation et la spec associée.
2. Créer la branche feature depuis `develop`.
3. Écrire le test qui échoue avant le code, sauf prototype explicitement jetable.
4. Implémenter une unité cohérente et vérifier tests, lint, types et build.
5. Mettre à jour la documentation technique du service.
6. Écrire la recette manuelle dans `docs/manual-tests/` avant de demander la validation.
7. Ouvrir une PR vers `develop` avec les critères d’acceptation et les commandes exécutées.
8. Attendre la validation humaine de la recette ; puis merger en squash, mettre à jour les statuts, `TODO.md`, le pointeur de submodule et l’environnement local.
