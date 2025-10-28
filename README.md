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
## Step 2: Create MongoDB Config File 
 - FIle name : mongo-configmap.yaml

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

## Step 3: DEploy MongoDB StatefulSet  (Replica Pods)
  
