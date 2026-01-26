# TP 05 (Alternatif) - Haute Disponibilité avec Docker

## Version alternative - Sans compte OVH

> **Note :** Cette version utilise Docker avec Nginx comme Load Balancer et plusieurs conteneurs web.

---

## Contexte GameShelf

Le site GameShelf gagne en popularité ! Nous devons mettre en place une architecture haute disponibilité avec Docker.

**Objectif de ce TP :** Déployer plusieurs conteneurs web derrière un load balancer Nginx.

---

## Prérequis

- Docker installé et fonctionnel
- Terminal Linux/macOS ou WSL2

---

## Partie 1 : Architecture cible

```
                    ┌─────────────────┐
                    │  Nginx LB       │
                    │  (Port 8080)    │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
              ▼              ▼              ▼
        ┌──────────┐  ┌──────────┐  ┌──────────┐
        │  Web 01  │  │  Web 02  │  │  Web 03  │
        │  Nginx   │  │  Nginx   │  │  Nginx   │
        └──────────┘  └──────────┘  └──────────┘
```

---

## Partie 2 : Préparation du projet

### Étape 2.1 : Créer le dossier du projet

```bash
mkdir -p ~/gameshelf-ha-docker
cd ~/gameshelf-ha-docker
```

### Étape 2.2 : Structure des dossiers

```bash
mkdir -p {web/html,loadbalancer}
```

---

## Partie 3 : Configuration des serveurs web

### Étape 3.1 : Page HTML pour les serveurs web

```bash
nano web/html/index.html
```

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GameShelf - Serveur Web</title>
    <style>
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #1a1a2e 0%, #16213e 100%);
            color: #eee;
            margin: 0;
            padding: 0;
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
        }
        .container {
            text-align: center;
            padding: 40px;
            background: rgba(255,255,255,0.1);
            border-radius: 20px;
            backdrop-filter: blur(10px);
            max-width: 600px;
        }
        h1 {
            font-size: 3em;
            margin-bottom: 10px;
            color: #f39c12;
        }
        .server-badge {
            display: inline-block;
            padding: 10px 25px;
            background: #e74c3c;
            border-radius: 25px;
            font-size: 1.2em;
            margin: 10px 0;
            font-weight: bold;
        }
        .badge {
            display: inline-block;
            padding: 5px 15px;
            background: #3498db;
            border-radius: 20px;
            font-size: 0.9em;
            margin: 5px;
        }
        .status {
            margin-top: 20px;
            padding: 15px 30px;
            background: #27ae60;
            border-radius: 10px;
            display: inline-block;
        }
        .info {
            margin-top: 20px;
            padding: 15px;
            background: rgba(0,0,0,0.3);
            border-radius: 10px;
            font-family: monospace;
        }
        #hostname {
            color: #f39c12;
            font-size: 1.5em;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>GameShelf</h1>

        <div class="server-badge">
            Serveur: <span id="hostname">Chargement...</span>
        </div>

        <p>Architecture Haute Disponibilité</p>

        <div>
            <span class="badge">Load Balanced</span>
            <span class="badge">Docker</span>
            <span class="badge">Nginx</span>
        </div>

        <div class="status">
            Conteneur opérationnel
        </div>

        <div class="info">
            Rafraîchissez la page pour voir le load balancing en action !
        </div>

        <!-- FLAG{Dkr_H4_W3b_S3rv3r_0nL1n3} -->
    </div>

    <script>
        // Récupère le hostname via le header X-Served-By ou une API
        fetch('/hostname')
            .then(r => r.text())
            .then(h => document.getElementById('hostname').textContent = h.trim())
            .catch(() => document.getElementById('hostname').textContent = 'Web Server');
    </script>
</body>
</html>
```

### Étape 3.2 : Configuration Nginx pour les serveurs web

```bash
nano web/nginx.conf
```

```nginx
server {
    listen 80;
    server_name _;

    root /usr/share/nginx/html;
    index index.html;

    # Endpoint pour afficher le hostname
    location /hostname {
        return 200 $hostname;
        add_header Content-Type text/plain;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "OK - $hostname\n";
        add_header Content-Type text/plain;
    }

    location / {
        try_files $uri $uri/ =404;
    }

    # Header pour identifier le serveur
    add_header X-Served-By $hostname always;
}
```

### Étape 3.3 : Dockerfile pour les serveurs web

```bash
nano web/Dockerfile
```

```dockerfile
FROM nginx:alpine

# Copier la configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copier le contenu web
COPY html/ /usr/share/nginx/html/

# Exposer le port
EXPOSE 80

# Healthcheck
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost/health || exit 1
```

---

## Partie 4 : Configuration du Load Balancer

### Étape 4.1 : Configuration Nginx LB

```bash
nano loadbalancer/nginx.conf
```

```nginx
upstream gameshelf_backend {
    # Round Robin par défaut
    server web1:80;
    server web2:80;
    server web3:80;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://gameshelf_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout settings
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
    }

    # Page de status du Load Balancer
    location /lb-status {
        return 200 "Load Balancer OK\nBackends: web1, web2, web3\n";
        add_header Content-Type text/plain;
    }
}
```

### Étape 4.2 : Dockerfile pour le Load Balancer

```bash
nano loadbalancer/Dockerfile
```

```dockerfile
FROM nginx:alpine

# Copier la configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Exposer le port
EXPOSE 80

# Healthcheck
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=3 \
    CMD wget --quiet --tries=1 --spider http://localhost/lb-status || exit 1
```

---

## Partie 5 : Docker Compose

### Étape 5.1 : Créer le fichier docker-compose.yml

```bash
nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  # Load Balancer
  loadbalancer:
    build: ./loadbalancer
    container_name: gameshelf-lb
    ports:
      - "8080:80"
    depends_on:
      - web1
      - web2
      - web3
    networks:
      - gameshelf-network
    restart: unless-stopped
    labels:
      - "project=gameshelf"
      - "role=loadbalancer"

  # Serveur Web 1
  web1:
    build: ./web
    container_name: gameshelf-web-01
    networks:
      - gameshelf-network
    restart: unless-stopped
    labels:
      - "project=gameshelf"
      - "role=webserver"

  # Serveur Web 2
  web2:
    build: ./web
    container_name: gameshelf-web-02
    networks:
      - gameshelf-network
    restart: unless-stopped
    labels:
      - "project=gameshelf"
      - "role=webserver"

  # Serveur Web 3
  web3:
    build: ./web
    container_name: gameshelf-web-03
    networks:
      - gameshelf-network
    restart: unless-stopped
    labels:
      - "project=gameshelf"
      - "role=webserver"

networks:
  gameshelf-network:
    driver: bridge
    name: gameshelf-ha-network
```

---

## Partie 6 : Déploiement

### Étape 6.1 : Construire les images

```bash
docker compose build
```

### Étape 6.2 : Lancer l'infrastructure

```bash
docker compose up -d
```

### Étape 6.3 : Vérifier l'état des conteneurs

```bash
docker compose ps
```

Vous devriez voir 4 conteneurs en état "Up" :
- gameshelf-lb
- gameshelf-web-01
- gameshelf-web-02
- gameshelf-web-03

### Étape 6.4 : Voir les logs

```bash
docker compose logs -f
```

(Ctrl+C pour quitter)

---

## Partie 7 : Tests de haute disponibilité

### Étape 7.1 : Accéder au site

Ouvrez votre navigateur : **http://localhost:8080**

### Étape 7.2 : Observer le Round Robin

Rafraîchissez plusieurs fois la page. Le nom du serveur change :
- gameshelf-web-01
- gameshelf-web-02
- gameshelf-web-03

### Étape 7.3 : Vérifier avec curl

```bash
for i in {1..10}; do
  echo "Requête $i:"
  curl -s http://localhost:8080/hostname
  echo ""
done
```

### Étape 7.4 : Test de failover

1. Arrêtez un serveur web :

```bash
docker stop gameshelf-web-01
```

2. Testez l'accès :

```bash
curl http://localhost:8080
```

Le site reste accessible ! Les requêtes vont sur web-02 et web-03.

3. Vérifiez la répartition :

```bash
for i in {1..6}; do
  curl -s http://localhost:8080/hostname
done
```

4. Redémarrez le serveur :

```bash
docker start gameshelf-web-01
```

5. Après quelques secondes, web-01 reprend du trafic.

### Étape 7.5 : Tester le health check

```bash
# Health check du LB
curl http://localhost:8080/lb-status

# Health check des backends (accès direct)
docker exec gameshelf-web-01 wget -qO- http://localhost/health
docker exec gameshelf-web-02 wget -qO- http://localhost/health
docker exec gameshelf-web-03 wget -qO- http://localhost/health
```

---

## Partie 8 : Scaling dynamique

### Étape 8.1 : Ajouter des instances avec docker compose

```bash
# Scaler à 5 instances web
docker compose up -d --scale web1=1 --scale web2=1 --scale web3=1
```

Note : Avec la configuration actuelle, il faut modifier le nginx.conf du LB pour ajouter des backends.

### Étape 8.2 : Version avec scaling automatique

Modifiez `loadbalancer/nginx.conf` pour utiliser la résolution DNS Docker :

```nginx
upstream gameshelf_backend {
    # Résolution DNS dynamique
    server web1:80;
    server web2:80;
    server web3:80;
}
```

---

## Partie 9 : Monitoring basique

### Étape 9.1 : Script de monitoring

```bash
nano monitor.sh
```

```bash
#!/bin/bash

echo "=== État des conteneurs ==="
docker compose ps

echo ""
echo "=== Test des backends ==="
for container in gameshelf-web-01 gameshelf-web-02 gameshelf-web-03; do
    status=$(docker exec $container wget -qO- http://localhost/health 2>/dev/null || echo "DOWN")
    echo "$container: $status"
done

echo ""
echo "=== Test du Load Balancer ==="
curl -s http://localhost:8080/lb-status

echo ""
echo "=== Répartition (10 requêtes) ==="
for i in {1..10}; do
    curl -s http://localhost:8080/hostname
done | sort | uniq -c
```

```bash
chmod +x monitor.sh
./monitor.sh
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé l'infrastructure Docker Compose (LB + 3 web)
- [ ] Vérifié le Round Robin (serveurs alternent)
- [ ] Testé le failover (arrêt d'un conteneur)
- [ ] Observé la reprise automatique
- [ ] Vérifié les health checks

---

## Flag de validation

**Flag dans la page HTML :**
```
FLAG{Dkr_H4_W3b_S3rv3r_0nL1n3}
```

**Flag du Load Balancer** (accédez à http://localhost:8080/lb-status) :
Le LB est fonctionnel si vous voyez "Load Balancer OK".

**Flag global pour 3 serveurs fonctionnels :**
```
FLAG{H4_D0ck3r_3xB4ck3nds_R0und_R0b1n}
```

---

## Transition vers le TP suivant

*"Notre site est maintenant résilient ! Mais les sessions utilisateurs ne sont pas partagées entre les serveurs... Il nous faut une base de données. Prochain TP : base de données managée !"*

---

## Nettoyage

```bash
docker compose down
docker compose down -v  # Supprime aussi les volumes
```

---

## Structure du projet

```
gameshelf-ha-docker/
├── docker-compose.yml
├── loadbalancer/
│   ├── Dockerfile
│   └── nginx.conf
├── web/
│   ├── Dockerfile
│   ├── nginx.conf
│   └── html/
│       └── index.html
└── monitor.sh
```

---

## Commandes utiles

| Commande | Description |
|----------|-------------|
| `docker compose up -d` | Démarrer l'infrastructure |
| `docker compose ps` | État des conteneurs |
| `docker compose logs -f` | Voir les logs |
| `docker compose down` | Arrêter l'infrastructure |
| `docker stop <container>` | Simuler une panne |

---

## En cas de problème

| Problème | Solution |
|----------|----------|
| Port 8080 occupé | Changez le port dans docker-compose.yml |
| Build échoue | Vérifiez les Dockerfiles et chemins |
| LB ne répond pas | Vérifiez que les backends sont up |
| Pas de Round Robin | Videz le cache du navigateur |

---

**Durée estimée : 1 heure**

**Difficulté : Intermédiaire**
