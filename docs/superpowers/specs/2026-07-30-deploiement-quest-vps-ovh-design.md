# Déploiement Quest sur le VPS OVH — Design

Date : 2026-07-30

## Contexte

Suite au déploiement de Snoroc sur le VPS OVH (`51.178.37.35`, Dokploy déjà installé et
opérationnel), on ajoute Quest sur le même serveur, dans un projet Dokploy séparé.

Quest est en développement actif depuis ~2 semaines : `main` ne contient qu'un commit initial
vide sur les deux repos, tout le travail réel est sur `develop`. Pas de domaine acheté pour ce
projet — utilisation de sous-domaines `nip.io` temporaires en attendant.

Repos :
- API : `AlexandreMoreau2002/quest-api`, NestJS + Prisma + PostgreSQL, Dockerfile existant,
  port 3001, healthcheck `/health`.
- Web : `AlexandreMoreau2002/quest-web`, Next.js (mode SSR par défaut, pas d'export statique),
  pas de Dockerfile.

## Architecture cible

```
Internet
   │
   ▼
Traefik (Dokploy, déjà en place)
   │
   ├─ quest.51.178.37.35.nip.io      → frontend Next.js (Nixpacks, process Node)
   └─ quest-api.51.178.37.35.nip.io  → backend NestJS (Docker, :3001)
                                            │
                                            └─ PostgreSQL — service Dokploy, volume persistant
```

## Composants

1. **Projet Dokploy** : nouveau projet `quest`, séparé du projet `snoroc` (isolation des projets,
   règle déjà appliquée pour Snoroc).
2. **Backend** : application Dokploy type Docker, repo `quest-api`, branche `develop`, build via
   le Dockerfile existant du repo. Variables d'env : `DATABASE_URL` construite depuis
   `POSTGRES_USER`/`POSTGRES_PASSWORD`/`POSTGRES_DB` générés. Domaine :
   `quest-api.51.178.37.35.nip.io`, port interne 3001.
3. **Base de données** : service PostgreSQL Dokploy avec volume persistant.
4. **Frontend** : application Dokploy type Nixpacks (Next.js supporté nativement, pas de
   Dockerfile à écrire). Variable d'env `NEXT_PUBLIC_API_URL` pointant vers le domaine backend.
   Domaine : `quest.51.178.37.35.nip.io`.
5. **HTTPS** : Let's Encrypt via Traefik, fonctionne avec `nip.io` car ces sous-domaines résolvent
   directement vers l'IP publique du serveur.
6. **Secrets** : générés et saisis uniquement dans l'UI/API Dokploy, jamais commités.

## Hors scope

- Bascule vers la branche `main` (prévue plus tard, quand le projet sera jugé stable)
- Achat et configuration d'un vrai nom de domaine pour Quest
- Migrations Prisma (à exécuter une fois le schéma stabilisé, si nécessaire au premier déploiement)

## Ordre d'exécution

1. Créer le projet Dokploy `quest`.
2. Créer le service PostgreSQL avec volume persistant.
3. Déployer le backend (Docker, branche `develop`), variables d'env, domaine
   `quest-api.51.178.37.35.nip.io`.
4. Déployer le frontend (Nixpacks, branche `develop`), variable `NEXT_PUBLIC_API_URL`, domaine
   `quest.51.178.37.35.nip.io`.
5. Vérifier les deux applications et les certificats HTTPS.
