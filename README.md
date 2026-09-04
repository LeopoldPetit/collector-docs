# collector-docs

Livrables écrits du projet **Collector.shop**, réalisé dans le cadre du bloc de compétences *"Superviser et assurer le développement des applications logicielles"*.

## Rôle dans le projet

Ce repo centralise toute la documentation transverse qui n'appartient pas à un service applicatif en particulier :

- Indicateurs qualité (ISO 25010) et justification anti dette technique
- Cycle de vie DevSecOps détaillé
- Cartographie des compétences et plan de formation
- Protocole d'expérimentation bac à sable (RabbitMQ, Keycloak, Minikube)
- Backlog complet et schémas d'architecture / CI/CD
- Plan de remédiation sécurité (phase 3)

## Contexte du projet

**US mère** : *"Publication d'un article par un vendeur avec contrôle automatique avant mise en vente"*.

Le prototype couvre : processus qualité (ISO 25010), pipeline CI/CD + tests, sécurité minimale, une fonctionnalité métier, et un plan de remédiation sécurité.

## Repos du projet

| Repo | Contenu |
|---|---|
| [`collector-infra`](https://github.com/LeopoldPetit/collector-infra) | Docker Compose, charts Helm, workflows CI/CD réutilisables, Vault, cert-manager |
| [`collector-catalog-api`](https://github.com/LeopoldPetit/collector-catalog-api) | API métier NestJS (articles, catégories, orchestration du contrôle) |
| [`collector-moderation-worker`](https://github.com/LeopoldPetit/collector-moderation-worker) | Worker de contrôle automatisé (consommateur RabbitMQ) |
| [`collector-frontend`](https://github.com/LeopoldPetit/collector-frontend) | App React (catalogue public + espace vendeur) |
| `collector-docs` | Ce repo — livrables écrits |

## Contenu attendu

```
00-plan-general.md              # rôle, US mère, macro-planning, compétences, indicateurs qualité, cycle DevSecOps
01-architecture.md              # stack technique, schémas d'architecture et de pipeline CI/CD, sécurité, observabilité
02-backlog-repos-commits.md     # backlog priorisé, séquence d'exécution, stratégie de branches/commits
03-plan-remediation.md          # plan de remédiation sécurité (phase 3, après analyse des tests/scans)
```

## Les 3 phases du projet

| Phase | Contenu | Repos concernés |
|---|---|---|
| Phase 1 — Structuration | Indicateurs qualité, cycle DevSecOps, schéma CI/CD, compétences/formation | `collector-docs`, `collector-infra` |
| Phase 2 — Développement & déploiement | Backlog détaillé, archi technique, expérimentations, implémentation POC, pipeline CI/CD, sécurité, observabilité | `collector-infra`, `collector-catalog-api`, `collector-moderation-worker`, `collector-frontend` |
| Phase 3 — Remédiation | Analyse des résultats de tests/métriques, vulnérabilités, plan de remédiation priorisé | `collector-docs` |

## Livrable final

Soutenance individuelle (20 min de présentation + 15 min d'échange) devant jury.
