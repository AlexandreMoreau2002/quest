# Design — Déploiement du MVP (Step 2)

## Contexte

Le MVP brouillon (web Next.js + API NestJS/Prisma/PostgreSQL) existe en local (step 1, réalisé). L'objectif de ce step 2 est de le rendre accessible en ligne, sans se soucier de la finition fonctionnelle (repoussée au step 3). Contrainte : hébergement 100% gratuit, zéro action manuelle récurrente — seuls des commits et la vérification des checks CI doivent être nécessaires après la mise en place initiale.

## Décisions

- **Pas de preprod/staging** : un seul environnement de déploiement par service, basé sur `develop`.
- **Pas de domaine custom** : URLs par défaut des hébergeurs (`*.onrender.com`, `*.vercel.app`).
- **Pas de monitoring/alerting** : hors scope tant qu'il n'y a pas d'utilisateurs réels.
- **Pas de Dokploy** : Dokploy est un PaaS auto-hébergé gratuit en logiciel, mais nécessite un VPS payant. Écarté pour rester 100% gratuit sans infra à gérer. Option à reconsidérer plus tard si le projet grandit et qu'un contrôle total (pas de cold start, pas de limites free tier) devient nécessaire.
- **Migrations automatiques, seed manuel/local uniquement** : les migrations Prisma doivent s'appliquer automatiquement à chaque déploiement (déjà le cas via `docker-entrypoint.sh`). Le seed ne doit **pas** tourner en production — il reste un geste local pour peupler une base de dev.
- **Repo racine hors CI/CD** : le repo racine (`quest/`) ne contient que de la documentation et les pointeurs de submodules ; il n'est pas déployable et ne reçoit pas de pipeline. Le git flow strict (main protégée, develop = intégration, feature/*) et le déploiement s'appliquent aux submodules `api` et `web` (leurs propres repos GitHub), pas au repo racine.
- **Secrets** : stockés uniquement dans les dashboards Render et Vercel (pas de secrets GitHub Actions), car le CI n'a pas besoin de toucher à la vraie base de données.

## Architecture

```
push develop (repo quest-api) → GitHub Actions (lint + test + build, Postgres éphémère en service CI)
                                → si vert : Render auto-deploy
                                    → build via Dockerfile existant
                                    → docker-entrypoint.sh : prisma migrate deploy (toujours) — pas de seed (RUN_SEED absent)
                                    → API en ligne, connectée à Neon (DATABASE_URL en variable d'env Render)

push develop (repo quest-web) → GitHub Actions (lint + test + build)
                                → si vert : Vercel auto-deploy
                                    → Web en ligne, NEXT_PUBLIC_API_URL pointe vers l'URL Render (variable d'env Vercel)
```

## Composants

### 1. Neon (base de données)
Projet Postgres gratuit, un seul environnement (pas de branching DB pour ce step). La connection string est utilisée uniquement comme variable d'env `DATABASE_URL` sur Render — jamais dans GitHub Actions ni committée.

### 2. Render (API)
Web Service de type Docker, pointant sur le repo `quest-api`, branche `develop`, déploiement automatique à chaque push. Réutilise le `Dockerfile` existant tel quel. Variables d'env à configurer dans le dashboard Render :
- `DATABASE_URL` — connection string Neon
- `CORS_ORIGIN` — URL du déploiement Vercel

Le conteneur applique `prisma migrate deploy` au démarrage (comportement déjà présent dans `docker-entrypoint.sh`), donc chaque déploiement met la base à jour automatiquement sans action manuelle.

### 3. Vercel (Web)
Projet Next.js standard, pointant sur le repo `quest-web`, branche `develop`, déploiement automatique à chaque push. Variable d'env à configurer dans le dashboard Vercel :
- `NEXT_PUBLIC_API_URL` — URL du service Render

### 4. GitHub Actions — CI qualité
Un workflow par submodule, déclenché sur push et PR vers `develop` :

**`api/.github/workflows/ci.yml`**
- Service Postgres éphémère (conteneur GitHub Actions, non lié à Neon)
- Étapes : install → lint → test (jest, avec `DATABASE_URL` pointant vers le Postgres éphémère) → build
- N'exécute pas de déploiement : Render se déclenche indépendamment via son intégration GitHub native sur push vers `develop`. Si un déploiement doit être bloqué par un CI rouge, ce sera configuré côté Render (auto-deploy conditionné, à vérifier disponible en free tier) — sinon acceptable en l'état que CI et déploiement soient parallèles pour ce step.

**`web/.github/workflows/ci.yml`**
- Étapes : install → lint → test (vitest) → build
- Même logique : Vercel se déploie indépendamment via son intégration GitHub native.

## Changement de code nécessaire

`api/docker-entrypoint.sh` : le seed (`prisma db seed`) devient conditionné par la variable d'env `RUN_SEED=true`, au lieu de s'exécuter systématiquement dès que `prisma/seed.ts` existe. `RUN_SEED=true` sera défini uniquement dans `api/compose.yaml` (environnement local). Render ne définira pas cette variable, donc le seed ne tournera jamais en production.

Comportement cible du script :
```sh
if [ -x "$prisma_cli" ] && [ -f "$prisma_schema" ]; then
  "$prisma_cli" migrate deploy
  if [ "${RUN_SEED:-}" = 'true' ] && [ -f './prisma/seed.ts' ]; then
    "$prisma_cli" db seed
  else
    echo 'Seed skipped (RUN_SEED not set to true, or seed not configured).'
  fi
...
```

## Composants réutilisés sans modification

- `api/Dockerfile` — build multi-stage existant
- `api/package.json` script `validate` — sert de base aux étapes du job CI api
- Migrations Prisma déjà présentes dans `api/prisma/`

## Hors scope (repoussé au step 3 ou plus tard)

- Environnements de preview/staging séparés
- Domaine custom
- Monitoring/alerting
- Dokploy / VPS auto-hébergé
- Blocage strict du déploiement Render/Vercel sur échec CI (à affiner si besoin une fois le flow de base en place)

## Vérification de cohérence CORS

À vérifier lors de l'implémentation : que le module NestJS lit bien `CORS_ORIGIN` (ou équivalent) depuis une variable d'env plutôt qu'une valeur codée en dur, pour pointer vers l'URL Vercel en production.
