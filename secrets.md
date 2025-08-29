# Secrets dans Kubernetes

## Exercice Pratique : Gestion des données sensibles

### Objectif

Créer et utiliser différents types de Secrets pour sécuriser les données sensibles.

### 1. Création de Secrets basiques

```yaml
# db-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
  labels:
    app: myapp
type: Opaque
stringData:
  username: admin
  password: mySecretPassword123
  database: myapp
```

```bash
kubectl apply -f db-secret.yaml
kubectl get secrets
kubectl describe secret db-credentials
kubectl get secret db-credentials -o yaml
```

### 2. Scénarios à tester

#### 2.1 Utilisation via variables d'environnement

```yaml
# app-with-env.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-env
spec:
  containers:
    - name: app
      image: nginx:1.29
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
      envFrom:
        - secretRef:
            name: db-credentials
```

```bash
kubectl apply -f app-with-env.yaml
kubectl get pod app-with-env
kubectl exec app-with-env -- printenv | grep -E "DB_|username|password|database"
```

#### 2.2 Utilisation via volumes montés

```yaml
# app-with-volume.yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-volume
spec:
  containers:
    - name: app
      image: nginx:1.29
      volumeMounts:
        - name: secret-volume
          mountPath: /etc/secrets
          readOnly: true
  volumes:
    - name: secret-volume
      secret:
        secretName: db-credentials
```

```bash
kubectl apply -f app-with-volume.yaml
kubectl exec app-with-volume -- ls -la /etc/secrets
kubectl exec app-with-volume -- cat /etc/secrets/username
kubectl exec app-with-volume -- cat /etc/secrets/password
```

#### 2.3 Secret TLS pour HTTPS

```bash
# Générer un certificat de test
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=myapp.local/O=test"

# Créer le secret TLS
kubectl create secret tls tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Vérifier le secret TLS
kubectl get secret tls-secret
kubectl describe secret tls-secret
```

### 3. Application complète avec MySQL

```yaml
# mysql-app.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secrets
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: rootpassword
  MYSQL_DATABASE: testdb
  MYSQL_USER: appuser
  MYSQL_PASSWORD: apppassword
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          envFrom:
            - secretRef:
                name: mysql-secrets
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"
---
apiVersion: v1
kind: Service
metadata:
  name: mysql-service
spec:
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

```bash
kubectl apply -f mysql-app.yaml
kubectl get pods -l app=mysql
kubectl wait --for=condition=Ready pod -l app=mysql --timeout=120s

# Tester la connexion MySQL
kubectl exec deployment/mysql -- mysqladmin ping -u root -prootpassword
kubectl exec deployment/mysql -- mysql -u appuser -papppassword testdb -e "SELECT 'Connection successful' as status;" || echo "MySQL may still be initializing, try again in a few seconds"
```

### 4. Debug et vérification

```bash
# Voir les secrets disponibles
kubectl get secrets

# Examiner le contenu d'un secret (attention: visible en base64)
kubectl get secret db-credentials -o yaml

# Décoder les valeurs base64
kubectl get secret db-credentials -o jsonpath='{.data.username}' | base64 --decode; echo
kubectl get secret db-credentials -o jsonpath='{.data.password}' | base64 --decode; echo

# Vérifier les montages dans les pods
kubectl describe pod app-with-volume
kubectl exec app-with-volume -- ls -la /etc/secrets/
```

### 5. Nettoyage

```bash
kubectl delete secret db-credentials mysql-secrets tls-secret
kubectl delete pod app-with-env app-with-volume
kubectl delete deployment mysql
kubectl delete service mysql-service
```