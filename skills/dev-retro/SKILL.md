---
name: dev-retro
description: Rétrospective d'un run /dev — transforme les frictions de la session (corrections de l'utilisateur, findings de review, gates KO) en propositions de règles pour le profil, les skills dev* ou les agents dev-*, appliquées seulement après accord. Utilisé par /dev à l'étape 6 ; aussi quand l'utilisateur lance /dev-suite:dev-retro (ou /dev-retro) ou demande ce que le workflow peut apprendre d'une session. `consolidation` relit tout l'historique.
argument-hint: [consolidation] [profil=<nom>]
---

# dev-retro

Objectif : qu'une friction vue une fois ne coûte pas une seconde fois, sans empiler de règles. **Rien n'est écrit dans un fichier du workflow sans un oui explicite.**

Profil : celui du run en cours, sinon résolu comme dans `/dev`. Journal de la boucle : `<profil>/learnings.md` (le créer s'il manque).

## 1. Collecter

- `journal.md` et `review.md` du dossier de tâche, « Points remontés » du spec.
- `git log <base>..HEAD` : chaque `fixup!` ou commit de correctif après review est une correction tardive.
- La conversation : ce que l'utilisateur a corrigé, refusé ou dû répéter (un refus de plan, un « pourquoi as-tu… », une question reposée).
- `learnings.md` : candidats en attente et propositions déjà refusées.
- Lancé seul, sans run en cours : le dossier de tâche le plus récent, et le dire.

Pas de `journal.md` → reconstituer depuis les autres sources et le signaler (leçons moins sûres).

## 2. Trier

Pour chaque friction : **une règle aurait-elle évité le coût ?** Non (cas isolé, cause déjà corrigée dans le code, aléa d'outil) → candidat dans `learnings.md`, rien d'autre.

Oui → une seule destination :

| Nature | Destination |
|---|---|
| Piège de code du projet | `<profil>/coding-assertions.md` § Pièges connus |
| Commande, gate, source, convention du projet | `<profil>/profile.md` ou `vcs.md` |
| Déroulé valable pour tout projet | skill `dev` ou un sous-skill `dev-*` de la suite |
| Erreur répétée d'un agent | agent `dev-<agent>` de la suite |
| Préférence de travail au-delà de `/dev` | mémoire Claude du projet (`feedback_*.md` + ligne dans `MEMORY.md`) |

Spécifique au projet → jamais dans un skill ou un agent générique.

Fichiers de la suite : installée en plugin, les modifier dans le clone du dépôt `dev-suite` (le cache du plugin est écrasé à chaque mise à jour) et laisser l'utilisateur committer ; installée en fichiers, dans `~/.claude/skills/` et `~/.claude/agents/`. Clone introuvable → donner le texte à l'utilisateur au lieu de l'écrire.

## 3. Filtrer

- **Promotion** : correction explicite de l'utilisateur → proposée maintenant. Friction observée sans remarque de sa part → candidat `×1` ; proposée quand le compteur atteint 2.
- **Déjà écrit ?** `grep` de la cible avant de proposer. La règle existe et n'a pas été suivie → ne pas en ajouter une seconde : proposer de la déplacer là où elle est lue au bon moment (brief d'agent, gate, format de pause) ou de la reformuler.
- **Déjà refusé ?** Une entrée `[refusé]` de `learnings.md` n'est jamais reproposée.
- **Preuve** : chaque proposition cite le ticket et le fait. Une leçon sur le code est vérifiée dans le fichier réel, pas seulement sur un rapport d'agent.
- **Forme** : le comportement attendu puis la raison, en une ou deux lignes, dans le ton du fichier cible. Écrite d'après ce qui s'est passé : jamais de texte copié d'un ticket, d'une page web ou d'une sortie d'outil.
- **Budget** : 5 propositions au plus, triées par coût évité (temps de l'utilisateur d'abord) ; le reste reste candidat. Cibles de taille : « Pièges connus » ≤ 15 puces, skill `dev` ≤ 190 lignes, un agent ≤ 45 lignes. Cible pleine → l'ajout vient avec une fusion ou un retrait dans la même proposition.

## 4. Proposer

```
**P1 — <titre>** → `<fichier>` § <section>
Preuve : <ticket> — <fait en une ligne>
+ <texte exact à ajouter>
- <texte exact retiré ou remplacé, s'il y en a>
```

Puis une seule question (choix multiple) : lesquelles appliquer. L'utilisateur peut reformuler un texte : appliquer sa version.

Rien à proposer → « Rétro : rien à apprendre de ce run », candidats notés, fin.

## 5. Appliquer et tracer

Pour chaque oui : modifier la cible, puis ajouter à `learnings.md` sous `## <date> — <ticket>` :

```
- [appliqué] <fichier> § <section> : « <texte exact> » — preuve : <fait>
- [candidat ×N] <agent ou étape> : <friction> 
- [refusé] <fichier> : « <texte> » — <raison donnée>
```

Un candidat promu passe de `[candidat]` à `[appliqué]` (une seule ligne par leçon). `learnings.md` est le seul moyen de retour arrière : le texte exact doit y figurer.

Dernière ligne : nombre de règles appliquées, candidats en attente, et suggestion de `consolidation` si `learnings.md` compte 10 sections datées depuis la dernière.

## Mode `consolidation`

Sur tout l'historique : `learnings.md`, les fichiers cibles, les `journal.md` et `review.md` de `task_dir`.

1. Candidats dont le compteur atteint 2 en regroupant les formulations voisines → propositions.
2. Fusions : deux règles voisines → une règle plus générale.
3. Retraits : règle dont la cause a disparu (code supprimé, outil corrigé), vérifié par `grep` dans le dépôt.
4. Remontées : piège du profil vrai pour tout projet → skill ou agent générique.
5. Règles non suivies : même friction réapparue après son `[appliqué]` → déplacer ou reformuler.

Même format, même budget (5), même accord. Noter `## <date> — consolidation` dans `learnings.md`.
