# deploy_bamboo_in_k8s_cluster
deploying bamboo in k8s cluster using helm,k8s and ngrok

1. Create a Namespace
Define a dedicated namespace for Bamboo.
kubectl create namespace bamboo

2. Prepare and deploy the Database"PostgreSQL"using Helm.
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install bamboo-postgres bitnami/postgresql --namespace bamboo \
  --set auth.username=bamboo,auth.password=securepassword,auth.database=bamboo

3- create the pv and the pvc to presist the data as shown
but first apply this 2 commands : 
sudo mkdir -p /mnt/data/bamboo
sudo chmod -R 777 /mnt/data/bamboo

# bamboo-pv.yaml
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

---
# bamboo-pvc.yaml
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


4- create a deployment for bamboo as show
# bamboo-deployment.yaml
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
5- create a nodeport service for this bamboo deployment
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

6- sign up with ngrok at https://dashboard.ngrok.com/get-started/setup/linux     
then expose the deployment with ngrok http http://localhost:32085 
      
