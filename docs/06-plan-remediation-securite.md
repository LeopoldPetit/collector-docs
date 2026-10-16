# Plan de remédiation sécurité (phase 3) — Collector.shop

Ce plan est rédigé à partir des résultats des tests de charge (Étape 10), des audits de dépendances et des scans de la CI. Chaque action est priorisée et justifiée par son risque métier.

Statut du document : **version provisoire**. Le rapport Trivy complet n'est pas encore relu en local (voir section 2.2). Il doit être complété avant la soutenance.

## 1. Périmètre et sources

| Source | Données utilisées |
|---|---|
| `npm audit --omit=dev` (API, worker, frontend) | Vulnérabilités des dépendances de production |
| Trivy (CI, `CRITICAL,HIGH`, bloquant) | Vulnérabilités des images Docker |
| Tests de charge k6 (`collector-infra/load-tests/README.md`) | Temps de réponse, erreurs, volume |
| Audit des charts et du code (vérifications effectuées) | Rate limiting, Vault, NetworkPolicy, pagination, alerting |

## 2. Vulnérabilités classées par sévérité

### 2.1 Dépendances npm

| Service | Critique | Haute | Modérée | Faible | Total |
|---|---|---|---|---|---|
| `collector-catalog-api` | 0 | 0 | 0 | 0 | 0 |
| `collector-moderation-worker` | 0 | 0 | 0 | 0 | 0 |
| `collector-frontend` | 0 | 0 | 0 | 0 | 0 |

Historique : une vulnérabilité haute sur `platform-express`, `multer` et `proxy-addr` a été corrigée par `npm audit fix --omit=dev`.

### 2.2 Images Docker (Trivy)

| Image | Critique | Haute | Action |
|---|---|---|---|
| `collector-catalog-api` | Pas de blocage à la dernière CI verte | Idem | Récupérer le rapport du dernier job CI et le reporter ici |
| `collector-moderation-worker` | Idem | Idem | Idem |
| `collector-frontend` | Idem | Idem | Idem |

Les failles précédemment remontées concernaient les paquets système de l'image de base et les dépendances de npm embarquées dans l'image. Elles ont été traitées (voir [indicateurs qualité](01-indicateurs-qualite-iso25010.md)).

Le binaire Trivy n'est pas installé localement. Pour finaliser ce tableau : lancer `trivy image --severity CRITICAL,HIGH <image:tag>` sur les trois images, ou relever le dernier job CI.

## 3. Analyse des tests de charge

| Indicateur | Résultat | Seuil | Lecture |
|---|---|---|---|
| p95 `GET /articles` | 466 ms | < 500 ms | Proche du seuil. Marge faible. |
| p95 `POST /articles` | 679 ms | < 800 ms | Marge de 121 ms |
| Taux d'erreur | 0 % sur 9 173 requêtes | 0 % | Aucune erreur sous charge |
| Débit | environ 83 req/s | n/a | Suffisant pour le POC |
| Volume reçu | environ 2,7 Go | n/a | Excessif : environ 360 Ko par `GET /articles` |
| Données créées | environ 1 730 brouillons | n/a | Aucune limite de création par vendeur |

Trois constats :

1. **Pas de pagination.** Chaque `GET /articles` renvoie tout le catalogue publié. Le volume croît avec le catalogue, et le p95 se dégradera.
2. **Pas de limite de débit.** Un client a pu créer environ 1 730 brouillons sans être freiné.
3. **Marge faible sur `GET`.** Le p95 est à 466 ms pour un seuil de 500 ms.

## 4. Plan d'actions priorisé

Priorité : P1 = à traiter avant la mise en production, P2 = à traiter ensuite, P3 = évolution.

| Priorité | Action | Risque métier | Constat | Effort indicatif |
|---|---|---|---|---|
| P1 | Limiter le débit (rate limiting) sur `POST /articles` et `GET /articles` (Ingress ou guard NestJS) | Spam et fraude : publications massives qui saturent le contrôle de modération et polluent le catalogue | 1 730 brouillons créés sans limite | Moyen |
| P1 | Paginer `GET /articles` (`limit`, `cursor`, taille maximale) | Indisponibilité et coût de bande passante lors d'un pic d'audience | 360 Ko par réponse | Faible |
| P1 | Désactiver le flux *password grant* sur le client `collector-frontend` en production, et passer à *authorization code + PKCE* | Exposition des identifiants saisis dans le client | `directAccessGrantsEnabled: true` pour les tests | Moyen |
| P1 | Supprimer les comptes de démonstration (`vendeur-demo`, `acheteur-demo`) du realm de production | Accès non autorisé avec des identifiants connus | Mots de passe identiques aux identifiants | Faible |
| P1 | Mettre en place le pattern *transactional outbox* pour `article.submitted` | Article bloqué en `DRAFT` si l'API tombe entre l'insertion et la publication | Constat de l'expérimentation A | Moyen |
| P2 | Intégrer Vault aux services (secrets runtime injectés, rotation) | Fuite de secrets en cas d'accès aux charts ou aux values | Vault présent en dev uniquement | Élevé |
| P2 | Ajouter des NetworkPolicy (API accessible uniquement par l'Ingress, worker sans accès externe) | Déplacement latéral en cas de compromission d'un pod | Aucune NetworkPolicy | Moyen |
| P2 | Porter la couverture du worker au niveau de l'API et rendre le job SonarCloud bloquant | Régression non détectée dans le contrôle de modération, publication d'articles non conformes | Couverture statements 59 %, job non bloquant | Moyen |
| P2 | Alerting Prometheus : pic de 5xx, taux de 401 et 403 anormal, file RabbitMQ qui grossit | Détection tardive d'une panne ou d'une tentative d'accès | Dashboard sans règle d'alerte | Faible |
| P2 | Ajouter un test d'acceptation automatisé du flux de publication dans la CI | Régression silencieuse du flux de bout en bout | Test manuel uniquement | Moyen |
| P3 | Signer les images (cosign) et générer un changelog | Image falsifiée déployée | Non réalisé | Faible |
| P3 | Filtrage des données personnelles dans la messagerie (email, téléphone) | Fuite de coordonnées entre utilisateurs | Pas de fonctionnalité de chat dans le POC | À évaluer si le chat est ajouté |

## 5. Justification des actions P1

- **Rate limiting** : sans limite, un vendeur peut inonder la file `moderation` et le catalogue. Le risque est une fraude sur les prix et un catalogue pollué, avant même que le contrôle automatique n'agisse.
- **Pagination** : avec 360 Ko par réponse, un pic de trafic consomme la bande passante et dégrade le service pour tous. C'est un risque de disponibilité direct.
- **Flux d'authentification** : le *password grant* oblige le client à manipuler les identifiants. Il est acceptable pour des tests, pas pour la production.
- **Comptes de démonstration** : des identifiants identiques aux noms d'utilisateur sont un accès trivial. Ils doivent être retirés avant toute mise en ligne.
- **Outbox** : un article resté en `DRAFT` sans contrôle est invisible pour le vendeur et bloque la mise en vente. C'est un risque opérationnel et de confiance.

## 6. Suivi

| Action | Responsable | Échéance | Statut |
|---|---|---|---|
| Compléter le tableau Trivy (section 2.2) | Lead Developer | Avant soutenance | À faire |
| Revue et validation du plan | Lead Developer | Semaine 5 | À faire |
| Actions P1 | Équipe | Avant mise en production | Non démarré |
| Actions P2 | Équipe | Après P1 | Non démarré |

Le plan de formation associé est décrit dans la [cartographie des compétences](03-competences-formation.md).
