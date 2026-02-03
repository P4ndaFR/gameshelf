# TP 13 - Télémétrie : Métriques

## Contexte GameShelf

Les logs nous ont aidés à trouver des bugs, mais les clients se plaignent que l'application devient de plus en plus lente au fil du temps. Les logs ne montrent rien d'anormal... Il faut regarder les métriques système !

**Objectif de ce TP :** Utiliser les métriques Clever Cloud pour diagnostiquer un problème de performance.

---

## Prérequis

- TP 12 complété (logging en place)
- Compte Clever Cloud actif
- Application GameShelf déployée

---

## Partie 1 : Comprendre les métriques

### Métriques vs Logs

| Aspect | Logs | Métriques |
|--------|------|-----------|
| Format | Texte structuré | Valeurs numériques |
| Volume | Élevé | Faible |
| Usage | Debug, audit | Monitoring, alerting |
| Question | "Que s'est-il passé ?" | "Comment ça va ?" |
| Rétention | Jours/semaines | Mois/années |

### Types de métriques

```
┌─────────────────────────────────────────────────────────────┐
│                    TYPES DE MÉTRIQUES                       │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  COMPTEUR (Counter)     JAUGE (Gauge)      HISTOGRAMME     │
│  ─────────────────      ─────────────      ───────────     │
│  Toujours croissant     Valeur actuelle    Distribution    │
│  Ex: requêtes totales   Ex: mémoire        Ex: latence     │
│                                                             │
│       /│                    ─┐              │   ▄          │
│      / │                   ┌─┘              │  ▄█▄         │
│     /  │                   │                │ ▄███▄        │
│    /   │                   └─┐              │▄█████▄       │
│   ─────┴─►                 ──┴─►            └───────►      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Métriques clés à surveiller

| Catégorie | Métriques | Seuil d'alerte typique |
|-----------|-----------|------------------------|
| **CPU** | Utilisation % | > 80% |
| **Mémoire** | Usage, disponible | > 85% |
| **Réseau** | Bytes in/out, erreurs | Anomalies |
| **Application** | Requêtes/s, latence, erreurs | Selon SLA |
| **Base de données** | Connexions, temps de requête | Selon limites |

---

## Partie 2 : Les métriques sur Clever Cloud

### Étape 2.1 : Accéder aux métriques

1. Connectez-vous à [console.clever-cloud.com](https://console.clever-cloud.com)

2. Sélectionnez votre application `gameshelf-api`

3. Cliquez sur **"Vue d'ensemble"** (Overview)

4. Dans la section **"Métrique serveur"**, cliquez sur le **logo Grafana**

5. Cliquez sur **"Activer Grafana"**

6. Une fois la page rechargée, cliquez sur **"Ouvrir dans Grafana"**

### Étape 2.2 : Comprendre le dashboard

Clever Cloud fournit automatiquement :

- **CPU** : Utilisation processeur
- **Memory** : RAM utilisée vs disponible
- **Network** : Trafic entrant/sortant
- **Disk** : I/O disque (si applicable)

### Étape 2.3 : Périodes d'observation

Vous pouvez voir les métriques sur différentes périodes :
- **5 minutes** : Temps réel
- **1 heure** : Court terme
- **24 heures** : Journée
- **7 jours** : Tendances

---

## Partie 3 : Ajouter un problème de mémoire

### Le scénario

Un développeur a ajouté un "cache" pour améliorer les performances. Mais ce cache a un bug : il ne libère jamais la mémoire !

### Étape 3.1 : Mettre à jour l'application

```bash
cd ~/gameshelf-clever
nano app.py
```

Ajoutez ce code **après les imports** :

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
import threading

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

# ============================================================
# ATTENTION : Ce "cache" contient un bug intentionnel !
# Il provoque une fuite mémoire progressive.
# Votre mission : identifier le problème via les métriques.
# ============================================================
class LeakyCache:
    """
    Un cache qui ne nettoie jamais ses entrées.
    Bug intentionnel pour le TP.
    """
    def __init__(self):
        self._cache = {}
        self._access_history = []  # BUG: Cette liste grandit indéfiniment !

    def get(self, key):
        # On stocke CHAQUE accès avec un timestamp et des données
        # Bug: on ne limite jamais la taille de l'historique
        self._access_history.append({
            'key': key,
            'timestamp': time.time(),
            'data': 'x' * 1000  # 1KB de données par accès
        })
        return self._cache.get(key)

    def set(self, key, value):
        self._cache[key] = value
        # Encore plus de données stockées...
        self._access_history.append({
            'key': key,
            'timestamp': time.time(),
            'action': 'set',
            'data': 'y' * 2000  # 2KB de données
        })

    def stats(self):
        return {
            'cache_entries': len(self._cache),
            'history_entries': len(self._access_history),
            'estimated_memory_kb': len(self._access_history) * 2  # Approximation
        }

# Instance globale du cache
game_cache = LeakyCache()


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

@app.after_request
def after_request(response):
    if hasattr(g, 'start_time'):
        elapsed = (time.time() - g.start_time) * 1000
        logger.info(f"[{g.request_id}] {request.method} {request.path} - {response.status_code} in {elapsed:.2f}ms")
    return response

@app.route('/')
def home():
    return jsonify({
        "service": f"{APP_NAME} API",
        "version": "4.0.0",
        "environment": ENVIRONMENT,
        "hostname": socket.gethostname(),
        "endpoints": ["/", "/health", "/games", "/games/<id>", "/popular", "/cache-stats", "/flag"]
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
    except Exception as e:
        db_status = "error"
        logger.error(f"Health check: Database FAILED - {str(e)}")

    try:
        s3 = get_s3_client()
        s3.list_buckets()
        s3_status = "connected"
    except Exception as e:
        s3_status = "error"
        logger.error(f"Health check: S3 FAILED - {str(e)}")

    status = "healthy" if db_status == "connected" and s3_status == "connected" else "degraded"

    # Ajouter les stats du cache
    cache_stats = game_cache.stats()

    return jsonify({
        "status": status,
        "database": db_status,
        "storage": s3_status,
        "hostname": socket.gethostname(),
        "cache": cache_stats
    })

@app.route('/games')
def get_games():
    logger.info("Fetching all games")

    # Utilisation du cache (buggé)
    cached = game_cache.get('all_games')
    if cached:
        logger.info("Cache hit for all_games")
        return jsonify(cached)

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

        result = {"count": len(games), "games": games}

        # Mise en cache
        game_cache.set('all_games', result)

        logger.info(f"Successfully fetched {len(games)} games")
        return jsonify(result)
    except Exception as e:
        logger.error(f"Failed to fetch games: {str(e)}")
        return jsonify({"error": str(e)}), 500

@app.route('/games/<int:game_id>')
def get_game(game_id):
    logger.info(f"Fetching game with id={game_id}")

    # Chaque accès à un jeu utilise le cache (et fait fuiter de la mémoire)
    cache_key = f'game_{game_id}'
    cached = game_cache.get(cache_key)
    if cached:
        return jsonify(cached)

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
            game_cache.set(cache_key, dict(game))
            return jsonify(game)
        return jsonify({"error": "Game not found"}), 404
    except Exception as e:
        logger.error(f"Failed to fetch game {game_id}: {str(e)}")
        return jsonify({"error": str(e)}), 500


# ============================================================
# Cet endpoint est particulièrement gourmand en mémoire !
# Il simule une fonctionnalité "jeux populaires" qui accède
# beaucoup au cache.
# ============================================================
@app.route('/popular')
def get_popular_games():
    """
    Retourne les jeux populaires.
    Cette route accède beaucoup au cache et accélère la fuite mémoire.
    """
    logger.info("Fetching popular games")

    # Simuler beaucoup d'accès au cache
    for i in range(50):  # 50 accès au cache à chaque appel !
        game_cache.get(f'popular_query_{i}_{time.time()}')

    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("""
            SELECT g.id, g.name, c.name as category, g.price
            FROM games g
            JOIN categories c ON g.category_id = c.id
            ORDER BY g.stock DESC
            LIMIT 5
        """)
        games = cur.fetchall()
        cur.close()
        conn.close()

        # Mettre en cache avec une clé unique (donc jamais de hit)
        cache_key = f'popular_{time.time()}'
        game_cache.set(cache_key, games)

        return jsonify({
            "count": len(games),
            "games": games,
            "cache_stats": game_cache.stats()
        })
    except Exception as e:
        logger.error(f"Failed to fetch popular games: {str(e)}")
        return jsonify({"error": str(e)}), 500


@app.route('/cache-stats')
def cache_stats():
    """Endpoint pour voir les statistiques du cache."""
    stats = game_cache.stats()
    logger.info(f"Cache stats: {stats}")
    return jsonify({
        "cache": stats,
        "warning": "Memory usage grows with each request!" if stats['history_entries'] > 100 else None
    })


@app.route('/customers')
def get_customers():
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
        logger.error(f"Failed to fetch customers: {str(e)}")
        return jsonify({"error": str(e)}), 500

@app.route('/rentals')
def get_rentals():
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
        logger.error(f"Failed to fetch rentals: {str(e)}")
        return jsonify({"error": str(e)}), 500

@app.route('/images')
def list_images():
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
        return jsonify({'count': len(images), 'images': images})
    except Exception as e:
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
            "message": "Bravo ! L'API avec métriques fonctionne !",
            "flag": "FLAG{M3tr1cs_R3v34l_H1dd3n_Bugs}",
            "database": "connected",
            "storage": s3_status,
            "cache": game_cache.stats()
        })
    except Exception as e:
        logger.error(f"Flag retrieval failed: {str(e)}")
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    logger.info(f"Starting GameShelf API on port {port}")
    app.run(host='0.0.0.0', port=port)
```

### Étape 3.2 : Déployer

```bash
git add .
git commit -m "Add caching (with intentional memory leak)"
git push clever main:master
```

---

## Partie 4 : Observer les métriques de base

### Étape 4.1 : État initial

1. Ouvrez **Grafana** (voir étape 2.1)
2. Notez les valeurs actuelles :
   - Mémoire utilisée : ______ MB
   - CPU : ______ %

### Étape 4.2 : Générer du trafic normal

```bash
export APP_URL="https://app-xxxxxxxx.cleverapps.io"

# Vérifier l'état initial du cache
curl "$APP_URL/cache-stats"

# Quelques requêtes normales
for i in {1..10}; do
    curl -s "$APP_URL/games" > /dev/null
    curl -s "$APP_URL/games/1" > /dev/null
done

# Vérifier le cache après
curl "$APP_URL/cache-stats"
```

### Étape 4.3 : Observer les métriques

Après les requêtes, observez dans Clever Cloud :
- La mémoire a-t-elle augmenté ?
- De combien ?

---

## Partie 5 : Provoquer le problème

### Étape 5.1 : Générer beaucoup de trafic

L'endpoint `/popular` est le plus gourmand. Générons du trafic intensif :

```bash
# Script de charge
for i in {1..100}; do
    curl -s "$APP_URL/popular" > /dev/null
    echo "Request $i - checking memory..."
    if [ $((i % 20)) -eq 0 ]; then
        curl -s "$APP_URL/cache-stats" | python3 -c "import sys, json; d=json.load(sys.stdin); print(f'Cache entries: {d[\"cache\"][\"history_entries\"]}, Est. memory: {d[\"cache\"][\"estimated_memory_kb\"]}KB')"
    fi
done
```

### Étape 5.2 : Observer la croissance

Pendant le script, observez :

1. **Grafana** : La courbe de mémoire monte-t-elle ?
2. **Le endpoint `/cache-stats`** : Le nombre d'entrées augmente-t-il ?
3. **Le endpoint `/health`** : Les stats du cache sont visibles

### Étape 5.3 : Corrélation

Créez un tableau de suivi :

| Requêtes | Cache entries | Mémoire estimée | Mémoire réelle (CC) |
|----------|---------------|-----------------|---------------------|
| 0 | | | |
| 20 | | | |
| 40 | | | |
| 60 | | | |
| 80 | | | |
| 100 | | | |

---

## Partie 6 : Diagnostic du problème

### Mission : Identifier le bug via les métriques

En observant les métriques, vous devez identifier :

1. **Quel type de ressource est affecté ?** (CPU, mémoire, réseau...)
2. **Quelle est la tendance ?** (croissante, stable, pic...)
3. **Quelle corrélation avec les requêtes ?**

### Questions d'investigation

1. La mémoire redescend-elle après l'arrêt des requêtes ?
2. Y a-t-il une limite au-delà de laquelle l'application crashe ?
3. Quel endpoint consomme le plus de mémoire ?

### Réponses attendues

Le bug identifié :

1. **Ressource affectée** : Mémoire (RAM)
2. **Tendance** : Croissance linéaire continue (jamais de libération)
3. **Cause** : La classe `LeakyCache` stocke un historique illimité

---

## Partie 7 : Alertes et notifications

### Étape 7.1 : Configurer des alertes sur Clever Cloud

1. Allez dans votre application
2. Cliquez sur **"Notifications"**
3. Ajoutez une notification :
   - **Type** : Webhook ou Email
   - **Événement** : Scaling, Deployment, Error

### Étape 7.2 : Seuils recommandés

| Métrique | Seuil Warning | Seuil Critical |
|----------|---------------|----------------|
| Mémoire | > 70% | > 85% |
| CPU | > 70% | > 90% |
| Temps de réponse | > 1s | > 5s |
| Taux d'erreur | > 1% | > 5% |

### Étape 7.3 : Webhook personnalisé (optionnel)

Vous pouvez créer un webhook pour recevoir les alertes :

```python
# Exemple de récepteur de webhook
from flask import Flask, request
import json

app = Flask(__name__)

@app.route('/webhook', methods=['POST'])
def receive_alert():
    data = request.json
    print(f"Alert received: {json.dumps(data, indent=2)}")
    # Envoyer à Slack, Discord, etc.
    return "OK", 200
```

---

## Partie 8 : Corriger le bug (optionnel)

### Solution : Cache avec limite

```python
class BoundedCache:
    """Cache avec limite de taille."""

    def __init__(self, max_entries=1000):
        self._cache = {}
        self._max_entries = max_entries
        self._access_order = []

    def get(self, key):
        if key in self._cache:
            # LRU: déplacer en fin de liste
            self._access_order.remove(key)
            self._access_order.append(key)
            return self._cache[key]
        return None

    def set(self, key, value):
        if key not in self._cache:
            # Si plein, supprimer le plus ancien
            if len(self._cache) >= self._max_entries:
                oldest = self._access_order.pop(0)
                del self._cache[oldest]
            self._access_order.append(key)
        self._cache[key] = value

    def stats(self):
        return {
            'entries': len(self._cache),
            'max_entries': self._max_entries,
            'usage_percent': (len(self._cache) / self._max_entries) * 100
        }
```

---

## Partie 9 : Bonnes pratiques

### RED Method

Pour les services, surveillez :
- **R**ate : Requêtes par seconde
- **E**rrors : Taux d'erreur
- **D**uration : Latence

### USE Method

Pour les ressources, surveillez :
- **U**tilization : % d'utilisation
- **S**aturation : File d'attente
- **E**rrors : Erreurs

### Golden Signals (Google SRE)

1. **Latency** : Temps de réponse
2. **Traffic** : Requêtes par seconde
3. **Errors** : Taux d'erreur
4. **Saturation** : Charge système

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Accédé à Grafana via Clever Cloud
- [ ] Observé les métriques de base (CPU, mémoire)
- [ ] Déployé l'application avec le cache buggé
- [ ] Généré du trafic pour provoquer la fuite mémoire
- [ ] Observé la croissance de la mémoire dans les métriques
- [ ] Corrélé les métriques avec le nombre de requêtes
- [ ] Identifié le bug (fuite mémoire dans le cache)
- [ ] Compris comment configurer des alertes
- [ ] Récupéré le flag

---

## Flag de validation

```
FLAG{M3tr1cs_R3v34l_H1dd3n_Bugs}
```

---

## Questions de réflexion

1. Comment auriez-vous détecté ce bug uniquement avec des logs ?
2. Quelle serait la conséquence en production si ce bug n'était pas détecté ?
3. Comment automatiser la détection de fuites mémoire ?

---

## Transition vers le TP suivant

*"Les métriques nous ont permis de détecter une fuite mémoire invisible dans les logs ! Mais notre API est accessible à tous sans authentification. N'importe qui peut accéder aux données clients. Dans le prochain TP, nous allons sécuriser l'API avec Keycloak !"*

---

**Durée estimée : 1h15**

**Difficulté : Intermédiaire**
