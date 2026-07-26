# Lexique métier — Quest

> Vocabulaire de référence pour le domaine métier de Quest. À consulter avant d'employer un terme ambigu (Node, Card, Step, Quest, Space, HUD...).

## Termes validés

| Terme | Couche | Définition |
|---|---|---|
| **Space** | Métier + Front | Le conteneur global du graphe — la carte/canvas sur laquelle tout vit. Un seul Space actif pour l'instant, mais le modèle reste ouvert à en accueillir plusieurs plus tard. |
| **Nœud** (`Node`) | Métier + Back | La brique unique du graphe. Remplace l'ancienne distinction séparée Quest/Step : un Nœud a un `type` qui détermine son rôle (Objectif ou Étape). |
| **Objectif** | Métier | Un Nœud avec `type = OBJECTIF`. Un but final vers lequel convergent des Étapes (ex : "Gagner beaucoup d'argent", "Freelance"). Peut lui-même être l'ancêtre ou le descendant d'un autre Objectif. |
| **Étape** | Métier | Un Nœud avec `type = ETAPE`. Une action concrète à accomplir, qui peut alimenter un ou plusieurs Objectifs. |
| **Lien** (`Edge`) | Back | Une flèche orientée entre deux Nœuds ("mène à"). Un Nœud peut avoir plusieurs liens entrants (plusieurs parents) et sortants — le graphe est un DAG, pas un arbre strict. |
| **Ancêtres d'un Objectif** | Métier | Tous les Nœuds qui remontent jusqu'à lui via des Liens. Sert de base au calcul de sa progression. |
| **Validation** | Métier | Action manuelle qui bascule un Objectif en statut "complété", indépendamment de son pourcentage de progression (même à 100%, il faut valider explicitement). |
| **Card** | Front | La représentation visuelle d'un Nœud sur le Space — l'élément que l'utilisateur voit, déplace et clique. |
| **HUD** | Front | L'ensemble des composants d'interface qui se superposent au Space (barres d'outils, panneaux latéraux, modales, recherche...). Distinct du Space (le fond/la carte qui bouge) et des Cards (les éléments posés dessus). |
| **Projet** | Métier | Un Objectif racine et l'ensemble des Étapes qui remontent jusqu'à lui (mêmes Nœuds que ceux utilisés pour calculer sa progression). Ce n'est pas une entité stockée à part : c'est une vue calculée à partir d'un Objectif + ses Ancêtres. Une Étape peut appartenir à plusieurs Projets si elle alimente plusieurs Objectifs. |

## Décisions structurantes liées au lexique

- Un seul type de Nœud en base (pas deux tables séparées Quest/Step) — le `type` fait la différence.
- La progression d'un Objectif = (Nœuds ancêtres complétés) / (total Nœuds ancêtres), calculée automatiquement — pas de pondération manuelle pour le MVP.
- Un Objectif peut atteindre 100% automatiquement mais reste au statut précédent tant qu'il n'est pas validé manuellement.

*Dernière mise à jour : 2026-07-26 — ajout du terme Projet pendant le brainstorming serveur MCP.*
