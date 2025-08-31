# Les Deployments dans Kubernetes

## Exercice Pratique : Gestion de version d'application

### Objectif

Déployer et gérer une application web avec différentes versions.

### 1. Création du Deployment

```yaml
# webapp-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
        version: v1
    spec:
      containers:
        - name: webapp
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
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 20
```

```bash 
kubectl apply -f webapp-deployment.yaml
kubectl get deployments
kubectl get pods -l app=webapp
kubectl describe deployment webapp
```

### 2. Scénarios à tester

#### 2.1 Mise à l'échelle horizontale

```bash
# Augmenter le nombre de réplicas
kubectl scale deployment webapp --replicas=5
kubectl get pods -l app=webapp
kubectl rollout status deployment/webapp

# Diminuer le nombre de réplicas
kubectl scale deployment webapp --replicas=2
kubectl get pods -l app=webapp -w
```

#### 2.2 Mise à jour d'image

```bash
# Mettre à jour vers une nouvelle version
kubectl set image deployment/webapp webapp=nginx:1.27
kubectl rollout status deployment/webapp
kubectl get pods -l app=webapp

# Vérifier l'historique
kubectl rollout history deployment/webapp

# Observer le rolling update en temps réel
kubectl get pods -l app=webapp -w
kubectl set image deployment/webapp webapp=nginx:1.25
```

#### 2.3 Rollback en cas de problème

```bash
# Simuler une mise à jour problématique
kubectl set image deployment/webapp webapp=nginx:invalid-tag
kubectl rollout status deployment/webapp

# Observer l'état
kubectl get pods -l app=webapp
kubectl describe deployment webapp

# Effectuer un rollback
kubectl rollout undo deployment/webapp
kubectl rollout status deployment/webapp
kubectl get pods -l app=webapp
```

#### 2.4 Pause et reprise de rollout

```bash 
# Démarrer une mise à jour
kubectl set image deployment/webapp webapp=nginx:1.26

# Mettre en pause immédiatement
kubectl rollout pause deployment/webapp
kubectl get pods -l app=webapp

# Reprendre le rollout
kubectl rollout resume deployment/webapp
kubectl rollout status deployment/webapp
```

### 3. Nettoyage

```bash
kubectl delete deployment webapp
```