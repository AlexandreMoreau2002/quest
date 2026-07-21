# Livraison d’une story

## Avant PR

- Critères d’acceptation lus et couverts.
- Tests ciblés, lint, types et build passés.
- Documentation technique créée ou mise à jour dans `<service>/docs/story-<id>-<slug>.md`.
- Recette créée dans `docs/manual-tests/story-<id>-<slug>.md` depuis le template.
- `sprint-status.yaml` passé à `review`.

## Après validation humaine

1. Merger la PR en squash vers `develop`.
2. Passer la story à `done` et mettre à jour `TODO.md`.
3. Mettre à jour la documentation concernée.
4. Mettre le submodule et le repo racine sur `develop`, supprimer la branche feature et vérifier les worktrees propres.
5. Proposer la prochaine story prête selon `sprint-status.yaml`.
