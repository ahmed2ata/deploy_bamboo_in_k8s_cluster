# Deploy Bamboo in a Kubernetes Cluster

This guide provides step-by-step instructions for deploying Atlassian Bamboo in a Kubernetes cluster using Helm, Kubernetes manifests, and Ngrok for external access.

---

## Prerequisites
- A running Kubernetes cluster.
- `kubectl` installed and configured.
- Helm installed and configured.
- Ngrok installed (sign up and set up via [Ngrok Setup](https://dashboard.ngrok.com/get-started/setup/linux)).

---

## Steps to Deploy Bamboo

### 1. Create a Namespace
Create a dedicated namespace for the Bamboo deployment:
```bash
kubectl create namespace bamboo
```

### 2. Deploy PostgreSQL Database Using Helm
Install PostgreSQL using the Bitnami Helm chart:
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install bamboo-postgres bitnami/postgresql --namespace bamboo \
  --set auth.username=bamboo,auth.password=securepassword,auth.database=bamboo
```

### 3. Create Persistent Volume (PV) and Persistent Volume Claim (PVC)
Persistent storage is required for Bamboo to retain data.

#### Create the Host Directory
Run the following commands on your Kubernetes nodes:
```bash
sudo mkdir -p /mnt/data/bamboo
sudo chmod -R 777 /mnt/data/bamboo
```

#### Apply the PV and PVC Configurations

**`bamboo-pv.yaml`**:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: bamboo-pv
  labels:
    app: bamboo
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data/bamboo
```

**`bamboo-pvc.yaml`**:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bamboo-pvc
  namespace: bamboo
spec:
  accessModes:
    - ReadWriteOnce
  resources:
      requests:
        storage: 10Gi
```
Apply the files:
```bash
kubectl apply -f bamboo-pv.yaml
kubectl apply -f bamboo-pvc.yaml
```

### 4. Deploy Bamboo
Create a deployment for Bamboo with the following configuration:

**`bamboo-deployment.yaml`**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bamboo
  namespace: bamboo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bamboo
  template:
    metadata:
      labels:
        app: bamboo
    spec:
      containers:
        - name: bamboo
          image: atlassian/bamboo:latest
          ports:
            - containerPort: 8085
          volumeMounts:
            - name: bamboo-data
              mountPath: /var/atlassian/application-data/bamboo
          env:
            - name: BAMBOO_HOME
              value: /var/atlassian/application-data/bamboo
            - name: DATABASE_URL
              value: jdbc:postgresql://bamboo-postgres:5432/bamboo
            - name: DATABASE_USERNAME
              value: bamboo
            - name: DATABASE_PASSWORD
              value: securepassword
      volumes:
        - name: bamboo-data
          persistentVolumeClaim:
            claimName: bamboo-pvc
```
Apply the deployment:
```bash
kubectl apply -f bamboo-deployment.yaml
```

### 5. Create a NodePort Service
Expose the Bamboo deployment using a NodePort service:

**`bamboo-service.yaml`**:
```yaml
apiVersion: v1
kind: Service
metadata:
  name: bamboo
  namespace: bamboo
spec:
  type: NodePort
  selector:
    app: bamboo
  ports:
    - name: bamboo-port
      protocol: TCP
      port: 8085      # Bamboo's internal service port
      targetPort: 8085 # Bamboo container port
      nodePort: 32085  # External access NodePort
```
Apply the service:
```bash
kubectl apply -f bamboo-service.yaml
```

### 6. Expose Bamboo with Ngrok
1. Start Ngrok and expose the NodePort service:
   ```bash
   ngrok http http://localhost:32085
   ```

2. Copy the public Ngrok URL displayed in the terminal (e.g., `https://<random-id>.ngrok.io`).
3. Access Bamboo using the Ngrok URL in your browser.

---

- **Security**: Ensure sensitive credentials (e.g., database password) are managed securely (consider using Kubernetes Secrets).
- **Customization**: Adjust storage size, replicas, or ports as per your requirements.
- **Scaling**: This guide deploys a single Bamboo instance. For production, consider scaling and setting up proper ingress.


