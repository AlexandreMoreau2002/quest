# Node inspector rename — fonctionnement

Ce document explique comment l’inspector d’un node Quest s’assemble visuellement, comment le renommage fonctionne, et comment le flux de création d’un enfant depuis un drag déclenche l’édition automatique du titre.

## Design system de l’inspector

L’inspector n’est pas un simple panneau HTML. Il suit la même grammaire visuelle que le reste de Quest :

- panneau `glass-panel` avec fond dégradé, bordure légère et blur ;
- hiérarchie verticale claire ;
- eyebrow en haut pour le type du node ;
- texte d’aide court ;
- champ de titre mis en avant ;
- badge d’état ;
- pile d’actions secondaires / dangereuses ;
- adaptation mobile en `bottom-sheet`.

Vue simplifiée :

```text
┌──────────────────────────────────────┐
│ ETAPE                           ×    │  eyebrow + fermeture
│ Texte d’aide court                   │  copie de contexte
│                                      │
│ Titre du node                         │  label
│ [ Titre éditable................... ] │  input principal
│ Press Enter or leave the field...     │  hint
│ [statut actif / completed]            │  state badge
│ [Marquer complété]                    │  action secondaire
│ [Supprimer]                           │  action dangereuse
└──────────────────────────────────────┘
```

Sur mobile, le même contenu devient un `bottom-sheet` ancré en bas de l’écran. Le fond et les marges se resserrent, mais le champ titre reste le point d’entrée principal.

## Machine d’état du renommage

Le renommage n’est pas géré par un mode global “édition / lecture”. Le composant garde plutôt un brouillon contrôlé :

- `selectedNodeId` : node courant ;
- `draftNodeId` : id du node dont le titre est en cours d’édition ;
- `titleDraft` : valeur tapée dans l’input ;
- `renameError` : erreur visible seulement pour le node concerné.

Comportement :

```text
Sélection d’un node
    ├─ input affiche le titre courant
    ├─ clic / tap dans le champ -> modification du brouillon
    ├─ Enter -> commit + blur
    ├─ blur -> commit
    └─ Escape -> restore + blur
```

Règles clés :

- `Enter` force la sauvegarde puis déclenche un `blur`, sans double enregistrement.
- `blur` sauvegarde si ce n’est pas le blur artificiel provoqué après `Enter` ou `Escape`.
- `Escape` annule le brouillon en restaurant le titre précédent.
- Un titre vide ne part jamais en PATCH.

## Flux de création d’un enfant par drag

Le geste de création d’un enfant suit un enchaînement précis :

```text
drag handle du parent
    ↓
release sur le canvas vide
    ↓
POST node enfant avec un titre temporaire valide
    ↓
POST edge parent ↔ enfant
    ↓
sélection du nouvel enfant
    ↓
focus automatique sur l’input titre
    ↓
saisie utilisateur
    ↓
Enter / blur
    ↓
PATCH /nodes/:nodeId { title }
```

Le titre temporaire sert uniquement à satisfaire le contrat de création côté API. Dès que le node enfant existe, l’inspector ouvre l’édition avec un brouillon vide pour que l’utilisateur remplace immédiatement ce titre.

## Erreurs, rollback et local-first

Le hook `useSpaceMap` met à jour l’UI localement avant ou pendant la persistance selon la source du graphe.

### Quand l’API est disponible

1. le titre est mis à jour dans l’état local ;
2. le client appelle `PATCH /nodes/:nodeId` ;
3. si la requête réussit, la réponse serveur remplace l’objet local ;
4. si la requête échoue, le titre précédent est restauré et une erreur est affichée.

### Quand l’API est absente

Si l’application démarre sans API, la carte reste en mode local. Dans ce cas :

- aucun PATCH n’est envoyé ;
- le renommage reste fonctionnel en local ;
- l’UI n’affiche pas d’erreur réseau pour ce geste.

### Cas de rollback

```text
titre saisi
    ↓
optimistic update local
    ↓
PATCH échoue
    ↓
restauration de l’ancien titre
    ↓
affichage du message d’erreur sur le node courant
```

Le rollback est ciblé : il ne touche que le node dont le titre a échoué, pas l’ensemble du graphe.

## Vue des fichiers impactés

Cette documentation décrit le comportement observé dans :

- `web/src/components/space-canvas.tsx`
- `web/src/hooks/use-space-map.ts`
- `web/src/lib/api/client.ts`
- `web/src/app/globals.css`
- `web/src/i18n/locales/fr.json`
- `web/src/i18n/locales/en.json`
- `api/src/nodes/nodes.controller.ts`
- `api/src/nodes/nodes.service.ts`

