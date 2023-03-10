---
author: "Haoxian WANG"
title: "Setup Devtron with K3D"
date: 2023-02-21T11:00:06+09:00
description: "Quick notes for my K3D cluster setup - Devtron part"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: Haoxian
authorEmoji: 👻
tags: 
- K3D
- Devtron
- Kubernetes 
- Docker
---

## Context
With K3D, it's not trivial for a beginner to work on it. I ran into Devtron accidentally and find that it's relatively simple to use with an interesting Web GUI. The installation is pretty neat and quick. 

## Install Helm 
A handy script from the official site of helm could help you: 
```shell
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```
Please be careful and check the shell script before execution. 
For more os-specific installation methods, please check https://helm.sh/docs/intro/install/ 

## Install Devtron 

1. installation with helm
```shell
helm install devtron devtron/devtron-operator --create-namespace --namespace devtroncd --set components.devtron.service.type=NodePort --set components.devtron.service.nodePort=30080
```

2. Run the following command to get the admin password 
```shell
kubectl -n devtroncd get secret devtron-secret -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d
```

3. Run the following command to get the dashboard URL for the service type -- `LoadBalancer` 
```shell
kubectl get svc -n devtroncd devtron-service -o jsonpath='{.status.loadBalancer.ingress}'
```
Use this command - `$ kubectl edit svc devtron-service -n devtroncd` to edit the service and change the service type to NodePort and NodePort port number to 30080 which you assigned while creating the cluster.
```YAML
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
kind: Service
metadata:
  annotations:
    helm.sh/resource-policy: keep
    meta.helm.sh/release-name: devtron
    meta.helm.sh/release-namespace: devtroncd
  creationTimestamp: "2022-12-31T00:19:47Z"
  finalizers:
  - svccontroller.k3s.cattle.io/daemonset
  labels:
    app: devtron
    app.kubernetes.io/managed-by: Helm
    release: devtron
  name: devtron-service
  namespace: devtroncd
  resourceVersion: "811"
  uid: b4e4f754-26c6-4f3b-afa0-651c1e3ed997
spec:
  allocateLoadBalancerNodePorts: true
  clusterIP: 10.43.203.52
  clusterIPs:
  - 10.43.203.52
  externalTrafficPolicy: Cluster
  internalTrafficPolicy: Cluster
  ipFamilies:
  - IPv4
  ipFamilyPolicy: SingleStack
  ports:
  - name: devtron
		nodePort: 32267  # change it to -> 30080 or the port you choose for devtron when creating the cluster 
    port: 80
    protocol: TCP
    targetPort: devtron
  selector:
    app: devtron
  sessionAffinity: None
  type: LoadBalancer # Change this to -> NodePort so that it expose directly to load balancer of k3d
		status:
		  loadBalancer:
		    ingress:
		    - ip: 172.31.0.2
		    - ip: 172.31.0.3
		    - ip: 172.31.0.4
```

Then following the hint to get your admin password with 
```shell
kubectl -n devtroncd get secret devtron-secret -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d
``` 
And you will be fine to visit `http://localhost:{THE_PORT_YOU_CHOSE_FOR_MAPPING_30080}`. 
If you used the same as the previous article, the URL will be http://localhost:8082

References 
https://devtron.ai/blog/k3d-for-local-kubernetes-development/ 