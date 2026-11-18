# TP 3, étape 6 : la conclusion. Lequel aujourd'hui, et à quel signal changer d'avis

**Consigne :** conclure. Lequel aujourd'hui, et à quel signal changeriez-vous d'avis ?

---

## 1. Notre choix aujourd'hui : le monolithe hexagonal

**UrbanHub tourne aujourd'hui en un seul processus Node.js qui contient trois hexagones** :
Qualité de l'air, Trafic et Alertes. Il a une seule base, avec une table (ou un groupe de tables)
par hexagone. C'est le dessin A de l'[étape 2](etape2-deux-dessins.md).

**Capteurs reste un service à part**, parce qu'il est écrit en Python. On ne peut pas le fusionner
sans le réécrire, et réécrire un service qui marche n'apporte rien.

### Pourquoi : ce que les étapes précédentes ont montré

| Critère | Ce que nous avons mesuré ou constaté | Ce que cela dit |
|---|---|---|
| **Appels** | environ 1 µs en fonction, environ 1 ms en HTTP local, soit plus de 1 000 fois plus lent (étape 3) | Les microservices ajoutent un coût **et** des pannes possibles, sans rien rapporter tant que la charge est faible |
| **Transactions** | En microservices, une mesure est gardée alors que l'alerte n'est pas partie. Elle ne se rattrape que si une autre mesure arrive (étape 3) | Le monolithe écrit la mesure et la trace de l'alerte dans la même transaction |
| **Un défaut nouveau** | En microservices, le capteur reçoit 500 alors que sa mesure est en base. S'il la renvoie, elle est comptée deux fois (étape 3) | Le monolithe n'a pas ce défaut : la publication ne peut pas échouer |
| **Montée en charge** | Bloquée par SQLite **dans les deux dessins** (étape 3, § 3) | Les microservices ne nous font gagner aucune copie tant que nous restons sur SQLite |
| **Déploiement** | En microservices, Trafic se livre en recréation, avec environ 10 s de coupure (étape 5). Il faudrait 4 images, 4 chaînes de livraison, une passerelle, un backend principal et un cluster (étape 4) | Nous sommes **un seul groupe** de 3 ou 4 personnes : nous livrons ensemble, quatre chaînes ne nous font gagner aucune autonomie |
| **Panne** | En microservices, Trafic survit à une panne d'Alertes. C'est le **seul vrai gain** mesuré (étape 3) | C'est un gain réel, mais le cas d'utilisation d'Alertes fait aujourd'hui 23 lignes (47 pour tout le service) : un `Set` et un journal. Le risque qu'il emporte tout le processus est faible |

Le volume est petit : `troncons.json` compte **3 tronçons**, et le cours suit une poignée de
capteurs. Rien ne justifie le coût d'exploitation que Segment a décrit (des ingénieurs absorbés
par l'exploitation) ni le coût d'orchestration de Prime Video (5.1).

### Ce que nous gardons pour ne pas nous enfermer

Le cours le dit (slide 59) : un monolithe **bien découpé en hexagones** se découpe ensuite sans
réécrire la logique métier. Nous gardons donc cinq garde-fous :

1. **Les ports entre hexagones restent des ports.** Qualité de l'air appelle `PublicateurAlertes`,
   jamais `ReceptionAlertes` directement. L'adaptateur « en processus » se remplacera par
   l'adaptateur HTTP, qui existe déjà.
2. **Aucune jointure SQL entre les tables de deux hexagones** (étape 2, § 1).
3. **Le test de la règle de dépendance** s'étend à une deuxième règle : un hexagone n'importe pas
   les fichiers d'un autre, seulement ses ports.
4. **Une racine de composition par hexagone**, appelée par une racine commune. Les trois
   `principal.ts` actuels restent utilisables seuls.
5. **Les tests de contrat** (dépôts, états) restent dans `npm test`. Ce sont eux qui jugeront
   l'adaptateur PostgreSQL le jour où il faudra plusieurs copies.

Le monolithe ne demande donc **aucune ligne de plus dans les centres**. Seuls changent les
adaptateurs entre hexagones et la racine de composition.

---

## 2. Les signaux qui nous feraient changer d'avis

Un signal n'est utile que s'il est **observable** : chacun s'appuie sur une métrique ou un fait
que la supervision de l'[étape 4](etape4-vue-ensemble.md) fournit.

| # | Signal (observable) | Seuil | Ce que nous ferions | Premier service concerné |
|---|---|---|---|---|
| 1 | **Une panne d'un hexagone en arrête d'autres** : incident où une erreur d'Alertes a coupé la réception des mesures ou des relevés | **1 incident** de ce type | Sortir **Alertes** en service à part, avec sa base `alertes.db`. Le délai, le réessai et l'idempotence sont **déjà écrits** | Alertes |
| 2 | **Une autre équipe** prend un contexte et ne peut plus livrer à son rythme : des livraisons de Trafic attendent celles de l'air | une équipe distincte, ou plus d'**une livraison par semaine bloquée** | Sortir **Trafic**. La stratégie de recréation de l'étape 5 suffit au départ | Trafic |
| 3 | **Un hexagone demande plus de ressources que les autres** : le temps de réponse de `POST /releves` monte (95ᵉ centile au-dessus de 200 ms), alors que `POST /mesures` reste rapide | 2 semaines au-dessus du seuil | D'abord **PostgreSQL** derrière les mêmes ports (tests de contrat). **Ensuite** sortir le service chargé et le passer à 2 copies en mise à jour progressive (étape 5) | probablement Trafic (un relevé par tronçon et par minute) |
| 4 | **Des exigences de disponibilité différentes** : les opérateurs exigent que les alertes passent même pendant la livraison du trafic | une exigence écrite (par exemple 99,9 % pour Alertes) | Sortir Alertes, avec 2 copies | Alertes |
| 5 | **Une autre technologie** pour un contexte : Alertes réécrit en Java/Spring, comme dans le cours | une décision technique écrite | Service à part : deux langages ne partagent pas un processus | Alertes |

### Le signal inverse : quand regrouper

Le cours cite ce défi (3.3, défi 6) : **deux services qui se parlent à chaque requête n'en font
qu'un.**

Si un jour le backend principal appelle Qualité de l'air et Trafic pour **chaque** requête de
l'écran, et que les deux sont toujours livrés ensemble, alors ce découpage nous coûte sans
rapporter. Nous les regrouperions.

### Dans quel ordre découper

Si les signaux arrivent, nous découperons dans cet ordre :
1. **Alertes en premier.** Il sert plusieurs contextes, il est le plus petit, et tout ce qu'il
   faut pour le sortir est déjà écrit : le publicateur HTTP, le réessai, l'idempotence.
2. **Trafic ensuite.** C'est celui dont la charge grandit avec la ville.
3. **Qualité de l'air en dernier.** Il restera le cœur du monolithe tant qu'il n'a pas de signal
   à lui.

---

## 3. En une phrase

**Aujourd'hui : un monolithe hexagonal, plus le service Capteurs en Python, parce que nous sommes
une seule équipe avec trois petits contextes, et que le seul gain mesuré des microservices (Trafic
qui survit à une panne d'Alertes) ne paie pas quatre chaînes de livraison et un cluster. Nous
sortirons Alertes au premier incident où une panne d'un hexagone en arrête un autre, ou dès qu'une
deuxième équipe ou une exigence de disponibilité l'impose ; et nous passerons à PostgreSQL avant
de dupliquer quoi que ce soit.**

### Un mot sur le TP 4

Le TP 4 nous demande de faire tourner deux microservices qui se parlent. Ce n'est pas une
contradiction : le TP 4 **exerce** le découpage que le signal n° 1 déclencherait, sur le service
qui sortirait en premier. Ce qu'il construit (le coupe-circuit, Kafka) est exactement ce dont
nous aurions besoin ce jour-là.
