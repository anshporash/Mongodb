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
