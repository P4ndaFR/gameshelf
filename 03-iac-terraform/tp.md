# TP 03 - Infrastructure as Code avec Terraform + Docker

## Contexte GameShelf

GameShelf souhaite automatiser le déploiement de son infrastructure. Nous allons découvrir Terraform, un outil d'Infrastructure as Code, en utilisant Docker pour simuler notre infrastructure localement.

**Objectif de ce TP :** Comprendre les concepts de Terraform en déployant des conteneurs Docker.

---

## Prérequis

- Docker installé et fonctionnel
- Terraform installé
- Terminal (Linux/macOS ou WSL2 sur Windows)
- Éditeur de texte (VS Code recommandé)

---

## Partie 1 : Concepts de l'Infrastructure as Code

### Qu'est-ce que l'IaC ?

L'Infrastructure as Code consiste à décrire son infrastructure dans des fichiers de configuration versionnables.

**Avantages :**
- **Reproductibilité** : Même résultat à chaque exécution
- **Versionnement** : Historique des changements avec Git
- **Collaboration** : Revue de code, merge requests
- **Documentation** : Le code EST la documentation

### Terraform vs Docker Compose

| Aspect | Terraform | Docker Compose |
|--------|-----------|----------------|
| Scope | Multi-cloud (AWS, GCP, Azure, Docker...) | Docker uniquement |
| État | Fichier tfstate | Pas de gestion d'état |
| Langage | HCL | YAML |
| Cas d'usage | Infrastructure complète | Conteneurs locaux |

> Dans ce TP, nous utilisons Terraform avec Docker pour apprendre les concepts. En production cloud, seul le provider change !

---

## Partie 2 : Premier projet Terraform

### Étape 2.1 : Créer le dossier du projet

```bash
mkdir -p ~/gameshelf-terraform-docker
cd ~/gameshelf-terraform-docker
```

### Étape 2.2 : Créer le fichier versions.tf

```bash
nano versions.tf
```

```hcl
terraform {
  required_version = ">= 1.0.0"

  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {
  # Utilise le socket Docker par défaut
  # unix:///var/run/docker.sock
}
```

### Étape 2.3 : Créer le fichier variables.tf

```bash
nano variables.tf
```

```hcl
variable "container_name" {
  description = "Nom du conteneur"
  type        = string
  default     = "gameshelf-web"
}

variable "external_port" {
  description = "Port externe (accessible depuis le navigateur)"
  type        = number
  default     = 8080
}

variable "internal_port" {
  description = "Port interne du conteneur"
  type        = number
  default     = 80
}

variable "nginx_image" {
  description = "Image Docker à utiliser"
  type        = string
  default     = "nginx:alpine"
}
```

### Étape 2.4 : Créer le fichier main.tf

```bash
nano main.tf
```

```hcl
# Télécharger l'image Nginx
resource "docker_image" "nginx" {
  name         = var.nginx_image
  keep_locally = true
}

# Créer un réseau Docker
resource "docker_network" "gameshelf_network" {
  name = "gameshelf-network"
}

# Créer le conteneur web
resource "docker_container" "web" {
  name  = var.container_name
  image = docker_image.nginx.image_id

  # Mapping de port
  ports {
    internal = var.internal_port
    external = var.external_port
  }

  # Connecter au réseau
  networks_advanced {
    name = docker_network.gameshelf_network.name
  }

  # Variables d'environnement
  env = [
    "NGINX_HOST=gameshelf.local",
    "NGINX_PORT=80"
  ]

  # Labels (équivalent des métadonnées cloud)
  labels {
    label = "project"
    value = "gameshelf"
  }

  labels {
    label = "environment"
    value = "development"
  }

  labels {
    label = "managed_by"
    value = "terraform"
  }

  # Redémarrage automatique
  restart = "unless-stopped"

  # Monter un volume pour la page personnalisée
  volumes {
    host_path      = "${path.cwd}/html"
    container_path = "/usr/share/nginx/html"
    read_only      = true
  }
}
```

### Étape 2.5 : Créer le fichier outputs.tf

```bash
nano outputs.tf
```

```hcl
output "container_id" {
  description = "ID du conteneur créé"
  value       = docker_container.web.id
}

output "container_name" {
  description = "Nom du conteneur"
  value       = docker_container.web.name
}

output "container_ip" {
  description = "Adresse IP du conteneur"
  value       = docker_container.web.network_data[0].ip_address
}

output "access_url" {
  description = "URL pour accéder au site"
  value       = "http://localhost:${var.external_port}"
}

output "network_name" {
  description = "Nom du réseau Docker"
  value       = docker_network.gameshelf_network.name
}

output "validation_flag" {
  description = "Flag de validation du TP"
  value       = "FLAG{Dk4rT9mN2xPv7wQz_TERRAFORM_DOCKER}"
}
```

### Étape 2.6 : Créer la page HTML personnalisée

```bash
mkdir -p html
nano html/index.html
```

```html
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>GameShelf - Infrastructure Terraform</title>
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
            background: #3498db;
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
        <h1>GameShelf</h1>
        <p>Infrastructure déployée avec Terraform</p>

        <div>
            <span class="badge">Docker</span>
            <span class="badge">Terraform</span>
            <span class="badge">Nginx</span>
        </div>

        <div class="status">
            Conteneur opérationnel
        </div>

        <div class="info">
            <strong>Détails de l'infrastructure :</strong><br>
            - Provider: Docker<br>
            - Image: nginx:alpine<br>
            - Réseau: gameshelf-network<br>
            - Géré par: Terraform
        </div>

        <p class="footer">
            TP03 - Infrastructure as Code | Master 1 Cloud Computing
        </p>

        <!-- FLAG{Dk4rT9mN2xPv7wQz_TERRAFORM_DOCKER} -->
    </div>
</body>
</html>
```

---

## Partie 3 : Déploiement avec Terraform

### Étape 3.1 : Initialiser le projet

```bash
terraform init
```

Vous devriez voir :
```
Initializing the backend...
Initializing provider plugins...
- Finding kreuzwerker/docker versions matching "~> 3.0"...
- Installing kreuzwerker/docker v3.0.2...

Terraform has been successfully initialized!
```

### Étape 3.2 : Valider la syntaxe

```bash
terraform validate
```

### Étape 3.3 : Prévisualiser les changements

```bash
terraform plan
```

Analysez le résultat :
- `docker_image.nginx` sera créé
- `docker_network.gameshelf_network` sera créé
- `docker_container.web` sera créé

### Étape 3.4 : Appliquer les changements

```bash
terraform apply
```

Tapez `yes` quand demandé.

### Étape 3.5 : Vérifier le déploiement

```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.

Outputs:

access_url = "http://localhost:8080"
container_id = "abc123..."
container_ip = "172.18.0.2"
container_name = "gameshelf-web"
network_name = "gameshelf-network"
validation_flag = "FLAG{Dk4rT9mN2xPv7wQz_TERRAFORM_DOCKER}"
```

### Étape 3.6 : Tester dans le navigateur

Ouvrez votre navigateur à l'adresse : **http://localhost:8080**

Vous devriez voir la page GameShelf !

---

## Partie 4 : Exploration Docker

### Étape 4.1 : Voir les conteneurs

```bash
docker ps
```

Vous verrez votre conteneur `gameshelf-web`.

### Étape 4.2 : Voir les réseaux

```bash
docker network ls
```

Vous verrez le réseau `gameshelf-network`.

### Étape 4.3 : Inspecter le conteneur

```bash
docker inspect gameshelf-web
```

Retrouvez les labels définis dans Terraform.

### Étape 4.4 : Voir les logs

```bash
docker logs gameshelf-web
```

---

## Partie 5 : Modification de l'infrastructure

### Étape 5.1 : Changer le port

Modifiez `variables.tf` :

```hcl
variable "external_port" {
  default = 9090  # Changé de 8080 à 9090
}
```

### Étape 5.2 : Appliquer le changement

```bash
terraform plan
```

Observez : Terraform veut **détruire et recréer** le conteneur (le port ne peut pas être changé à chaud).

```bash
terraform apply
```

### Étape 5.3 : Vérifier

Le site est maintenant accessible sur **http://localhost:9090**

---

## Partie 6 : Nettoyage

### Étape 6.1 : Détruire l'infrastructure

```bash
terraform destroy
```

Tapez `yes` pour confirmer.

### Étape 6.2 : Vérifier

```bash
docker ps
docker network ls
```

Le conteneur et le réseau ont disparu !

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé les fichiers Terraform
- [ ] Exécuté `terraform init` avec succès
- [ ] Exécuté `terraform apply` et déployé le conteneur
- [ ] Accédé au site sur http://localhost:8080
- [ ] Modifié le port et réappliqué
- [ ] Exécuté `terraform destroy`

---

## Flag de validation

Le flag apparaît dans :
1. Les outputs Terraform après `apply`
2. Le code source de la page HTML (Ctrl+U)

```
FLAG{Dk4rT9mN2xPv7wQz_TERRAFORM_DOCKER}
```

---

## Transition vers le TP suivant

*"Excellent ! Nous savons maintenant déployer de l'infrastructure avec Terraform. Mais notre conteneur utilise l'image Nginx par défaut... Comment automatiser la configuration de Nginx avec notre propre contenu ? C'est là qu'Ansible entre en jeu !"*

---

## Structure du projet final

```
gameshelf-terraform-docker/
├── versions.tf          # Provider Docker
├── variables.tf         # Variables configurables
├── main.tf              # Infrastructure (conteneur, réseau)
├── outputs.tf           # Valeurs de sortie
├── html/
│   └── index.html       # Page web personnalisée
├── terraform.tfstate    # État (généré)
└── .terraform/          # Plugins (généré)
```

---

## En cas de problème

| Problème | Solution |
|----------|----------|
| "Cannot connect to Docker daemon" | Lancez Docker Desktop ou `sudo systemctl start docker` |
| "Permission denied" sur le socket | Ajoutez-vous au groupe docker : `sudo usermod -aG docker $USER` |
| Port déjà utilisé | Changez `external_port` dans variables.tf |
| WSL2 ne voit pas Docker | Activez l'intégration WSL dans Docker Desktop |

---

**Difficulté : Intermédiaire**
