# TP 06 - Conteneurisation avec Docker

## Contexte GameShelf

Notre infrastructure avec VMs et Load Balancer fonctionne bien, mais le déploiement reste complexe. Chaque serveur doit avoir exactement les mêmes dépendances installées. Docker va nous permettre de packager notre application avec toutes ses dépendances.

**Objectif de ce TP :** Conteneuriser l'API GameShelf et la publier sur un registry.

---

## Prérequis

- Docker installé (voir TP précédents ou Partie 2)
- Un compte Docker Hub (gratuit)
- Terminal Linux/macOS ou WSL2

---

## Partie 1 : Conteneurs vs Machines Virtuelles

### Comparaison

```
┌─────────────────────────────────────────────────────────┐
│                    MACHINE VIRTUELLE                     │
├─────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                  │
│  │  App A  │  │  App B  │  │  App C  │                  │
│  ├─────────┤  ├─────────┤  ├─────────┤                  │
│  │  Libs   │  │  Libs   │  │  Libs   │                  │
│  ├─────────┤  ├─────────┤  ├─────────┤                  │
│  │Guest OS │  │Guest OS │  │Guest OS │  ← OS complet    │
│  └─────────┘  └─────────┘  └─────────┘                  │
│  ┌─────────────────────────────────────┐                │
│  │            Hyperviseur              │                │
│  └─────────────────────────────────────┘                │
│  ┌─────────────────────────────────────┐                │
│  │              Host OS                │                │
│  └─────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                      CONTENEURS                          │
├─────────────────────────────────────────────────────────┤
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                  │
│  │  App A  │  │  App B  │  │  App C  │                  │
│  ├─────────┤  ├─────────┤  ├─────────┤                  │
│  │  Libs   │  │  Libs   │  │  Libs   │  ← Léger !      │
│  └─────────┘  └─────────┘  └─────────┘                  │
│  ┌─────────────────────────────────────┐                │
│  │          Docker Engine              │                │
│  └─────────────────────────────────────┘                │
│  ┌─────────────────────────────────────┐                │
│  │              Host OS                │  ← Partagé     │
│  └─────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────┘
```

### Avantages des conteneurs

| Aspect | VM | Conteneur |
|--------|-----|-----------|
| Démarrage | Minutes | Secondes |
| Taille | Go | Mo |
| Isolation | Forte (hyperviseur) | Processus (kernel partagé) |
| Portabilité | Moyenne | Excellente |
| Performance | Overhead | Proche du natif |

---

## Partie 2 : Installation de Docker

### Option A : Linux/WSL2

```bash
# Installer Docker
curl -fsSL https://get.docker.com | sh

# Ajouter l'utilisateur au groupe docker
sudo usermod -aG docker $USER

# Appliquer les changements
newgrp docker

# Vérifier
docker --version
docker run hello-world
```

### Option B : Docker Desktop (Windows/macOS)

1. Téléchargez Docker Desktop : [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/)
2. Installez et lancez
3. Vérifiez : `docker --version`

---

## Partie 3 : Création de l'API GameShelf

### Étape 3.1 : Créer le projet

```bash
mkdir -p ~/gameshelf-api
cd ~/gameshelf-api
```

### Étape 3.2 : Créer l'API Python Flask

```bash
nano app.py
```

```python
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

# Base de données simulée
GAMES = [
    {"id": 1, "name": "Catan", "category": "Stratégie", "players": "3-4", "price": 39.99},
    {"id": 2, "name": "Pandemic", "category": "Coopératif", "players": "2-4", "price": 44.99},
    {"id": 3, "name": "Codenames", "category": "Party Game", "players": "4-8", "price": 24.99},
    {"id": 4, "name": "Ticket to Ride", "category": "Famille", "players": "2-5", "price": 42.99},
    {"id": 5, "name": "7 Wonders", "category": "Stratégie", "players": "3-7", "price": 49.99},
]

@app.route('/')
def home():
    return jsonify({
        "service": "GameShelf API",
        "version": "1.0.0",
        "hostname": socket.gethostname(),
        "endpoints": ["/", "/health", "/games", "/games/<id>", "/flag"]
    })

@app.route('/health')
def health():
    return jsonify({
        "status": "healthy",
        "hostname": socket.gethostname()
    })

@app.route('/games')
def get_games():
    return jsonify({
        "count": len(GAMES),
        "games": GAMES
    })

@app.route('/games/<int:game_id>')
def get_game(game_id):
    game = next((g for g in GAMES if g["id"] == game_id), None)
    if game:
        return jsonify(game)
    return jsonify({"error": "Game not found"}), 404

@app.route('/flag')
def get_flag():
    return jsonify({
        "message": "Bravo ! Vous avez conteneurisé l'API GameShelf !",
        "flag": "FLAG{D0ck3r_C0nt41n3r1z3d_AP1_Succ3ss}",
        "hostname": socket.gethostname()
    })

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)
```

### Étape 3.3 : Créer le fichier requirements.txt

```bash
nano requirements.txt
```

```
flask==3.0.0
gunicorn==21.2.0
```

### Étape 3.4 : Tester localement (optionnel)

```bash
pip3 install -r requirements.txt
python3 app.py
```

Testez : `curl http://localhost:5000/games`

Arrêtez avec Ctrl+C.

---

## Partie 4 : Création du Dockerfile

### Étape 4.1 : Comprendre le Dockerfile

Un Dockerfile est une recette pour construire une image Docker.

**Instructions principales :**

| Instruction | Description |
|-------------|-------------|
| `FROM` | Image de base |
| `WORKDIR` | Répertoire de travail |
| `COPY` | Copier des fichiers |
| `RUN` | Exécuter une commande lors du build |
| `ENV` | Variable d'environnement |
| `EXPOSE` | Port exposé (documentation) |
| `CMD` | Commande par défaut au démarrage |

### Étape 4.2 : Créer le Dockerfile

```bash
nano Dockerfile
```

```dockerfile
# Image de base Python
FROM python:3.11-slim

# Métadonnées
LABEL maintainer="votre.email@universite.fr"
LABEL version="1.0.0"
LABEL description="API GameShelf - Cours Cloud Computing M1"

# Répertoire de travail
WORKDIR /app

# Copier les dépendances d'abord (cache Docker)
COPY requirements.txt .

# Installer les dépendances
RUN pip install --no-cache-dir -r requirements.txt

# Copier le code source
COPY app.py .

# Variable d'environnement pour le port
ENV PORT=5000

# Exposer le port
EXPOSE 5000

# Créer un utilisateur non-root pour la sécurité
RUN useradd -m appuser
USER appuser

# Commande de démarrage avec Gunicorn (production)
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

### Étape 4.3 : Créer le fichier .dockerignore

```bash
nano .dockerignore
```

```
__pycache__
*.pyc
*.pyo
.git
.gitignore
.env
*.md
Dockerfile
docker-compose.yml
.dockerignore
venv/
.venv/
```

---

## Partie 5 : Construction de l'image

### Étape 5.1 : Construire l'image

```bash
docker build -t gameshelf-api:1.0.0 .
```

**Explication :**
- `-t gameshelf-api:1.0.0` : Tag de l'image (nom:version)
- `.` : Contexte de build (répertoire courant)

### Étape 5.2 : Voir les images

```bash
docker images
```

Vous verrez votre image `gameshelf-api` (environ 150-200 MB).

### Étape 5.3 : Analyser les couches

```bash
docker history gameshelf-api:1.0.0
```

Chaque instruction du Dockerfile crée une couche.

---

## Partie 6 : Exécution du conteneur

### Étape 6.1 : Lancer le conteneur

```bash
docker run -d -p 8080:5000 --name gameshelf gameshelf-api:1.0.0
```

**Explication :**
- `-d` : Mode détaché (arrière-plan)
- `-p 8080:5000` : Mapper le port 8080 (hôte) vers 5000 (conteneur)
- `--name gameshelf` : Nom du conteneur

### Étape 6.2 : Vérifier le conteneur

```bash
# Liste des conteneurs
docker ps

# Logs
docker logs gameshelf

# Logs en temps réel
docker logs -f gameshelf
```

### Étape 6.3 : Tester l'API

```bash
# Page d'accueil
curl http://localhost:8080/

# Health check
curl http://localhost:8080/health

# Liste des jeux
curl http://localhost:8080/games

# Un jeu spécifique
curl http://localhost:8080/games/1

# Le flag !
curl http://localhost:8080/flag
```

### Étape 6.4 : Exécuter une commande dans le conteneur

```bash
# Shell interactif
docker exec -it gameshelf /bin/bash

# Voir les processus
docker exec gameshelf ps aux

# Voir les variables d'environnement
docker exec gameshelf env
```

---

## Partie 7 : Docker Compose

Docker Compose permet de définir des applications multi-conteneurs.

### Étape 7.1 : Créer docker-compose.yml

```bash
nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  api:
    build: .
    container_name: gameshelf-api
    ports:
      - "8080:5000"
    environment:
      - PORT=5000
      - FLASK_ENV=production
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s
    labels:
      - "project=gameshelf"
      - "environment=development"

  # Nginx comme reverse proxy (bonus)
  nginx:
    image: nginx:alpine
    container_name: gameshelf-proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - api
    restart: unless-stopped

networks:
  default:
    name: gameshelf-network
```

### Étape 7.2 : Créer la config Nginx

```bash
nano nginx.conf
```

```nginx
upstream api {
    server api:5000;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    location /health {
        proxy_pass http://api/health;
        access_log off;
    }
}
```

### Étape 7.3 : Lancer avec Docker Compose

```bash
# Arrêter le conteneur précédent
docker stop gameshelf
docker rm gameshelf

# Lancer avec Compose
docker compose up -d

# Voir les logs
docker compose logs -f

# État des services
docker compose ps
```

### Étape 7.4 : Tester

```bash
# Via Nginx (port 80)
curl http://localhost/games

# Directement sur l'API (port 8080)
curl http://localhost:8080/games
```

---

## Partie 8 : Publication sur Docker Hub

### Étape 8.1 : Créer un compte Docker Hub

1. Allez sur [hub.docker.com](https://hub.docker.com/)
2. Créez un compte gratuit
3. Notez votre **username**

### Étape 8.2 : Se connecter

```bash
docker login
```

Entrez votre username et mot de passe Docker Hub.

### Étape 8.3 : Tagger l'image

```bash
# Format: docker.io/USERNAME/IMAGE:TAG
docker tag gameshelf-api:1.0.0 VOTRE_USERNAME/gameshelf-api:1.0.0
docker tag gameshelf-api:1.0.0 VOTRE_USERNAME/gameshelf-api:latest
```

### Étape 8.4 : Pousser l'image

```bash
docker push VOTRE_USERNAME/gameshelf-api:1.0.0
docker push VOTRE_USERNAME/gameshelf-api:latest
```

### Étape 8.5 : Vérifier sur Docker Hub

1. Allez sur [hub.docker.com](https://hub.docker.com/)
2. Votre image apparaît dans vos repositories

### Étape 8.6 : Tester le pull

```bash
# Supprimer l'image locale
docker rmi VOTRE_USERNAME/gameshelf-api:latest

# Pull depuis Docker Hub
docker pull VOTRE_USERNAME/gameshelf-api:latest

# Exécuter
docker run -d -p 9090:5000 VOTRE_USERNAME/gameshelf-api:latest

# Tester
curl http://localhost:9090/flag
```

---

## Partie 9 : Bonnes pratiques Docker

### Optimisation du Dockerfile

```dockerfile
# Multi-stage build (exemple avancé)
FROM python:3.11-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip wheel --no-cache-dir --no-deps --wheel-dir /wheels -r requirements.txt

FROM python:3.11-slim

# Copier les wheels pré-construits
COPY --from=builder /wheels /wheels
RUN pip install --no-cache /wheels/*

WORKDIR /app
COPY app.py .

RUN useradd -m appuser
USER appuser

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]
```

### Liste des bonnes pratiques

1. **Utiliser des images slim/alpine** - Plus légères
2. **Un processus par conteneur** - Microservices
3. **Utilisateur non-root** - Sécurité
4. **Cache des layers** - Dépendances avant code
5. **.dockerignore** - Exclure les fichiers inutiles
6. **Tags explicites** - Éviter `latest` en production
7. **Multi-stage builds** - Images plus petites
8. **Health checks** - Monitoring

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé l'application Flask GameShelf
- [ ] Écrit un Dockerfile fonctionnel
- [ ] Construit l'image Docker
- [ ] Lancé le conteneur et testé l'API
- [ ] Créé un docker-compose.yml
- [ ] Publié l'image sur Docker Hub
- [ ] Récupéré le flag via `/flag`

---

## Flag de validation

Accédez à l'endpoint `/flag` de votre API :

```bash
curl http://localhost:8080/flag
```

```
FLAG{D0ck3r_C0nt41n3r1z3d_AP1_Succ3ss}
```

---

## Transition vers le TP suivant

*"Parfait ! Notre API est maintenant conteneurisée et publiée. Mais comment gérer plusieurs conteneurs en production ? Comment les mettre à l'échelle automatiquement ? Dans le prochain TP, nous allons découvrir Kubernetes !"*

---

## Commandes Docker essentielles

| Commande | Description |
|----------|-------------|
| `docker build -t name .` | Construire une image |
| `docker run -d -p H:C name` | Lancer un conteneur |
| `docker ps` | Lister les conteneurs |
| `docker logs name` | Voir les logs |
| `docker exec -it name bash` | Shell dans le conteneur |
| `docker stop name` | Arrêter |
| `docker rm name` | Supprimer |
| `docker images` | Lister les images |
| `docker rmi name` | Supprimer une image |
| `docker compose up -d` | Lancer avec Compose |
| `docker compose down` | Arrêter avec Compose |

---

## Structure du projet

```
gameshelf-api/
├── app.py              # Application Flask
├── requirements.txt    # Dépendances Python
├── Dockerfile          # Recette Docker
├── .dockerignore       # Fichiers à ignorer
├── docker-compose.yml  # Orchestration locale
└── nginx.conf          # Config reverse proxy
```

---

## En cas de problème

| Problème | Solution |
|----------|----------|
| "Permission denied" | Ajoutez-vous au groupe docker |
| Port déjà utilisé | Changez le port mappé (-p 9090:5000) |
| Build échoue | Vérifiez la syntaxe du Dockerfile |
| Push refused | Vérifiez docker login et le tag |

---

**Durée estimée : 1 heure**

**Difficulté : Intermédiaire**
