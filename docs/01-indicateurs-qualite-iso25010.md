# Indicateurs qualité ISO 25010 — Collector.shop

Ce document présente les quatre indicateurs retenus pour piloter la qualité du POC, leur seuil, leur valeur mesurée et la justification anti dette technique.

Les valeurs ci-dessous sont mesurées sur les branches de travail courantes :

- `collector-catalog-api` : `feature/quality-gates` (commit `1523bc5`)
- `collector-moderation-worker` : `feature/quality-gates` (commit `e32695e`)
- `collector-frontend` : `feature/seller-article-status` (commit `b64bf30`)

## 1. Synthèse

| # | Indicateur | Caractéristique ISO 25010 | Seuil | Valeur mesurée | Statut |
|---|---|---|---|---|---|
| 1 | Couverture de tests (%) | Fiabilité, Maintenabilité | API : statements 85, branches 60, functions 80, lines 85 | API : statements 88,23 ; branches 66,66 ; functions 81,48 ; lines 86,95 | Respecté (API) |
| 2 | Dette technique, duplication, complexité | Maintenabilité | Quality Gate SonarCloud | Quality Gate passé sur les PR API #10 et worker #3 | Respecté, métriques détaillées à reporter |
| 3 | Temps de réponse API p95 sous charge | Efficacité de performance, Fiabilité | GET < 500 ms, POST < 800 ms | GET 466 ms, POST 679 ms, 0 % d'erreur | Respecté |
| 4 | Vulnérabilités par sévérité (dépendances + image) | Sécurité | 0 critique, 0 haute (bloquant CI) | `npm audit --omit=dev` : 0 vulnérabilité sur les trois services | Respecté côté dépendances npm, scan Trivy à reporter depuis la CI |

## 2. Indicateur 1 — Couverture de tests

Mesure réalisée avec `npx jest --coverage` (API et worker) et `vitest run --coverage` (frontend).

| Service | Tests | Statements | Branches | Functions | Lines | Seuil bloquant |
|---|---|---|---|---|---|---|
| `collector-catalog-api` | 22 (2 suites) | 88,23 % (165/187) | 66,66 % (22/33) | 81,48 % (22/27) | 86,95 % (140/161) | Oui (Jest) |
| `collector-moderation-worker` | 12 (2 suites) | 59,34 % (54/91) | 72,72 % (16/22) | 76,92 % (10/13) | 63,63 % (49/77) | Non |
| `collector-frontend` | 17 (5 fichiers) | 83,13 % (355/427) | 87,62 % (85/97) | 62,50 % (15/24) | 83,13 % (355/427) | Non |

Lecture :

- L'API respecte tous ses seuils. Le seuil est bloquant en CI.
- Le worker est sous le seuil de l'API : c'est un écart connu, à corriger dans le plan de remédiation (voir [plan de remédiation](06-plan-remediation-securite.md)).
- Le frontend a une bonne couverture des lignes mais peu de fonctions testées (62,5 %).

## 3. Indicateur 2 — Dette technique, duplication, complexité

- Analyse SonarCloud branchée sur les pipelines (organisation `leopoldpetit`).
- Quality Gate SonarCloud passé sur la PR `collector-catalog-api` #10 (seuil bloquant) et la PR `collector-moderation-worker` #3 (non bloquant).
- Les métriques détaillées (dette en minutes, duplication en %, complexité cyclomatique, issues par sévérité) doivent être reportées depuis le tableau de bord SonarCloud au moment de la soutenance. Elles ne sont pas encore figées dans ce document.

## 4. Indicateur 3 — Temps de réponse API sous charge

Test de charge k6 v2.3.0 sur le script `collector-infra/load-tests/articles.js`, cluster Minikube, via l'Ingress.

| Scénario | VUs | Endpoint | p95 mesuré | Seuil | Erreurs |
|---|---|---|---|---|---|
| `get_articles` | 20 | `GET /articles` | 466 ms | < 500 ms | 0 % |
| `post_articles` | 5 | `POST /articles` | 679 ms | < 800 ms | 0 % |

Bilan : 9 173 requêtes, environ 83 requêtes par seconde, 0 % d'erreur. Environ 1 730 brouillons ont été créés pendant le test.

Point d'attention : chaque `GET /articles` renvoie environ 360 Ko, car l'API ne pagine pas la liste. Le volume reçu est d'environ 2,7 Go sur le test. Le p95 reste sous le seuil, mais le volume est un risque de performance réel (voir le plan de remédiation).

## 5. Indicateur 4 — Vulnérabilités par sévérité

Deux sources, toutes deux utilisées en CI :

- **Dépendances npm** : `npm audit --omit=dev --audit-level=high`. Résultat sur les trois services : 0 vulnérabilité (critique, haute, modérée, faible).
- **Images Docker** : Trivy avec `CRITICAL,HIGH` en `exit-code 1` via le workflow réutilisable `collector-infra/.github/workflows/reusable-lint-test-scan-build.yml`. Le binaire Trivy n'est pas installé en local ; les résultats à reporter sont ceux du dernier job CI réussi.

Historique : le scan a détecté des failles sur l'image API (`node:22-alpine` : paquets système, dépendances npm embarquées dans npm). Elles ont été traitées : image de base mise à jour, `apk upgrade`, npm retiré de l'image d'exécution, Prisma CLI épinglé en `5.22.0`, utilisateur `node`.

## 6. Justification anti dette technique

Chaque indicateur bloque une dérive précise avant qu'elle ne s'accumule :

- **Couverture** : le seuil bloquant empêche la fusion de code non testé. Le code du POC reste modifiable sans régression invisible.
- **SonarCloud** : la dette est visible à chaque PR, pas seulement en fin de projet.
- **Performance** : le test de charge est reproductible et chiffré. Une régression de p95 est détectable avant la mise en production.
- **Vulnérabilités** : un scan bloquant empêche d'empiler des CVE critiques ou hautes non traitées.
