# Cartographie des compétences et plan de formation — Collector.shop

Ce document reprend la cartographie des compétences du plan général, indique la preuve de chaque compétence dans le POC, identifie les écarts et propose le plan de formation associé.

## 1. Cartographie

| Compétence | Rôle porteur | Preuve dans le POC | Niveau constaté | Écart |
|---|---|---|---|---|
| Développement backend (NestJS, TypeScript, API REST) | Développeur confirmé | `collector-catalog-api` : modules, DTO, guards, tests unitaires et d'intégration (22 tests) | Maîtrisé | Aucun |
| Développement frontend (React) | Développeur confirmé | `collector-frontend` : Vite, catalogue public, espace vendeur, suivi du statut (17 tests) | Maîtrisé | Couverture des fonctions à améliorer (62,5 %) |
| Modélisation de données, SQL (PostgreSQL) | Développeur confirmé | Schéma Prisma, migration initiale, Prisma 5.22 | Maîtrisé | Pas de pagination sur `GET /articles` |
| Messagerie asynchrone (RabbitMQ) | Lead Developer | Files `moderation` et `moderation-results` durables, messages persistants, consommation au redémarrage du worker | Maîtrisé | Pas d'outbox : insertion en base et publication ne sont pas atomiques |
| Sécurité applicative (OAuth2/OIDC, Keycloak, scan de vulnérabilités) | Lead Developer | Realm Keycloak, JWT vérifié par JWKS, rôles `vendeur` et `acheteur`, scans npm audit et Trivy | Maîtrisé | Pas de rate limiting, pas de rotation des secrets |
| CI/CD (GitHub Actions, Docker) | Lead Developer | Workflow réutilisable lint, test, scan, build, push GHCR, appelé par les trois services | Maîtrisé | Pas de signature d'images |
| Orchestration (Kubernetes, Helm) | Lead Developer, montée en compétence des développeurs | Minikube, chart générique `collector-service`, Ingress NGINX, déploiement depuis GHCR | Acquis par le lead | Les développeurs n'ont pas encore contribué aux charts. Pas de NetworkPolicy. |
| Observabilité (Prometheus, Grafana, Loki) | Lead Developer | Stack `kube-prometheus-stack`, Loki, dashboard Grafana, métriques Ingress | Acquis par le lead | Pas d'alerting démontré |
| Gestion des secrets (Vault) | Lead Developer | Vault présent dans `docker-compose.yml` | Partiel | Vault non utilisé dans les charts Helm |

## 2. Plan de formation

### 2.1 Formation courte : sécuriser une chaîne CI/CD et un déploiement Kubernetes

| Élément | Détail |
|---|---|
| Public | Les deux développeurs confirmés |
| Durée | 2 à 3 jours |
| Objectif | Contribuer aux pipelines CI/CD et aux Helm charts, et pas seulement au code applicatif |
| Contenu | Sécurisation des workflows GitHub Actions (permissions, épinglage des actions par SHA), scan d'images et de dépendances, Helm et valeurs par environnement, Kubernetes : NetworkPolicy, Secrets, contextes de sécurité des pods |
| Mise en pratique | Ajouter une NetworkPolicy et un contexte de sécurité restrictif sur un service du POC |
| Critère de réussite | Un développeur ajoute une étape de scan dans un pipeline et modifie un chart Helm, relu par le lead |

### 2.2 Atelier interne : Vault et bonnes pratiques OWASP

| Élément | Détail |
|---|---|
| Animateur | Lead Developer |
| Durée | 1/2 journée |
| Contenu | Injection de secrets depuis Vault (Kubernetes auth), rotation, Top 10 OWASP appliqué au flux de publication (contrôle d'accès, validation des entrées, journalisation) |
| Mise en pratique | Migrer un secret du POC (mot de passe base ou secret client Keycloak) vers Vault |
| Critère de réussite | Aucun secret runtime n'est présent dans les charts ou les values |

### 2.3 Calendrier indicatif

| Action | Semaine | Prérequis |
|---|---|---|
| Atelier Vault et OWASP | Semaine 5 | Plan de remédiation validé |
| Formation CI/CD et Kubernetes | Semaine 5 ou après la soutenance | Budget et disponibilité des développeurs |

## 3. Lien avec le plan de remédiation

Les écarts de compétences recoupent les actions du [plan de remédiation sécurité](06-plan-remediation-securite.md) : intégration de Vault, NetworkPolicy, rate limiting et alerting. La formation rend ces actions réalisables par l'équipe, sans dépendre d'une seule personne.
