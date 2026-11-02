---
name: dev-grill
description: Interroger l'utilisateur par rounds jusqu'à ce qu'aucune décision ne reste ouverte sur un brief, un plan ou une idée. Utilisé par /dev avant le spec et par dev-init ; aussi quand l'utilisateur demande de le « challenger » ou « griller » sur un plan.
---

# dev-grill

Objectif : une compréhension partagée avant d'écrire quoi que ce soit.

1. Construire l'arbre des décisions : chaque décision ouvre celles qui en dépendent.
2. **Frontière** = les décisions dont les prérequis sont tranchés. Poser la frontière en un round, **au plus 5 questions**, les plus bloquantes d'abord ; le reste attend le round suivant.
3. Format de chaque question :

   ```
   ❓ **Q1 — <titre>** : <question, options s'il y en a>
   ➡️ <réponse recommandée et pourquoi, en une ligne>
   ```

4. Les **faits** (code, config, données, maquette, historique git) ne sont jamais demandés à l'utilisateur : les chercher soi-même ou via un agent `Explore`. Une recherche en cours bloque seulement les questions qui en dépendent ; poser les autres.
5. Les **décisions** appartiennent à l'utilisateur : attendre ses réponses. Ne jamais répondre à sa place.
6. Après chaque round, recalculer la frontière. Fin quand elle est vide ; récapituler les décisions en une liste courte et demander confirmation avant d'agir.
