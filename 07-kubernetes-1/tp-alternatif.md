# TP 07 (Alternatif) - Kubernetes avec Minikube

## Version alternative - Cluster Kubernetes local

> **Note :** Cette version utilise Minikube pour créer un cluster Kubernetes local. Utilisez-la si vous n'avez pas accès à OVH.

---

## Contexte GameShelf

Notre API est conteneurisée, mais comment la déployer à grande échelle ? Comment gérer plusieurs réplicas, les mises à jour sans interruption, et l'auto-réparation en cas de panne ? C'est là qu'intervient Kubernetes.

**Objectif de ce TP :** Déployer l'API GameShelf sur un cluster Kubernetes local.

---

## Prérequis

- TP 06 complété (Docker + image GameShelf)
- Docker installé et fonctionnel
- 4 Go de RAM minimum disponibles

---

## Partie 1 : Introduction à Kubernetes

### Qu'est-ce que Kubernetes ?

Kubernetes (K8s) est un orchestrateur de conteneurs qui automatise :
- Le déploiement
- La mise à l'échelle
- La gestion des conteneurs

### Architecture Kubernetes

```
┌─────────────────────────────────────────────────────────────┐
│                     CLUSTER KUBERNETES                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │                   CONTROL PLANE                      │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────┐  │    │
│  │  │API Server│ │  etcd    │ │Scheduler │ │  CCM   │  │    │
│  │  └──────────┘ └──────────┘ └──────────┘ └────────┘  │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   NODE 1    │  │   NODE 2    │  │   NODE 3    │          │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │          │
│  │ │  Pod A  │ │  │ │  Pod B  │ │  │ │  Pod C  │ │          │
│  │ │  Pod D  │ │  │ │  Pod E  │ │  │ │  Pod F  │ │          │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │          │
│  │   kubelet   │  │   kubelet   │  │   kubelet   │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
└─────────────────────────────────────────────────────────────┘
```

### Concepts clés

| Concept | Description |
|---------|-------------|
| **Pod** | Plus petite unité déployable (1+ conteneurs) |
| **Deployment** | Gère les réplicas et mises à jour des Pods |
| **Service** | Expose les Pods avec une IP stable |
| **Namespace** | Isolation logique des ressources |

---

## Partie 2 : Installation de Minikube

### Étape 2.1 : Installation

**Linux/WSL :**
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

**macOS :**
```bash
brew install minikube
```

### Étape 2.2 : Installation de kubectl

**Linux/WSL :**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

**macOS :**
```bash
brew install kubectl
```

### Étape 2.3 : Démarrer Minikube

```bash
minikube start --driver=docker
minikube status
kubectl get nodes
```

---

## Partie 3 : Déploiement de l'API

### Étape 3.1 : Créer le projet

```bash
mkdir -p ~/gameshelf-k8s
cd ~/gameshelf-k8s
```

### Étape 3.2 : Namespace

```bash
nano namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gameshelf
```

```bash
kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=gameshelf
```

### Étape 3.3 : Deployment

```bash
nano deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gameshelf-api
  namespace: gameshelf
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gameshelf-api
  template:
    metadata:
      labels:
        app: gameshelf-api
    spec:
      containers:
      - name: api
        image: VOTRE_USERNAME/gameshelf-api:latest
        ports:
        - containerPort: 5000
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 10
```

```bash
kubectl apply -f deployment.yaml
kubectl get pods
```

### Étape 3.4 : Service

```bash
nano service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gameshelf-api
  namespace: gameshelf
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 5000
  selector:
    app: gameshelf-api
```

```bash
kubectl apply -f service.yaml
```

---

## Partie 4 : Accès à l'API

### Port-forward

```bash
kubectl port-forward service/gameshelf-api 8080:80
```

Dans un autre terminal :
```bash
curl http://localhost:8080/
curl http://localhost:8080/flag
```

---

## Partie 5 : Scaling et auto-réparation

```bash
# Scaler
kubectl scale deployment gameshelf-api --replicas=5
kubectl get pods

# Auto-réparation
kubectl delete pod <NOM_DU_POD>
kubectl get pods  # Un nouveau pod est créé
```

---

## Flag de validation

```
FLAG{K8s_M1n1kub3_L0c4l_Cl0ud}
```

---

## Nettoyage

```bash
minikube stop
minikube delete
```

---

**Durée estimée : 1 heure**

**Difficulté : Intermédiaire**
