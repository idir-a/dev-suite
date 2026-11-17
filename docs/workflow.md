# Le workflow dev-suite

Ce document décrit tout ce qui se passe entre `/dev-suite:dev <ticket>` et le commit local : les étapes, les pauses, les modes, les agents, les revues, les gates et le profil qui adapte le tout à chaque dépôt.

## Sommaire

1. [Vue d'ensemble](#1-vue-densemble)
2. [Démarrage : résolution du profil](#2-démarrage--résolution-du-profil)
3. [Les étapes 0 à 6](#3-les-étapes-0-à-6)
4. [Les pauses](#4-les-pauses)
5. [Les modes : feature, bug, epic, POC, retour de recette](#5-les-modes)
6. [Les agents](#6-les-agents)
7. [Les revues et les gates](#7-les-revues-et-les-gates)
8. [Le dossier de tâche](#8-le-dossier-de-tâche)
9. [Le profil](#9-le-profil)
10. [Garde-fous](#10-garde-fous)
11. [Dépannage](#11-dépannage)
12. [Apprendre d'un run : la rétro](#12-apprendre-dun-run--la-rétro)

---

## 1. Vue d'ensemble

```mermaid
flowchart TD
    A["/dev-suite:dev &lt;ticket | maquette | texte&gt;"] --> P{Profil du dépôt ?}
    P -- non --> I["/dev-suite:dev-init"] --> A
    P -- oui --> M{Mode}
    M --> S0["0. Préflight<br/>branche, baseline"]
    S0 --> S1["1. Sources → grilling → spec"]
    S1 --> G1(("⏸ spec"))
    G1 --> S2["2. Plan + seams de test"]
    S2 --> G2(("⏸ plan"))
    G2 --> S3["3. Implémentation<br/>dev-executor"]
    S3 --> S4["4. Revues ∥ + gates"]
    S4 --> G3(("⏸ review"))
    G3 -- "gate KO / finding retenu" --> S3
    G3 -- accord --> S5["5. Rapport → push + MR ?"]
    S5 --> S6["6. Rétro<br/>leçons proposées, appliquées sur accord"]
```

Principes :

- **La session décide, les agents exécutent.** Le spec, le plan, le diagnostic et la présentation des revues restent dans la session principale. Le code est écrit par `dev-executor` ; les revues sont faites par des agents en contexte neuf.
- **Trois pauses obligatoires** : après le spec, après le plan, après la review. Rien n'avance sans ton accord. Un petit ticket peut passer en [voie courte](#voie-courte) : une seule pause avant le code.
- **Chaque run apprend.** Les frictions sont notées dans `journal.md` pendant le run ; la rétro de fin les transforme en propositions de règles, appliquées seulement sur ton accord.
- **Tout finit en commit local.** Push et MR/PR uniquement sur un oui explicite, demandé à chaque fois.
- **Un seul workflow, un profil par dépôt.** Le générique vit dans le plugin ; les commandes, conventions et pièges du projet vivent dans `~/.claude/dev-profiles/<nom>/`.

## 2. Démarrage : résolution du profil

Au lancement, `dev` cherche le profil dans cet ordre :

1. argument explicite `profil=<nom>` ;
2. un `~/.claude/dev-profiles/*/profile.md` dont la section `match` contient l'URL de `git remote get-url origin` ;
3. … ou le nom du dossier racine du dépôt.

Aucun profil trouvé → `dev` propose `/dev-suite:dev-init` et s'arrête.

```mermaid
flowchart TD
    A["Lancement de dev"] --> B{"Argument profil=nom ?"}
    B -- oui --> L["Charger ~/.claude/dev-profiles/nom/"]
    B -- non --> C{"match contient l'URL du remote origin ?"}
    C -- oui --> L
    C -- non --> D{"match contient le nom du dossier racine ?"}
    D -- oui --> L
    D -- non --> E["Proposer /dev-suite:dev-init puis arrêt"]
    L --> F["Lire profile.md, vcs.md, coding-assertions.md"]
    F --> G["Appliquer les sections extra aux étapes et modes nommés"]
```

Une fois trouvé, il lit `profile.md`, `vcs.md` et `coding-assertions.md`. Les règles du profil (section `extra`) s'ajoutent à l'étape ou au mode qu'elles nomment ; en cas de contradiction, **le profil gagne**.

### Créer un profil avec `dev-init`

1. Lecture du dépôt sans rien demander : remotes, branche par défaut, format des 30 derniers commits, scripts (`package.json`, `Makefile`…), commitlint et hooks, `CLAUDE.md`, `CONTRIBUTING.md`, CI, outil de MR installé (`gh`, `glab`).
2. Questions (`dev-grill`) uniquement sur ce qui ne se déduit pas : tracker, outil de maquette, base des hotfix, gates, niveau de sécurité, spécificités.
3. Écriture de `~/.claude/dev-profiles/<nom>/` (3 fichiers). **Rien n'est écrit dans le dépôt.**

```mermaid
flowchart LR
    R["Dépôt<br/>remotes, scripts, commits,<br/>hooks, CLAUDE.md, CI"] -->|lecture seule| I["dev-init"]
    I -->|"questions restantes (dev-grill)"| U(("Toi"))
    U --> I
    I --> P1["profile.md"]
    I --> P2["vcs.md"]
    I --> P3["coding-assertions.md"]
    subgraph H["~/.claude/dev-profiles/nom/ (hors dépôt)"]
        P1
        P2
        P3
    end
```

## 3. Les étapes 0 à 6

### Étape 0 — Préflight

| Action | Détail |
|---|---|
| Titre de session | `<ticket> <ce qui va être fait>`, si l'outil de titre existe. |
| Reprise ? | Un dossier de tâche existe pour ce ticket → l'étape atteinte est lue dans l'en-tête de `journal.md` et `git log`, et le run reprend là (ou bascule en mode retour de recette si le travail était livré). |
| Travail déjà entamé ? | `git ls-remote --heads` filtré sur le ticket + statut et assigné du tracker. Branche existante ou autre dev assigné → question sur la base. |
| Branche | Branche simple par défaut (`git status` doit être vide), `<prefix>/<ticket>-<slug>` depuis la base. Worktree **seulement sur demande**, toujours hors du dépôt. |
| Dossier de tâche | Ajouté à `.git/info/exclude`. |
| Baseline | Sortie du typecheck et liste des tests en échec, dans le scratchpad. Les gates comparent à cette baseline : on n'exige pas un vert absolu préexistant. |

Lancé en plan mode, `dev` fait d'abord la collecte en lecture seule (étapes 1 et 2), puis le préflight git une fois le plan validé.

```mermaid
flowchart TD
    A["git fetch remote base"] --> B{"Branche du ticket existe<br/>ou autre dev assigné ?"}
    B -- oui --> Q["Question : partir de sa branche ou de la base ?"]
    B -- non --> W
    Q --> W{"Worktree demandé ?"}
    W -- non --> S{"git status vide ?"}
    S -- non --> X["Arrêt et question"]
    S -- oui --> BR["git switch --no-track -c prefix/ticket-slug base"]
    W -- oui --> WT["git worktree add hors du dépôt<br/>+ install des dépendances<br/>+ .env copiés par toi"]
    BR --> EX["task_dir dans .git/info/exclude"]
    WT --> EX
    EX --> BL["Baseline typecheck + tests en échec<br/>dans le scratchpad"]
```

### Étape 1 — Sources → spec

1. **Tracker** : description, **tous les commentaires**, tous les champs, rendu HTML (pour les liens cachés derrière un texte), liens distants. On en tire l'objectif, les critères d'acceptation et le type (qui fixe le mode).
2. **Maquettes** : toutes les URLs collectées partout (epic et enfants compris), dédupliquées, affichées avec leur source. Plusieurs liens pour un écran → question. Lien illisible → arrêt : pas d'UI implémentée « à l'œil ». Les screenshots servent de référence à l'étape 4.
3. **Grilling** (`dev-grill`) : questions par rounds de 5 maximum, chacune avec une réponse recommandée. Les faits (code, config, maquette) sont cherchés par Claude, jamais demandés. Fin quand aucune décision ne reste ouverte.
4. **Spec** écrit dans le dossier de tâche : objectif, AC, décisions, hors périmètre, « Points remontés ».

```mermaid
flowchart TD
    T["Tracker : description, commentaires,<br/>champs, rendu HTML, liens distants"] --> BR["Brief consolidé"]
    MQ["Maquettes : toutes les URLs,<br/>dédupliquées, source affichée"] --> BR
    TX["Texte libre"] --> BR
    MQ -.->|lien illisible| STOP["Arrêt : pas d'UI à l'œil"]
    BR --> GR["dev-grill"]
    GR --> SP["spec.md"]
    SP --> P(("⏸ Pause 1"))
```

Le grilling, round par round :

```mermaid
flowchart TD
    A["Arbre des décisions"] --> F["Frontière = décisions dont<br/>les prérequis sont tranchés"]
    F --> N{"Frontière vide ?"}
    N -- non --> R["Round : 5 questions max,<br/>chacune avec réponse recommandée"]
    R --> FA["Faits cherchés par Claude<br/>(code, config, maquette)"]
    R --> AN["Tes réponses"]
    FA --> F
    AN --> F
    N -- oui --> C["Récapitulatif + confirmation"]
```

⏸ **Pause 1 : accord sur le spec.**

### Étape 2 — Plan

- Phases ordonnées, fichiers touchés, réutilisation de l'existant, impacts listés par le profil, docs à mettre à jour.
- Toute omission par rapport au ticket ou à la maquette est listée et soumise.
- **Ajouts hors ticket** : toute fonction, variante, prop, composant ou icône que le ticket ne demande pas est justifiée par un critère d'acceptation, ou listée et soumise ; par défaut, elle n'est pas ajoutée.
- **Composant partagé** : un changement de comportement d'un composant ou util utilisé ailleurs est annoncé, avec ses autres usages et l'effet attendu sur chacun.
- **Preuve de lecture des maquettes** : table « écran → nœud de maquette lu → screenshot ». Une ligne sans screenshot = maquette non lue.
- **Seams de test** : les interfaces publiques qui seront testées, au plus haut niveau possible. Si rien n'est à tester (mise en page pure), c'est écrit.

⏸ **Pause 2 : accord sur le plan.**

### Étape 3 — Implémentation

`dev` note le sha courant (`<sha avant>`), puis lance `dev-executor` avec : le plan et la phase, les chemins de `vcs.md` et `coding-assertions.md`, les commandes du profil, les seams validés.

L'executor travaille en **TDD par tranches verticales** : un test rouge → le code minimal → le test suivant. Typecheck et test ciblé après chaque tranche, suite complète à la fin, un commit par unité logique selon `vcs.md`.

```mermaid
flowchart TD
    A["Lire Pièges connus, vcs.md,<br/>coding-assertions.md, plan"] --> B["Seam suivant"]
    B --> C["Écrire un test"]
    C --> D{"Rouge ?"}
    D -- "non (test inutile)" --> C
    D -- oui --> E["Code minimal"]
    E --> F["typecheck + test ciblé"]
    F -- rouge --> G["Corriger la cause"] --> F
    F -- vert --> H{"Autre seam dans la phase ?"}
    H -- oui --> B
    H -- non --> I["Suite complète"]
    I --> J["Commit selon vcs.md"]
    J --> K["Rapport : commits, sorties brutes, points ouverts"]
```

Tout point ouvert (décision produit, limite, contenu à corriger, ordre de livraison) est ajouté à « Points remontés » au fil de l'eau.

### Étape 4 — Revues + gates

Trois revues **en parallèle**, en lecture seule, sur `<sha avant>..HEAD` (voir [§7](#7-les-revues-et-les-gates)), puis les gates du profil.

Gate KO, finding retenu, ou finding sécurité critique/haut → retour à l'étape 3 avec la liste ; les revues sont relancées sur le diff des correctifs.

```mermaid
stateDiagram-v2
    [*] --> Implementation
    Implementation --> Revues: commits posés
    Revues --> Gates
    Gates --> Pause3
    Pause3 --> Implementation: gate KO ou finding retenu
    Pause3 --> Rapport: accord
    Rapport --> [*]: commit local
    Rapport --> Push: oui explicite
    Push --> [*]
```

⏸ **Pause 3 : accord sur la review.**

### Étape 5 — Fin

Rapport :

- commits (`git log --oneline <base>..HEAD`), fichiers touchés ;
- actions à faire par toi (migrations, contenu…), numérotées, avec la commande exacte et une estimation ;
- ordre de livraison, points non vérifiés, « Points remontés ».

Puis **une seule question : push + MR ?**

```mermaid
flowchart TD
    A["Rapport"] --> B{"Commits fixup! restants ?"}
    B -- non --> Q
    B -- oui --> C{"Branche déjà poussée ?"}
    C -- oui --> Q
    C -- non --> D{"Fondre les fixup ? (question séparée)"}
    D -- non --> Q
    D -- oui --> E["rebase --autosquash<br/>+ typecheck + tests<br/>+ preuve : arbre identique"] --> Q
    Q{"Push + MR ?"} -- non --> L["Reste en local"]
    Q -- oui --> M["Push + MR selon profil vcs<br/>titre et description relus"]
```

S'il reste des commits `fixup!`, une question séparée propose d'abord `git rebase -i --autosquash <base>` — uniquement si la branche n'a jamais été poussée — avec relance du typecheck et des tests, et preuve que l'arbre final est identique. Branche déjà poussée : jamais de force push.

### Étape 6 — Rétro

Après ta réponse à la question push + MR (oui ou non), `dev-retro` relit le run et propose au plus 5 règles. Détail : [section 12](#12-apprendre-dun-run--la-rétro).

## 4. Les pauses

Format fixe pour qu'elles se lisent vite :

- **Première ligne = l'état** : « Étape 2/5 — plan. À valider : … ».
- **5 décisions au maximum**, les plus bloquantes d'abord ; le reste en « pour info ». Le détail reste dans les fichiers du dossier de tâche, cités par chemin.
- **Review** : les gates passées en une ligne, puis seulement les findings à trancher.
- **Rapport final** : actions numérotées avec durée estimée, puis la question push + MR.

Une liste de tâches (une par étape, rétro comprise, une seule en cours) affiche la progression.

## 5. Les modes

| Mode | Déclencheur |
|---|---|
| Feature | par défaut (story, tâche, maquette, texte) |
| Bug | ticket de type Bug, ou description d'un dysfonctionnement |
| Epic | ticket de type Epic |
| POC | argument `poc`, type Spike, ou « POC » dans le résumé ou les labels |
| Retour de recette | ticket déjà traité (dossier de tâche ou commits existants), ou tableau/commentaires de recette en entrée |

```mermaid
flowchart TD
    A["Arguments + ticket"] --> B{"Argument poc, type Spike<br/>ou POC dans résumé/labels ?"}
    B -- oui --> POC["Mode POC"]
    B -- non --> C{"Type Epic ?"}
    C -- oui --> EP["Mode epic"]
    C -- non --> D{"Type Bug ou<br/>dysfonctionnement décrit ?"}
    D -- oui --> BUG["Mode bug"]
    D -- non --> FT["Mode feature"]
```

### Voie courte

Pour un petit ticket, quel que soit le mode. Critères, tous requis : diff attendu petit (≈ 30 lignes hors tests, 1 ou 2 fichiers), pas de migration, pas de nouvelle maquette à intégrer, pas de changement de comportement d'un composant partagé. `dev` la propose à l'étape 1, tu confirmes.

| | Voie normale | Voie courte |
|---|---|---|
| Documents | `spec.md` + `plan.md` | un seul `spec-plan.md` |
| Pauses | spec, plan, review | une avant le code ; review seulement s'il y a un finding ou un gate KO |
| Revues | `dev-checker` + `dev-reviewer` | `dev-checker` seul, avec `coding-assertions.md` |
| Gates | toutes | typecheck et tests toujours ; un gate marqué `si <condition>` dans le profil est sauté si la condition est fausse, et déclaré « non fait » |

Si le diff réel dépasse les critères, le run repasse en voie normale et l'annonce.

### Mode bug

```mermaid
flowchart LR
    B0["Base ?<br/>(main, release, hotfix)"] --> D["dev-diagnose"]
    D --> SP["Spec : constaté / attendu<br/>+ cause racine file:line"]
    SP --> PL["Plan : 1. test qui échoue<br/>2. fix à la cause racine"]
    PL --> IM["executor : rouge → fix → vert"]
    IM --> RV["Revues + mutation check<br/>+ scénario ne reproduit plus"]
```

Le diagnostic (`dev-diagnose`) :

1. **Boucle de feedback d'abord** : une commande unique, déjà exécutée, qui reproduit le symptôme exact — rapide, déterministe, lançable sans humain. Pas d'hypothèse avant.
2. **Réduction** du scénario jusqu'au minimum qui reproduit.
3. **3 à 5 hypothèses** classées et falsifiables, montrées avant d'être testées.
4. **Instrumentation** ciblée, une variable à la fois, logs préfixés `[DEBUG-xxxx]`.
5. **Cause racine vérifiée** dans le fichier réel, et liste de tous les appelants : le correctif va dans le code partagé.
6. **Test de non-régression avant le fix**, au bon seam. Pas de seam correct → c'est signalé comme finding, pas de test factice.
7. **Nettoyage** : `grep DEBUG-xxxx` vide, hypothèse confirmée dans le corps du commit.

```mermaid
flowchart TD
    A["Construire une commande de repro"] --> B{"Rouge sur ce symptôme,<br/>déterministe, rapide ?"}
    B -- non --> C{"Encore une piste ?<br/>test, curl, headless,<br/>replay, bisect…"}
    C -- oui --> A
    C -- non --> X["Arrêt : demander accès,<br/>artefact ou instrumentation"]
    B -- oui --> D["Réduire le scénario"]
    D --> E["3 à 5 hypothèses falsifiables<br/>montrées avant test"]
    E --> F["Sonde DEBUG-xxxx<br/>une variable à la fois"]
    F --> G{"Hypothèse confirmée ?"}
    G -- non --> E
    G -- oui --> H["Cause racine file:line<br/>+ tous les appelants"]
    H --> I{"Seam correct pour<br/>le test de non-régression ?"}
    I -- non --> J["Finding : seam absent"]
    I -- oui --> K["Test rouge → fix → vert"]
    J --> L
    K --> L["Boucle d'origine verte,<br/>grep DEBUG vide, hypothèse dans le commit"]
```

Mutation check du checker :

```mermaid
sequenceDiagram
    participant C as dev-checker
    participant W as Working tree
    participant T as Test de non-régression
    C->>W: git checkout sha-avant -- fichiers du fix
    C->>T: lancer
    T-->>C: doit être ROUGE
    C->>W: git checkout HEAD -- mêmes fichiers
    C->>W: git status identique à l'entrée ?
```

En review, `dev-checker` fait un **mutation check** : il remet le code fautif, vérifie que le test échoue, puis restaure.

### Mode epic

1. Tickets enfants via le tracker (ou liste demandée).
2. Collecte des maquettes sur l'epic et chaque enfant.
3. `epic.md` : tickets, ordre, dépendances, fichiers touchés par ticket.
4. ⏸ Choix d'exécution + specs en bloc ou ticket par ticket.

```mermaid
flowchart TD
    A["epic.md validé"] --> B{"Une page, une maquette,<br/>mêmes fichiers, tous prêts,<br/>personne d'autre dessus ?"}
    B -- oui --> EP["Epic page"]
    B -- non --> C{"Tickets indépendants<br/>(fichiers disjoints) ?"}
    C -- oui --> PA["Parallèle"]
    C -- non --> SE["Séquentiel"]
```

Branches en mode parallèle :

```mermaid
gitGraph
    commit id: "base"
    branch "feat/epic"
    commit id: "epic.md (local)" type: HIGHLIGHT
    branch "feat/epic--T1"
    commit id: "T1"
    checkout "feat/epic"
    branch "feat/epic--T2"
    commit id: "T2"
    checkout "feat/epic"
    merge "feat/epic--T1" id: "merge T1"
    merge "feat/epic--T2" id: "merge T2"
    branch "feat/epic--T3"
    commit id: "T3 (dépend de T1)"
    checkout "feat/epic"
    merge "feat/epic--T3" id: "merge T3"
    commit id: "review finale"
```

| Exécution | Quand | Comment |
|---|---|---|
| **Epic page** | une page, une maquette, mêmes fichiers, tous les tickets prêts, personne d'autre dessus | un spec et un plan pour toute la page, une phase et un commit par ticket enfant, une review sur tout le diff |
| **Séquentiel** | par défaut | étapes 1 à 4 ticket par ticket sur la branche d'epic |
| **Parallèle** | tickets indépendants | une session + un worktree hors dépôt par ticket, sous-branche `feat/<epic>-<slug>--<ticket>`, merge `--no-ff` dans la branche d'epic sur accord ; deux tickets qui touchent les mêmes fichiers ne tournent jamais en même temps |

Review finale sur toute l'epic : `dev-checker` contre `epic.md`, `dev-reviewer` en opus (duplication et incohérences entre tickets), `dev-security` selon le profil. Les commits référencent le **ticket enfant**.

En séquentiel, une fois le plan d'epic et les specs validés en bloc, **les tickets s'enchaînent sans pause** : arrêt seulement sur une décision ouverte, un gate KO ou non faisable, ou un finding à trancher.

### Mode retour de recette

Pour des défauts remontés sur un travail déjà livré.

1. Les défauts sont listés un par un, avec leur source.
2. Chacun est **rejoué sur la base à jour** ; un défaut déjà corrigé est écarté avec la preuve.
3. Le dossier de tâche existant est repris (nouvelle section datée), pas recréé.
4. Mode bug pour chaque défaut reproduit, un commit par défaut.
5. Rapport : table « défaut → reproduit ? → commit ou raison de l'écart ».

### Mode POC

- Spec = une hypothèse + des critères de succès mesurables. Plan = le minimum qui prouve ou réfute.
- Relâché : seulement les tests qui prouvent l'hypothèse (la gate « aucun nouvel échec » reste), `dev-reviewer` limité aux bugs avérés.
- Inchangé : les 3 pauses, typecheck à 0, vérif visuelle si maquette, commits conventionnels.
- Rapport `poc-report.md` : verdict (validée / invalidée / partielle) avec preuves, raccourcis pris, ce qu'il faut pour la vraie feature avec estimation.
- **Jamais de MR vers la branche principale** ; push de la branche `poc/` seulement sur oui.

## 6. Les agents

| Agent | Modèle | Écrit ? | Rôle |
|---|---|---|---|
| `dev-executor` | sonnet (opus si le profil le demande) | oui : code + commits | Implémente une phase. TDD aux seams validés. Ne crée pas de branche, ne pousse jamais, ne se déclare pas « terminé ». |
| `dev-checker` | opus | non (sauf mutation check, restauré) | Conformité : chaque AC, phase et élément de maquette marqué rempli / partiel / non rempli avec preuve. |
| `dev-reviewer` | sonnet (opus en review finale d'epic) | non | Qualité sans connaître le spec : bugs, règles du projet, réutilisation, code mort, smells. |
| `dev-security` | opus | non | Audit sécurité du diff, optionnel. |

Qui fait quoi sur un ticket :

```mermaid
sequenceDiagram
    actor U as Toi
    participant S as Session (dev)
    participant E as dev-executor
    participant C as dev-checker
    participant R as dev-reviewer
    participant X as dev-security
    S->>U: questions (dev-grill)
    U-->>S: décisions
    S->>U: ⏸ spec
    U-->>S: accord
    S->>U: ⏸ plan + seams
    U-->>S: accord
    S->>E: plan, phase, profil, seams
    E-->>S: commits + sorties brutes
    par revues en parallèle
        S->>C: diff + spec + plan + maquette
        S->>R: diff + coding-assertions (sans spec)
        S->>X: diff si zone sensible
    end
    C-->>S: conformité
    R-->>S: qualité
    X-->>S: sécurité
    S->>S: gates du profil
    S->>U: ⏸ review (3 sections)
    U-->>S: accord
    S->>U: rapport + push et MR ?
```

Les modèles sont fixés à chaque appel ; `haiku` n'est jamais utilisé. Le profil peut les surcharger (section `models`).

Pourquoi `dev-reviewer` ne voit pas le spec : un relecteur qui connaît l'intention lit le code à travers elle et laisse passer ce qui « devrait » marcher. Le regard non ancré attrape ce que le checker, ancré sur le spec, ne voit pas.

## 7. Les revues et les gates

### Trois axes, jamais fusionnés

| Axe | Agent | Référentiel |
|---|---|---|
| **Conformité** | `dev-checker` | spec, plan, AC, screenshots de maquette |
| **Qualité** | `dev-reviewer` | `coding-assertions.md` + baseline de smells |
| **Sécurité** | `dev-security` | OWASP Top 10 / ASVS appliqués au diff + règles du profil |

Les findings sont présentés en sections séparées, avec une ligne de synthèse par axe. Une fusion classerait tout sur une seule échelle et masquerait un axe par l'autre (un code propre qui fait la mauvaise chose, ou l'inverse).

### Quand l'audit sécurité tourne

Section `security` du profil :

- `off` : jamais (sauf demande explicite « sécu ») ;
- `auto` (défaut) : seulement si le diff touche une zone sensible — entrées utilisateur et formulaires, appels API et en-têtes, auth/session/cookies, `postMessage`/iframes/allowlists d'origine, rendu HTML brut, dépendances et lockfile, CI et déploiement, variables d'env et secrets ;
- `always` : à chaque review.

```mermaid
flowchart TD
    A["Étape 4"] --> B{"Demande explicite « sécu » ?"}
    B -- oui --> RUN["Lancer dev-security"]
    B -- non --> C{"security du profil"}
    C -- off --> SKIP["Pas d'audit"]
    C -- always --> RUN
    C -- auto --> D{"Le diff touche une zone sensible ?<br/>entrées, API, auth, postMessage,<br/>HTML brut, dépendances, CI, secrets"}
    D -- oui --> RUN
    D -- non --> SKIP
    RUN --> E{"Finding critique ou haut ?"}
    E -- oui --> BLK["Bloquant : retour étape 3"]
    E -- non --> OK["Section Sécurité de la review"]
```

Chaque finding a une sévérité, un `file:line`, un **scénario concret** et un correctif. Critique ou haute = bloquant.

### Gates

Celles du profil (section `gates`), toutes exigées. Par défaut :

1. typecheck : 0 erreur ;
2. tests : aucun nouvel échec vs baseline ;
3. build(s) OK (séquentiels si le profil le dit) ;
4. vérif visuelle vs maquette s'il y a une UI : rendu, DOM, comportement, textes — jamais « pixel-perfect » sans l'avoir vérifié ;
5. docs à jour si un comportement documenté change.

Une gate qui ne peut pas être faite (environnement manquant, contenu absent, maquette illisible) est **bloquante**, jamais validée par défaut : la pause ne propose que de la débloquer maintenant, ou de la reporter en action nommée dans le rapport final.

Un gate coûteux peut porter une condition dans le profil (`si le diff touche …`) : il est alors sauté en voie courte quand la condition est fausse, et déclaré « non fait ».

Le résultat des revues et des gates, puis ton arbitrage sur chaque finding (corrigé / écarté + raison), sont écrits dans `review.md`.

## 8. Le dossier de tâche

`<task_dir>/<yyyy_mm>/<yyyy_mm_dd>_<ticket>-<slug>/` (défaut `task_dir` : `.dev/tasks/`), exclu de git par `.git/info/exclude`, jamais commité.

| Fichier | Écrit à |
|---|---|
| `spec.md` (avec « Points remontés ») | étape 1, complété jusqu'à la fin |
| `plan.md` (avec les seams de test) | étape 2 |
| `spec-plan.md` | voie courte, à la place des deux précédents |
| `journal.md` (état du run + une ligne par friction) | tout le run |
| `review.md` (findings, gates, arbitrages) | étape 4 |
| `epic.md` + un sous-dossier par ticket | mode epic |
| `poc-report.md` | mode POC |

## 9. Le profil

`~/.claude/dev-profiles/<nom>/`, hors de tout dépôt. Exemple complet : [`profiles/example/`](../profiles/example/).

```mermaid
flowchart LR
    subgraph PL["Plugin dev-suite (générique)"]
        D["dev"]
        AG["agents dev-*"]
    end
    subgraph PR["Profil (par dépôt)"]
        P["profile.md<br/>commandes, gates, vcs,<br/>sources, security, extra"]
        V["vcs.md"]
        CA["coding-assertions.md<br/>+ Pièges connus"]
    end
    P -->|"étapes, gates, modes"| D
    V -->|"commits"| AG
    CA -->|"règles de code, référentiel qualité"| AG
    AG -.->|piège relevé en review| CA
```

### `profile.md`

| Section | Obligatoire | Contenu |
|---|---|---|
| `match` | oui | URL(s) du remote et/ou nom du dossier racine |
| `commands` | oui | typecheck, test ciblé, tests, build(s) (séquentiels ?), dev server |
| `gates` | non | liste ordonnée exigée avant le ship |
| `vcs` | non | remote de MR, base, préfixes de branche, création de MR (`gh`, `glab`, push options), hotfix et reports |
| `sources` | non | tracker (et comment le lire), outil de maquette |
| `task_dir` | non | défaut `.dev/tasks/` |
| `security` | non | `off`, `auto` ou `always` + règles propres au projet |
| `models` | non | surcharges de modèle par agent ou type de travail |
| `extra` | non | sous-sections `### Étape n — …` ou `### Mode … — …` ajoutées à l'étape ou au mode nommé |

### `vcs.md`

Format des messages (convention, langue, longueur, référence au ticket), commande de commit sûre (`git add -- <fichiers>`, `git commit … -- <fichiers>`, `git show --stat HEAD`), politique `fixup!`, interdits (`--amend`, `--no-verify`, push sans oui).

### `coding-assertions.md`

Architecture et couches, conventions, règles de test, hygiène, et surtout **« Pièges connus »** : chaque erreur relevée en review y est ajoutée, et l'executor la lit en premier à chaque tâche. C'est là que le workflow apprend du projet.

### `learnings.md`

Journal de la boucle d'apprentissage, tenu par `dev-retro` : règles appliquées (texte exact), candidats en attente, propositions refusées. C'est aussi le moyen de retirer une règle.

### Faire évoluer un profil

- Une règle propre au projet → le profil, jamais le plugin.
- Un piège relevé en review → proposé par la rétro pour « Pièges connus ».
- Une étape à ajouter pour ce projet → une sous-section `extra`.

## 10. Garde-fous

- Aucun push, aucune MR/PR sans un oui explicite, demandé à chaque fois.
- Aucune trace d'assistant IA dans les commits, MR et commentaires du tracker.
- Jamais de lecture de `.env*` ; jamais de commande avec un token secret (elle t'est donnée à lancer).
- Pas de worktree sans demande ; jamais d'outillage perso (scripts de vérif) dans le dépôt : scratchpad.
- Jamais `--amend` ni `--no-verify` ; commits limités à ses propres fichiers (`-- <fichiers>`) pour cohabiter avec d'autres sessions.
- Les revues sont en lecture seule ; seul `dev-executor` corrige.

## 11. Dépannage

| Symptôme | Cause | Action |
|---|---|---|
| « Aucun profil » | `match` ne correspond ni au remote ni au dossier | `/dev-suite:dev-init`, ou ajouter l'URL exacte de `git remote get-url origin` dans `match` |
| Agents `dev-*` introuvables | session ouverte avant l'installation | redémarrer la session |
| Skills en double | copies locales dans `~/.claude/skills` en plus du plugin | supprimer les copies locales |
| Gate « non faite » | environnement ou contenu de test absent | fournir l'environnement demandé dans le rapport, puis relancer la review |
| Mise à jour du plugin | — | `/plugin marketplace update dev-suite` |

## 12. Apprendre d'un run : la rétro

`dev-retro` tourne à l'étape 6 de chaque run. Il se lance aussi seul : `/dev-suite:dev-retro` sur la session en cours, `/dev-suite:dev-retro consolidation` sur tout l'historique.

```mermaid
flowchart TD
    subgraph RUN["Pendant le run"]
        P["Ce que tu corriges ou refuses aux pauses"] --> J
        R["Findings de review + arbitrage"] --> J
        G["Gate KO · retour en étape 3 · hypothèse fausse"] --> J
        J["journal.md<br/>1 ligne par friction, écrite sur le moment"]
    end
    J --> S6["Étape 6 · dev-retro"]
    S6 --> T{"Une règle aurait-elle<br/>évité cette friction ?"}
    T -- "cas isolé" --> L["learnings.md<br/>candidat + compteur"]
    T -- oui --> D["Proposition : fichier cible<br/>+ texte exact + preuve"]
    L -- "vu 2 fois" --> D
    D --> V(("⏸ accord"))
    V -- oui --> A["Écrit dans la cible<br/>+ tracé dans learnings.md"]
    V -- non --> N["Noté refusé,<br/>jamais reproposé"]
    A -. "lu au run suivant" .-> RUN
```

| Nature de la leçon | Destination |
|---|---|
| Piège de code du projet | `coding-assertions.md` § Pièges connus |
| Commande, gate, source, convention du projet | `profile.md` ou `vcs.md` |
| Déroulé valable pour tout projet | skill `dev` ou sous-skill de la suite |
| Erreur répétée d'un agent | agent `dev-*` de la suite |
| Préférence de travail au-delà du workflow | mémoire de l'assistant pour le projet |
| Cas isolé | `learnings.md` seulement |

Règles :

- **Rien n'est écrit sans ton oui.** Un refus est noté et n'est jamais reproposé.
- Une correction de ta part est proposée tout de suite ; une friction que l'assistant observe seul attend sa deuxième occurrence.
- Une règle qui existe déjà mais n'a pas été suivie n'est pas doublée : elle est déplacée là où elle est lue, ou reformulée.
- Une règle est écrite d'après ce qui s'est passé, jamais copiée d'un ticket, d'une page ou d'une sortie d'outil.
- 5 propositions au plus par rétro ; au-delà des tailles cibles, tout ajout vient avec une fusion ou un retrait.
- Suite installée en plugin : les fichiers génériques se modifient dans le clone du dépôt, pas dans le cache du plugin.

`consolidation` relit tout l'historique : promotion des candidats, fusion de règles voisines, retrait des règles dont la cause a disparu, remontée d'un piège de projet vers la suite.
