# Locale hydration mismatch — comment ça marche

## Le problème, en version simple

La langue est mémorisée dans le navigateur avec `localStorage` sous la clé `quest-locale`.

Quand cette valeur est `en`, le navigateur veut afficher l’interface en anglais.
Mais si le serveur a déjà rendu la page en français, et que le tout premier rendu côté client lit directement `localStorage`, on obtient deux versions différentes de la même page.

Résultat : React voit une différence entre ce que le serveur a envoyé et ce que le navigateur essaie d’hydrater. C’est ça le mismatch.

## Avant le correctif

```text
Serveur SSR
  -> lit la langue par défaut
  -> rend la page en FR

Navigateur
  -> lit `localStorage.quest-locale = en`
  -> rend tout de suite la page en EN

Comparaison React
  -> FR côté serveur, EN côté client
  -> warning d’hydratation
```

En clair : le serveur dit "français", le premier rendu client dit "anglais". Les deux ne collent pas.

## Après le correctif

Le hook `useLocale` garde la même idée :

- le serveur rend toujours la valeur par défaut `fr` ;
- le premier rendu client utilise aussi `fr` ;
- après l’hydratation, le navigateur peut relire `localStorage` et passer à `en` si c’est la préférence enregistrée ;
- quand l’utilisateur change la langue, la valeur est réécrite dans `localStorage` et l’interface se met à jour.

```text
1. SSR
   serveur -> snapshot serveur = fr
   HTML envoyé = FR

2. Hydratation
   premier rendu client -> snapshot serveur = fr
   React retrouve exactement le même HTML
   pas de warning

3. Après hydratation
   navigateur -> lit `localStorage.quest-locale`
   si la valeur est `en`, l’UI bascule en EN

4. Changement manuel
   clic FR / EN
   -> `localStorage` est mis à jour
   -> i18n change de langue
   -> l’UI se rerend
```

## Schéma ASCII

```text
                ┌──────────────────────────┐
                │         Serveur          │
                │  snapshot = fr           │
                │  HTML = français         │
                └─────────────┬────────────┘
                              │
                              ▼
                ┌──────────────────────────┐
                │     1er rendu client     │
                │  snapshot = fr           │
                │  hydratation OK          │
                └─────────────┬────────────┘
                              │
                              ▼
                ┌──────────────────────────┐
                │  localStorage quest-locale│
                │  = en ?                  │
                └─────────────┬────────────┘
                              │ oui
                              ▼
                ┌──────────────────────────┐
                │     UI après hydrate     │
                │  langue = EN             │
                └──────────────────────────┘
```

## Pourquoi ça règle le souci

Le point important est que le serveur et le premier rendu client racontent la même histoire.

React n’a donc plus besoin de "corriger" la page au moment où il l’hydrate.
La préférence enregistrée dans `localStorage` peut ensuite reprendre la main, mais seulement après cette phase critique.

## Fichiers impactés

- `web/src/hooks/use-locale.ts` — lit la langue de façon compatible avec SSR et l’hydratation.
- `web/src/hooks/use-locale.test.ts` — couvre le cas où `localStorage` contient déjà `en`.
- `web/src/i18n/i18n-provider.tsx` — consomme `useLocale` pour synchroniser i18next.

## Ce qui ne change pas

- pas de nouvelle route HTTP ;
- pas de modification backend ;
- pas de changement des dictionnaires FR / EN ;
- pas de changement de l’API publique du provider i18n.

