# Journal d'apprentissage /dev — example

Tenu par `dev-retro`. Le texte exact de chaque règle appliquée figure ici : retirer une règle = supprimer ce texte dans le fichier cible.

## 2026-01-15 — SHOP-123

- [appliqué] coding-assertions.md § Pièges connus : « Un handler `src/api/**` valide son entrée avec zod avant tout accès aux données. » — preuve : SHOP-123, finding sécurité corrigé après review
- [candidat ×1] executor : test ciblé lancé sans chemin → suite complète
- [refusé] profile.md § gates : « build exigé sur chaque ticket » — trop lent pour les petits tickets
