# Guide perso oral - TaskFlow (Etape 1 + Etape 2)

Ce document est un pense-bete pour presenter rapidement ce qui etait demande, ce qui a ete fait, le resultat attendu, et comment le prouver en live.

---

## Etape 1 - Containerisation (45 min)

### 1) Ce qu'on devait faire

- Faire tourner les 3 services avec une seule commande depuis la racine: `docker compose up --build`.
- Avoir 3 services:
  - frontend (nginx)
  - backend (node)
  - cache (redis)
- Ne pas exposer Redis vers l'exterieur.
- Utiliser des variables d'env (pas de secret en dur).
- Avoir un volume nomme pour la persistance Redis.
- Isoler les services via reseaux Docker.
- Backend Dockerfile:
  - multi-stage
  - dependances prod seulement dans l'image finale
  - user non-root
  - port 3001 expose
- Frontend Dockerfile:
  - base `nginx:alpine`
  - copie fichiers statiques + conf nginx
  - port 80 expose
- Fichiers hygiene:
  - `.env.example`
  - `.dockerignore`

### 2) Comment on doit le faire (methode)

- Backend:
  - Copier `package*.json` avant le code pour utiliser le cache Docker.
  - Faire `npm ci --omit=dev` dans un stage build.
  - Copier seulement le necessaire dans stage final.
  - Ajouter `USER` non-root + `HEALTHCHECK`.
  - Ajouter labels OCI (version, authors, source).
- Frontend:
  - Image nginx alpine.
  - Copier `nginx.conf`, `index.html`, `app.js`, `styles.css`.
- Compose:
  - `depends_on` avec `condition: service_healthy`.
  - `healthcheck` Redis avec `redis-cli ping`.
  - `restart: unless-stopped`.
  - reseau interne pour backend/cache.
  - volume `redis-data`.

### 3) Resultat attendu

- En local clone propre + Docker Desktop => la stack demarre sans installer Node/Redis localement.
- Frontend accessible via port web.
- Backend repond sur `/health`.
- Redis est prive (pas de port mappe vers host).
- Les donnees Redis restent apres `docker compose down` puis `docker compose up -d` (tant qu'on ne fait pas `down -v`).

### 4) Comment tester / montrer a l'oral

Ordre de demo court (2-3 min):

1. Montrer les fichiers cle:
   - `backend/Dockerfile`
   - `frontend/Dockerfile`
   - `docker-compose.yml`
   - `.env.example`
   - `backend/.dockerignore` et `frontend/.dockerignore`

2. Build + run:

```bash
docker compose up --build -d
docker compose ps
```

3. Verifier backend:

```bash
curl http://localhost:3001/health
```

4. Verifier Redis non expose:

```bash
docker compose ps
# Pas de mapping de port pour service cache
```

5. Prouver persistance Redis:

```bash
docker compose exec cache redis-cli SET demo "ok"
docker compose down
docker compose up -d
docker compose exec cache redis-cli GET demo
```

Tu dois voir `ok`.

### 5) Questions frequentes jury (reponses courtes)

- Pourquoi `unless-stopped` ?
  - Si backend crash a 3h du matin, il redemarre automatiquement.
- Pourquoi `depends_on: service_healthy` ?
  - Le backend attend Redis pret, evite les erreurs au boot.
- Pourquoi multi-stage ?
  - Image plus petite, moins de surface d'attaque.
- Pourquoi user non-root ?
  - Reduction du risque securite.

---

## Etape 2 - Pipeline CI/CD (1h15)

### 1) Ce qu'on devait faire

- Workflow GitHub Actions dans `.github/workflows/ci.yml`.
- Triggers:
  - push sur `main` et `staging`
  - pull_request vers `main`
  - tags `v*`
- Jobs attendus:
  1. test (Node 18 + 20)
  2. lint (bloquant)
  3. audit npm (bloquant sur HIGH/CRITICAL)
  4. build Docker + scan Trivy CRITICAL + push Docker Hub (tag seulement)
  5. deploy-staging
  6. smoke-test staging (`/health`)
  7. deploy-production (tag)

### 2) Comment on doit le faire (methode)

- `test` en matrix pour Node 18/20.
- `lint` et `audit` apres `test` avec `needs`.
- `audit`:
  - `npm audit --audit-level=high`
  - upload rapport json en artefact si echec
  - fail final du job
- `build` (tag `v*`):
  - login Docker Hub avec secrets
  - build backend/frontend
  - trivy CRITICAL bloquant + rapports artefacts
  - push images
  - controle taille image backend
- Staging:
  - build staging dedie puis deploy-staging
  - smoke-test apres deploy
- Production:
  - deploy seulement apres build tag

### 3) Resultat attendu

- Push/PR standard => tests + lint + audit.
- Tag `vX.Y.Z` => build images, scan securite, push Docker Hub, deploy production.
- Push sur `staging` => build staging, deploy staging, smoke test OK.
- Pipeline rouge si:
  - test KO
  - lint KO
  - audit HIGH/CRITICAL
  - Trivy trouve CRITICAL

### 4) Comment tester / montrer a l'oral

Demo cible (3-5 min):

1. Ouvrir Actions et montrer le workflow vert.
2. Montrer les triggers dans `ci.yml`.
3. Montrer sequence des jobs:
   - `test` -> `lint` + `audit`
   - branche `staging`: `build-staging` -> `deploy-staging` -> `smoke-test`
   - tag `v*`: `build` -> `deploy-production`
4. Montrer artefacts de securite:
   - `audit-report`
   - `trivy-backend-report`
   - `trivy-frontend-report`
5. Montrer Docker Hub:
   - image backend avec tag de version
   - image frontend avec tag de version

### 5) Commandes utiles pour provoquer les cas en demo

- Cas test normal (push branche):

```bash
git checkout staging
git commit --allow-empty -m "ci: test staging pipeline"
git push origin staging
```

- Cas release (tag):

```bash
git checkout main
git pull
git tag v1.0.0
git push origin v1.0.0
```

### 6) Preconditions pour que la CI/CD passe vraiment

- Secrets GitHub configures:
  - `DOCKER_USERNAME`
  - `DOCKER_PASSWORD`
  - `KUBE_CONFIG_STAGING`
  - `KUBE_CONFIG_PRODUCTION`
- Environnements GitHub existants:
  - `staging`
  - `production`
- Manifests Kubernetes presents:
  - `k8s/staging/*`
  - `k8s/production/*`

Sans ces elements, la partie deploy ne peut pas reussir.

---

## Mini script oral (si tu bloques)

"Sur l'etape 1, l'objectif etait de rendre l'app portable et demarrable avec une seule commande Docker Compose. On a isole les services avec des reseaux, protege Redis, ajoute la persistance par volume, et securise le backend avec multi-stage + user non-root + healthcheck.

Sur l'etape 2, on a mis un pipeline CI/CD complet: qualite (tests/lint), securite (audit npm + Trivy), build et livraison Docker, puis deploiement automatique staging/production selon la branche ou le tag."

---

## Repartition binome conseillee (optionnel)

- Eleve A:
  - etape 1 complete
  - jobs test/lint/audit
- Eleve B:
  - build/push Docker + Trivy
  - deploy-staging/smoke/deploy-production

Important: chacun doit quand meme comprendre tout le pipeline pour les questions individuelles.
