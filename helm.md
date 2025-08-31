# Introduction à Helm

## Objectifs

- Comprendre les concepts de base de Helm
- Utiliser des charts existants
- Créer un chart simple
- Gérer les releases

## Pré-requis : Installation de Helm

### Linux

```bash
# Méthode 1 : Script d'installation officiel
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Méthode 2 : Téléchargement manuel
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

# Méthode 3 : Package manager
# Ubuntu/Debian
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### macOS

```bash
# Avec Homebrew
brew install helm

# Avec MacPorts
sudo port install helm3
```

### Windows

```powershell
# Avec Chocolatey
choco install kubernetes-helm

# Avec Scoop
scoop install helm

# Avec Winget
winget install Helm.Helm
```

### Vérification de l'installation

```bash
# Vérifier la version installée
helm version

# Voir l'aide
helm help
```

## Exercice 1 : Utilisation de base

```bash
# Vérifier la version de Helm
helm version

# Ajouter le repo bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami

# Mettre à jour les repos
helm repo update

# Rechercher nginx
helm search repo nginx

# Installer nginx avec version spécifique pour éviter les bugs
helm install my-nginx bitnami/nginx --version 18.1.6

# Alternative si problème avec bitnami, utiliser le repo officiel
# helm repo add nginx-stable https://helm.nginx.com/stable
# helm install my-nginx nginx-stable/nginx-ingress

# Vérifier l'installation
kubectl get all -l app.kubernetes.io/instance=my-nginx
helm status my-nginx
```

## Exercice 2 : Personnalisation de Values

```bash
# Voir les values par défaut
helm show values bitnami/nginx > values.yaml

# Créer un fichier custom-values.yaml
cat <<EOF > custom-values.yaml
replicaCount: 2
service:
  type: NodePort
  nodePorts:
    http: 30080
resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "200m"
EOF

# Installer avec les values personnalisées et version spécifique
helm install my-release bitnami/nginx -f custom-values.yaml --version 18.1.6

# Vérifier l'installation
kubectl get all -l app.kubernetes.io/instance=my-release
kubectl get service my-release-nginx
```

## Exercice 3 : Créer un Chart simple

```bash
# Créer la structure du chart
helm create my-app

# Examiner la structure créée
ls -la my-app/
cat my-app/Chart.yaml
```

```yaml
# my-app/values.yaml (éditer le fichier)
replicaCount: 1

image:
  repository: nginx
  tag: "1.29"
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: false

resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

```yaml
# my-app/templates/deployment.yaml (simplifier le fichier existant)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: { { include "my-app.fullname" . } }
  labels:
    { { - include "my-app.labels" . | nindent 4 } }
spec:
  replicas: { { .Values.replicaCount } }
  selector:
    matchLabels:
      { { - include "my-app.selectorLabels" . | nindent 6 } }
  template:
    metadata:
      labels:
        { { - include "my-app.selectorLabels" . | nindent 8 } }
    spec:
      containers:
        - name: { { .Chart.Name } }
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: { { .Values.image.pullPolicy } }
          ports:
            - containerPort: 80
          resources:
            { { - toYaml .Values.resources | nindent 12 } }
```

```bash
# Vérifier la syntaxe
helm lint my-app

# Tester le rendu des templates
helm template my-app ./my-app

# Installer le chart
helm install my-custom-app my-app

# Vérifier l'installation
kubectl get all -l app.kubernetes.io/instance=my-custom-app
```

## Exercice 4 : Gestion des releases

```bash
# Lister les releases
helm list

# Voir le statut d'une release
helm status my-custom-app

# Mettre à jour une release
helm upgrade my-custom-app my-app --set replicaCount=3

# Voir l'historique
helm history my-custom-app

# Rollback si nécessaire
helm rollback my-custom-app 1

# Vérifier le rollback
helm status my-custom-app
```

## Exercice 5 : Debug et test

```bash
# Voir ce qui sera installé sans l'installer
helm template my-app ./my-app

# Installer en mode dry-run
helm install test-release my-app --dry-run --debug

# Vérifier les values calculées
helm get values my-custom-app

# Voir les manifests déployés
helm get manifest my-custom-app
```

## Nettoyage

```bash
# Supprimer toutes les releases
helm uninstall my-nginx my-release my-custom-app

# Supprimer les repos ajoutés
helm repo remove bitnami
```
