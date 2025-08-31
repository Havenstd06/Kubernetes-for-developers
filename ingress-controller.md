# Ingress Controller dans Kubernetes

## Exercice Pratique : Exposition d'applications avec Ingress

### Objectif

Déployer et configurer NGINX Ingress Controller pour exposer plusieurs services.

### 1. Installation de NGINX Ingress Controller

```bash
# Ajouter le repository Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# Installer NGINX Ingress Controller
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
    --version 4.8.3 \
    --namespace ingress-nginx \
    --create-namespace \
    --set controller.service.type=LoadBalancer

# Attendre que le controller soit prêt
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s

# Vérifier l'installation
kubectl get pods -n ingress-nginx
kubectl get services -n ingress-nginx
```

### 2. Applications de test

```yaml
# test-apps.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
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
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: httpd:2.4
          ports:
            - containerPort: 80
          resources:
            requests:
              memory: "64Mi"
              cpu: "250m"
            limits:
              memory: "128Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
```

```bash
kubectl apply -f test-apps.yaml
kubectl get deployments
kubectl get services
kubectl get pods
```

### 3. Scénarios à tester

#### 3.1 Ingress de base avec routage par path

```yaml
# basic-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: basic-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: backend-svc
                port:
                  number: 80
```

```bash
kubectl apply -f basic-ingress.yaml
kubectl get ingress
kubectl describe ingress basic-ingress

# Obtenir l'IP du LoadBalancer
kubectl get service -n ingress-nginx ingress-nginx-controller
```

#### 3.2 Test de connectivité

```bash
# Récupérer l'External IP du LoadBalancer
export INGRESS_IP=$(kubectl get service -n ingress-nginx ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Ingress accessible sur: $INGRESS_IP"

# Tester l'accès au frontend
curl -H "Host: app.local" http://$INGRESS_IP/

# Tester l'accès au backend via /api
curl -H "Host: app.local" http://$INGRESS_IP/api/

# Alternative si LoadBalancer non disponible: port-forward
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80 &
curl -H "Host: app.local" http://localhost:8080/
```

#### 3.3 Ingress avec SSL/TLS

```bash
# Générer un certificat de test
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=app.local"

# Créer le secret TLS
kubectl create secret tls app-tls-secret \
  --key tls.key \
  --cert tls.crt
```

```yaml
# ssl-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ssl-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - app-ssl.local
      secretName: app-tls-secret
  rules:
    - host: app.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

```bash
kubectl apply -f ssl-ingress.yaml
kubectl describe ingress ssl-ingress

# Tester HTTPS
curl -k -H "Host: app.local" https://$INGRESS_IP/
```

### 4. Debug et monitoring

```bash
# Vérifier l'état de l'Ingress
kubectl get ingress
kubectl describe ingress basic-ingress

# Logs du controller NGINX
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller --tail=50

# Vérifier la configuration NGINX générée
kubectl exec -n ingress-nginx deployment/ingress-nginx-controller -- cat /etc/nginx/nginx.conf | grep -A 10 "server_name app.local"

# Événements liés aux Ingress
kubectl get events --field-selector involvedObject.kind=Ingress
```

### 5. Nettoyage

```bash
kubectl delete ingress basic-ingress ssl-ingress
kubectl delete secret app-tls-secret
kubectl delete service frontend-svc backend-svc
kubectl delete deployment frontend backend
helm uninstall ingress-nginx -n ingress-nginx
kubectl delete namespace ingress-nginx
```
