# Qualité — Quest

- TDD pour toute logique métier, transformation de graphe, contrat API et mutation CRUD.
- Un test co-localisé par module qui contient de la logique ; un test de flux HTTP complet par story API.
- Les composants de présentation ne font aucun appel HTTP : le client API et les hooks portent les effets asynchrones.
- Les DTO NestJS valident tout payload entrant ; aucune requête SQL brute hors Prisma.
- Les erreurs API utilisent une réponse NestJS standard avec message clair et code HTTP correct.
- Pas de secrets, URLs locales figées ou données sensibles dans le code ; les variables sont décrites dans `.env.example`.
- Aucun composant de carte ne mélange calcul de position, logique de pan et rendu visuel.
