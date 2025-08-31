# Les Init Containers dans Kubernetes

## Exercice Pratique : Vérification de dépendances avec Init Containers

### Objectif

Créer un pod avec un init container qui vérifie si un service est disponible avant de démarrer le conteneur principal.

### 1. Création du service backend

```yaml
# backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  labels:
    app: backend
spec:
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  selector:
    app: backend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  labels:
    app: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: nginx:1.29
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f backend-service.yaml
kubectl get service backend-service
kubectl get pods -l app=backend
```

### 2. Création du pod avec init container

```yaml
# web-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
  labels:
    app: web-app
spec:
  initContainers:
    - name: init-service-check
      image: busybox:1.28
      command: [ 'sh', '-c' ]
      args: [ 'until nslookup backend-service.default.svc.cluster.local; do echo "Waiting for backend service"; sleep 2; done;' ]
  containers:
    - name: web-app
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
```

```bash
kubectl apply -f web-pod.yaml
kubectl get pod web-pod
kubectl describe pod web-pod
kubectl logs web-pod -c init-service-check
```

### 3. Scénarios à tester

#### 3.1 Observer le comportement normal

```bash
# Voir l'état du pod
kubectl get pod web-pod -w

# Voir les logs de l'init container
kubectl logs web-pod -c init-service-check

# Voir les logs du conteneur principal
kubectl logs web-pod -c web-app
```

#### 3.2 Tester sans le service backend

```bash
# Supprimer le service backend
kubectl delete service backend-service

# Créer un nouveau pod
kubectl delete pod web-pod
kubectl apply -f web-pod.yaml

# Observer que l'init container reste bloqué
kubectl get pod web-pod
kubectl logs web-pod -c init-service-check
```

#### 3.3 Recréer le service pour débloquer

```bash
# Recréer le service
kubectl apply -f backend-service.yaml

# Observer que le pod démarre maintenant
kubectl get pod web-pod -w
kubectl describe pod web-pod
```

### 4. Init container avancé avec vérification HTTP

```yaml
# web-pod-advanced.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod-advanced
spec:
  initContainers:
    - name: init-service-check
      image: curlimages/curl:7.85.0
      command: [ 'sh', '-c' ]
      args:
        - |
          until curl -f http://backend-service.default.svc.cluster.local; do
            echo "Waiting for backend service to respond"
            sleep 5
          done
          echo "Backend service is ready!"
  containers:
    - name: web-app
      image: nginx:1.29
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f web-pod-advanced.yaml
kubectl get pod web-pod-advanced
kubectl logs web-pod-advanced -c init-service-check
```

### 5. Nettoyage

```bash
kubectl delete pod web-pod web-pod-advanced
kubectl delete deployment backend-app
kubectl delete service backend-service
rm -f backend-service.yaml web-pod.yaml web-pod-advanced.yaml
```