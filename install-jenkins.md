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
    image: jenkins/jenkins:lts
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
          image: jenkins/jenkins:lts
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
  tag: "lts"
  
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
    - kubernetes:latest
    - workflow-aggregator:latest
    - git:latest
    - configuration-as-code:latest
    - blueocean:latest
    - docker-workflow:latest

  JCasC:
    enabled: true
    defaultConfig: true

persistence:
  enabled: true
  size: "10Gi"
  storageClass: "standard"

agent:
  enabled: true
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"

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
    containers:
      - name: jenkins-master
        image: jenkins/jenkins:lts
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

    plugins:
      - name: kubernetes
        version: "latest"
      - name: workflow-job
        version: "latest"
      - name: workflow-aggregator
        version: "latest"
      - name: git
        version: "latest"
      - name: job-dsl
        version: "latest"
      - name: configuration-as-code
        version: "latest"
      - name: blueocean
        version: "latest"

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
