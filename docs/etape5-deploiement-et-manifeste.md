# TP 3, étape 5 : la stratégie de déploiement et le manifeste Kubernetes

**Consigne :** proposer une stratégie de déploiement et écrire le manifeste Kubernetes d'un
service.

**Le service choisi : Trafic**, celui que nous avons écrit au TP 2. Son manifeste est
[`k8s/trafic.yaml`](../k8s/trafic.yaml), et son image se construit avec
[`Dockerfile.trafic`](../Dockerfile.trafic). Nous avons dû créer ce Dockerfile : l'étape 4
avait noté qu'il manquait.

---

## 1. La stratégie : pas la même pour tous, et pas la même aujourd'hui que demain

Le cours (3.4) propose quatre stratégies : recréation, mise à jour progressive, bleu-vert et
canari. Le cours le dit : **la stratégie se choisit service par service**. Chez nous, elle dépend
d'abord d'une question : **le service garde-t-il un état dans un fichier local ?**

| Service | État local aujourd'hui | Stratégie **aujourd'hui** | Stratégie **cible** (après le passage à PostgreSQL) |
|---|---|---|---|
| **Trafic** | `releves.db` (relevés et états des tronçons) | **Recréation**, 1 copie | **Progressive**, 2 copies ; **canari** quand on change de règle |
| Qualité de l'air | `mesures.db` | **Recréation**, 1 copie | Progressive, 2 copies |
| Alertes | en mémoire, puis `alertes.db` | **Recréation**, 1 copie | Progressive, 2 copies |
| Backend principal | **aucun** | **Progressive**, 2 copies, dès le départ | Progressive |
| Passerelle | aucun | Progressive | Progressive |

### Pourquoi la recréation pour Trafic aujourd'hui

C'est la stratégie la plus modeste du cours. Nous la choisissons parce que les autres
**casseraient la règle de gestion** tant que l'état est dans SQLite.

| Stratégie | Ce qui se passerait avec Trafic sur SQLite | Verdict |
|---|---|---|
| Progressive (`RollingUpdate`) | L'ancienne et la nouvelle copie tournent **en même temps**. Avec un seul volume, deux processus écrivent le même fichier. Le volume `ReadWriteOnce` peut même refuser de se monter si la nouvelle copie part sur un autre nœud. Avec deux volumes, chaque copie a **sa** fenêtre de 15 min et **ses** états : un bouchon peut être signalé deux fois, ou jamais (étape 3, § 3) | non |
| Bleu-vert | Deux environnements complets, donc deux bases. Au moment de la bascule, le vert n'a **ni** les relevés des 15 dernières minutes **ni** les états « déjà signalé ». Il resignale les bouchons en cours | non |
| Canari | Une petite part des relevés va à la nouvelle version, sur une autre base. **Aucune** des deux copies ne voit tous les relevés d'un tronçon : les moyennes sont fausses dans les deux | non |
| **Recréation** | L'ancienne copie s'arrête, **puis** la nouvelle ouvre le même fichier. Un seul écrivain à la fois ; la fenêtre et les états sont intacts | **oui** |

### Le prix de la recréation, chiffré

Une **coupure** de Trafic à chaque livraison. Nous l'estimons à **une dizaine de secondes** :
- jusqu'à 5 s pour arrêter l'ancienne copie (`terminationGracePeriodSeconds: 5`) ;
- le démarrage du nouveau processus ;
- jusqu'à 5 s avant que la sonde `readinessProbe` le déclare prêt.

Ce chiffre est une estimation à partir des réglages, pas une mesure : nous n'avions pas de cluster
pour mesurer (voir § 4).

Nous l'acceptons pour trois raisons :
1. **La règle porte sur 15 minutes.** Perdre 10 s de relevés change à peine une moyenne sur
   15 min. Et si la boucle de comptage renvoie ses relevés, rien n'est perdu.
2. **Aucun signalement n'est perdu ni doublé.** Les états sont dans le même fichier, retrouvé
   intact.
3. **Trafic se livre seul.** La coupure ne touche ni la qualité de l'air ni les alertes : c'est le
   bénéfice du découpage de l'étape 3.

### Le signal pour passer à la stratégie cible

Le jour où la coupure n'est plus acceptable (une livraison par jour, ou des relevés qu'on ne peut
pas renvoyer), on branche un adaptateur **PostgreSQL** derrière `DepotReleves` et `EtatsTroncons`.
Les tests de contrat du TP 2 (`contratDepotReleves`, `contratEtatsTroncons`) jugent ce nouvel
adaptateur. Ensuite, dans le manifeste :

```diff
-  replicas: 1
-  strategy: { type: Recreate }
+  replicas: 2
+  strategy:
+    type: RollingUpdate
+    rollingUpdate: { maxUnavailable: 0, maxSurge: 1 }   # jamais moins de 2 copies prêtes
```

Le volume `trafic-donnees` disparaît, et `BASE_RELEVES` vient d'un Secret. **Le centre ne change
pas.**

**Le canari pour un changement de règle.** Passer de `SeuilRelatif` à `SeuilRelatifAvecMinimum`
change le nombre de signalements. Avec une base partagée, on envoie d'abord 10 % du trafic à la
nouvelle version. On regarde ensuite les signalements par heure (la métrique de l'étape 4) avant
d'élargir. C'est le seul service où le canari a un vrai sens : une règle métier qui change, et un
chiffre pour en juger.

---

## 2. Le manifeste : [`k8s/trafic.yaml`](../k8s/trafic.yaml)

Il décrit quatre objets :

| Objet | Rôle | Ce qui le justifie |
|---|---|---|
| `ConfigMap trafic-config` | `REGLE`, `URL_SIGNALEMENTS` et le fichier `troncons.json` | La configuration hors du code (étape 4, § 3.1). On ajoute un tronçon sans reconstruire l'image |
| `PersistentVolumeClaim trafic-donnees` | 1 Gi, `ReadWriteOnce` | `releves.db` doit **survivre** au remplacement du Pod : c'était tout l'objet de l'étape 6 du TP 2 (les états « déjà signalé » survivent au redémarrage) |
| `Deployment trafic` | **1** copie, stratégie **`Recreate`** | § 1 |
| `Service trafic` | l'adresse stable `http://trafic:8003` | Le backend principal l'appelle par son nom (étape 4) |

### Les choix ligne par ligne

| Ligne | Pourquoi |
|---|---|
| `replicas: 1` | **Pas 2**, contrairement au manifeste du cours pour Qualité de l'air. Deux copies sur SQLite cassent la règle (étape 3, § 3) |
| `strategy: { type: Recreate }` | L'ancienne copie lâche le fichier **avant** que la nouvelle l'ouvre |
| `URL_SIGNALEMENTS: http://alertes:8002/alertes` | Corrige le défaut de l'étape 1 : le défaut du code vise le port 8004, où personne n'écoute. La **traduction** du signalement pour Alertes reste à écrire (étape 2) |
| `TRONCONS: /config/troncons.json` | Le référentiel vient du ConfigMap, monté en lecture seule |
| `REGLE: avec-minimum` | La Stratégie choisie à l'assemblage, sans faux bouchon la nuit (TP 2, étape 5) |
| `readinessProbe` et `livenessProbe` en `tcpSocket` | Le service n'a pas encore de `GET /sante` (étape 4) : on vérifie que le port écoute. À remplacer par une sonde HTTP quand la route existera |
| `terminationGracePeriodSeconds: 5` | `principal.ts` ne gère pas le signal d'arrêt (`SIGTERM`). Sans ce réglage, Kubernetes attendrait 30 s, et la coupure de la recréation durerait 30 s de plus. Couper n'abîme rien : chaque écriture SQLite est atomique |
| `USER node` (Dockerfile) et `fsGroup: 1000` | Le processus ne tourne pas en root, mais peut quand même écrire dans le volume |
| `resources` | Une réservation modeste, et une limite mémoire qui empêche une fuite d'emporter le nœud |
| `image: urbanhub/trafic:1.0.0` | En pratique, l'étiquette est l'empreinte du commit livré (étape 4, § 3.3) |

---

## 3. Le Dockerfile : [`Dockerfile.trafic`](../Dockerfile.trafic)

Il suit celui du cours (Qualité de l'air), avec deux différences :
- il ne copie que `urbanhub-trafic/` ;
- il ajoute `USER node`.

```powershell
docker build -f Dockerfile.trafic -t urbanhub/trafic:<empreinte> .
```

---

## 4. Ce que nous avons vérifié, et ce que nous n'avons pas pu vérifier

| Vérification | Résultat |
|---|---|
| Le YAML se lit, et contient bien les quatre objets | **oui** : `ConfigMap`, `PersistentVolumeClaim`, `Deployment`, `Service` (lu avec PyYAML) |
| Le service démarre avec **exactement** la configuration du Pod : `TRONCONS` extrait du ConfigMap, `BASE_RELEVES` dans un dossier `donnees`, `REGLE=avec-minimum` | **oui**. Un relevé valide reçoit **202** ; un tronçon inconnu, **404** ; un relevé invalide, **422** ; et `releves.db` est créé dans le dossier du volume |
| `npm test` | **67 / 67** : le manifeste et le Dockerfile ne touchent pas au code |
| `kubectl apply --dry-run` | **non vérifié** : `kubectl` est installé, mais aucun cluster ne répond sur ce poste |
| `docker build` | **non vérifié** : le moteur Docker n'était pas démarré |

À faire quand un cluster est disponible (par exemple Docker Desktop avec Kubernetes, ou kind) :

```powershell
docker build -f Dockerfile.trafic -t urbanhub/trafic:1.0.0 .
kubectl apply -f k8s/trafic.yaml
kubectl rollout status deployment/trafic
kubectl port-forward service/trafic 8003:8003
```

Puis, depuis un autre terminal, un relevé de test doit recevoir **202** :

```bash
curl -X POST localhost:8003/releves -H "Content-Type: application/json" \
  -d '{"troncon":"rue-de-rivoli-1","vitesse":25,"horodatage":"2026-11-18T08:00:00"}'
```
