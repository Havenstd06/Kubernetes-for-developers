# Jobs dans Kubernetes

## Exercice Pratique : Gestion des Jobs et CronJobs

### Objectif

Créer et gérer différents types de Jobs pour comprendre l'exécution de tâches batch dans Kubernetes.

### 1. Job simple

```yaml
# print-date-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: print-date
  labels:
    app: print-date
spec:
  template:
    spec:
      containers:
        - name: print-date
          image: busybox:1.28
          command: [ "sh", "-c", "date; echo 'Job terminé avec succès!'" ]
          resources:
            requests:
              memory: "32Mi"
              cpu: "100m"
            limits:
              memory: "64Mi"
              cpu: "200m"
      restartPolicy: Never
```

```bash
kubectl apply -f print-date-job.yaml
kubectl get jobs
kubectl get pods -l job-name=print-date
kubectl logs -l job-name=print-date
kubectl describe job print-date
```

### 2. Scénarios à tester

#### 2.1 Job avec gestion d'erreurs

```yaml
# retry-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: retry-job
spec:
  backoffLimit: 4
  activeDeadlineSeconds: 120
  template:
    spec:
      containers:
        - name: retry-container
          image: busybox:1.28
          command:
            - sh
            - -c
            - |
              echo "Tentative de traitement..."
              if [ $RANDOM -gt 20000 ]; then 
                echo "Succès!"
                exit 0
              else 
                echo "Échec simulé!"
                exit 1
              fi
      restartPolicy: Never
```

```bash
kubectl apply -f retry-job.yaml
kubectl get jobs retry-job -w
kubectl describe job retry-job
kubectl get pods -l job-name=retry-job
```

#### 2.2 Job parallèle avec completion tracking

```yaml
# parallel-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: parallel-job
spec:
  completions: 5
  parallelism: 2
  completionMode: Indexed
  template:
    spec:
      containers:
        - name: worker
          image: busybox:1.28
          command:
            - sh
            - -c
            - |
              index=${JOB_COMPLETION_INDEX:-0}
              echo "Worker $index: Starting processing"
              sleep $((3 + RANDOM % 5))
              echo "Worker $index: Processing completed"
      restartPolicy: Never
```

```bash
kubectl apply -f parallel-job.yaml
kubectl get pods -l job-name=parallel-job -w
kubectl logs -l job-name=parallel-job --prefix=true
```

#### 2.3 CronJob avec gestion d'historique

```yaml
# hello-cron.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/2 * * * *"
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  concurrencyPolicy: Allow
  startingDeadlineSeconds: 60
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: hello
              image: busybox:1.28
              command:
                - sh
                - -c
                - |
                  echo "=== CronJob Execution ==="
                  echo "Date: $(date)"
                  echo "Hostname: $(hostname)"
                  echo "========================="
          restartPolicy: Never
```

```bash
kubectl apply -f hello-cron.yaml
kubectl get cronjobs
kubectl get jobs --watch
```

### 3. Job de traitement de données avancé

```yaml
# data-processor.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-processor
spec:
  completions: 4
  parallelism: 2
  completionMode: Indexed
  template:
    spec:
      containers:
        - name: processor
          image: python:3.9-alpine
          command:
            - python
            - -c
            - |
              import time
              import random
              import os
              import sys

              job_index = os.environ.get('JOB_COMPLETION_INDEX', '0')
              print(f"[Batch {job_index}] Starting data processing...")

              # Simuler différents types de traitement
              tasks = ['validation', 'transformation', 'aggregation', 'export']
              task = tasks[int(job_index)]

              print(f"[Batch {job_index}] Executing {task} task")

              # Simuler un temps de traitement variable
              processing_time = random.randint(5, 15)
              for i in range(processing_time):
                  time.sleep(1)
                  if i % 3 == 0:
                      progress = (i + 1) / processing_time * 100
                      print(f"[Batch {job_index}] Progress: {progress:.1f}%")

              print(f"[Batch {job_index}] {task} completed successfully!")
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
      restartPolicy: Never
```

```bash
kubectl apply -f data-processor.yaml
kubectl get jobs data-processor -w
kubectl logs -l job-name=data-processor --prefix=true -f
```

### 4. Gestion des Jobs

```bash
# Lister tous les jobs
kubectl get jobs

# Voir l'état détaillé d'un job
kubectl describe job parallel-job

# Supprimer un job spécifique
kubectl delete job print-date

# Supprimer les jobs terminés automatiquement
kubectl delete jobs --field-selector status.successful=1

# Suspendre/reprendre un cronjob
kubectl patch cronjob hello-cron -p '{"spec":{"suspend":true}}'
kubectl patch cronjob hello-cron -p '{"spec":{"suspend":false}}'
```

### 5. Nettoyage

```bash
kubectl delete job retry-job parallel-job data-processor
kubectl delete cronjob hello-cron
```