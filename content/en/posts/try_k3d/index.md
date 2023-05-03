---
author: "Haoxian WANG"
title: "[K8S] Setup K3D"
date: 2023-02-18T13:00:06+09:00
description: "Quick notes for my K3D cluster setup"
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: Haoxian
authorEmoji: 👻
tags: 
- K3D
- K8S
- Kubernetes 
- Docker
---

## Context
Recently, I am trying to make full use of my i7-12700 CPU and the 96G RAM that I install for it. The ideal thing for me is to be able to practice MLOps on a k8s cluster. However, it's never easy for bare-metal devices to make a whole cluster with many nodes. Maybe with Docker Desktop it will be easy, but then I may lose the chance to learn. 

I have tested microk8s before. However, there are always some weird things that stop me from just getting the basic control plane. This is probably because it was installed with snap. Anyways, I ran into different solutions. Another solution that I have tested is Minikube. Nice and clean but for a single node. I would like to make it more complicated. K3S is a good one and I have installed it on three low-end laptops with my NAS to make a 1 server and 3 agents cluster. It was funny and I succeeded to run some applications on it. Now that I am on a single machine and I want to simulate a multi-node cluster, the ideal way would be to create several VMs and set up the cluster with them which is tedious. To avoid this, some automation tools like Ansible are one of the best choices.  I am not yet ready to get my hands dirty on VMs while my objective is to learn to make things on K8S. K3D is a good alternative based on K3S. 

K3D is a docker version of K3S. It uses docker to simulate nodes and Docker in Docker to run apps on it. 

## Quick start with K3D 
The installation is as handy as the docker installation script: 
```Shell
wget -q -O - https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash
```

To create a cluster:
```Shell 
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
```Shell
kubectl get nodes
```
```Output
NAME                        STATUS   ROLES                  AGE   VERSION
k3d-test-cluster-server-0   Ready    control-plane,master   59s   v1.25.6+k3s1
```
Check the cluster status
```Shell
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
The important part is load blancer with forwarded ports.  
As we know we need a public IP for our cluster. Cloud Providers allow you to use their "Load Balancer" to get a public IP and use it to access the apps inside. However, for on-premise implementation, we need to use solutions like [MetaLB](https://metallb.universe.tf/). In K3D, we use a container to mimic this by exposing its ports. We will use 8082 for Devtron console and 30080-30100 for our applications. 


A complete configuration file is like below 
```YAML
# k3d configuration file, saved as e.g. /home/me/myk3dcluster.yaml
apiVersion: k3d.io/v1alpha4 # this will change in the future as we make everything more stable
kind: Simple # internally, we also have a Cluster config, which is not yet available externally
metadata:
  name: mycluster # name that you want to give to your cluster (will still be prefixed with `k3d-`)
servers: 1 # same as `--servers 1`
agents: 2 # same as `--agents 2`
kubeAPI: # same as `--api-port myhost.my.domain:6445` (where the name would resolve to 127.0.0.1)
  host: "myhost.my.domain" # important for the `server` setting in the kubeconfig
  hostIP: "127.0.0.1" # where the Kubernetes API will be listening on
  hostPort: "6445" # where the Kubernetes API listening port will be mapped to on your host system
image: rancher/k3s:v1.20.4-k3s1 # same as `--image rancher/k3s:v1.20.4-k3s1`
network: my-custom-net # same as `--network my-custom-net`
subnet: "172.28.0.0/16" # same as `--subnet 172.28.0.0/16`
token: superSecretToken # same as `--token superSecretToken`
volumes: # repeatable flags are represented as YAML lists
  - volume: /my/host/path:/path/in/node # same as `--volume '/my/host/path:/path/in/node@server:0;agent:*'`
    nodeFilters:
      - server:0
      - agent:*
ports:
  - port: 8080:80 # same as `--port '8080:80@loadbalancer'`
    nodeFilters:
      - loadbalancer
env:
  - envVar: bar=baz # same as `--env 'bar=baz@server:0'`
    nodeFilters:
      - server:0
registries: # define how registries should be created or used
  create: # creates a default registry to be used with the cluster; same as `--registry-create registry.localhost`
    name: registry.localhost
    host: "0.0.0.0"
    hostPort: "5000"
    proxy: # omit this to have a "normal" registry, set this to create a registry proxy (pull-through cache)
      remoteURL: https://registry-1.docker.io # mirror the DockerHub registry
      username: "" # unauthenticated
      password: "" # unauthenticated
    volumes:
      - /some/path:/var/lib/registry # persist registry data locally
  use:
    - k3d-myotherregistry:5000 # some other k3d-managed registry; same as `--registry-use 'k3d-myotherregistry:5000'`
  config: | # define contents of the `registries.yaml` file (or reference a file); same as `--registry-config /path/to/config.yaml`
    mirrors:
      "my.company.registry":
        endpoint:
          - http://my.company.registry:5000
hostAliases: # /etc/hosts style entries to be injected into /etc/hosts in the node containers and in the NodeHosts section in CoreDNS
  - ip: 1.2.3.4
    hostnames: 
      - my.host.local
      - that.other.local
  - ip: 1.1.1.1
    hostnames:
      - cloud.flare.dns
options:
  k3d: # k3d runtime settings
    wait: true # wait for cluster to be usable before returining; same as `--wait` (default: true)
    timeout: "60s" # wait timeout before aborting; same as `--timeout 60s`
    disableLoadbalancer: false # same as `--no-lb`
    disableImageVolume: false # same as `--no-image-volume`
    disableRollback: false # same as `--no-Rollback`
    loadbalancer:
      configOverrides:
        - settings.workerConnections=2048
  k3s: # options passed on to K3s itself
    extraArgs: # additional arguments passed to the `k3s server|agent` command; same as `--k3s-arg`
      - arg: --tls-san=my.host.domain
        nodeFilters:
          - server:*
    nodeLabels:
      - label: foo=bar # same as `--k3s-node-label 'foo=bar@agent:1'` -> this results in a Kubernetes node label
        nodeFilters:
          - agent:1
  kubeconfig:
    updateDefaultKubeconfig: true # add new cluster to your default Kubeconfig; same as `--kubeconfig-update-default` (default: true)
    switchCurrentContext: true # also set current-context to the new cluster's context; same as `--kubeconfig-switch-context` (default: true)
  runtime: # runtime (docker) specific options
    gpuRequest: all # same as `--gpus all`
    labels:
      - label: bar=baz # same as `--runtime-label 'bar=baz@agent:1'` -> this results in a runtime (docker) container label
        nodeFilters:
          - agent:1
```

Now you have a running K3D cluster on your machine. 
In the next post, I will show you how to install devtron for CI/CD and so on. 

## References
https://k3d.io/v5.4.7/ 


## Notes 
I finally moved to K3S because I got more machines and I wanted to have a real cluster.