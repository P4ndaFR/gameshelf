# TP 09 - PaaS : Déploiement sur Clever Cloud

## Contexte GameShelf

Nous avons vu comment déployer sur des VMs (IaaS) et sur Kubernetes. C'est puissant mais complexe. Et si on pouvait déployer une application en quelques minutes sans gérer l'infrastructure ?

**Objectif de ce TP :** Découvrir le PaaS en déployant l'API GameShelf sur Clever Cloud.

---

## Prérequis

- Navigateur web
- Git installé
- Code de l'API GameShelf (TP 06)

---

## Partie 1 : IaaS vs PaaS vs SaaS

### Comparaison des modèles

```
┌─────────────────────────────────────────────────────────────┐
│              RESPONSABILITÉS PAR MODÈLE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ON-PREMISE     IaaS          PaaS          SaaS           │
│   ──────────     ────          ────          ────           │
│   Application    Application   Application   ░░░░░░░░░░░    │
│   Data           Data          Data          ░░░░░░░░░░░    │
│   Runtime        Runtime       ░░░░░░░░░░░   ░░░░░░░░░░░    │
│   Middleware     Middleware    ░░░░░░░░░░░   ░░░░░░░░░░░    │
│   OS             ░░░░░░░░░░░   ░░░░░░░░░░░   ░░░░░░░░░░░    │
│   Virtualization ░░░░░░░░░░░   ░░░░░░░░░░░   ░░░░░░░░░░░    │
│   Servers        ░░░░░░░░░░░   ░░░░░░░░░░░   ░░░░░░░░░░░    │
│   Storage        ░░░░░░░░░░░   ░░░░░░░░░░░   ░░░░░░░░░░░    │
│   Networking     ░░░░░░░░░░░   ░░░░░░░░░░░   ░░░░░░░░░░░    │
│                                                              │
│   ░░░ = Géré par le fournisseur                             │
└─────────────────────────────────────────────────────────────┘
```

### Exemples de chaque modèle

| Modèle | Exemples | Vous gérez |
|--------|----------|------------|
| **IaaS** | OVH, AWS EC2, GCP Compute | VMs, OS, Runtime, App |
| **PaaS** | Clever Cloud, Heroku, Railway | App uniquement |
| **SaaS** | Gmail, Salesforce, Notion | Configuration seulement |

### Avantages du PaaS

- **Déploiement rapide** : Push & Deploy
- **Scaling automatique** : Sans configuration
- **Maintenance zéro** : OS, sécurité, patches = gérés
- **Focus sur le code** : Pas de DevOps nécessaire

### Inconvénients du PaaS

- **Moins de contrôle** : Pas d'accès SSH
- **Vendor lock-in** : Migration plus difficile
- **Coût à l'échelle** : Peut devenir cher

---

## Partie 2 : Création du compte Clever Cloud

### Étape 2.1 : Inscription

1. Rendez-vous sur [https://www.clever-cloud.com](https://www.clever-cloud.com)

2. Cliquez sur **"Login"** puis **"Sign up"**

3. Choisissez votre méthode :
   - **Email** (recommandé) : Compte classique
   - **GitHub** : Connexion rapide + déploiement facilité


5. Si vous utilisez GitHub :
   - Autorisez l'accès à votre compte
   - Clever Cloud pourra déployer depuis vos repos

6. Si vous utilisez Email :
   - Remplissez le formulaire
   - Validez votre email

### Étape 2.2 : Configuration du profil

1. Complétez votre profil si demandé

2. Choisissez **Personal** comme type d'organisation

3. Notez que Clever Cloud offre des **crédits gratuits** pour démarrer

---

## Partie 3 : Préparation de l'application

### Étape 3.1 : Créer le projet localement

```bash
mkdir -p ~/gameshelf-clever
cd ~/gameshelf-clever
```

### Étape 3.2 : Créer l'application Flask

```bash
nano app.py
```

```python
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

# Configuration depuis variables d'environnement
APP_NAME = os.environ.get('APP_NAME', 'GameShelf')
ENVIRONMENT = os.environ.get('ENVIRONMENT', 'development')

GAMES = [
    {"id": 1, "name": "Catan", "category": "Stratégie", "players": "3-4", "price": 39.99},
    {"id": 2, "name": "Pandemic", "category": "Coopératif", "players": "2-4", "price": 44.99},
    {"id": 3, "name": "Codenames", "category": "Party Game", "players": "4-8", "price": 24.99},
    {"id": 4, "name": "Ticket to Ride", "category": "Famille", "players": "2-5", "price": 42.99},
    {"id": 5, "name": "7 Wonders", "category": "Stratégie", "players": "3-7", "price": 49.99},
    {"id": 6, "name": "Dixit", "category": "Party Game", "players": "3-6", "price": 34.99},
]

@app.route('/')
def home():
    return jsonify({
        "service": f"{APP_NAME} API",
        "version": "1.0.0",
        "environment": ENVIRONMENT,
        "hostname": socket.gethostname(),
        "platform": "Clever Cloud",
        "endpoints": ["/", "/health", "/games", "/games/<id>", "/flag"]
    })

@app.route('/health')
def health():
    return jsonify({
        "status": "healthy",
        "environment": ENVIRONMENT,
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
        "message": "Bravo ! L'API tourne sur Clever Cloud !",
        "flag": "FLAG{Cl3v3r_Cl0ud_P44S_D3pl0y3d}",
        "platform": "Clever Cloud PaaS",
        "environment": ENVIRONMENT
    })

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
```

### Étape 3.3 : Créer requirements.txt

```bash
nano requirements.txt
```

```
flask==3.0.0
gunicorn==21.2.0
```

### Étape 3.4 : Initialiser Git

```bash
git init
git add .
git commit -m "Initial commit - GameShelf API pour Clever Cloud"
```

---

## Partie 4 : Déploiement sur Clever Cloud

### Étape 4.1 : Créer une nouvelle application

1. Connectez-vous à [console.clever-cloud.com](https://console.clever-cloud.com)

2. Cliquez sur **"Create"** > **"An application"**

3. Sélectionnez la source :
   - **"Brand new app"** (si repo pas sur GitHub)
   - Ou sélectionnez votre repo GitHub

### Étape 4.2 : Choisir le type d'application

1. Sélectionnez **"Python"**

2. Cliquez sur **"Next"**

### Étape 4.3 : Configuration de l'application

1. **Name** : `gameshelf-api`

2. **Region** : Paris (par)

3. **Instance type** : Choisissez **"Nano"** (gratuit pour les tests)
   - 512 MB RAM
   - 1 vCPU partagé

4. Cliquez sur **"Next"**

### Étape 4.4 : Variables d'environnement

Ajoutez ces variables :

| Variable | Valeur |
|----------|--------|
| `CC_PYTHON_MODULE` | `app:app` |
| `CC_PYTHON_VERSION` | `3.11` |
| `APP_NAME` | `GameShelf` |
| `ENVIRONMENT` | `production` |

> **Important** : `CC_PYTHON_MODULE` indique à Clever Cloud quel module WSGI lancer (ici l'objet `app` du fichier `app.py`).

Cliquez sur **"Next"**

### Étape 4.5 : Addons (optionnel)

Pour l'instant, on n'ajoute pas d'addon. Cliquez sur **"Create"**.

### Étape 4.6 : Obtenir l'URL Git

Après création, Clever Cloud vous donne une URL Git :

```
git remote add clever git+ssh://git@push-<region>.clever-cloud.com/<app_id>.git
```

Ou copiez-la depuis la console.

### Étape 4.7 : Pousser le code

```bash
git remote add clever <URL_GIT_CLEVER>
git push clever main
```

> Si votre branche s'appelle `master` : `git push clever master`

### Étape 4.8 : Suivre le déploiement

Dans la console Clever Cloud :
1. Allez dans votre application
2. Cliquez sur **"Logs"**
3. Observez le build et le démarrage

Ou via CLI :
```bash
clever logs --follow
```

---

## Partie 5 : Accès à l'application

### Étape 5.1 : Obtenir l'URL

Dans la console Clever Cloud :
1. Allez dans **"Domain names"**
2. Vous verrez une URL automatique : `https://app-<id>.cleverapps.io`

### Étape 5.2 : Tester l'API

```bash
# Remplacez par votre URL
export APP_URL="https://app-xxxxxxxx.cleverapps.io"

# Tester
curl $APP_URL/
curl $APP_URL/health
curl $APP_URL/games
curl $APP_URL/flag
```

### Étape 5.3 : Ajouter un domaine personnalisé (optionnel)

1. Dans **"Domain names"**
2. Ajoutez votre domaine : `api.gameshelf.com`
3. Configurez le DNS (CNAME vers Clever Cloud)

---

## Partie 6 : Scaling

### Étape 6.1 : Scaling vertical (changer l'instance)

1. Dans votre application, allez dans **"Scalability"**

2. Changez le type d'instance :
   - Nano (512 MB) → XS (1 GB) → S (2 GB)

3. L'application redémarre automatiquement avec plus de ressources

### Étape 6.2 : Scaling horizontal (plusieurs instances)

1. Dans **"Scalability"**

2. Augmentez le nombre d'instances :
   - Minimum : 1
   - Maximum : 3

3. Clever Cloud ajoute automatiquement des instances selon la charge

### Étape 6.3 : Autoscaling

Clever Cloud peut scaler automatiquement selon :
- CPU
- Mémoire
- Requêtes par seconde

Configuration dans **"Scalability" > "Autoscaling"**.

---

## Partie 7 : Variables d'environnement et Secrets

### Étape 7.1 : Ajouter des variables

1. Allez dans **"Environment variables"**

2. Ajoutez des variables :
   ```
   SECRET_KEY=ma_cle_secrete_123
   DEBUG=false
   API_VERSION=1.0.0
   ```

3. Cliquez sur **"Update changes"**

4. L'application redémarre automatiquement

### Étape 7.2 : Variables depuis la CLI

Si vous avez installé le CLI Clever Cloud :

```bash
# Installer le CLI
npm install -g clever-tools

# Se connecter
clever login

# Lier l'application
clever link <app_id>

# Ajouter une variable
clever env set MY_VAR="my_value"

# Voir les variables
clever env
```

---

## Partie 8 : Logs et Monitoring

### Étape 8.1 : Voir les logs

Dans la console :
1. Allez dans **"Logs"**
2. Filtrez par type (access, app, error)

Via CLI :
```bash
clever logs --follow
```

### Étape 8.2 : Métriques

Dans **"Metrics"** :
- CPU usage
- Memory usage
- Network I/O
- Request count

### Étape 8.3 : Alertes

Configurez des alertes dans **"Notifications"** :
- Déploiement réussi/échoué
- Scaling events
- Erreurs

---

## Partie 9 : Comparaison IaaS vs PaaS

### Ce que nous avons fait en IaaS (VMs + Terraform + Ansible)

1. Créer une VM
2. Configurer SSH
3. Installer Python
4. Installer pip
5. Installer les dépendances
6. Configurer systemd
7. Installer Nginx
8. Configurer le reverse proxy
9. Gérer les certificats SSL
10. Configurer les backups
11. Monitorer les ressources

**Temps estimé : 2-4 heures**

### Ce que nous avons fait en PaaS (Clever Cloud)

1. Push Git
2. ✅ Déployé !

**Temps estimé : 10 minutes**

### Quand choisir quoi ?

| Critère | IaaS | PaaS |
|---------|------|------|
| Contrôle total | ✅ | ❌ |
| Rapidité de déploiement | ❌ | ✅ |
| Coût à petite échelle | ✅ | ✅ |
| Coût à grande échelle | ✅ | Variable |
| Complexité ops | Élevée | Faible |
| Customisation | Totale | Limitée |

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé un compte Clever Cloud
- [ ] Préparé l'application Flask
- [ ] Créé une application Python sur Clever Cloud
- [ ] Déployé via Git push
- [ ] Testé l'API sur l'URL Clever Cloud
- [ ] Configuré des variables d'environnement
- [ ] Observé les logs
- [ ] Récupéré le flag via `/flag`

---

## Flag de validation

Accédez à l'endpoint `/flag` de votre application :

```bash
curl https://app-xxxxxxxx.cleverapps.io/flag
```

```
FLAG{Cl3v3r_Cl0ud_P44S_D3pl0y3d}
```

---

## Transition vers le TP suivant

*"Déployer une application en quelques minutes, c'est magique ! Mais notre API a besoin de stocker des données. Dans le prochain TP, nous allons ajouter une base de données PostgreSQL managée !"*

---

## Structure du projet

```
gameshelf-clever/
├── app.py              # Application Flask
└── requirements.txt    # Dépendances Python
```

---

## Commandes CLI Clever Cloud

| Commande | Description |
|----------|-------------|
| `clever login` | Se connecter |
| `clever create` | Créer une app |
| `clever link` | Lier un repo à une app |
| `clever deploy` | Déployer |
| `clever logs` | Voir les logs |
| `clever env` | Gérer les variables |
| `clever scale` | Configurer le scaling |
| `clever restart` | Redémarrer |

---

## En cas de problème

| Problème | Solution |
|----------|----------|
| Build échoue | Vérifiez requirements.txt |
| App ne démarre pas | Vérifiez `CC_PYTHON_MODULE` et les logs |
| Port incorrect | Utilisez `$PORT` (variable d'env) |
| Push refusé | Vérifiez les droits Git et la clé SSH |

---

**Durée estimée : 45 minutes**

**Difficulté : Facile**
