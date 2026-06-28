# Install Jenkins - Complete Guide

Tutorial instalasi Jenkins menggunakan 4 metode: Docker Compose, Kubernetes YAML, Helm, dan Jenkins Operator.

---

## Metode 1: Docker Compose

### Prerequisites
- Docker >= 20.x
- Docker Compose >= 2.x

### Step 1: Buat direktori project

```bash
mkdir jenkins-docker && cd jenkins-docker
```

### Step 2: Buat file `docker-compose.yml`

```yaml
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:2.555.3-lts-jdk21
    container_name: jenkins
    restart: unless-stopped
    privileged: true
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    networks:
      - jenkins_net

volumes:
  jenkins_home:
    driver: local

networks:
  jenkins_net:
    driver: bridge
```

### Step 3: Jalankan Jenkins

```bash
docker compose up -d
```

### Step 4: Cek status container

```bash
docker compose ps
docker compose logs -f jenkins
```

### Step 5: Ambil initial admin password

```bash
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

### Step 6: Akses Jenkins

Buka browser ke `http://localhost:8080`, masukkan password dari step 5, lalu install suggested plugins.

### Step 7: Stop / Remove Jenkins

```bash
# Stop
docker compose down

# Stop + hapus volume (data hilang)
docker compose down -v
```

---

## Metode 2: Kubernetes YAML (Manual Manifest)

### Prerequisites
- Kubernetes cluster (minikube / kubeadm / EKS / GKE)
- kubectl sudah terkonfigurasi

### Step 1: Buat Namespace

```bash
kubectl create namespace jenkins
```

### Step 2: Buat ServiceAccount dan RBAC

```yaml
# jenkins-rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-role
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/exec", "pods/log", "persistentvolumeclaims", "events"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["apps"]
    resources: ["deployments", "statefulsets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["secrets", "configmaps"]
    verbs: ["get", "list", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-role-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-role
subjects:
  - kind: ServiceAccount
    name: jenkins
    namespace: jenkins
```

```bash
kubectl apply -f jenkins-rbac.yaml
```

### Step 3: Buat PersistentVolumeClaim

```yaml
# jenkins-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jenkins-pvc
  namespace: jenkins
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

```bash
kubectl apply -f jenkins-pvc.yaml
```

### Step 4: Buat Deployment

```yaml
# jenkins-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jenkins
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jenkins
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      serviceAccountName: jenkins
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
      containers:
        - name: jenkins
          image: jenkins/jenkins:2.555.3-lts-jdk21
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 50000
              name: agent
          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "2Gi"
              cpu: "1000m"
          env:
            - name: JAVA_OPTS
              value: "-Xmx1500m"
          volumeMounts:
            - name: jenkins-data
              mountPath: /var/jenkins_home
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 90
            periodSeconds: 10
            failureThreshold: 5
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
      volumes:
        - name: jenkins-data
          persistentVolumeClaim:
            claimName: jenkins-pvc
```

```bash
kubectl apply -f jenkins-deployment.yaml
```

### Step 5: Buat Service

```yaml
# jenkins-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: jenkins
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: jenkins
  ports:
    - name: http
      port: 8080
      targetPort: 8080
      nodePort: 30080
    - name: agent
      port: 50000
      targetPort: 50000
```

```bash
kubectl apply -f jenkins-service.yaml
```

### Step 6: Cek status Pod

```bash
kubectl get all -n jenkins
kubectl logs -f deployment/jenkins -n jenkins
```

### Step 7: Ambil initial admin password

```bash
kubectl exec -n jenkins $(kubectl get pod -n jenkins -l app=jenkins -o jsonpath='{.items[0].metadata.name}') \
  -- cat /var/jenkins_home/secrets/initialAdminPassword
```

### Step 8: Akses Jenkins

```bash
# Jika menggunakan minikube
minikube service jenkins -n jenkins

# Jika menggunakan NodePort langsung
http://<NODE_IP>:30080
```

---

## Metode 3: Helm Chart

### Prerequisites
- Helm >= 3.x
- Kubernetes cluster
- kubectl terkonfigurasi

### Step 1: Tambah Helm repo Jenkins

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

### Step 2: Cek versi chart tersedia

```bash
helm search repo jenkins/jenkins
```

### Step 3: Buat Namespace

```bash
kubectl create namespace jenkins
```

### Step 4: Buat file `values.yaml` custom

```yaml
# values.yaml
controller:
  image: "jenkins/jenkins"
  tag: "2.555.3-lts-jdk21"
  
  resources:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "2000m"
      memory: "4096Mi"

  serviceType: NodePort
  nodePort: 30080

  adminUser: "admin"
  adminPassword: "admin123"  # ganti dengan password yang kuat

  installPlugins:
    - kubernetes:4423.vb_59f230b_ce53
    - workflow-aggregator:608.v67378e9d3db_1
    - git:5.10.1
    - configuration-as-code:2089.v970a_0b_a_8cc6d
    - blueocean:1.27.25
    - docker-workflow:634.vedc7242b_eda_7

  JCasC:
    enabled: true
    defaultConfig: true

persistence:
  enabled: true
  size: "10Gi"
  storageClass: "standard"

agent:
  enabled: true
  maxConcurrent: 10  # maksimal 10 agent pod berjalan bersamaan
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"

controller:
  numExecutors: 0  # master tidak eksekusi job langsung, semua ke agent

rbac:
  create: true
  readSecrets: true

serviceAccount:
  create: true
  name: jenkins
```

### Step 5: Install Jenkins via Helm

```bash
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --values values.yaml
```

### Step 6: Cek status deployment

```bash
helm status jenkins -n jenkins
kubectl get all -n jenkins
kubectl rollout status deployment/jenkins -n jenkins
```

### Step 7: Ambil admin password

```bash
kubectl exec --namespace jenkins -it svc/jenkins -c jenkins \
  -- /bin/cat /run/secrets/additional/chart-admin-password
```

### Step 8: Port-forward untuk akses lokal

```bash
kubectl port-forward svc/jenkins 8080:8080 -n jenkins
```

Akses di `http://localhost:8080`

### Step 9: Upgrade Helm release

```bash
helm upgrade jenkins jenkins/jenkins \
  --namespace jenkins \
  --values values.yaml
```

### Step 10: Uninstall

```bash
helm uninstall jenkins -n jenkins
kubectl delete pvc -n jenkins --all
```

---

### Backup dan Restore Manual (Helm)

Jenkins yang diinstall via Helm tidak punya fitur backup otomatis. Backup dilakukan menggunakan **CronJob Kubernetes** yang upload ke S3, dan restore dilakukan manual saat dibutuhkan.

#### Persiapan: Buat Secret AWS S3

```yaml
# jenkins-s3-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-aws-s3
  namespace: jenkins
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "YOUR_ACCESS_KEY_ID"
  AWS_SECRET_ACCESS_KEY: "YOUR_SECRET_ACCESS_KEY"
  AWS_DEFAULT_REGION: "ap-southeast-1"
  S3_BUCKET: "your-jenkins-backup-bucket"
```

```bash
kubectl apply -f jenkins-s3-secret.yaml
```

#### Backup: Buat CronJob

CronJob ini berjalan setiap malam pukul 02.00 dan menyimpan backup ke S3 dengan retensi 30 backup terakhir.

```yaml
# jenkins-backup-cronjob.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: jenkins-backup
  namespace: jenkins
spec:
  schedule: "0 2 * * *"        # setiap hari pukul 02.00
  concurrencyPolicy: Forbid     # jangan jalankan 2 backup sekaligus
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: amazon/aws-cli:2.27.8
              command:
                - /bin/sh
                - -c
                - |
                  set -e
                  TIMESTAMP=$(date +%Y%m%d-%H%M%S)
                  BACKUP_NAME="jenkins-backup-${TIMESTAMP}.tar.gz"
                  RETENTION_COUNT=30

                  echo "[INFO] Memulai backup: ${BACKUP_NAME}"

                  # Buat archive dari jenkins_home
                  tar -czf /tmp/${BACKUP_NAME} \
                    --exclude=/var/jenkins_home/workspace \
                    --exclude=/var/jenkins_home/war \
                    --exclude=/var/jenkins_home/caches \
                    --exclude=/var/jenkins_home/logs \
                    /var/jenkins_home

                  # Upload ke S3
                  aws s3 cp /tmp/${BACKUP_NAME} \
                    s3://${S3_BUCKET}/jenkins/backups/${BACKUP_NAME}
                  echo "[INFO] Upload selesai: s3://${S3_BUCKET}/jenkins/backups/${BACKUP_NAME}"

                  # Hapus file temp lokal
                  rm -f /tmp/${BACKUP_NAME}

                  # Retention: hapus backup lama, sisakan 30 terakhir
                  EXISTING=$(aws s3 ls s3://${S3_BUCKET}/jenkins/backups/ | sort | awk '{print $4}')
                  TOTAL=$(echo "${EXISTING}" | grep -c . || true)

                  if [ "${TOTAL}" -gt "${RETENTION_COUNT}" ]; then
                    DELETE_COUNT=$((TOTAL - RETENTION_COUNT))
                    echo "[INFO] Menghapus ${DELETE_COUNT} backup lama"
                    echo "${EXISTING}" | head -n ${DELETE_COUNT} | while read FILE; do
                      aws s3 rm s3://${S3_BUCKET}/jenkins/backups/${FILE}
                      echo "[INFO] Dihapus: ${FILE}"
                    done
                  fi

                  echo "[INFO] Backup selesai. Total: $(aws s3 ls s3://${S3_BUCKET}/jenkins/backups/ | wc -l)"
              env:
                - name: AWS_ACCESS_KEY_ID
                  valueFrom:
                    secretKeyRef:
                      name: jenkins-aws-s3
                      key: AWS_ACCESS_KEY_ID
                - name: AWS_SECRET_ACCESS_KEY
                  valueFrom:
                    secretKeyRef:
                      name: jenkins-aws-s3
                      key: AWS_SECRET_ACCESS_KEY
                - name: AWS_DEFAULT_REGION
                  valueFrom:
                    secretKeyRef:
                      name: jenkins-aws-s3
                      key: AWS_DEFAULT_REGION
                - name: S3_BUCKET
                  valueFrom:
                    secretKeyRef:
                      name: jenkins-aws-s3
                      key: S3_BUCKET
              resources:
                requests:
                  cpu: "100m"
                  memory: "128Mi"
                limits:
                  cpu: "500m"
                  memory: "512Mi"
              volumeMounts:
                - name: jenkins-home
                  mountPath: /var/jenkins_home
                  readOnly: true
          volumes:
            - name: jenkins-home
              persistentVolumeClaim:
                claimName: jenkins
```

```bash
kubectl apply -f jenkins-backup-cronjob.yaml

# Cek CronJob terdaftar
kubectl get cronjob -n jenkins

# Trigger backup manual (tanpa menunggu jadwal)
kubectl create job --from=cronjob/jenkins-backup jenkins-backup-manual -n jenkins

# Monitor progress backup
kubectl logs -f job/jenkins-backup-manual -n jenkins
```

#### Cek daftar backup di S3

```bash
kubectl run s3-list --rm -it --restart=Never \
  --image=amazon/aws-cli:2.27.8 \
  --env="AWS_ACCESS_KEY_ID=YOUR_KEY" \
  --env="AWS_SECRET_ACCESS_KEY=YOUR_SECRET" \
  --env="AWS_DEFAULT_REGION=ap-southeast-1" \
  -- s3 ls s3://your-jenkins-backup-bucket/jenkins/backups/ --recursive --human-readable
```

#### Restore: Dari S3 ke Jenkins

> **Penting:** Matikan Jenkins dulu sebelum restore agar tidak ada konflik write ke `jenkins_home`.

**Step 1 — Scale down Jenkins:**

```bash
kubectl scale deployment jenkins -n jenkins --replicas=0
kubectl rollout status deployment/jenkins -n jenkins
```

**Step 2 — Jalankan pod restore:**

```yaml
# jenkins-restore-job.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: jenkins-restore
  namespace: jenkins
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: restore
          image: amazon/aws-cli:2.27.8
          command:
            - /bin/sh
            - -c
            - |
              set -e

              echo "[INFO] Memulai proses restore..."

              # Tampilkan daftar backup tersedia
              echo "[INFO] Backup tersedia:"
              aws s3 ls s3://${S3_BUCKET}/jenkins/backups/ | sort

              # Ambil backup terbaru (atau set manual: BACKUP_FILE=jenkins-backup-YYYYMMDD-HHMMSS.tar.gz)
              LATEST=$(aws s3 ls s3://${S3_BUCKET}/jenkins/backups/ | sort | tail -n 1 | awk '{print $4}')

              if [ -z "${LATEST}" ]; then
                echo "[ERROR] Tidak ada backup ditemukan!"
                exit 1
              fi

              echo "[INFO] Restore dari: ${LATEST}"

              # Download dari S3
              aws s3 cp s3://${S3_BUCKET}/jenkins/backups/${LATEST} /tmp/${LATEST}

              # Kosongkan jenkins_home dulu (kecuali folder workspace)
              find /var/jenkins_home -mindepth 1 \
                ! -path "/var/jenkins_home/workspace*" \
                -delete || true

              # Extract backup
              tar -xzf /tmp/${LATEST} -C /

              # Hapus file temp
              rm -f /tmp/${LATEST}

              echo "[INFO] Restore selesai dari: ${LATEST}"
          env:
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: jenkins-aws-s3
                  key: AWS_ACCESS_KEY_ID
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: jenkins-aws-s3
                  key: AWS_SECRET_ACCESS_KEY
            - name: AWS_DEFAULT_REGION
              valueFrom:
                secretKeyRef:
                  name: jenkins-aws-s3
                  key: AWS_DEFAULT_REGION
            - name: S3_BUCKET
              valueFrom:
                secretKeyRef:
                  name: jenkins-aws-s3
                  key: S3_BUCKET
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
      volumes:
        - name: jenkins-home
          persistentVolumeClaim:
            claimName: jenkins
```

```bash
kubectl apply -f jenkins-restore-job.yaml

# Monitor proses restore
kubectl logs -f job/jenkins-restore -n jenkins
```

**Step 3 — Nyalakan kembali Jenkins setelah restore selesai:**

```bash
kubectl scale deployment jenkins -n jenkins --replicas=1
kubectl rollout status deployment/jenkins -n jenkins
```

**Step 4 — Bersihkan job restore:**

```bash
kubectl delete job jenkins-restore -n jenkins
```

#### Restore dari backup tertentu (bukan yang terbaru)

Edit bagian ini di `jenkins-restore-job.yaml`:

```bash
# Ganti baris LATEST dengan nama file spesifik
LATEST="jenkins-backup-20260628-020000.tar.gz"
```

---

## Metode 4: Jenkins Operator

Jenkins Operator adalah Kubernetes Operator yang mengelola Jenkins secara full otomatis termasuk backup, restore, dan konfigurasi.

### Prerequisites
- Kubernetes >= 1.17
- kubectl terkonfigurasi
- Helm >= 3.x (opsional)

### Step 1: Install Jenkins Operator via Helm

```bash
# Tambah repo
helm repo add jenkins https://raw.githubusercontent.com/jenkinsci/kubernetes-operator/master/chart
helm repo update

# Buat namespace
kubectl create namespace jenkins

# Install operator
helm install jenkins-operator jenkins/jenkins-operator \
  --namespace jenkins \
  --set jenkins.enabled=false
```

### Step 2: Verifikasi Operator berjalan

```bash
kubectl get pods -n jenkins
kubectl logs deployment/jenkins-operator -n jenkins
```

### Step 3: Buat Secret untuk admin credentials

```bash
kubectl create secret generic jenkins-operator-credentials \
  --from-literal=user=admin \
  --from-literal=password=admin123 \
  --namespace jenkins
```

### Step 3.1: Buat Secret untuk AWS S3

```yaml
# jenkins-s3-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: jenkins-aws-s3
  namespace: jenkins
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "YOUR_ACCESS_KEY_ID"
  AWS_SECRET_ACCESS_KEY: "YOUR_SECRET_ACCESS_KEY"
  AWS_DEFAULT_REGION: "ap-southeast-1"
  S3_BUCKET: "your-jenkins-backup-bucket"
```

```bash
kubectl apply -f jenkins-s3-secret.yaml
```

### Step 3.2: Buat ConfigMap untuk script backup dan restore

```yaml
# jenkins-backup-scripts.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-backup-scripts
  namespace: jenkins
data:
  backup.sh: |
    #!/bin/bash
    set -e

    TIMESTAMP=$(date +%Y%m%d-%H%M%S)
    BACKUP_NAME="jenkins-backup-${TIMESTAMP}.tar.gz"
    RETENTION_COUNT=30

    echo "[INFO] Memulai backup: ${BACKUP_NAME}"

    # Buat archive dari jenkins_home, exclude folder yang tidak penting
    tar -czf /tmp/${BACKUP_NAME} \
      --exclude=/var/jenkins_home/workspace \
      --exclude=/var/jenkins_home/war \
      --exclude=/var/jenkins_home/caches \
      --exclude=/var/jenkins_home/logs \
      /var/jenkins_home

    # Upload ke S3
    aws s3 cp /tmp/${BACKUP_NAME} s3://${S3_BUCKET}/jenkins/backups/${BACKUP_NAME}
    echo "[INFO] Upload selesai: s3://${S3_BUCKET}/jenkins/backups/${BACKUP_NAME}"

    # Hapus file temp lokal
    rm -f /tmp/${BACKUP_NAME}

    # Retention: hapus backup lama, sisakan RETENTION_COUNT terakhir
    EXISTING=$(aws s3 ls s3://${S3_BUCKET}/jenkins/backups/ | sort | awk '{print $4}')
    TOTAL=$(echo "${EXISTING}" | wc -l)

    if [ "${TOTAL}" -gt "${RETENTION_COUNT}" ]; then
      DELETE_COUNT=$((TOTAL - RETENTION_COUNT))
      echo "[INFO] Menghapus ${DELETE_COUNT} backup lama (retention: ${RETENTION_COUNT})"
      echo "${EXISTING}" | head -n ${DELETE_COUNT} | while read FILE; do
        aws s3 rm s3://${S3_BUCKET}/jenkins/backups/${FILE}
        echo "[INFO] Dihapus: ${FILE}"
      done
    fi

    echo "[INFO] Backup selesai. Total backup: $(aws s3 ls s3://${S3_BUCKET}/jenkins/backups/ | wc -l)"

  restore.sh: |
    #!/bin/bash
    set -e

    echo "[INFO] Memulai proses restore..."

    # Ambil backup terbaru dari S3
    LATEST=$(aws s3 ls s3://${S3_BUCKET}/jenkins/backups/ | sort | tail -n 1 | awk '{print $4}')

    if [ -z "${LATEST}" ]; then
      echo "[ERROR] Tidak ada backup ditemukan di s3://${S3_BUCKET}/jenkins/backups/"
      exit 1
    fi

    echo "[INFO] Restore dari backup: ${LATEST}"

    # Download dari S3
    aws s3 cp s3://${S3_BUCKET}/jenkins/backups/${LATEST} /tmp/${LATEST}

    # Extract ke /
    tar -xzf /tmp/${LATEST} -C /

    # Hapus file temp
    rm -f /tmp/${LATEST}

    echo "[INFO] Restore selesai dari: ${LATEST}"
```

```bash
kubectl apply -f jenkins-backup-scripts.yaml
```

### Step 4: Buat Custom Resource Jenkins

```yaml
# jenkins-instance.yaml
apiVersion: jenkins.io/v1alpha2
kind: Jenkins
metadata:
  name: jenkins
  namespace: jenkins
spec:
  configurationAsCode:
    enabled: true
    configurations: []

  master:
    disableCSRFProtection: false
    executorCount: 0
    maxParallelAgents: 10
    containers:
      - name: jenkins-master
        image: jenkins/jenkins:2.555.3-lts-jdk21
        imagePullPolicy: Always
        resources:
          limits:
            cpu: "1500m"
            memory: "3Gi"
          requests:
            cpu: "500m"
            memory: "500Mi"
        env:
          - name: JAVA_OPTS
            value: >-
              -Xmx1500m
              -XX:MaxRAMFraction=1
              -Djenkins.install.runSetupWizard=false
        volumeMounts:
          - name: jenkins-home
            mountPath: /var/jenkins_home

      - name: backup
        image: amazon/aws-cli:2.27.8
        command:
          - /bin/sh
          - -c
          - "while true; do sleep 3600; done"
        env:
          - name: AWS_ACCESS_KEY_ID
            valueFrom:
              secretKeyRef:
                name: jenkins-aws-s3
                key: AWS_ACCESS_KEY_ID
          - name: AWS_SECRET_ACCESS_KEY
            valueFrom:
              secretKeyRef:
                name: jenkins-aws-s3
                key: AWS_SECRET_ACCESS_KEY
          - name: AWS_DEFAULT_REGION
            valueFrom:
              secretKeyRef:
                name: jenkins-aws-s3
                key: AWS_DEFAULT_REGION
          - name: S3_BUCKET
            valueFrom:
              secretKeyRef:
                name: jenkins-aws-s3
                key: S3_BUCKET
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "300m"
            memory: "256Mi"
        volumeMounts:
          - name: jenkins-home
            mountPath: /var/jenkins_home
          - name: backup-scripts
            mountPath: /bin/backup.sh
            subPath: backup.sh
          - name: backup-scripts
            mountPath: /bin/restore.sh
            subPath: restore.sh

    volumes:
      - name: backup-scripts
        configMap:
          name: jenkins-backup-scripts
          defaultMode: 0755

    plugins:
      - name: kubernetes
        version: "4423.vb_59f230b_ce53"
      - name: workflow-job
        version: "1581.ve4b_d0db_fcb_b_b_"
      - name: workflow-aggregator
        version: "608.v67378e9d3db_1"
      - name: git
        version: "5.10.1"
      - name: job-dsl
        version: "3654.vdf58f53e2d15"
      - name: configuration-as-code
        version: "2089.v970a_0b_a_8cc6d"
      - name: blueocean
        version: "1.27.25"

  seedJobs:
    - id: jenkins-operator
      targets: "cicd/jobs/*.jenkins"
      description: "Jenkins Operator Seed Job"
      repositoryBranch: main
      repositoryUrl: https://github.com/your-org/your-repo.git

  backup:
    containerName: backup
    action:
      exec:
        command:
          - /bin/backup.sh
    interval: 30
    makeBackupBeforePodDeletion: true

  restore:
    containerName: backup
    action:
      exec:
        command:
          - /bin/restore.sh
    recoveryOnce: 0

  notifications:
    - loggingLevel: WARNING
      verbose: true
      name: slack-notification
      slack:
        webHookURLSecretKeySelector:
          secret:
            name: jenkins-slack-webhook
          key: webhookurl
```

```bash
kubectl apply -f jenkins-instance.yaml
```

### Step 5: Monitor proses instalasi operator

```bash
# Cek status Jenkins CR
kubectl get jenkins -n jenkins

# Cek detail
kubectl describe jenkins jenkins -n jenkins

# Cek pod yang dibuat operator
kubectl get pods -n jenkins -w
```

### Step 6: Ambil credentials dari secret

```bash
kubectl get secret jenkins-operator-credentials-jenkins \
  -n jenkins \
  -o jsonpath='{.data.user}' | base64 -d

kubectl get secret jenkins-operator-credentials-jenkins \
  -n jenkins \
  -o jsonpath='{.data.password}' | base64 -d
```

### Step 7: Port-forward untuk akses

```bash
kubectl port-forward svc/jenkins-operator-http-jenkins 8080:8080 -n jenkins
```

Akses di `http://localhost:8080`

### Step 8: Cek log operator jika ada masalah

```bash
kubectl logs deployment/jenkins-operator -n jenkins -f
kubectl logs pod/jenkins-jenkins-0 -n jenkins -f
```

---

## Perbandingan Metode

| Fitur | Docker Compose | K8s YAML | Helm | Operator |
|-------|---------------|----------|------|----------|
| Kompleksitas setup | Rendah | Sedang | Sedang | Tinggi |
| Production ready | Tidak | Ya | Ya | Ya |
| Auto backup/restore | Manual | Manual | Manual | Otomatis |
| GitOps friendly | Tidak | Ya | Ya | Ya |
| Plugin management | Manual | Manual | values.yaml | CR spec |
| Cocok untuk | Dev/lokal | Staging | Production | Production enterprise |

## Rekomendasi

- **Development / Lokal** → Docker Compose
- **Staging / Kecil** → Kubernetes YAML
- **Production** → Helm (lebih fleksibel dan mudah upgrade)
- **Enterprise / Production besar** → Jenkins Operator (full lifecycle management)

---

## Konfigurasi Max Agent Pod (Limit 10)

### Helm — via `values.yaml`

Sudah dikonfigurasi di `values.yaml` pada Metode 3:

```yaml
agent:
  maxConcurrent: 10

controller:
  numExecutors: 0
```

### Kubernetes YAML — via Jenkins UI / JCasC

Jika install menggunakan Kubernetes YAML manual, set limit lewat Jenkins UI:

1. Masuk ke **Manage Jenkins → Clouds → Kubernetes**
2. Set **Max concurrent agents**: `10`
3. Klik **Save**

Atau via ConfigMap JCasC:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: jenkins-casc
  namespace: jenkins
data:
  jenkins.yaml: |
    jenkins:
      numExecutors: 0
    clouds:
      - kubernetes:
          name: "kubernetes"
          serverUrl: "https://kubernetes.default.svc"
          namespace: "jenkins"
          maxRequestsPerHostStr: "10"
          containerCapStr: "10"
          templates:
            - name: "jenkins-agent"
              namespace: "jenkins"
              label: "jenkins-agent"
              containers:
                - name: "jnlp"
                  image: "jenkins/inbound-agent:3383.vc8881d4b_0e76-1-jdk21"
                  resourceRequestCpu: "200m"
                  resourceRequestMemory: "256Mi"
                  resourceLimitCpu: "500m"
                  resourceLimitMemory: "512Mi"
```

```bash
kubectl apply -f jenkins-casc.yaml -n jenkins
```

### Rangkuman Pod Count

| Kondisi | Master Pod | Agent Pod | Total |
|---------|-----------|-----------|-------|
| Idle | 1 | 0 | **1** |
| Ada build berjalan | 1 | 1–10 | **2–11** |
| Full capacity | 1 | 10 | **11** |
