# Guide de test manuel — locale hydration mismatch

## Prérequis

- Lancer le web app en local.
- Ouvrir l’application dans un navigateur.
- Ouvrir les DevTools avec l’onglet Console visible.

## Préparation du navigateur

1. Ouvrir la Console.
2. Vérifier / nettoyer la clé `quest-locale`.
3. Pour repartir de zéro :

```js
localStorage.removeItem('quest-locale')
```

## Scénario 1 — premier chargement en français

1. Supprimer `quest-locale` si besoin.
2. Faire un rechargement complet de la page.
3. Vérifier que l’interface arrive en français.
4. Vérifier que la Console reste propre, sans warning d’hydratation.

Attendu :

- la première version affichée est en FR ;
- aucun message du type "hydration failed" ;
- aucun warning sur un texte différent entre serveur et client.

## Scénario 2 — langue anglaise déjà persistée

1. Dans la Console, écrire :

```js
localStorage.setItem('quest-locale', 'en')
```

2. Faire un refresh complet de la page.
3. Vérifier qu’aucun warning d’hydratation n’apparaît dans la Console.
4. Vérifier qu’après hydratation l’interface passe en EN.
5. Recharger une seconde fois et vérifier que le résultat reste stable.

Attendu :

- la préférence `en` est bien relue depuis `localStorage` ;
- la page ne casse pas au chargement ;
- la Console ne signale pas de mismatch.

## Scénario 3 — bascule FR / EN dans l’UI

1. Cliquer sur le bouton FR.
2. Cliquer sur le bouton EN.
3. Vérifier que la langue affichée change.
4. Recharger la page.
5. Vérifier que la langue choisie reste la même après refresh.

Attendu :

- `quest-locale` est mis à jour ;
- la langue survive au refresh ;
- le comportement reste stable après plusieurs clics.

## Scénario 4 — retour en français

1. Dans la Console :

```js
localStorage.setItem('quest-locale', 'fr')
```

2. Faire un refresh complet.
3. Vérifier que l’interface revient en français.
4. Vérifier que la Console reste sans warning.

## Cas limite — stockage supprimé pendant la session

1. Ouvrir la page avec `quest-locale = en`.
2. Supprimer la clé dans la Console :

```js
localStorage.removeItem('quest-locale')
```

3. Recharger la page.
4. Vérifier que la langue retombe proprement sur FR.

## Checklist finale

- [ ] `quest-locale` peut être supprimée puis recréée sans erreur.
- [ ] Un premier chargement sans préférence arrive en FR.
- [ ] Un refresh avec `quest-locale = en` finit en EN.
- [ ] FR / EN fonctionne dans l’UI et persiste au refresh.
- [ ] La Console ne montre aucun warning d’hydratation.
- [ ] Aucun changement backend n’est nécessaire pour valider le fix.
