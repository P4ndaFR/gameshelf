# TP 08 - Kubernetes (2/2) : Rolling Updates, ConfigMaps et Ingress

## Contexte GameShelf

Notre API tourne sur Kubernetes, mais comment déployer une nouvelle version sans interrompre le service ? Comment gérer la configuration externe ? Comment router le trafic HTTP de manière élégante ?

**Objectif de ce TP :** Maîtriser les déploiements sans interruption et la configuration avancée.

---

## Prérequis

- TP 07 complété (cluster K8s fonctionnel avec l'API déployée)
- kubectl configuré
- Image Docker de l'API publiée

---

## Partie 1 : ConfigMaps - Configuration externalisée

### Pourquoi externaliser la configuration ?

- **Séparation code/config** : Une même image pour tous les environnements
- **Mise à jour sans rebuild** : Changer la config sans reconstruire l'image
- **Sécurité** : Les secrets ne sont pas dans l'image

### Étape 1.1 : Créer un ConfigMap

```bash
cd ~/gameshelf-k8s
nano configmap.yaml
```

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gameshelf-config
  namespace: gameshelf
data:
  # Variables d'environnement simples
  APP_NAME: "GameShelf"
  APP_VERSION: "2.0.0"
  ENVIRONMENT: "production"
  LOG_LEVEL: "INFO"

  # Configuration JSON (fichier)
  app-config.json: |
    {
      "features": {
        "recommendations": true,
        "rentals": true,
        "events": false
      },
      "api": {
        "rateLimit": 100,
        "timeout": 30
      }
    }
```

```bash
kubectl apply -f configmap.yaml
kubectl get configmaps
kubectl describe configmap gameshelf-config
```

### Étape 1.2 : Mettre à jour le Deployment

```bash
nano deployment-v2.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gameshelf-api
  namespace: gameshelf
  labels:
    app: gameshelf-api
spec:
  replicas: 3
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
        image: VOTRE_USERNAME/gameshelf-api:latest
        ports:
        - containerPort: 5000
        # Variables d'environnement depuis ConfigMap
        env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: gameshelf-config
              key: APP_NAME
        - name: APP_VERSION
          valueFrom:
            configMapKeyRef:
              name: gameshelf-config
              key: APP_VERSION
        - name: ENVIRONMENT
          valueFrom:
            configMapKeyRef:
              name: gameshelf-config
              key: ENVIRONMENT
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: gameshelf-config
              key: LOG_LEVEL
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        # Monter le fichier de config
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
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
      volumes:
      - name: config-volume
        configMap:
          name: gameshelf-config
          items:
          - key: app-config.json
            path: app-config.json
```

```bash
kubectl apply -f deployment-v2.yaml
```

### Étape 1.3 : Vérifier les variables d'environnement

```bash
# Obtenir le nom d'un pod
POD=$(kubectl get pods -l app=gameshelf-api -o jsonpath='{.items[0].metadata.name}')

# Voir les variables d'environnement
kubectl exec $POD -- env | grep -E "APP_|ENVIRONMENT|LOG_"

# Voir le fichier de config monté
kubectl exec $POD -- cat /app/config/app-config.json
```

---

## Partie 2 : Secrets - Données sensibles

### Étape 2.1 : Créer un Secret

```bash
nano secret.yaml
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gameshelf-secrets
  namespace: gameshelf
type: Opaque
stringData:
  # En clair dans le YAML, encodé en base64 à la création
  DB_PASSWORD: "super_secret_password_123"
  API_KEY: "sk-gameshelf-prod-abc123xyz"
  JWT_SECRET: "my_jwt_secret_key_very_long"
```

```bash
kubectl apply -f secret.yaml

# Le secret est encodé en base64
kubectl get secret gameshelf-secrets -o yaml
```

### Étape 2.2 : Utiliser le Secret dans le Deployment

Ajoutez dans la section `env` du Deployment :

```yaml
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: gameshelf-secrets
      key: DB_PASSWORD
- name: API_KEY
  valueFrom:
    secretKeyRef:
      name: gameshelf-secrets
      key: API_KEY
```

---

## Partie 3 : Rolling Updates - Mise à jour sans interruption

### Étape 3.1 : Comprendre les stratégies de déploiement

| Stratégie | Description | Avantages | Inconvénients |
|-----------|-------------|-----------|---------------|
| **RollingUpdate** | Remplacement progressif | Zéro downtime | Plus lent |
| **Recreate** | Tout supprimer puis recréer | Simple | Downtime |

### Étape 3.2 : Configurer le Rolling Update

```bash
nano deployment-rolling.yaml
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gameshelf-api
  namespace: gameshelf
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 pod en plus pendant la mise à jour
      maxUnavailable: 0  # Aucun pod indisponible
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
        image: VOTRE_USERNAME/gameshelf-api:v2
        ports:
        - containerPort: 5000
        envFrom:
        - configMapRef:
            name: gameshelf-config
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

### Étape 3.3 : Simuler une mise à jour

Dans un terminal, surveillez les pods :
```bash
kubectl get pods -w
```

Dans un autre terminal, appliquez la mise à jour :
```bash
kubectl apply -f deployment-rolling.yaml
```

Observez le remplacement progressif des pods.

### Étape 3.4 : Historique des déploiements

```bash
# Voir l'historique
kubectl rollout history deployment/gameshelf-api

# Détails d'une révision
kubectl rollout history deployment/gameshelf-api --revision=2

# Statut du déploiement
kubectl rollout status deployment/gameshelf-api
```

### Étape 3.5 : Rollback (retour en arrière)

En cas de problème :

```bash
# Revenir à la version précédente
kubectl rollout undo deployment/gameshelf-api

# Revenir à une version spécifique
kubectl rollout undo deployment/gameshelf-api --to-revision=1
```

---

## Partie 4 : Tester le Zero Downtime

### Étape 4.1 : Script de test de disponibilité

```bash
nano test-availability.sh
```

```bash
#!/bin/bash

API_URL="${1:-http://localhost:8080}"
SUCCESS=0
FAILURE=0
TOTAL=0

echo "Test de disponibilité sur $API_URL"
echo "Appuyez sur Ctrl+C pour arrêter"
echo ""

while true; do
    TOTAL=$((TOTAL + 1))
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "$API_URL/health" 2>/dev/null)

    if [ "$RESPONSE" = "200" ]; then
        SUCCESS=$((SUCCESS + 1))
        echo -ne "\r[OK: $SUCCESS | FAIL: $FAILURE | TOTAL: $TOTAL] "
    else
        FAILURE=$((FAILURE + 1))
        echo -ne "\r[OK: $SUCCESS | FAIL: $FAILURE | TOTAL: $TOTAL] Code: $RESPONSE "
    fi

    sleep 0.2
done
```

```bash
chmod +x test-availability.sh
```

### Étape 4.2 : Lancer le test

Terminal 1 :
```bash
export API_IP=$(kubectl get service gameshelf-api-lb -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
./test-availability.sh http://$API_IP
```

Terminal 2 :
```bash
# Mettre à jour le deployment (changer l'image ou une annotation)
kubectl set image deployment/gameshelf-api api=VOTRE_USERNAME/gameshelf-api:v3

# Ou forcer un redéploiement
kubectl rollout restart deployment/gameshelf-api
```

Observez : le compteur de succès continue sans échec !

---

## Partie 5 : Ingress Controller

### Pourquoi Ingress ?

- **Un seul point d'entrée** au lieu de multiples LoadBalancers
- **Routage HTTP** basé sur l'URL ou le hostname
- **Terminaison TLS** centralisée
- **Coût réduit** (1 LB au lieu de N)

### Étape 5.1 : Installer l'Ingress Controller (NGINX)

Pour OVH Managed Kubernetes :

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml
```

Vérifier l'installation :
```bash
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx
```

Attendre l'IP externe :
```bash
kubectl get service ingress-nginx-controller -n ingress-nginx -w
```

### Étape 5.2 : Modifier le Service en ClusterIP

L'Ingress va router vers le Service, donc pas besoin de LoadBalancer :

```bash
kubectl patch service gameshelf-api-lb -p '{"spec": {"type": "ClusterIP"}}'

# Ou recréer
kubectl delete service gameshelf-api-lb
kubectl apply -f service.yaml  # Le ClusterIP
```

### Étape 5.3 : Créer l'Ingress

```bash
nano ingress.yaml
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gameshelf-ingress
  namespace: gameshelf
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: gameshelf-api
            port:
              number: 80
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gameshelf-api
            port:
              number: 80
```

```bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress gameshelf-ingress
```

### Étape 5.4 : Tester via l'Ingress

```bash
# IP de l'Ingress Controller
INGRESS_IP=$(kubectl get service ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "Ingress accessible sur: http://$INGRESS_IP"

curl http://$INGRESS_IP/
curl http://$INGRESS_IP/games
curl http://$INGRESS_IP/flag
```

### Étape 5.5 : Ingress avec hostname (bonus)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gameshelf-ingress-host
  namespace: gameshelf
spec:
  ingressClassName: nginx
  rules:
  - host: api.gameshelf.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gameshelf-api
            port:
              number: 80
```

Pour tester localement, ajoutez dans `/etc/hosts` :
```
<INGRESS_IP> api.gameshelf.local
```

---

## Partie 6 : Horizontal Pod Autoscaler (HPA)

### Étape 6.1 : Activer le Metrics Server

Le Metrics Server est généralement pré-installé sur OVH Managed Kubernetes.

Vérifier :
```bash
kubectl top nodes
kubectl top pods
```

### Étape 6.2 : Créer un HPA

```bash
nano hpa.yaml
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: gameshelf-api-hpa
  namespace: gameshelf
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: gameshelf-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

```bash
kubectl apply -f hpa.yaml
kubectl get hpa
```

### Étape 6.3 : Observer l'autoscaling

```bash
kubectl get hpa -w
```

Le nombre de réplicas s'ajuste automatiquement selon la charge.

---

## Validation du TP

Pour valider ce TP, vous devez avoir :

- [ ] Créé un ConfigMap et l'injecté dans les pods
- [ ] Créé un Secret pour les données sensibles
- [ ] Configuré une stratégie Rolling Update
- [ ] Testé une mise à jour sans interruption (zero downtime)
- [ ] Effectué un rollback
- [ ] Installé et configuré un Ingress Controller
- [ ] Créé un Ingress pour router le trafic
- [ ] (Bonus) Configuré un HPA

---

## Flag de validation

Après une mise à jour réussie sans downtime, accédez à l'API :

```bash
curl http://$INGRESS_IP/flag
```

```
FLAG{K8s_R0ll1ng_Upd4t3_Z3r0_D0wnt1m3}
```

---

## Transition vers le TP suivant

*"Excellent ! Notre application se déploie sans interruption et scale automatiquement. Mais tout ça semble complexe... Et si on simplifiait avec un PaaS ? Dans le prochain TP, nous découvrirons Clever Cloud !"*

---

## Structure du projet

```
gameshelf-k8s/
├── namespace.yaml
├── configmap.yaml
├── secret.yaml
├── deployment-v2.yaml
├── deployment-rolling.yaml
├── service.yaml
├── service-lb.yaml
├── ingress.yaml
├── hpa.yaml
└── test-availability.sh
```

---

## Commandes utiles

| Commande | Description |
|----------|-------------|
| `kubectl rollout status deploy/<name>` | Statut du déploiement |
| `kubectl rollout history deploy/<name>` | Historique |
| `kubectl rollout undo deploy/<name>` | Rollback |
| `kubectl set image deploy/<name> <container>=<image>` | Changer l'image |
| `kubectl rollout restart deploy/<name>` | Forcer redéploiement |

---

## En cas de problème

| Problème | Solution |
|----------|----------|
| Rollout bloqué | Vérifiez readinessProbe |
| Ingress 404 | Vérifiez le pathType et le backend |
| HPA ne scale pas | Vérifiez que metrics-server fonctionne |
| Secret non lu | Vérifiez le nom et la clé |

---

**Durée estimée : 1 heure**

**Difficulté : Avancé**
