# TP 10 - Base de Données Managée avec Clever Cloud

## Contexte GameShelf

Notre API tourne sur Clever Cloud, mais les données sont en mémoire et disparaissent à chaque redémarrage. Nous avons besoin d'une vraie base de données pour stocker les jeux, clients et locations.

**Objectif de ce TP :** Créer une base PostgreSQL managée et connecter l'API.

---

## Prérequis

- TP 09 complété (application sur Clever Cloud)
- Client PostgreSQL (`psql`) installé

---

## Partie 1 : Création de l'addon PostgreSQL

### Étape 1.1 : Accéder à la console Clever Cloud

1. Connectez-vous à [console.clever-cloud.com](https://console.clever-cloud.com)

2. Allez dans votre application `gameshelf-api`

### Étape 1.2 : Ajouter un addon PostgreSQL

1. Cliquez sur **"Service dependencies"** ou **"Add-ons"**

2. Cliquez sur **"Link an add-on"** puis **"New add-on"**

3. Sélectionnez **"PostgreSQL"**

4. Choisissez le plan **"DEV"** (gratuit) :
   - 256 MB RAM
   - 10 connexions max
   - Parfait pour le développement

5. **Nom** : `gameshelf-db`

6. **Région** : Paris (même région que l'app)

7. Cliquez sur **"Create"**

### Étape 1.3 : Lier l'addon à l'application

1. L'addon est automatiquement lié à votre application

2. Des variables d'environnement sont injectées automatiquement :
   - `POSTGRESQL_ADDON_HOST`
   - `POSTGRESQL_ADDON_PORT`
   - `POSTGRESQL_ADDON_DB`
   - `POSTGRESQL_ADDON_USER`
   - `POSTGRESQL_ADDON_PASSWORD`
   - `POSTGRESQL_ADDON_URI`

### Étape 1.4 : Vérifier les variables

Dans l'application :
1. Allez dans **"Environment variables"**
2. Vous verrez les variables PostgreSQL injectées

---

## Partie 2 : Connexion à la base de données

### Étape 2.1 : Récupérer les informations de connexion

Dans l'addon PostgreSQL :
1. Cliquez sur `gameshelf-db`
2. Allez dans **"Information"** ou **"Environment variables"**
3. Notez :
   - Host
   - Port
   - Database
   - User
   - Password

### Étape 2.2 : Installer psql

**Linux/WSL :**
```bash
sudo apt update
sudo apt install -y postgresql-client
```

**macOS :**
```bash
brew install postgresql
```

### Étape 2.3 : Se connecter avec psql

```bash
psql "postgresql://USER:PASSWORD@HOST:PORT/DATABASE"
```

Ou avec les paramètres séparés :
```bash
psql -h HOST -p PORT -U USER -d DATABASE
```

---

## Partie 3 : Création du schéma

### Étape 3.1 : Créer les tables

```sql
-- Table des catégories
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table des jeux
CREATE TABLE games (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    category_id INTEGER REFERENCES categories(id),
    min_players INTEGER CHECK (min_players > 0),
    max_players INTEGER CHECK (max_players >= min_players),
    duration_minutes INTEGER,
    price DECIMAL(10, 2),
    stock INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table des clients
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone VARCHAR(20),
    loyalty_points INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Table des locations
CREATE TABLE rentals (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id) ON DELETE CASCADE,
    game_id INTEGER REFERENCES games(id) ON DELETE CASCADE,
    rental_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    due_date TIMESTAMP,
    return_date TIMESTAMP,
    status VARCHAR(50) DEFAULT 'active' CHECK (status IN ('active', 'returned', 'overdue')),
    deposit DECIMAL(10, 2)
);

-- Table de configuration (pour le flag)
CREATE TABLE app_config (
    key VARCHAR(100) PRIMARY KEY,
    value TEXT NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Index pour les performances
CREATE INDEX idx_games_category ON games(category_id);
CREATE INDEX idx_rentals_customer ON rentals(customer_id);
CREATE INDEX idx_rentals_status ON rentals(status);
```

### Étape 3.2 : Insérer les données de test

```sql
-- Catégories
INSERT INTO categories (name, description) VALUES
('Stratégie', 'Jeux de réflexion et de planification à long terme'),
('Coopératif', 'Jeux où les joueurs travaillent ensemble'),
('Party Game', 'Jeux d''ambiance pour animer les soirées'),
('Famille', 'Jeux accessibles à tous les âges'),
('Expert', 'Jeux complexes pour joueurs expérimentés');

-- Jeux
INSERT INTO games (name, category_id, min_players, max_players, duration_minutes, price, stock) VALUES
('Catan', 1, 3, 4, 90, 39.99, 5),
('Pandemic', 2, 2, 4, 60, 44.99, 3),
('Codenames', 3, 4, 8, 30, 24.99, 8),
('Ticket to Ride', 4, 2, 5, 45, 42.99, 4),
('7 Wonders', 1, 3, 7, 45, 49.99, 2),
('Dixit', 3, 3, 6, 30, 34.99, 6),
('Terraforming Mars', 5, 1, 5, 120, 69.99, 2),
('Azul', 4, 2, 4, 40, 39.99, 4),
('Wingspan', 5, 1, 5, 70, 54.99, 3),
('The Crew', 2, 2, 5, 20, 14.99, 5);

-- Clients
INSERT INTO customers (email, first_name, last_name, phone, loyalty_points) VALUES
('alice.dupont@email.com', 'Alice', 'Dupont', '0612345678', 150),
('bob.martin@email.com', 'Bob', 'Martin', '0623456789', 75),
('charlie.bernard@email.com', 'Charlie', 'Bernard', '0634567890', 200),
('diana.petit@email.com', 'Diana', 'Petit', '0645678901', 50);

-- Locations
INSERT INTO rentals (customer_id, game_id, due_date, status, deposit) VALUES
(1, 2, CURRENT_TIMESTAMP + INTERVAL '7 days', 'active', 20.00),
(2, 4, CURRENT_TIMESTAMP + INTERVAL '7 days', 'active', 20.00),
(3, 1, CURRENT_TIMESTAMP - INTERVAL '1 day', 'overdue', 20.00);

-- Flag de validation
INSERT INTO app_config (key, value) VALUES
('validation_flag', 'FLAG{P0stgr3SQL_Cl3v3r_Cl0ud_C0nn3ct3d}'),
('app_version', '2.0.0'),
('db_schema_version', '1');
```

### Étape 3.3 : Vérifier les données

```sql
-- Compter les enregistrements
SELECT 'categories' as table_name, COUNT(*) FROM categories
UNION ALL
SELECT 'games', COUNT(*) FROM games
UNION ALL
SELECT 'customers', COUNT(*) FROM customers
UNION ALL
SELECT 'rentals', COUNT(*) FROM rentals;

-- Lister les jeux avec leur catégorie
SELECT g.name, c.name as category, g.price, g.stock
FROM games g
JOIN categories c ON g.category_id = c.id
ORDER BY g.name;

-- Récupérer le flag
SELECT value FROM app_config WHERE key = 'validation_flag';
```

---

## Partie 4 : Mise à jour de l'application

### Étape 4.1 : Modifier app.py

```bash
cd ~/gameshelf-clever
nano app.py
```

```python
from flask import Flask, jsonify
import os
import socket
import psycopg2
from psycopg2.extras import RealDictCursor

app = Flask(__name__)

# Configuration
APP_NAME = os.environ.get('APP_NAME', 'GameShelf')
ENVIRONMENT = os.environ.get('ENVIRONMENT', 'development')

# Connexion à la base de données
def get_db_connection():
    """Crée une connexion à PostgreSQL depuis les variables Clever Cloud."""
    return psycopg2.connect(
        host=os.environ.get('POSTGRESQL_ADDON_HOST'),
        port=os.environ.get('POSTGRESQL_ADDON_PORT'),
        database=os.environ.get('POSTGRESQL_ADDON_DB'),
        user=os.environ.get('POSTGRESQL_ADDON_USER'),
        password=os.environ.get('POSTGRESQL_ADDON_PASSWORD'),
        cursor_factory=RealDictCursor
    )

@app.route('/')
def home():
    return jsonify({
        "service": f"{APP_NAME} API",
        "version": "2.0.0",
        "environment": ENVIRONMENT,
        "hostname": socket.gethostname(),
        "database": "PostgreSQL (Clever Cloud)",
        "endpoints": ["/", "/health", "/games", "/games/<id>", "/customers", "/rentals", "/flag"]
    })

@app.route('/health')
def health():
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute('SELECT 1')
        cur.close()
        conn.close()
        db_status = "connected"
    except Exception as e:
        db_status = f"error: {str(e)}"

    return jsonify({
        "status": "healthy" if db_status == "connected" else "degraded",
        "database": db_status,
        "hostname": socket.gethostname()
    })

@app.route('/games')
def get_games():
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
        return jsonify({"count": len(games), "games": games})
    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route('/games/<int:game_id>')
def get_game(game_id):
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
        return jsonify({"error": str(e)}), 500

@app.route('/flag')
def get_flag():
    try:
        conn = get_db_connection()
        cur = conn.cursor()
        cur.execute("SELECT value FROM app_config WHERE key = 'validation_flag'")
        result = cur.fetchone()
        cur.close()
        conn.close()
        return jsonify({
            "message": "Bravo ! L'API est connectée à PostgreSQL !",
            "flag": result['value'] if result else "Flag not found",
            "source": "database"
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
```

### Étape 4.2 : Mettre à jour requirements.txt

```bash
nano requirements.txt
```

```
flask==3.0.0
gunicorn==21.2.0
psycopg2-binary==2.9.9
```

### Étape 4.3 : Déployer

```bash
git add .
git commit -m "Add PostgreSQL connection"
git push clever main
```

---

## Partie 5 : Test de l'application

### Étape 5.1 : Tester les endpoints

```bash
export APP_URL="https://app-xxxxxxxx.cleverapps.io"

# Health check avec statut DB
curl $APP_URL/health

# Liste des jeux (depuis PostgreSQL)
curl $APP_URL/games

# Un jeu spécifique
curl $APP_URL/games/1

# Liste des clients
curl $APP_URL/customers

# Locations en cours
curl $APP_URL/rentals

# Flag
curl $APP_URL/flag
```

---

## Partie 6 : Backup et Restore

### Étape 6.1 : Backups automatiques

Clever Cloud effectue des backups automatiques quotidiens. Consultez-les dans :
**Add-on > gameshelf-db > Backups**

### Étape 6.2 : Backup manuel

```bash
# Export de la base
pg_dump "postgresql://USER:PASSWORD@HOST:PORT/DATABASE" > backup.sql

# Ou format custom (plus performant)
pg_dump -Fc "postgresql://USER:PASSWORD@HOST:PORT/DATABASE" > backup.dump
```

### Étape 6.3 : Restore

```bash
# Depuis un fichier SQL
psql "postgresql://USER:PASSWORD@HOST:PORT/DATABASE" < backup.sql

# Depuis un dump custom
pg_restore -d "postgresql://USER:PASSWORD@HOST:PORT/DATABASE" backup.dump
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé un addon PostgreSQL sur Clever Cloud
- [ ] Connecté avec psql
- [ ] Créé le schéma (tables)
- [ ] Inséré les données de test
- [ ] Modifié l'application pour utiliser PostgreSQL
- [ ] Déployé la nouvelle version
- [ ] Testé les endpoints avec données réelles
- [ ] Récupéré le flag via `/flag`

---

## Flag de validation

```bash
curl $APP_URL/flag
```

```
FLAG{P0stgr3SQL_Cl3v3r_Cl0ud_C0nn3ct3d}
```

---

## Transition vers le TP suivant

*"Notre API stocke maintenant les données de manière persistante ! Mais GameShelf doit aussi stocker les images des boîtes de jeux. Dans le prochain TP, nous allons configurer un stockage objet S3 !"*

---

## Commandes SQL utiles

```sql
-- Lister les tables
\dt

-- Décrire une table
\d games

-- Statistiques
SELECT COUNT(*) FROM games;
SELECT AVG(price) FROM games;

-- Jeux en rupture
SELECT name, stock FROM games WHERE stock = 0;

-- Clients les plus fidèles
SELECT first_name, last_name, loyalty_points
FROM customers
ORDER BY loyalty_points DESC;
```

---

**Durée estimée : 1 heure**

**Difficulté : Intermédiaire**
