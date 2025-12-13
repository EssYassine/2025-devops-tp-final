# TP Final DevOps : Stratégie et Mise en Œuvre

Ce document décrit la stratégie DevOps mise en place pour le projet **Christmas Gift List**, réalisé dans le cadre du TP final du module DevOps (M1 Cyber). 

L'objectif est de présenter toutes les étapes nécessaires pour passer d'un code local à une application entièrement conteneurisée, déployable sur différents environnements, avec une pipeline CI/CD automatisée.  

### Environnement

- **Système hôte :** Ubuntu 24.04.3 (machine virtuelle)  
- **Contributeurs :** **Yassine ESSAOURI** et **Ahmed ENNOURI**
- **Dépôt de base :** [Anthony-Jhoiro/2025-devops-tp-final](https://github.com/Anthony-Jhoiro/2025-devops-tp-final.git)

### Configuration initiale

Avant de commencer, préparez votre environnement avec les commandes suivantes :

```bash
# Mettre à jour les paquets
sudo apt update

# Installer Docker et Git
sudo apt install -y docker.io git

# Activer et démarrer Docker
sudo systemctl enable --now docker

# (Optionnel) Ajouter votre utilisateur au groupe docker pour éviter d'utiliser sudo à chaque commande
sudo usermod -aG docker $USER
```
> **📌 Note :** Après avoir ajouté votre utilisateur au groupe `docker`, déconnectez-vous et reconnectez-vous pour que les changements prennent effet.

Vérifiez que tout fonctionne correctement :
```bash
docker --version
git --version
```

💡 **Astuce :** Vous pouvez tester Docker sans sudo avec :
```bash
docker run hello-world
```


## Partie 1 – Gestion du code avec Git

### 1.1 Introduction

Le projet utilise Git pour la gestion du code, le versioning et le workflow CI/CD.  

L’objectif est de permettre un travail collaboratif efficace, avec des branches claires pour la production, la pré-production et les nouvelles fonctionnalités.

### 1.2 Fork du dépôt

Le projet ne doit pas être modifié au niveau du code, mais nous devons mettre en place une infrastructure DevOps complète.  
Pour cela, nous avons effectué un **fork** du dépôt GitHub original.

Commandes utilisées :

```bash
git clone https://github.com/EssYassine/2025-devops-tp-final.git
cd 2025-devops-tp-final/
```

Ensuite, nous avons ajouté le dépôt original comme upstream afin de pouvoir synchroniser si nécessaire :

```bash
git remote add upstream https://github.com/Anthony-Jhoiro/2025-devops-tp-final.git
git fetch upstream
```

### 1.3 Remotes configurés

Après cette configuration, les remotes du projet sont :

- origin → fork personnel (dépôt principal de travail)

- upstream → dépôt original (lecture seule)

Pour vérifier :
```bash
git remote -v
```

### 1.4 Branches et workflow

Pour ce projet, nous avons mis en place une branche principale `main` et une branche de pré-production `develop`.  

> **📌 Note :** Les branches `feature/*` seront créées à partir de `develop` lorsque nous aurons besoin d’ajouter des configurations DevOps ou des tests supplémentaires.

#### Branches utilisées

Le projet utilise une stratégie simple mais adaptée à un pipeline CI/CD :

| Branche             | Description                                                                                  | Règles / Remarques                                                                 |
|--------------------|----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------|
| main             | Branche de production                                                                        | Le code déployé en production provient toujours de `main`. Protégée : merge uniquement via PR + pipeline vert. |
| develop          | Branche de pré-production / staging                                                          | Déploiement automatique vers l'environnement de test. Toute feature doit être fusionnée ici avant d’aller en production. |
| feature/<nom>    | Branches temporaires pour toute nouvelle fonctionnalité ou ajout de configuration DevOps     | Exemple : `feature/add-ci`, `feature/docker-compose`.                             |

#### Commandes utilisées

1. Création de la branche `develop` localement :
```bash
git checkout -b develop
```

2. Pousser develop sur le fork et configurer le suivi :
```bash
git push -u origin develop
```

3. Vérification des branches locales et distantes :
```bash
git branch -a
```

Exemple de sortie :
```bash
* develop
  main
  remotes/origin/HEAD -> origin/main
  remotes/origin/develop
  remotes/origin/main
  remotes/upstream/main
```

4. Exemple de création d’une future branche feature :
```bash
git checkout -b feature/<nom>
git push -u origin feature/<nom>
```

> **📌 Notes :** 
> - Pour l’instant, aucune branche feature n’a encore été créée.
> - Tout développement doit passer par develop avant de fusionner dans main.
> - Les merges vers main déclencheront la création d’un tag Git et d’une release GitHub.

#### Workflow recommandé

1. Créer une branche feature :
```bash
git checkout -b feature/<nom>
```

2. Travailler et push :
```bash
git push -u origin feature/<nom>
```

3. Ouvrir une Pull Request vers develop.

4. Fusionner develop vers main après validation pour déclencher le déploiement en production.

### 1.5 Releases GitHub

À chaque merge dans main :

- Création d’un tag Git (v1.0.0, v1.1.0, …)

- Création d’une release GitHub

- Mise à jour automatique des images Docker Hub



## Partie 2 – Dockerisation du projet

### 2.1 Introduction

L’objectif de la dockerisation est de permettre un déploiement reproductible, portable et compatible avec un pipeline CI/CD.

Le projet utilise trois conteneurs principaux :

| Service  | Technologie          | Description                         |
| -------- | -------------------- | ----------------------------------- |
| frontend | React + Vite         | Servi via Nginx                     |
| backend  | Go 1.22              | API REST + accès base de données    |
| database | PostgreSQL (à venir) | Conteneur pour stockage des données |

### 2.2 Dockerfile Backend (Go)

Fichier : `backend/Dockerfile`
```dockerfile
# -------------------------
# Build stage
# -------------------------
FROM golang:1.22-alpine AS builder

WORKDIR /app

# Copy go.mod and go.sum first (layer caching)
COPY go.mod go.sum ./
RUN go mod download

# Copy the rest of the backend code
COPY . .

# Build the Go binary
RUN go build -o server ./cmd/server

# -------------------------
# Runtime stage
# -------------------------
FROM alpine:3.19

WORKDIR /app

# Copy the compiled binary
COPY --from=builder /app/server .

# Copy migrations (if the binary needs them)
COPY migrations ./migrations

EXPOSE 8080

CMD ["./server"]
```

Justification: 

- Multi-stage build pour réduire la taille de l’image finale.

- Compilation avec `golang:alpine`.

- Exécution dans une image Alpine légère.

- Migration copiée pour utilisation par le backend.

### 2.3 Dockerfile Frontend (React + Vite)

Fichier : `frontend/Dockerfile`
```dockerfile
# -------------------------
# Build stage
# -------------------------
FROM node:20-alpine AS builder

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm install

COPY . .
RUN npm run build

# -------------------------
# Production stage
# -------------------------
FROM nginx:alpine

# Copy build output to nginx HTML dir
COPY --from=builder /app/dist /usr/share/nginx/html

# Expose nginx default port
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

Justification:

- Build séparé → dossier `dist/` généré par Vite.

- Nginx pour servir la SPA statique.

- Image optimisée pour production.

### 2.4 Tests locaux des Dockerfiles

Backend :
```bash
docker build -t christmas-backend ./backend
docker run -p 8080:8080 christmas-backend
```

> **⚠️ Note :** Le backend nécessite que la variable d’environnement `DATABASE_URL` soit définie pour pouvoir se connecter à la base de données.
> Sans cette variable, le conteneur plante avec une erreur `panic: environment variable DATABASE_URL is not set`.
> Pour les tests locaux, il est recommandé d’utiliser docker-compose afin de lancer également la base de données et de passer correctement la variable au backend.

Frontend :
```bash
docker build -t christmas-frontend ./frontend
docker run -p 3000:80 christmas-frontend
```

> ✅ Le frontend fonctionne correctement seul, il est servi par Nginx sur le port 3000.


### 2.5 Résumé de la dockerisation

- Le backend et le frontend sont isolés dans des conteneurs.

- Le frontend est servi par Nginx, le backend par un binaire Go.

- Les images sont prêtes pour CI/CD et publication sur Docker Hub.

- Prochaine étape : `docker-compose` pour orchestrer les services et la base de données.

## Partie 3 – Orchestration avec Docker Compose

### 3.1 Introduction

Pour tester l’ensemble des services localement et faciliter le déploiement, nous utilisons **Docker Compose**.  

Il permet de lancer **simultanément** le frontend, le backend et la base de données PostgreSQL, avec les variables d’environnement correctement configurées.

### 3.2 Création de Docker Compose

Fichier : `docker-compose.yml`
```yaml
version: "3.9"

services:
  db:
    image: postgres:16-alpine
    container_name: christmas-db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: christmas
    ports:
      - "5432:5432"
    volumes:
      - db_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: christmas-backend
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/christmas?sslmode=disable
    ports:
      - "8080:8080"
    depends_on:
      db:
        condition: service_healthy

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: christmas-frontend
    ports:
      - "3000:80"
    environment:
      VITE_API_URL: http://backend:8080
    depends_on:
      - backend

volumes:
  db_data:
```


### 3.3 Explications

- db (PostgreSQL)

    - Stockage persistant avec db_data

    - Healthcheck pour garantir que la base est prête avant de démarrer le backend

- backend (Go API)

    - Connecté à la DB via DATABASE_URL

    - Dépend de la DB (depends_on) pour attendre qu’elle soit opérationnelle

- frontend (React + Nginx)

    - Utilise la variable VITE_API_URL pour pointer vers le backend

    - Appels API internes dans React utilisent cette variable pour fonctionner dans Docker

- Ports exposés

    - Frontend : 3000

    - Backend : 8080

    - PostgreSQL : 5432

- Volumes

    - Permet de garder la DB persistante entre les redémarrages

### 3.4 Lancer le stack

Depuis la racine du projet :
```bash
docker-compose up --build
```

- Tous les services sont construits et lancés.

- Frontend → http://localhost:3000

- Backend → http://localhost:8080/api/people

Pour arrêter :
```bash
docker-compose down
```

- Ajouter -v pour supprimer également le volume de la DB.

### 3.5 Vérification

- Le frontend doit s’afficher correctement.

- Les appels API depuis le frontend doivent retourner des données JSON depuis le backend.

- La base PostgreSQL doit être opérationnelle et accessible par le backend.

### 3.6 Résumé

- Docker Compose permet de lancer l’ensemble du projet en une seule commande.

- Frontend, backend et DB sont isolés mais communiquent correctement.

- Cette configuration est prête pour être intégrée à un pipeline CI/CD.

## Partie 4 – Pipeline CI/CD avec GitHub Actions

### 4.1 Introduction

L’objectif de cette étape est d’automatiser :

- La construction des images Docker pour le frontend et le backend
- Les tests (unitaires ou e2e si disponibles)
- Le push des images sur Docker Hub
- Le déploiement vers un environnement pré-production ou production
- La création automatique d’une release GitHub lors d’un merge sur `main`

### 4.2 Pré-requis

1. **Compte Docker Hub** pour publier les images  
2. **Secrets GitHub** configurés dans le dépôt :  

| Secret                  | Description                                      |
|-------------------------|--------------------------------------------------|
| DOCKER_HUB_USERNAME      | Nom d’utilisateur Docker Hub                     |
| DOCKER_HUB_ACCESS_TOKEN  | Token d’accès Docker Hub                          |

### 4.3 Workflow GitHub Actions

Fichier : `.github/workflows/ci-cd.yml`
```yaml
name: CI/CD Pipeline

on:
  push:
    branches:
      - develop
      - main

env:
  DOCKERHUB_USERNAME: ${{ secrets.DOCKERHUB_USERNAME }}
  DOCKERHUB_PASSWORD: ${{ secrets.DOCKERHUB_PASSWORD }}
  IMAGE_FRONTEND: christmas-frontend
  IMAGE_BACKEND: christmas-backend

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      # 1. Checkout repo
      - name: Checkout repository
        uses: actions/checkout@v3

      # 2. Set up Docker Buildx
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # 3. Log in to Docker Hub
      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ env.DOCKERHUB_USERNAME }}
          password: ${{ env.DOCKERHUB_PASSWORD }}

      # 4. Build and push frontend
      - name: Build & push frontend
        uses: docker/build-push-action@v5
        with:
          context: ./frontend
          file: ./frontend/Dockerfile
          push: true
          tags: ${{ env.DOCKERHUB_USERNAME }}/${{ env.IMAGE_FRONTEND }}:latest

      # 5. Build and push backend
      - name: Build & push backend
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          file: ./backend/Dockerfile
          push: true
          tags: ${{ env.DOCKERHUB_USERNAME }}/${{ env.IMAGE_BACKEND }}:latest

  deploy-preprod:
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/develop'
    steps:
      - name: Deploy to preprod
        run: |
          echo "Déploiement préprod à mettre en place sur votre plateforme (Render, Railway, etc.)"
          # Ici tu ajoutes la commande pour déployer avec Docker ou via l'API du provider

  deploy-prod:
    runs-on: ubuntu-latest
    needs: build-and-push
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Tag & create GitHub release
        uses: ncipollo/release-action@v1
        with:
          tag: v${{ github.run_number }}
          name: Release ${{ github.run_number }}
          draft: false
          prerelease: false

      - name: Deploy to production
        run: |
          echo "Déploiement prod à mettre en place sur votre plateforme (Render, Railway, etc.)"
          # Ici tu ajoutes la commande pour déployer avec Docker ou via l'API du provider
```

### 4.4 Explications

1. Déclenchement du workflow

    - Sur develop → build + push Docker Hub (pré-prod)

    - Sur main → build + push Docker Hub + release GitHub (prod)

2. Docker Build et Push

    - Images backend et frontend sont construites et taguées latest

    - Elles sont ensuite poussées sur Docker Hub pour un déploiement ultérieur

3. Création de Release

    - Lors d’un merge sur main, une release GitHub est créée automatiquement avec un tag basé sur le numéro du run.

4. Secrets GitHub

    - DOCKER_HUB_USERNAME et DOCKER_HUB_ACCESS_TOKEN permettent la connexion sécurisée à Docker Hub.

### 4.5 Résultat attendu

- À chaque push sur develop → images Docker mises à jour sur Docker Hub

- À chaque merge sur main → images Docker mises à jour + release GitHub créée

- Pré-requis pour un déploiement sur un provider cloud (Render, Railway, etc.)

## Plan complet – Étapes restantes pour finaliser le TP DevOps

### 1. Gestion des environnements (préprod / prod)

- Décider :

    - develop → préproduction

    - main → production

- Configurer les variables d’environnement par environnement

### 2. Création automatique des releases GitHub

- Ajouter la création de tag automatique dans la CI

- Générer une release GitHub sur merge main

- Vérifier :

    - tag visible (v1.0.0 ou vX)

    - release créée automatiquement

### 3. Déploiement de l’application (cloud)

- Choisir un provider (Render / Railway / autre)

- Déployer :

    - frontend

    - backend

    - base de données

- Utiliser les images Docker Hub

- Configurer les variables d’environnement

- Obtenir une URL publique

- Vérifier que l’app fonctionne à distance

### 4. Documentation finale (`docs/documentation.md`)

- Architecture globale

- Git workflow

- Docker & docker-compose

- CI/CD

- Déploiement cloud

- Variables d’environnement

- Choix techniques (pourquoi ces outils)

### 5. Õptionnel - Bonus

- Monitoring (logs, healthcheck)

- Kubernetes

- Déploiement on-premise

- Tests e2e intégrés à la CI

- Reverse proxy Nginx

## Limitations et contraintes rencontrées

### 1. Contraintes de temps

Le projet a été réalisé dans un temps limité, en parallèle d’autres modules académiques et d’obligations personnelles.  

Cette contrainte de temps a nécessité des choix pragmatiques dans les outils et les architectures retenues, en privilégiant des solutions simples, stables et déjà maîtrisées, plutôt que des approches plus avancées ou expérimentales.

### 2. Courbe de compréhension

Bien que les concepts DevOps abordés dans ce projet aient été vus en cours, leur mise en œuvre complète dans un contexte réel (CI/CD, Docker multi-services, gestion des environnements, automatisation des releases) a nécessité un temps d’appropriation important.

Certaines parties ont demandé des phases de recherche, de tests et de corrections successives avant d’obtenir un pipeline fonctionnel et reproductible.

### 3. Contraintes de santé

Des contraintes de santé ponctuelles ont également eu un impact sur la disponibilité et la capacité de travail durant certaines périodes du projet.  
Ces contraintes ont limité le temps continu pouvant être consacré au développement et à la configuration, imposant une organisation plus segmentée du travail.

### 4. Perspectives et finalités

Avec plus de temps et de disponibilité, les étapes discutées au préalable pourraient être appliquées.
