# Guide de test manuel — i18n de l'interface carte

## Prérequis

- `cd web && npm run dev`
- Ouvrir `http://localhost:3000` dans le navigateur.
- L'API (`quest-api`) peut être démarrée ou non : l'application fonctionne aussi en mode local (badge "Mode exploration" au lieu de "API connectée").

## Scénario 1 — Tout le texte visible est en français correct

1. Charger la page d'accueil.
2. Vérifier la topbar : eyebrow "EXPÉDITION ACTIVE" (ou "Mode exploration"/"API connectée" selon l'état de l'API), nom de l'espace.
3. Vérifier le panneau de gauche (création d'objectif) : titre "Choisis un cap.", labels de champs, texte du bouton "Créer l'objectif", astuce en bas.
4. Cliquer sur un nœud de la carte pour ouvrir le panneau de droite (inspecteur) : vérifier le fil d'ariane, le titre, le badge de statut ("En cours" / "Accompli" / "À venir" / "Verrouillé"), le texte d'ajout d'étape et le bouton "Ajouter l'étape".
5. **Aucune clé brute ne doit apparaître** nulle part (ex. jamais `topbar.eyebrow` ou `errors.questNotSaved` affiché tel quel à l'écran) — ce serait le signe d'une clé mal orthographiée.

## Scénario 2 — Toggle FR / EN

1. Repérer les deux boutons `FR` / `EN` dans la topbar.
2. Vérifier que `FR` est visuellement actif/surligné.
3. Survoler `EN` : une infobulle native du navigateur doit afficher "Bientôt disponible". Le bouton doit apparaître grisé/désactivé.
4. Cliquer sur `EN` : rien ne doit se passer (pas de changement de langue, pas d'erreur console).
5. Cliquer sur `FR` (déjà actif) : rien ne doit casser, l'interface reste identique.

## Scénario 3 — Changement de thème

1. Cliquer successivement sur les 3 pastilles de thème (Atlas Nocturne, Pirate, Ville futuriste).
2. Pour chaque thème, vérifier que :
   - le toggle FR/EN reste lisible (contraste correct, texte non tronqué) ;
   - tous les textes de l'interface restent lisibles et correctement traduits (pas de régression de copie liée au changement de thème).

## Scénario 4 — Vue mobile (FAB + bottom sheet)

1. Redimensionner la fenêtre à une largeur < 760px (ou utiliser le mode responsive du navigateur).
2. Vérifier qu'un bouton flottant (FAB) "+" apparaît en bas à droite.
3. Cliquer dessus : un bottom-sheet doit s'ouvrir avec le libellé "Créer un objectif" (`t('map.createObjectiveFab')`).
4. Sélectionner une étape sur la carte : le bottom-sheet doit afficher l'inspecteur avec le même contenu qu'en desktop (titre, statut, formulaire d'ajout d'étape), correctement traduit.

## Cas limite — API indisponible

1. Couper l'API (ex. `docker stop quest-api-1` ou équivalent local).
2. Recharger la page : le badge topbar doit passer à "Mode exploration" (pas une clé brute).
3. Tenter de créer un objectif ou une étape.
4. Si un message d'erreur apparaît (selon le comportement de fallback), il doit être un texte français complet et cohérent (ex. "La quête n'a pas été enregistrée…"), jamais une clé brute type `errors.questNotSaved`.
5. Redémarrer l'API et recharger : le badge doit repasser à "API connectée".

## Checklist finale

- [ ] Tout le texte visible (topbar, panneau gauche, panneau droit, cartes de nœuds, empty state) est en français correct.
- [ ] Aucune clé i18n brute n'est visible à l'écran.
- [ ] `FR` actif, `EN` grisé/désactivé avec l'infobulle "Bientôt disponible", clic sans effet.
- [ ] Les 3 thèmes gardent le toggle de langue et tout le texte lisibles.
- [ ] Le FAB mobile et son bottom-sheet fonctionnent et affichent la bonne copie française.
- [ ] Le message d'erreur (API coupée) est un texte français complet, pas une clé brute.
