# StatefulSets dans Kubernetes

## Exercice Pratique : Déploiement d'application stateful

### Objectif

Comprendre le déploiement et la gestion d'applications stateful avec identités stables.

### 1. Service Headless et StatefulSet

```yaml
# mongodb-statefulset.yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
  labels:
    app: mongodb
spec:
  clusterIP: None
  selector:
    app: mongodb
  ports:
    - port: 27017
      targetPort: 27017
      name: mongodb
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  labels:
    app: mongodb
spec:
  serviceName: mongodb-service
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:4.4
          command: [ "mongod", "--replSet", "rs0", "--bind_ip_all" ]
          ports:
            - containerPort: 27017
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          volumeMounts:
            - name: data
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: gp2
        resources:
          requests:
            storage: 10Gi
```

```bash
kubectl apply -f mongodb-statefulset.yaml
kubectl get statefulsets
kubectl get pods -l app=mongodb
kubectl get pvc
kubectl describe statefulset mongodb
```

### 2. Scénarios à tester

#### 2.1 Vérification de l'ordre de création

```bash
# Observer la création séquentielle des pods
kubectl get pods -l app=mongodb -w

# Vérifier l'état de chaque pod
kubectl get pods -l app=mongodb
kubectl describe pod mongodb-0

# Vérifier les PVCs associés
kubectl get pvc -l app=mongodb
```

#### 2.2 Test de résolution DNS stable

```bash
# Tester la résolution DNS de chaque pod
kubectl run dns-test --image=busybox:1.28 --restart=Never -- nslookup mongodb-0.mongodb-service.default.svc.cluster.local
kubectl logs dns-test
kubectl delete pod dns-test

kubectl run dns-test --image=busybox:1.28 --restart=Never -- nslookup mongodb-1.mongodb-service.default.svc.cluster.local
kubectl logs dns-test
kubectl delete pod dns-test

# Vérifier la résolution du service headless
kubectl run dns-test --image=busybox:1.28 --restart=Never -- nslookup mongodb-service.default.svc.cluster.local
kubectl run dns-test --image=busybox:1.28 --restart=Never -- nslookup mongodb-service.default.svc.cluster.local
kubectl delete pod dns-test
```

#### 2.3 Test de mise à jour avec partition

```bash
# Appliquer une mise à jour avec partition
kubectl patch statefulset mongodb -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# Mettre à jour l'image
kubectl set image statefulset/mongodb mongodb=mongo:5.0

# Observer que seul mongodb-2 est mis à jour
kubectl get pods -l app=mongodb
kubectl describe pod mongodb-2

# Continuer la mise à jour
kubectl patch statefulset mongodb -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}'
kubectl get pods -l app=mongodb -w

# Compléter la mise à jour
kubectl patch statefulset mongodb -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

### 3. Gestion et scaling

```bash
# Scaling du StatefulSet
kubectl scale statefulset mongodb --replicas=5
kubectl get pods -l app=mongodb -w

# Scaling down (observe l'ordre inverse)
kubectl scale statefulset mongodb --replicas=2
kubectl get pods -l app=mongodb -w

# Vérifier la persistance des données
kubectl exec mongodb-0 -- ls -la /data/db
kubectl get pvc
```

### 4. Nettoyage

```bash
kubectl delete statefulset mongodb
kubectl delete service mongodb-service
kubectl delete pvc -l app=mongodb
```
