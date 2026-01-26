# TP 05 - Haute Disponibilité et Load Balancing

## Contexte GameShelf

Le site GameShelf gagne en popularité ! Pendant les fêtes de Noël et les sorties de jeux très attendus, notre unique serveur ne suffit plus. Nous devons mettre en place une architecture haute disponibilité.

**Objectif de ce TP :** Configurer un cluster de 3 VMs avec un load balancer et 2 serveurs web en utilisant Ansible.

---

## Prérequis

- TP 02, 03 et 04 complétés
- Clé SSH générée (TP 01)
- Ansible installé sur votre machine

---

## Partie 1 : Formation des groupes

### Étape 1.1 : Créer un groupe sur la Student App

Ce TP se fait en **groupe de 3 étudiants**. Chaque groupe recevra **3 VMs** pour configurer un cluster haute disponibilité.

1. Rendez-vous sur la Student App :

   **https://app-3057b668-1ef3-4577-9236-ce5b3326451d.cleverapps.io**

2. Connectez-vous avec votre email ISEN

3. Cliquez sur **TP05 - Haute Disponibilité**

4. **Un seul membre** du groupe crée le groupe :
   - Entrez un nom de groupe (ex: `groupe-alpha`)
   - Ajoutez les 2 autres membres avec leur email et clé SSH publique
   - Validez la création

5. Une fois le groupe créé, **attendez que le professeur assigne les 3 VMs** à votre groupe

### Étape 1.2 : Récupérer les informations des VMs

Une fois les VMs assignées, la Student App affiche :
- Les adresses IP des 3 VMs
- Les commandes SSH pour s'y connecter

Notez ces informations :
- **VM 1 (Load Balancer)** : `<IP_LB>`
- **VM 2 (Web Server 1)** : `<IP_WEB1>`
- **VM 3 (Web Server 2)** : `<IP_WEB2>`

### Étape 1.3 : Tester la connexion SSH

Vérifiez que chaque membre peut se connecter aux 3 VMs :

```bash
ssh ubuntu@<IP_LB>
ssh ubuntu@<IP_WEB1>
ssh ubuntu@<IP_WEB2>
```

---

## Partie 2 : Concepts de haute disponibilité

### Qu'est-ce que la haute disponibilité (HA) ?

La HA garantit qu'un service reste accessible même en cas de panne d'un composant.

**Principes clés :**
- **Redondance** : Plusieurs serveurs identiques
- **Load Balancing** : Répartition de charge
- **Health Checks** : Détection automatique des pannes
- **Failover** : Basculement automatique

### Architecture cible

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    │   (Nginx LB)    │
                    │   <IP_LB>:80    │
                    └────────┬────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
              ▼                             ▼
        ┌──────────┐                  ┌──────────┐
        │  Web 01  │                  │  Web 02  │
        │  Nginx   │                  │  Nginx   │
        │ <IP_WEB1>│                  │ <IP_WEB2>│
        └──────────┘                  └──────────┘
```

### Types de Load Balancing

| Algorithme | Description |
|------------|-------------|
| **Round Robin** | Alternance entre les serveurs |
| **Least Connections** | Vers le serveur le moins chargé |
| **IP Hash** | Même client = même serveur (sessions) |
| **Weighted** | Pondération selon la capacité |

---

## Partie 3 : Configuration du projet Ansible

### Étape 3.1 : Créer le dossier du projet

```bash
mkdir -p ~/gameshelf-ha
cd ~/gameshelf-ha
```

### Étape 3.2 : Créer l'inventaire

Créez l'inventaire avec vos IPs :

```bash
nano inventory.ini
```

```ini
[loadbalancer]
lb ansible_host=<IP_LB>

[webservers]
web1 ansible_host=<IP_WEB1>
web2 ansible_host=<IP_WEB2>

[all:vars]
ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_python_interpreter=/usr/bin/python3
```

**Remplacez** `<IP_LB>`, `<IP_WEB1>` et `<IP_WEB2>` par les vraies adresses IP.

### Étape 3.3 : Créer la configuration Ansible

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

### Étape 3.4 : Tester la connexion

```bash
ansible all -m ping
```

Résultat attendu : tous les serveurs répondent `pong`.

---

## Partie 4 : Playbook pour les serveurs web

### Étape 4.1 : Créer le dossier des templates

```bash
mkdir -p templates
```

### Étape 4.2 : Template de la page d'accueil

```bash
nano templates/index.html.j2
```

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ app_name }} - Serveur {{ ansible_facts['hostname'] }}</title>
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
        .server-badge {
            display: inline-block;
            padding: 10px 25px;
            background: #e74c3c;
            border-radius: 25px;
            font-size: 1.2em;
            margin: 10px 0;
            font-weight: bold;
        }
        .badge {
            display: inline-block;
            padding: 5px 15px;
            background: #3498db;
            border-radius: 20px;
            font-size: 0.9em;
            margin: 5px;
        }
        .status {
            margin-top: 20px;
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

        <div class="server-badge">
            Serveur: {{ ansible_facts['hostname'] }}
        </div>

        <p>Architecture Haute Disponibilité</p>

        <div>
            <span class="badge">Load Balanced</span>
            <span class="badge">HA</span>
            <span class="badge">Nginx</span>
        </div>

        <div class="status">
            Serveur opérationnel
        </div>

        <div class="info">
            <strong>Informations du serveur :</strong><br>
            - Hostname: {{ ansible_facts['hostname'] }}<br>
            - IP: {{ ansible_facts['default_ipv4']['address'] | default('N/A') }}<br>
            - OS: {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}<br>
            - Géré par: Ansible
        </div>

        <p class="footer">
            TP05 - Haute Disponibilité | Master 1 Cloud Computing
        </p>

        <!-- FLAG{H4_Sv2r_{{ ansible_facts['hostname'] }}_0nL1n3} -->
    </div>
</body>
</html>
```

### Étape 4.3 : Template de configuration Nginx pour les serveurs web

```bash
nano templates/nginx-web.conf.j2
```

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    root /var/www/gameshelf;
    index index.html;

    access_log /var/log/nginx/gameshelf_access.log;
    error_log /var/log/nginx/gameshelf_error.log;

    # Endpoint de health check pour le Load Balancer
    location /health {
        access_log off;
        return 200 "OK - {{ ansible_facts['hostname'] }}\n";
        add_header Content-Type text/plain;
    }

    location / {
        try_files $uri $uri/ =404;
    }

    # Headers de sécurité
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Served-By "{{ ansible_facts['hostname'] }}" always;
}
```

### Étape 4.4 : Créer le playbook pour les serveurs web

```bash
nano playbook-webservers.yml
```

```yaml
---
- name: Configuration des serveurs web GameShelf
  hosts: webservers
  become: yes

  vars:
    app_name: "GameShelf"

  tasks:
    - name: Mise à jour du cache apt
      apt:
        update_cache: yes

    - name: Installation de Nginx
      apt:
        name: nginx
        state: present

    - name: S'assurer que Nginx est démarré
      service:
        name: nginx
        state: started
        enabled: yes

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

    - name: Configurer Nginx
      template:
        src: templates/nginx-web.conf.j2
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
      service:
        name: nginx
        state: reloaded
```

### Étape 4.5 : Exécuter le playbook sur les serveurs web

```bash
ansible-playbook playbook-webservers.yml
```

### Étape 4.6 : Vérifier les serveurs web

Testez chaque serveur individuellement :

```bash
curl http://<IP_WEB1>
curl http://<IP_WEB2>
```

Vérifiez les health checks :

```bash
curl http://<IP_WEB1>/health
curl http://<IP_WEB2>/health
```

---

## Partie 5 : Playbook pour le Load Balancer

### Étape 5.1 : Template de configuration Nginx pour le Load Balancer

```bash
nano templates/nginx-lb.conf.j2
```

```nginx
upstream gameshelf_backend {
    # Round Robin par défaut
{% for host in groups['webservers'] %}
    server {{ hostvars[host]['ansible_host'] }}:80;
{% endfor %}
}

server {
    listen 80;
    listen [::]:80;

    server_name _;

    access_log /var/log/nginx/lb_access.log;
    error_log /var/log/nginx/lb_error.log;

    location / {
        proxy_pass http://gameshelf_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeout settings
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
    }

    # Page de status du Load Balancer
    location /lb-status {
        return 200 "Load Balancer OK\nBackends: {% for host in groups['webservers'] %}{{ hostvars[host]['ansible_host'] }}{% if not loop.last %}, {% endif %}{% endfor %}\n";
        add_header Content-Type text/plain;
    }

    # Health check passthrough
    location /health {
        proxy_pass http://gameshelf_backend;
    }
}
```

### Étape 5.2 : Créer le playbook pour le Load Balancer

```bash
nano playbook-loadbalancer.yml
```

```yaml
---
- name: Configuration du Load Balancer GameShelf
  hosts: loadbalancer
  become: yes

  tasks:
    - name: Mise à jour du cache apt
      apt:
        update_cache: yes

    - name: Installation de Nginx
      apt:
        name: nginx
        state: present

    - name: S'assurer que Nginx est démarré
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Configurer Nginx comme Load Balancer
      template:
        src: templates/nginx-lb.conf.j2
        dest: /etc/nginx/sites-available/loadbalancer
        owner: root
        group: root
        mode: '0644'
      notify: Reload Nginx

    - name: Activer la configuration du Load Balancer
      file:
        src: /etc/nginx/sites-available/loadbalancer
        dest: /etc/nginx/sites-enabled/loadbalancer
        state: link
      notify: Reload Nginx

    - name: Désactiver le site par défaut
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Reload Nginx

  handlers:
    - name: Reload Nginx
      service:
        name: nginx
        state: reloaded
```

### Étape 5.3 : Exécuter le playbook du Load Balancer

```bash
ansible-playbook playbook-loadbalancer.yml
```

---

## Partie 6 : Playbook complet (optionnel)

Pour configurer tout le cluster en une seule commande, créez un playbook global :

```bash
nano playbook-ha.yml
```

```yaml
---
- name: Configuration des serveurs web GameShelf
  hosts: webservers
  become: yes

  vars:
    app_name: "GameShelf"

  tasks:
    - name: Mise à jour du cache apt
      apt:
        update_cache: yes

    - name: Installation de Nginx
      apt:
        name: nginx
        state: present

    - name: S'assurer que Nginx est démarré
      service:
        name: nginx
        state: started
        enabled: yes

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
      notify: Reload Nginx Web

    - name: Configurer Nginx
      template:
        src: templates/nginx-web.conf.j2
        dest: /etc/nginx/sites-available/gameshelf
        owner: root
        group: root
        mode: '0644'
      notify: Reload Nginx Web

    - name: Activer le virtual host
      file:
        src: /etc/nginx/sites-available/gameshelf
        dest: /etc/nginx/sites-enabled/gameshelf
        state: link
      notify: Reload Nginx Web

    - name: Désactiver le site par défaut
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Reload Nginx Web

  handlers:
    - name: Reload Nginx Web
      service:
        name: nginx
        state: reloaded

- name: Configuration du Load Balancer GameShelf
  hosts: loadbalancer
  become: yes

  tasks:
    - name: Mise à jour du cache apt
      apt:
        update_cache: yes

    - name: Installation de Nginx
      apt:
        name: nginx
        state: present

    - name: S'assurer que Nginx est démarré
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Configurer Nginx comme Load Balancer
      template:
        src: templates/nginx-lb.conf.j2
        dest: /etc/nginx/sites-available/loadbalancer
        owner: root
        group: root
        mode: '0644'
      notify: Reload Nginx LB

    - name: Activer la configuration du Load Balancer
      file:
        src: /etc/nginx/sites-available/loadbalancer
        dest: /etc/nginx/sites-enabled/loadbalancer
        state: link
      notify: Reload Nginx LB

    - name: Désactiver le site par défaut
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: Reload Nginx LB

  handlers:
    - name: Reload Nginx LB
      service:
        name: nginx
        state: reloaded
```

Exécutez tout en une commande :

```bash
ansible-playbook playbook-ha.yml
```

---

## Partie 7 : Tests de haute disponibilité

### Étape 7.1 : Tester le Load Balancer

Accédez au Load Balancer dans votre navigateur :

```
http://<IP_LB>
```

### Étape 7.2 : Observer le Round Robin

Rafraîchissez plusieurs fois la page. Le nom du serveur change :
- `web1`
- `web2`

### Étape 7.3 : Vérifier avec curl

```bash
for i in {1..10}; do
  echo "Requête $i:"
  curl -s http://<IP_LB>/health
done
```

Vous verrez les requêtes réparties entre les serveurs.

### Étape 7.4 : Vérifier le header X-Served-By

```bash
for i in {1..10}; do
  curl -s -I http://<IP_LB> | grep X-Served-By
done
```

### Étape 7.5 : Test de failover

1. Connectez-vous à un serveur et arrêtez Nginx :

```bash
ssh ubuntu@<IP_WEB1>
sudo systemctl stop nginx
exit
```

2. Testez l'accès via le Load Balancer :

```bash
curl http://<IP_LB>
```

Le site reste accessible ! Toutes les requêtes vont sur web2.

3. Vérifiez :

```bash
for i in {1..5}; do
  curl -s http://<IP_LB>/health
done
```

4. Redémarrez Nginx sur web1 :

```bash
ssh ubuntu@<IP_WEB1>
sudo systemctl start nginx
exit
```

5. Après quelques secondes, le Load Balancer détecte le retour de web1.

### Étape 7.6 : Vérifier le status du Load Balancer

```bash
curl http://<IP_LB>/lb-status
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé un groupe de 3 étudiants dans la Student App
- [ ] Récupéré les 3 VMs assignées à votre groupe
- [ ] Configuré l'inventaire Ansible avec les 3 VMs
- [ ] Exécuté le playbook des serveurs web
- [ ] Exécuté le playbook du Load Balancer
- [ ] Vérifié la répartition de charge (Round Robin)
- [ ] Testé le failover (arrêt d'un serveur)
- [ ] Vérifié que le site reste accessible malgré la panne

---

## Flag de validation

**Flags des serveurs** (dans le code source HTML de chaque serveur) :
- `FLAG{H4_Sv2r_<hostname_web1>_0nL1n3}`
- `FLAG{H4_Sv2r_<hostname_web2>_0nL1n3}`

**Flag global** :
```
FLAG{H4_Cl5st3r_Ans1bl3_L0adB4l4nc3r}
```

---

## Structure du projet

```
gameshelf-ha/
├── ansible.cfg
├── inventory.ini
├── playbook-webservers.yml
├── playbook-loadbalancer.yml
├── playbook-ha.yml          # (optionnel) Playbook complet
└── templates/
    ├── index.html.j2
    ├── nginx-web.conf.j2
    └── nginx-lb.conf.j2
```

---

## Commandes utiles

| Commande | Description |
|----------|-------------|
| `ansible all -m ping` | Tester la connectivité |
| `ansible webservers -m command -a "systemctl status nginx"` | État de Nginx sur les web servers |
| `ansible loadbalancer -m command -a "systemctl status nginx"` | État de Nginx sur le LB |
| `curl -I http://<IP_LB>` | Voir les headers (X-Served-By) |
| `curl http://<IP_LB>/lb-status` | Status du Load Balancer |

---

## En cas de problème

| Problème | Solution |
|----------|----------|
| "Connection refused" SSH | Vérifiez que votre clé SSH est dans le groupe |
| "Permission denied" | Vérifiez l'utilisateur (ubuntu) et la clé SSH |
| LB inaccessible | Vérifiez que Nginx est démarré sur le LB |
| Health check échoue | Vérifiez l'endpoint /health sur les serveurs |
| Round Robin ne fonctionne pas | Videz le cache du navigateur ou utilisez curl |

---

## Transition vers le TP suivant

*"Notre site est maintenant résilient à la panne d'un serveur. Mais que se passe-t-il si on perd les deux serveurs web ? Il faudrait pouvoir recréer rapidement des instances... Et si on pouvait packager notre application pour la déployer instantanément ? Prochain TP : conteneurisation avec Docker !"*

---

**Difficulté : Intermédiaire**
