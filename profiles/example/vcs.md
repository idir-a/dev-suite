# Convention VCS — example

## Message

- Conventional commits : `type(scope): sujet (SHOP-123)`, en anglais, minuscules, sans point final, ≤ 100 caractères par ligne.
- Corps : le pourquoi en bullet points ; omis si le sujet suffit.
- Aucune trace d'assistant IA (ni `Co-Authored-By`, ni lien) dans commits, PR, commentaires.

## Commande

1. Avant : `git log -1 --oneline` et `git status --short`.
2. `git add -- <mes fichiers>` puis `git commit -m "<en-tête>" -m "- point 1" -- <mes fichiers>`.
3. Après : `git show --stat HEAD`.

- Jamais `--amend`, jamais `--no-verify` ; hook rejeté → corriger la cause, nouveau commit.
- Correctif de review sur un commit non poussé : `git commit --fixup=<sha>` ; fondu avant le push par `git rebase -i --autosquash`, sur oui, branche jamais poussée.
- Jamais le `task_dir` dans un commit. Jamais de push ni de PR sans oui explicite.
