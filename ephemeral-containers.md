# Les Conteneurs éphémères dans Kubernetes

## Exercice pratique

### Objectif

Utiliser les conteneurs éphémères pour déboguer une application Java distroless.

### Étapes

### 1. Déployer l'application

```yaml
# my-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  containers:
    - name: app
      image: nginx
```

```bash
kubectl apply -f my-app.yaml
kubectl get pod my-app
kubectl describe pod my-app
```

### 2. Scénarios de débogage

#### 2.1 Ajout d'un conteneur éphémère pour inspection générale

```bash
# Ajouter un debugger avec des outils complets
kubectl debug -it my-app --image=busybox --target=app

# Dans le conteneur éphémère, inspecter :
ps aux
ls -la /proc/1/root
cat /proc/1/environ
```

#### 2.2 Inspection des fichiers avec partage de namespace

```bash
# Partager le namespace de processus
kubectl debug -it my-app --image=alpine --target=app

# Dans le conteneur éphémère :
ls -la /proc/1/root/etc/nginx/conf.d/
cat /proc/1/environ
```

#### 2.3. Debug avec copy du pod

```bash 
# Créer une copie du pod avec shell
kubectl debug my-app -it --copy-to=my-app-debug --container=debugger --image=busybox  --profile=general

# Examiner la copie
kubectl get pod my-app-debug
kubectl exec -it my-app-debug -c debugger -- sh
```

### 3. Nettoyage

```bash
kubectl delete pod my-app my-app-debug
```