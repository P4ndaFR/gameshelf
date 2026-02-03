# TP 15 - Gestion des Secrets avec HashiCorp Vault sur Clever Cloud

## Contexte GameShelf

Notre API est sécurisée avec Keycloak, mais nos secrets (credentials DB, clés S3, secrets Keycloak) sont stockés en variables d'environnement. Si quelqu'un accède à la console Clever Cloud ou à notre CI/CD, il voit tous les secrets en clair !

**Objectif de ce TP :** Déployer HashiCorp Vault sur Clever Cloud et configurer l'API GameShelf pour récupérer ses secrets depuis Vault.

---

## Prérequis

- TP 14 complété
- Compte Clever Cloud actif
- Application GameShelf déployée

---

## Partie 1 : Pourquoi un gestionnaire de secrets ?

### Le problème des secrets

```
┌─────────────────────────────────────────────────────────────┐
│                OÙ SONT VOS SECRETS ?                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ❌ Dans le code        → Visible dans Git                  │
│  ❌ En .env             → Risque de commit accidentel       │
│  ⚠️  Vars environnement → Visible console, logs, dumps      │
│  ✅ Vault               → Chiffré, audité, rotatif          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Fonctionnalités d'un gestionnaire de secrets

| Fonctionnalité | Description |
|----------------|-------------|
| **Stockage chiffré** | Secrets chiffrés au repos |
| **Contrôle d'accès** | Qui peut lire quels secrets |
| **Audit** | Logs de tous les accès |
| **Rotation** | Changement automatique des secrets |
| **Injection dynamique** | Secrets injectés à la demande |
| **Révocation** | Invalider un secret compromis |

### HashiCorp Vault

Vault est le standard de l'industrie pour la gestion des secrets :

- Open source
- Multi-cloud
- Support de nombreux backends (DB, Cloud, PKI...)
- API REST complète
- Haute disponibilité

---

## Partie 2 : Déploiement de Vault sur Clever Cloud

### Étape 2.1 : Créer l'application Docker

1. Connectez-vous à [console.clever-cloud.com](https://console.clever-cloud.com)

2. Cliquez sur **"Create"** > **"An application"**

3. Sélectionnez **"Docker"**

4. Configurez l'application :
   - **Name** : `gameshelf-vault`
   - **Region** : Paris
   - **Size** : S (minimum 1 GB RAM pour Vault)

5. Notez l'URL Git fournie par Clever Cloud

### Étape 2.2 : Créer une base PostgreSQL pour Vault

1. Dans le menu principal, cliquez sur **"Create"** > **"An add-on"**

2. Sélectionnez **"PostgreSQL"**

3. Choisissez le plan **"DEV"** (gratuit)

4. **Nom** : `vault-db`

5. **Lier à une application** : Sélectionnez `gameshelf-vault`

6. Cliquez sur **"Create"**

### Étape 2.3 : Créer le projet Vault

```bash
mkdir -p ~/gameshelf-vault
cd ~/gameshelf-vault
```

### Étape 2.4 : Créer le Dockerfile

```bash
nano Dockerfile
```

```dockerfile
FROM hashicorp/vault:1.15

# Copier la configuration
COPY vault-config.hcl /vault/config/vault-config.hcl
COPY entrypoint.sh /entrypoint.sh

RUN chmod +x /entrypoint.sh

# Vault écoute sur le port 8200
EXPOSE 8200

ENTRYPOINT ["/entrypoint.sh"]
```

### Étape 2.5 : Créer la configuration Vault

```bash
nano vault-config.hcl
```

```hcl
ui = true

listener "tcp" {
  address     = "0.0.0.0:8200"
  tls_disable = true
}

storage "postgresql" {
  connection_url = "postgres://POSTGRESQL_ADDON_USER:POSTGRESQL_ADDON_PASSWORD@POSTGRESQL_ADDON_HOST:POSTGRESQL_ADDON_PORT/POSTGRESQL_ADDON_DB?sslmode=require"
}

api_addr = "https://VAULT_EXTERNAL_URL"

disable_mlock = true
```

### Étape 2.6 : Créer le script d'entrée

```bash
nano entrypoint.sh
```

```bash
#!/bin/sh
set -e

# Remplacer les variables dans la configuration
sed -i "s|POSTGRESQL_ADDON_USER|${POSTGRESQL_ADDON_USER}|g" /vault/config/vault-config.hcl
sed -i "s|POSTGRESQL_ADDON_PASSWORD|${POSTGRESQL_ADDON_PASSWORD}|g" /vault/config/vault-config.hcl
sed -i "s|POSTGRESQL_ADDON_HOST|${POSTGRESQL_ADDON_HOST}|g" /vault/config/vault-config.hcl
sed -i "s|POSTGRESQL_ADDON_PORT|${POSTGRESQL_ADDON_PORT}|g" /vault/config/vault-config.hcl
sed -i "s|POSTGRESQL_ADDON_DB|${POSTGRESQL_ADDON_DB}|g" /vault/config/vault-config.hcl
sed -i "s|VAULT_EXTERNAL_URL|${VAULT_EXTERNAL_URL}|g" /vault/config/vault-config.hcl

echo "=== Vault Configuration ==="
cat /vault/config/vault-config.hcl
echo "==========================="

# Démarrer Vault
exec vault server -config=/vault/config/vault-config.hcl
```

### Étape 2.7 : Configurer les variables d'environnement

Dans l'application `gameshelf-vault` sur Clever Cloud, ajoutez :

| Variable | Valeur |
|----------|--------|
| `PORT` | `8200` |
| `VAULT_EXTERNAL_URL` | `https://app-xxxxxxxx.cleverapps.io` (URL de votre app Vault) |

### Étape 2.8 : Déployer Vault

```bash
git init
git add .
git commit -m "Initial Vault setup"
git remote add clever <URL_GIT_CLEVER_VAULT>
git push clever main:master
```

### Étape 2.9 : Vérifier le déploiement

1. Attendez que le déploiement soit terminé
2. Accédez à l'URL de votre Vault : `https://app-vault-xxxxx.cleverapps.io`
3. Vous devriez voir l'interface d'initialisation de Vault

---

## Partie 3 : Initialisation de Vault

### Étape 3.1 : Initialiser Vault via l'interface web

1. Accédez à l'URL de votre Vault

2. Sur l'écran d'initialisation, configurez :
   - **Key shares** : 1 (pour la démo ; en production, utilisez 5)
   - **Key threshold** : 1 (pour la démo ; en production, utilisez 3)

3. Cliquez sur **"Initialize"**

4. **IMPORTANT** : Sauvegardez précieusement :
   - **Initial root token** : `hvs.xxxxx` (pour l'administration)
   - **Unseal key** : `xxxxx` (pour débloquer Vault)

### Étape 3.2 : Débloquer (Unseal) Vault

1. Entrez votre **Unseal Key**
2. Cliquez sur **"Unseal"**
3. Vault passe en état "unsealed" (débloqué)

### Étape 3.3 : Se connecter

1. Sélectionnez la méthode **"Token"**
2. Entrez votre **Root Token**
3. Cliquez sur **"Sign in"**

---

## Partie 4 : Configuration des secrets

### Étape 4.1 : Activer le moteur de secrets KV

1. Dans l'interface Vault, allez dans **"Secrets Engines"**
2. Cliquez sur **"Enable new engine"**
3. Sélectionnez **"KV"** (Key-Value)
4. Configurez :
   - **Path** : `gameshelf`
   - **Version** : 2 (avec versioning)
5. Cliquez sur **"Enable Engine"**

### Étape 4.2 : Créer les secrets via l'interface

1. Cliquez sur **"gameshelf/"**
2. Cliquez sur **"Create secret"**

**Secret 1 : database**
- **Path** : `database`
- **Secret data** :
  ```
  host = <votre POSTGRESQL_ADDON_HOST de gameshelf-api>
  port = <votre POSTGRESQL_ADDON_PORT>
  database = <votre POSTGRESQL_ADDON_DB>
  username = <votre POSTGRESQL_ADDON_USER>
  password = <votre POSTGRESQL_ADDON_PASSWORD>
  ```

**Secret 2 : s3**
- **Path** : `s3`
- **Secret data** :
  ```
  endpoint = cellar-c2.services.clever-cloud.com
  access_key = <votre CELLAR_ADDON_KEY_ID>
  secret_key = <votre CELLAR_ADDON_KEY_SECRET>
  ```

**Secret 3 : keycloak**
- **Path** : `keycloak`
- **Secret data** :
  ```
  url = <URL de votre Keycloak>
  realm = gameshelf
  client_id = gameshelf-api
  client_secret = <votre client secret>
  ```

**Secret 4 : app**
- **Path** : `app`
- **Secret data** :
  ```
  secret_key = flask_secret_key_very_secure_123
  environment = production
  ```

### Étape 4.3 : Créer une politique d'accès

1. Allez dans **"Policies"**
2. Cliquez sur **"Create ACL policy"**
3. Configurez :
   - **Name** : `gameshelf-app`
   - **Policy** :

```hcl
# Lecture seule sur les secrets GameShelf
path "gameshelf/data/*" {
  capabilities = ["read", "list"]
}

# Lecture des métadonnées
path "gameshelf/metadata/*" {
  capabilities = ["read", "list"]
}
```

4. Cliquez sur **"Create policy"**

### Étape 4.4 : Créer un token applicatif

1. Allez dans **"Access"** > **"Auth Methods"**
2. Cliquez sur **"token/"** (activé par défaut)
3. Allez dans l'onglet **"Tokens"**
4. Cliquez sur **"Create token"**
5. Configurez :
   - **Policies** : `gameshelf-app`
   - **TTL** : `768h` (32 jours)
   - **Renewable** : Yes
6. Cliquez sur **"Create token"**
7. **Sauvegardez le token généré** : `hvs.xxxxx`

---

## Partie 5 : Intégration dans l'API GameShelf

### Étape 5.1 : Mettre à jour requirements.txt

```bash
cd ~/gameshelf-clever
nano requirements.txt
```

```
flask==3.0.0
gunicorn==21.2.0
psycopg2-binary==2.9.9
boto3==1.34.0
PyJWT==2.8.0
cryptography==41.0.0
requests==2.31.0
hvac==2.1.0
```

### Étape 5.2 : Mettre à jour app.py

```python
from flask import Flask, jsonify, request, g
import os
import socket
import psycopg2
from psycopg2.extras import RealDictCursor
import boto3
from botocore.client import Config
import logging
import time
import random
import jwt
import requests
from functools import wraps
import hvac

# Configuration du logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(name)s - %(message)s'
)
logger = logging.getLogger('gameshelf')

app = Flask(__name__)

# Configuration
APP_NAME = os.environ.get('APP_NAME', 'GameShelf')
ENVIRONMENT = os.environ.get('ENVIRONMENT', 'development')

# Configuration Vault
VAULT_ADDR = os.environ.get('VAULT_ADDR', '')
VAULT_TOKEN = os.environ.get('VAULT_TOKEN', '')
VAULT_PATH = 'gameshelf'
USE_VAULT = bool(VAULT_ADDR and VAULT_TOKEN)

# Cache pour les secrets
_secrets_cache = {}
_secrets_cache_time = 0
SECRETS_CACHE_DURATION = 300  # 5 minutes

# Client Vault
_vault_client = None


def get_vault_client():
    """Crée et retourne un client Vault."""
    global _vault_client
    if _vault_client is None and USE_VAULT:
        _vault_client = hvac.Client(
            url=VAULT_ADDR,
            token=VAULT_TOKEN
        )
        if _vault_client.is_authenticated():
            logger.info("Connected to Vault")
        else:
            logger.error("Failed to authenticate with Vault")
            _vault_client = None
    return _vault_client


def get_secret(path):
    """Récupère un secret depuis Vault avec cache."""
    global _secrets_cache, _secrets_cache_time

    # Vérifier le cache
    cache_key = path
    if cache_key in _secrets_cache and (time.time() - _secrets_cache_time) < SECRETS_CACHE_DURATION:
        return _secrets_cache[cache_key]

    if not USE_VAULT:
        return None

    try:
        client = get_vault_client()
        if not client:
            return None

        response = client.secrets.kv.v2.read_secret_version(
            path=path,
            mount_point=VAULT_PATH
        )
        data = response['data']['data']

        # Mettre en cache
        _secrets_cache[cache_key] = data
        _secrets_cache_time = time.time()

        logger.info(f"Secret '{path}' retrieved from Vault")
        return data
    except Exception as e:
        logger.error(f"Failed to get secret '{path}': {e}")
        return None


def get_config(secret_path, key, env_fallback):
    """Récupère une config depuis Vault ou fallback sur env var."""
    if USE_VAULT:
        secret = get_secret(secret_path)
        if secret and key in secret:
            return secret[key]
    return os.environ.get(env_fallback, '')


# Configuration Keycloak (depuis Vault ou env vars)
def get_keycloak_config():
    if USE_VAULT:
        secret = get_secret('keycloak')
        if secret:
            return {
                'url': secret.get('url', ''),
                'realm': secret.get('realm', 'gameshelf'),
                'client_id': secret.get('client_id', 'gameshelf-api'),
                'client_secret': secret.get('client_secret', '')
            }
    return {
        'url': os.environ.get('KEYCLOAK_URL') or os.environ.get('KEYCLOAK_ADDON_URL', ''),
        'realm': os.environ.get('KEYCLOAK_REALM', 'gameshelf'),
        'client_id': os.environ.get('KEYCLOAK_CLIENT_ID', 'gameshelf-api'),
        'client_secret': os.environ.get('KEYCLOAK_CLIENT_SECRET', '')
    }


# Cache pour les clés JWKS
_jwks_cache = None
_jwks_cache_time = 0
JWKS_CACHE_DURATION = 300


def get_jwks():
    """Récupère les clés publiques de Keycloak (avec cache)."""
    global _jwks_cache, _jwks_cache_time

    if _jwks_cache and (time.time() - _jwks_cache_time) < JWKS_CACHE_DURATION:
        return _jwks_cache

    try:
        kc_config = get_keycloak_config()
        certs_url = f"{kc_config['url']}/realms/{kc_config['realm']}/protocol/openid-connect/certs"
        response = requests.get(certs_url, timeout=10)
        response.raise_for_status()
        _jwks_cache = response.json()
        _jwks_cache_time = time.time()
        logger.info("JWKS cache refreshed")
        return _jwks_cache
    except Exception as e:
        logger.error(f"Failed to fetch JWKS: {e}")
        return _jwks_cache


def get_public_key(token):
    """Extrait la clé publique correspondant au token."""
    jwks = get_jwks()
    if not jwks:
        raise Exception("Unable to fetch JWKS")

    unverified_header = jwt.get_unverified_header(token)
    kid = unverified_header.get('kid')

    for key in jwks.get('keys', []):
        if key.get('kid') == kid:
            return jwt.algorithms.RSAAlgorithm.from_jwk(key)

    raise Exception(f"Public key not found for kid: {kid}")


def verify_token(token):
    """Vérifie et décode un token JWT."""
    try:
        kc_config = get_keycloak_config()
        public_key = get_public_key(token)
        decoded = jwt.decode(
            token,
            public_key,
            algorithms=['RS256'],
            audience=kc_config['client_id'],
            options={"verify_exp": True}
        )
        return decoded
    except jwt.ExpiredSignatureError:
        logger.warning("Token expired")
        raise
    except jwt.InvalidTokenError as e:
        logger.warning(f"Invalid token: {e}")
        raise


def require_auth(f):
    """Décorateur pour exiger l'authentification."""
    @wraps(f)
    def decorated(*args, **kwargs):
        auth_header = request.headers.get('Authorization')

        if not auth_header:
            return jsonify({'error': 'Authorization header missing'}), 401

        if not auth_header.startswith('Bearer '):
            return jsonify({'error': 'Invalid authorization format'}), 401

        token = auth_header.split(' ')[1]

        try:
            decoded = verify_token(token)
            g.user = decoded
            g.username = decoded.get('preferred_username', 'unknown')
            g.roles = decoded.get('realm_access', {}).get('roles', [])
            logger.info(f"Authenticated user: {g.username}, roles: {g.roles}")
        except jwt.ExpiredSignatureError:
            return jsonify({'error': 'Token expired'}), 401
        except Exception as e:
            logger.error(f"Authentication failed: {e}")
            return jsonify({'error': 'Invalid token'}), 401

        return f(*args, **kwargs)
    return decorated


def require_role(role):
    """Décorateur pour exiger un rôle spécifique."""
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            if not hasattr(g, 'roles'):
                return jsonify({'error': 'Not authenticated'}), 401

            if role not in g.roles and 'admin' not in g.roles:
                logger.warning(f"User {g.username} denied access - requires role: {role}")
                return jsonify({'error': f'Requires role: {role}'}), 403

            return f(*args, **kwargs)
        return decorated
    return decorator


def get_s3_client():
    """Crée un client S3 pour Cellar (depuis Vault ou env vars)."""
    if USE_VAULT:
        secret = get_secret('s3')
        if secret:
            return boto3.client(
                's3',
                endpoint_url=f"https://{secret.get('endpoint', '')}",
                aws_access_key_id=secret.get('access_key'),
                aws_secret_access_key=secret.get('secret_key'),
                config=Config(
                    signature_version='s3v4',
                    s3={'addressing_style': 'virtual'}
                )
            )

    return boto3.client(
        's3',
        endpoint_url=f"https://{os.environ.get('CELLAR_ADDON_HOST', '')}",
        aws_access_key_id=os.environ.get('CELLAR_ADDON_KEY_ID'),
        aws_secret_access_key=os.environ.get('CELLAR_ADDON_KEY_SECRET'),
        config=Config(
            signature_version='s3v4',
            s3={'addressing_style': 'virtual'}
        )
    )


def get_db_connection():
    """Crée une connexion à PostgreSQL (depuis Vault ou env vars)."""
    if USE_VAULT:
        secret = get_secret('database')
        if secret:
            return psycopg2.connect(
                host=secret.get('host'),
                port=secret.get('port'),
                database=secret.get('database'),
                user=secret.get('username'),
                password=secret.get('password'),
                cursor_factory=RealDictCursor
            )

    return psycopg2.connect(
        host=os.environ.get('POSTGRESQL_ADDON_HOST'),
        port=os.environ.get('POSTGRESQL_ADDON_PORT'),
        database=os.environ.get('POSTGRESQL_ADDON_DB'),
        user=os.environ.get('POSTGRESQL_ADDON_USER'),
        password=os.environ.get('POSTGRESQL_ADDON_PASSWORD'),
        cursor_factory=RealDictCursor
    )


# Middleware pour le logging
@app.before_request
def before_request():
    g.start_time = time.time()
    g.request_id = f"{time.time()}-{random.randint(1000, 9999)}"

@app.after_request
def after_request(response):
    if hasattr(g, 'start_time'):
        elapsed = (time.time() - g.start_time) * 1000
        user = getattr(g, 'username', 'anonymous')
        logger.info(f"[{g.request_id}] {user} - {request.method} {request.path} - {response.status_code} in {elapsed:.2f}ms")
    return response


@app.route('/')
def home():
    return jsonify({
        "service": f"{APP_NAME} API",
        "version": "6.0.0",
        "environment": ENVIRONMENT,
        "hostname": socket.gethostname(),
        "auth": "Keycloak OIDC",
        "secrets": "HashiCorp Vault" if USE_VAULT else "Environment Variables",
        "endpoints": {
            "public": ["/", "/health", "/games"],
            "authenticated": ["/games/<id>", "/profile"],
            "staff_only": ["/customers", "/rentals"],
            "admin_only": ["/admin/stats", "/vault-status"]
        }
    })


@app.route('/health')
def health():
    db_status = "unknown"
    keycloak_status = "unknown"
    vault_status = "unknown"

    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute('SELECT 1')
        cur.close()
        conn.close()
        db_status = "connected"
    except Exception as e:
        db_status = "error"
        logger.error(f"DB health check failed: {e}")

    try:
        kc_config = get_keycloak_config()
        response = requests.get(f"{kc_config['url']}/realms/{kc_config['realm']}", timeout=5)
        keycloak_status = "connected" if response.ok else "error"
    except:
        keycloak_status = "unreachable"

    if USE_VAULT:
        try:
            client = get_vault_client()
            vault_status = "connected" if client and client.is_authenticated() else "error"
        except:
            vault_status = "error"
    else:
        vault_status = "disabled"

    return jsonify({
        "status": "healthy" if db_status == "connected" else "degraded",
        "database": db_status,
        "keycloak": keycloak_status,
        "vault": vault_status,
        "hostname": socket.gethostname()
    })


@app.route('/vault-status')
@require_auth
@require_role('admin')
def vault_status():
    """Statut détaillé de Vault - admin uniquement."""
    if not USE_VAULT:
        return jsonify({
            "enabled": False,
            "message": "Vault is not configured. Set VAULT_ADDR and VAULT_TOKEN."
        })

    try:
        client = get_vault_client()
        if not client:
            return jsonify({"enabled": True, "status": "error", "message": "Cannot connect to Vault"}), 500

        # Vérifier les secrets disponibles
        secrets_status = {}
        for secret_name in ['database', 's3', 'keycloak', 'app']:
            secret = get_secret(secret_name)
            secrets_status[secret_name] = {
                "available": secret is not None,
                "keys": list(secret.keys()) if secret else []
            }

        return jsonify({
            "enabled": True,
            "status": "connected",
            "vault_addr": VAULT_ADDR,
            "secrets": secrets_status,
            "cache_duration": SECRETS_CACHE_DURATION
        })
    except Exception as e:
        return jsonify({"enabled": True, "status": "error", "message": str(e)}), 500


@app.route('/games')
def get_games():
    """Liste des jeux - accessible à tous."""
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("""
            SELECT g.id, g.name, c.name as category,
                   g.min_players, g.max_players, g.price
            FROM games g
            JOIN categories c ON g.category_id = c.id
            ORDER BY g.name
        """)
        games = cur.fetchall()
        cur.close()
        conn.close()
        return jsonify({"count": len(games), "games": games})
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route('/games/<int:game_id>')
@require_auth
def get_game(game_id):
    """Détails d'un jeu - nécessite authentification."""
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("""
            SELECT g.id, g.name, c.name as category,
                   g.min_players, g.max_players, g.duration_minutes,
                   g.price, g.stock
            FROM games g
            JOIN categories c ON g.category_id = c.id
            WHERE g.id = %s
        """, (game_id,))
        game = cur.fetchone()
        cur.close()
        conn.close()

        if game:
            return jsonify(game)
        return jsonify({"error": "Game not found"}), 404
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route('/profile')
@require_auth
def get_profile():
    """Profil de l'utilisateur connecté."""
    return jsonify({
        "username": g.username,
        "roles": g.roles,
        "token_info": {
            "email": g.user.get('email'),
            "name": g.user.get('name'),
            "exp": g.user.get('exp')
        }
    })


@app.route('/customers')
@require_auth
@require_role('staff')
def get_customers():
    """Liste des clients - réservé au staff."""
    logger.info(f"Staff access by {g.username}")
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("""
            SELECT id, email, first_name, last_name, loyalty_points
            FROM customers
            ORDER BY last_name, first_name
        """)
        customers = cur.fetchall()
        cur.close()
        conn.close()
        return jsonify({"count": len(customers), "customers": customers})
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route('/rentals')
@require_auth
@require_role('staff')
def get_rentals():
    """Liste des locations - réservé au staff."""
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("""
            SELECT r.id, c.first_name || ' ' || c.last_name as customer,
                   g.name as game, r.rental_date, r.due_date, r.status
            FROM rentals r
            JOIN customers c ON r.customer_id = c.id
            JOIN games g ON r.game_id = g.id
            ORDER BY r.rental_date DESC
        """)
        rentals = cur.fetchall()
        cur.close()
        conn.close()
        return jsonify({"count": len(rentals), "rentals": rentals})
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route('/admin/stats')
@require_auth
@require_role('admin')
def admin_stats():
    """Statistiques admin - réservé aux administrateurs."""
    logger.info(f"Admin access by {g.username}")
    try:
        conn = get_db_connection()
        cur = conn.cursor()

        cur.execute("SELECT COUNT(*) as total FROM games")
        games_count = cur.fetchone()['total']

        cur.execute("SELECT COUNT(*) as total FROM customers")
        customers_count = cur.fetchone()['total']

        cur.execute("SELECT COUNT(*) as total FROM rentals WHERE status = 'active'")
        active_rentals = cur.fetchone()['total']

        cur.execute("SELECT SUM(price) as total FROM games")
        total_value = cur.fetchone()['total']

        cur.close()
        conn.close()

        return jsonify({
            "stats": {
                "games": games_count,
                "customers": customers_count,
                "active_rentals": active_rentals,
                "inventory_value": float(total_value) if total_value else 0
            },
            "accessed_by": g.username,
            "timestamp": time.time()
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route('/flag')
@require_auth
@require_role('admin')
def get_flag():
    """Flag - réservé aux administrateurs."""
    vault_connected = False
    if USE_VAULT:
        try:
            client = get_vault_client()
            vault_connected = client and client.is_authenticated()
        except:
            pass

    return jsonify({
        "message": "Bravo ! L'API utilise Vault pour les secrets !" if vault_connected else "API fonctionnelle mais Vault non configuré",
        "flag": "FLAG{V4ult_S3cr3ts_0n_Cl3v3r_Cl0ud}" if vault_connected else "FLAG{K3ycl04k_0IDC_S3cur3d}",
        "user": g.username,
        "roles": g.roles,
        "vault_enabled": USE_VAULT,
        "vault_connected": vault_connected
    })


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    logger.info(f"Starting GameShelf API on port {port}")
    logger.info(f"Vault: {'enabled' if USE_VAULT else 'disabled'}")
    app.run(host='0.0.0.0', port=port)
```

### Étape 5.3 : Configurer les variables d'environnement

Dans l'application `gameshelf-api` sur Clever Cloud, ajoutez :

| Variable | Valeur |
|----------|--------|
| `VAULT_ADDR` | `https://app-vault-xxxxx.cleverapps.io` |
| `VAULT_TOKEN` | Le token applicatif créé à l'étape 4.4 |

### Étape 5.4 : Déployer

```bash
git add .
git commit -m "Add Vault integration"
git push clever main:master
```

---

## Partie 6 : Tester l'intégration

### Étape 6.1 : Vérifier le health check

```bash
export API_URL="https://app-gameshelf-xxxxx.cleverapps.io"

curl "$API_URL/health"
```

Vous devriez voir :
```json
{
  "status": "healthy",
  "database": "connected",
  "keycloak": "connected",
  "vault": "connected"
}
```

### Étape 6.2 : Tester le statut Vault (admin)

```bash
# Obtenir un token admin
export KEYCLOAK_URL="<URL_KEYCLOAK>"
TOKEN=$(curl -s -X POST \
  "$KEYCLOAK_URL/realms/gameshelf/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=gameshelf-api" \
  -d "client_secret=VOTRE_CLIENT_SECRET" \
  -d "username=admin-user" \
  -d "password=admin123" \
  -d "grant_type=password" | jq -r '.access_token')

# Vérifier le statut Vault
curl -H "Authorization: Bearer $TOKEN" "$API_URL/vault-status"
```

### Étape 6.3 : Récupérer le flag

```bash
curl -H "Authorization: Bearer $TOKEN" "$API_URL/flag"
```

---

## Partie 7 : Bonnes pratiques

### Gestion des tokens Vault

| Pratique | Description |
|----------|-------------|
| **TTL court** | Tokens avec durée de vie limitée (24h-7j) |
| **Renouvellement** | Configurer le renouvellement automatique |
| **Révocation** | Révoquer immédiatement les tokens compromis |
| **Audit** | Activer les logs d'audit |

### Rotation des secrets

1. Mettre à jour le secret dans Vault (nouvelle version)
2. L'application récupère automatiquement la nouvelle version (après expiration du cache)
3. Aucun redéploiement nécessaire !

### Architecture production

```
┌─────────────────────────────────────────────────────────────┐
│                   ARCHITECTURE VAULT                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐  │
│   │  GameShelf  │────►│    Vault    │────►│  PostgreSQL │  │
│   │     API     │     │   Server    │     │   Backend   │  │
│   └─────────────┘     └─────────────┘     └─────────────┘  │
│         │                   │                              │
│         │              Secrets:                            │
│         │              - database                          │
│         │              - s3                                │
│         │              - keycloak                          │
│         │              - app                               │
│         │                                                  │
│         └── Token Auth (hvs.xxx)                           │
│                                                            │
└─────────────────────────────────────────────────────────────┘
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Déployé Vault sur Clever Cloud (Docker)
- [ ] Créé la base PostgreSQL pour Vault
- [ ] Initialisé et débloqué Vault
- [ ] Créé les secrets (database, s3, keycloak, app)
- [ ] Créé la politique d'accès
- [ ] Créé le token applicatif
- [ ] Modifié l'API pour utiliser Vault
- [ ] Testé le health check avec Vault connecté
- [ ] Testé le endpoint /vault-status
- [ ] Récupéré le flag avec Vault activé

---

## Flag de validation

```
FLAG{V4ult_S3cr3ts_0n_Cl3v3r_Cl0ud}
```

---

## Conclusion du cours

Félicitations ! Vous avez terminé le cours Cloud Computing.

### Récapitulatif des compétences acquises

| Module | Compétence |
|--------|------------|
| 1-2 | Infrastructure Cloud, VMs |
| 3-4 | Infrastructure as Code (Terraform, Ansible) |
| 5 | Haute disponibilité, Load Balancing |
| 6 | Conteneurisation (Docker) |
| 7-8 | Orchestration (Kubernetes) |
| 9-10 | PaaS, Base de données managée |
| 11 | Stockage objet (S3) |
| 12-13 | Observabilité (Logs, Métriques) |
| 14 | Authentification (Keycloak, OIDC) |
| 15 | Gestion des secrets (Vault) |

### Architecture finale GameShelf

```
┌─────────────────────────────────────────────────────────────┐
│                   ARCHITECTURE GAMESHELF                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   Utilisateurs ──► Keycloak ──► API GameShelf               │
│                        │              │                     │
│                        │         ┌────┴────┐                │
│                        │         │         │                │
│                        ▼         ▼         ▼                │
│                     Vault    PostgreSQL   S3                │
│                    (secrets)   (data)   (files)             │
│                        │         │         │                │
│                        └────┬────┴────┬────┘                │
│                             │         │                     │
│                          Logs    Métriques                  │
│                             │         │                     │
│                             ▼         ▼                     │
│                        Observabilité Clever Cloud           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

**Durée estimée : 1h30**

**Difficulté : Avancé**

---

*Merci d'avoir suivi ce cours ! Bon courage pour la suite de votre parcours Cloud !*
