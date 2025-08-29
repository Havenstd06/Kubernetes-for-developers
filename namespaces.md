# Namespaces dans Kubernetes

## Exercice Pratique : Isolation multi-environnements

### Objectif

Déployer une application dans différents environnements en utilisant les namespaces.

### 1. Création des namespaces

```yaml
# environments.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: development
---
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    environment: staging
---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    environment: production
```

```bash
kubectl apply -f environments.yaml
kubectl get namespaces
kubectl get namespaces --show-labels
```

### 2. Déploiement dans différents environnements

```yaml
# dev-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: dev
  labels:
    app: myapp
    environment: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        environment: dev
    spec:
      containers:
        - name: myapp
          image: nginx:1.29
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: dev
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

```bash
kubectl apply -f dev-app.yaml
kubectl get all -n dev
kubectl describe deployment myapp -n dev
```

### 3. Scénarios à tester

#### 3.1 Déploiement multi-environnements avec configurations différentes

```yaml
# staging-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: staging
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
        environment: staging
    spec:
      containers:
        - name: myapp
          image: nginx:1.29
          resources:
            requests:
              memory: "128Mi"
              cpu: "500m"
            limits:
              memory: "256Mi"
              cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: staging
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f staging-app.yaml
kubectl get pods -n dev
kubectl get pods -n staging
# Comparer les ressources
kubectl describe deployment myapp -n dev
kubectl describe deployment myapp -n staging
```

#### 3.2 Test d'isolation entre namespaces

```bash
# Tester la résolution DNS entre namespaces
kubectl run test-pod --image=busybox:1.28 -n dev --rm -it -- nslookup myapp-service


# Essayer d'accéder au service d'un autre namespace
kubectl run test-pod --image=busybox:1.28 -n dev --rm -it -- nslookup myapp-service.staging.svc.cluster.local

# Vérifier que les pods sont regroupés par namespace
kubectl get pods -n dev
kubectl get pods -n staging
kubectl get pods --all-namespaces | grep myapp
```

### 4. Gestion et navigation

```bash
# Changer de namespace par défaut
kubectl config set-context --current --namespace=dev
kubectl get pods  # Maintenant dans dev par défaut

# Revenir au namespace default
kubectl config set-context --current --namespace=default

# Voir les ressources dans tous les namespaces
kubectl get pods --all-namespaces
kubectl get services --all-namespaces

# Filtrer par labels
kubectl get namespaces -l environment=development
```

### 5. Nettoyage

```bash
kubectl delete namespace dev staging prod
```
