# Node inspector rename — guide de test

Ce guide couvre la validation manuelle du renommage dans l’inspector, du drag de création enfant jusqu’au rollback en cas d’erreur.

## Prérequis

- le frontend Quest est lancé ;
- l’API Quest est soit disponible, soit volontairement arrêtée selon le scénario ;
- un espace contient au moins un node existant ;
- au moins un node `OBJECTIF` ou `ETAPE` est visible dans le canvas.

## 1. Renommer un node existant

1. Ouvrir l’application.
2. Cliquer sur un node déjà présent dans le canvas.
3. Vérifier que l’inspector s’ouvre à droite sur desktop, ou en bas sur mobile.
4. Cliquer dans le champ titre.
5. Remplacer le texte par un nouveau titre.
6. Presser `Enter`.
7. Vérifier que le titre du node se met à jour.
8. Recliquer sur le node, modifier à nouveau le titre.
9. Sortir du champ avec `Tab` ou un clic ailleurs.
10. Vérifier que la sauvegarde se fait aussi au `blur`.

Attendu :

- le titre est conservé après rafraîchissement si l’API est disponible ;
- aucune erreur n’apparaît ;
- l’input reste lisible et focusable.

## 2. Tester `Escape`

1. Sélectionner un node.
2. Modifier le titre dans l’inspector.
3. Presser `Escape`.

Attendu :

- la valeur du champ revient au titre précédent ;
- aucune requête de mise à jour n’est nécessaire pour ce cancel ;
- le node garde son ancien titre.

## 3. Créer un enfant par drag puis focus automatique

1. Sélectionner un node parent.
2. Attraper un handle de connexion du parent.
3. Glisser le lien vers une zone vide du canvas.
4. Relâcher sur le canvas vide.
5. Vérifier qu’un nouveau node enfant apparaît.
6. Vérifier qu’il est sélectionné.
7. Vérifier que le champ titre de l’inspector reçoit immédiatement le focus.
8. Taper un nouveau titre.
9. Presser `Enter`.

Attendu :

- le nouvel enfant est créé avec un titre temporaire puis renommé ;
- le focus part bien sur l’input sans clic supplémentaire ;
- la sauvegarde part sur le node nouvellement créé.

## 4. Tester le titre vide

1. Sélectionner un node.
2. Effacer entièrement le contenu du champ titre.
3. Presser `Enter` ou sortir du champ.

Attendu :

- aucun `PATCH` n’est envoyé avec un titre vide ;
- le titre affiché revient à la valeur précédente ;
- aucun état d’erreur n’est laissé visible.

## 5. Tester l’API hors ligne

1. Arrêter l’API ou lancer le frontend dans un contexte où l’API n’est pas joignable.
2. Ouvrir l’application.
3. Sélectionner un node déjà présent dans la carte locale.
4. Renommer le node.

Attendu :

- l’application reste utilisable en mode local ;
- le titre est modifié dans l’UI ;
- aucun rollback réseau ne casse la carte ;
- le bandeau de connexion indique le mode local.

## 6. Tester sur mobile

1. Réduire la largeur de la fenêtre sous `760px`, ou utiliser le mode mobile du navigateur.
2. Ouvrir un node dans l’inspector.
3. Vérifier que l’inspector devient un bottom-sheet.
4. Renommer un node existant.
5. Créer un enfant par drag si le geste est réalisable sur l’outil de test utilisé.

Attendu :

- l’inspector reste lisible dans le bottom-sheet ;
- le champ titre reste accessible ;
- les boutons d’action restent utilisables ;
- la fermeture et le renommage fonctionnent comme sur desktop.

## 7. Vérifier un rollback sur erreur

1. Laisser l’API disponible mais provoquer une erreur de mise à jour sur un node cible si votre environnement de test le permet.
2. Tenter un renommage valide.
3. Déclencher la sauvegarde.

Attendu :

- le titre est d’abord modifié localement ;
- puis la valeur précédente est restaurée si le PATCH échoue ;
- le message d’erreur de renommage apparaît sur le node courant ;
- un nouveau changement de texte efface l’erreur.

## Checklist finale

- [ ] un node existant peut être renommé ;
- [ ] `Enter` enregistre ;
- [ ] le `blur` enregistre ;
- [ ] `Escape` annule ;
- [ ] un enfant créé par drag prend le focus sur le titre ;
- [ ] un titre vide ne part pas au serveur ;
- [ ] le mode local reste fonctionnel sans API ;
- [ ] l’inspector mobile s’affiche correctement ;
- [ ] une erreur de PATCH déclenche un rollback ciblé.
