# Namespaces dans Kubernetes

## Exercice Pratique : Gestion multi-environnements

### Objectif

Déployer une application dans différents environnements en utilisant les namespaces.

### Étapes

1. **Créer les Namespaces**

```bash
kubectl create namespace dev
kubectl create namespace staging
kubectl create namespace prod
```

2. **Créer un Déploiement**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

3. **Créer un Service**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp-service
  namespace: dev
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

4. **Tâches à Réaliser**

- Déployer l'application dans chaque namespace
- Configurer différentes ressources par environnement
- Tester l'isolation entre les namespaces
- Implémenter une NetworkPolicy

### Limitations et points importants

1. **Ce qui est partagé entre namespaces**

- Nœuds du cluster
- Volumes persistants
- Services de type NodePort ou LoadBalancer

2. **Ce qui est isolé**

- Ressources applicatives (Pods, Deployments, Services...)
- ResourceQuotas
- ConfigMaps et Secrets
- ServiceAccounts

3. **Bonnes pratiques de nommage**

- Utiliser des préfixes cohérents
- Éviter les noms trop génériques
- Documenter la convention de nommage

4. **Monitoring et maintenance**

```bash
# Vérifier les quotas
kubectl get resourcequota -n dev

# Voir les événements par namespace
kubectl get events -n dev

# Comparer les ressources entre namespaces
kubectl top pods --all-namespaces
```