# Assertions de code — example

## Architecture

- `src/components` n'importe jamais `src/features`.
- Réutiliser l'existant (composants, hooks, utils) avant d'écrire.

## Conventions

- TypeScript strict, pas de `any` sans commentaire justifiant.
- Validation des entrées avec zod aux frontières (API, formulaires).

## Tests

- `__tests__/` à côté du code, `*.test.ts(x)`.
- Un test passe par l'interface publique (seam validé au plan) : pas de mock d'un collaborateur interne, pas de valeur attendue recalculée comme le code la calcule.

## Pièges connus (relevés en review)

- (vide au départ ; ajouter chaque piège relevé en review)

## Hygiène

- Commentaires au strict minimum.
- Ne jamais lire un fichier `.env*`.
- Pas de script de vérification perso dans le dépôt.
