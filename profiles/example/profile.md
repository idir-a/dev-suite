# Profil /dev — example (application web TypeScript fictive)

## match
- git@github.com:acme/shop-web.git
- shop-web

## commands
- typecheck : `pnpm typecheck`
- test ciblé : `pnpm vitest run <fichier>`
- tests : `pnpm vitest run`
- build : `pnpm build` (séquentiels : non)
- dev : `pnpm dev`

## gates
1. typecheck : 0 erreur.
2. Tests : aucun nouvel échec vs baseline.
3. `pnpm build` OK.
4. Vérif visuelle vs maquette si UI.
5. `docs/` à jour si un comportement documenté change.

## vcs
- Remote MR : `origin` ; base : `origin/main` ; préfixes : `feat`, `fix`, `poc` ; branche `<prefix>/SHOP-123-<slug>`.
- MR : `gh pr create --base main`.
- Hotfix : base `origin/release/*`, report sur `main` après merge.

## sources
- Tracker : GitHub Issues (`gh issue view <n> --comments`).
- Maquettes : Figma.

## task_dir
`.dev/tasks/`

## security
auto
- Toute route `src/api/**` = zone sensible (validation zod obligatoire côté serveur).

## models
(vide = défauts de /dev)

## extra

### Étape 2 — Plan
Lire `docs/architecture.md`. Seams typiques : handlers `src/api/**`, hooks `src/hooks/**`.

### Mode POC
Pas de build de production exigé ; `pnpm dev` suffit.
