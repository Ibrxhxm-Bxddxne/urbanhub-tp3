# TP 3, étape 1 : les contextes délimités du backend et leurs données

**Consigne :** lister les contextes délimités de notre backend (une partie du métier où chaque mot
a un seul sens) et les données de chacun.

**Méthode :** nous avons relu le code du dépôt (`urbanhub-air/`, `urbanhub-alertes/`,
`urbanhub-trafic/`) et cherché trois choses :
- les mots du métier ;
- les données que chaque partie garde ;
- les endroits où un même mot change de sens.

Un contexte se termine là où un mot change de sens.

---

## 1. Les contextes trouvés

| # | Contexte | Ce qu'il fait (en une phrase) | Code | Langage |
|---|---|---|---|---|
| 1 | **Qualité de l'air** | Il range les mesures de pollution, puis lève une alerte quand la moyenne PM10 du jour dépasse 50 µg/m³ | `urbanhub-air/` | TypeScript |
| 2 | **Alertes** | Il reçoit une alerte et prévient les opérateurs **une seule fois** par alerte | `urbanhub-alertes/` | TypeScript (Java/Spring dans le cours) |
| 3 | **Trafic** | Il range les relevés de vitesse, puis signale un tronçon **une fois** quand sa vitesse moyenne sur 15 min tombe sous 40 % de la limite | `urbanhub-trafic/` | TypeScript |
| 4 | **Capteurs** | Il reçoit la mesure d'un appareil et la transmet à Qualité de l'air | pas dans ce dépôt : code du cours, en Python | Python |

Les contextes 1 à 3 sont codés dans ce dépôt et testés (`npm test` : 67 / 67).

Le contexte 4 existe dans UrbanHub (diagramme des cas d'utilisation, slide 8 du cours). Nous
l'écrivons dans la liste parce que Qualité de l'air dépend de lui pour recevoir ses mesures.

### Ce que nous n'avons pas compté comme contexte

- **L'annuaire des capteurs** (ajouter, lire, modifier et supprimer un capteur). Il n'a aucune
  règle de gestion : c'est un simple CRUD. Le cours (cas 5, slide 28) conclut qu'il ne mérite pas
  d'hexagone. Il reste un **candidat** de contexte, mais il n'est pas codé.
- **Le référentiel des tronçons** (`troncons.json`). Il appartient au contexte Trafic : seule la
  règle de congestion se sert de la vitesse limite. Ce n'est pas un contexte à lui seul.

---

## 2. Les données de chaque contexte

### Qualité de l'air

| Donnée | Champs | Où elle vit |
|---|---|---|
| `Mesure` | `capteur`, `polluant`, `valeur` (µg/m³), `horodatage` (ISO) | base SQLite `mesures.db`, table `mesure` ([`depotSqlite.ts`](https://github.com/Ibrxhxm-Bxddxne/cours-hexagonale/blob/079855e5092161ed93a7fc16c2486d844d591ab4/urbanhub-air/src/adaptateurs/depotSqlite.ts)) |
| `Alerte` | `capteur`, `polluant`, `jour`, `moyenne`, `seuil`, `identifiant` = `capteur:polluant:jour` | **pas stockée** : elle est calculée, puis publiée vers Alertes |
| La règle | `SeuilMoyenneJournaliere("PM10", 50)` | dans le code ([`regles.ts`](https://github.com/Ibrxhxm-Bxddxne/cours-hexagonale/blob/079855e5092161ed93a7fc16c2486d844d591ab4/urbanhub-air/src/regles.ts)) |

### Alertes

| Donnée | Champs | Où elle vit |
|---|---|---|
| `AlerteRecue` | `identifiant`, `moyenne` : il ne lit que ces deux champs | message reçu sur `POST /alertes` ([`service.ts`](https://github.com/Ibrxhxm-Bxddxne/cours-hexagonale/blob/079855e5092161ed93a7fc16c2486d844d591ab4/urbanhub-alertes/src/service.ts)) |
| Les alertes déjà traitées | l'ensemble des `identifiant` déjà vus | **en mémoire** (`Set`) ; le code lui-même note « en production : une table » |
| Les opérateurs à prévenir | port `Operateurs` | aujourd'hui, une ligne dans le journal |

### Trafic

| Donnée | Champs | Où elle vit |
|---|---|---|
| `Releve` | `troncon`, `vitesse` (km/h), `horodatage` (à la seconde, sans fuseau) | base SQLite `releves.db`, table `releve`, indexée par `(troncon, horodatage)` |
| `Troncon` | `identifiant`, `vitesseLimite` | fichier [`troncons.json`](https://github.com/Ibrxhxm-Bxddxne/cours-hexagonale/blob/079855e5092161ed93a7fc16c2486d844d591ab4/urbanhub-trafic/troncons.json) (3 tronçons) |
| `EtatTroncon` | `fluide` ou `congestionne`, un par tronçon | même base, table `etat_troncon` (depuis l'étape 6 du TP 2) |
| `Signalement` | `troncon`, `vitesseMoyenne`, `vitesseLimite`, `debut`, `identifiant` = `troncon:debut` | **pas stocké** : il est publié sur `POST /signalements` |

### Capteurs (cours, hors de ce dépôt)

| Donnée | Champs | Où elle vit |
|---|---|---|
| La mesure brute d'un appareil | celle que Qualité de l'air recevra | en transit seulement : `TransmissionHttp` la transmet aussitôt (slide 44) |

### Ce qui n'est partagé par personne

**Aucune base n'est partagée.** Qualité de l'air ouvre `mesures.db` ; Trafic ouvre `releves.db`
(variable `BASE_RELEVES`). Alertes n'a pas encore de base. Les contextes ne se parlent que par des
messages HTTP. C'est déjà la règle du cours : « aucun service ne lit la base d'un autre ».

---

## 3. La preuve que ce sont des contextes différents : les mêmes mots, des sens différents

| Mot | Dans Qualité de l'air | Dans Trafic | Dans Alertes |
|---|---|---|---|
| **mesure** | un taux de polluant en µg/m³ (`Mesure`) | le mot n'existe pas : on dit **relevé**, une vitesse en km/h | — |
| **capteur** | l'identifiant de l'appareil de pollution, la clé de l'alerte | absent : la clé est le **tronçon** | — |
| **seuil** | une valeur **absolue** : 50 µg/m³ | une **part** de la limite : 40 % de la vitesse limite du tronçon | — |
| **moyenne** | sur le **jour** calendaire | sur une **fenêtre glissante** de 15 min | un nombre à afficher, sans calcul |
| **alerte / signalement** | le **fait** qu'un seuil est franchi (`Alerte`) | le même fait se dit **signalement** | un **message à traiter une seule fois** et à transmettre aux opérateurs |
| **identifiant** | `capteur:polluant:jour` | `troncon:debut` | une clé opaque, pour reconnaître un doublon |
| **horodatage** | ISO, au moins à la minute | à la seconde, sans fuseau | — |

Si on fusionnait ces modèles en un seul, il faudrait :
- un « seuil » qui soit à la fois absolu et relatif ;
- une « moyenne » à la fois journalière et glissante ;
- une « alerte » qui veuille dire à la fois « seuil franchi » et « message à dédoublonner ».

Chaque service aurait des champs qui n'ont pas de sens pour lui. Le découpage en contextes évite
ce mélange.

---

## 4. Les liens entre les contextes (la carte des contextes)

| De | Vers | Comment | Ce qui passe la frontière | État aujourd'hui |
|---|---|---|---|---|
| Capteurs | Qualité de l'air | `POST /mesures` (port 8001) | une mesure | code du cours |
| Qualité de l'air | Alertes | `POST /alertes` (port 8002), avec réessai | une `Alerte` | codé et testé (TP 1, cours) |
| Trafic | ? | `POST /signalements` (port 8004 par défaut) | un `Signalement` | **personne ne reçoit encore** |

### Ce que la carte révèle

1. **Trafic n'a pas de destinataire.**
   - Son publicateur vise `http://localhost:8004/signalements`, mais aucun service n'écoute ce
     port.
   - Alertes ne peut pas le recevoir tel quel : d'après son code, il lit `moyenne`, alors qu'un
     signalement porte `vitesseMoyenne`. Il afficherait « moyenne undefined ».
   - Pour que Trafic passe par Alertes, il faut un **contrat de message commun** : par exemple
     `identifiant`, `source`, `texte`. Chaque émetteur traduirait son modèle dans ce contrat, dans
     son propre adaptateur.
   - Cette traduction à la frontière est exactement ce que le cours attend d'un contexte délimité
     (slide 71, « Le vocabulaire »).
2. **Alertes est le seul contexte qui en sert plusieurs.**
   - S'il tombe, Qualité de l'air (et bientôt Trafic) ne peuvent plus prévenir personne.
   - C'est un argument pour qu'il reste un **service à part**, avec ses propres copies. Nous le
     reprendrons aux étapes 3 et 6.
3. **Alertes garde son état en mémoire.** Au redémarrage, il oublie les identifiants déjà traités.
   C'est le défaut relevé au TP 1 (étape 4). En microservice, ce contexte aura besoin de **sa
   propre base**.

---

## Résumé

| Contexte | Données propres | Base aujourd'hui | Règle métier |
|---|---|---|---|
| Qualité de l'air | mesures | `mesures.db` (SQLite) | moyenne PM10 du jour > 50 µg/m³ |
| Trafic | relevés, tronçons, états | `releves.db` (SQLite) + `troncons.json` | moyenne sur 15 min < 40 % de la limite, signalée une fois |
| Alertes | identifiants déjà traités | aucune (mémoire) | une alerte n'agit qu'une fois |
| Capteurs | aucune (transit) | aucune | transmettre la mesure |

Ces quatre contextes sont la base des deux dessins de l'étape 2 :
- en monolithe, ils deviennent **trois hexagones dans un même processus** (Capteurs reste un
  service Python à part) ;
- en microservices, ils deviennent **un service par contexte, chacun avec sa base**.
