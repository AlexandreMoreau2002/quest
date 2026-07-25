# i18n — comment ça marche (explication simple)

## Le principe, en une image

Avant : chaque bouton, chaque titre était écrit "en dur" dans le code du composant.
```tsx
<h1>Choisis un cap.</h1>
```

Maintenant : le composant demande le texte via une **clé**, et un dictionnaire lui répond dans la bonne langue.
```tsx
<h1>{t('creationPanel.title')}</h1>
```

C'est comme un menu de restaurant écrit en code ("plat n°12") : le serveur (le composant) ne connaît que le numéro, et la cuisine (le fichier de traduction) sait ce que ça veut dire dans la langue du client.

Deux fichiers contiennent le dictionnaire complet :
- `web/src/i18n/locales/fr.json` — toutes les clés en français
- `web/src/i18n/locales/en.json` — toutes les clés en anglais

Les deux fichiers ont **exactement les mêmes clés** (même structure, mêmes noms), seule la valeur change. C'est vérifié automatiquement par un test (voir plus bas), donc impossible d'ajouter une clé côté FR en oubliant de l'ajouter côté EN sans que la CI le signale.

## Le toggle FR / EN dans la barre du haut

Dans la topbar, il y a deux boutons : `FR` et `EN`.

- `FR` est actif (surligné) — c'est la langue affichée aujourd'hui.
- `EN` est visible mais **désactivé** (`disabled`), grisé, avec une infobulle native du navigateur "Bientôt disponible" au survol. Cliquer dessus ne fait rien.

Pourquoi le montrer désactivé plutôt que de le cacher complètement ? Parce que ça prouve que la mécanique fonctionne réellement, pas juste sur le papier :
- les deux fichiers de traduction existent et sont synchronisés (`en.json` a vraiment un texte pour chaque clé, pas des chaînes vides) ;
- le mécanisme de changement de langue (`i18n.changeLanguage`, exposé via le hook `useLocale`) est réellement câblé et testé unitairement (`web/src/hooks/use-locale.test.ts`) ;
- il ne manque "que" la décision produit de livrer une vraie expérience anglaise (traductions relues, contenu adapté), pas la plomberie technique.

Autrement dit : la route est goudronnée et testée, mais le panneau "ouverture prochaine" reste affiché tant que le contenu anglais n'est pas validé.

## Les deux gardes-fous automatiques

### 1. Test de parité — `web/src/i18n/locales.test.ts`

Ce test compare la liste des clés de `fr.json` et de `en.json`. Si quelqu'un ajoute une clé dans un seul des deux fichiers (ou fait une faute de frappe dans le nom de la clé), le test échoue immédiatement. Ça empêche un fichier de traduction de "dériver" silencieusement par rapport à l'autre.

### 2. Garde-fou anti-régression — `web/src/i18n/no-hardcoded-strings.test.ts`

Ce test relit le contenu brut des fichiers migrés (`quest-map.tsx`, `branch-menu.tsx`, `use-quest-map.ts`) et vérifie qu'aucune ancienne chaîne française "en dur" (ex. `"EXPÉDITION ACTIVE"`, `"CARTE VIDE"`) n'y est réapparue.

Pourquoi c'est utile : si un développeur (ou un futur refactor un peu pressé) recopie accidentellement le texte au lieu d'utiliser `t('...')`, ou annule par erreur un commit de migration, ce test le détecte tout de suite au lieu de laisser la régression passer inaperçue jusqu'en recette manuelle.

## Comment ajouter une nouvelle clé de traduction

1. Ouvrir `web/src/i18n/locales/fr.json`, ajouter la nouvelle clé avec le texte français.
2. Ouvrir `web/src/i18n/locales/en.json`, ajouter la **même clé** avec le texte anglais (même si l'anglais n'est pas encore "livré" à l'utilisateur, la clé doit exister — sinon le test de parité échoue).
3. Dans le composant, utiliser `t('mon.chemin.de.cle')` au lieu d'écrire le texte en dur.
4. Lancer `npm test` dans `web/` pour vérifier que la parité et le garde-fou anti-régression passent toujours.

## Fichiers impactés

- `web/src/i18n/config.ts` — configuration i18next (langues supportées, fallback, chargement des ressources).
- `web/src/i18n/i18n-provider.tsx` — provider React qui initialise i18next côté client.
- `web/src/i18n/locales/fr.json` — dictionnaire français (langue active).
- `web/src/i18n/locales/en.json` — dictionnaire anglais (clés prêtes, pas encore activées pour l'utilisateur).
- `web/src/hooks/use-locale.ts` — hook exposant la langue courante et `changeLanguage`.
- `web/src/components/language-toggle.tsx` — le bouton FR / EN dans la topbar.
- `web/src/components/quest-map.tsx` — carte principale, migrée vers `t('...')`.
- `web/src/components/branch-menu.tsx` — menu de création de branche par drag, migré.
- `web/src/hooks/use-quest-map.ts` — messages d'erreur (ex. quête non enregistrée), migrés.
- `web/src/app/layout.tsx` — layout racine, monte le provider i18n.
- `web/src/i18n/locales.test.ts` — test de parité des clés FR/EN.
- `web/src/i18n/no-hardcoded-strings.test.ts` — garde-fou anti-régression.

## Limitation connue

`web/src/app/layout.tsx` fixe `<html lang="fr">` de façon statique, côté serveur. La langue affichée à l'utilisateur est en réalité une préférence stockée côté client (via `useLocale` / i18next), donc l'attribut `lang` du document ne suit pas dynamiquement un changement de langue en cours de session. Comme EN est aujourd'hui désactivé, ce n'est pas un problème visible — ça deviendra à traiter le jour où EN sera vraiment activé.
