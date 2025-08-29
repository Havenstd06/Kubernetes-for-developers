# Services dans Kubernetes

## Exercice Pratique : Exposition et découverte de services

### Objectif

Comprendre les différents types de Services et leur utilisation pour exposer des applications.

### 1. Service ClusterIP

```yaml
# backend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 3
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
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f backend-app.yaml
kubectl get deployments
kubectl get services
kubectl get endpoints backend-service
kubectl describe service backend-service
```

### 2. Scénarios à tester

#### 2.1 Test de connectivité ClusterIP

```bash
# Tester l'accès depuis un pod temporaire
kubectl run test-pod --image=busybox:1.28 --restart=Never -- sh -c "wget -qO- http://backend-service"
kubectl logs test-pod
kubectl delete pod test-pod

# Vérifier la résolution DNS
kubectl run test-pod --image=busybox:1.28 --restart=Never -- sh -c "nslookup backend-service"
kubectl logs test-pod
kubectl delete pod test-pod

# Vérifier les endpoints
kubectl get endpoints backend-service -o wide
```

#### 2.2 Service NodePort

```yaml
# nodeport-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: nodeport-service
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

```bash
kubectl apply -f nodeport-service.yaml
kubectl get services nodeport-service
kubectl describe service nodeport-service

# Tester l'accès via NodePort (si possible depuis votre environnement)
# curl http://<node-ip>:30080
echo "NodePort accessible sur le port 30080 de tous les nœuds"
```

#### 2.3 Service avec sélecteur personnalisé

```yaml
# custom-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: custom-service
spec:
  selector:
    app: backend
    version: v1
  ports:
    - port: 8080
      targetPort: 80
      name: http
```

```bash
kubectl apply -f custom-service.yaml
kubectl get service custom-service

# Ajouter le label version aux pods existants
kubectl label pods -l app=backend version=v1

# Vérifier les nouveaux endpoints
kubectl get endpoints custom-service
kubectl describe endpoints custom-service
```

### 3. Service Discovery et debugging

```bash
# Voir tous les services
kubectl get services -o wide

# Tester le service discovery DNS
kubectl run debug-pod --image=busybox:1.28 -- sh -c "nslookup backend-service.default.svc.cluster.local"
kubectl logs debug-pod
kubectl delete pod debug-pod

# Vérifier les variables d'environnement de service discovery
kubectl run env-pod --image=busybox:1.28 -- sh -c "env | grep BACKEND_SERVICE"
kubectl logs env-pod
kubectl delete pod env-pod

# Debug des endpoints
kubectl get endpoints
kubectl describe endpoints backend-service

# Voir les logs des pods backend
kubectl logs -l app=backend --tail=5
```

### 4. Nettoyage

```bash
kubectl delete service backend-service nodeport-service custom-service
kubectl delete deployment backend
```
