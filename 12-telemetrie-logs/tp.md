# TP 12 - Télémétrie : Logs

## Contexte GameShelf

Notre API fonctionne avec PostgreSQL et S3, mais un client signale des problèmes intermittents : parfois les requêtes sont lentes, parfois elles échouent. Sans visibilité sur ce qui se passe en production, impossible de diagnostiquer !

**Objectif de ce TP :** Utiliser les logs Clever Cloud pour diagnostiquer un problème réel dans l'application.

---

## Prérequis

- TP 11 complété (API avec PostgreSQL et S3)
- Compte Clever Cloud actif
- Application GameShelf déployée

---

## Partie 1 : L'importance des logs

### Pourquoi les logs ?

```
┌─────────────────────────────────────────────────────────────┐
│                    OBSERVABILITÉ                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐               │
│   │  LOGS   │    │MÉTRIQUES│    │ TRACES  │               │
│   │         │    │         │    │         │               │
│   │ Quoi ?  │    │ Combien?│    │   Où ?  │               │
│   │ Quand ? │    │ Tendance│    │ Chemin  │               │
│   └─────────┘    └─────────┘    └─────────┘               │
│                                                             │
│   Les 3 piliers de l'observabilité                         │
└─────────────────────────────────────────────────────────────┘
```

### Types de logs

| Type | Contenu | Exemple |
|------|---------|---------|
| **Access logs** | Requêtes HTTP entrantes | `GET /games 200 45ms` |
| **Application logs** | Messages de l'application | `INFO: Game created: Catan` |
| **Error logs** | Erreurs et exceptions | `ERROR: Database connection failed` |
| **Audit logs** | Actions sensibles | `User admin deleted game 42` |

### Niveaux de log

```
CRITICAL  ─────  Système en panne
ERROR     ─────  Erreur bloquante
WARNING   ─────  Problème potentiel
INFO      ─────  Information normale
DEBUG     ─────  Détails techniques
```

---

## Partie 2 : Les logs sur Clever Cloud

### Étape 2.1 : Accéder aux logs

1. Connectez-vous à [console.clever-cloud.com](https://console.clever-cloud.com)

2. Sélectionnez votre application `gameshelf-api`

3. Cliquez sur **"Logs"** dans le menu de gauche

### Étape 2.2 : Comprendre l'interface

L'interface de logs Clever Cloud propose :

- **Logs en temps réel** : Flux continu des logs
- **Filtrage par type** : stdout, stderr, access
- **Recherche** : Filtrer par mot-clé
- **Téléchargement** : Exporter les logs

### Étape 2.3 : Utiliser le CLI

```bash
# Installer le CLI Clever Cloud (si pas déjà fait)
npm install -g clever-tools

# Se connecter
clever login

# Lier l'application
clever link <app_id>

# Voir les logs en temps réel (mode follow par défaut)
clever logs

# Voir les logs de la dernière heure
clever logs --since 1h

# Voir les logs des 30 dernières minutes
clever logs --since 30m

# Voir les logs entre deux dates
clever logs --since "2026-02-03T10:00:00" --until "2026-02-03T12:00:00"
```

---

## Partie 3 : Structurer les logs applicatifs

### Étape 3.1 : Configurer le logging Python

Modifiez `app.py` pour ajouter un logging structuré :

```bash
cd ~/gameshelf-clever
nano app.py
```

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

# Configuration S3 (Cellar)
S3_ENDPOINT = os.environ.get('CELLAR_ADDON_HOST', 'cellar-c2.services.clever-cloud.com')
S3_ACCESS_KEY = os.environ.get('CELLAR_ADDON_KEY_ID')
S3_SECRET_KEY = os.environ.get('CELLAR_ADDON_KEY_SECRET')
S3_BUCKET = os.environ.get('S3_BUCKET', 'gameshelf-images')

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

# Middleware pour mesurer le temps de réponse
@app.before_request
def before_request():
    g.start_time = time.time()
    g.request_id = f"{time.time()}-{random.randint(1000, 9999)}"
    logger.info(f"[{g.request_id}] Request started: {request.method} {request.path}")

@app.after_request
def after_request(response):
    if hasattr(g, 'start_time'):
        elapsed = (time.time() - g.start_time) * 1000
        logger.info(f"[{g.request_id}] Request completed: {response.status_code} in {elapsed:.2f}ms")
    return response

@app.route('/')
def home():
    logger.info("Home endpoint accessed")
    return jsonify({
        "service": f"{APP_NAME} API",
        "version": "3.0.0",
        "environment": ENVIRONMENT,
        "hostname": socket.gethostname(),
        "endpoints": ["/", "/health", "/games", "/games/<id>", "/customers", "/rentals", "/search", "/flag"]
    })

@app.route('/health')
def health():
    db_status = "unknown"
    s3_status = "unknown"

    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute('SELECT 1')
        cur.close()
        conn.close()
        db_status = "connected"
        logger.info("Health check: Database OK")
    except Exception as e:
        db_status = f"error"
        logger.error(f"Health check: Database FAILED - {str(e)}")

    try:
        s3 = get_s3_client()
        s3.list_buckets()
        s3_status = "connected"
        logger.info("Health check: S3 OK")
    except Exception as e:
        s3_status = "error"
        logger.error(f"Health check: S3 FAILED - {str(e)}")

    status = "healthy" if db_status == "connected" and s3_status == "connected" else "degraded"

    return jsonify({
        "status": status,
        "database": db_status,
        "storage": s3_status,
        "hostname": socket.gethostname()
    })

@app.route('/games')
def get_games():
    logger.info("Fetching all games")
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("""
            SELECT g.id, g.name, c.name as category,
                   g.min_players, g.max_players, g.duration_minutes,
                   g.price, g.stock
            FROM games g
            JOIN categories c ON g.category_id = c.id
            ORDER BY g.name
        """)
        games = cur.fetchall()
        cur.close()
        conn.close()
        logger.info(f"Successfully fetched {len(games)} games")
        return jsonify({"count": len(games), "games": games})
    except Exception as e:
        logger.error(f"Failed to fetch games: {str(e)}")
        return jsonify({"error": str(e)}), 500

@app.route('/games/<int:game_id>')
def get_game(game_id):
    logger.info(f"Fetching game with id={game_id}")
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
            logger.info(f"Found game: {game['name']}")
            return jsonify(game)
        logger.warning(f"Game not found: id={game_id}")
        return jsonify({"error": "Game not found"}), 404
    except Exception as e:
        logger.error(f"Failed to fetch game {game_id}: {str(e)}")
        return jsonify({"error": str(e)}), 500

@app.route('/customers')
def get_customers():
    logger.info("Fetching all customers")
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
        logger.info(f"Successfully fetched {len(customers)} customers")
        return jsonify({"count": len(customers), "customers": customers})
    except Exception as e:
        logger.error(f"Failed to fetch customers: {str(e)}")
        return jsonify({"error": str(e)}), 500

@app.route('/rentals')
def get_rentals():
    logger.info("Fetching all rentals")
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
        logger.info(f"Successfully fetched {len(rentals)} rentals")
        return jsonify({"count": len(rentals), "rentals": rentals})
    except Exception as e:
        logger.error(f"Failed to fetch rentals: {str(e)}")
        return jsonify({"error": str(e)}), 500


# ============================================================
# ATTENTION : Cette route contient un bug intentionnel !
# Votre mission : trouver le problème en analysant les logs
# ============================================================
@app.route('/search')
def search_games():
    """
    Recherche de jeux avec un bug caché.
    Le bug se manifeste de manière intermittente.
    """
    query = request.args.get('q', '')
    logger.info(f"Search request received: q='{query}'")

    if not query:
        logger.warning("Empty search query received")
        return jsonify({"error": "Query parameter 'q' is required"}), 400

    try:
        conn = get_db_connection()
        cur = conn.cursor()

        # Bug intentionnel : parfois la connexion n'est pas fermée correctement
        # Cela provoque une fuite de connexions

        # Simulation d'un traitement lourd
        import time

        # Le bug : 30% du temps, on "oublie" de fermer la connexion
        # et on fait un sleep qui bloque
        if random.random() < 0.3:
            logger.debug("Taking slow path...")
            time.sleep(2)  # Simule un traitement lent
            # BUG: on oublie de fermer la connexion ici !
            cur.execute("""
                SELECT g.id, g.name, c.name as category, g.price
                FROM games g
                JOIN categories c ON g.category_id = c.id
                WHERE LOWER(g.name) LIKE LOWER(%s)
            """, (f'%{query}%',))
            games = cur.fetchall()
            # Pas de cur.close() ni conn.close() !
            logger.warning(f"SLOW SEARCH completed for '{query}' - found {len(games)} results")
            return jsonify({"count": len(games), "games": games, "search_time": "slow"})

        # Chemin normal
        cur.execute("""
            SELECT g.id, g.name, c.name as category, g.price
            FROM games g
            JOIN categories c ON g.category_id = c.id
            WHERE LOWER(g.name) LIKE LOWER(%s)
        """, (f'%{query}%',))
        games = cur.fetchall()
        cur.close()
        conn.close()

        logger.info(f"Search completed for '{query}' - found {len(games)} results")
        return jsonify({"count": len(games), "games": games, "search_time": "fast"})

    except Exception as e:
        logger.error(f"Search failed for '{query}': {str(e)}")
        return jsonify({"error": str(e)}), 500


@app.route('/images')
def list_images():
    """Liste les images dans le bucket S3."""
    logger.info("Listing images from S3")
    try:
        s3 = get_s3_client()
        response = s3.list_objects_v2(Bucket=S3_BUCKET, Prefix='games/')
        images = []
        for obj in response.get('Contents', []):
            images.append({
                'key': obj['Key'],
                'size': obj['Size'],
                'url': f"https://{S3_BUCKET}.{S3_ENDPOINT}/{obj['Key']}"
            })
        logger.info(f"Found {len(images)} images")
        return jsonify({'count': len(images), 'images': images})
    except Exception as e:
        logger.error(f"Failed to list images: {str(e)}")
        return jsonify({'error': str(e)}), 500

@app.route('/flag')
def get_flag():
    try:
        s3 = get_s3_client()
        s3.list_buckets()
        s3_status = "connected"
    except:
        s3_status = "error"

    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("SELECT value FROM app_config WHERE key = 'validation_flag'")
        result = cur.fetchone()
        cur.close()
        conn.close()

        return jsonify({
            "message": "Bravo ! L'API avec logging fonctionne !",
            "flag": "FLAG{L0gs_4r3_Y0ur_B3st_Fr13nd}",
            "database": "connected",
            "storage": s3_status
        })
    except Exception as e:
        logger.error(f"Flag retrieval failed: {str(e)}")
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    logger.info(f"Starting GameShelf API on port {port}")
    app.run(host='0.0.0.0', port=port)
```

### Étape 3.2 : Déployer l'application

```bash
git add .
git commit -m "Add structured logging"
git push clever main:master
```

---

## Partie 4 : Générer du trafic et observer les logs

### Étape 4.1 : Ouvrir les logs en temps réel

Dans un terminal :
```bash
clever logs
```

### Étape 4.2 : Générer du trafic

Dans un autre terminal :
```bash
export APP_URL="https://app-xxxxxxxx.cleverapps.io"

# Requêtes normales
for i in {1..10}; do
    curl -s "$APP_URL/games" > /dev/null
    echo "Request $i sent"
done

# Requêtes de recherche (où se cache le bug !)
for i in {1..20}; do
    curl -s "$APP_URL/search?q=catan" > /dev/null
    curl -s "$APP_URL/search?q=pandemic" > /dev/null
    echo "Search $i sent"
done
```

### Étape 4.3 : Observer les patterns dans les logs

Regardez attentivement les logs. Vous devriez voir :
- Des requêtes rapides et des requêtes lentes
- Des messages `WARNING` avec "SLOW SEARCH"
- Peut-être des erreurs de connexion à la base

---

## Partie 5 : Investigation du problème

### Mission : Trouver le bug !

En analysant les logs, vous devez identifier :

1. **Quel endpoint pose problème ?**
2. **Quel est le symptôme visible dans les logs ?**
3. **Quelle est la cause probable ?**

### Indices dans les logs

Recherchez ces patterns (utilisez `--since` pour récupérer les logs récents) :
```bash
# Filtrer les warnings
clever logs --since 1h | grep "WARNING"

# Filtrer les erreurs
clever logs --since 1h | grep "ERROR"

# Filtrer les recherches lentes
clever logs --since 1h | grep "SLOW"

# Voir les temps de réponse
clever logs --since 1h | grep "completed"
```

### Questions d'investigation

Répondez à ces questions en vous basant sur les logs :

1. Quel pourcentage des requêtes `/search` sont lentes ?
2. Quelle est la différence de temps entre une requête rapide et une lente ?
3. Après plusieurs requêtes lentes, voyez-vous des erreurs apparaître ?

---

## Partie 6 : Diagnostic du bug

### Le problème identifié

En analysant les logs, vous devriez avoir trouvé que :

1. **L'endpoint `/search`** a des performances incohérentes
2. **30% des requêtes** prennent environ 2 secondes (voir "SLOW SEARCH")
3. **Les connexions DB** ne sont pas toujours fermées

### Explication technique

Le bug dans le code :
```python
if random.random() < 0.3:
    time.sleep(2)
    # ... exécute la requête ...
    # BUG: Pas de cur.close() ni conn.close() !
```

**Conséquences :**
- Fuite de connexions PostgreSQL
- Requêtes lentes qui bloquent les workers
- Potentielle saturation du pool de connexions

### Vérification

Pour confirmer la fuite de connexions, vous pouvez vérifier dans PostgreSQL :

```sql
-- Nombre de connexions actives
SELECT count(*) FROM pg_stat_activity
WHERE datname = 'votre_database';
```

---

## Partie 7 : Bonnes pratiques de logging

### Format de log structuré

```python
# Mauvais
print(f"Erreur: {e}")

# Bon
logger.error(f"Database connection failed", extra={
    'error_type': type(e).__name__,
    'error_message': str(e),
    'endpoint': '/games',
    'request_id': g.request_id
})
```

### Niveaux de log appropriés

| Niveau | Quand l'utiliser |
|--------|------------------|
| `DEBUG` | Détails techniques pour développeurs |
| `INFO` | Événements normaux (requête reçue, action terminée) |
| `WARNING` | Situation anormale mais gérée |
| `ERROR` | Erreur qui empêche une action |
| `CRITICAL` | Système inutilisable |

### Ce qu'il faut logger

- **Toujours** : Erreurs, exceptions, authentifications
- **Souvent** : Début/fin d'opérations, temps de réponse
- **Parfois** : Données métier importantes
- **Jamais** : Mots de passe, tokens, données personnelles sensibles

---

## Partie 8 : Corriger le bug (optionnel)

Si vous voulez corriger le bug, modifiez la fonction `search_games()` :

```python
@app.route('/search')
def search_games():
    query = request.args.get('q', '')
    logger.info(f"Search request received: q='{query}'")

    if not query:
        return jsonify({"error": "Query parameter 'q' is required"}), 400

    conn = None
    cur = None
    try:
        conn = get_db_connection()
        cur = conn.cursor()

        cur.execute("""
            SELECT g.id, g.name, c.name as category, g.price
            FROM games g
            JOIN categories c ON g.category_id = c.id
            WHERE LOWER(g.name) LIKE LOWER(%s)
        """, (f'%{query}%',))
        games = cur.fetchall()

        logger.info(f"Search completed for '{query}' - found {len(games)} results")
        return jsonify({"count": len(games), "games": games})

    except Exception as e:
        logger.error(f"Search failed: {str(e)}")
        return jsonify({"error": str(e)}), 500
    finally:
        # Toujours fermer les connexions !
        if cur:
            cur.close()
        if conn:
            conn.close()
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Accédé aux logs via la console Clever Cloud
- [ ] Utilisé le CLI `clever logs`
- [ ] Ajouté le logging structuré à l'application
- [ ] Généré du trafic pour observer les logs
- [ ] Identifié l'endpoint problématique (`/search`)
- [ ] Trouvé le symptôme (requêtes lentes à 30%)
- [ ] Compris la cause (fuite de connexions)
- [ ] Récupéré le flag

---

## Flag de validation

```
FLAG{L0gs_4r3_Y0ur_B3st_Fr13nd}
```

---

## Questions de réflexion

1. Comment auriez-vous pu détecter ce bug sans logs structurés ?
2. Quels autres indicateurs auraient pu alerter sur ce problème ?
3. Comment automatiser la détection de ce type de bug ?

---

## Transition vers le TP suivant

*"Les logs nous ont permis de trouver un bug de performance ! Mais pour avoir une vue d'ensemble de la santé de notre application, nous avons besoin de métriques. Dans le prochain TP, nous allons explorer les métriques et créer des alertes !"*

---

**Durée estimée : 1h15**

**Difficulté : Intermédiaire**
