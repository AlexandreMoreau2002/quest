# Git Flow — Quest

1. Vérifier que `develop` est à jour dans le submodule concerné.
2. Créer `feature/<story-id>-<slug>` depuis `develop`.
3. Commiter uniquement des changements cohérents, au format Conventional Commits.
4. Pousser la branche et ouvrir une PR GitHub avec `gh pr create --base develop`.
5. Ne merger qu’après validation de la recette manuelle par le propriétaire.
6. Merger en squash, passer la story à `done`, mettre à jour `TODO.md`, puis mettre à jour le pointeur du submodule dans le repo racine via une PR vers `develop`.

`main` : uniquement README initiaux de service, sur autorisation explicite. Aucun autre travail quotidien ne va directement sur `main`.
