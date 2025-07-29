# MongoDB ReplicaSet on Kubernetes (Minikube) using Helm Chart (Community Operator)

This repository provides a step-by-step guide to deploy a **MongoDB ReplicaSet** with **authentication** and **external access** using the **MongoDB Community Kubernetes Operator** deployed via **Helm**.

---

## 📋 Prerequisites

- [Minikube](https://minikube.sigs.k8s.io/docs/start/) (v1.32+ recommended)
- `kubectl` installed
- `helm` CLI installed
- Basic knowledge of Kubernetes and MongoDB

---

## 🚀 Setup Steps

### 1. Create Namespace

```bash
kubectl create namespace mongo
```
---

### 2. Install MongoDB Community Operator via Helm
```bash
helm repo add mongodb https://mongodb.github.io/helm-charts
helm repo update
helm install mongodb-operator mongodb/community-operator -n mongo
```
---

### 3. Create MongoDB Admin Secret
```bash
kubectl -n mongo create secret generic mongodb-password \
  --from-literal=MONGODB_DATABASE=admin \
  --from-literal=MONGODB_USERNAME=admin \
  --from-literal=MONGODB_PASSWORD=YourStrongPassword
```
----  
### 4. Deploy MongoDB ReplicaSet Resource

Save the following as mongodb-replicaset.yaml:

```bash
apiVersion: mongodbcommunity.mongodb.com/v1
kind: MongoDBCommunity
metadata:
  name: mongo-cluster
  namespace: mongo
spec:
  members: 3
  type: ReplicaSet
  version: "6.0.13"
  security:
    authentication:
      modes: ["SCRAM"]
  users:
    - name: admin
      db: admin
      passwordSecretRef:
        name: mongodb-password
      roles:
        - name: root
          db: admin
  additionalMongodConfig:
    net:
      bindIpAll: true
```	  
Apply it:

```bash
kubectl apply -f mongodb-replicaset.yaml
```
---
### 5. Expose One Replica Node via NodePort
Create mongodb-external-service.yaml:
```bash
apiVersion: v1
kind: Service
metadata:
  name: mongo-external
  namespace: mongo
spec:
  type: NodePort
  selector:
    statefulset.kubernetes.io/pod-name: mongo-cluster-0
  ports:
    - name: mongo
      port: 27017
      targetPort: 27017
      nodePort: 32019
```
Apply it:
```bash
kubectl apply -f mongodb-external-service.yaml
```
---
### 6. Get Minikube Node IP
```bash
minikube ip
```
---

### 7. Use Minikube service to access node port
Instead of accessing the NodePort manually, use:
```bash
minikube service mongo-external -n mongo
```
This will:

* Automatically open a tunnel to the exposed port

* Show the correct external URL like:
http://127.0.0.1:<some_port>

You can then connect from MongoDB Compass or other such tools using this port.
---
### 8. Connect from MongoDB Compass or Shell
Replace the IP and password below:

```bash
mongodb://admin:<password>@127.0.0.1:<some_port>/?authSource=admin&directConnection=true
```
---
### 9. Clean Up Resources (Optional)
```bash
kubectl delete -f mongodb-external-service.yaml
kubectl delete -f mongodb-replicaset.yaml
kubectl delete secret mongodb-password -n mongo
helm uninstall mongodb-operator -n mongo
kubectl delete namespace mongo
```
---

📝 Notes
Ensure Minikube has enough resources: minikube start --cpus=4 --memory=8g

For external access to all replicas, consider Ingress or custom LoadBalancer solutions.

This guide is ideal for local testing or learning purposes.
----

📫 Feedback or Contributions
Feel free to fork and improve! For issues or suggestions, raise a pull request or open an issue on this repo.

