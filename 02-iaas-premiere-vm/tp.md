# TP 02 - IaaS : Première Machine Virtuelle

## Contexte GameShelf

GameShelf a besoin d'un serveur pour héberger son site vitrine. Une machine virtuelle a été pré-provisionnée pour vous sur le cloud. Vous allez vous y connecter et installer un serveur web Nginx.

**Objectif de ce TP :** Se connecter à une VM et installer manuellement un serveur web.

---

## Prérequis

- TP 01 complété (clé SSH générée)
- Un terminal (Linux/macOS) ou WSL2 (Windows)
- Connaissances de base en ligne de commande

---

## Partie 1 : Récupération de votre VM

### Étape 1.1 : Accéder à la Student App

1. Ouvrez votre navigateur et rendez-vous sur :

   **https://app-3057b668-1ef3-4577-9236-ce5b3326451d.cleverapps.io**

2. Connectez-vous avec vos identifiants étudiants

### Étape 1.2 : Enregistrer votre clé SSH publique

1. Affichez votre clé publique dans le terminal :

```bash
cat ~/.ssh/id_ed25519.pub
```

2. Copiez l'intégralité de la clé (commence par `ssh-ed25519`)

3. Dans la Student App, collez votre clé SSH publique dans le champ prévu

4. Validez l'enregistrement

### Étape 1.3 : Récupérer l'adresse IP de votre VM

1. Une fois votre clé enregistrée, la Student App affiche les informations de votre VM :
   - **Adresse IP publique** : `xxx.xxx.xxx.xxx`
   - **Utilisateur** : `ubuntu`

2. Notez ces informations, vous en aurez besoin pour la suite

---

## Partie 2 : Connexion à la VM

### Étape 2.1 : Première connexion SSH

Ouvrez votre terminal et connectez-vous :

```bash
ssh ubuntu@<VOTRE_IP_PUBLIQUE>
```

Exemple :
```bash
ssh ubuntu@51.68.123.45
```

### Étape 2.2 : Accepter l'empreinte du serveur

Lors de la première connexion, vous verrez :

```
The authenticity of host '51.68.123.45' can't be established.
ED25519 key fingerprint is SHA256:xxxxx...
Are you sure you want to continue connecting (yes/no)?
```

Tapez `yes` et appuyez sur Entrée.

### Étape 2.3 : Vous êtes connecté !

Vous devriez voir un prompt comme :

```
ubuntu@gameshelf-web-01:~$
```

Vérifiez les informations système :

```bash
# Version Ubuntu
cat /etc/os-release

# Ressources disponibles
free -h
df -h
```

---

## Partie 3 : Installation de Nginx

### Étape 3.1 : Mise à jour du cache des paquets

Avant toute installation, mettez à jour la liste des paquets :

```bash
sudo apt update
```

### Étape 3.2 : Installation de Nginx

```bash
sudo apt install nginx -y
```

### Étape 3.3 : Vérification du service

```bash
# Vérifier que Nginx est actif
sudo systemctl status nginx
```

Vous devriez voir : `Active: active (running)`

### Étape 3.4 : Test dans le navigateur

1. Ouvrez votre navigateur web

2. Tapez l'adresse IP de votre VM : `http://<VOTRE_IP_PUBLIQUE>`

3. Vous devriez voir la page par défaut de Nginx : **"Welcome to nginx!"**

---

## Partie 4 : Personnalisation de la page d'accueil

### Étape 4.1 : Localiser le fichier de la page d'accueil

```bash
ls /var/www/html/
```

Vous verrez le fichier `index.nginx-debian.html`.

### Étape 4.2 : Créer une page personnalisée pour GameShelf

Créez un nouveau fichier `index.html` :

```bash
sudo nano /var/www/html/index.html
```

Collez le contenu suivant :

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GameShelf - Votre boutique de jeux de société</title>
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
        }
        h1 {
            font-size: 3em;
            margin-bottom: 10px;
            color: #f39c12;
        }
        p {
            font-size: 1.2em;
            color: #bbb;
        }
        .status {
            margin-top: 30px;
            padding: 15px 30px;
            background: #27ae60;
            border-radius: 10px;
            display: inline-block;
        }
        .footer {
            margin-top: 40px;
            font-size: 0.9em;
            color: #666;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>GameShelf</h1>
        <p>Votre boutique de jeux de société préférée</p>
        <div class="status">
            Serveur opérationnel
        </div>
        <p class="footer">
            Hébergé sur le Cloud | Ubuntu 22.04 | Nginx
        </p>
        <!-- FLAG{Qm3xK9pL2wNv5tRz_FIRST_VM_DEPLOYED} -->
    </div>
</body>
</html>
```

**Pour sauvegarder dans nano :**
1. Appuyez sur `Ctrl + O` puis Entrée
2. Appuyez sur `Ctrl + X` pour quitter

### Étape 4.3 : Vérifier la nouvelle page

1. Actualisez votre navigateur à l'adresse `http://<VOTRE_IP_PUBLIQUE>`

2. Vous devriez voir la page GameShelf personnalisée

---

## Partie 5 : Configuration du pare-feu

### Étape 5.1 : Activer UFW

UFW (Uncomplicated Firewall) permet de sécuriser votre serveur :

```bash
# Autoriser SSH (important !)
sudo ufw allow ssh

# Autoriser HTTP
sudo ufw allow http

# Autoriser HTTPS (pour plus tard)
sudo ufw allow https

# Activer le pare-feu
sudo ufw enable
```

Tapez `y` pour confirmer.

### Étape 5.2 : Vérifier les règles

```bash
sudo ufw status
```

Vous devriez voir :
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere
80/tcp                     ALLOW       Anywhere
443/tcp                    ALLOW       Anywhere
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Récupéré l'IP de votre VM via la Student App
- [ ] Connecté en SSH à la VM
- [ ] Installé Nginx
- [ ] Personnalisé la page d'accueil GameShelf
- [ ] Configuré le pare-feu UFW

---

## Flag de validation

Le flag est caché dans le code source de la page. Pour le récupérer :

1. Ouvrez votre navigateur à l'adresse `http://<VOTRE_IP_PUBLIQUE>`

2. Faites un clic droit > **"Afficher le code source"** (ou `Ctrl + U`)

3. Cherchez le commentaire HTML contenant le flag :

```
FLAG{Qm3xK9pL2wNv5tRz_FIRST_VM_DEPLOYED}
```

> Ce flag prouve que vous avez configuré votre première VM avec succès.

---

## Transition vers le TP suivant

*"Excellent ! Notre serveur web est opérationnel. Mais imaginez devoir refaire toutes ces étapes manuellement pour chaque nouvelle VM... Dans le prochain TP, nous allons automatiser tout ça avec Terraform !"*

---

## Commandes utiles à retenir

| Commande | Description |
|----------|-------------|
| `ssh user@ip` | Connexion SSH |
| `sudo apt update` | Mettre à jour la liste des paquets |
| `sudo apt install <paquet>` | Installer un paquet |
| `sudo systemctl status nginx` | Vérifier le statut de Nginx |
| `sudo systemctl restart nginx` | Redémarrer Nginx |
| `sudo ufw status` | Voir les règles du pare-feu |

---

## En cas de problème

| Problème | Solution |
|----------|----------|
| "Connection refused" SSH | Vérifiez que la VM est bien démarrée dans la Student App |
| "Permission denied" SSH | Vérifiez que vous utilisez le bon user (`ubuntu`) et que votre clé SSH est bien enregistrée |
| Page Nginx inaccessible | Vérifiez le pare-feu (`sudo ufw status`) |
| Erreur lors de apt install | Lancez `sudo apt update` d'abord |

---

## Pour aller plus loin

- Consultez les logs Nginx : `sudo tail -f /var/log/nginx/access.log`
- Testez la charge avec `curl` depuis votre machine locale
- Explorez la configuration Nginx dans `/etc/nginx/`
