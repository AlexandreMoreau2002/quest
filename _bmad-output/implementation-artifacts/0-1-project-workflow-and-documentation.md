# Story 0.1 — Workflow projet et documentation

## Objectif

Installer pour Quest le même cadre de delivery que Cloudbreak, adapté à une architecture Next.js + NestJS avec submodules.

## Critères d’acceptation

1. Le repo racine contient un guide projet et un carnet de bord.
2. Les services `api/` et `web/` ont chacun leurs règles Claude adaptées à leur responsabilité.
3. Le statut BMad décrit les stories du sprint CRUD et carte.
4. Les règles Git Flow, qualité et livraison décrivent la PR obligatoire, la recette humaine et le handoff post-merge.
5. Un template de recette manuelle est disponible avant toute story fonctionnelle.

## Hors scope

- Installation de la totalité du runtime BMad de Cloudbreak.
- Ajout de hooks Claude dépendants d’outils absents de Quest.
- Implémentation de la story 1.1.

## Fichiers concernés

- `CLAUDE.md`, `TODO.md`, `.claude/`, `rules/`
- `_bmad-output/implementation-artifacts/`
- `docs/manual-tests/`
- `api/CLAUDE.md`, `web/CLAUDE.md`
