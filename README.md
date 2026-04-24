# TaskFlow

Application web de gestion de tâches. Interface Kanban avec backend Node.js et persistance Redis.

## Stack technique

| Couche   | Technologie              | Rôle                                   |
|----------|--------------------------|----------------------------------------|
| Frontend | HTML/CSS/JS vanilla      | Interface Kanban, servie par Nginx     |
| Backend  | Node.js (sans framework) | API REST — logique métier              |
| Stockage | Redis 7                  | Persistance des tâches et stats        |

## Structure du projet

```
taskflow/
├── frontend/
│   └── index.html          ← interface Kanban
├── backend/
│   ├── server.js           ← API REST
│   ├── server.test.js      ← tests unitaires
│   └── package.json
├── .env.example
├── .gitignore
└── README.md
```

## Lancer le projet avec Docker Compose

### Prérequis

- Docker Desktop démarré

### 1. Cloner le projet

```bash
git clone https://github.com/[FORMATEUR]/taskflow.git
cd taskflow
```

### 2. Créer le fichier de configuration

```bash
cp .env.example .env
```

### 3. Build et démarrage des 3 services

```bash
docker compose up --build -d
```

### 4. Vérifications rapides

```bash
docker compose ps
curl http://localhost:3001/health
```

### 5. Arrêt des services

```bash
docker compose down
```

### Prouver la persistance Redis (oral)

```bash
# Ecrire une valeur dans Redis
docker compose exec cache redis-cli SET demo "ok"

# Stopper la stack puis la relancer
docker compose down
docker compose up -d

# Vérifier que la valeur est encore présente
docker compose exec cache redis-cli GET demo
```

Ne pas utiliser `docker compose down -v` pour ce test, sinon le volume est supprimé.

## Lancer le backend sans Docker (optionnel)

```bash
docker run -d -p 6379:6379 --name redis-dev redis:7-alpine
cd backend
npm install
npm start
```

## Tests et lint

```bash
cd backend
npm test        # tests unitaires — aucune connexion Redis requise
npm run lint    # vérification ESLint
```

## API

| Méthode | Route        | Body                                           | Description             |
|---------|--------------|------------------------------------------------|-------------------------|
| GET     | /health      | —                                              | État de l'app           |
| GET     | /tasks       | —                                              | Liste toutes les tâches |
| POST    | /tasks       | `{ title, description?, priority? }`           | Créer une tâche         |
| PUT     | /tasks/:id   | `{ title?, description?, status?, priority? }` | Modifier une tâche      |
| DELETE  | /tasks/:id   | —                                              | Supprimer une tâche     |

Valeurs `status` : `todo` · `in-progress` · `done`  
Valeurs `priority` : `low` · `medium` · `high`

## Variables d'environnement

| Variable      | Défaut                   | Description                   |
|---------------|--------------------------|-------------------------------|
| `PORT`        | `3001`                   | Port du backend               |
| `APP_ENV`     | `development`            | Environnement                 |
| `APP_VERSION` | `1.0.0`                  | Version affichée dans /health |
| `REDIS_URL`   | `redis://localhost:6379` | URL de connexion Redis        |

---

Ce projet est la base du projet final DevOps — Bachelor 3 Développement.  
Votre mission : le containeriser, automatiser sa livraison, et le déployer sur Kubernetes.
