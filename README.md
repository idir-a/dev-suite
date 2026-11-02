# dev-suite

Plugin Claude Code : un workflow de développement complet — **feature, bug, epic ou POC** — d'un ticket, d'une maquette ou d'une description jusqu'au **commit local**. Chaque dépôt n'apporte qu'un **profil** (commandes, gates, conventions, spécificités).

Documentation complète du workflow : [`docs/workflow.md`](docs/workflow.md).

## Installation

```text
/plugin marketplace add idir-a/dev-suite
/plugin install dev-suite@dev-suite
```

Redémarrer la session pour charger les agents.

## Démarrer sur un dépôt

1. Dans le dépôt : `/dev-suite:dev-init` — lit les scripts, commits et CI, pose les questions restantes, écrit `~/.claude/dev-profiles/<nom>/`. Rien n'est écrit dans le dépôt.
2. `/dev-suite:dev <ticket | URL maquette | description>` — ajouter `poc` pour un POC, `sécu` pour forcer l'audit sécurité.

## Contenu

| Élément | Rôle |
|---|---|
| `dev` (skill) | Orchestrateur : préflight → spec → plan → implémentation → revues + gates → rapport. 3 pauses pour accord. |
| `dev-grill` (skill) | Rounds de ≤ 5 questions avec réponse recommandée ; les faits sont cherchés, pas demandés. |
| `dev-diagnose` (skill) | Bug : commande de repro d'abord, 3–5 hypothèses falsifiables, logs `[DEBUG-xxxx]`, test de non-régression. |
| `dev-init` (skill) | Crée le profil d'un dépôt. |
| `dev-executor` (agent, sonnet) | Code + commits, TDD aux seams validés. Ne pousse jamais. |
| `dev-checker` (agent, opus) | Conformité au spec/plan/AC/maquette, mutation check en mode bug. Lecture seule. |
| `dev-reviewer` (agent, sonnet) | Qualité, sans spec ni plan (regard non ancré). Lecture seule. |
| `dev-security` (agent, opus) | Audit sécurité du diff, optionnel (`security` : `off`, `auto` ou `always`). Lecture seule. |

## Profil

`~/.claude/dev-profiles/<nom>/` :

- `profile.md` — sections `match`, `commands`, `gates`, `vcs`, `sources`, `task_dir`, `security`, `models`, `extra` (règles par étape ou par mode) ;
- `vcs.md` — convention de commit ;
- `coding-assertions.md` — règles de code et « Pièges connus ».

Exemple complet : [`profiles/example/`](profiles/example/).

## Garde-fous

- Aucun push, aucune MR/PR sans un oui explicite.
- Aucune trace d'assistant IA dans commits et MR.
- Jamais de lecture de `.env*`, jamais de commande avec un token secret.
- Dossier de tâche (spec, plan, rapports) exclu de git via `.git/info/exclude`.
