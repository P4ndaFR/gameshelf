# TP 08 - Kubernetes (2/2) : Rolling Updates, ConfigMaps et bonnes pratiques

## Version cluster partage

> **Note :** Ce TP utilise le cluster Kubernetes partage. Vous travaillez dans votre namespace personnel.

---

## Contexte GameShelf

Notre API tourne sur Kubernetes, mais comment deployer une nouvelle version sans interrompre le service ? Comment gerer la configuration externe ?

**Objectif de ce TP :** Maitriser les deploiements sans interruption et la configuration avancee.

---

## Prerequis

- TP 07 complete (API deployee dans votre namespace)
- kubectl configure avec le kubeconfig du cluster
- Votre namespace : `prenom-nom`

---

## Partie 1 : Preparation

### Etape 1.1 : Verifier votre environnement

```bash
# Definir votre namespace
export MY_NS="prenom-nom"  # Remplacez !

# Verifier le contexte
kubectl config set-context --current --namespace=$MY_NS

# Verifier vos ressources
kubectl get all
```

### Etape 1.2 : Creer le dossier de travail

```bash
cd ~/gameshelf-k8s
```

---

## Partie 2 : ConfigMaps - Configuration externalisee

### Pourquoi externaliser la configuration ?

- **Separation code/config** : Une meme image pour tous les environnements
- **Mise a jour sans rebuild** : Changer la config sans reconstruire l'image
- **Flexibilite** : Adapter le comportement par namespace/environnement

### Etape 2.1 : Creer un ConfigMap

```bash
nano configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gameshelf-config
data:
  APP_NAME: "GameShelf"
  APP_VERSION: "2.0.0"
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"
  STUDENT_NAMESPACE: "VOTRE_NAMESPACE"
```

> **Important :** Remplacez `VOTRE_NAMESPACE` par votre namespace reel.

```bash
kubectl apply -f configmap.yaml
kubectl get configmaps
kubectl describe configmap gameshelf-config
```

### Etape 2.2 : Mettre a jour le Deployment avec ConfigMap

```bash
nano deployment-v2.yaml
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
        version: v2
    spec:
      containers:
      - name: api
        image: VOTRE_DOCKERHUB_USERNAME/gameshelf-api:latest
        ports:
        - containerPort: 5000
        # Injecter toutes les cles du ConfigMap comme variables d'env
        envFrom:
        - configMapRef:
            name: gameshelf-config
        # Variables supplementaires
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
          initialDelaySeconds: 5
          periodSeconds: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 5
```

> Remplacez `VOTRE_DOCKERHUB_USERNAME` par votre username.

```bash
kubectl apply -f deployment-v2.yaml
```

### Etape 2.3 : Verifier les variables d'environnement

```bash
# Obtenir le nom d'un pod
POD=$(kubectl get pods -l app=gameshelf-api -o jsonpath='{.items[0].metadata.name}')

# Voir les variables d'environnement
kubectl exec $POD -- env | grep -E "APP_|ENVIRONMENT|LOG_|POD_|STUDENT_"
```

---

## Partie 3 : Secrets - Donnees sensibles

### Etape 3.1 : Creer un Secret

```bash
nano secret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gameshelf-secrets
type: Opaque
stringData:
  DB_PASSWORD: "mon_password_secret"
  API_KEY: "sk-gameshelf-demo-12345"
```

```bash
kubectl apply -f secret.yaml

# Le secret est encode en base64
kubectl get secret gameshelf-secrets -o yaml
```

### Etape 3.2 : Utiliser le Secret (optionnel)

Ajoutez dans la section `env` du Deployment :

```yaml
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: gameshelf-secrets
      key: DB_PASSWORD
```

---

## Partie 4 : Rolling Updates - Mise a jour sans interruption

### Etape 4.1 : Comprendre les strategies

| Strategie | Description |
|-----------|-------------|
| **RollingUpdate** | Remplacement progressif (par defaut) |
| **Recreate** | Tout supprimer puis recreer (downtime) |

### Etape 4.2 : Configurer le Rolling Update

Modifiez votre deployment pour ajouter la strategie :

```bash
nano deployment-rolling.yaml
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
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 pod en plus pendant la mise a jour
      maxUnavailable: 0  # Aucun pod indisponible
  selector:
    matchLabels:
      app: gameshelf-api
  template:
    metadata:
      labels:
        app: gameshelf-api
        version: v2
      annotations:
        # Annotation pour forcer un redeploy
        rollout-timestamp: "TIMESTAMP"
    spec:
      containers:
      - name: api
        image: VOTRE_DOCKERHUB_USERNAME/gameshelf-api:latest
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: gameshelf-config
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
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
          initialDelaySeconds: 5
          periodSeconds: 3
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 10
          periodSeconds: 5
```

### Etape 4.3 : Observer une mise a jour

**Terminal 1 - Surveillez les pods :**
```bash
kubectl get pods -w
```

**Terminal 2 - Declenchez la mise a jour :**
```bash
# Methode 1 : Forcer un redeploy
kubectl rollout restart deployment/gameshelf-api

# Ou Methode 2 : Changer une annotation
kubectl patch deployment gameshelf-api -p '{"spec":{"template":{"metadata":{"annotations":{"rollout-timestamp":"'$(date +%s)'"}}}}}'
```

Observez le remplacement progressif des pods :
1. Un nouveau pod est cree
2. Il devient Ready
3. Un ancien pod est termine
4. Repetez jusqu'a ce que tous les pods soient mis a jour

### Etape 4.4 : Historique et rollback

```bash
# Voir l'historique des deploiements
kubectl rollout history deployment/gameshelf-api

# Statut du deploiement
kubectl rollout status deployment/gameshelf-api

# Revenir a la version precedente (si probleme)
kubectl rollout undo deployment/gameshelf-api

# Revenir a une version specifique
kubectl rollout undo deployment/gameshelf-api --to-revision=1
```

---

## Partie 5 : Test de Zero Downtime

### Etape 5.1 : Script de test

Creez un script pour tester la disponibilite :

```bash
nano test-availability.sh
```

```bash
#!/bin/bash

CLUSTER_URL="${1}"
MY_NS="${2}"
SUCCESS=0
FAILURE=0
TOTAL=0

echo "Test de disponibilite sur $CLUSTER_URL/$MY_NS/"
echo "Appuyez sur Ctrl+C pour arreter"
echo ""

while true; do
    TOTAL=$((TOTAL + 1))
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "$CLUSTER_URL/$MY_NS/health" 2>/dev/null)

    if [ "$RESPONSE" = "200" ]; then
        SUCCESS=$((SUCCESS + 1))
        echo -ne "\r[OK: $SUCCESS | FAIL: $FAILURE | TOTAL: $TOTAL]"
    else
        FAILURE=$((FAILURE + 1))
        echo -ne "\r[OK: $SUCCESS | FAIL: $FAILURE | TOTAL: $TOTAL] Code: $RESPONSE"
    fi

    sleep 0.5
done
```

```bash
chmod +x test-availability.sh
```

### Etape 5.2 : Tester

**Terminal 1 :**
```bash
# Remplacez par l'URL du cluster et votre namespace
./test-availability.sh http://[CLUSTER_URL] votre-namespace
```

**Terminal 2 :**
```bash
# Declenchez une mise a jour
kubectl rollout restart deployment/gameshelf-api
```

Observez : le compteur de succes continue sans (ou avec tres peu) d'echecs !

---

## Partie 6 : Mise a jour du ConfigMap

### Etape 6.1 : Modifier le ConfigMap

```bash
kubectl edit configmap gameshelf-config
```

Changez `APP_VERSION` en `2.1.0`.

### Etape 6.2 : Probleme - Les pods ne voient pas le changement

Les variables d'environnement sont lues au demarrage du pod. Pour appliquer les changements :

```bash
# Redemarrer les pods pour prendre en compte le nouveau ConfigMap
kubectl rollout restart deployment/gameshelf-api

# Verifier
POD=$(kubectl get pods -l app=gameshelf-api -o jsonpath='{.items[0].metadata.name}')
kubectl exec $POD -- env | grep APP_VERSION
```

---

## Partie 7 : Bonnes pratiques

### Labels et annotations

```yaml
metadata:
  labels:
    app: gameshelf-api      # Nom de l'application
    version: v2             # Version
    environment: production # Environnement
    team: backend           # Equipe responsable
  annotations:
    description: "API GameShelf pour le TP Cloud"
    maintainer: "votre.email@univ.fr"
```

### Resource requests et limits

Toujours definir les ressources pour eviter les problemes :

```yaml
resources:
  requests:    # Minimum garanti
    memory: "128Mi"
    cpu: "100m"
  limits:      # Maximum autorise
    memory: "256Mi"
    cpu: "200m"
```

### Probes

- **readinessProbe** : Le pod est-il pret a recevoir du trafic ?
- **livenessProbe** : Le pod est-il en vie ? (sinon, restart)

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 5   # Attendre au demarrage
  periodSeconds: 3         # Frequence de check
  failureThreshold: 3      # Nombre d'echecs avant "not ready"

livenessProbe:
  httpGet:
    path: /health
    port: 5000
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3      # Nombre d'echecs avant restart
```

---

## Partie 8 : Commandes utiles recapitulatives

```bash
# Deploiement
kubectl apply -f <fichier.yaml>
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
kubectl rollout restart deployment/<name>

# Debug
kubectl describe pod <name>
kubectl logs <pod>
kubectl logs -f <pod>  # Follow
kubectl exec -it <pod> -- sh

# ConfigMap / Secret
kubectl get configmap
kubectl describe configmap <name>
kubectl get secret
kubectl get secret <name> -o yaml
```

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Cree un ConfigMap et l'injecte dans les pods
- [ ] Cree un Secret pour les donnees sensibles
- [ ] Configure une strategie Rolling Update
- [ ] Teste une mise a jour sans interruption
- [ ] Effectue un rollback
- [ ] Verifie que les pods utilisent le ConfigMap
- [ ] Compris l'importance des probes

---

## Flag de validation

Apres avoir reussi une mise a jour sans downtime :

```
FLAG{K8s_R0ll1ng_Upd4t3_Z3r0_D0wnt1m3}
```

Soumettez ce flag sur la plateforme du cours.

---

## Transition vers le TP suivant

*"Excellent ! Notre application se deploie sans interruption. Mais tout ca semble complexe... Et si on simplifiait avec un PaaS ? Dans le prochain TP, nous decouvrirons Clever Cloud !"*

---

## Structure du projet

```
gameshelf-k8s/
├── deployment.yaml         # TP07
├── service.yaml            # TP07
├── ingress.yaml            # TP07
├── configmap.yaml          # TP08
├── secret.yaml             # TP08
├── deployment-v2.yaml      # TP08
├── deployment-rolling.yaml # TP08
└── test-availability.sh    # TP08
```

---

## En cas de probleme

| Probleme | Solution |
|----------|----------|
| Rollout bloque | Verifiez readinessProbe et les logs |
| ConfigMap non lu | Restart les pods apres modification |
| Pods en CrashLoopBackOff | Verifiez les logs : `kubectl logs <pod>` |
| Resources insuffisantes | Reduisez les requests ou les replicas |

---

**Difficulte : Avance**
