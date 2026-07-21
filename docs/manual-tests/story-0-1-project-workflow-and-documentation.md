# Recette — Story 0.1 : workflow projet et documentation

## Statut

`À valider`

## Prérequis

- Branche ou PR : `feature/0-1-project-workflow`
- Aucun service n’est nécessaire.

## Parcours de validation

### AC-1 — Source de vérité

**Given** le repo racine ouvert

**When** on consulte `CLAUDE.md`, `TODO.md` et `_bmad-output/implementation-artifacts/sprint-status.yaml`

**Then** le workflow, le sprint CRUD/carte et la story active sont identifiables sans autre contexte.

### AC-2 — Règles de livraison

**Given** une nouvelle story à développer

**When** on consulte `rules/git-flow.md`, `rules/story-delivery.md` et `rules/quality.md`

**Then** la branche feature, la PR vers `develop`, les validations et la recette humaine à fournir sont explicites.

### AC-3 — Règles des services

**Given** le sousmodule `api/` ou `web/`

**When** on lit son `CLAUDE.md`

**Then** les responsabilités, commandes attendues et règles spécifiques sont adaptées au service.

## Commandes automatisées exécutées

```text
ruby -e "require 'yaml'; YAML.load_file('_bmad-output/implementation-artifacts/sprint-status.yaml')"
git diff --check
```

## Résultat de validation humaine

`En attente`
