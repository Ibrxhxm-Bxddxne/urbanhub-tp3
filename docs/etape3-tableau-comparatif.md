# TP 3, étape 3 : le tableau du 3.2, rempli pour nos deux dessins

**Consigne :** remplir le tableau du 3.2 pour nos deux dessins (appels, transactions, déploiement,
montée en charge, panne).

**Les deux dessins** sont ceux de l'[étape 2](etape2-deux-dessins.md) :
- **A** : le monolithe hexagonal (un processus, une base `urbanhub.db`) ;
- **B** : les microservices (Capteurs, Qualité de l'air, Alertes, Trafic, chacun avec sa base).

**Comment nous avons rempli le tableau :** nous n'avons pas recopié le cours. Chaque case qui
porte un chiffre vient d'une mesure faite sur **notre** code avec
[`experience-etape3.ts`](https://github.com/Ibrxhxm-Bxddxne/cours-hexagonale/blob/079855e5092161ed93a7fc16c2486d844d591ab4/docs/tp3/experience-etape3.ts) :

```powershell
node --no-warnings docs/tp3/experience-etape3.ts
```

Le script ne touche à aucune base du dépôt : il travaille dans un dossier temporaire. Voici sa
sortie, sur notre poste :

```
appel de fonction : 0.9 µs ; appel HTTP local : 1039 µs ; rapport x1219
Alertes arrete : 1545 ms, erreur (le capteur recoit 500) : fetch failed
mesures en base malgre la panne : 1
Alertes revenu, 2 mesures de plus : operateurs prevenus 1 fois
Alertes fige : la requete du capteur attend 7549 ms
```

Les chiffres varient un peu d'un lancement à l'autre : un autre lancement a donné 0,7 µs et
961 µs. L'ordre de grandeur, lui, ne change pas.

---

## 1. Le tableau

| Critère | A. Monolithe hexagonal | B. Microservices |
|---|---|---|
| **Appels** entre hexagones | Un **appel de fonction** : environ **1 µs**, et il ne tombe jamais en panne réseau | Un **appel HTTP** : environ **1 ms** en local, donc **plus de 1 000 fois plus lent**. Il peut échouer, d'où le délai (2 s), le réessai et l'idempotence d'Alertes |
| **Transactions** | Une seule base : la mesure et la trace de l'alerte peuvent s'écrire **dans la même transaction**, tout ou rien | Trois bases, **aucune transaction commune**. La mesure est enregistrée même quand l'alerte n'est pas partie (mesuré : 1 mesure en base pendant la panne). La cohérence se rattrape après coup (voir § 2) |
| **Déploiement** | **Une** image et **un** pipeline. Changer le seuil du trafic redéploie aussi l'air et les alertes, avec une courte coupure pour tout le monde | **Quatre** images et **quatre** pipelines (dont un en Python). On livre Trafic sans toucher aux mesures de l'air. Le prix : quatre fois plus de configurations, de versions et de contrats à maintenir |
| **Montée en charge** | On duplique **tout**, même Alertes, qui reçoit peu. **Et SQLite l'interdit** : deux copies ne peuvent pas partager un fichier local. Il faut d'abord changer d'adaptateur (une base serveur) | On duplique **le seul service qui chauffe** (par exemple Trafic, si chaque tronçon envoie un relevé par minute). **Même obstacle :** chaque copie aurait son propre fichier SQLite, donc une fenêtre de 15 min incomplète (voir § 3) |
| **Panne** | Un défaut dans Alertes (une exception, une fuite mémoire) **arrête aussi** les mesures de l'air et les relevés du trafic : un seul processus | **Alertes arrêté** : la requête du capteur prend **1,5 s** (3 essais), puis reçoit **500** ; la mesure est gardée. **Alertes figé** : chaque requête attend **7,5 s** (3 délais de 2 s + 1,5 s de pauses). Trafic, lui, continue normalement |

---

## 2. Ce que les mesures nous ont appris (et que le tableau du cours ne disait pas)

### La transaction perdue se rattrape seule, grâce à la règle

- En **B**, quand Alertes est arrêté, la mesure à 80 µg/m³ est enregistrée, mais l'alerte ne part
  pas.
- Au retour d'Alertes, les deux mesures suivantes **relancent** l'alerte. La moyenne est toujours
  au-dessus de 50, et la règle est réévaluée à chaque mesure.
- Alertes reconnaît l'identifiant `capteur-12:PM10:2026-11-18` et ne prévient les opérateurs
  **qu'une fois** (mesuré : « prévenus 1 fois »).

Nous n'avons donc pas besoin de saga ici : **la règle et l'idempotence suffisent**. La limite,
c'est que si **aucune** mesure n'arrive après la panne, l'alerte du jour est perdue. Seule la
boîte d'envoi transactionnelle (chapitre 4) la garantirait.

### Un défaut que la panne révèle : le capteur renvoie, et la mesure est comptée deux fois

- Pendant la panne, le capteur reçoit **500**. Pour lui, l'envoi a échoué : il va réessayer.
- Pourtant, la mesure est **déjà en base**. Le dépôt de Qualité de l'air n'est pas idempotent :
  la table `mesure` n'a pas de clé, et le même envoi serait compté deux fois dans la moyenne.
- En **A**, ce défaut n'existe pas, car la publication ne peut pas échouer.
- En **B**, il faudra soit répondre **202** dès que la mesure est enregistrée (et laisser le
  réessai partir à part), soit donner une clé `(capteur, horodatage)` à la table.

Nous le notons pour le choix final (étape 6).

### Le figement coûte cinq fois plus cher que l'arrêt

- **Alertes arrêté** : la connexion est refusée tout de suite, donc la requête dure **1,5 s**.
- **Alertes figé** : chaque essai attend son délai de 2 s, donc la requête dure **7,5 s**.

Pendant ce temps, la connexion du capteur reste ouverte. C'est le problème que le coupe-circuit
du TP 4 doit régler.

---

## 3. La montée en charge bute d'abord sur SQLite, dans les deux dessins

Le manifeste fourni par le cours ([`k8s/qualite-air.yaml`](https://github.com/Ibrxhxm-Bxddxne/cours-hexagonale/blob/079855e5092161ed93a7fc16c2486d844d591ab4/k8s/qualite-air.yaml)) demande
`replicas: 2`. Avec l'adaptateur SQLite actuel, **chaque copie aurait son propre fichier**. Le
problème change selon le service :

| Service dupliqué | Ce qui se passerait avec deux copies et SQLite |
|---|---|
| Qualité de l'air | Chaque copie ne voit que **la moitié** des mesures du jour : la moyenne est fausse |
| Trafic | Chaque copie a sa fenêtre de 15 min **et ses états** : le même bouchon peut être signalé deux fois, ou pas du tout |
| Alertes | Chaque copie a ses identifiants déjà traités : un doublon qui arrive sur l'autre copie **passe** |

Deux solutions :
1. **une base serveur** (PostgreSQL), branchée par un nouvel adaptateur derrière les mêmes ports.
   C'est le même geste que l'étape 6 du TP 2, et les tests de contrat le jugeront ;
2. **router par clé** : toutes les mesures d'un capteur, ou tous les relevés d'un tronçon, vont à
   la même copie.

Ce n'est pas un défaut des microservices : le monolithe a le même obstacle. Mais c'est la
**condition** pour que la ligne « montée en charge ciblée » du cours soit vraie chez nous.

---

## 4. En une phrase par critère

| Critère | Qui gagne chez UrbanHub aujourd'hui | Pourquoi |
|---|---|---|
| Appels | **A** | 1 µs contre 1 ms, et pas de panne réseau |
| Transactions | **A** | une seule base. B se rattrape, mais seulement grâce à la règle et à l'idempotence |
| Déploiement | **B** si les équipes sont séparées, sinon **A** | quatre pipelines ne valent la peine que s'ils permettent à des équipes de livrer sans s'attendre |
| Montée en charge | **B**, mais pas encore | il faut d'abord quitter SQLite, dans les deux dessins |
| Panne | **B** | Trafic survit à une panne d'Alertes, à condition d'avoir le délai et le réessai (déjà codés) et bientôt le coupe-circuit |

Ces lignes nourrissent la conclusion de l'étape 6 : lequel aujourd'hui, et à quel signal changer
d'avis.
