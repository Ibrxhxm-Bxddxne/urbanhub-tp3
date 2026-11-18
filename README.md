# UrbanHub : monolithe ou microservices (TP 3)

Dépôt du groupe pour le TP 3 du module Architecture hexagonale (EADL BC01-FM03, A. Mekouar).

**Groupe :** `<noms des membres du groupe>`

**La question du TP :** la ville ajoute des capteurs et des services. UrbanHub reste-t-il un seul
bloc (un monolithe), ou se découpe-t-il en services ? Nous comparons les deux formes sur notre
backend, puis nous proposons, pour chacune, son déploiement et sa gestion.

**Notre réponse :** aujourd'hui, **un monolithe hexagonal**, plus le service Capteurs en Python. Nous
sortirons Alertes au premier signal, puis Trafic, et nous passerons à PostgreSQL avant de dupliquer
quoi que ce soit. Le raisonnement est dans l'[étape 6](docs/etape6-conclusion.md).

**Où est le code étudié :** dans le dépôt UrbanHub du binôme,
https://github.com/Ibrxhxm-Bxddxne/cours-hexagonale. On y trouve trois hexagones (Qualité de
l'air, Alertes, Trafic) et leurs 67 tests.
- Les liens vers le code pointent sur son commit `079855e`, pour que ce qu'on lit corresponde à ce
  qui a été étudié.
- Les commandes citées dans les documents (`npm test`, `node docs/tp3/experience-etape3.ts`,
  `docker build -f Dockerfile.trafic .`) se lancent **dans ce dépôt-là**.

---

## Les livrables

| Livrable demandé | Où le trouver |
|---|---|
| **Les deux schémas** (monolithe hexagonal, microservices) | [`docs/etape2-deux-dessins.md`](docs/etape2-deux-dessins.md) |
| **La vue d'ensemble** (passerelle, backend principal, services, bases ; qui appelle qui et pourquoi) | [`docs/etape4-vue-ensemble.md`](docs/etape4-vue-ensemble.md), § 1 et 2 |
| **La gestion de chaque service** (configuration par environnement, supervision, livraison) | [`docs/etape4-vue-ensemble.md`](docs/etape4-vue-ensemble.md), § 3 |
| **Le comparatif** (tableau du 3.2, avec des mesures sur notre code) | [`docs/etape3-tableau-comparatif.md`](docs/etape3-tableau-comparatif.md) |
| **Le manifeste Kubernetes** (service Trafic) | [`k8s/trafic.yaml`](k8s/trafic.yaml), avec son image [`Dockerfile.trafic`](Dockerfile.trafic) ; la stratégie de déploiement est expliquée dans [`docs/etape5-deploiement-et-manifeste.md`](docs/etape5-deploiement-et-manifeste.md) |

### Les étapes du TP

| Étape | Document |
|---|---|
| 1. Les contextes délimités et leurs données | [`docs/etape1-contextes-delimites.md`](docs/etape1-contextes-delimites.md) |
| 2. UrbanHub en monolithe hexagonal, puis en microservices | [`docs/etape2-deux-dessins.md`](docs/etape2-deux-dessins.md) |
| 3. Le tableau du 3.2, rempli avec des mesures | [`docs/etape3-tableau-comparatif.md`](docs/etape3-tableau-comparatif.md) |
| 4. La vue d'ensemble et la gestion de chaque service | [`docs/etape4-vue-ensemble.md`](docs/etape4-vue-ensemble.md) |
| 5. La stratégie de déploiement et le manifeste | [`docs/etape5-deploiement-et-manifeste.md`](docs/etape5-deploiement-et-manifeste.md) |
| 6. La conclusion : lequel aujourd'hui, et à quel signal changer d'avis | [`docs/etape6-conclusion.md`](docs/etape6-conclusion.md) |

### Les chiffres qui portent la conclusion (étape 3)

| Mesure sur notre code | Résultat |
|---|---|
| Appel de fonction, puis appel HTTP local | environ 1 µs, puis environ 1 ms (plus de 1 000 fois plus lent) |
| Alertes arrêté | la requête du capteur dure 1,5 s, puis reçoit 500 ; la mesure est quand même gardée |
| Alertes figé | la requête du capteur attend 7,5 s |
| Alertes revenu | l'alerte repart, et les opérateurs ne sont prévenus qu'une fois |

---

## Rendu

| | |
|---|---|
| **Lien du dépôt** | https://github.com/Ibrxhxm-Bxddxne/urbanhub-tp3 |
| **Empreinte du commit** | `7bf1980` (TP3 : livrables, schémas, comparatif, vue d'ensemble, manifeste, conclusion) |
| **Chemin du livrable** | ce README ; les documents dans [`docs/`](docs/) ; le manifeste dans [`k8s/trafic.yaml`](k8s/trafic.yaml) |

Le dépôt est ouvert à **Mekouar-Ayoub** en collaborateur.
