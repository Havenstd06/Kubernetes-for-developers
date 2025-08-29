# SecurityContext dans Kubernetes

## Exercice Pratique : Configuration de la sécurité des conteneurs

### Objectif

Comprendre et configurer le SecurityContext pour sécuriser les pods.

### 1. SecurityContext utilisateur et groupe

```yaml
# security-user-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-user-pod
  labels:
    app: security-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    runAsNonRoot: true
  containers:
    - name: sec-ctx-demo
      image: busybox:1.28
      command: [ "sh", "-c", "sleep 3600" ]
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      emptyDir: { }
```

```bash
kubectl apply -f security-user-pod.yaml
kubectl get pod security-user-pod
kubectl describe pod security-user-pod
kubectl exec security-user-pod -- id
kubectl exec security-user-pod -- touch /data/test.txt
kubectl exec security-user-pod -- ls -l /data/test.txt
```

### 2. Scénarios à tester

#### 2.1 Conteneur sans privilèges

```yaml
# no-caps-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-caps-pod
spec:
  containers:
    - name: no-caps-container
      image: busybox:1.28
      command: [ "sh", "-c", "sleep 3600" ]
      securityContext:
        allowPrivilegeEscalation: false
        runAsNonRoot: true
        runAsUser: 1000
        capabilities:
          drop: [ "ALL" ]
```

```bash
kubectl apply -f no-caps-pod.yaml
kubectl get pod no-caps-pod

# Vérifier les capabilities (doit être vide)
kubectl exec no-caps-pod -- cat /proc/1/status | grep Cap

# Tester des actions qui nécessitent des privilèges (doivent échouer)
kubectl exec no-caps-pod -- ping -c 1 8.8.8.8 || echo "Ping blocked - no NET_RAW capability"
```

#### 2.2 Système de fichiers en lecture seule

```yaml
# readonly-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: readonly-pod
spec:
  containers:
    - name: readonly-container
      image: nginx:1.29
      ports:
        - containerPort: 8080
      securityContext:
        readOnlyRootFilesystem: true
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        allowPrivilegeEscalation: false
      volumeMounts:
        - name: tmp-vol
          mountPath: /tmp
        - name: var-run
          mountPath: /var/run
        - name: var-cache
          mountPath: /var/cache/nginx
        - name: var-log
          mountPath: /var/log/nginx
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
  volumes:
    - name: tmp-vol
      emptyDir: { }
    - name: var-run
      emptyDir: { }
    - name: var-cache
      emptyDir: { }
    - name: var-log
      emptyDir: { }
    - name: nginx-conf
      configMap:
        name: nginx-readonly-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-readonly-config
data:
  default.conf: |
    server {
        listen 8080;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
```

```bash
kubectl apply -f readonly-pod.yaml
kubectl get pod readonly-pod
kubectl describe pod readonly-pod

# Tester l'écriture (doit échouer sur root filesystem)
kubectl exec readonly-pod -- touch /test.txt || echo "Write blocked on root filesystem"

# Tester l'écriture sur volumes autorisés (doit réussir)
kubectl exec readonly-pod -- touch /tmp/test.txt
kubectl exec readonly-pod -- ls -la /tmp/test.txt

# Vérifier que nginx fonctionne
kubectl exec readonly-pod -- curl -s localhost:8080 || echo "Testing nginx..."
```

#### 2.3 Capabilities spécifiques pour nginx

```yaml
# nginx-caps-pod.yaml
# nginx-caps-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-caps-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.29
      ports:
        - containerPort: 8080
      securityContext:
        runAsNonRoot: true
        runAsUser: 101
        runAsGroup: 101
        allowPrivilegeEscalation: false
        capabilities:
          drop: [ "ALL" ]
          add: [ "NET_BIND_SERVICE" ]
      volumeMounts:
        - name: tmp-vol
          mountPath: /tmp
        - name: var-run
          mountPath: /var/run
        - name: var-cache
          mountPath: /var/cache/nginx
        - name: var-log
          mountPath: /var/log/nginx
        - name: nginx-conf
          mountPath: /etc/nginx/conf.d/default.conf
          subPath: default.conf
  volumes:
    - name: tmp-vol
      emptyDir: { }
    - name: var-run
      emptyDir: { }
    - name: var-cache
      emptyDir: { }
    - name: var-log
      emptyDir: { }
    - name: nginx-conf
      configMap:
        name: nginx-nonroot-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-nonroot-config
data:
  default.conf: |
    server {
        listen 8080;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
    }
```

```bash
kubectl apply -f nginx-caps-pod.yaml
kubectl get pod nginx-caps-pod

# Vérifier que nginx peut se lier au port 80
kubectl exec nginx-caps-pod -- curl -s localhost:8080 | head -5
kubectl describe pod nginx-caps-pod
```

### 3. Debug et vérification de sécurité

```bash
# Voir les SecurityContext appliqués
kubectl get pod security-user-pod -o yaml | grep -A 10 securityContext

# Vérifier les capabilities d'un conteneur
kubectl exec no-caps-pod -- cat /proc/1/status | grep -E "CapInh|CapPrm|CapEff"

# Tester les permissions de fichiers
kubectl exec security-user-pod -- whoami || echo "Command not available"
kubectl exec security-user-pod -- id

# Vérifier l'état des pods sécurisés
kubectl get pods -l app=security-demo
```

### 4. Nettoyage

```bash
kubectl delete pod security-user-pod no-caps-pod readonly-pod nginx-caps-pod
```
