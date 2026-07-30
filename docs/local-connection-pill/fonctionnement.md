# Affichage local du statut de connexion

## Objectif

Le pill de connexion aide au développement local. Il affiche `API connectée`
quand la carte utilise l'API, ou `Mode local` quand elle utilise les données de
secours. Sur un hôte public, ce repère purement visuel est masqué.

Le comportement de chargement des données, de connexion à l'API et de secours
local ne change pas.

## Détection du hostname sans risque SSR

Le composant initialise la visibilité du pill à `false`. Il ne lit donc pas
`window` pendant le rendu serveur ni pendant le premier rendu client. Après le
montage, un effet client lit `window.location.hostname` et active le pill
uniquement si le hostname appartient à la liste locale.

```text
Rendu SSR / premier rendu client
            │
            └── pill absent, aucun accès à window
                         │
                         ▼
                 effet client après montage
                         │
                         ▼
              lecture de location.hostname
                    │              │
             hôte local         hôte public
                    │              │
                    ▼              ▼
              pill visible      pill absent
```

Cette séquence conserve le même markup initial entre serveur et client et évite
un mismatch d'hydratation lié à un accès à `window` pendant le rendu.

## Règle locale / publique

Le pill est visible uniquement pour les valeurs suivantes de
`window.location.hostname` :

| Hostname navigateur | Résultat |
| --- | --- |
| `localhost` | Pill visible |
| `127.0.0.1` | Pill visible |
| `[::1]` | Pill visible ; forme bracketée de l'adresse IPv6 loopback dans le navigateur |
| Tout autre hostname, par exemple `quest.example.com` | Pill masqué |

Le port n'entre pas dans la décision : `localhost:3000` reste local car le
navigateur fournit `localhost` via `location.hostname`.

Le contenu du pill reste lié à la source courante :

- `API connectée` lorsque la source vaut `api` ;
- `Mode local` lorsque la source vaut `local`.

Sur un hostname public, seule cette présentation est masquée. Les actions de
la carte et les appels éventuels à l'API restent inchangés.

## Fichiers impactés

- `web/src/components/space-canvas.tsx` — détection SSR-safe du hostname et
  affichage conditionnel du pill.
- `web/src/components/space-canvas.test.tsx` — tests de visibilité sur les
  hostnames locaux et publics.
- `docs/local-connection-pill/fonctionnement.md` — fonctionnement et flux SSR.
- `docs/local-connection-pill/guide-test.md` — scénarios de validation manuelle.
- `http/local-connection-pill.http` — note display-only, sans endpoint API.
