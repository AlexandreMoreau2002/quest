# Phase C — Guide de test manuel

## Prérequis

- `cd web && npm run dev`, ouvrir `http://localhost:3000`.
- L'API est optionnelle : l'app fonctionne aussi en mode local (bandeau "Mode exploration" au lieu de "API connectée"). Pour tester le re-parentage réel, lancer aussi `cd api && npm run start:dev` (nécessite PostgreSQL via Docker, voir `api/README.md`).

## Scénarios

### 1. Changement de thème

**Given** la carte est affichée avec le thème Atlas Nocturne (par défaut).
**When** je clique sur le 2ᵉ point (Pirate) dans la barre du haut.
**Then** le fond, les bordures des cartes, les panneaux, les connexions et les boutons changent immédiatement de palette (tons ambrés/bruns).
**When** je clique sur le 3ᵉ point (Futuriste).
**Then** la palette passe au cyan/magenta néon.

### 2. Persistance du thème

**Given** j'ai sélectionné le thème Pirate.
**When** je recharge la page (F5).
**Then** le thème Pirate reste actif — pas de flash visible du thème Nocturne avant que Pirate ne s'affiche.

### 3. Variante mobile

**Given** je suis sur desktop avec le panneau de création à gauche et l'inspecteur à droite.
**When** je réduis la fenêtre sous 760px de large.
**Then** le panneau de création disparaît, remplacé par un bouton rond "+" en bas à droite.
**When** je clique sur le "+".
**Then** un tiroir remonte du bas de l'écran avec le formulaire de création d'objectif ; je peux le remplir et créer un objectif normalement.
**When** je sélectionne une carte.
**Then** le panneau de détail apparaît aussi comme un tiroir ancré en bas de l'écran (pas flottant au milieu).
**When** j'élargis la fenêtre au-dessus de 760px.
**Then** les panneaux redeviennent des panneaux fixes desktop normaux.

### 4. Non-régression Phases A/B (à vérifier dans au moins 2 thèmes différents)

**Given** la carte est affichée.
**When** je glisse sur la carte vide, je zoome à la molette.
**Then** le déplacement et le zoom restent fluides.
**When** je survole une carte.
**Then** un petit bouton "+" apparaît exactement sur le bord de la carte, du côté opposé à son parent — sans scintiller.
**When** je fais glisser ce bouton vers un espace vide et je relâche.
**Then** un menu s'ouvre avec 3 choix (ajouter un objectif lié / ajouter une étape / lier un élément existant).
**When** je choisis "Ajouter une étape" et je valide un titre.
**Then** une nouvelle carte apparaît, reliée par une ligne à la carte depuis laquelle j'ai tiré la branche (pas forcément à l'objectif central).
**When** j'appuie sur Tab pour naviguer au clavier jusqu'à une carte, puis Maj+Tab pour atteindre son ancre, puis Entrée.
**Then** le même menu de création s'ouvre.

## Cas limites

- **Navigation privée** : le thème choisi ne peut pas être sauvegardé (pas de `localStorage` persistant) → l'app doit se rabattre proprement sur Atlas Nocturne à chaque ouverture, sans erreur.
- **Redimensionnement en direct pendant qu'un panneau mobile est ouvert** : passer sous 760px avec le tiroir de création ouvert, puis repasser au-dessus — le tiroir doit se fermer proprement sans laisser d'overlay visible.

## Checklist finale

- [ ] Les 3 thèmes changent bien l'intégralité de l'interface (fond, cartes, panneaux, connexions, contrôles de zoom)
- [ ] Le thème persiste après rechargement, sans flash
- [ ] Le flux mobile FAB → tiroir → création fonctionne de bout en bout
- [ ] L'inspecteur mobile s'affiche en tiroir ancré en bas
- [ ] Pan/zoom/sélection fonctionnent toujours dans au moins 2 thèmes
- [ ] L'ancre de branche apparaît sans scintiller et reste collée au bord de la carte pendant un déplacement de la vue
- [ ] Une étape créée par glisser-déposer depuis une autre étape (pas l'objectif) se connecte bien à cette étape
- [ ] L'alternative clavier (Tab / Maj+Tab / Entrée) fonctionne toujours
