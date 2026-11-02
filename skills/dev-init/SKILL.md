---
name: dev-init
description: Créer ou mettre à jour le profil /dev d'un dépôt dans ~/.claude/dev-profiles/<nom>/ (profile.md, vcs.md, coding-assertions.md), à partir du dépôt et de quelques questions. Utiliser quand /dev ne trouve pas de profil, ou quand l'utilisateur demande d'initialiser /dev sur un projet.
argument-hint: [nom du profil]
---

# dev-init

Le profil vit **hors du dépôt** : rien n'est écrit dans le dépôt (`git status` inchangé à la fin).

## 1. Lire le dépôt (sans demander)

- Remotes (`git remote -v`), branche par défaut, préfixes de branches et format des 30 derniers commits (`git log --oneline -30`).
- Scripts (`package.json`, `Makefile`, `pyproject.toml`, `justfile`…) → typecheck, test ciblé, test complet, lint, build(s), dev server.
- Conventions : commitlint, hooks (lefthook, husky, pre-commit), `CLAUDE.md`, `CONTRIBUTING.md`, `docs/`, linters.
- CI (`.gitlab-ci.yml`, `.github/workflows/`) ; outil de MR/PR installé (`glab`, `gh`).
- Mémoire projet de Claude Code si elle existe (pièges déjà relevés).
- Jamais de lecture de `.env*`.

## 2. Questions

`dev-suite:dev-grill` seulement sur ce qui ne se déduit pas : tracker (Jira, GitLab, GitHub) et clé de projet, outil de maquette, base des hotfix, gates à exiger, niveau `security` (`auto` recommandé), `task_dir`, spécificités du projet (plusieurs marques/thèmes, CMS et migrations, déploiement).

## 3. Écrire `~/.claude/dev-profiles/<nom>/`

`profile.md` :

```markdown
# Profil /dev — <nom>

## match
- <url du remote origin>
- <nom du dossier racine>

## commands
- typecheck : `…`
- test ciblé : `… <fichier>`
- tests : `…`
- build : `…` (séquentiels : oui/non)
- dev : `…`

## gates
1. …

## vcs
- remote MR : `origin` ; base : `develop` ; préfixes : feat, fix, poc
- MR : `glab mr create` | `gh pr create` | push options GitLab
- hotfix : …

## sources
- tracker : … ; maquettes : …

## task_dir
`.dev/tasks/`

## security
auto
- règles propres au projet : …

## models
(vide = défauts de /dev)

## extra
### Étape 1 — …
### Mode bug — …
```

`vcs.md` : format de message (conventional commits ou autre, langue, longueur, ticket), commande de commit sûre (`git add -- <fichiers>`, `git commit … -- <fichiers>`, `git show --stat HEAD`), jamais `--amend` ni `--no-verify`, politique `fixup!`, aucune trace de Claude, jamais de push sans oui.

`coding-assertions.md` : architecture et couches, conventions de code, tests (seams, pas de test tautologique ni couplé), hygiène, section « Pièges connus » vide.

## 4. Fin

Afficher les trois chemins et les sections laissées vides. Proposer de lancer `/dev-suite:dev` sur un premier ticket.
