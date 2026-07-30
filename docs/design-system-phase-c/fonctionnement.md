# Phase C — Thèmes de démonstration & variante mobile : fonctionnement

## Le principe de thémabilité

Imagine que chaque carte, chaque panneau, chaque ligne sur la carte de Quest porte un "costume". Le costume, ce sont les couleurs, les ombres, la police du titre, l'arrondi des coins. En dessous du costume, la personne (la carte, le panneau, la connexion) est toujours la même : même taille, même comportement, mêmes informations affichées.

Techniquement, le "costume" est une liste de variables CSS nommées `--q-quelquechose` (ex. `--q-accent`, `--q-surface`, `--q-font-display`). Chaque thème (`.qt-nocturne`, `.qt-pirate`, `.qt-futurist`) redéfinit la valeur de ces mêmes variables — jamais leur nom. Le code des composants (la carte, le panneau, la connexion) ne référence jamais une couleur en dur, seulement `var(--q-accent)` etc. Donc changer de thème = changer une classe CSS sur l'élément racine de la carte, rien d'autre ne bouge dans le code.

## Comment le switcher fonctionne

Trois points colorés en haut à droite de la barre du haut. Cliquer sur un point change la classe posée sur l'élément racine (`quest-shell`) : `qt-nocturne` → `qt-pirate` → `qt-futurist`. Le choix est sauvegardé dans le stockage local du navigateur (`localStorage`, clé `quest-theme`), donc il survit à un rechargement de page.

Le hook `useTheme` utilise `useSyncExternalStore` plutôt qu'un simple `useState` + `useEffect`. La différence pratique : avec `useEffect`, le thème par défaut (Nocturne) s'affichait brièvement avant que le thème réellement sauvegardé ne s'applique — un "flash" visible à chaque rechargement. `useSyncExternalStore` est le mécanisme React conçu pour lire une source de vérité externe (ici le `localStorage`) de façon synchrone dès le premier rendu côté client, ce qui supprime ce flash tout en évitant les erreurs d'hydratation Next.js (le rendu serveur ne connaît jamais le `localStorage`, donc il doit toujours partir du thème par défaut, puis le client corrige immédiatement sans flash intermédiaire visible).

## La variante mobile

Sous 760px de large, il n'y a plus la place pour deux panneaux fixes sur les côtés. Deux changements :

- Le panneau de création d'objectif (habituellement à gauche) devient un bouton rond "+" flottant en bas à droite. Cliquer dessus fait remonter le même formulaire depuis le bas de l'écran, comme un tiroir (bottom-sheet), avec un petit repère horizontal en haut du tiroir pour rappeler qu'on peut le fermer.
- Le panneau de détail (habituellement à droite, visible quand une carte est sélectionnée) devient aussi un tiroir qui remonte du bas, plutôt qu'un panneau fixe à droite.

Le contenu du formulaire (labels, champs, bouton) est strictement identique entre desktop et mobile — seul son emballage visuel change, via un hook `useMediaQuery('(max-width: 760px)')` qui détecte la largeur d'écran.

## Fichiers impactés

- `web/src/app/theme-tokens.css` — valeurs des tokens Pirate et Futuriste
- `web/src/app/layout.tsx` — chargement des polices d'affichage par thème
- `web/src/hooks/use-theme.ts` — état du thème actif + persistance
- `web/src/components/theme-switcher.tsx` — les 3 points cliquables
- `web/src/hooks/use-media-query.ts` — détection desktop/mobile
- `web/src/components/quest-map.tsx` — branchement du thème et du mode mobile
- `web/src/app/globals.css` — styles bottom-sheet/FAB, corrections d'opacité des cartes

## Corrections apportées pendant la vérification finale

En testant les trois thèmes bout en bout, plusieurs problèmes ont été trouvés et corrigés dans cette même phase :

- **Connexions qui ne menaient nulle part** : quand on créait une étape en tirant une branche depuis une autre étape (pas l'objectif), la carte apparaissait mais la ligne restait toujours reliée à l'objectif central. La fonction qui construit la carte (`buildQuestGraph`) ignorait complètement le lien parent réel de l'étape (`parentStepId`). Corrigé : chaque étape se connecte maintenant à son vrai parent, qu'il s'agisse de l'objectif ou d'une autre étape.
- **Scintillement au survol d'une carte** : le petit bouton "+" qui apparaît au survol d'une carte est positionné exactement sur son bord. Le curseur qui frôlait cette zone faisait apparaître/disparaître le bouton en boucle rapide. Corrigé par un court délai avant de le cacher, annulé si le curseur revient dessus ou sur la carte.
- **Ancre qui dérive pendant un déplacement de la carte (pan)** : la position à l'écran du bouton "+" n'était recalculée que lors d'un changement de survol, pas en continu pendant qu'on déplace la carte. Corrigé en la recalculant à chaque mouvement de la vue.
- **Cartes un peu transparentes** sur le fond étoilé animé : un fond opaque a été ajouté derrière le dégradé de couleur des cartes.

## Limitations connues

- Un utilisateur clavier doit utiliser Maj+Tab (pas Tab) pour atteindre le bouton "+" d'ancrage une fois qu'il apparaît, à cause de l'ordre dans lequel les éléments sont écrits dans la page (limitation héritée de la Phase B, non résolue ici).
- Le re-parentage vers un objectif (et non une étape) est refusé par l'API, car un objectif n'est pas une étape dans le modèle de données actuel.
