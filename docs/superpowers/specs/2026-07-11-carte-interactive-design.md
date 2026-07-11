# Design — Carte interactive (sous-projet MVP)

## Contexte

Quest est un SaaS de progression personnelle qui transforme les objectifs de vie en quêtes interactives, visualisées comme un parcours RPG. Le MVP global couvre : authentification, création de quêtes, sous-objectifs, checkpoints, carte interactive, progression, IA de génération d'étapes.

Ce document couvre uniquement le sous-projet **Carte interactive** — décidé comme premier chantier car c'est l'élément différenciant du produit. Les autres sous-projets (Fondations auth, IA copilote) feront l'objet de specs séparées.

## Décisions de scope

- **Style visuel** : 2.5D (profondeur simulée par ombres/dégradés), pas de vraie caméra 3D. Choix guidé par la priorité "MVP rapide d'abord, wahou ensuite" — le vrai 3D (WebGL/Three.js) est repoussé à une itération future une fois le concept validé.
- **Thème** : un seul thème pour le MVP — Île / Plage. Les autres thèmes (Galaxie, Montagne, Finance, etc.) et la personnalisation arrivent en V2.
- **Multi-domaine** : une quête appartient à un Space principal (qui porte le thème). Les dépendances entre domaines (ex: l'étape "1000€/mois" en Finance débloque une étape en Voyage) sont modélisées via des liens croisés (CrossLink) entre étapes, affichés comme un indicateur "portail" cliquable sur la carte — plutôt qu'un graphe unique mêlant tous les domaines.
- **Embranchements** : gérés dès le MVP via une simple relation d'auto-référence sur les étapes (`parent_step_id`) — pas de table Quest séparée pour les quêtes secondaires. Une quête secondaire est une branche d'étapes qui part d'une étape parente.
- **Interaction** : clic sur une étape → panneau latéral (la carte reste visible derrière). Une transition zoom/plein écran avec animations est notée pour une itération V2 (y compris mobile), mais hors scope MVP.

## Architecture

- **Frontend** : Next.js (React) + React Flow pour le rendu du graphe (nœuds = étapes, arêtes = chemins/embranchements). React Flow gère nativement zoom/pan/drag ; le rendu visuel des nœuds est entièrement personnalisé avec nos propres composants stylisés (générés en partie via GPT).
- **Backend** : API séparée en Node.js/NestJS + PostgreSQL (via Prisma). Stack JS/TS de bout en bout pour permettre l'introduction de Three.js plus tard sans changer d'écosystème. L'API est pensée dès le départ comme consommable par un futur client mobile (pas de couplage au rendu web).

## Modèle de données

### User
| Champ | Type | Contrainte |
|---|---|---|
| id | uuid | PK |
| email | string | unique |
| password_hash | string | |
| created_at | timestamp | |

### Space
Domaine de vie (ex: Finance, Formation). Porte le thème visuel.

| Champ | Type | Contrainte |
|---|---|---|
| id | uuid | PK |
| user_id | uuid | FK → User |
| name | string | |
| theme | enum | fixe `island` pour le MVP |
| created_at | timestamp | |

### Quest
Objectif principal d'un Space.

| Champ | Type | Contrainte |
|---|---|---|
| id | uuid | PK |
| space_id | uuid | FK → Space |
| title | string | |
| description | text | nullable |
| status | enum | `active` / `completed` / `locked` |
| created_at | timestamp | |

### Step
Étape sur le chemin d'une quête. Les quêtes secondaires sont des branches de Steps (pas des Quest distinctes).

| Champ | Type | Contrainte |
|---|---|---|
| id | uuid | PK |
| quest_id | uuid | FK → Quest |
| parent_step_id | uuid | FK → Step, nullable (self-ref : chaîne linéaire si un seul enfant, embranchement si plusieurs) |
| title | string | |
| icon | string | clé d'icône |
| order | int | ordre parmi les enfants d'un même parent |
| progress_percent | int | 0–100 |
| status | enum | `locked` / `active` / `completed` |
| created_at | timestamp | |

### SubGoal
Checklist à l'intérieur d'une Step (ex: React ✓, Docker ☐).

| Champ | Type | Contrainte |
|---|---|---|
| id | uuid | PK |
| step_id | uuid | FK → Step |
| label | string | |
| is_done | bool | default false |

### CrossLink
Dépendance croisée entre étapes de Spaces différents.

| Champ | Type | Contrainte |
|---|---|---|
| id | uuid | PK |
| source_step_id | uuid | FK → Step |
| target_step_id | uuid | FK → Step |

### ProgressEvent
Historique — alimentera la timeline et les bilans mensuels (hors scope carte, mais capturé dès maintenant pour ne pas perdre de données).

| Champ | Type | Contrainte |
|---|---|---|
| id | uuid | PK |
| step_id | uuid | FK → Step |
| user_id | uuid | FK → User |
| type | enum | `checkpoint_completed` / `subgoal_done` / `note_added` |
| payload | jsonb | données libres selon le type |
| created_at | timestamp | |

## Relations (schéma)

```
User 1───N Space 1───N Quest 1───N Step 1───N SubGoal
                                     │
                                     │ (self-FK parent_step_id)
                                     └── enfants (suite linéaire ou embranchement)

Step N───N Step   (via CrossLink : source_step_id / target_step_id)
Step 1───N ProgressEvent
```

## Rendu visuel

- Chaque Step est une carte : icône dans un cercle + titre + % de progression, reliée aux autres par des chemins en pointillés.
- Le détail (sous-objectifs, ressources, actions) n'apparaît qu'au clic, dans le panneau latéral — pas sur la carte elle-même.
- Fond du thème Île/Plage : abstrait, peu détaillé, sert uniquement d'ambiance (≈20% du rendu perçu). Les 80% restants viennent des cartes/étapes/connexions/animations.

## Hors scope MVP

- Thèmes multiples et personnalisables
- Vraie caméra 3D (WebGL/Three.js)
- Transition zoom/plein écran au clic sur une étape
- Génération IA du contenu des étapes (sous-projet séparé : "IA copilote")
- Authentification et gestion de compte (sous-projet séparé : "Fondations")

## Tests

- Modèle de données : tests unitaires sur les contraintes (self-référence Step, CrossLink cross-space, cascade de suppression Quest → Step → SubGoal).
- Frontend : rendu correct d'une chaîne linéaire, d'un embranchement (plusieurs enfants), et d'un CrossLink (indicateur portail cliquable) avec des données de test (fixtures).
- Pas de tests end-to-end complets dans ce sous-projet (dépend de l'auth, hors scope).
