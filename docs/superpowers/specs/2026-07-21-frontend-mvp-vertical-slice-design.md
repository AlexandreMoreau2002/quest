# Design — Frontend MVP : carte de quête interactive

## Statut

Direction visuelle validée en maquette interactive le 21 juillet 2026. Ce document devient la référence pour le premier vertical slice frontend. Il complète la [spec de carte interactive](2026-07-11-carte-interactive-design.md), sans remplacer le modèle de données ni les décisions de scope du MVP global.

## Objectif

Livrer une première expérience web réellement utilisable pour créer une quête, voir ses premières étapes sur une carte plein écran, puis explorer et sélectionner cette carte.

Le résultat doit faire ressentir un monde personnel à explorer, sans attendre l’authentification, l’API ou la génération IA.

## Périmètre du vertical slice

### Inclus

- Une route dédiée à la carte, utilisable sur une fenêtre entière.
- Un formulaire local de création de quête : titre et une ou plusieurs étapes initiales.
- Une carte affichant les étapes comme des nœuds connectés.
- Une navigation de carte au clic-glisser dans toute la surface disponible, hors composants superposés interactifs.
- Zoom par molette, trackpad, boutons et geste de pincement supporté par le navigateur.
- Inertie après un déplacement rapide ; la force du geste influence la distance parcourue après le relâchement.
- Sélection d’une étape et affichage de son détail dans un panneau latéral.
- Données de démonstration locales et création locale en mémoire.
- Styles fondés sur des gradients, un ciel étoilé abstrait et des cartes translucides.

### Explicitement hors scope

- Authentification, persistance, API et base de données.
- Génération IA des étapes.
- Icônes illustrées, assets de fond détaillés ou thème final.
- Branches créées depuis l’interface, CrossLinks, édition et suppression.
- Responsive mobile final : le desktop est la référence de ce slice.

## Expérience utilisateur

1. La personne arrive sur une carte qui prend tout le viewport du navigateur.
2. Elle peut immédiatement explorer le monde par glisser-déposer et zoomer, sans que les nœuds se désolidarisent de leurs connexions.
3. Le panneau gauche permet de saisir un objectif et ses premières étapes.
4. La validation crée une nouvelle quête locale et ajoute ses nœuds à la carte.
5. Le clic sur un nœud ouvre ou met à jour le panneau de détail à droite, tout en laissant la carte visible.

Les éléments superposés qui gardent leur propre interaction sont : la navigation, le panneau de création, les commandes de zoom et le panneau de détail. Le reste de la fenêtre est une zone de navigation de carte.

## Direction visuelle

- Fond : dégradé bleu profond et nébuleuses discrètes ; champ d’étoiles CSS abstrait.
- Profondeur : translucence, ombres douces, bordures claires à faible contraste et gradients sur les nœuds.
- États : vert menthe pour terminé, orange chaud pour actif, indigo désaturé pour verrouillé.
- Étapes : formes géométriques simples avec titre et état ; aucune icône ni illustration n’est requise dans cette version.
- La carte est plus importante visuellement que les panneaux : les panneaux sont compacts et flottants.

## Architecture frontend

Le frontend sera initialisé dans le submodule `web/` avec Next.js, React et TypeScript. Le vertical slice restera entièrement client-side : un provider de données contient les fixtures, la quête en cours et les actions de création/sélection.

La carte s’appuie sur `@xyflow/react` pour le graphe, le zoom et les connexions. Les composants de nœud sont personnalisés. Un contrôleur de viewport, isolé du rendu des nœuds, ajoute l’inertie de déplacement et définit quelles zones peuvent initier un pan. Cela évite de mélanger physique d’interaction, données de quête et présentation.

### Modules prévus

| Module | Responsabilité |
|---|---|
| `app/map/page.tsx` | Assemble l’expérience de carte sans contenir la logique métier. |
| `features/quest/types.ts` | Définit les types locaux `Quest`, `Step`, `StepStatus` et les valeurs de création. |
| `features/quest/fixtures.ts` | Fournit un jeu de données démonstration stable. |
| `features/quest/QuestProvider.tsx` | Gère les quêtes en mémoire, la quête active, la création et l’étape sélectionnée. |
| `features/map/QuestMap.tsx` | Traduit les données de quête en nœuds et arêtes React Flow. |
| `features/map/QuestNode.tsx` | Rend une étape et ses trois états visuels. |
| `features/map/useMomentumPan.ts` | Ajoute l’inertie au viewport et ignore les surfaces interactives superposées. |
| `features/quest/QuestComposer.tsx` | Collecte le titre et les étapes initiales, avec validation locale. |
| `features/quest/StepInspector.tsx` | Affiche l’étape sélectionnée. |
| `app/globals.css` | Héberge uniquement les tokens globaux, le fond et les règles de viewport. |

## Données locales

```ts
type StepStatus = 'completed' | 'active' | 'locked';

type Step = {
  id: string;
  title: string;
  status: StepStatus;
  progressPercent: number;
  parentStepId: string | null;
  order: number;
};

type Quest = {
  id: string;
  title: string;
  steps: Step[];
};
```

Une nouvelle quête démarre avec sa première étape active et les suivantes verrouillées. Les positions de carte sont dérivées de l’ordre des étapes pour ce slice ; elles ne sont pas encore sauvegardées.

## Qualité et tests

- Tests unitaires pour la création d’une quête et la génération des statuts initiaux.
- Tests unitaires pour la conversion `Quest` → nœuds/arêtes, notamment une chaîne de trois étapes.
- Test de composant pour le panneau de création : titre requis, au moins une étape requise, soumission valide.
- Test de composant pour le panneau de détail après sélection d’un nœud.
- Vérification manuelle documentée pour pan, inertie, zoom et zones interactives superposées.

## Workflow de réalisation

Le projet suit un flux BMad allégé, adapté au niveau de maturité actuel :

1. Cette spec définit le besoin et le design du vertical slice.
2. Un plan d’implémentation détaille les tâches, tests et commits.
3. Chaque story est développée sur une branche dédiée, testée et revue avant la suivante.
4. Les décisions de produit ou de design issues de Claude Design sont ajoutées à cette spec avant d’être implémentées.

## Backlog initial

1. **Fondation carte** — initialiser le frontend et rendre une carte plein écran avec fixtures, zoom, pan et inertie.
2. **Création locale de quête** — créer une quête et ses étapes dans le store client puis la visualiser.
3. **Inspection d’étape** — sélectionner un nœud et afficher son détail dans le panneau flottant.
4. **Itération design** — intégrer les maquettes issues de Claude Design, sans ajouter de dépendance fonctionnelle hors scope.

## Critères d’acceptation du premier slice

- La route carte occupe 100 % du viewport desktop.
- Quatre étapes de démonstration sont reliées et leurs liens restent attachés pendant les déplacements et le zoom.
- La carte se déplace par glisser-déposer sur toute zone non recouverte par un composant interactif.
- Un geste rapide entraîne une inertie perceptible puis s’arrête naturellement.
- La création locale d’une quête ajoute ses étapes à la carte sans rechargement.
- Cliquer une étape actualise le panneau de détail.
- Tous les tests automatisés du frontend passent.
