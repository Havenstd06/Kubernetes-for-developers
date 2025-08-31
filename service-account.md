# ServiceAccounts dans Kubernetes

## Exercice Pratique : Authentification et autorisation des pods

### Objectif

Créer et gérer des ServiceAccounts avec des permissions RBAC spécifiques.

### 1. ServiceAccount et permissions de base

```yaml
# rbac-setup.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: default
  labels:
    app: rbac-demo
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
  - apiGroups: [ "" ]
    resources: [ "pods" ]
    verbs: [ "get", "watch", "list" ]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
  - kind: ServiceAccount
    name: app-service-account
    namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f rbac-setup.yaml
kubectl get serviceaccounts
kubectl get roles
kubectl get rolebindings
kubectl describe serviceaccount app-service-account
```

### 2. Scénarios à tester

#### 2.1 Pod avec ServiceAccount personnalisé

```yaml
# pod-with-sa.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-sa
spec:
  serviceAccountName: app-service-account
  containers:
    - name: kubectl-client
      image: bitnami/kubectl:latest
      command: [ "sleep", "infinity" ]
      resources:
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"
          cpu: "500m"
```

```bash
kubectl apply -f pod-with-sa.yaml
kubectl get pod pod-with-sa
kubectl wait --for=condition=Ready pod/pod-with-sa --timeout=60s

# Tester les permissions autorisées
kubectl exec pod-with-sa -- kubectl get pods
kubectl exec pod-with-sa -- kubectl describe pod pod-with-sa

# Tester les permissions refusées (doit échouer)
kubectl exec pod-with-sa -- kubectl get services || echo "Access denied - as expected"
kubectl exec pod-with-sa -- kubectl delete pod pod-with-sa || echo "Delete denied - as expected"
```

#### 2.2 Application de monitoring avec permissions étendues

```yaml
# monitoring-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitoring-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: monitoring-role
rules:
  - apiGroups: [ "" ]
    resources: [ "pods", "services", "endpoints" ]
    verbs: [ "get", "list", "watch" ]
  - apiGroups: [ "" ]
    resources: [ "pods/log" ]
    verbs: [ "get" ]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: monitoring-binding
subjects:
  - kind: ServiceAccount
    name: monitoring-sa
roleRef:
  kind: Role
  name: monitoring-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: monitoring-pod
spec:
  serviceAccountName: monitoring-sa
  containers:
    - name: monitor
      image: bitnami/kubectl:latest
      command: [ "sh", "-c" ]
      args:
        - |
          while true; do
            echo "=== Monitoring Check ==="
            echo "Pods: $(kubectl get pods --no-headers | wc -l)"
            echo "Services: $(kubectl get services --no-headers | wc -l)"
            sleep 30
          done
```

```bash
kubectl apply -f monitoring-rbac.yaml
kubectl get pod monitoring-pod
kubectl logs monitoring-pod -f

# Tester les permissions spécifiques
kubectl exec monitoring-pod -- kubectl get pods
kubectl exec monitoring-pod -- kubectl get services
kubectl exec monitoring-pod -- kubectl logs pod-with-sa
```

#### 2.3 ServiceAccount sécurisé sans token automatique

```yaml
# secure-sa.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: secure-sa
automountServiceAccountToken: false
---
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  serviceAccountName: secure-sa
  containers:
    - name: app
      image: nginx:1.29
      ports:
        - containerPort: 80
```

```bash
kubectl apply -f secure-sa.yaml
kubectl get pod secure-pod

# Vérifier qu'aucun token n'est monté
kubectl exec secure-pod -- ls /var/run/secrets/kubernetes.io/serviceaccount/ || echo "No token mounted - as expected"

# Vérifier que les appels API échouent
kubectl exec secure-pod -- curl -k https://kubernetes.default.svc.cluster.local/api/v1/pods || echo "API access denied without token"
```

### 3. Vérification des permissions

```bash
# Vérifier les permissions d'un ServiceAccount
kubectl auth can-i --list --as=system:serviceaccount:default:app-service-account

# Vérifier une permission spécifique
kubectl auth can-i get pods --as=system:serviceaccount:default:app-service-account
kubectl auth can-i delete pods --as=system:serviceaccount:default:app-service-account

# Voir les tokens associés
kubectl get secrets | grep app-service-account

# Debug RBAC
kubectl describe role pod-reader
kubectl describe rolebinding read-pods
```

### 4. Nettoyage

```bash
kubectl delete serviceaccount app-service-account monitoring-sa secure-sa
kubectl delete role pod-reader monitoring-role
kubectl delete rolebinding read-pods monitoring-binding
kubectl delete pod pod-with-sa monitoring-pod secure-pod
```