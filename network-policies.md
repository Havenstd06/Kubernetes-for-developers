# NetworkPolicies dans Kubernetes

## Exercice Pratique : Contrôle du trafic réseau

### Objectif

Comprendre les NetworkPolicies en contrôlant le trafic entre deux pods.

### 1. Configuration initiale

```yaml
# app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-a
  labels:
    app: web-a
spec:
  containers:
    - name: nginx
      image: nginx:1.29
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-a-svc
spec:
  selector:
    app: web-a
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: v1
kind: Pod
metadata:
  name: web-b
  labels:
    app: web-b
spec:
  containers:
    - name: nginx
      image: nginx:1.29
      ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: web-b-svc
spec:
  selector:
    app: web-b
  ports:
    - port: 80
      targetPort: 80
```

```bash
# Déployer les applications
kubectl apply -f app.yaml
kubectl get pods
kubectl get services

# Attendre que les pods soient prêts
kubectl wait --for=condition=Ready pod/web-a --timeout=60s
kubectl wait --for=condition=Ready pod/web-b --timeout=60s

# Test de connectivité initiale
kubectl exec web-a -- curl -s --max-time 10 web-b-svc
kubectl exec web-b -- curl -s --max-time 10 web-a-svc
```

### 2. Scénarios à tester

#### 2.1 Bloquer tout le trafic

```yaml
# deny-all.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: { }
  policyTypes:
    - Ingress
    - Egress
```

```bash
# Appliquer la politique de déni total
kubectl apply -f deny-all.yaml
kubectl get networkpolicies

# Tester (doit échouer)
kubectl exec web-a -- curl -s --max-time 5 web-b-svc || echo "Connection blocked as expected"
kubectl exec web-b -- curl -s --max-time 5 web-a-svc || echo "Connection blocked as expected"
```

#### 2.2 Autoriser un trafic spécifique

```yaml
# allow-a-to-b.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-a-to-b
spec:
  podSelector:
    matchLabels:
      app: web-b
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web-a
      ports:
        - protocol: TCP
          port: 80
```

```bash
# Appliquer la politique d'autorisation
kubectl apply -f allow-a-to-b.yaml
kubectl describe networkpolicy allow-a-to-b

# Test de web-a vers web-b (doit réussir)
kubectl exec web-a -- curl -s --max-time 10 web-b-svc

# Test de web-b vers web-a (doit toujours échouer)
kubectl exec web-b -- curl -s --max-time 5 web-a-svc || echo "Connection still blocked"
```

#### 2.3 Autoriser le trafic bidirectionnel

```yaml
# allow-bidirectional.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-a-egress
spec:
  podSelector:
    matchLabels:
      app: web-a
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: web-b
      ports:
        - protocol: TCP
          port: 80
    - to: [ ]
      ports:
        - protocol: UDP
          port: 53
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-a-ingress
spec:
  podSelector:
    matchLabels:
      app: web-a
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web-b
      ports:
        - protocol: TCP
          port: 80
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-b-egress
spec:
  podSelector:
    matchLabels:
      app: web-b
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: web-a
      ports:
        - protocol: TCP
          port: 80
    - to: [ ]
      ports:
        - protocol: UDP
          port: 53
```

```bash
# Appliquer les politiques bidirectionnelles
kubectl apply -f allow-bidirectional.yaml

# Tester dans les deux sens
kubectl exec web-a -- curl -s --max-time 10 web-b-svc
kubectl exec web-b -- curl -s --max-time 10 web-a-svc
```

### 3. Debug et vérification

```bash
# Lister toutes les NetworkPolicies
kubectl get networkpolicies

# Voir les détails d'une politique
kubectl describe networkpolicy deny-all
kubectl describe networkpolicy allow-a-to-b

# Vérifier les labels des pods
kubectl get pods --show-labels

# Voir les événements
kubectl get events --sort-by=.metadata.creationTimestamp
```

### 4. Nettoyage

```bash
kubectl delete -f app.yaml
kubectl delete networkpolicy deny-all allow-a-to-b allow-web-a-egress allow-web-a-ingress allow-web-b-egress 
```