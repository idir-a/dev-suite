---
name: dev
description: Workflow de développement complet et réutilisable — feature, bug, epic ou POC — d'un ticket, d'une maquette ou d'une description jusqu'au commit local, piloté par le profil du dépôt (~/.claude/dev-profiles/). Utiliser quand l'utilisateur lance /dev-suite:dev (ou /dev) ou demande de développer une feature, une epic, un POC ou de corriger un bug dans un dépôt qui a un profil.
argument-hint: <ticket | poc <ticket|texte> | URL maquette | description> [profil=<nom>] [sécu]
---

# /dev

Orchestrateur autonome : spec, plan et présentation des revues restent dans la session ; le code est délégué à `dev-suite:dev-executor`, les revues à `dev-suite:dev-checker`, `dev-suite:dev-reviewer` et `dev-suite:dev-security`. Trois pauses pour l'accord de l'utilisateur : après le spec, le plan et la review.

## Profil (en tout premier)

1. Résoudre le profil : argument `profil=<nom>`, sinon le dossier de `~/.claude/dev-profiles/*/profile.md` dont `match` contient `git remote get-url origin` ou le nom du dossier racine du dépôt.
2. Aucun profil → proposer `/dev-suite:dev-init` et s'arrêter.
3. Lire `profile.md`, puis `vcs.md` et `coding-assertions.md` du même dossier. Dans la suite, `profil § X` renvoie à la section X de `profile.md` ; une section absente → le défaut écrit ici.
4. Chaque section `extra` du profil s'applique à l'étape ou au mode qu'elle nomme, en plus de ce fichier ; en cas de contradiction, le profil gagne.

## Règles non négociables

- Dossier de tâche `<task_dir>/<yyyy_mm>/<yyyy_mm_dd>_<ticket>-<slug>/` (défaut `task_dir` : `.dev/tasks/`), exclu par `.git/info/exclude`, **jamais commité**.
- Commit local uniquement : **aucun push, aucune MR/PR sans un oui explicite**, demandé à chaque fois.
- **Aucune trace de Claude** dans commits, MR/PR, commentaires du tracker.
- Jamais lire un `.env*` ; jamais exécuter une commande qui exige un token secret : la donner à l'utilisateur.
- Pas de worktree sans demande ; jamais d'outillage perso (scripts de vérif) dans le dépôt : scratchpad.
- **Modèles** (paramètre `model` de chaque appel `Agent`) : `dev-suite:dev-executor` → `sonnet` (`opus` si le profil le dit pour ce type de travail) ; `dev-suite:dev-checker` et `dev-suite:dev-security` → `opus` ; `dev-suite:dev-reviewer` → `sonnet`, `opus` en revue finale d'epic. Jamais `haiku`. Surcharges : profil § models.

## Choisir le mode

| Mode | Déclencheur |
|---|---|
| Feature | défaut (story, tâche, maquette, texte) |
| Bug | type Bug, ou description d'un dysfonctionnement |
| Epic | type Epic |
| POC | argument `poc`, type Spike, ou « POC » dans le résumé/labels |

## Format des pauses et rapports

- Chaque pause commence par l'état (« Étape 2/5 — plan. À valider : … ») puis la décision attendue ; une liste de tâches (une par étape, une seule en cours) tient la progression.
- Au plus 5 décisions par pause, les plus bloquantes d'abord ; le reste en « pour info » ; le détail reste dans les fichiers du dossier de tâche, cités par chemin.
- Review : gates passées en une ligne, puis les findings à trancher ; un écart hors périmètre va dans « Points remontés ».
- Rapport final : actions utilisateur numérotées avec estimation concrète (« jouer la migration : ~2 min »), puis la seule question push + MR.

## 0. Préflight

0. Titre de session dès le ticket lu (`mcp__ccd_session_mgmt__set_session_title`, `session_id: "self"`, si l'outil existe) : `<ticket> <ce qui va être fait>`.
   Lancé en plan mode : collecte en lecture seule (étapes 1 et 2) d'abord, préflight git une fois le plan validé.
1. `git fetch <remote> <base>` (profil § vcs ; défaut `origin`, branche par défaut du dépôt).
   - **Travail déjà entamé ?** `git ls-remote --heads <remote>` filtré sur le(s) ticket(s), plus statut et assigné du tracker. Branche existante ou autre dev assigné → le signaler et demander la base. Branche locale et distante divergentes → `git diff <local> <distant> -- <chemins>` avant de choisir.
2. Branche simple par défaut : `git status --short` vide (sinon stop et question), puis `git switch --no-track -c <prefix>/<ticket>-<slug> <base>`. Worktree seulement si l'utilisateur le demande : **hors du dépôt** (`git worktree add ../<repo>-wt/<slug> --no-track -b <branche> <base>`), install des dépendances, `.env*` copiés par l'utilisateur.
   `<prefix>` : `feat`, `fix`, `poc` (profil § vcs pour d'autres conventions).
3. `task_dir` présent dans `.git/info/exclude` (`git rev-parse --git-common-dir` pour un worktree). Si le profil demande un symlink de mémoire (profil § extra), le recréer s'il manque.
4. Baseline dans le scratchpad : sortie de la commande typecheck et liste des tests en échec (profil § commands). On compare, on n'exige pas le vert absolu.

## 1. Sources → spec

- **Tracker** (profil § sources) : description, **commentaires**, tous les champs, rendu HTML (liens derrière un texte), liens distants. Extraire objectif, critères d'acceptation, type (→ mode).
- **Maquettes** : collecter **toutes** les URLs de maquette (description, chaque commentaire, champs, liens distants ; epic **et** enfants), les dédupliquer, afficher la liste avec la source de chaque lien ou dire où on a cherché. Plusieurs liens pour un écran → demander lequel fait foi. Lien illisible → le signaler et s'arrêter : pas d'UI « à l'œil ». Figma : charger `figma:figma-design-to-code`, puis `get_design_context` + `get_screenshot` par nœud ; garder les screenshots pour l'étape 4.
- **Texte libre** : tel quel.
- **Grilling** : `dev-suite:dev-grill` sur le brief consolidé, jusqu'à ce qu'aucune décision ne reste ouverte.
- **Spec** dans le dossier de tâche : objectif, AC, décisions du grilling, hors périmètre, section « Points remontés » (alimentée jusqu'à la fin).

⏸ Pause : accord sur le spec.

## 2. Plan

Phases ordonnées, fichiers touchés, réutilisation de l'existant, impacts listés par le profil (§ extra), docs à mettre à jour. Toute omission vs ticket ou maquette est listée et soumise.

**Seams de test** : les interfaces publiques testées, au plus haut niveau possible, validées à cette pause. Rien à tester (pure mise en page) → l'écrire.

⏸ Pause : accord sur le plan.

## 3. Implémentation

Noter `<sha avant>` (`git rev-parse HEAD`), puis agent `dev-suite:dev-executor` avec en brief : chemin du plan et de la phase, chemins de `vcs.md` et `coding-assertions.md` du profil, commandes du profil, seams validés, branche déjà posée. Correctifs issus d'une review : `--fixup=<sha visé>` si le profil l'autorise (vcs.md).

Tout point ouvert (décision produit/design, limite connue, contenu à corriger, ordre de livraison) → « Points remontés » du spec.

## 4. Review + gates

Revues en parallèle, lecture seule (seul `dev-suite:dev-executor` corrige), sur `<sha avant>..HEAD` :

- **Conformité** : `dev-suite:dev-checker` — spec, plan, AC, screenshots de maquette.
- **Qualité** : `dev-suite:dev-reviewer` — `coding-assertions.md` comme référentiel, **sans** spec ni plan (regard non ancré).
- **Sécurité** (optionnel) : `dev-suite:dev-security` selon profil § security — `off` ; `auto` (défaut) : seulement si le diff touche une zone sensible (entrées utilisateur/formulaires, appels API et en-têtes, auth/session/cookies, `postMessage`/iframes/allowlists d'origine, rendu HTML brut, dépendances/lockfile, CI/déploiement, env/secrets) ; `always` ; ou demande explicite (« sécu »).

Présentation en sections séparées **Conformité**, **Qualité**, **Sécurité**, sans fusion ni reclassement entre elles, une ligne de synthèse par axe. Puis les gates du profil (§ gates), toutes exigées. Défaut si le profil n'en liste pas :

1. typecheck : 0 erreur ;
2. tests : aucun nouvel échec vs baseline ;
3. build(s) OK (séquentiels si le profil le dit) ;
4. vérif visuelle vs screenshots de maquette s'il y a une UI : rendu, DOM, comportement, textes ; jamais « pixel-perfect » sans l'avoir vérifié ; un état (survol, clic) se mesure après sa transition ;
5. docs à jour si un comportement documenté change.

Gate KO, finding non écarté (avec raison) par l'utilisateur, ou finding sécurité critique/haut → retour en 3 avec la liste. Revues relancées sur le diff des correctifs.

⏸ Pause : accord sur la review.

## 5. Fin

Rapport : `git log --oneline <base>..HEAD`, fichiers touchés, actions à faire par l'utilisateur (migrations, contenu) avec commande exacte et ordre de livraison, points non vérifiés, « Points remontés ». Puis une seule question : push + MR ?

`fixup!` restants : proposer d'abord (question séparée) `GIT_SEQUENCE_EDITOR=: git rebase -i --autosquash <base>`, seulement si la branche n'a jamais été poussée, puis relancer typecheck + tests ; preuve : `git diff <sha avant rebase> HEAD` vide. Branche déjà poussée : commits normaux, jamais de force push.

Sur oui : création de MR/PR selon profil § vcs (`glab`, `gh`, ou push options GitLab sur une seule ligne). Titre et description relus : aucune trace de Claude.

## Mode bug

1. **Base** : demander la branche de base avec une recommandation tirée du ticket ; branches de hotfix et remotes : profil § vcs.
2. **Diagnostic** (avant le spec) : `dev-suite:dev-diagnose`.
3. **Spec** : scénario, constaté, attendu + cause racine (file:line) + périmètre partagé.
4. **Plan** : phase 1 = test de non-régression qui **échoue sur le code actuel** ; puis le correctif à la cause racine.
5. **Implement** : `dev-suite:dev-executor` rapporte la sortie rouge avant le fix, puis vert ; l'hypothèse confirmée va dans le corps du commit.
6. **Review** : gates + « le scénario ne reproduit plus » ; `dev-suite:dev-checker` fait le mutation check (le test échoue sans le fix).
7. **Fin** : MR vers la base ; report/cherry-pick vers les autres branches selon profil § vcs, sur oui.

## Mode epic

1. Tickets enfants via le tracker ; à défaut, demander la liste.
2. Collecte des maquettes sur l'epic et chaque enfant.
3. Plan d'epic sans code : `<task_dir>/…/<epic>-<slug>/epic.md` = tickets, ordre, dépendances, fichiers touchés par ticket ; spec/plan de chaque ticket dans un sous-dossier.
4. ⏸ Pause : accord + choix **epic page**, **séquentiel** ou **parallèle** (recommander « epic page » quand ses critères sont remplis) + specs **en bloc** ou **ticket par ticket**.
5. Branche d'epic `feat/<epic>-<slug>`.
6. Séquentiel : étapes 1 à 4 ticket par ticket sur la branche d'epic (Mode bug pour un enfant Bug).
7. Parallèle : deux sessions ne committent **jamais** sur la même branche ; chaque ticket indépendant dans sa session + worktree hors dépôt, sous-branche `feat/<epic>-<slug>--<ticket>` ; deux tickets qui touchent les mêmes fichiers jamais en même temps ; ticket fini → merge local `--no-ff` dans la branche d'epic sur accord.
8. Review finale sur tout le diff de l'epic : `dev-suite:dev-checker` contre `epic.md`, `dev-suite:dev-reviewer` en `opus` (duplication et incohérences entre tickets), `dev-suite:dev-security` selon le profil, gates ; puis étape 5.

Commits : référencer le **ticket enfant**.

### Variante « epic page »

Critères (tous requis) : une seule page et une seule maquette ; les tickets touchent les mêmes fichiers ; tous prêts pour le dev (un ticket bloqué sort du lot) ; personne d'autre dessus. Sinon → séquentiel.

1. Un spec pour la page : sections dans l'ordre de la maquette, table « AC → ticket », relations entre sections au niveau page. ⏸
2. Un plan : une phase par ticket (ordre des dépendances), nettoyage en dernière phase. ⏸
3. `dev-suite:dev-executor` enchaîne les phases, **un commit par ticket enfant** (+ nettoyage référencé sur l'epic).
4. Review une fois sur tout le diff, vérif visuelle de la page entière (desktop + mobile). ⏸
5. Étape 5 avec la table « AC → ticket → commit ».

## Mode POC

1. Branche `poc/<ticket>-<slug>` (ou `poc/<slug>`).
2. Spec : l'hypothèse + critères de succès mesurables. Plan : le minimum qui prouve ou réfute.
3. Relâché : tests seulement ceux qui prouvent l'hypothèse (la gate « aucun nouvel échec » reste) ; `dev-suite:dev-reviewer` limité aux bugs avérés ; restrictions supplémentaires : profil § extra.
4. Inchangé : les 3 pauses, typecheck à 0, vérif visuelle si maquette, commits conventionnels.
5. Rapport `<task_dir>/…/poc-report.md` + résumé : hypothèse, verdict (validée / invalidée / partielle) avec preuves, raccourcis pris, ce qu'il faut pour la vraie feature avec estimation.
6. **Jamais de MR vers la branche principale** ; push de la branche `poc/` seulement sur oui.
