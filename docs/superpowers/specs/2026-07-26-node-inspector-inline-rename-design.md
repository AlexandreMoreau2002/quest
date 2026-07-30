# Inspector de node et renommage inline

## Objectif

Rendre l’inspector d’un node cohérent avec le design system Quest et permettre
de renommer immédiatement les nodes créés par le geste de connexion vers le
canvas vide.

## Contexte observé

L’inspector actuel affiche un titre non éditable et des boutons HTML génériques.
La création par drag fabrique un titre temporaire (`Nouvelle étape (...)`),
mais aucun état d’édition n’est activé ensuite. L’API possède déjà
`PATCH /nodes/:nodeId` avec un champ `title`.

## Design retenu

### Inspector

L’inspector reste un overlay `glass-panel`, mais adopte une hiérarchie complète :

```text
┌────────────────────────────┐
│ ÉTAPE                      ×│  eyebrow + fermeture
│ [ titre éditable        ]  │  input design system
│ ACTIF                       │  badge d’état
│ [Marquer complété]          │  action secondaire
│ [Supprimer]                 │  action danger
└────────────────────────────┘
```

Le champ titre est toujours disponible quand un node est sélectionné. `Enter`
ou blur sauvegarde un titre non vide; `Escape` restaure le titre précédent.
L’input reçoit le focus automatiquement uniquement pour un node nouvellement
créé par drag, afin que la frappe commence directement.

### Création enfant

Le node est créé avec un titre technique valide pour satisfaire le DTO API,
puis relié au parent. Dès sa création, il est sélectionné et placé en mode
édition avec un brouillon vide. La sauvegarde utilise `updateNodeTitle`; si le
brouillon est vide, on restaure le titre technique sans envoyer de PATCH invalide.

```text
drag handle -> vide
    │
    ├─ POST node avec titre temporaire
    ├─ POST edge parent ↔ enfant
    └─ sélection + focus input vide
             └─ frappe -> Enter/blur -> PATCH title
```

### État et erreurs

`useSpaceMap` expose `updateNodeTitle`, met à jour le graphe localement puis
persiste via le client API lorsque la source est `api`. En cas d’échec, le hook
conserve le titre précédent et remonte une erreur existante à l’interface.

## Vérification

- tests du client API pour le PATCH title;
- tests du hook pour la mise à jour locale et la persistance;
- tests du composant pour l’inspector, Enter/blur/Escape et le focus de création;
- lint, tests et build web;
- recette manuelle avec node existant et enfant créé par drag.
