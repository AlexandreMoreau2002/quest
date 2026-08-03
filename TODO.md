# TODO — Quest

> Carnet de bord du développement. À consulter au début de chaque session.

## En cours

- Story 1.7 — Serveur MCP dev (tools Projet/Objectif/Étape via l'API NestJS), dépend de 1.5.

## Prochaines priorités

1. [ ] Story 1.7 — Serveur MCP dev (tools Projet/Objectif/Étape via l'API NestJS), dépend de 1.5.
2. [ ] Epic 2 — Mise en production : un environnement dev existe déjà (voir section "Infra Dokploy" plus bas) mais `quest-api` y est désynchronisé de `develop` — auto-deploy à corriger avant de considérer l'epic avancé.
3. [ ] Epic 3 — Authentification et espaces utilisateur (les crosslinks entre projets sont déjà couverts nativement par le DAG, plus besoin de story dédiée).

## Sprint actif — Graphe unifié Space/Node/Edge

| Story | Contenu | Statut |
|---|---|---|
| 1.1 | PostgreSQL, Prisma et seed des objectifs | `done` — modèle remplacé par 1.5 |
| 1.2 | CRUD NestJS objectifs et étapes | `done` — modèle remplacé par 1.5 |
| 1.3 | Carte Next.js reliée à l’API | `done` — modèle remplacé par 1.6 |
| Design system A/B/C | Tokens, ancrages/drag-to-create, thèmes + mobile | `done` |
| i18n | react-i18next, FR par défaut, toggle FR/EN fonctionnel | `done` |
| 1.5 | API du graphe unifié (Node/Edge, progression, validation) | `done` |
| 1.6 | Canvas Space unifié (drag & drop, connexions, HUD) | `done` |
| 1.7 | Serveur MCP dev (tools Projet/Objectif/Étape via l'API) | `ready-for-dev` — dépend de 1.5 |

## Dette technique

- [ ] Définir le modèle utilisateur et l’authentification après validation de la carte MVP (epic 3).
- [ ] Vérifier qu'aucun reliquat de l'heuristique clavier vs souris de l'ancien `NodeAnchor` (event.detail === 0) ne survit après la Task 8 du plan graphe unifié (le composant est supprimé, remplacé par le drag & connect natif React Flow).

## Infra Dokploy — état au 2026-08-03

VPS OVH (`51.178.37.35`, host SSH configuré en local sous `vps-ovh-projets`), **serveur partagé avec d'autres projets perso** (Cloudbreak, snoroc — pas dédié à Quest). Dokploy installé et fonctionnel, Traefik intégré, HTTPS via nip.io. Ce setup n'a pas été fait via le workflow de stories habituel (pas de story Epic 2 formellement ouverte pour ça) — découvert et audité le 2026-08-03 suite à une question de session, à partir des notes déjà écrites côté `cloudbreak/TODO.md` (setup Dokploy identique, fait le 2026-08-02).

| Service | URL | Statut constaté le 2026-08-03 |
|---|---|---|
| `quest-web` | https://quest-dev.51.178.37.35.nip.io | ✅ à jour — sert bien le build post graphe-unifié (drag & connect natif, toggle FR/EN visibles) |
| `quest-api` | même domaine, path-routé via Traefik sur `/api/*` (pas de sous-domaine dédié comme `cloudbreak-dev-api`) | 🔴 **périmée** — `GET /api/health` et `GET /api/spaces` répondent 200, mais `GET /api/spaces/:id/graph` répond 404 "Cannot GET". Cette route existe dans `develop` depuis le commit `4bb8565` (API PR #4, mergé 2026-07-30) mais pas dans le build actuellement en ligne. |

**Conséquence concrète :** le front interroge bien l'API déployée (`useSpaceMap` appelle `loadFirstSpace()` avec succès sur `/spaces` mais échoue sur `/spaces/:id/graph`), l'appel casse silencieusement (`.catch(() => {})`), et l'app retombe sur les données de démo locales (`L'atlas d'Alex`, fichier `web/src/lib/map/fallback.ts`) sans aucun message d'erreur visible — ça ressemble à s'y méprendre à une vraie session utilisateur, ce qui rend le problème facile à manquer.

**Hypothèse la plus probable :** même piège que celui déjà rencontré et documenté sur Cloudbreak (`cloudbreak/TODO.md`, notes du 2026-08-02) — `autoDeploy: true` peut être réglé dans Dokploy sans qu'un webhook GitHub existe réellement, donc les déploiements précédents de `quest-api` étaient probablement manuels et personne n'en a redéclenché un depuis le merge du graphe unifié.

**À vérifier (nécessite l'accès SSH `vps-ovh-projets`, indisponible depuis cette session) :**
- [ ] `gh api repos/AlexandreMoreau2002/quest-api/hooks` et `.../quest-web/hooks` — confirmer si les webhooks GitHub existent vraiment (scope `admin:repo_hook` requis, pas dispo par défaut sur `gh` dans cette session).
- [ ] Dans Dokploy (DB Postgres interne, conteneur `dokploy-postgres`, table `application`) : vérifier `customGitBranch` de l'app `quest-api` (probablement pas `develop`, ou jamais resynchronisée).
- [ ] Une fois corrigé, redéployer `quest-api` manuellement depuis l'UI Dokploy pour resynchroniser sur `develop`, puis revérifier `/api/spaces/:id/graph`.
- [ ] Écrire une doc technique dédiée de la procédure Dokploy (comme demandé côté Cloudbreak) si on veut la reproduire proprement pour un futur environnement prod séparé, pour Quest comme pour les autres projets du VPS.

**Non vérifié depuis cette session** (pas d'accès SSH ici) : méthode de build exacte de chaque app Dokploy (Dockerfile pour `quest-api` vraisemblable vu son `Dockerfile` local ; Nixpacks probable pour `quest-web` comme `cloudbreak-ops`, à confirmer), nom de la base Postgres de dev utilisée par `quest-api`, et si un domaine définitif est prévu (actuellement nip.io comme Cloudbreak). Note : `web/package.json` déclare déjà `engines.node >= 22`, donc le piège Nixpacks/Node 18 rencontré sur `cloudbreak-ops` ne devrait pas se reproduire ici.

## Mergé sur develop ✅

| Story | PR | Date |
|---|---|---|
| Story 0.1 — Workflow projet et documentation | [root PR #1](https://github.com/AlexandreMoreau2002/quest/pull/1), API PR #1, web PR #1 | 2026-07-21 |
| Story 1.1/1.2/1.3 + Design system (A/B/C) + i18n | [API PR #3](https://github.com/AlexandreMoreau2002/quest-api/pull/3), [Web PR #2](https://github.com/AlexandreMoreau2002/quest-web/pull/2) | 2026-07-26 |
| Story 1.5/1.6 — Graphe unifié Space/Node/Edge | [API PR #4](https://github.com/AlexandreMoreau2002/quest-api/pull/4), [Web PR #3](https://github.com/AlexandreMoreau2002/quest-web/pull/3) | 2026-07-30 |
