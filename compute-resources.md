# Gestion des resources (Requests et Limits)

## Objectifs

- Comprendre la différence entre requests et limits
- Configurer des ressources pour les pods
- Observer l'impact des limits
- Travailler avec LimitRange et ResourceQuota

## Pré-requis

Installer Metrics server :

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

## Exercice 1 : Pod basique avec resources

1. Créer un pod avec requests et limits :

```yaml
# resource-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        requests:
          memory: "64Mi"
          cpu: "250m"
        limits:
          memory: "128Mi"
          cpu: "500m"
```

2. Vérifier l'état du pod :

```bash
kubectl apply -f resource-demo.yaml
kubectl get pod resource-demo
kubectl describe pod resource-demo
```

## Exercice 2 : Démonstration des Limits

1. Créer un pod qui dépasse ses limites :

```yaml
# memory-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: memory-demo
spec:
  containers:
    - name: memory-demo
      image: nginx
      resources:
        requests:
          memory: "50Mi"
          cpu: "250m"
        limits:
          memory: "100Mi"
          cpu: "500m"
      command: [ "sh", "-c" ]
      args: [ "while true; do echo 'Consuming memory...'; dd if=/dev/zero of=/tmp/memory.tmp bs=1M count=200; sleep 5; done" ]
```

2. Observer le comportement :

```bash
kubectl apply -f memory-demo.yaml
kubectl get pod memory-demo
kubectl describe pod memory-demo
```

## Exercice 3 : LimitRange

1. Créer un LimitRange pour le namespace :

```yaml
# mem-limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
spec:
  limits:
    - default:
        memory: "256Mi"
        cpu: "500m"
      defaultRequest:
        memory: "128Mi"
        cpu: "250m"
      type: Container
```

```bash
kubectl apply -f mem-limit-range.yaml
```

2. Créer un pod sans spécifier de ressources :

```yaml
# default-resources.yaml
apiVersion: v1
kind: Pod
metadata:
  name: default-resources
spec:
  containers:
    - name: nginx
      image: nginx
```

```bash
kubectl apply -f default-resources.yaml
```

3. Vérifier les ressources attribuées automatiquement :

```bash
kubectl describe pod default-resources
```

## Exercice 4 : ResourceQuota

1. Créer un ResourceQuota :

```yaml
# compute-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 1Gi
    limits.cpu: "4"
    limits.memory: 2Gi
    pods: "4"
```

```bash
kubectl apply -f compute-quota.yaml
```

2. Tenter de déployer plusieurs pods :

```yaml
# nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 5
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          resources:
            requests:
              memory: "256Mi"
              cpu: "500m"
            limits:
              memory: "512Mi"
              cpu: "1"
```

```bash
kubectl apply -f nginx-deployment.yaml
```

3. Observer le comportement :

```bash
kubectl describe quota compute-quota
kubectl get pods
kubectl describe deployment nginx-deployment
```

## Exercice 5 : Les classes QoS

1. Libérer les ressources prises par les exercices précédants :

```bash
kubectl delete deployment nginx-deployment
kubectl delete pod resource-demo memory-demo default-resources --ignore-not-found
```

1. Pod Guaranteed (requests = limits) :

```yaml
# qos-guaranteed.yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-guaranteed
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        requests:
          memory: "256Mi"
          cpu: "500m"
        limits:
          memory: "256Mi"
          cpu: "500m"
```

```bash
kubectl apply -f qos-guaranteed.yaml 
```

2. Pod Burstable (requests < limits) :

```yaml
# qos-burstable.yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-burstable
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        requests:
          memory: "128Mi"
          cpu: "250m"
        limits:
          memory: "256Mi"
          cpu: "500m"
```

```bash
kubectl apply -f qos-burstable.yaml 
```

3. Pod BestEffort (pas de requests ni limits) :

```yaml
# qos-besteffort.yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-besteffort
spec:
  containers:
    - name: nginx
      image: nginx
```

```bash
kubectl apply -f qos-besteffort.yaml
```

4. Vérifier les QoS classes :

```bash
kubectl get pod -o custom-columns=NAME:.metadata.name,QoS:.status.qosClass
```

## Exercice 6 : Monitoring des Resources

1. Vérifier l'utilisation des ressources :

```bash
# Par pod
kubectl top pod

# Par nœud
kubectl top node
```

## Nettoyage

```bash
kubectl delete pod resource-demo memory-demo default-resources qos-guaranteed qos-burstable qos-besteffort --ignore-not-found
kubectl delete limitrange mem-limit-range --ignore-not-found
kubectl delete resourcequota compute-quota --ignore-not-found
kubectl delete deployment nginx-deployment --ignore-not-found
```