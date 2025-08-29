# ConfigMaps dans Kubernetes

## Objectifs

- Créer des ConfigMaps de différentes manières
- Utiliser les ConfigMaps dans les pods
- Gérer les mises à jour de configuration
- Comprendre les meilleures pratiques

## Exercice 1 : Création de ConfigMaps

### 1.1 Création depuis des littéraux

```bash
kubectl create configmap literal-config \
  --from-literal=DB_HOST=mysql \
  --from-literal=DB_PORT=3306
```

### 1.2 Création depuis un fichier

1. Créer un fichier `app.properties` :

```properties
app.name=MyApp
app.env=production
log.level=INFO
```

2. Créer le ConfigMap :

```bash
kubectl create configmap properties-config --from-file=app.properties
```

### 1.3 Création depuis un répertoire

1. Créer plusieurs fichiers de configuration :

```bash
mkdir config
echo "server.port=8080" > config/server.properties
echo "cache.ttl=3600" > config/cache.properties
```

2. Créer le ConfigMap :

```bash
kubectl create configmap dir-config --from-file=config/
```

### 1.4 Création depuis un YAML

```yaml
# yaml-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: yaml-config
data:
  database.conf: |
    host=postgresql
    port=5432
    name=myapp
  api.conf: |
    endpoint=/api/v1
    timeout=30
```

```bash
kubectl apply -f yaml-config.yaml
```

### 1.5 Lister les configmaps

```bash
kubectl get configmaps
kubectl get configmaps -o yaml
kubectl describe configmap dir-config
kubectl describe configmap literal-config
kubectl describe configmap properties-config 
kubectl describe configmap yaml-config
```

## Exercice 2 : Utilisation des ConfigMaps

### 2.1 Variables d'environnement

```yaml
# env-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: env-pod
spec:
  containers:
    - name: app
      image: nginx
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: literal-config
              key: DB_PORT
```

```bash
kubectl apply -f env-pod.yaml
kubectl get pod env-pod 
kubectl describe pod env-pod 
```

### 2.2 Toutes les variables d'un ConfigMap

```yaml
# envfrom-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: envfrom-pod
spec:
  containers:
    - name: app
      image: nginx
      envFrom:
        - configMapRef:
            name: literal-config
```

```bash
kubectl apply -f envfrom-pod.yaml
kubectl get pod envfrom-pod
kubectl describe pod envfrom-pod
kubectl exec -ti envfrom-pod -- printenv | grep DB_
```

### 2.3 Montage en volume

```yaml
# volume-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: yaml-config
```

```bash
kubectl apply -f volume-pod.yaml
kubectl get pod volume-pod
kubectl describe pod volume-pod
kubectl exec -ti volume-pod -- ls -la /etc/config
```

### 2.4 Montage d'un fichier spécifique

```yaml
# single-file-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: single-file-pod
spec:
  containers:
    - name: app
      image: nginx
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app/database.conf
          subPath: database.conf
  volumes:
    - name: config-volume
      configMap:
        name: yaml-config
        items:
          - key: database.conf
            path: database.conf
```

```bash
kubectl apply -f single-file-pod.yaml
kubectl get pod single-file-pod
kubectl exec -ti single-file-pod -- cat /etc/app/database.conf
```

## Exercice 3 : Cas pratique - Application Web

1. Créer une configuration nginx :

```yaml
# nginx-config.yaml
# nginx-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  # Configuration nginx complète et valide
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

        location / {
          root /usr/share/nginx/html;
          index index.html;
        }
      }
    }
  # Page d'accueil personnalisée
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>ConfigMap Demo</title>
    </head>
    <body>
      <h1>Welcome to my configured nginx!</h1>
      <p>This page is served from a ConfigMap</p>
    </body>
    </html>
```

2. Déployer nginx avec cette configuration :

```yaml
# nginx-configured.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-configured
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: nginx-config
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
  volumes:
    - name: nginx-config
      configMap:
        name: nginx-config
```

```bash
kubectl apply -f nginx-config.yaml
kubectl apply -f nginx-configured.yaml
kubectl get pod nginx-configured
kubectl exec -ti nginx-configured -- curl localhost
```

## Exercice 4 : Mise à jour des ConfigMaps

1. Mettre à jour un ConfigMap :

```bash
# Modifier la page d'accueil
kubectl patch configmap nginx-config --patch='
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
      <title>ConfigMap Demo - Updated</title>
    </head>
    <body>
      <h1>Updated: Welcome to my configured nginx!</h1>
      <p>This page was updated via ConfigMap patch</p>
    </body>
    </html>
'
```

2. Observer le comportement :

```bash
kubectl exec nginx-configured -- cat /etc/nginx/conf.d/default.conf
```

3. Forcer le rechargement (en pratique) :

```bash
kubectl delete pod nginx-configured
kubectl apply -f nginx-configured.yaml
kubectl exec nginx-configured -- cat /usr/share/nginx/html/index.html
```

## Nettoyage

```bash
kubectl delete configmap literal-config properties-config dir-config yaml-config nginx-config
kubectl delete pod env-pod envfrom-pod volume-pod single-file-pod nginx-configured
rm -rf config/
rm app.properties yaml-config.yaml nginx-config.yaml nginx-configured.yaml env-pod.yaml envfrom-pod.yaml volume-pod.yaml single-file-pod.yaml
```

## Vérification et Debug

```bash
# Voir les ConfigMaps
kubectl get configmaps

# Voir le contenu d'un ConfigMap
kubectl describe configmap nginx-config

# Voir les variables d'environnement dans un pod
kubectl exec env-pod -- env

# Vérifier les fichiers montés
kubectl exec volume-pod -- ls -l /etc/config
```