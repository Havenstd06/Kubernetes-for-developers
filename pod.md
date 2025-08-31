# Cycle de vie des pods dans Kubernetes

## Exercice Pratique : États et gestion des erreurs

### Objectif

Comprendre les différents états d'un pod et observer les comportements en cas d'erreur.

### 1. Pod avec image invalide

```yaml
# bad-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  labels:
    app: test
spec:
  containers:
    - name: bad-container
      image: nginx:doesnotexist
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
kubectl apply -f bad-pod.yaml
kubectl get pod bad-pod
kubectl describe pod bad-pod
kubectl get events --field-selector involvedObject.name=bad-pod --sort-by=.metadata.creationTimestamp
```

### 2. Scénarios à tester

#### 2.1 Observer CrashLoopBackOff

```yaml
# crash-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: crash-pod
spec:
  containers:
    - name: crash-container
      image: busybox:1.28
      command: [ "/bin/sh", "-c", "echo 'Starting...'; sleep 10; echo 'Crashing now!'; exit 1" ]
      resources:
        requests:
          memory: "32Mi"
          cpu: "100m"
        limits:
          memory: "64Mi"
          cpu: "200m"
```

```bash
kubectl apply -f crash-pod.yaml
kubectl get pod crash-pod -w
# Attendre plusieurs cycles de redémarrage puis Ctrl+C
kubectl describe pod crash-pod
kubectl logs crash-pod --previous
```

#### 2.2 Tester les politiques de redémarrage

```yaml
# restart-never-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: restart-never-pod
spec:
  restartPolicy: Never
  containers:
    - name: test-container
      image: busybox:1.28
      command: [ "/bin/sh", "-c", "echo 'Task completed'; sleep 5; echo 'Exiting with error'; exit 1" ]
```

```bash
kubectl apply -f restart-never-pod.yaml
kubectl get pod restart-never-pod -w
# Observer que le pod ne redémarre pas

# Tester avec OnFailure
kubectl delete pod restart-never-pod
sed 's/Never/OnFailure/' restart-never-pod.yaml | kubectl apply -f -
kubectl get pod restart-never-pod -w
```

#### 2.3 Probes défaillantes

```yaml
# probe-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.29
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          path: /nonexistent-endpoint
          port: 80
        initialDelaySeconds: 10
        periodSeconds: 5
        failureThreshold: 3
      readinessProbe:
        httpGet:
          path: /
          port: 80
        initialDelaySeconds: 5
        periodSeconds: 3
        failureThreshold: 2
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

```bash
kubectl apply -f probe-pod.yaml
kubectl get pod probe-pod -w
# Observer les redémarrages dus à la liveness probe
kubectl describe pod probe-pod
kubectl logs probe-pod --previous
```

### 3. Correction des problèmes

```bash
# Corriger l'image invalide
kubectl patch pod bad-pod -p '{"spec":{"containers":[{"name":"bad-container","image":"nginx:1.29"}]}}'
kubectl get pod bad-pod -w

# Corriger la liveness probe
kubectl delete pod probe-pod
sed 's|/nonexistent-endpoint|/|' probe-pod.yaml | kubectl apply -f -
kubectl get pod probe-pod
```

### 4. Debug et monitoring

```bash
# Voir l'historique des événements
kubectl get events --sort-by=.metadata.creationTimestamp

# Suivre les logs en temps réel
kubectl logs crash-pod -f

# Voir les métriques des pods
kubectl top pods

# Vérifier les détails d'un pod spécifique
kubectl get pod bad-pod -o yaml
```

### 5. Nettoyage

```bash
kubectl delete pod bad-pod crash-pod restart-never-pod probe-pod
```