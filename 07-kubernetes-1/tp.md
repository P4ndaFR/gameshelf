# TP 07 - Kubernetes (1/2) : Premiers pas avec OVH Managed Kubernetes

## Contexte GameShelf

Notre API est conteneurisée, mais comment la déployer à grande échelle ? Comment gérer plusieurs réplicas, les mises à jour sans interruption, et l'auto-réparation en cas de panne ? C'est là qu'intervient Kubernetes.

**Objectif de ce TP :** Déployer l'API GameShelf sur un cluster Kubernetes managé OVH.

---

## Prérequis

- TP 06 complété (Docker + image sur Docker Hub)
- Compte OVH avec projet Public Cloud actif
- Terminal avec kubectl installé

---

## Partie 1 : Introduction à Kubernetes

### Qu'est-ce que Kubernetes ?

Kubernetes (K8s) est un orchestrateur de conteneurs qui automatise :
- Le déploiement de conteneurs
- La mise à l'échelle automatique
- L'auto-réparation en cas de panne
- La répartition de charge

### Pourquoi un Kubernetes managé ?

| Aspect | K8s auto-géré | K8s managé (OVH) |
|--------|---------------|------------------|
| Installation Control Plane | Vous | OVH |
| Mises à jour | Vous | OVH |
| Haute dispo Control Plane | Vous | OVH |
| Monitoring Control Plane | Vous | OVH |
| Coût | Moins cher | Plus simple |

### Architecture Kubernetes

```
┌─────────────────────────────────────────────────────────────┐
│                     CLUSTER KUBERNETES                       │
├─────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────┐    │
│  │           CONTROL PLANE (Géré par OVH)              │    │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐             │    │
│  │  │API Server│ │  etcd    │ │Scheduler │             │    │
│  │  └──────────┘ └──────────┘ └──────────┘             │    │
│  └─────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐          │
│  │   NODE 1    │  │   NODE 2    │  │   NODE 3    │          │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │          │
│  │ │  Pod A  │ │  │ │  Pod B  │ │  │ │  Pod C  │ │          │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │          │
│  │   kubelet   │  │   kubelet   │  │   kubelet   │          │
│  └─────────────┘  └─────────────┘  └─────────────┘          │
│        ↑                                                     │
│   (Vous gérez les nodes)                                     │
└─────────────────────────────────────────────────────────────┘
```

### Concepts clés

| Concept | Description |
|---------|-------------|
| **Pod** | Plus petite unité déployable (1+ conteneurs) |
| **Deployment** | Gère les réplicas et mises à jour des Pods |
| **Service** | Expose les Pods avec une IP stable |
| **Namespace** | Isolation logique des ressources |
| **Node Pool** | Groupe de VMs qui exécutent les Pods |

---

## Partie 2 : Création du cluster Kubernetes OVH

### Étape 2.1 : Accéder à la console OVH

1. Connectez-vous au [Manager OVH](https://www.ovh.com/manager/)

2. Allez dans **Public Cloud > GameShelf-Infra**

3. Dans le menu de gauche, cliquez sur **Containers and Orchestration > Managed Kubernetes Service**

### Étape 2.2 : Créer un nouveau cluster

1. Cliquez sur **"Créer un cluster Kubernetes"**

2. **Configuration du cluster :**
   - **Nom** : `gameshelf-k8s`
   - **Région** : GRA (Gravelines) ou SBG (Strasbourg)
   - **Version Kubernetes** : Choisissez la dernière version stable (ex: 1.28)

3. Cliquez sur **"Suivant"**

### Étape 2.3 : Configuration du Node Pool

1. **Nom du pool** : `worker-pool`

2. **Type de node** : Choisissez **b2-7** (petit) ou **d2-4** (encore plus petit)
   - b2-7 : 2 vCPU, 7 Go RAM
   - d2-4 : 2 vCPU, 4 Go RAM

3. **Nombre de nodes** : **2** (minimum pour la HA)

4. **Autoscaling** : Désactivé (pour ce TP)

5. Cliquez sur **"Suivant"**

### Étape 2.4 : Configuration réseau

1. **Réseau privé** : Laissez par défaut (Ext-Net)

2. Cliquez sur **"Créer"**

### Étape 2.5 : Attendre la création

Le cluster met **5-10 minutes** à être créé. Vous pouvez suivre l'avancement dans le Manager.

Statut final attendu : **Ready**

---

## Partie 3 : Configuration de kubectl

### Étape 3.1 : Installation de kubectl

**Linux/WSL :**
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
kubectl version --client
```

**macOS :**
```bash
brew install kubectl
```

### Étape 3.2 : Télécharger le kubeconfig

1. Dans le Manager OVH, cliquez sur votre cluster `gameshelf-k8s`

2. Allez dans l'onglet **"Service"**

3. Cliquez sur **"kubeconfig"** pour télécharger le fichier

4. Sauvegardez-le dans `~/.kube/config` :

```bash
mkdir -p ~/.kube
mv ~/Téléchargements/kubeconfig.yml ~/.kube/config
chmod 600 ~/.kube/config
```

### Étape 3.3 : Vérifier la connexion

```bash
# Infos du cluster
kubectl cluster-info

# Voir les nodes
kubectl get nodes

# Version du cluster
kubectl version
```

Vous devriez voir vos 2 nodes en statut **Ready**.

---

## Partie 4 : Premier déploiement

### Étape 4.1 : Créer le dossier du projet

```bash
mkdir -p ~/gameshelf-k8s
cd ~/gameshelf-k8s
```

### Étape 4.2 : Créer un namespace

```bash
nano namespace.yaml
```

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: gameshelf
  labels:
    project: gameshelf
    environment: development
```

```bash
kubectl apply -f namespace.yaml
kubectl get namespaces
```

### Étape 4.3 : Configurer le namespace par défaut

```bash
kubectl config set-context --current --namespace=gameshelf
```

---

## Partie 5 : Déploiement de l'API

### Étape 5.1 : Créer le Deployment

```bash
nano deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gameshelf-api
  namespace: gameshelf
  labels:
    app: gameshelf-api
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: gameshelf-api
  template:
    metadata:
      labels:
        app: gameshelf-api
        version: v1
    spec:
      containers:
      - name: api
        image: VOTRE_DOCKERHUB_USERNAME/gameshelf-api:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
          name: http
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
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
          timeoutSeconds: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 3
```

> **Important :** Remplacez `VOTRE_DOCKERHUB_USERNAME` par votre username Docker Hub.

### Étape 5.2 : Appliquer le Deployment

```bash
kubectl apply -f deployment.yaml
```

### Étape 5.3 : Vérifier le déploiement

```bash
# Voir le deployment
kubectl get deployments

# Voir les pods
kubectl get pods

# Voir les pods avec plus de détails
kubectl get pods -o wide

# Suivre la création en temps réel
kubectl get pods -w
```

### Étape 5.4 : Voir les logs

```bash
# Logs d'un pod
kubectl logs <NOM_DU_POD>

# Logs de tous les pods
kubectl logs -l app=gameshelf-api

# Logs en temps réel
kubectl logs -f <NOM_DU_POD>
```

### Étape 5.5 : Décrire un pod

```bash
kubectl describe pod <NOM_DU_POD>
```

---

## Partie 6 : Exposition avec un Service

### Étape 6.1 : Créer un Service ClusterIP

```bash
nano service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gameshelf-api
  namespace: gameshelf
  labels:
    app: gameshelf-api
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 5000
    protocol: TCP
    name: http
  selector:
    app: gameshelf-api
```

```bash
kubectl apply -f service.yaml
kubectl get services
```

### Étape 6.2 : Créer un Service LoadBalancer

Pour exposer l'API sur Internet, utilisez un LoadBalancer :

```bash
nano service-lb.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gameshelf-api-lb
  namespace: gameshelf
  labels:
    app: gameshelf-api
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 5000
    protocol: TCP
    name: http
  selector:
    app: gameshelf-api
```

```bash
kubectl apply -f service-lb.yaml
```

### Étape 6.3 : Obtenir l'IP externe

```bash
kubectl get service gameshelf-api-lb
```

Attendez que la colonne **EXTERNAL-IP** passe de `<pending>` à une vraie IP.

> **Note :** Cela peut prendre 1-2 minutes car OVH provisionne un Load Balancer.

### Étape 6.4 : Tester l'API

```bash
# Remplacez par votre EXTERNAL-IP
export API_IP=$(kubectl get service gameshelf-api-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "API accessible sur: http://$API_IP"

# Tester
curl http://$API_IP/
curl http://$API_IP/health
curl http://$API_IP/games
curl http://$API_IP/flag
```

---

## Partie 7 : Scaling

### Étape 7.1 : Augmenter les réplicas

```bash
kubectl scale deployment gameshelf-api --replicas=5
kubectl get pods
```

### Étape 7.2 : Observer la répartition

```bash
# Faire plusieurs requêtes
for i in {1..10}; do
  curl -s http://$API_IP/ | jq -r '.hostname'
done
```

Les requêtes sont réparties entre les pods sur les différents nodes.

### Étape 7.3 : Réduire les réplicas

```bash
kubectl scale deployment gameshelf-api --replicas=2
kubectl get pods
```

---

## Partie 8 : Auto-réparation

### Étape 8.1 : Simuler une panne

```bash
# Lister les pods
kubectl get pods

# Supprimer un pod (simuler un crash)
kubectl delete pod <NOM_DU_POD>

# Observer la recréation immédiate
kubectl get pods -w
```

Kubernetes recrée automatiquement le pod pour maintenir le nombre de réplicas !

### Étape 8.2 : Voir les événements

```bash
kubectl get events --sort-by=.metadata.creationTimestamp | tail -20
```

---

## Partie 9 : Explorer via le Dashboard

### Étape 9.1 : Installer le Dashboard (optionnel)

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```

### Étape 9.2 : Créer un token d'accès

```bash
kubectl create serviceaccount dashboard-admin -n kubernetes-dashboard
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kubernetes-dashboard:dashboard-admin

kubectl -n kubernetes-dashboard create token dashboard-admin
```

### Étape 9.3 : Accéder au Dashboard

```bash
kubectl proxy
```

Ouvrez : http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/

Utilisez le token pour vous connecter.

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé un cluster Kubernetes managé sur OVH
- [ ] Configuré kubectl avec le kubeconfig
- [ ] Créé un namespace `gameshelf`
- [ ] Déployé l'API avec un Deployment (3 réplicas)
- [ ] Créé un Service LoadBalancer
- [ ] Testé l'accès à l'API via l'IP externe
- [ ] Scalé le Deployment
- [ ] Observé l'auto-réparation
- [ ] Récupéré le flag via `/flag`

---

## Flag de validation

Accédez à l'endpoint `/flag` :

```bash
curl http://$API_IP/flag
```

```
FLAG{0VH_K8s_M4n4g3d_Cl0ud_Succ3ss}
```

---

## Transition vers le TP suivant

*"Notre API tourne sur Kubernetes managé OVH ! Mais comment faire une mise à jour sans interruption ? Comment externaliser la configuration ? Dans le prochain TP, nous verrons les Rolling Updates, ConfigMaps et Ingress !"*

---

## Structure du projet

```
gameshelf-k8s/
├── namespace.yaml      # Namespace
├── deployment.yaml     # Deployment de l'API
├── service.yaml        # Service ClusterIP
└── service-lb.yaml     # Service LoadBalancer
```

---

## Commandes kubectl essentielles

| Commande | Description |
|----------|-------------|
| `kubectl get pods` | Lister les pods |
| `kubectl get services` | Lister les services |
| `kubectl describe pod <name>` | Détails d'un pod |
| `kubectl logs <pod>` | Voir les logs |
| `kubectl exec -it <pod> -- sh` | Shell dans un pod |
| `kubectl scale deploy <name> --replicas=N` | Scaler |
| `kubectl delete pod <name>` | Supprimer un pod |

---

## Coûts OVH

| Ressource | Prix estimé |
|-----------|-------------|
| Cluster K8s (Control Plane) | Gratuit |
| 2 nodes b2-7 | ~0.06€/h chaque |
| LoadBalancer | ~0.01€/h |

> **Important :** Supprimez les ressources après le TP pour éviter les frais.

---

## Nettoyage (après le cours)

```bash
# Supprimer les ressources K8s
kubectl delete namespace gameshelf

# Dans le Manager OVH :
# 1. Supprimer le cluster Kubernetes
# 2. Vérifier qu'aucun LoadBalancer ne reste actif
```

---

## En cas de problème

| Problème | Solution |
|----------|----------|
| kubeconfig ne fonctionne pas | Vérifiez le chemin ~/.kube/config |
| Pods en "Pending" | Pas assez de ressources sur les nodes |
| Pods en "ImagePullBackOff" | Vérifiez le nom de l'image Docker Hub |
| LoadBalancer en "Pending" | Attendez 1-2 minutes |
| Nodes "NotReady" | Le cluster est encore en cours de création |

---

**Durée estimée : 1 heure**

**Difficulté : Intermédiaire**
