# TP 11 - Stockage Objet S3

## Contexte GameShelf

Notre API fonctionne avec PostgreSQL, mais nous devons stocker les images des boîtes de jeux, les règles en PDF, et d'autres fichiers. Le stockage objet est parfait pour ça.

**Objectif de ce TP :** Configurer un stockage objet compatible S3 pour les fichiers.

---

## Prérequis

- TP 10 complété (API avec PostgreSQL)
- Compte Clever Cloud actif

---

## Partie 1 : Concepts du stockage objet

### Stockage objet vs Stockage fichier

| Aspect | Stockage fichier | Stockage objet |
|--------|------------------|----------------|
| Structure | Hiérarchique (dossiers) | Plat (buckets/clés) |
| Métadonnées | Limitées | Riches et personnalisables |
| Accès | Système de fichiers | HTTP/API REST |
| Scalabilité | Limitée | Quasi-illimitée |
| Cas d'usage | Apps traditionnelles | Web, CDN, backups |

### Terminologie S3

- **Bucket** : Conteneur de haut niveau (comme un disque)
- **Object** : Fichier stocké avec une clé unique
- **Key** : Chemin/nom de l'objet (ex: `images/catan.jpg`)
- **ACL** : Permissions d'accès
- **Presigned URL** : Lien temporaire d'accès

---

## Partie 2 : Création du stockage Clever Cloud

### Étape 2.1 : Ajouter un addon Cellar

1. Dans la console Clever Cloud, allez dans votre application

2. **Service dependencies** > **Link an add-on** > **New add-on**

3. Sélectionnez **"Cellar S3 storage"**

4. Choisissez le plan :
   - **S** : 100 Go (suffisant pour le TP)

5. **Nom** : `gameshelf-storage`

6. Cliquez sur **"Create"**

### Étape 2.2 : Récupérer les credentials

Les variables injectées :
- `CELLAR_ADDON_HOST` : Endpoint S3
- `CELLAR_ADDON_KEY_ID` : Access Key
- `CELLAR_ADDON_KEY_SECRET` : Secret Key

---

## Partie 3 : Configuration du client S3

### Étape 3.1 : Installer AWS CLI

L'AWS CLI fonctionne avec tout stockage compatible S3.

```bash
# Linux/WSL
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm -rf aws awscliv2.zip

# macOS
brew install awscli
```

### Étape 3.2 : Configurer le profil Cellar

```bash
aws configure --profile cellar
```

Entrez :
- **Access Key ID** : Votre `CELLAR_ADDON_KEY_ID`
- **Secret Access Key** : Votre `CELLAR_ADDON_KEY_SECRET`
- **Region** : `us-east-1` (ou laisser vide)
- **Output format** : `json`

### Étape 3.3 : Créer un alias pour Cellar

```bash
# Ajouter dans ~/.bashrc ou ~/.zshrc
alias s3cellar='aws --profile cellar --endpoint-url https://cellar-c2.services.clever-cloud.com s3'
```

```bash
source ~/.bashrc
```

---

## Partie 4 : Opérations de base S3

### Étape 4.1 : Créer un bucket

```bash
s3cellar mb s3://gameshelf-images
```

### Étape 4.2 : Lister les buckets

```bash
s3cellar ls
```

### Étape 4.3 : Préparer des images de test

```bash
mkdir -p ~/gameshelf-images
cd ~/gameshelf-images

# Créer des images placeholder (ou téléchargez de vraies images)
echo "Image placeholder pour Catan" > catan.txt
echo "Image placeholder pour Pandemic" > pandemic.txt
echo "Image placeholder pour Codenames" > codenames.txt
```

### Étape 4.4 : Uploader des fichiers

```bash
# Upload un fichier
s3cellar cp catan.txt s3://gameshelf-images/games/catan.txt

# Upload plusieurs fichiers
s3cellar cp . s3://gameshelf-images/games/ --recursive

# Upload avec type MIME
s3cellar cp image.jpg s3://gameshelf-images/games/catan.jpg --content-type image/jpeg
```

### Étape 4.5 : Lister les objets

```bash
s3cellar ls s3://gameshelf-images/
s3cellar ls s3://gameshelf-images/games/
```

### Étape 4.6 : Télécharger des fichiers

```bash
s3cellar cp s3://gameshelf-images/games/catan.txt ./downloaded-catan.txt
```

### Étape 4.7 : Supprimer des fichiers

```bash
s3cellar rm s3://gameshelf-images/games/catan.txt
```

---

## Partie 5 : Accès public aux images

### Étape 5.1 : Rendre un bucket public

Créez une policy pour accès public en lecture :

```bash
nano bucket-policy.json
```

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::gameshelf-images/*"
    }
  ]
}
```

```bash
s3cellar put-bucket-policy --bucket gameshelf-images --policy file://bucket-policy.json
```

### Étape 5.2 : Accéder aux images publiquement

L'URL publique suit ce format :
```
https://<bucket>.cellar-c2.services.clever-cloud.com/<key>
```

Exemple :
```
https://gameshelf-images.cellar-c2.services.clever-cloud.com/games/catan.jpg
```

---

## Partie 6 : Intégration dans l'API

### Étape 6.1 : Mettre à jour l'application

```bash
cd ~/gameshelf-clever
nano app.py
```

Ajoutez les imports et fonctions S3 :

```python
from flask import Flask, jsonify, request
import os
import socket
import psycopg2
from psycopg2.extras import RealDictCursor
import boto3
from botocore.client import Config

app = Flask(__name__)

# Configuration
APP_NAME = os.environ.get('APP_NAME', 'GameShelf')
ENVIRONMENT = os.environ.get('ENVIRONMENT', 'development')

# Configuration S3 (Cellar)
S3_ENDPOINT = os.environ.get('CELLAR_ADDON_HOST', 'cellar-c2.services.clever-cloud.com')
S3_ACCESS_KEY = os.environ.get('CELLAR_ADDON_KEY_ID')
S3_SECRET_KEY = os.environ.get('CELLAR_ADDON_KEY_SECRET')
S3_BUCKET = 'gameshelf-images'

def get_s3_client():
    """Crée un client S3 pour Cellar."""
    return boto3.client(
        's3',
        endpoint_url=f'https://{S3_ENDPOINT}',
        aws_access_key_id=S3_ACCESS_KEY,
        aws_secret_access_key=S3_SECRET_KEY,
        config=Config(signature_version='s3v4')
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

# ... (garder les routes existantes) ...

@app.route('/images')
def list_images():
    """Liste les images dans le bucket S3."""
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

@app.route('/images/<path:key>')
def get_image_url(key):
    """Génère une URL présignée pour une image."""
    try:
        s3 = get_s3_client()
        url = s3.generate_presigned_url(
            'get_object',
            Params={'Bucket': S3_BUCKET, 'Key': key},
            ExpiresIn=3600  # 1 heure
        )
        return jsonify({'key': key, 'url': url, 'expires_in': 3600})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/upload', methods=['POST'])
def upload_image():
    """Upload une image (pour test)."""
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400

    file = request.files['file']
    if file.filename == '':
        return jsonify({'error': 'No file selected'}), 400

    try:
        s3 = get_s3_client()
        key = f"games/{file.filename}"
        s3.upload_fileobj(file, S3_BUCKET, key)
        url = f"https://{S3_BUCKET}.{S3_ENDPOINT}/{key}"
        return jsonify({
            'message': 'Upload successful',
            'key': key,
            'url': url
        })
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/flag')
def get_flag():
    try:
        # Vérifier la connexion S3
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
            "message": "Bravo ! L'API utilise PostgreSQL et S3 !",
            "flag": "FLAG{S3_C3ll4r_St0r4g3_C0nf1gur3d}",
            "database": "connected",
            "storage": s3_status
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
```

### Étape 6.2 : Mettre à jour requirements.txt

```
flask==3.0.0
gunicorn==21.2.0
psycopg2-binary==2.9.9
boto3==1.34.0
```

### Étape 6.3 : Déployer

```bash
git add .
git commit -m "Add S3 storage integration"
git push clever main
```

---

## Partie 7 : Test des fonctionnalités S3

```bash
export APP_URL="https://app-xxxxxxxx.cleverapps.io"

# Lister les images
curl $APP_URL/images

# Obtenir une URL présignée
curl "$APP_URL/images/games/catan.jpg"

# Upload une image (avec curl)
curl -X POST -F "file=@/path/to/image.jpg" $APP_URL/upload

# Flag
curl $APP_URL/flag
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé un addon Cellar (S3) sur Clever Cloud
- [ ] Configuré AWS CLI avec le profil Cellar
- [ ] Créé un bucket
- [ ] Uploadé des fichiers
- [ ] Configuré l'accès public
- [ ] Intégré S3 dans l'API
- [ ] Testé les endpoints /images et /upload
- [ ] Récupéré le flag

---

## Flag de validation

```
FLAG{S3_C3ll4r_St0r4g3_C0nf1gur3d}
```

---

## Transition vers le TP suivant

*"Nos fichiers sont maintenant stockés de manière durable et accessible ! Mais comment savoir ce qui se passe dans notre application en production ? Dans le prochain TP, nous allons mettre en place la centralisation des logs !"*

---

**Durée estimée : 1 heure**

**Difficulté : Intermédiaire**
