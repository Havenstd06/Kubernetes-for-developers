# Lab : Admission Control avec Pod Security Standards

## Objectifs

- Comprendre le contrôle d'admission dans Kubernetes
- Implémenter des politiques de sécurité avec Pod Security Standards
- Tester la validation et le rejet de pods non conformes
- Observer le comportement des admission controllers natifs

## Exercice 1 : Configuration des namespaces de sécurité

### 1.1 Créer des namespaces avec différents niveaux

```yaml
# security-namespaces.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: privileged-ns
  labels:
    pod-security.kubernetes.io/enforce: privileged
    pod-security.kubernetes.io/audit: privileged
    pod-security.kubernetes.io/warn: privileged
---
apiVersion: v1
kind: Namespace
metadata:
  name: baseline-ns
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/audit: baseline
    pod-security.kubernetes.io/warn: baseline
---
apiVersion: v1
kind: Namespace
metadata:
  name: restricted-ns
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### 1.2 Appliquer et vérifier

```bash
kubectl apply -f security-namespaces.yaml
kubectl get namespaces --show-labels | grep "pod-security"
```

## Exercice 2 : Pod privilégié (violation de sécurité)

### 2.1 Créer un pod privilégié

```yaml
# privileged-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
    - name: nginx
      image: nginx
      securityContext:
        privileged: true
```

### 2.2 Tester dans différents namespaces

```bash
# Dans privileged-ns (doit réussir)
kubectl apply -f privileged-pod.yaml -n privileged-ns

# Dans baseline-ns (doit être rejeté)
kubectl apply -f privileged-pod.yaml -n baseline-ns

# Dans restricted-ns (doit être rejeté)
kubectl apply -f privileged-pod.yaml -n restricted-ns
```

## Exercice 3 : Pod conforme au niveau restricted

### 3.1 Créer un pod strictement conforme

```yaml
# restricted-compliant-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 101
    runAsGroup: 101
    fsGroup: 101
    seccompProfile:
      type: RuntimeDefault
  containers:
    - name: nginx
      image: nginxinc/nginx-unprivileged:alpine
      ports:
        - containerPort: 8080
      securityContext:
        allowPrivilegeEscalation: false
        readOnlyRootFilesystem: true
        capabilities:
          drop:
            - ALL
      volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: var-run
          mountPath: /var/run
        - name: var-cache
          mountPath: /var/cache/nginx
        - name: var-log
          mountPath: /var/log/nginx
  volumes:
    - name: tmp
      emptyDir: { }
    - name: var-run
      emptyDir: { }
    - name: var-cache
      emptyDir: { }
    - name: var-log
      emptyDir: { }
```

### 3.2 Tester la conformité

```bash
# Déployer dans restricted-ns (doit réussir)
kubectl apply -f restricted-compliant-pod.yaml -n restricted-ns

# Vérifier le déploiement
kubectl get pod secure-pod -n restricted-ns
kubectl describe pod secure-pod -n restricted-ns
```

## Exercice 4 : LimitRange (admission controller natif)

### 4.1 Créer un LimitRange

```yaml
# limit-range.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
  namespace: baseline-ns
spec:
  limits:
    - default:
        memory: "256Mi"
        cpu: "200m"
      defaultRequest:
        memory: "128Mi"
        cpu: "100m"
      max:
        memory: "1Gi"
        cpu: "1"
      min:
        memory: "64Mi"
        cpu: "50m"
      type: Container
```

### 4.2 Tester l'application automatique des limites

```yaml
# no-limits-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-limits-pod
spec:
  containers:
    - name: nginx
      image: nginx
```

```bash
# Appliquer le LimitRange
kubectl apply -f limit-range.yaml

# Déployer un pod sans limites
kubectl apply -f no-limits-pod.yaml -n baseline-ns

# Vérifier que les limites ont été appliquées automatiquement
kubectl describe pod no-limits-pod -n baseline-ns | grep -A 10 "Limits:"
```

### 4.3 Tester le rejet pour excès de ressources

```yaml
# excessive-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: excessive-pod
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        requests:
          memory: "2Gi"  # Dépasse max: 1Gi
```

```bash
# Tenter de déployer (doit être rejeté)
kubectl apply -f excessive-pod.yaml -n baseline-ns
```

## Exercice 5 : ResourceQuota (admission controller)

### 5.1 Créer un quota strict

```yaml
# resource-quota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: strict-quota
  namespace: baseline-ns
spec:
  hard:
    requests.cpu: "1"
    requests.memory: "1Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
    pods: "2"
```

### 5.2 Tester le respect du quota

```yaml
# quota-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: quota-test
  namespace: baseline-ns
spec:
  replicas: 3  # Dépasse pods: "2"
  selector:
    matchLabels:
      app: quota-test
  template:
    metadata:
      labels:
        app: quota-test
    spec:
      containers:
        - name: nginx
          image: nginx
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
```

```bash
# Appliquer le quota
kubectl apply -f resource-quota.yaml

# Tenter le déploiement (seulement 2 pods créés)
kubectl apply -f quota-deployment.yaml

# Vérifier les quotas utilisés
kubectl describe quota strict-quota -n baseline-ns
kubectl get pods -n baseline-ns
```

## Exercice 6 : Modes warn et audit

### 6.1 Namespace avec mode warn uniquement

```yaml
# warn-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: warn-only-ns
  labels:
    pod-security.kubernetes.io/warn: restricted
    # Pas de mode enforce = pod accepté avec warning
```

### 6.2 Tester avec un pod non conforme

```bash
# Appliquer le namespace
kubectl apply -f warn-namespace.yaml

# Déployer un pod privilégié (accepté mais avec warning)
kubectl apply -f privileged-pod.yaml -n warn-only-ns

# Observer le warning mais pas de rejet
kubectl get pod privileged-pod -n warn-only-ns
```

## Diagnostic et troubleshooting

```bash
# Voir les admission controllers configurés
kubectl get validatingwebhookconfiguration
kubectl get mutatingwebhookconfiguration

# Pour EKS, vérifier Pod Security Admission (pas visible via API)
kubectl get namespace kube-system -o yaml | grep pod-security

# Tester directement l'admission
kubectl apply --dry-run=server -f privileged-pod.yaml -n restricted-ns
```

## Validation des apprentissages

### Tests de compréhension

1. Créer un pod qui viole exactement une règle baseline
2. Configurer un namespace en mode audit seulement
3. Expliquer pourquoi kube-system est souvent exempté
4. Identifier les différences entre warn, audit et enforce

### Critères de validation

- Le pod privilégié est rejeté dans baseline/restricted
- Le pod conforme fonctionne dans tous les namespaces
- Les LimitRange appliquent des valeurs par défaut
- Les ResourceQuota empêchent les dépassements
- Les warnings apparaissent en mode warn

## Nettoyage complet

```bash
# Supprimer les pods par nom (pas par fichier)
kubectl delete pod privileged-pod -n privileged-ns --ignore-not-found
kubectl delete pod secure-pod -n restricted-ns --ignore-not-found
kubectl delete pod no-limits-pod -n baseline-ns --ignore-not-found
kubectl delete pod excessive-pod -n baseline-ns --ignore-not-found

# Supprimer les quotas et limits (avant les namespaces)
kubectl delete limitrange resource-limits -n baseline-ns --ignore-not-found
kubectl delete resourcequota strict-quota -n baseline-ns --ignore-not-found
kubectl delete deployment quota-test -n baseline-ns --ignore-not-found

# Supprimer les namespaces (supprime automatiquement tout le contenu)
kubectl delete namespace privileged-ns baseline-ns restricted-ns warn-only-ns --ignore-not-found
```

## Points clés

1. **Pod Security Standards** : GA depuis v1.25, remplace PodSecurityPolicy
2. **Trois niveaux** : privileged (permissif) → baseline → restricted (strict)
3. **Trois modes** : enforce (bloque), warn (avertit), audit (log)
4. **Configuration** : Via des labels sur les namespaces
5. **Production** : Privilégier restricted sauf justification technique
