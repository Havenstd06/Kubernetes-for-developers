# Les Hooks Lifecycle dans Kubernetes

## Exercice Pratique : postStart et preStop Hooks

### Objectif

Comprendre et utiliser les hooks lifecycle pour contrôler les événements du cycle de vie des conteneurs.

### 1. Hook postStart basique

```yaml
# poststart-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: poststart-demo
  labels:
    app: poststart-demo
spec:
  containers:
    - name: web
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
      lifecycle:
        postStart:
          exec:
            command: [ "/bin/sh", "-c", "echo \"Container started at $(date)\" > /usr/share/nginx/html/startup.html" ]
```

```bash
kubectl apply -f poststart-demo.yaml
kubectl get pod poststart-demo
kubectl describe pod poststart-demo
kubectl exec poststart-demo -- cat /usr/share/nginx/html/startup.html
```

### 2. Scénarios à tester

#### 2.1 Hook preStop pour arrêt gracieux

```yaml
# prestop-nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: prestop-nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.29
      ports:
        - containerPort: 80
      lifecycle:
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - |
                echo "Graceful shutdown initiated at $(date)" >> /var/log/shutdown.log
                nginx -s quit
                while pgrep nginx > /dev/null; do 
                  echo "Waiting for nginx to stop..."
                  sleep 1
                done
                echo "Nginx stopped gracefully at $(date)" >> /var/log/shutdown.log
  terminationGracePeriodSeconds: 30
```

```bash
kubectl apply -f prestop-nginx.yaml
kubectl get pod prestop-nginx

# Tester l'arrêt gracieux
kubectl delete pod prestop-nginx --grace-period=30
kubectl get pod prestop-nginx -w
```

#### 2.2 Hooks combinés avec logging

```yaml
# lifecycle-complete.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-complete
spec:
  containers:
    - name: app
      image: nginx:1.29
      ports:
        - containerPort: 80
      volumeMounts:
        - name: log-volume
          mountPath: /var/log/app
      lifecycle:
        postStart:
          exec:
            command:
              - /bin/sh
              - -c
              - |
                echo "[$(date)] PostStart: Container initialization started" >> /var/log/app/lifecycle.log
                # Simuler une initialisation
                sleep 2
                echo "[$(date)] PostStart: Application ready" >> /var/log/app/lifecycle.log
                echo "Application initialized successfully" > /usr/share/nginx/html/status.html
        preStop:
          exec:
            command:
              - /bin/sh
              - -c
              - |
                echo "[$(date)] PreStop: Shutdown signal received" >> /var/log/app/lifecycle.log
                echo "Application shutting down" > /usr/share/nginx/html/status.html
                # Arrêt gracieux
                nginx -s quit
                # Attendre l'arrêt complet
                timeout=10
                while pgrep nginx > /dev/null && [ $timeout -gt 0 ]; do
                  echo "[$(date)] PreStop: Waiting for shutdown..." >> /var/log/app/lifecycle.log
                  sleep 1
                  timeout=$((timeout-1))
                done
                echo "[$(date)] PreStop: Shutdown complete" >> /var/log/app/lifecycle.log
  terminationGracePeriodSeconds: 20
  volumes:
    - name: log-volume
      emptyDir: { }
```

```bash
kubectl apply -f lifecycle-complete.yaml
kubectl get pod lifecycle-complete

# Vérifier les logs du postStart
kubectl exec lifecycle-complete -- cat /var/log/app/lifecycle.log
kubectl exec lifecycle-complete -- cat /usr/share/nginx/html/status.html

# Tester le preStop
kubectl delete pod lifecycle-complete
kubectl logs pod lifecycle-complete
```

#### 2.3 Hook avec gestion d'erreurs

```yaml
# lifecycle-error-handling.yaml
apiVersion: v1
kind: Pod
metadata:
  name: lifecycle-error-test
spec:
  containers:
    - name: app
      image: nginx:1.29
      lifecycle:
        postStart:
          exec:
            command:
              - /bin/sh
              - -c
              - |
                # Test avec possibilité d'échec
                if [ "$((RANDOM % 2))" -eq 0 ]; then
                  echo "PostStart succeeded" > /tmp/poststart-result
                  exit 0
                else
                  echo "PostStart failed" > /tmp/poststart-result
                  exit 1
                fi
```

```bash
kubectl apply -f lifecycle-error-handling.yaml
kubectl get pod lifecycle-error-test
kubectl describe pod lifecycle-error-test
kubectl exec lifecycle-error-test -- cat /tmp/poststart-result || echo "Pod may have failed"
```

### 3. Deployment avec hooks

```yaml
# web-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app-hooks
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: web
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
          lifecycle:
            postStart:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    echo "Pod $(hostname) started at $(date)" > /usr/share/nginx/html/info.html
                    echo "<h1>Web Application</h1>" >> /usr/share/nginx/html/info.html
                    echo "<p>Started: $(date)</p>" >> /usr/share/nginx/html/info.html
            preStop:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - |
                    echo "Draining connections..."
                    sleep 5
                    nginx -s quit
          readinessProbe:
            httpGet:
              path: /info.html
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
```

```bash
kubectl apply -f web-app-deployment.yaml
kubectl get pods -l app=web-app
kubectl get deployment web-app-hooks

# Tester l'accès aux pages créées par postStart
kubectl port-forward deployment/web-app-hooks 8080:80
# Dans un autre terminal : curl localhost:8080/info.html
```

### 4. Debug et monitoring des hooks

```bash
# Voir les événements liés aux hooks
kubectl describe pod lifecycle-complete

# Voir les logs en temps réel
kubectl logs lifecycle-complete -f

# Vérifier l'état des pods
kubectl get pods -o wide

# Événements du cluster
kubectl get events --sort-by=.metadata.creationTimestamp
```

### 5. Nettoyage

```bash
kubectl delete pod poststart-demo prestop-nginx lifecycle-complete lifecycle-error-test
kubectl delete deployment web-app-hooks
```
