# Configuration Docker pour Serverly

::callout{type="info" title="Prérequis"}
Assurez-vous d'avoir Docker installé sur votre système avant de commencer.
::

Cette documentation vous guide dans la configuration de Docker pour votre projet Serverly.

## Installation de Docker

::code-group

```bash [Linux (Ubuntu/Debian)]
# Mise à jour des paquets
sudo apt update

# Installation des dépendances
sudo apt install apt-transport-https ca-certificates curl gnupg lsb-release

# Ajout de la clé GPG officielle de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Installation de Docker
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io
```

```bash [macOS]
# Installation via Homebrew
brew install --cask docker

# Ou téléchargez Docker Desktop depuis le site officiel
open https://www.docker.com/products/docker-desktop
```

```powershell [Windows]
# Installation via Chocolatey
choco install docker-desktop

# Ou téléchargez Docker Desktop depuis le site officiel
start https://www.docker.com/products/docker-desktop
```

::

## Configuration initiale

::tip{title="Conseil de sécurité"}
Pour Linux, ajoutez votre utilisateur au groupe docker pour éviter d'utiliser sudo :

```bash
sudo usermod -aG docker $USER
```

Redémarrez ensuite votre session.
::

### Vérification de l'installation

```bash
# Vérifier la version de Docker
docker --version

# Tester l'installation
docker run hello-world
```

## Dockerfile pour Serverly

Créez un `Dockerfile` à la racine de votre projet :

```dockerfile
# Utiliser l'image Node.js officielle
FROM node:18-alpine

# Définir le répertoire de travail
WORKDIR /app

# Copier les fichiers de dépendances
COPY package*.json ./

# Installer les dépendances
RUN npm ci --only=production

# Copier le code source
COPY . .

# Construire l'application
RUN npm run build

# Exposer le port
EXPOSE 3000

# Commande de démarrage
CMD ["npm", "run", "start"]
```

::note
Ce Dockerfile est optimisé pour la production. Pour le développement, vous pourriez vouloir installer toutes les dépendances.
::

## Docker Compose

Créez un fichier `docker-compose.yml` pour orchestrer vos services :

```yaml
version: "3.8"

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
    depends_on:
      - db
      - redis
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=serverly
      - POSTGRES_USER=serverly
      - POSTGRES_PASSWORD=${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

## Variables d'environnement

Créez un fichier `.env` pour vos variables sensibles :

```bash
# Base de données
DATABASE_URL=postgresql://serverly:your_password@db:5432/serverly
DB_PASSWORD=your_secure_password

# Redis
REDIS_URL=redis://redis:6379

# Application
NODE_ENV=production
JWT_SECRET=your_jwt_secret
API_KEY=your_api_key
```

::warning
Ne commitez jamais votre fichier `.env` ! Ajoutez-le à votre `.gitignore`.
::

## Configuration Nginx

Créez un fichier `nginx.conf` pour le reverse proxy :

```nginx
events {
    worker_connections 1024;
}

http {
    upstream app {
        server app:3000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://app;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_cache_bypass $http_upgrade;
        }
    }
}
```

## Commandes utiles

::feature-cards{:features="[
{
title: 'Construction',
description: 'Construire l\'image Docker',
icon: 'heroicons:cog-6-tooth',
command: 'docker build -t serverly .'
},
{
title: 'Démarrage',
description: 'Lancer tous les services',
icon: 'heroicons:play',
command: 'docker-compose up -d'
},
{
title: 'Logs',
description: 'Consulter les logs en temps réel',
icon: 'heroicons:document-text',
command: 'docker-compose logs -f'
},
{
title: 'Arrêt',
description: 'Arrêter tous les services',
icon: 'heroicons:stop',
command: 'docker-compose down'
}
]"}
::

### Commandes de base

```bash
# Construire et démarrer
docker-compose up --build

# Démarrer en arrière-plan
docker-compose up -d

# Voir les logs
docker-compose logs -f app

# Accéder au conteneur
docker-compose exec app sh

# Redémarrer un service
docker-compose restart app

# Nettoyer
docker-compose down -v
docker system prune -a
```

## Optimisations

::badge-list{:badges="['Performance', 'Sécurité', 'Monitoring']" variant="primary"}
::

### Multi-stage build

Pour optimiser la taille de l'image :

```dockerfile
# Stage de build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage de production
FROM node:18-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force
COPY --from=builder /app/dist ./dist
EXPOSE 3000
CMD ["npm", "start"]
```

### Monitoring avec Watchtower

Ajoutez ce service à votre `docker-compose.yml` :

```yaml
watchtower:
  image: containrrr/watchtower
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
  command: --interval 30
  restart: unless-stopped
```

## Déploiement

::tip{title="Déploiement automatisé"}
Utilisez GitHub Actions pour automatiser le déploiement de vos conteneurs Docker.
::

### Avec Docker Swarm

```bash
# Initialiser le swarm
docker swarm init

# Déployer la stack
docker stack deploy -c docker-compose.yml serverly

# Voir les services
docker service ls
```

### Avec Kubernetes

Créez un fichier `k8s-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: serverly-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: serverly
  template:
    metadata:
      labels:
        app: serverly
    spec:
      containers:
        - name: serverly
          image: serverly:latest
          ports:
            - containerPort: 3000
          env:
            - name: NODE_ENV
              value: "production"
---
apiVersion: v1
kind: Service
metadata:
  name: serverly-service
spec:
  selector:
    app: serverly
  ports:
    - port: 80
      targetPort: 3000
  type: LoadBalancer
```

## Dépannage

::callout{type="warning" title="Problèmes courants"}
Voici les erreurs les plus fréquentes et leurs solutions.
::

### Port déjà utilisé

```bash
# Trouver le processus utilisant le port
lsof -i :3000

# Arrêter le processus
kill -9 <PID>
```

### Problèmes de permissions

```bash
# Réinitialiser les permissions
sudo chown -R $USER:$USER .
```

### Nettoyage des ressources

```bash
# Supprimer tous les conteneurs arrêtés
docker container prune

# Supprimer toutes les images non utilisées
docker image prune -a

# Supprimer tous les volumes non utilisés
docker volume prune
```

::note
Cette configuration Docker est un exemple de base. Adaptez-la selon vos besoins spécifiques et votre environnement de production.
::

## Liens utiles

- [Documentation officielle Docker](https://docs.docker.com/)
- [Guide des meilleures pratiques](https://docs.docker.com/develop/dev-best-practices/)
- [Docker Compose reference](https://docs.docker.com/compose/compose-file/)
