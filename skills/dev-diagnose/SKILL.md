---
name: dev-diagnose
description: Boucle de diagnostic pour un bug ou une régression de performance — reproduction par commande, hypothèses falsifiables, instrumentation ciblée, correctif à la cause racine avec test de non-régression. Utilisé par /dev en mode bug ; aussi quand l'utilisateur demande de diagnostiquer ou débugger.
---

# dev-diagnose

Masquer tout secret dans ce qui est affiché (`<REDACTED>`). Jamais lire un `.env*`.

## 1. Boucle de feedback (avant toute hypothèse)

Construire **une commande unique** qui reproduit le symptôme exact du rapport, dans cet ordre de préférence : test qui échoue au bon seam ; curl contre le dev server ; script navigateur headless (scratchpad) ; replay d'une requête capturée ; harnais jetable ; bisection automatisée (`git bisect run`) ; comparaison ancienne/nouvelle version.

Critère de sortie, la commande a été **exécutée au moins une fois** (montrer invocation + sortie) et elle est :
- rouge sur **ce** bug (symptôme exact, pas « ne plante pas ») ;
- déterministe (bug intermittent : taux de reproduction élevé et stable) ;
- rapide (secondes) et lançable sans humain.

Impossible → le dire, lister ce qui a été tenté, demander un accès, un artefact (HAR, logs) ou l'autorisation d'instrumenter. Pas d'hypothèse sans boucle.

## 2. Reproduire et réduire

Vérifier que c'est le symptôme décrit, pas un voisin. Réduire le scénario un élément à la fois jusqu'à ce que chaque élément restant soit nécessaire.

## 3. Hypothèses

3 à 5 hypothèses classées, chacune falsifiable : « si X est la cause, changer Y fait disparaître le bug ». **Les montrer à l'utilisateur** avant de tester (il peut re-classer) ; sans réponse, continuer avec son propre classement.

## 4. Instrumenter

Une sonde par prédiction, une variable à la fois. Debugger > logs ciblés aux frontières. Chaque log préfixé `[DEBUG-xxxx]` (même préfixe pour toute la session). Perf : mesurer une baseline puis bisecter, pas de logs.

## 5. Cause racine et périmètre

- Cause racine **vérifiée dans le fichier réel** (file:line), jamais seulement sur le rapport d'un agent.
- Lister tous les appelants du code fautif : le correctif va dans le code partagé, pas sur le seul chemin du ticket.

## 6. Correctif + non-régression

Test de non-régression **avant** le fix, au seam qui exerce le vrai schéma du bug. Le voir échouer, corriger, le voir passer, puis relancer la boucle du point 1 sur le scénario d'origine. Pas de seam correct → le signaler comme finding (l'architecture empêche de verrouiller le bug), pas de test factice.

## 7. Nettoyage

- [ ] boucle d'origine verte ; test de non-régression vert (ou absence de seam documentée) ;
- [ ] `grep -rn "DEBUG-xxxx"` vide ; prototypes jetables supprimés ;
- [ ] hypothèse confirmée écrite dans le corps du commit ;
- [ ] « qu'est-ce qui aurait évité ce bug ? » → une ligne dans « Points remontés » si la réponse est architecturale.
