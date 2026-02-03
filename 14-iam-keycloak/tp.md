# TP 14 - IAM : Authentification avec Keycloak

## Contexte GameShelf

Notre API expose des données clients et des informations de location. Actuellement, n'importe qui peut accéder à ces endpoints ! Il faut sécuriser l'accès avec une authentification robuste.

**Objectif de ce TP :** Configurer l'addon Keycloak sur Clever Cloud et sécuriser l'API GameShelf avec OpenID Connect.

---

## Prérequis

- TP 13 complété
- Compte Clever Cloud actif
- Application GameShelf déployée

---

## Partie 1 : Concepts d'authentification

### Authentification vs Autorisation

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   AUTHENTIFICATION              AUTORISATION                │
│   ─────────────────             ────────────                │
│   "Qui êtes-vous ?"             "Que pouvez-vous faire ?"   │
│                                                             │
│   ┌─────────┐                   ┌─────────┐                 │
│   │ Login   │                   │ Rôles   │                 │
│   │ MDP     │                   │ Perms   │                 │
│   │ Token   │                   │ Scopes  │                 │
│   └─────────┘                   └─────────┘                 │
│                                                             │
│   Identity ───────► Access Decision ───────► Resource       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Protocoles d'authentification

| Protocole | Usage | Caractéristiques |
|-----------|-------|------------------|
| **OAuth 2.0** | Autorisation | Délégation d'accès, tokens |
| **OpenID Connect** | Authentification | Basé sur OAuth 2.0, ID tokens |
| **SAML** | Enterprise SSO | XML, lourd, legacy |
| **JWT** | Format de token | Compact, signé, stateless |

### Flux OpenID Connect

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐
│  Client  │     │   Keycloak   │     │   GameShelf  │
│  (App)   │     │   (IdP)      │     │   API        │
└────┬─────┘     └──────┬───────┘     └──────┬───────┘
     │                  │                     │
     │  1. Login        │                     │
     ├─────────────────►│                     │
     │                  │                     │
     │  2. Auth + Token │                     │
     │◄─────────────────┤                     │
     │                  │                     │
     │  3. API Call + Token                   │
     ├────────────────────────────────────────►
     │                  │                     │
     │                  │  4. Validate Token  │
     │                  │◄────────────────────┤
     │                  │                     │
     │  5. Response     │                     │
     │◄────────────────────────────────────────
```

---

## Partie 2 : Création de l'addon Keycloak

### Étape 2.1 : Créer l'addon Keycloak

1. Connectez-vous à [console.clever-cloud.com](https://console.clever-cloud.com)

2. Dans le menu principal, cliquez sur **"Create"** > **"An add-on"**

3. Sélectionnez **"Keycloak"**

4. Choisissez le plan :
   - **DEV** : Pour le développement (gratuit)

5. **Nom** : `gameshelf-keycloak`

6. **Lier à une application** : Sélectionnez `gameshelf-api` pour lier l'addon pendant la création

7. Cliquez sur **"Create"**

### Étape 2.2 : Récupérer les informations de connexion

Une fois l'addon créé, des variables d'environnement sont automatiquement injectées :

- `KEYCLOAK_ADDON_URL` : URL de l'instance Keycloak
- `KEYCLOAK_ADDON_ADMIN` : Nom d'utilisateur admin
- `KEYCLOAK_ADDON_PASSWORD` : Mot de passe admin

### Étape 2.3 : Accéder à la console Keycloak

1. Dans l'addon Keycloak, cliquez sur **"Open Keycloak console"** ou accédez à l'URL fournie

2. Connectez-vous avec les credentials admin fournis par Clever Cloud

---

## Partie 3 : Configuration de Keycloak

### Étape 3.1 : Créer un Realm

1. Dans la console Keycloak, cliquez sur la liste déroulante **"master"** (en haut à gauche)
2. Cliquez sur **"Create Realm"**
3. Configurez :
   - **Realm name** : `gameshelf`
   - Cliquez sur **"Create"**

### Étape 3.2 : Créer un Client pour l'API

1. Dans le realm `gameshelf`, allez dans **"Clients"**
2. Cliquez sur **"Create client"**
3. Configuration du client :
   - **Client type** : OpenID Connect
   - **Client ID** : `gameshelf-api`
   - Cliquez sur **"Next"**
4. Capability config :
   - **Client authentication** : ON
   - **Authorization** : OFF
   - **Authentication flow** : Cochez "Service accounts roles" et "Direct access grants"
   - Cliquez sur **"Next"**
5. Login settings :
   - **Root URL** : `https://app-gameshelf-xxxxx.cleverapps.io`
   - **Valid redirect URIs** : `https://app-gameshelf-xxxxx.cleverapps.io/*`
   - Cliquez sur **"Save"**

### Étape 3.3 : Récupérer le secret du client

1. Allez dans **"Clients"** > `gameshelf-api`
2. Onglet **"Credentials"**
3. Copiez le **"Client secret"**

### Étape 3.4 : Créer des rôles

1. Allez dans **"Realm roles"**
2. Créez ces rôles :
   - `admin` : Accès total
   - `staff` : Accès lecture/écriture
   - `customer` : Accès lecture seule

### Étape 3.5 : Créer des utilisateurs

1. Allez dans **"Users"**
2. Cliquez sur **"Create new user"**
3. Créez ces utilisateurs :

**Utilisateur Admin :**
- Username : `admin-user`
- Email : `admin@gameshelf.com`
- First name : `Admin`
- Last name : `GameShelf`
- Email verified : ON
- Cliquez sur **"Create"**
- Onglet **"Credentials"** > **"Set password"** : `admin123`
- Onglet **"Role mapping"** > **"Assign role"** > `admin`

**Utilisateur Staff :**
- Username : `staff-user`
- Email : `staff@gameshelf.com`
- Rôle : `staff`
- Password : `staff123`

**Utilisateur Customer :**
- Username : `customer-user`
- Email : `customer@gameshelf.com`
- Rôle : `customer`
- Password : `customer123`

---

## Partie 4 : Sécuriser l'API GameShelf

### Étape 4.1 : Mettre à jour requirements.txt

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
```

### Étape 4.2 : Mettre à jour app.py

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

# Configuration Keycloak (variables injectées par l'addon)
KEYCLOAK_URL = os.environ.get('KEYCLOAK_URL') or os.environ.get('KEYCLOAK_ADDON_URL', '')
KEYCLOAK_REALM = os.environ.get('KEYCLOAK_REALM', 'gameshelf')
KEYCLOAK_CLIENT_ID = os.environ.get('KEYCLOAK_CLIENT_ID', 'gameshelf-api')
KEYCLOAK_CLIENT_SECRET = os.environ.get('KEYCLOAK_CLIENT_SECRET', '')

# URL pour récupérer la clé publique
KEYCLOAK_CERTS_URL = f"{KEYCLOAK_URL}/realms/{KEYCLOAK_REALM}/protocol/openid-connect/certs"
KEYCLOAK_TOKEN_URL = f"{KEYCLOAK_URL}/realms/{KEYCLOAK_REALM}/protocol/openid-connect/token"

# Cache pour la clé publique
_jwks_cache = None
_jwks_cache_time = 0
JWKS_CACHE_DURATION = 300  # 5 minutes

# Configuration S3 (Cellar)
S3_ENDPOINT = os.environ.get('CELLAR_ADDON_HOST', 'cellar-c2.services.clever-cloud.com')
S3_ACCESS_KEY = os.environ.get('CELLAR_ADDON_KEY_ID')
S3_SECRET_KEY = os.environ.get('CELLAR_ADDON_KEY_SECRET')
S3_BUCKET = os.environ.get('S3_BUCKET', 'gameshelf-images')


def get_jwks():
    """Récupère les clés publiques de Keycloak (avec cache)."""
    global _jwks_cache, _jwks_cache_time

    if _jwks_cache and (time.time() - _jwks_cache_time) < JWKS_CACHE_DURATION:
        return _jwks_cache

    try:
        response = requests.get(KEYCLOAK_CERTS_URL, timeout=10)
        response.raise_for_status()
        _jwks_cache = response.json()
        _jwks_cache_time = time.time()
        logger.info("JWKS cache refreshed")
        return _jwks_cache
    except Exception as e:
        logger.error(f"Failed to fetch JWKS: {e}")
        return _jwks_cache  # Retourner le cache même expiré si erreur


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
        public_key = get_public_key(token)
        decoded = jwt.decode(
            token,
            public_key,
            algorithms=['RS256'],
            audience=KEYCLOAK_CLIENT_ID,
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
    """Crée un client S3 pour Cellar."""
    return boto3.client(
        's3',
        endpoint_url=f'https://{S3_ENDPOINT}',
        aws_access_key_id=S3_ACCESS_KEY,
        aws_secret_access_key=S3_SECRET_KEY,
        config=Config(
            signature_version='s3v4',
            s3={'addressing_style': 'virtual'}
        )
    )


def get_db_connection():
    """Crée une connexion à PostgreSQL."""
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


# Routes publiques (pas d'authentification requise)
@app.route('/')
def home():
    return jsonify({
        "service": f"{APP_NAME} API",
        "version": "5.0.0",
        "environment": ENVIRONMENT,
        "hostname": socket.gethostname(),
        "auth": "Keycloak OIDC",
        "endpoints": {
            "public": ["/", "/health", "/games"],
            "authenticated": ["/games/<id>", "/profile"],
            "staff_only": ["/customers", "/rentals"],
            "admin_only": ["/admin/stats"]
        }
    })


@app.route('/health')
def health():
    db_status = "unknown"
    keycloak_status = "unknown"

    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute('SELECT 1')
        cur.close()
        conn.close()
        db_status = "connected"
    except Exception as e:
        db_status = "error"

    try:
        response = requests.get(f"{KEYCLOAK_URL}/realms/{KEYCLOAK_REALM}", timeout=5)
        keycloak_status = "connected" if response.ok else "error"
    except:
        keycloak_status = "unreachable"

    return jsonify({
        "status": "healthy" if db_status == "connected" else "degraded",
        "database": db_status,
        "keycloak": keycloak_status,
        "hostname": socket.gethostname()
    })


# Route publique - Liste des jeux
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


# Route authentifiée - Détails d'un jeu (avec stock)
@app.route('/games/<int:game_id>')
@require_auth
def get_game(game_id):
    """Détails d'un jeu - nécessite authentification pour voir le stock."""
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


# Route authentifiée - Profil utilisateur
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


# Route staff - Liste des clients
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


# Route staff - Liste des locations
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


# Route admin - Statistiques
@app.route('/admin/stats')
@require_auth
@require_role('admin')
def admin_stats():
    """Statistiques admin - réservé aux administrateurs."""
    logger.info(f"Admin access by {g.username}")
    try:
        conn = get_db_connection()
        cur = conn.cursor()

        # Stats globales
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
    return jsonify({
        "message": "Bravo ! L'API est sécurisée avec Keycloak !",
        "flag": "FLAG{K3ycl04k_0IDC_S3cur3d}",
        "user": g.username,
        "roles": g.roles
    })


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    logger.info(f"Starting GameShelf API on port {port}")
    app.run(host='0.0.0.0', port=port)
```

### Étape 4.3 : Ajouter les variables d'environnement

Dans Clever Cloud, ajoutez ces variables à l'application GameShelf :

| Variable | Valeur |
|----------|--------|
| `KEYCLOAK_URL` | URL de votre addon Keycloak (ou utilisez `KEYCLOAK_ADDON_URL` injectée automatiquement) |
| `KEYCLOAK_REALM` | `gameshelf` |
| `KEYCLOAK_CLIENT_ID` | `gameshelf-api` |
| `KEYCLOAK_CLIENT_SECRET` | Le secret copié à l'étape 3.3 |

### Étape 4.4 : Déployer

```bash
git add .
git commit -m "Add Keycloak authentication"
git push clever main:master
```

---

## Partie 5 : Tester l'authentification

### Étape 5.1 : Tester les routes publiques

```bash
export API_URL="https://app-gameshelf-xxxxx.cleverapps.io"

# Route publique - doit fonctionner sans token
curl "$API_URL/"
curl "$API_URL/health"
curl "$API_URL/games"
```

### Étape 5.2 : Tester sans authentification

```bash
# Route protégée - doit retourner 401
curl "$API_URL/games/1"
curl "$API_URL/customers"
curl "$API_URL/admin/stats"
```

Vous devriez voir :
```json
{"error": "Authorization header missing"}
```

### Étape 5.3 : Obtenir un token

```bash
export KEYCLOAK_URL="<URL_DE_VOTRE_ADDON_KEYCLOAK>"
export REALM="gameshelf"

# Token pour admin-user
TOKEN=$(curl -s -X POST \
  "$KEYCLOAK_URL/realms/$REALM/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=gameshelf-api" \
  -d "client_secret=VOTRE_CLIENT_SECRET" \
  -d "username=admin-user" \
  -d "password=admin123" \
  -d "grant_type=password" | jq -r '.access_token')

echo "Token: $TOKEN"
```

### Étape 5.4 : Tester avec le token admin

```bash
# Profil utilisateur
curl -H "Authorization: Bearer $TOKEN" "$API_URL/profile"

# Route staff
curl -H "Authorization: Bearer $TOKEN" "$API_URL/customers"

# Route admin
curl -H "Authorization: Bearer $TOKEN" "$API_URL/admin/stats"

# Flag
curl -H "Authorization: Bearer $TOKEN" "$API_URL/flag"
```

### Étape 5.5 : Tester avec un token customer

```bash
# Token pour customer-user
CUSTOMER_TOKEN=$(curl -s -X POST \
  "$KEYCLOAK_URL/realms/$REALM/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=gameshelf-api" \
  -d "client_secret=VOTRE_CLIENT_SECRET" \
  -d "username=customer-user" \
  -d "password=customer123" \
  -d "grant_type=password" | jq -r '.access_token')

# Ceci devrait fonctionner
curl -H "Authorization: Bearer $CUSTOMER_TOKEN" "$API_URL/profile"
curl -H "Authorization: Bearer $CUSTOMER_TOKEN" "$API_URL/games/1"

# Ceci devrait retourner 403 (Forbidden)
curl -H "Authorization: Bearer $CUSTOMER_TOKEN" "$API_URL/customers"
curl -H "Authorization: Bearer $CUSTOMER_TOKEN" "$API_URL/admin/stats"
```

---

## Partie 6 : Comprendre les tokens JWT

### Étape 6.1 : Décoder un token

Allez sur [jwt.io](https://jwt.io) et collez votre token.

### Étape 6.2 : Structure du token

Un JWT contient 3 parties séparées par des points :

```
HEADER.PAYLOAD.SIGNATURE
```

**Header :**
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "xxxx"
}
```

**Payload (claims) :**
```json
{
  "exp": 1234567890,
  "iat": 1234567800,
  "sub": "user-uuid",
  "preferred_username": "admin-user",
  "realm_access": {
    "roles": ["admin", "default-roles-gameshelf"]
  }
}
```

### Étape 6.3 : Claims importants

| Claim | Description |
|-------|-------------|
| `exp` | Expiration time (timestamp) |
| `iat` | Issued at (timestamp) |
| `sub` | Subject (user ID) |
| `preferred_username` | Nom d'utilisateur |
| `realm_access.roles` | Rôles de l'utilisateur |
| `aud` | Audience (client ID) |

---

## Partie 7 : Bonnes pratiques de sécurité

### Configuration Keycloak recommandée

| Paramètre | Valeur recommandée | Raison |
|-----------|-------------------|--------|
| Token lifetime | 5-15 minutes | Limiter l'impact d'un vol |
| Refresh token | 30-60 minutes | Permettre le renouvellement |
| Password policy | Complexe | Éviter les mots de passe faibles |
| Brute force | Activé | Bloquer les attaques |

### Sécurisation de l'API

1. **Toujours vérifier la signature** du token
2. **Vérifier l'expiration** (`exp`)
3. **Vérifier l'audience** (`aud`)
4. **Logger les accès** refusés
5. **Ne jamais logger** les tokens complets

### Principe du moindre privilège

```
Customer ──► Lecture catalogue
     │
Staff ──────► + Clients, Locations
     │
Admin ──────► + Stats, Config, Tout
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé l'addon Keycloak sur Clever Cloud
- [ ] Accédé à la console Keycloak
- [ ] Créé le realm `gameshelf`
- [ ] Créé le client `gameshelf-api`
- [ ] Créé les rôles (admin, staff, customer)
- [ ] Créé les utilisateurs de test
- [ ] Modifié l'API pour utiliser l'authentification
- [ ] Testé les routes publiques (sans token)
- [ ] Testé les routes protégées (avec token)
- [ ] Vérifié le RBAC (admin vs staff vs customer)
- [ ] Récupéré le flag avec un token admin

---

## Flag de validation

```
FLAG{K3ycl04k_0IDC_S3cur3d}
```

---

## Résumé des endpoints

| Endpoint | Authentification | Rôle requis |
|----------|-----------------|-------------|
| `/` | Non | - |
| `/health` | Non | - |
| `/games` | Non | - |
| `/games/<id>` | Oui | Tous |
| `/profile` | Oui | Tous |
| `/customers` | Oui | staff |
| `/rentals` | Oui | staff |
| `/admin/stats` | Oui | admin |
| `/flag` | Oui | admin |

---

## Transition vers le TP suivant

*"Notre API est maintenant sécurisée ! Mais regardez notre code... Le secret Keycloak est en variable d'environnement, c'est bien. Mais les credentials de la base de données aussi. Et si on pouvait gérer tous nos secrets de manière plus sécurisée avec rotation automatique ? Dans le prochain TP, nous allons découvrir HashiCorp Vault !"*

---

**Durée estimée : 1h**

**Difficulté : Avancé**
