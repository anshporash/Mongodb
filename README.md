#  **Mongodb**

##  **Mongodb cluster with single string ( url )**

This guide explains how to deploy a MongoDB replica set on kubernetes with:
   - 2MongoDB replica pod (one high config ,one low config)
   - 1 Arbiter for replica set quorum
   - A single connection string to connect via MongoDB Compass
   - Zero-downtime replica set configuration 
 ---
 ## Perequisites

 - kubectl configured
 - Helm (Optional)
 - MongoDB image ( mongo:6 or latest )
 - Persistent storage ( PVC )

---

## Step 1: Create Namespace 

  ```bash
    sudo kubectl create namespace mongodb-rs

 ```
---
## Step 2: Create MongoDB Config File 
 - **File name : mongo-configmap.yaml**

  ```bash
    apiVersion: v1
kind: ConfigMap
metadata:
  name: mongo-init
  namespace: mongodb-rs
data:
  init.sh: |
    #!/bin/bash
    mongosh --host mongo-0.mongo:27017 <<EOF
    rs.initiate({
      _id: "rs0",
      members: [
        { _id: 0, host: "mongo-0.mongo:27017", priority: 2 },
        { _id: 1, host: "mongo-1.mongo:27017", priority: 1 },
        { _id: 2, host: "arbiter.mongo:27017", arbiterOnly: true }
      ]
    })
    EOF
     

 ```
---

## Step 3: Deploy MongoDB StatefulSet  (Replica Pods)

 - **File name : mongo-statefulset.yaml**
 ```bash
  apiVersion: apps/v1
  kind: StatefulSet
metadata:
  name: mongo
  namespace: mongodb-rs
spec:
  serviceName: "mongo"
  replicas: 2
  selector:
    matchLabels:
      app: mongo
  template:
    metadata:
      labels:
        app: mongo
    spec:
      containers:
      - name: mongo
        image: mongo:6
        command: ["mongod"]
        args: ["--replSet", "rs0", "--bind_ip_all"]
        ports:
        - containerPort: 27017
        volumeMounts:
        - name: mongo-persistent-storage
          mountPath: /data/db
        resources:
          requests:
            cpu: "500m"
            memory: "1Gi"
          limits:
            cpu: "2"
            memory: "4Gi"
      volumes:
      - name: mongo-persistent-storage
        persistentVolumeClaim:
          claimName: mongo-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mongo-pvc
  namespace: mongodb-rs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi


```
---
## STEP 4:Create Arbiter Deployment 
 - **File name : mongo-arbiter.yaml**

 ```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  name: arbiter
  namespace: mongodb-rs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: arbiter
  template:
    metadata:
      labels:
        app: arbiter
    spec:
      containers:
      - name: mongo-arbiter
        image: mongo:6
        command: ["mongod"]
        args: ["--replSet", "rs0", "--bind_ip_all"]
        ports:
        - containerPort: 27017
```
---

## STEP 5: Headless Service  
 - **File name : mongo-service.yaml**

 ```bash
 apiVersion: v1
kind: Service
metadata:
  name: mongo
  namespace: mongodb-rs
spec:
  ports:
  - port: 27017
    name: mongo
  clusterIP: None
  selector:
    app: mongo

```
---

## STEP 6: Initialize Replica Set
- After all pods are running execute :
   ```bash
   Kubectl exec -it mongo-0 -n mongodb-rs --bash
   ```
 - Now you will be inside the MongoDB container (you will see a prompt like root@mongo-0:/#).
- **This intializes the replica set with:**
    - mongo-0 (Primary)
    - mongo-1 (Secondary)
    - arbiter (Arbiter)
1. **Open Mongo Shell (mongosh)**
  ```bash
    mongosh
  ```

2. **Priority set up**
 ```bash
    rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-0.mongo.mongodb-rs.svc.cluster.local:27017", priority: 2 },
    { _id: 1, host: "mongo-1.mongo.mongodb-rs.svc.cluster.local:27017", priority: 1 },
    { _id: 2, host: "arbiter-7d9c44d66f-w2cxg.arbiter.mongodb-rs.svc.cluster.local:27017", arbiterOnly: true }
  ]
})
  ```
-  Note :- The arbiter’s hostname may vary (you can check it with kubectl get pods -n mongodb-rs -o wide).
Replace the arbiter pod name (arbiter-7d9c44d66f-w2cxg) with your actual arbiter pod name.

- After pressing Enter,you should see:
  ```bash
  {"ok" : 1 }
    ```
 - This means your replica set was initialized successfully!
3. Check Replica Set Status
- Run this command inside `mongosh`:
 ```bash
rs.status()
   ```

- you should see output like this:
  ```bash
  {
  set: 'rs0',
  members: [
    { _id: 0, name: "mongo-0.mongo.mongodb-rs.svc.cluster.local:27017", stateStr: "PRIMARY" },
    { _id: 1, name: "mongo-1.mongo.mongodb-rs.svc.cluster.local:27017", stateStr: "SECONDARY" },
    { _id: 2, name: "arbiter-7d9c44d66f-w2cxg.arbiter.mongodb-rs.svc.cluster.local:27017", stateStr: "ARBITER" }
  ]
  }
  ```
  - Means everything is working:
    - `PRIMARY`- the main writable node
    - `SECONDARY`- replica
    - `ARBITER`- voting node  

4. Exit the shell
 ```bash
 exit
   ``` 
## STEP 7: Connection String (for MongoDB Compass)
- Use this connection URL in MongoDB Compass:
   ```bash
    
     ```
   - MongoDB Compass will automatically connect to the available primary node - ensuring no downtime if one node fails.
     


