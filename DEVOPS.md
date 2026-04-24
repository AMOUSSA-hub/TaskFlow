# DEVOPS.md - TaskFlow

## Contexte

Objectif global du projet: rendre TaskFlow production-ready avec:

- containerisation Docker
- pipeline CI/CD GitHub Actions
- deploiement Kubernetes sur 2 environnements (staging et production)

Ce document synthétise nos choix techniques, la sécurité (Trivy), et les difficultés rencontrées.

---

## Choix Technique 1 - Image de base backend

### Options évaluées

- `node:18-alpine`
- `node:20-alpine`

### Commandes de mesure (a exécuter)

```bash
# Build variantes
cp backend/Dockerfile backend/Dockerfile.node18-alpine
# puis remplacer node:20-alpine -> node:18-alpine dans ce fichier

docker build -f backend/Dockerfile.node18-alpine -t taskflow-backend:node18-alpine ./backend
docker build -f backend/Dockerfile -t taskflow-backend:node20-alpine ./backend

# Taille image
# Linux/macOS:
docker image inspect taskflow-backend:node18-alpine --format '{{.Size}}'
docker image inspect taskflow-backend:node20-alpine --format '{{.Size}}'

# Windows PowerShell:
docker image inspect taskflow-backend:node18-alpine --format "{{.Size}}"
docker image inspect taskflow-backend:node20-alpine --format "{{.Size}}"

# CVE via Trivy (container officiel)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL taskflow-backend:node18-alpine
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL taskflow-backend:node20-alpine
```

### Résultat retenu

- Choix final: `node:20-alpine`

### Justification

- Version Node plus récente (support/patch sécurité plus actuels).
- Image alpine compacte.
- Compatible avec notre Dockerfile multi-stage et notre pipeline CI.

### Chiffres à reporter (oral)

- Taille `node:18-alpine`: `A COMPLETER`
- CVE HIGH/CRITICAL `node:18-alpine`: `A COMPLETER`
- Taille `node:20-alpine`: `A COMPLETER`
- CVE HIGH/CRITICAL `node:20-alpine`: `A COMPLETER`

---

## Choix Technique 2 - Politique de redémarrage Docker Compose

### Options évaluées

- `unless-stopped`
- `on-failure`
- `always`

### Résultat retenu

- Choix final: `unless-stopped` (appliqué à frontend/backend/cache dans `docker-compose.yml`)

### Justification

- Si le backend plante à 3h du matin, il redémarre automatiquement.
- Si un opérateur stoppe manuellement un conteneur (`docker stop`), il ne redémarre pas tout seul (contrairement à `always`).
- Comportement équilibré entre résilience et contrôle opérationnel.

### Comportement attendu à 3h du matin

- `unless-stopped`: redémarrage automatique en cas de crash.
- `on-failure`: redémarrage seulement si code de sortie != 0.
- `always`: redémarrage systématique, même après arrêt manuel.

---

## Choix Technique 3 - Stratégie RollingUpdate Kubernetes

### Options évaluées

- `maxUnavailable: 0 / maxSurge: 1`
- `maxUnavailable: 1 / maxSurge: 1`
- `maxUnavailable: 1 / maxSurge: 0`

### Résultat retenu

- Staging: `maxUnavailable: 1`, `maxSurge: 1`
- Production: `maxUnavailable: 0`, `maxSurge: 1`

### Justification

- Staging: compromis vitesse/disponibilité pour itérer plus vite.
- Production: privilégie la continuité de service (zéro indisponibilité voulue pendant rollout).

### Preuve orale

```bash
kubectl rollout status deployment/taskflow-backend -n taskflow-staging
kubectl rollout status deployment/taskflow-backend -n taskflow-production
```

---

## Choix Technique 4 - Nombre de replicas

### Options évaluées

- 1 staging / 3 production
- 2 staging / 3 production
- 2 staging / 5 production
- 1 staging / 1 production

### Résultat retenu

- Staging: 2 replicas backend
- Production: 3 replicas backend

### Justification

- 2 replicas en staging permettent de tester le rolling update et le self-healing dans de bonnes conditions.
- 3 replicas en production améliorent la disponibilité et la tolérance à la panne.
- Cohérent avec readiness/liveness probes et stratégie de déploiement choisie.

### Preuve orale

```bash
kubectl get deploy -n taskflow-staging
kubectl get deploy -n taskflow-production
kubectl get pods -n taskflow-production -w
```

---

## Sécurité - Ce que Trivy a trouvé et gestion

### Ce qui est en place

- Trivy est intégré dans la CI (`.github/workflows/ci.yml`) sur backend et frontend.
- Le scan est bloquant sur `CRITICAL`.
- Le rapport Trivy est toujours uploadé en artefact, même en cas d'échec.

### Politique de gestion des vulnérabilités

- `CRITICAL`: build en échec (pas de push/deploy)
- `HIGH`: visible dans rapport, traité selon impact et patch disponible
- Correction prioritaire:
  - mise à jour image de base
  - mise à jour dépendances applicatives
  - rebuild + re-scan

### A montrer à l'oral

- Un run GitHub Actions
- Les artefacts `trivy-backend-report` et `trivy-frontend-report`
- La règle bloquante sur CRITICAL dans `ci.yml`

---

## Difficulté rencontrée et résolution

### Problème

Le job `deploy-staging` était initialement dépendant d'un `build` exécuté uniquement sur tags `v*`.
Conséquence: sur push `staging`, le build était `skipped` et le déploiement staging ne s'exécutait pas.

### Résolution

- Création d'un job `build-staging` dédié à la branche `staging`.
- `deploy-staging` dépend désormais de `build-staging`.
- `smoke-test` s'exécute après `deploy-staging`.

### Bénéfice

Le flux staging est maintenant cohérent:

- push `staging` -> test/lint/audit -> build-staging -> deploy-staging -> smoke-test

et le flux production reste propre:

- tag `v*` -> build (scan/push) -> deploy-production

---

## Commandes de démo rapide (soutenance)

```bash
# Docker local
docker compose up --build -d
docker compose ps
curl http://localhost:3001/health

# Persistance Redis (étape 1)
docker compose exec cache redis-cli SET demo "ok"
docker compose down
docker compose up -d
docker compose exec cache redis-cli GET demo

# Kubernetes (étape 3, utile pour raconter les choix 3/4)
kubectl get pods -n taskflow-staging
kubectl get pods -n taskflow-production
kubectl rollout status deployment/taskflow-backend -n taskflow-production
```
