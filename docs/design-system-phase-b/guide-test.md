# Guide de test manuel — Design system Phase B

## Prérequis

- API lancée : `cd api && npm run start:dev` (nécessite Postgres via Docker ; voir `api/README.md` — `docker compose up --build -d` puis vérifier `curl --fail http://localhost:3001/health`).
- Web lancée : `cd web && npm run dev`, puis ouvrir `http://localhost:3000`.
- Un espace avec au moins un objectif contenant plusieurs étapes (le seed par défaut convient).

## Scénario 1 — Créer une branche par drag depuis une carte précise

**Given** la carte est ouverte, une carte A (ex. « Développer mon activité freelance ») est sélectionnée dans le panneau de droite, et une carte B différente (ex. « Lancer Cloudbreak ») est visible sur la carte.

**When**
1. Je survole la carte B avec la souris.
2. Une ancre (bouton `+`) apparaît sur le bord de la carte B (pas sur A).
3. Je clique sur l'ancre, je déplace la souris vers un espace vide, puis je clique à nouveau.
4. Le menu s'ouvre. Je choisis « Ajouter une étape », je saisis un titre, je valide.

**Then**
- Une nouvelle carte apparaît sur la carte.
- Dans l'API (`GET /spaces/:id/map`), le `parentStepId` de la nouvelle étape correspond à l'id de la carte **B** (celle d'où le drag est parti), pas à la carte A précédemment sélectionnée.

## Scénario 2 — Créer une branche au clavier

**Given** la carte est ouverte.

**When**
1. Je clique sur une zone vide de la carte (pour ne garder aucun état de survol souris actif).
2. Je navigue jusqu'à une carte avec `Tab`.
3. Depuis un élément qui suit la carte dans l'ordre du DOM (ex. les contrôles de zoom), j'utilise `Shift+Tab` pour revenir jusqu'à l'ancre de la carte visée (annoncée « Ajouter une branche depuis <titre> »).
4. J'appuie sur `Entrée`.

**Then**
- Le menu de branche s'ouvre directement (pas besoin de terminer un « drag » à la souris).
- Je peux choisir une action (étape, objectif lié, élément existant) au clavier.

## Scénario 3 — Lier un élément existant

**Given** la carte contient au moins deux cartes en plus de la carte source.

**When**
1. Je démarre un drag de branche depuis une carte A (souris ou clavier).
2. Dans le menu, je choisis « Lier un élément existant ».
3. Je recherche puis sélectionne une carte B déjà existante dans la même quête.

**Then**
- Une requête `PATCH /steps/:B` est envoyée avec `{ "parentStepId": "<A>" }`.
- La carte se met à jour : B est maintenant rattachée à A dans l'arbre.

## Scénario 4 — Re-parentage invalide (autre quête) rejeté

**Given** deux quêtes distinctes existent, chacune avec au moins une étape.

**When**
1. Via `curl` ou l'onglet réseau du navigateur, j'envoie :
   ```
   PATCH /steps/<step-de-la-quête-1>
   { "parentStepId": "<step-de-la-quête-2>" }
   ```

**Then**
- La réponse est `409 Conflict` avec un message explicite (« The parent step must belong to the same quest. »).
- La carte n'est pas modifiée côté UI si la requête a été tentée depuis l'interface (le message d'erreur générique du panneau s'affiche : « Le re-parentage a échoué. »).

## Cas limites

- **Aucun élément existant à lier** : si la carte n'a qu'une seule carte (juste l'objectif, aucune étape), le mode « Lier un élément existant » du menu affiche une liste de recherche vide — aucune erreur ne doit apparaître, juste une liste vide.
- **Drag relâché en dehors de la zone valide** : si je relâche le second clic du drag en dehors de la zone de la carte (ex. sur le panneau latéral avec `data-map-overlay`), le menu doit tout de même s'ouvrir à la position du clic sans planter l'application (le menu peut être partiellement recouvert par le panneau, mais reste utilisable).
- **Auto-parentage** : `PATCH /steps/:id` avec `parentStepId` égal à `:id` lui-même doit renvoyer `409 Conflict` (« A step cannot be its own parent. »).

## Checklist finale

- [ ] Hover sur une carte → une seule ancre apparaît, sur le bord extérieur correct.
- [ ] Drag depuis une carte précise → menu → « Ajouter une étape » → la nouvelle carte est bien attachée à la carte de départ (vérifié via l'API), pas à la carte précédemment sélectionnée.
- [ ] Navigation clavier (Tab / Shift+Tab / Entrée) → l'ancre est atteignable et Entrée ouvre le menu directement.
- [ ] « Lier un élément existant » fonctionne et persiste via `PATCH /steps/:id`.
- [ ] Re-parentage vers une étape d'une autre quête → `409 Conflict`.
- [ ] Auto-parentage (`parentStepId === id`) → `409 Conflict`.
- [ ] Comportements Phase A toujours fonctionnels : pan, zoom, sélection, anneau de focus sur les cartes, courbes organiques des liens, fond étoilé, état vide.
- [ ] `npm run test`, `npm run lint`, `npm run build` passent côté `api/` et `npm run validate` passe côté `web/`.
