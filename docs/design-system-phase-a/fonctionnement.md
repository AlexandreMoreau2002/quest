# Design System — Phase A : comment ça marche

Explication simple, comme si on racontait ça à quelqu'un qui découvre le sujet pour la première fois.

## 1. Les tokens : la boîte à couleurs qu'on peut changer d'un coup

Imagine une boîte de crayons. Au lieu d'écrire directement "violet clair" dans chaque dessin, tu colles une étiquette sur chaque crayon : "couleur-accent", "couleur-fond", "couleur-texte". Le jour où tu veux changer de thème (par exemple passer d'un thème "nuit étoilée" à un thème "jour"), tu n'as qu'à changer le contenu des étiquettes — pas besoin de refaire tous les dessins un par un.

C'est exactement ce que fait le fichier `web/src/app/tokens.css`. Il définit des variables CSS qui commencent toutes par `--q-` (comme "quest") : `--q-bg`, `--q-accent`, `--q-done`, `--q-locked`, etc. Toutes ces variables sont regroupées dans une classe appelée `.qt-nocturne` (le thème "nocturne" — sombre et étoilé, le seul thème pour l'instant).

Cette classe `.qt-nocturne` est posée sur l'élément racine de la carte, dans `web/src/components/quest-map.tsx` :

```
<main className="quest-shell qt-nocturne atlas-calm" ...>
```

**Le problème que ça résout** : aucun composant (carte, panneau, bouton, nœud) n'écrit de couleur en dur. Ils utilisent tous `var(--q-accent)`, `var(--q-done)`, etc. Résultat : plus tard, si on veut ajouter un thème "jour" ou un thème personnalisé, il suffira de créer une nouvelle classe `.qt-jour { --q-bg: ...; --q-accent: ...; }` et de la poser à la place de `.qt-nocturne` — sans toucher à un seul composant React. Le design et le code sont découplés.

## 2. Le statut d'une carte se voit à sa forme, pas seulement à sa couleur

Une carte de quête (objectif ou étape) peut être dans 3 états : **verrouillée** (pas encore accessible), **en cours** (active), ou **accomplie** (terminée). Plutôt que de compter uniquement sur la couleur (ce qui pose problème pour les daltoniens, ou juste en un coup d'œil rapide), chaque état a une **forme** différente :

- **Verrouillée** → bordure en pointillés (`border-style: dashed`), légèrement transparente et désaturée (`opacity: .78`, `filter: saturate(.6)`). Elle a l'air "éteinte".
- **En cours** → bordure pleine, ET un petit point qui clignote doucement à côté du texte de statut (`.status-active .node-status::before`, animation `q-pulse` de 1,8s en boucle). Ce point pulsant attire l'œil, un peu comme un signal radar.
- **Accomplie** → bordure pleine, sans le point pulsant.

En plus de la forme, il y a la **taille** : une carte "objectif" (le grand cap à atteindre) est plus grande (272×132 px) qu'une carte "étape" (224×100 px, qui grandit légèrement si le titre est long). Ça permet de repérer immédiatement, même de loin sur la carte, ce qui est un objectif et ce qui est une étape — sans lire le texte.

Fichiers concernés : `web/src/app/globals.css` (règles `.status-locked`, `.status-active`, `.status-done`, tailles) et `web/src/lib/map/graph.ts` (constantes `OBJECTIVE_WIDTH`, `OBJECTIVE_HEIGHT`).

## 3. Les connexions touchent le bord des cartes, pas leur centre

Avant, les traits qui relient un objectif à ses étapes partaient et arrivaient au centre de chaque carte, en passant "à travers" le rectangle — visuellement pas terrible. Maintenant, chaque trait s'arrête pile sur le bord de la carte, là où la ligne imaginaire vers le centre de l'autre carte croise le rectangle.

C'est le rôle de la fonction `intersectRectangle` dans `web/src/lib/map/edge-geometry.ts`. Elle prend :
- un rectangle (`rect` : position + largeur + hauteur d'une carte),
- un point cible (`target` : en général le centre de l'autre carte),

et calcule le point exact où le segment "centre du rectangle → cible" coupe le bord du rectangle.

Petit schéma ASCII pour visualiser :

```
                    target (centre de l'autre carte)
                          *
                         /
                        /
        +--------------/----+
        |             /     |
        |            /      |
        |     C =====X      |   C = centre du rectangle
        |    (centre)       |   X = point d'intersection avec le bord
        |                    |       (c'est LÀ que le trait s'arrête)
        +--------------------+
```

La fonction compare de combien il faut "grandir" le vecteur centre→cible pour toucher le bord vertical (gauche/droite) ou le bord horizontal (haut/bas) du rectangle, et prend le plus petit des deux (`scale = Math.min(scaleX, scaleY)`) — c'est ce qui garantit qu'on touche exactement le bord, pas un coin au hasard ni un point à l'intérieur.

Ensuite, `web/src/components/quest-edge.tsx` utilise ce point d'intersection pour dessiner une courbe légèrement incurvée (pas une ligne droite toute raide) entre les deux cartes, ce qui donne l'effet "carte vivante" organique voulu par le design.

## 4. Le fond étoilé qui scintille

Le fond de la carte (`web/src/app/globals.css`, classe `.q-star`) affiche de petits points lumineux (`box-shadow` pour l'effet de halo) qui clignotent doucement grâce à l'animation `q-twinkle`, avec un délai différent pour chaque étoile pour que ça ne clignote pas toutes en même temps. La densité d'étoiles est elle-même un token (`--q-star-density`), réglable via la classe `.atlas-calm` posée sur la carte.

## Fichiers impactés

- `web/src/app/tokens.css` — définit toutes les variables `--q-*` du thème `.qt-nocturne`.
- `web/src/app/globals.css` — utilise les tokens pour styliser cartes, statuts, connexions, étoiles.
- `web/src/app/layout.tsx` — charge les polices et prépare la structure globale de la page.
- `web/src/components/quest-map.tsx` — pose la classe `.qt-nocturne` sur la racine de la carte, gère le rendu des nœuds (React Flow) et leur accessibilité clavier.
- `web/src/components/quest-edge.tsx` — dessine les connexions courbes entre cartes en utilisant `intersectRectangle`.
- `web/src/lib/map/edge-geometry.ts` — contient la fonction `intersectRectangle` qui calcule le point de contact sur le bord d'une carte.
- `web/src/lib/map/graph.ts` — définit les tailles de cartes (`OBJECTIVE_WIDTH/HEIGHT`), les statuts (`StepStatus`) et construit le graphe (nœuds + arêtes) affiché sur la carte.
