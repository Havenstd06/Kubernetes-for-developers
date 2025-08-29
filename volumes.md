# Volumes dans Kubernetes

## Exercice Pratique : Application web avec stockage multi-types

### Objectif

Créer une application web utilisant différents types de volumes pour la configuration, la persistance et les logs.

### 1. Configuration et stockage persistant

```yaml
# webapp-volumes.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
  labels:
    app: webapp
data:
  nginx.conf: |
    events {
      worker_connections 1024;
    }
    http {
      include /etc/nginx/mime.types;
      default_type application/octet-stream;

      server {
        listen 80;
        server_name localhost;
        root /usr/share/nginx/html;

        access_log /var/log/nginx/access.log;
        error_log /var/log/nginx/error.log;

        location / {
          index index.html;
        }

        location /uploads {
          alias /data/uploads/;
          autoindex on;
        }
      }
    }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: uploads-pvc
  labels:
    app: webapp
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: gp2
  resources:
    requests:
      storage: 1Gi
```

```bash
kubectl apply -f webapp-volumes.yaml
kubectl get configmaps
kubectl get pvc
kubectl describe pvc uploads-pvc
```

### 2. Pod avec volumes multiples

```yaml
# webapp-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
  labels:
    app: webapp
spec:
  containers:
    - name: nginx
      image: nginx:1.29
      ports:
        - containerPort: 80
      resources:
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"
          cpu: "500m"
      volumeMounts:
        - name: config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: uploads
          mountPath: /data/uploads
        - name: logs
          mountPath: /var/log/nginx
    - name: log-processor
      image: busybox:1.28
      command: [ "/bin/sh", "-c" ]
      args:
        - |
          while true; do
            if [ -f /logs/access.log ]; then
              echo "Processing access logs..."
              grep -E "(POST|PUT)" /logs/access.log | tail -5 >> /logs/uploads.log 2>/dev/null || true
            fi
            sleep 10
          done
      volumeMounts:
        - name: logs
          mountPath: /logs
  volumes:
    - name: config
      configMap:
        name: webapp-config
    - name: uploads
      persistentVolumeClaim:
        claimName: uploads-pvc
    - name: logs
      emptyDir: { }
```

```bash
kubectl apply -f webapp-pod.yaml
kubectl get pod webapp
kubectl describe pod webapp
kubectl get pvc
```

### 3. Scénarios à tester

#### 3.1 Vérification des montages de volumes

```bash
# Vérifier que tous les volumes sont montés
kubectl exec webapp -c nginx -- df -h
kubectl exec webapp -c nginx -- ls -la /etc/nginx/
kubectl exec webapp -c nginx -- ls -la /data/uploads/
kubectl exec webapp -c nginx -- ls -la /var/log/nginx/

# Vérifier depuis le sidecar
kubectl exec webapp -c log-processor -- ls -la /logs/
```

#### 3.2 Test de persistance des données

```bash
# Créer des fichiers de test dans le volume persistant
kubectl exec webapp -c nginx -- sh -c "echo 'Test upload file' > /data/uploads/test.txt"
kubectl exec webapp -c nginx -- ls -la /data/uploads/

# Supprimer et recréer le pod
kubectl delete pod webapp
kubectl apply -f webapp-pod.yaml
kubectl wait --for=condition=Ready pod/webapp --timeout=60s

# Vérifier que le fichier persiste
kubectl exec webapp -c nginx -- cat /data/uploads/test.txt
```

### 4. Debug et monitoring

```bash
# Voir l'utilisation des volumes
kubectl exec webapp -c nginx -- df -h

# Vérifier les permissions des volumes
kubectl exec webapp -c nginx -- ls -la /data/
kubectl exec webapp -c log-processor -- ls -la /logs/

# Voir les événements liés aux volumes
kubectl describe pod webapp
kubectl get events --field-selector involvedObject.name=webapp

# Vérifier l'état du PVC
kubectl describe pvc uploads-pvc
```

### 5. Nettoyage

```bash
kubectl delete pod webapp
kubectl delete pvc uploads-pvc
kubectl delete configmap webapp-config
```
