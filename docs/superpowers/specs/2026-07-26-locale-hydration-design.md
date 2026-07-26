# Correction du mismatch d’hydratation lié à la langue

## Objectif

Supprimer le warning Next.js de mismatch SSR/client lorsque la langue `en` est
déjà mémorisée dans `localStorage`, tout en conservant la préférence de langue
de l’utilisateur.

## Cause

Le serveur ne peut pas lire `localStorage` et rend donc toujours le français.
Le premier rendu client de `useLocale` lit directement `localStorage`; avec une
préférence `en`, il produit immédiatement des traductions différentes. React
compare ces deux arbres avant que l’hydratation soit terminée et régénère le
sous-arbre côté client.

## Design retenu

`useLocale` utilise `useSyncExternalStore` avec un snapshot serveur et un
snapshot client initial identiques (`fr`). Après l’hydratation, le snapshot
client lit la préférence persistée et notifie le composant, ce qui applique
`en` sans divergence pendant l’hydratation. Le changement explicite de langue
continue d’écrire dans `localStorage`, de changer i18next et de notifier les
abonnés.

Le comportement attendu est donc :

```text
SSR                 Hydratation             Après hydratation
fr  ───────────────> fr  ────────────────> préférence localStorage
                                                   (fr ou en)
```

Le périmètre reste limité au hook de locale et à ses tests; le hook de thème,
qui applique déjà ce modèle, sert de référence locale.

## Vérification

- conserver les tests existants du hook (`fr` par défaut, persistance, valeur
  corrompue);
- ajouter un test qui vérifie que le snapshot serveur reste `fr` alors qu’une
  préférence `en` est disponible côté navigateur;
- exécuter les tests web, ESLint et le build Next.js.
