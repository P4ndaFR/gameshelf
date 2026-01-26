# TP 04 - Configuration Management avec Ansible + Docker

## Contexte GameShelf

Notre infrastructure est déployée avec Terraform (Docker). Nous allons maintenant automatiser la configuration avec Ansible en ciblant un conteneur Docker.

**Objectif de ce TP :** Apprendre Ansible en configurant un conteneur Docker.

---

## Prérequis

- TP 03 complété (Terraform + Docker)
- Docker installé et fonctionnel
- Terminal Linux/macOS ou WSL2

---

## Partie 1 : Préparer le conteneur cible

### Étape 1.1 : Créer un Dockerfile pour une "VM" avec SSH

Créez un dossier de travail :

```bash
mkdir -p ~/gameshelf-ansible-docker
cd ~/gameshelf-ansible-docker
```

Créez le Dockerfile :

```bash
nano Dockerfile
```

```dockerfile
FROM ubuntu:22.04

# Éviter les prompts interactifs
ENV DEBIAN_FRONTEND=noninteractive

# Installer les paquets nécessaires (on garde le cache apt pour Ansible)
RUN apt-get update && apt-get install -y \
    openssh-server \
    python3 \
    python3-apt \
    sudo

# Configurer SSH
RUN mkdir /var/run/sshd
RUN echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config
RUN echo 'PasswordAuthentication no' >> /etc/ssh/sshd_config
RUN echo 'PubkeyAuthentication yes' >> /etc/ssh/sshd_config

# Créer l'utilisateur ubuntu avec sudo sans mot de passe
RUN useradd -m -s /bin/bash ubuntu && \
    echo 'ubuntu ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

# Créer le dossier .ssh avec les bonnes permissions
RUN mkdir -p /home/ubuntu/.ssh && \
    chmod 700 /home/ubuntu/.ssh && \
    chown ubuntu:ubuntu /home/ubuntu/.ssh

# Exposer le port SSH
EXPOSE 22

# Démarrer SSH
CMD ["/usr/sbin/sshd", "-D"]
```

### Étape 1.2 : Construire l'image

```bash
docker build -t ubuntu-ssh:22.04 .
```

### Étape 1.3 : Préparer la clé SSH

Si vous n'avez pas encore de clé SSH :

```bash
ssh-keygen -t ed25519 -C "ansible@local" -f ~/.ssh/id_ansible -N ""
```

### Étape 1.4 : Lancer le conteneur avec votre clé SSH

```bash
# Lancer le conteneur avec la clé publique
docker run -d \
  --name gameshelf-target \
  -p 2222:22 \
  -p 8080:80 \
  -v ~/.ssh/id_ed25519.pub:/home/ubuntu/.ssh/authorized_keys:ro \
  ubuntu-ssh:22.04
```

### Étape 1.5 : Tester la connexion SSH

```bash
ssh -p 2222 -i ~/.ssh/id_ed25519 ubuntu@localhost
```

Si vous voyez le prompt `ubuntu@<container_id>:~$`, la connexion fonctionne ! Tapez `exit` pour sortir.

---

## Partie 2 : Installation d'Ansible

### Étape 2.1 : Installer Ansible

```bash
# Ubuntu/Debian/WSL
sudo apt update
sudo apt install -y software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible
```

```bash
# macOS
brew install ansible
```

### Étape 2.2 : Vérifier l'installation

```bash
ansible --version
```

---

## Partie 3 : Configuration du projet Ansible

### Étape 3.1 : Créer l'inventaire

```bash
nano inventory.ini
```

```ini
[webservers]
gameshelf-web ansible_host=localhost ansible_port=2222 ansible_user=ubuntu

[webservers:vars]
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

### Étape 3.2 : Créer la configuration Ansible

```bash
nano ansible.cfg
```

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
retry_files_enabled = False

[privilege_escalation]
become = True
become_method = sudo
become_user = root
become_ask_pass = False
```

### Étape 3.3 : Tester la connexion

```bash
ansible all -m ping
```

Résultat attendu :
```yaml
gameshelf-web | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

---

## Partie 4 : Playbook d'installation Nginx

### Étape 4.1 : Créer le playbook

```bash
nano playbook-nginx.yml
```

```yaml
---
- name: Configuration du serveur web GameShelf
  hosts: webservers
  become: yes

  vars:
    app_name: "GameShelf"
    nginx_port: 80

  tasks:
    - name: Mise à jour du cache apt
      apt:
        update_cache: yes

    - name: Installation de Nginx
      apt:
        name: nginx
        state: present

    - name: S'assurer que Nginx est démarré
      shell: |
        if ! pgrep nginx > /dev/null; then
          nginx
        fi
      changed_when: false

    - name: Créer le répertoire web
      file:
        path: /var/www/gameshelf
        state: directory
        owner: www-data
        group: www-data
        mode: '0755'

    - name: Déployer la page d'accueil
      template:
        src: templates/index.html.j2
        dest: /var/www/gameshelf/index.html
        owner: www-data
        group: www-data
        mode: '0644'
      notify: Reload Nginx

    - name: Configurer le virtual host Nginx
      template:
        src: templates/nginx-vhost.conf.j2
        dest: /etc/nginx/sites-available/gameshelf
        owner: root
        group: root
        mode: '0644'
      notify: Reload Nginx

    - name: Activer le virtual host
      file:
        src: /etc/nginx/sites-available/gameshelf
        dest: /etc/nginx/sites-enabled/gameshelf
        state: link
      notify: Reload Nginx

    - name: Désactiver le site par défaut
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      command: nginx -s reload
```

### Étape 4.2 : Créer les templates

```bash
mkdir -p templates
```

```bash
nano templates/index.html.j2
```

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ app_name }} - Déployé par Ansible</title>
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
        .badge {
            display: inline-block;
            padding: 5px 15px;
            background: #9b59b6;
            border-radius: 20px;
            font-size: 0.9em;
            margin: 5px;
        }
        .status {
            margin-top: 30px;
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
            text-align: left;
            font-family: monospace;
            font-size: 0.9em;
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
        <h1>{{ app_name }}</h1>
        <p>Configuration automatisée avec Ansible</p>

        <div>
            <span class="badge">Ansible</span>
            <span class="badge">Docker</span>
            <span class="badge">Nginx</span>
        </div>

        <div class="status">
            Conteneur configuré automatiquement
        </div>

        <div class="info">
            <strong>Informations du conteneur :</strong><br>
            - Hostname: {{ ansible_facts['hostname'] }}<br>
            - OS: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}<br>
            - Port Nginx: {{ nginx_port }}<br>
            - Géré par: Ansible + Docker
        </div>

        <p class="footer">
            TP04 - Configuration Management | Master 1 Cloud Computing
        </p>

        <!-- FLAG{Dkr4ns1bl3_Cfg9pQw2x_ANSIBLE_DOCKER} -->
    </div>
</body>
</html>
```

```bash
nano templates/nginx-vhost.conf.j2
```

```nginx
server {
    listen {{ nginx_port }};
    listen [::]:{{ nginx_port }};

    server_name _;

    root /var/www/gameshelf;
    index index.html;

    access_log /var/log/nginx/gameshelf_access.log;
    error_log /var/log/nginx/gameshelf_error.log;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript;

    location / {
        try_files $uri $uri/ =404;
    }

    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
}
```

---

## Partie 5 : Exécution du Playbook

### Étape 5.1 : Vérifier la syntaxe

```bash
ansible-playbook playbook-nginx.yml --syntax-check
```

### Étape 5.2 : Exécuter le playbook

```bash
ansible-playbook playbook-nginx.yml
```

### Étape 5.3 : Vérifier le résultat

Ouvrez votre navigateur : **http://localhost:8080**

---

## Partie 6 : Idempotence

### Étape 6.1 : Relancer le playbook

```bash
ansible-playbook playbook-nginx.yml
```

Observez : `changed=0` ! Ansible détecte que tout est déjà configuré.

### Étape 6.2 : Modifier une variable

Dans le playbook, changez :
```yaml
app_name: "GameShelf - Docker Edition"
```

Relancez :
```bash
ansible-playbook playbook-nginx.yml
```

Seule la tâche "Déployer la page d'accueil" est en `changed`.

---

## Partie 7 : Commandes utiles

### Commandes ad-hoc

```bash
# Voir l'uptime
ansible all -m command -a "uptime"

# Lister les fichiers web
ansible all -m command -a "ls -la /var/www/gameshelf"

# Vérifier Nginx
ansible all -m shell -a "nginx -t"
```

### Collecter les facts

```bash
ansible all -m setup | less
```

---

## Partie 8 : Nettoyage

### Étape 8.1 : Arrêter le conteneur

```bash
docker stop gameshelf-target
docker rm gameshelf-target
```

### Étape 8.2 : Supprimer l'image (optionnel)

```bash
docker rmi ubuntu-ssh:22.04
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé un conteneur Docker avec SSH
- [ ] Installé Ansible
- [ ] Créé l'inventaire et testé la connexion
- [ ] Créé et exécuté le playbook
- [ ] Vérifié la page web sur http://localhost:8080
- [ ] Compris l'idempotence (changed=0)

---

## Flag de validation

Le flag est dans le code source de la page :

```
FLAG{Dkr4ns1bl3_Cfg9pQw2x_ANSIBLE_DOCKER}
```

---

## Transition vers le TP suivant

*"Notre serveur est déployé et configuré automatiquement. Mais que se passe-t-il avec beaucoup de trafic ? Un seul serveur ne suffira pas ! Prochain TP : haute disponibilité avec load balancer !"*

---

## Structure du projet

```
gameshelf-ansible-docker/
├── Dockerfile            # Image Docker avec SSH
├── ansible.cfg           # Configuration Ansible
├── inventory.ini         # Inventaire
├── playbook-nginx.yml    # Playbook principal
└── templates/
    ├── index.html.j2
    └── nginx-vhost.conf.j2
```

---

## En cas de problème

| Problème | Solution |
|----------|----------|
| "Connection refused" | Vérifiez que le conteneur tourne : `docker ps` |
| "Permission denied" | Vérifiez les permissions sur authorized_keys |
| "Host unreachable" | Vérifiez le port : `-p 2222:22` |
| Nginx ne démarre pas | Vérifiez les logs : `docker logs gameshelf-target` |

---

**Durée estimée : 1 heure**

**Difficulté : Intermédiaire**
