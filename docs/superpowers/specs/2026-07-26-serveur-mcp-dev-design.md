# Design — Serveur MCP dev (Story 1.7)

> Statut : brainstorming validé. À transformer en plan d'implémentation avec `superpowers:writing-plans` une fois la story 1.5 livrée.

## Contexte

Quest passe d'un modèle Space/Quest/Step (arbres isolés) à un graphe unifié Space/Node/Edge (voir [spec graphe unifié](2026-07-26-space-graphe-unifie-design.md) et [LEXIQUE.md](../../../LEXIQUE.md)). Une fois l'API Node/Edge livrée (story 1.5), on veut pouvoir créer et faire évoluer des Projets/Objectifs/Étapes en discutant directement avec Claude, sans passer par l'UI ni par des appels HTTP manuels.

C'est la première brique concrète d'un objectif plus large : un SaaS où un agent IA aide à planifier et réaliser des objectifs. Cette story reste volontairement un usage **dev-only** (Alexandre seul, via Claude Code) pour valider le concept avant d'envisager un agent intégré au produit.

## Concept métier ajouté : Projet

Ajouté à `LEXIQUE.md` :

> **Projet** — Un Objectif racine et l'ensemble des Étapes qui remontent jusqu'à lui (mêmes Nœuds que ceux utilisés pour calculer sa progression). Ce n'est pas une entité stockée à part : c'est une vue calculée à partir d'un Objectif + ses Ancêtres. Une Étape peut appartenir à plusieurs Projets si elle alimente plusieurs Objectifs.

Aucune nouvelle table, aucune migration Prisma : le Projet est purement une vue/agrégation côté API et MCP.

## Approches envisagées

1. **MCP en client HTTP fin de l'API NestJS** (retenue) — chaque tool MCP appelle un endpoint existant de l'API. Toute la logique métier (calcul de progression, validation) reste dans l'API, réutilisable plus tard par le produit web.
2. **MCP avec accès direct Prisma/DB** — écarté : duplique la logique métier hors de l'API et crée un second point d'accès à la base à maintenir.

## Périmètre de la story 1.7

- Nouveau dossier `mcp/` à la racine du repo (à décider : submodule dédié ou simple dossier — trancher lors du plan, par cohérence avec `api/`/`web/` qui sont des submodules).
- Serveur MCP Node/TypeScript utilisant le SDK MCP officiel (`@modelcontextprotocol/sdk`).
- Le serveur s'authentifie contre l'API NestJS (mécanisme dev à définir : token statique en attendant l'epic 3 auth).
- Dépendance dure : story 1.5 (API Node/Edge) doit être livrée avant de démarrer 1.7 — pas de double implémentation sur l'ancien modèle Quest/Step.

### Tools MCP exposés (v1)

| Tool | Rôle |
|---|---|
| `create_projet` | Crée un Objectif racine (Nœud `type=OBJECTIF` sans parent) |
| `add_etape` | Crée une Étape (Nœud `type=ETAPE`) et le Lien vers un Objectif/Étape existant |
| `list_projets` | Liste les Objectifs racine (candidats "Projet") |
| `get_projet` | Retourne un Objectif + ses Ancêtres + sa progression calculée |
| `valider_objectif` | Bascule un Objectif en statut "complété" (validation manuelle) |

Chaque tool = un appel HTTP vers l'API NestJS existante, pas de logique métier dans le MCP.

## Hors périmètre (v1)

- Authentification multi-utilisateur (attend l'epic 3).
- Exposition du MCP au produit web / agent intégré (deuxième usage envisagé mais pas cette story).
- Suppression de Nœuds/Liens, réorganisation du graphe.

## Placement roadmap

Story 1.7, positionnée après 1.6 (canvas Space unifié), avant l'epic 2 (mise en production) — dernière brique du chantier "graphe unifié" avant le déploiement. Mis à jour dans `TODO.md` et `sprint-status.yaml`.

## Tests

- Tests d'intégration du MCP : chaque tool appelé contre une instance locale de l'API (mêmes fixtures/seed que l'API).
- Recette manuelle (`docs/manual-tests/story-1-7-serveur-mcp-dev.md`) : scénario "je décris un objectif en langage naturel à Claude → les Nœuds/Liens attendus existent en base".
