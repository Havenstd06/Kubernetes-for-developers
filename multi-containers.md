# Multi-Container Pods dans Kubernetes

## Exercice Pratique : Partage de ressources entre conteneurs

### Objectif

Comprendre qu'un pod peut contenir plusieurs conteneurs qui partagent le même réseau et les mêmes volumes.

### 1. Pod avec partage de volume

```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  containers:
    - name: container-1
      image: busybox:1.28
      command: [ "/bin/sh", "-c" ]
      args:
        - |
          echo "Container 1 started" > /shared/from-container-1.txt
          while true; do
            echo "$(date): Hello from container 1" >> /shared/from-container-1.txt
            sleep 30
          done
      volumeMounts:
        - name: shared-volume
          mountPath: /shared
    - name: container-2
      image: busybox:1.28
      command: [ "/bin/sh", "-c" ]
      args:
        - |
          sleep 10
          while true; do
            echo "=== Container 2 reading files ==="
            ls -la /shared/
            if [ -f /shared/from-container-1.txt ]; then
              echo "Content from container 1:"
              cat /shared/from-container-1.txt
            fi
            sleep 30
          done
      volumeMounts:
        - name: shared-volume
          mountPath: /shared
  volumes:
    - name: shared-volume
      emptyDir: { }
```

```bash
kubectl apply -f multi-container-pod.yaml
kubectl get pod multi-container-pod
kubectl describe pod multi-container-pod
```

### 2. Tester le partage de volumes

```bash
# Voir les logs de chaque conteneur
kubectl logs multi-container-pod -c container-1
kubectl logs multi-container-pod -c container-2

# Vérifier que les fichiers sont partagés
kubectl exec multi-container-pod -c container-1 -- ls -la /shared/
kubectl exec multi-container-pod -c container-2 -- ls -la /shared/
kubectl exec multi-container-pod -c container-2 -- cat /shared/from-container-1.txt

# Tester l'écriture depuis container-2
kubectl exec multi-container-pod -c container-2 -- sh -c "echo 'Hello from container 2' > /shared/from-container-2.txt"
kubectl exec multi-container-pod -c container-1 -- cat /shared/from-container-2.txt
```

### 3. Tester le partage réseau

```bash
# Vérifier que les conteneurs partagent la même IP
kubectl exec multi-container-pod -c container-1 -- hostname -i
kubectl exec multi-container-pod -c container-2 -- hostname -i

# Vérifier que les conteneurs voient les mêmes interfaces réseau
kubectl exec multi-container-pod -c container-1 -- ip addr show
kubectl exec multi-container-pod -c container-2 -- ip addr show
```

### 4. Nettoyage

```bash
kubectl delete pod multi-container-pod
```
