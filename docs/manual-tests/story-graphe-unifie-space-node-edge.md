# Recette manuelle — Graphe unifié Space/Node/Edge

## Prérequis

- `docker compose up --build` (Postgres + API) depuis `api/`.
- `npm run db:seed` dans `api/` pour charger le graphe d'exemple.
- `npm run dev` dans `web/`.

## Scénario 1 — Vue d'ensemble

**Given** le seed a été chargé
**When** j'ouvre la page d'accueil
**Then** je vois tous les objectifs (Gagner beaucoup d'argent, Freelance, Cloudbreak, achat-revente) et leurs étapes sur un seul Space, y compris l'étape "Épargner 3 000 € de trésorerie" reliée à la fois à Cloudbreak et à achat-revente.

## Scénario 2 — Déplacer une Card

**Given** le Space est affiché
**When** je clique-maintiens une Card et la déplace
**Then** les liens connectés suivent la Card en continu pendant le déplacement
**And** après un rechargement de page, la Card reste à sa nouvelle position.

## Scénario 3 — Créer un lien

**Given** deux Cards existent sans lien entre elles
**When** je glisse depuis la poignée d'une Card vers une autre
**Then** un nouveau lien apparaît immédiatement et persiste après rechargement.

## Scénario 4 — Compléter une étape et voir la progression

**Given** une étape reliée à un objectif est "active"
**When** je la marque "complétée" depuis l'inspecteur
**Then** la progression de l'objectif augmente en conséquence (vérifiable via `GET /nodes/:id/progress`).

## Scénario 5 — Valider un objectif

**Given** un objectif affiche 100% de progression
**When** je clique sur "Valider l'objectif"
**Then** son statut passe à "complété" (avant ce clic, il restait "actif" malgré les 100%).

## Cas limite — Supprimer un nœud partagé

**Given** l'étape "Épargner 3 000 € de trésorerie" est reliée à deux objectifs
**When** je la supprime
**Then** les deux objectifs restent sur le Space, simplement sans lien entrant de cette étape (pas de suppression en cascade).

## Checklist finale

- [ ] Tous les nœuds du seed sont visibles simultanément
- [ ] Drag & drop fluide, liens qui suivent
- [ ] Création de lien par glisser-déposer
- [ ] Suppression de lien (sélection + Suppr)
- [ ] Calcul de progression correct pour une étape partagée
- [ ] Validation manuelle distincte du 100%
- [ ] Suppression de nœud sans cascade
