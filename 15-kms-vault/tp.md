# TP 15 - Gestion des Secrets avec HashiCorp Vault

## Contexte GameShelf

Notre API est sécurisée avec Keycloak, mais nos secrets (credentials DB, clés S3, secrets Keycloak) sont stockés en variables d'environnement. Si quelqu'un accède à la console Clever Cloud ou à notre CI/CD, il voit tous les secrets en clair !

**Objectif de ce TP :** Configurer HashiCorp Vault pour gérer nos secrets de manière sécurisée, avec rotation automatique.

---

## Prérequis

- TP 14 complété
- Docker installé localement
- docker-compose installé

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

## Partie 2 : Installation de Vault avec Docker

### Étape 2.1 : Créer le projet

```bash
mkdir -p ~/vault-demo
cd ~/vault-demo
```

### Étape 2.2 : Créer le fichier docker-compose.yml

```bash
nano docker-compose.yml
```

```yaml
version: '3.8'

services:
  vault:
    image: hashicorp/vault:1.15
    container_name: vault
    ports:
      - "8200:8200"
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: "gameshelf-root-token"
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    cap_add:
      - IPC_LOCK
    volumes:
      - vault-data:/vault/data
    command: server -dev

  vault-init:
    image: hashicorp/vault:1.15
    container_name: vault-init
    depends_on:
      - vault
    environment:
      VAULT_ADDR: "http://vault:8200"
      VAULT_TOKEN: "gameshelf-root-token"
    volumes:
      - ./init-vault.sh:/init-vault.sh
    entrypoint: ["/bin/sh", "-c", "sleep 5 && /init-vault.sh"]

volumes:
  vault-data:
```

### Étape 2.3 : Créer le script d'initialisation

```bash
nano init-vault.sh
chmod +x init-vault.sh
```

```bash
#!/bin/sh

echo "=== Initialisation de Vault pour GameShelf ==="

# Attendre que Vault soit prêt
until vault status > /dev/null 2>&1; do
    echo "Waiting for Vault..."
    sleep 2
done

echo "Vault is ready!"

# Activer le moteur de secrets KV v2
echo "Enabling KV secrets engine..."
vault secrets enable -path=gameshelf kv-v2 || true

# Stocker les secrets de la base de données
echo "Storing database secrets..."
vault kv put gameshelf/database \
    host="postgresql-addon-host.services.clever-cloud.com" \
    port="5432" \
    database="gameshelf_db" \
    username="db_user" \
    password="super_secret_db_password_123!"

# Stocker les secrets S3/Cellar
echo "Storing S3 secrets..."
vault kv put gameshelf/s3 \
    endpoint="cellar-c2.services.clever-cloud.com" \
    access_key="CELLAR_ACCESS_KEY_ID" \
    secret_key="super_secret_s3_key_456!"

# Stocker les secrets Keycloak
echo "Storing Keycloak secrets..."
vault kv put gameshelf/keycloak \
    url="https://keycloak.gameshelf.com" \
    realm="gameshelf" \
    client_id="gameshelf-api" \
    client_secret="super_secret_keycloak_789!"

# Stocker les clés de l'application
echo "Storing application secrets..."
vault kv put gameshelf/app \
    secret_key="flask_secret_key_very_secure" \
    api_key="gameshelf_api_key_abc123" \
    encryption_key="32_byte_encryption_key_here!!"

# Créer une politique pour l'application
echo "Creating app policy..."
vault policy write gameshelf-app - <<EOF
# Lecture seule sur les secrets GameShelf
path "gameshelf/data/*" {
  capabilities = ["read", "list"]
}

# Lecture des métadonnées
path "gameshelf/metadata/*" {
  capabilities = ["read", "list"]
}
EOF

# Créer une politique admin
echo "Creating admin policy..."
vault policy write gameshelf-admin - <<EOF
# Accès total sur les secrets GameShelf
path "gameshelf/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
EOF

# Activer l'authentification par token
echo "Creating app token..."
vault token create \
    -policy=gameshelf-app \
    -ttl=24h \
    -renewable \
    -display-name="gameshelf-app" \
    -id="app-token-gameshelf"

echo ""
echo "=== Vault initialization complete! ==="
echo ""
echo "Root token: gameshelf-root-token"
echo "App token:  app-token-gameshelf"
echo ""
echo "Vault UI: http://localhost:8200"
echo ""
```

### Étape 2.4 : Démarrer Vault

```bash
docker-compose up -d
```

### Étape 2.5 : Vérifier l'installation

```bash
# Attendre quelques secondes
sleep 10

# Vérifier le statut
docker logs vault

# Vérifier l'init
docker logs vault-init
```

### Étape 2.6 : Accéder à l'interface web

1. Ouvrez votre navigateur : [http://localhost:8200](http://localhost:8200)
2. Méthode d'authentification : **Token**
3. Token : `gameshelf-root-token`

---

## Partie 3 : Explorer Vault via l'interface

### Étape 3.1 : Naviguer dans les secrets

1. Dans le menu, cliquez sur **"Secrets"**
2. Cliquez sur **"gameshelf/"**
3. Explorez les différents secrets :
   - `database`
   - `s3`
   - `keycloak`
   - `app`

### Étape 3.2 : Voir un secret

1. Cliquez sur `database`
2. Observez les champs (host, port, username, password)
3. Notez le **versioning** (v1, v2...)

### Étape 3.3 : Voir les politiques

1. Allez dans **"Policies"**
2. Examinez `gameshelf-app` et `gameshelf-admin`
3. Comprenez les capabilities : read, list, create, update, delete

---

## Partie 4 : Utiliser Vault en ligne de commande

### Étape 4.1 : Installer le client Vault

```bash
# Linux/WSL
wget https://releases.hashicorp.com/vault/1.15.0/vault_1.15.0_linux_amd64.zip
unzip vault_1.15.0_linux_amd64.zip
sudo mv vault /usr/local/bin/
rm vault_1.15.0_linux_amd64.zip

# macOS
brew install vault
```

### Étape 4.2 : Configurer l'accès

```bash
export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN="gameshelf-root-token"
```

### Étape 4.3 : Commandes de base

```bash
# Statut de Vault
vault status

# Lister les secrets engines
vault secrets list

# Lister les secrets
vault kv list gameshelf/

# Lire un secret
vault kv get gameshelf/database

# Lire un champ spécifique
vault kv get -field=password gameshelf/database

# Lire en JSON
vault kv get -format=json gameshelf/database
```

### Étape 4.4 : Modifier un secret

```bash
# Mettre à jour le mot de passe de la DB
vault kv put gameshelf/database \
    host="postgresql-addon-host.services.clever-cloud.com" \
    port="5432" \
    database="gameshelf_db" \
    username="db_user" \
    password="new_rotated_password_$(date +%s)!"

# Voir l'historique des versions
vault kv metadata get gameshelf/database
```

### Étape 4.5 : Tester avec le token applicatif

```bash
# Utiliser le token app (droits limités)
export VAULT_TOKEN="app-token-gameshelf"

# Lecture - devrait fonctionner
vault kv get gameshelf/database

# Écriture - devrait échouer (403)
vault kv put gameshelf/database password="hack"
```

---

## Partie 5 : Intégrer Vault dans l'application

### Étape 5.1 : Créer une application de test

```bash
mkdir -p ~/gameshelf-vault-demo
cd ~/gameshelf-vault-demo
```

### Étape 5.2 : Créer le fichier requirements.txt

```bash
nano requirements.txt
```

```
flask==3.0.0
hvac==2.1.0
python-dotenv==1.0.0
```

### Étape 5.3 : Créer l'application

```bash
nano app.py
```

```python
from flask import Flask, jsonify
import hvac
import os
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger('gameshelf')

app = Flask(__name__)

# Configuration Vault
VAULT_ADDR = os.environ.get('VAULT_ADDR', 'http://localhost:8200')
VAULT_TOKEN = os.environ.get('VAULT_TOKEN', 'app-token-gameshelf')
VAULT_PATH = 'gameshelf'

# Client Vault
vault_client = None


def get_vault_client():
    """Crée et retourne un client Vault."""
    global vault_client
    if vault_client is None:
        vault_client = hvac.Client(
            url=VAULT_ADDR,
            token=VAULT_TOKEN
        )
        if not vault_client.is_authenticated():
            raise Exception("Failed to authenticate with Vault")
        logger.info("Connected to Vault")
    return vault_client


def get_secret(path, key=None):
    """Récupère un secret depuis Vault."""
    try:
        client = get_vault_client()
        response = client.secrets.kv.v2.read_secret_version(
            path=path,
            mount_point=VAULT_PATH
        )
        data = response['data']['data']
        if key:
            return data.get(key)
        return data
    except Exception as e:
        logger.error(f"Failed to get secret {path}: {e}")
        return None


@app.route('/')
def home():
    return jsonify({
        "service": "GameShelf API (Vault Demo)",
        "vault_status": "connected" if get_vault_client().is_authenticated() else "disconnected",
        "endpoints": ["/", "/health", "/config", "/secrets-demo", "/flag"]
    })


@app.route('/health')
def health():
    try:
        client = get_vault_client()
        vault_ok = client.is_authenticated()
    except:
        vault_ok = False

    return jsonify({
        "status": "healthy" if vault_ok else "degraded",
        "vault": "connected" if vault_ok else "error"
    })


@app.route('/config')
def get_config():
    """
    Montre comment l'application récupère sa configuration depuis Vault.
    Note: En production, ne jamais exposer les secrets via une API !
    """
    # Récupérer les secrets de manière sécurisée
    db_config = get_secret('database')
    s3_config = get_secret('s3')
    app_config = get_secret('app')

    # Masquer les valeurs sensibles pour la démo
    return jsonify({
        "database": {
            "host": db_config.get('host') if db_config else None,
            "port": db_config.get('port') if db_config else None,
            "database": db_config.get('database') if db_config else None,
            "username": db_config.get('username') if db_config else None,
            "password": "***MASKED***"  # Ne jamais exposer !
        },
        "s3": {
            "endpoint": s3_config.get('endpoint') if s3_config else None,
            "access_key": "***MASKED***",
            "secret_key": "***MASKED***"
        },
        "app": {
            "secret_key_set": bool(app_config.get('secret_key')) if app_config else False,
            "api_key_set": bool(app_config.get('api_key')) if app_config else False
        },
        "note": "Les vrais secrets sont récupérés depuis Vault mais masqués ici"
    })


@app.route('/secrets-demo')
def secrets_demo():
    """
    Démonstration de la récupération de secrets.
    Cette route montre le workflow sans exposer les vrais secrets.
    """
    steps = []

    # Étape 1: Connexion à Vault
    try:
        client = get_vault_client()
        steps.append({
            "step": 1,
            "action": "Connect to Vault",
            "status": "success",
            "details": f"Connected to {VAULT_ADDR}"
        })
    except Exception as e:
        steps.append({
            "step": 1,
            "action": "Connect to Vault",
            "status": "error",
            "details": str(e)
        })
        return jsonify({"steps": steps}), 500

    # Étape 2: Récupérer les secrets DB
    db_secret = get_secret('database')
    if db_secret:
        steps.append({
            "step": 2,
            "action": "Fetch database secrets",
            "status": "success",
            "details": f"Got {len(db_secret)} fields (host, port, database, username, password)"
        })
    else:
        steps.append({
            "step": 2,
            "action": "Fetch database secrets",
            "status": "error"
        })

    # Étape 3: Récupérer les secrets S3
    s3_secret = get_secret('s3')
    if s3_secret:
        steps.append({
            "step": 3,
            "action": "Fetch S3 secrets",
            "status": "success",
            "details": f"Got {len(s3_secret)} fields (endpoint, access_key, secret_key)"
        })

    # Étape 4: Récupérer les secrets Keycloak
    kc_secret = get_secret('keycloak')
    if kc_secret:
        steps.append({
            "step": 4,
            "action": "Fetch Keycloak secrets",
            "status": "success",
            "details": f"Got {len(kc_secret)} fields (url, realm, client_id, client_secret)"
        })

    # Étape 5: Utilisation
    steps.append({
        "step": 5,
        "action": "Use secrets in application",
        "status": "success",
        "details": "Secrets are now available in memory (not in env vars or config files)"
    })

    return jsonify({
        "demo": "Secret retrieval workflow",
        "steps": steps,
        "security_note": "Secrets are fetched at runtime, never stored in files or env vars"
    })


@app.route('/rotate-demo')
def rotate_demo():
    """Démonstration de la rotation de secrets."""
    import time

    try:
        client = get_vault_client()

        # Lire la version actuelle
        response = client.secrets.kv.v2.read_secret_version(
            path='database',
            mount_point=VAULT_PATH
        )
        current_version = response['data']['metadata']['version']

        # Simuler une rotation (en vrai, ce serait fait par un admin ou un process automatique)
        # Note: avec le token app, cette opération échouera (droits insuffisants)

        return jsonify({
            "demo": "Secret rotation",
            "current_version": current_version,
            "message": "En production, la rotation serait faite par un admin ou automatiquement",
            "benefits": [
                "Limiter l'impact d'une fuite",
                "Conformité réglementaire",
                "Révoquer les accès compromis"
            ]
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route('/flag')
def get_flag():
    """Flag de validation."""
    try:
        # Vérifier qu'on peut accéder à Vault
        client = get_vault_client()
        if not client.is_authenticated():
            return jsonify({"error": "Not connected to Vault"}), 500

        # Récupérer un secret pour prouver que ça fonctionne
        db_secret = get_secret('database')
        if not db_secret:
            return jsonify({"error": "Cannot read secrets"}), 500

        return jsonify({
            "message": "Bravo ! L'application utilise Vault pour les secrets !",
            "flag": "FLAG{V4ult_S3cr3ts_M4n4g3d}",
            "vault_addr": VAULT_ADDR,
            "secrets_retrieved": ["database", "s3", "keycloak", "app"]
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500


if __name__ == '__main__':
    logger.info("Starting GameShelf Vault Demo")
    logger.info(f"Vault address: {VAULT_ADDR}")
    app.run(host='0.0.0.0', port=5000, debug=True)
```

### Étape 5.4 : Exécuter l'application

```bash
# Créer un environnement virtuel
python3 -m venv venv
source venv/bin/activate

# Installer les dépendances
pip install -r requirements.txt

# Configurer les variables
export VAULT_ADDR="http://localhost:8200"
export VAULT_TOKEN="app-token-gameshelf"

# Lancer l'application
python app.py
```

### Étape 5.5 : Tester l'application

```bash
# Dans un autre terminal
curl http://localhost:5000/
curl http://localhost:5000/health
curl http://localhost:5000/config
curl http://localhost:5000/secrets-demo
curl http://localhost:5000/flag
```

---

## Partie 6 : Concepts avancés

### Rotation automatique des secrets

Vault peut générer et faire tourner automatiquement certains secrets :

```bash
# Exemple : activer le moteur de secrets pour les bases de données
vault secrets enable database

# Configurer la connexion PostgreSQL
vault write database/config/gameshelf-db \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@host:5432/gameshelf" \
    allowed_roles="readonly,readwrite" \
    username="vault_admin" \
    password="vault_admin_password"

# Créer un rôle avec credentials dynamiques
vault write database/roles/readonly \
    db_name=gameshelf-db \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}' IN ROLE readonly;" \
    default_ttl="1h" \
    max_ttl="24h"
```

### Authentification par AppRole (production)

En production, n'utilisez pas de tokens statiques :

```bash
# Activer AppRole
vault auth enable approle

# Créer un rôle pour l'application
vault write auth/approle/role/gameshelf-api \
    token_policies="gameshelf-app" \
    token_ttl=1h \
    token_max_ttl=24h \
    secret_id_ttl=10m

# L'application utilise role_id + secret_id pour s'authentifier
vault read auth/approle/role/gameshelf-api/role-id
vault write -f auth/approle/role/gameshelf-api/secret-id
```

### Transit Engine (chiffrement)

Vault peut aussi chiffrer des données sans exposer les clés :

```bash
# Activer le moteur de transit
vault secrets enable transit

# Créer une clé de chiffrement
vault write -f transit/keys/gameshelf-data

# Chiffrer des données
vault write transit/encrypt/gameshelf-data \
    plaintext=$(base64 <<< "données sensibles")

# Déchiffrer
vault write transit/decrypt/gameshelf-data \
    ciphertext="vault:v1:..."
```

---

## Partie 7 : Bonnes pratiques

### Principe du moindre privilège

```yaml
# Mauvais : accès total
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

# Bon : accès restreint et spécifique
path "gameshelf/data/database" {
  capabilities = ["read"]
}
```

### Ne jamais hardcoder les tokens

```python
# Mauvais
VAULT_TOKEN = "root-token-12345"

# Bon
VAULT_TOKEN = os.environ.get('VAULT_TOKEN')

# Encore mieux : AppRole
# L'application s'authentifie dynamiquement
```

### Audit trail

```bash
# Activer l'audit
vault audit enable file file_path=/vault/logs/audit.log

# Chaque opération est loggée avec qui, quoi, quand
```

### Disaster recovery

- Sauvegarder régulièrement les snapshots Vault
- Tester les procédures de restauration
- Avoir plusieurs unseal keys chez différentes personnes

---

## Partie 8 : Architecture production

### Architecture recommandée

```
┌─────────────────────────────────────────────────────────────┐
│                     ARCHITECTURE VAULT                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    ┌─────────┐     ┌─────────────┐     ┌─────────────┐    │
│    │   App   │────►│    Vault    │────►│   Backend   │    │
│    │         │     │   Server    │     │  (Consul)   │    │
│    └─────────┘     └─────────────┘     └─────────────┘    │
│         │                 │                               │
│         │           ┌─────┴─────┐                         │
│         │           │   Auth    │                         │
│         │           │  Backend  │                         │
│         │           └───────────┘                         │
│         │                                                 │
│    AppRole / JWT / Kubernetes Auth                        │
│                                                           │
└─────────────────────────────────────────────────────────────┘
```

### Comparaison avec d'autres solutions

| Solution | Type | Auto-hébergé | Intégrations |
|----------|------|--------------|--------------|
| **HashiCorp Vault** | Complet | Oui | Très nombreuses |
| **AWS Secrets Manager** | Cloud | Non | AWS |
| **Azure Key Vault** | Cloud | Non | Azure |
| **GCP Secret Manager** | Cloud | Non | GCP |
| **Doppler** | SaaS | Non | Multi-cloud |

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Installé Vault avec Docker
- [ ] Créé les secrets GameShelf
- [ ] Créé les politiques d'accès
- [ ] Utilisé l'interface web Vault
- [ ] Utilisé le CLI Vault
- [ ] Testé les différents tokens (root vs app)
- [ ] Créé l'application Python avec hvac
- [ ] Récupéré les secrets depuis l'application
- [ ] Compris les concepts de rotation et audit
- [ ] Récupéré le flag

---

## Flag de validation

```
FLAG{V4ult_S3cr3ts_M4n4g3d}
```

---

## Nettoyage

```bash
# Arrêter Vault
cd ~/vault-demo
docker-compose down -v

# Arrêter l'application Python
# Ctrl+C dans le terminal

# Désactiver l'environnement virtuel
deactivate
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

### Prochaines étapes suggérées

1. **Certification** : AWS, GCP, Azure, ou Kubernetes (CKA)
2. **GitOps** : ArgoCD, Flux
3. **Service Mesh** : Istio, Linkerd
4. **Serverless** : AWS Lambda, Cloud Functions
5. **SRE** : Practices, Chaos Engineering

---

**Durée estimée : 1h30**

**Difficulté : Avancé**

---

*Merci d'avoir suivi ce cours ! Bon courage pour la suite de votre parcours Cloud !*
