# TP1 - Configuration du Poste de Travail

## Objectifs

- Configurer un environnement de développement DevOps/Cloud
- Installer les outils essentiels : kubectl, git, psql, terraform, vault, curl, ping, docker
- Générer une clé SSH pour l'authentification

---

## 1. Configuration selon votre système d'exploitation

### Option A : Linux / macOS

Passez directement à la section [2. Installation des outils](#2-installation-des-outils).

### Option B : Windows (WSL2)

#### 1.1 Activer WSL2

Ouvrez PowerShell en tant qu'administrateur et exécutez :

```powershell
wsl --install
```

Cette commande installe WSL2 avec Ubuntu par défaut.

#### 1.2 Redémarrer le PC

Redémarrez votre ordinateur pour finaliser l'installation.

#### 1.3 Configurer Ubuntu

Au premier lancement, créez votre utilisateur et mot de passe Linux.

#### 1.4 Mettre à jour le système

```bash
sudo apt update && sudo apt upgrade -y
```

> **Note** : Pour la suite du TP, toutes les commandes seront exécutées dans le terminal WSL2 (Ubuntu).

---

## 2. Installation des outils

### 2.1 Outils de base (curl, ping, git)

#### Linux (Debian/Ubuntu) / WSL2

```bash
sudo apt update
sudo apt install -y curl iputils-ping git
```

#### macOS

```bash
# curl et ping sont préinstallés
# Installer git via Xcode Command Line Tools
xcode-select --install
```

Vérification :

```bash
curl --version
ping -c 1 google.com
git --version
```

---

### 2.2 Docker

#### Linux (Debian/Ubuntu) / WSL2

```bash
# Ajouter le dépôt Docker
sudo apt install -y ca-certificates gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Installer Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Ajouter l'utilisateur au groupe docker
sudo usermod -aG docker $USER
newgrp docker
```

#### macOS

Téléchargez et installez [Docker Desktop pour Mac](https://www.docker.com/products/docker-desktop/).

Vérification :

```bash
docker --version
docker run hello-world
```

---

### 2.3 kubectl

#### Linux / WSL2

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

#### macOS

```bash
# Avec Homebrew
brew install kubectl

# Ou manuellement (Apple Silicon)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
```

Vérification :

```bash
kubectl version --client
```

---

### 2.4 PostgreSQL Client (psql)

#### Linux (Debian/Ubuntu) / WSL2

```bash
sudo apt install -y postgresql-client
```

#### macOS

```bash
brew install libpq
echo 'export PATH="/opt/homebrew/opt/libpq/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Vérification :

```bash
psql --version
```

---

### 2.5 Terraform

#### Linux / WSL2

```bash
sudo apt install -y gnupg software-properties-common

wget -O- https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update
sudo apt install -y terraform

# Manual install (fallback)
TERRAFORM_VERSION=1.6.6

wget https://releases.hashicorp.com/terraform/${TERRAFORM_VERSION}/terraform_${TERRAFORM_VERSION}_linux_amd64.zip
unzip terraform_${TERRAFORM_VERSION}_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform version
```

#### macOS

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

Vérification :

```bash
terraform --version
```

---

### 2.6 Vault CLI

#### Linux / WSL2

```bash
# Le dépôt HashiCorp est déjà configuré (étape précédente)
sudo apt install -y vault

# Manual install (fallback)
wget https://releases.hashicorp.com/vault/1.15.6/vault_1.15.6_linux_amd64.zip
unzip vault_1.15.6_linux_amd64.zip
sudo mv vault /usr/local/bin/
vault version
```

#### macOS

```bash
brew install hashicorp/tap/vault
```

Vérification :

```bash
vault --version
```

---

## 3. Génération de la clé SSH

### 3.1 Générer une nouvelle clé SSH

```bash
ssh-keygen -t ed25519 -C "votre.email@exemple.com"
```

Lors de l'exécution :
- **Emplacement** : Appuyez sur Entrée pour accepter l'emplacement par défaut (`~/.ssh/id_ed25519`)
- **Passphrase** : Entrez une passphrase sécurisée (recommandé) ou laissez vide

### 3.2 Démarrer l'agent SSH et ajouter la clé

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

### 3.3 Afficher la clé publique

```bash
cat ~/.ssh/id_ed25519.pub
```

> **Important** : Copiez cette clé publique. Vous l'utiliserez pour vous authentifier sur GitHub, GitLab, ou d'autres services.

---

## 4. Récapitulatif - Vérification finale

Exécutez ce script pour vérifier que tous les outils sont installés :

```bash
echo "=== Vérification de l'environnement ==="
echo ""
echo "curl:      $(curl --version 2>/dev/null | head -n1 || echo 'NON INSTALLÉ')"
echo "ping:      $(ping -c 1 127.0.0.1 >/dev/null 2>&1 && echo 'OK' || echo 'NON INSTALLÉ')"
echo "git:       $(git --version 2>/dev/null || echo 'NON INSTALLÉ')"
echo "docker:    $(docker --version 2>/dev/null || echo 'NON INSTALLÉ')"
echo "kubectl:   $(kubectl version --client --short 2>/dev/null || kubectl version --client 2>/dev/null | head -n1 || echo 'NON INSTALLÉ')"
echo "psql:      $(psql --version 2>/dev/null || echo 'NON INSTALLÉ')"
echo "terraform: $(terraform --version 2>/dev/null | head -n1 || echo 'NON INSTALLÉ')"
echo "vault:     $(vault --version 2>/dev/null || echo 'NON INSTALLÉ')"
echo ""
echo "Clé SSH:   $(ls ~/.ssh/id_ed25519.pub 2>/dev/null && echo 'OK' || echo 'NON GÉNÉRÉE')"
```

---

## Checklist

- [ ] Environnement Linux/macOS ou WSL2 configuré
- [ ] curl installé
- [ ] ping fonctionnel
- [ ] git installé
- [ ] Docker installé et fonctionnel
- [ ] kubectl installé
- [ ] psql installé
- [ ] Terraform installé
- [ ] Vault CLI installé
- [ ] Clé SSH générée

---

## Flag de validation

Une fois toutes les étapes complétées avec succès :

```
FLAG{TP1_ENV_CONFIGURED_2024}
```
