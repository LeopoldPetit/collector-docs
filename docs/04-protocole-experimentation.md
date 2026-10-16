# Protocole d'expérimentation bac à sable — RabbitMQ, Keycloak, Minikube

Ce document décrit les expérimentations menées dans un environnement de bac à sable avant d'intégrer les briques au POC. Pour chaque expérimentation : l'hypothèse, le protocole, le résultat observé et la décision prise.

Environnement : macOS, Docker Desktop, `collector-infra/docker-compose.yml` (PostgreSQL, RabbitMQ, Keycloak, Vault) pour le développement local, puis Minikube avec le driver Docker pour le déploiement Kubernetes.

## 1. Expérimentation A — Messagerie RabbitMQ et tolérance à l'indisponibilité du worker

### 1.1 Hypothèse

Un article soumis pendant que le worker de modération est arrêté reste en file et est traité à son redémarrage. Aucun message n'est perdu.

### 1.2 Protocole

1. Démarrer RabbitMQ en local.
2. Arrêter le worker `collector-moderation-worker`.
3. Publier un article via `POST /articles` (rôle `vendeur`).
4. Vérifier que le message `article.submitted` est présent dans la file `moderation`.
5. Redémarrer le worker.
6. Vérifier la publication de `article.validated` et la mise à jour du statut de l'article.

### 1.3 Résultat

- Le message reste dans la file `moderation` pendant l'arrêt du worker.
- À son redémarrage, le worker consomme le message et publie `article.validated`.
- Le message publié porte `delivery_mode = 2` (persistant).

### 1.4 Constat et décision

Les files sont déclarées `durable`, mais la durabilité de la file ne suffit pas : **les messages doivent aussi être publiés en `persistent: true`**. Ce réglage est appliqué dans l'API et dans le worker.

Limite connue : l'insertion en base de l'article et la publication du message ne sont pas atomiques. Si l'API tombe entre les deux, l'article reste en brouillon sans message. Le pattern *transactional outbox* est repris dans le [plan de remédiation](06-plan-remediation-securite.md).

## 2. Expérimentation B — Contrôle d'accès Keycloak et JWT

### 2.1 Hypothèse

L'API accepte uniquement les JWT émis par le realm `collector-shop`, et réserve `POST /articles` au rôle `vendeur`.

### 2.2 Protocole

1. Obtenir un token par le flux *Resource Owner Password Credentials* (client public `collector-frontend`, `directAccessGrantsEnabled: true`) pour les utilisateurs `vendeur-demo` et `acheteur-demo`.
2. Appeler `GET /articles` sans token, puis avec token.
3. Appeler `POST /articles` sans token, avec le token acheteur, puis avec le token vendeur.

### 2.3 Résultat

| Appel | Attendu | Observé |
|---|---|---|
| `GET /articles` sans token | 200 (catalogue public) | 200 |
| `POST /articles` sans token | 401 | 401 |
| `POST /articles` avec token acheteur | 403 | 403 |
| `POST /articles` avec token vendeur | 201 | 201 |

Le test d'intégration `articles.integration.spec.ts` reproduit ces cas avec une clé RSA locale et un JWKS simulé. Il s'exécute dans `npm test`.

### 2.4 Constats et décisions

- Les utilisateurs de démonstration n'avaient ni `firstName` ni `lastName`. Keycloak refusait alors le login avec le message « Account is not fully set up ». Le fichier `realm/collector-shop-realm.json` a été corrigé.
- La durée de vie des tokens est de 300 secondes. Le script k6 obtient un token dans `setup()` au début du test. Un test plus long que 300 secondes devrait renouveler ce token.
- Juste après `docker compose up`, l'endpoint de token peut renvoyer une réponse non JSON tant que Keycloak démarre. Il faut attendre avant de demander un token.

## 3. Expérimentation C — Minikube, Ingress et images Docker

### 3.1 Hypothèse

Un cluster Minikube avec l'addon `ingress-nginx` peut tirer les images des trois services et les exposer en HTTPS.

### 3.2 Protocole

1. Démarrer Minikube (driver Docker, 2 CPU, 3 Go) et activer l'addon `ingress-nginx`.
2. Déployer les services avec `helm upgrade --install` et `pullPolicy` adapté.
3. Vérifier les pods et l'accès via l'Ingress.

### 3.3 Résultat

- Les pods dans la VM Minikube ne peuvent pas tirer les images du registre : erreur TLS, due au proxy d'entreprise.
- Contournement retenu : `docker pull` sur l'hôte, puis `minikube image load <image:tag>`.
- La forme `image@digest` échoue silencieusement avec `minikube image load`. Il faut utiliser `image:tag`.
- Tirer depuis GHCR dans le cluster a échoué avec une erreur x509, même contournement.
- Les builds Docker locales échouent à télécharger le moteur Prisma (même cause TLS). Elles ne sont pas utilisées pour le déploiement.
- Une fois les packages GHCR publics, le tirage anonyme fonctionne (le contrôle anonyme exige le header `Accept`, sinon on obtient un faux 404).

### 3.4 Décisions

- Déploiement final : images GHCR par tag SHA, `pullPolicy: IfNotPresent`.
- Le port-forward vers l'Ingress (`8443:443`) est fragile : il meurt quand le pod de l'Ingress redémarre. Il faut tuer le processus résiduel qui occupe le port avant de le relancer.
- Les métriques de l'Ingress exigent `--enable-metrics=true` sur le contrôleur. Ce réglage est documenté dans le README d'infra.

## 4. Synthèse

| Expérimentation | Hypothèse validée | Ajustement retenu |
|---|---|---|
| A — RabbitMQ | Oui, avec condition | Messages persistants, outbox à prévoir |
| B — Keycloak | Oui | Correction du realm, durée de token courte |
| C — Minikube | Oui, avec contournements | Images GHCR par tag SHA, pull par l'hôte si besoin |

## 5. Limites

- Le bac à sable n'est pas un cluster de production : certificats auto-signés, un seul nœud, pas de haute disponibilité.
- Les tests se font sur des données de démonstration.
- Les expérimentations ne couvrent pas la sécurité du réseau (pas de NetworkPolicy).
