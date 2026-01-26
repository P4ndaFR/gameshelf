# TP 07 - Kubernetes (1/2) : Deploiement sur cluster partage

## Version cluster partage

> **Note :** Un cluster Kubernetes OVH a ete pre-provisionne par l'enseignant. Chaque etudiant deploiera son application dans son propre namespace et l'exposera via un Ingress.

---

## Contexte GameShelf

Notre API est conteneurisee, mais comment la deployer a grande echelle ? Comment gerer plusieurs replicas, les mises a jour sans interruption, et l'auto-reparation en cas de panne ? C'est la qu'intervient Kubernetes.

**Objectif de ce TP :** Deployer l'API GameShelf sur un cluster Kubernetes partage.

---

## Prerequis

- TP 06 complete (Docker + image sur Docker Hub)
- Terminal avec kubectl installe
- Fichier kubeconfig fourni par l'enseignant

---

## Partie 1 : Introduction a Kubernetes

### Qu'est-ce que Kubernetes ?

Kubernetes (K8s) est un orchestrateur de conteneurs qui automatise :
- Le deploiement de conteneurs
- La mise a l'echelle automatique
- L'auto-reparation en cas de panne
- La repartition de charge

### Architecture du cluster partage

```
┌─────────────────────────────────────────────────────────────────┐
│                  CLUSTER KUBERNETES OVH                          │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────────────────────────────────────────────────┐    │
│  │         CONTROL PLANE (Gere par OVH)                    │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   NODE 1    │  │   NODE 2    │  │   NODE 3    │              │
│  └─────────────┘  └─────────────┘  └─────────────┘              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    INGRESS CONTROLLER                    │    │
│  │              (Entree unique pour tous)                   │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌───────────┐ ┌───────────┐ ┌───────────┐ ┌───────────┐        │
│  │ ns-alice  │ │ ns-bob    │ │ ns-carol  │ │ ns-david  │  ...   │
│  │  ┌─────┐  │ │  ┌─────┐  │ │  ┌─────┐  │ │  ┌─────┐  │        │
│  │  │ Pod │  │ │  │ Pod │  │ │  │ Pod │  │ │  │ Pod │  │        │
│  │  └─────┘  │ │  └─────┘  │ │  └─────┘  │ │  └─────┘  │        │
│  └───────────┘ └───────────┘ └───────────┘ └───────────┘        │
└─────────────────────────────────────────────────────────────────┘
```

### Concepts cles

| Concept | Description |
|---------|-------------|
| **Pod** | Plus petite unite deployable (1+ conteneurs) |
| **Deployment** | Gere les replicas et mises a jour des Pods |
| **Service** | Expose les Pods avec une IP stable |
| **Namespace** | Isolation logique des ressources (votre espace) |
| **Ingress** | Route le trafic HTTP vers les Services |

---

## Partie 2 : Configuration de kubectl

### Etape 2.1 : Installation de kubectl

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

### Etape 2.2 : Configurer le kubeconfig

L'enseignant vous fournira un fichier kubeconfig. Sauvegardez-le :

```bash
mkdir -p ~/.kube
# Copiez le contenu du kubeconfig fourni dans ce fichier
nano ~/.kube/config
chmod 600 ~/.kube/config
```

### Etape 2.3 : Verifier la connexion

```bash
# Infos du cluster
kubectl cluster-info

# Voir les nodes
kubectl get nodes
```

Vous devriez voir les nodes du cluster en statut **Ready**.

---

## Partie 3 : Votre namespace

### Etape 3.1 : Determiner votre namespace

Votre namespace est base sur votre email universitaire :
- Email : `prenom.nom@universite.fr`
- Namespace : `prenom-nom` (sans le domaine, points remplaces par des tirets)

Exemple :
- `jean.dupont@univ.fr` -> namespace `jean-dupont`

```bash
# Definissez votre namespace
export MY_NS="prenom-nom"  # Remplacez par votre namespace
```

### Etape 3.2 : Verifier votre namespace

```bash
kubectl get namespace $MY_NS
```

Si le namespace n'existe pas, creez-le :

```bash
kubectl create namespace $MY_NS
```

### Etape 3.3 : Configurer kubectl pour utiliser votre namespace

```bash
kubectl config set-context --current --namespace=$MY_NS
```

Maintenant toutes vos commandes kubectl utiliseront votre namespace par defaut.

---

## Partie 4 : Deploiement de l'API

### Etape 4.1 : Creer le dossier du projet

```bash
mkdir -p ~/gameshelf-k8s
cd ~/gameshelf-k8s
```

### Etape 4.2 : Creer le Deployment

Creez le fichier `deployment.yaml` :

```bash
nano deployment.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gameshelf-api
  labels:
    app: gameshelf-api
spec:
  replicas: 2
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
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
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

> **Important :** Remplacez `VOTRE_DOCKERHUB_USERNAME` par votre username Docker Hub.

### Etape 4.3 : Appliquer le Deployment

```bash
kubectl apply -f deployment.yaml
```

### Etape 4.4 : Verifier le deploiement

```bash
# Voir le deployment
kubectl get deployments

# Voir les pods
kubectl get pods

# Voir les pods avec plus de details
kubectl get pods -o wide

# Suivre la creation en temps reel
kubectl get pods -w
```

---

## Partie 5 : Exposition avec un Service

### Etape 5.1 : Creer le Service

Creez le fichier `service.yaml` :

```bash
nano service.yaml
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: gameshelf-api
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

---

## Partie 6 : Exposition via Ingress

### Etape 6.1 : Comprendre l'Ingress

L'Ingress permet d'exposer votre application sur Internet via le domaine du cluster. Chaque etudiant a un chemin unique base sur son namespace.

URL de votre application : `http://[DOMAINE_CLUSTER]/[VOTRE_NAMESPACE]/`

### Etape 6.2 : Creer l'Ingress

Creez le fichier `ingress.yaml` :

```bash
nano ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gameshelf-api
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /VOTRE_NAMESPACE(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: gameshelf-api
            port:
              number: 80
```

> **Important :** Remplacez `VOTRE_NAMESPACE` par votre namespace (ex: `jean-dupont`).

```bash
kubectl apply -f ingress.yaml
```

### Etape 6.3 : Verifier l'Ingress

```bash
kubectl get ingress
```

### Etape 6.4 : Tester l'acces

L'enseignant vous fournira l'adresse du cluster. Testez :

```bash
# Remplacez par les vraies valeurs
export CLUSTER_URL="http://[ADRESSE_CLUSTER]"
export MY_NS="votre-namespace"

curl $CLUSTER_URL/$MY_NS/
curl $CLUSTER_URL/$MY_NS/health
curl $CLUSTER_URL/$MY_NS/games
curl $CLUSTER_URL/$MY_NS/flag
```

Ou dans votre navigateur : `http://[ADRESSE_CLUSTER]/votre-namespace/`

---

## Partie 7 : Scaling

### Etape 7.1 : Augmenter les replicas

```bash
kubectl scale deployment gameshelf-api --replicas=3
kubectl get pods
```

### Etape 7.2 : Observer la repartition

```bash
# Faire plusieurs requetes
for i in {1..10}; do
  curl -s $CLUSTER_URL/$MY_NS/ | grep -o '"hostname":"[^"]*"'
done
```

Les requetes sont reparties entre les pods.

### Etape 7.3 : Reduire les replicas

```bash
kubectl scale deployment gameshelf-api --replicas=2
kubectl get pods
```

---

## Partie 8 : Auto-reparation

### Etape 8.1 : Simuler une panne

```bash
# Lister les pods
kubectl get pods

# Supprimer un pod (simuler un crash)
kubectl delete pod <NOM_DU_POD>

# Observer la recreation immediate
kubectl get pods -w
```

Kubernetes recree automatiquement le pod pour maintenir le nombre de replicas !

### Etape 8.2 : Voir les evenements

```bash
kubectl get events --sort-by=.metadata.creationTimestamp | tail -20
```

---

## Partie 9 : Commandes utiles

### Voir les logs

```bash
# Logs d'un pod
kubectl logs <NOM_DU_POD>

# Logs de tous les pods
kubectl logs -l app=gameshelf-api

# Logs en temps reel
kubectl logs -f <NOM_DU_POD>
```

### Executer une commande dans un pod

```bash
kubectl exec -it <NOM_DU_POD> -- sh
```

### Decrire une ressource

```bash
kubectl describe pod <NOM_DU_POD>
kubectl describe deployment gameshelf-api
kubectl describe service gameshelf-api
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Configure kubectl avec le kubeconfig du cluster
- [ ] Travaille dans votre namespace personnel
- [ ] Deploye l'API avec un Deployment (2 replicas)
- [ ] Cree un Service ClusterIP
- [ ] Cree un Ingress pour exposer l'API
- [ ] Teste l'acces via l'URL du cluster
- [ ] Scale le Deployment
- [ ] Observe l'auto-reparation
- [ ] Recupere le flag via `/flag`

---

## Flag de validation

Accedez a l'endpoint `/flag` :

```bash
curl $CLUSTER_URL/$MY_NS/flag
```

**Soumettez ce flag** sur la plateforme du cours.

---

## Transition vers le TP suivant

*"Notre API tourne sur Kubernetes ! Mais comment faire une mise a jour sans interruption ? Comment externaliser la configuration ? Dans le prochain TP, nous verrons les Rolling Updates, ConfigMaps et les bonnes pratiques !"*

---

## Structure du projet

```
gameshelf-k8s/
├── deployment.yaml     # Deployment de l'API
├── service.yaml        # Service ClusterIP
└── ingress.yaml        # Ingress pour l'exposition
```

---

## Commandes kubectl essentielles

| Commande | Description |
|----------|-------------|
| `kubectl get pods` | Lister les pods |
| `kubectl get services` | Lister les services |
| `kubectl get ingress` | Lister les ingress |
| `kubectl describe pod <name>` | Details d'un pod |
| `kubectl logs <pod>` | Voir les logs |
| `kubectl exec -it <pod> -- sh` | Shell dans un pod |
| `kubectl scale deploy <name> --replicas=N` | Scaler |
| `kubectl delete pod <name>` | Supprimer un pod |

---

## En cas de probleme

| Probleme | Solution |
|----------|----------|
| kubeconfig ne fonctionne pas | Verifiez le chemin ~/.kube/config |
| Pods en "Pending" | Pas assez de ressources, reduisez les replicas |
| Pods en "ImagePullBackOff" | Verifiez le nom de l'image Docker Hub |
| Ingress ne fonctionne pas | Verifiez le chemin et le namespace dans ingress.yaml |
| 404 sur l'URL | Verifiez que le path dans l'Ingress correspond a votre namespace |

---

**Difficulte : Intermediaire**
