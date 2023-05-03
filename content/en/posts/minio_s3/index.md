---
author: "Haoxian WANG"
title: "[K8S] Setup Minio S3 Storage in K3S"
date: 2023-03-27T11:00:06+09:00
description: "Setup Minio S3 Storage"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: Haoxian
authorEmoji: 👻
tags: 
- K3S
- Devtron
- K8S
- Kubernetes 
- Docker
- S3
- Minio
---


## Context
I wanted to test DVC, but it would be better to have a S3-compatible server as storage backend for DVC. I chose Minio because I am kind of with it. 

I set up Minio Operator at the very beginning and then found out it’s a little bit overkill for my use case since it can devour a lot of resources. Then I switched to Minio object storage on K8S which is more suitable for Homelab and learning purpose. For production purposes, it is recommended to use the Minio Operator.

Minio is an object storage server that is compatible with Amazon S3 APIs. It is designed to be simple, lightweight, and easy to set up. 

To use Minio on K8S, you will need to deploy the Minio server as a pod. This can be done using the Kubernetes deployment object. Once the pod is running, you can access the Minio server using the service object.
## Deployment
To deploy the Minio server, you will need to create a Kubernetes deployment object using the Minio Docker image. You can then expose the deployment as a service using a Kubernetes service object. The service object will allow you to access the Minio server using a stable IP address via the previous mentioned MetalLB with LoadBalancer or NodePort. 

### Creation of a persistent volume
To create a persistent volume with filesystem mode in Kubernetes, you can use the following example YAML file:

```YAML
kind: PersistentVolume
apiVersion: v1
metadata:
  name: minio-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /mnt/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - your-node-name

```

This file will create a persistent volume with a storage capacity of 20 gigabytes and a local storage path of `/mnt/data` on the node with the name `your-node-name`. You can adjust the storage capacity and the local storage path as needed. Once you have created the persistent volume YAML file, you can apply it to your Kubernetes cluster using the `kubectl apply` command:

```Shell
kubectl apply -f minio-pv.yaml

```

Replace `your-pv-file.yaml` with the path to your persistent volume YAML file. Once the persistent volume is created, you can use its name in the persistent volume claim YAML file to request storage from the persistent volume.

### Creation of a persistent volume claim

```YAML
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: minio-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
  volumeMode: Filesystem
  storageClassName: local-storage
  selector:
    matchLabels:
      type: local

```

This file will create a persistent volume claim with a storage request of 20 gigabytes and a storage class of `local-storage`. The `volumeMode` field is set to `Filesystem` to indicate that the persistent volume will be used as a file system. Once you have created the persistent volume claim YAML file, you can apply it to your Kubernetes cluster using the `kubectl apply` command:

```Shell
kubectl apply -f minio-pvc.yaml

```
### Deploy Minio server
```YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minio-deployment
  namespace: minio
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minio
  template:
    metadata:
      labels:
        app: minio
    spec:
      containers:
      - name: minio
        image: minio/minio
        args:
        - server
        - /data
        - --console-address=0.0.0.0:9001
        env:
        - name: MINIO_ACCESS_KEY
          value: minio
        - name: MINIO_SECRET_KEY
          value: minio123
        ports:
        - containerPort: 9000
        - containerPort: 9001
        volumeMounts:
        - name: minio-persistent-storage
          mountPath: /data
      volumes:
      - name: minio-persistent-storage
        persistentVolumeClaim:
          claimName: minio-pvc
---
apiVersion: v1
kind: Service
metadata:
  name: minio-service
  namespace: minio
spec:
  selector:
    app: minio
  ports:
  - name: console
    port: 9000
    targetPort: 9000
  - name: api
    port: 9001
    targetPort: 9001
  type: LoadBalancer
  externalTrafficPolicy: Local
```

### Use MC to access resources on Minio server
Assuming that you have set up Minio server and created a bucket, you can use the `mc` command-line tool to connect to the bucket. First, download and install the `mc` tool from the official website: [https://docs.min.io/docs/minio-client-complete-guide.html](https://docs.min.io/docs/minio-client-complete-guide.html)

Once you have installed `mc`, you can use the following command to configure it to connect to your Minio server:

```Shell
mc config host add myminio http://<minio-server-ip>:<minio-server-port> <access-key> <secret-key>
```

Replace `<minio-server-ip>` and `<minio-server-port>` with the IP address and port number of your Minio server, respectively. Replace `<access-key>` and `<secret-key>` with your Minio access key and secret key, respectively.

After you have configured `mc`, you can use it to interact with your Minio server. For example, you can list the contents of a bucket with the following command:

```Shell
mc ls myminio/mybucket
```

Replace `mybucket` with the name of your bucket. You can also copy files to and from your bucket using `mc`. For example, you can upload a file to your bucket with the following command:

```Shell
mc cp /path/to/local/file myminio/mybucket/path/to/remote/file
```

Replace `/path/to/local/file` with the path to the local file that you want to upload. Replace `mybucket/path/to/remote/file` with the path to the remote file in your bucket.

For more information on how to use `mc`, please refer to the official documentation: [https://docs.min.io/docs/minio-client-complete-guide.html](https://docs.min.io/docs/minio-client-complete-guide.html)