# Guide de test manuel — Design System Phase A

Recette manuelle pour valider les tokens sémantiques, l'anatomie des cartes, les connexions organiques, le fond étoilé et l'état vide de l'atlas de quêtes.

## Prérequis

```bash
cd web
npm run dev
```

Ouvrir ensuite `http://localhost:3000` (ou le port assigné si 3000 est déjà pris).

## Scénarios

### Scénario 1 — Créer un objectif

**Given** je suis sur la carte, avec le panneau de gauche ouvert ("Choisis un cap").
**When** je renseigne un titre dans le champ "Nouvel objectif" et un titre dans "Premier projet ou étape", puis je clique sur "Créer l'objectif".
**Then** une nouvelle carte "objectif" apparaît sur la carte, avec une taille visuellement plus grande que les cartes d'étape (272×132 px), une bordure pleine et une étiquette "OBJECTIF".
**And** une carte "étape" liée y est reliée par un trait courbe.

### Scénario 2 — Ajouter une étape à un objectif existant

**Given** un objectif est sélectionné (panneau de droite affiché).
**When** je saisis un titre dans le champ "Ajouter une étape à ..." et je clique sur "Ajouter l'étape".
**Then** une nouvelle carte "étape" (224×100 px) apparaît, reliée par un trait courbe qui touche le bord de la carte source (pas son centre).

### Scénario 3 — Observer chaque statut de nœud

**Given** la carte affiche des étapes à différents statuts.
**When** j'observe visuellement chaque carte.
**Then** :
- une carte **verrouillée** ("À venir") a une bordure en pointillés, un aspect légèrement estompé/désaturé ;
- une carte **active** ("En cours") a une bordure pleine et un petit point qui pulse doucement à côté du texte "En cours" ;
- une carte **accomplie** a une bordure pleine, sans point pulsant.

### Scénario 4 — Sélection et panneau de droite

**Given** je suis sur la carte.
**When** je clique sur une carte (objectif ou étape).
**Then** le panneau de droite se met à jour avec le titre, le statut et le contexte (quête parente) de la carte cliquée.

### Scénario 5 — Focus clavier (Tab puis Entrée/Espace)

**Given** je clique d'abord sur une zone vide de la carte (pour retirer le focus des champs de formulaire).
**When** j'appuie plusieurs fois sur `Tab`.
**Then** le focus se déplace d'élément en élément (arêtes puis cartes) et chaque carte focusée affiche un anneau de focus visible (halo autour de la carte).
**When** une carte est focusée et que j'appuie sur `Entrée` (ou `Espace`).
**Then** le panneau de droite se met à jour avec les informations de cette carte — exactement comme un clic à la souris.

> Point de vigilance corrigé pendant cette recette : `nodesFocusable` a été désactivé sur le composant `ReactFlow` (`web/src/components/quest-map.tsx`) pour éviter un double arrêt de tabulation (le conteneur interne de React Flow était focusable en plus de la carte elle-même), ce qui rendait environ un arrêt sur deux inactif au clavier.

### Scénario 6 — Redimensionnement de la fenêtre (comportement mobile existant)

**Given** je suis sur la carte en plein écran.
**When** je redimensionne la fenêtre du navigateur à une largeur mobile (ex. 375px).
**Then** la mise en page existante s'adapte (aucune régression attendue par cette phase, qui ne touche pas la logique responsive).

## Cas limites

### Carte vide (aucun objectif)

**Given** aucun objectif n'existe encore (première visite, ou tous les objectifs supprimés côté API).
**When** j'ouvre la page.
**Then** un état vide s'affiche au centre de la carte : "CARTE VIDE — Aucun objectif pour l'instant." avec une invitation à créer un premier objectif depuis le panneau de gauche.

### Plusieurs objectifs

**Given** plusieurs objectifs existent, chacun avec ses propres étapes.
**When** j'ouvre la carte.
**Then** chaque objectif et ses étapes sont positionnés sans chevauchement excessif (voir la logique de disposition en "branches" dans `web/src/lib/map/graph.ts`), avec une couleur d'accent différente par quête pour les distinguer visuellement.
**And** le sélecteur d'objectif dans le panneau de gauche liste bien tous les objectifs existants.

## Checklist finale

- [ ] Créer un objectif fonctionne et la carte apparaît avec la bonne taille (272×132 px).
- [ ] Ajouter une étape fonctionne et la carte apparaît avec la bonne taille (224×100 px, ou plus si le titre est long).
- [ ] Statut verrouillé : bordure en pointillés, aspect estompé.
- [ ] Statut actif : bordure pleine + point pulsant.
- [ ] Statut accompli : bordure pleine, sans point pulsant.
- [ ] Clic sur une carte met à jour le panneau de droite.
- [ ] Tab fait apparaître un anneau de focus visible sur chaque carte.
- [ ] Entrée sur une carte focusée met à jour le panneau de droite.
- [ ] Espace sur une carte focusée met à jour le panneau de droite.
- [ ] Les connexions sont courbes et touchent le bord des cartes (pas leur centre).
- [ ] Le fond étoilé scintille (petites étoiles avec halo, clignotement décalé).
- [ ] L'état vide s'affiche correctement quand aucun objectif n'existe.
- [ ] Le comportement au redimensionnement de fenêtre reste inchangé.
