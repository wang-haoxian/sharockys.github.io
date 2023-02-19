---
author: "Haoxian WANG"
title: "Setup K3D and Devtron for Homelab"
date: 2023-02-18T12:00:06+09:00
description: "Quick notes for my K3D cluster setup"
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
Recently, I am trying to make full use of my i7-12700 CPU and the 96G RAM that I install for it. The ideal thing for me is to be able to practice MLOps on a k8s cluster. However, it's never easy for bare-metal devices to make a whole cluster with many nodes. Maybe with Docker Desktop it will be easy, but then I may lose the chance to learn. 

I have tested microk8s before. However, there are always some weird things that stop me from just getting the basic control plane. This is probably because it was installed with snap. Anyways, I ran into different solutions. Another solution that I have tested is Minikube. Nice and clean but for a single node. I would like to make it more complicated. K3S is a good one and I have installed it on three low-end laptops with my NAS to make a 1 server and 3 agents cluster. It was funny and I succeeded to run some applications on it. Now that I am on a single machine and I want to simulate a multi-node cluster, the ideal way would be to create several VMs and set up the cluster with them which is tedious. To avoid this, some automation tools like Ansible are one of the best choices.  I am not yet ready to get my hands dirty on VMs while my objective is to learn to make things on K8S. K3D is a good alternative based on K3S. 

K3D is a docker version of K3S. It uses docker to simulate nodes and Docker in Docker to run apps on it. 

## Quick start with K3D 
The installation is as handy as the docker installation script: 
```Bash
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

To create a cluster:
```Bash 
k3d cluster create test-cluster
```

```Output
INFO[0000] Prep: Network                                
INFO[0000] Created network 'k3d-test-cluster'           
INFO[0000] Created image volume k3d-test-cluster-images 
INFO[0000] Starting new tools node...                   
INFO[0000] Starting Node 'k3d-test-cluster-tools'       
INFO[0001] Creating node 'k3d-test-cluster-server-0'    
INFO[0001] Creating LoadBalancer 'k3d-test-cluster-serverlb' 
INFO[0001] Using the k3d-tools node to gather environment information 
INFO[0001] HostIP: using network gateway 192.168.144.1 address 
INFO[0001] Starting cluster 'test-cluster'              
INFO[0001] Starting servers...                          
INFO[0001] Starting Node 'k3d-test-cluster-server-0'    
INFO[0005] All agents already running.                  
INFO[0005] Starting helpers...                          
INFO[0005] Starting Node 'k3d-test-cluster-serverlb'    
INFO[0011] Injecting records for hostAliases (incl. host.k3d.internal) and for 2 network members into CoreDNS configmap... 
INFO[0013] Cluster 'test-cluster' created successfully! 
INFO[0013] You can now use it like this:                
kubectl cluster-info
```
A single node cluster is created.
Then check the nodes:
```Bash
kubectl get nodes
```
```Output
NAME                        STATUS   ROLES                  AGE   VERSION
k3d-test-cluster-server-0   Ready    control-plane,master   59s   v1.25.6+k3s1
```
Check the cluster status
```Bash
kubectl cluster-info
```
```Output
Kubernetes control plane is running at https://0.0.0.0:38483
CoreDNS is running at https://0.0.0.0:38483/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://0.0.0.0:38483/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Create cluster with YAML config
It's also a good idea to use YAML file to create the cluster. This allows us to duplicate the cluster easily and easier for us to recreate the whole environment from disaster. For experimental purposes for Devtron, I will make a 3 nodes cluster with 1 server and 2 agents. The convenient part of K3D is that you can spin up more nodes later quickly. 

The config file is like below: 

```YAML 
# devtron_cluster.yaml
apiVersion: k3d.io/v1alpha4
kind: Simple
metadata:
  name: cluster
servers: 1
agents: 2
ports:
  - port: 8082:30000
    nodeFilters:
      - loadbalancer
  - port: 30080-30100:30080-30100
    nodeFilters:
      - loadbalancer
 ```

 Then use:
 ```Bash
 k3d cluster create --config devtron_cluster.yaml
 ```
to create the cluster. 

Explanations 
