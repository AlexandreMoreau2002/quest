# Guide de test manuel — pill de connexion local

## Prérequis

- Installer les dépendances du dossier `web`.
- Démarrer l'application avec `npm run dev` depuis `web`.
- Ouvrir la carte dans un navigateur qui permet de modifier l'URL ou de
  lancer un alias local pour tester chaque hostname.
- Pour le cas API, configurer une API accessible ; pour le cas local, laisser
  l'API indisponible afin d'obtenir `Mode local`.

## Scénarios

### 1. `localhost`

1. Ouvrir `http://localhost:3000`.
2. Attendre la fin du chargement initial.
3. Vérifier que le pill est visible dans la barre supérieure.
4. Vérifier qu'il indique `Mode local` si l'API est indisponible.
5. Avec une API disponible, recharger puis vérifier que le même pill indique
   `API connectée`.

Résultat attendu : le pill est visible ; son texte reflète la source courante.

### 2. `127.0.0.1`

1. Ouvrir `http://127.0.0.1:3000`.
2. Attendre le montage de la carte.
3. Vérifier la présence du pill et la cohérence de son texte avec la source.

Résultat attendu : le pill est visible.

### 3. IPv6 loopback navigateur réel (`[::1]`)

1. Vérifier que le serveur de développement écoute sur IPv6.
2. Ouvrir `http://[::1]:3000`.
3. Attendre le montage de la carte.
4. Vérifier la présence du pill.

Résultat attendu : le pill est visible. Le test doit être fait avec cette URL
bracketée : c'est la forme de hostname utilisée par le navigateur pour le
loopback IPv6 dans ce scénario.

### 4. Hostname public / production-like

1. Servir l'application derrière un hostname non local, par exemple
   `quest.example.com`, ou utiliser le domaine de préproduction disponible.
2. Ouvrir la carte et attendre la fin du montage.
3. Vérifier que la barre supérieure reste utilisable.
4. Vérifier que ni `API connectée` ni `Mode local` n'est affiché.
5. Créer ou déplacer un élément si le scénario est disponible, afin de
   confirmer que seule l'indication visuelle est masquée.

Résultat attendu : le pill est absent, sans régression fonctionnelle visible.

### 5. Contrôle SSR / hydratation

1. Charger une URL locale avec la console du navigateur ouverte.
2. Observer le premier affichage puis l'affichage après montage.
3. Vérifier qu'aucune erreur de hydration mismatch n'apparaît dans la console.
4. Répéter sur un hostname public et vérifier que le pill reste absent après
   le montage.

Résultat attendu : le premier rendu peut être sans pill, puis le pill apparaît
uniquement après l'effet client sur un hostname local.

## Cas limites

- Vérifier qu'un port différent (`localhost:3001`, si l'application y est
  servie) ne transforme pas un hostname local en hostname public.
- Vérifier qu'un nom ressemblant à localhost mais différent, par exemple
  `localhost.example.com`, masque le pill.
- Vérifier qu'un hostname public avec une API réellement disponible masque
  quand même le pill : le statut réseau ne doit pas modifier la règle
  d'affichage.

## Checklist finale

- [ ] `localhost` affiche le pill.
- [ ] `127.0.0.1` affiche le pill.
- [ ] `[::1]` affiche le pill dans un navigateur réel.
- [ ] Un hostname public masque le pill.
- [ ] Le texte local/API reste correct quand le pill est visible.
- [ ] Les actions de la carte continuent de fonctionner sur hôte public.
- [ ] Aucun avertissement ou erreur d'hydratation n'apparaît.

