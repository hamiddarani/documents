[TOC]

# Kubernetes

## What is kubernetes?

Kubernetes is a COE (**Container Orchestration Engine**)

The task of the **COE** is to manage multiple **Container Engine**

automationg deployment, scaling and management of containerized application.

## Kubernetes architecture

![](kubernetes-cluster-architecture.svg)

kube-api-server:

kube-scheduler:

kube-cotroller-manager:

etcd:

kubelet:

kube-proxy:

container-runtime: containerd, crio, docker

#### kubernetes Glossary

Label: label are key-value pairs that are used to group sets of objects.

Label Selector: it is used for selecting object based on one or more labels.

Annotations: they are key-value pairs and let you associate arbitrary metadata with objects. operator and third-party tools used them to expand kubernetes feature.

Names: Each object in your cluster has a name that is unique for that type of resource. the name of resource  should follow the DNS sub-domain (RFC 1123, 253 character) or dns-domain (RFC 1123, 63 character) (lowercase alphanumeric , "-" , ".")

UID: Every kubernetes object also has a UID that is unique across the whole cluster.

#### Kubernetes Resources

- Workload(Pod, Job, CoronJob, Deployment, ReplicaSet, DaemonSet, ...)
- Service Discover and Load Balancing(Service, Endpoints, Ingress, ...)
- Config and Storage(ConfigMap, Secret, PV, PVC, StorageClass, ...)
- Cluster(Node, Namespace, ResourceQuota, ServiceAccount, Role, ...)
- Metadata(CRD, Event, LimitRange, HPA, VPA, PDB, PSP, ...)

#### How to write kubernetes manifest

kubernetes manifests are specifications of the kubernetes objects written in **yaml** or **json** format.

each manifest includes the following sections:

- apiVersion (mandatory) **>>>>>** v1, apps/v1, batch/v1 (kubectl api-resources)
- kind (mandatory) **>>>>>** Pod, Deployment, DaemonSet, Job, ... (kubectl api-resources)
- metadata (mandatory) **>>>>>** ObjectMeta (name(required), labels, annotations, uid, namespace)
- spec (in some resources) **>>>>>** ResourceSpec
- status (in some resources) **>>>>>** ResourceStatus

you can create objects with these commands

```bash
kubectl create -f manifest.yaml # generate name
kubectl apply -f manifest.yaml
```

## Installation

- install kubernetes with k3s (for practicing) https://k3s.io/
- install kubectl
- copy `/etc/rancher/k3s/k3s.ym` to `~/.kube/config` or export `KUBECONFIG=path`
- kubectl completion bash > `/etc/bash_completion.d/kubectl`

```bash
kubectl get node -o wide
```

## Namespace

namespace is an abstract space used to run workloads in n almost isolated environment. it's called a virtual 

cluster , and you can create as much as you want. kubernetes creates the following namespaces by default

-  defualt
- kube-system
- kube-node-lease
- kube-public

```bash
# execute to create a new namespace
kubectl create namespace test
```

## POD

pod is an **atomic** unit of work in kubernetes which consists of **one or more containers** running on kubernetes cluster. kubernetes scheduler guarantees that all **containers of same pod** run on the **same worker node**. **Network Infrastructure**, **Storage Resources** and the **Lifecycle** of pod's containers are shared together. 

looking deep at kubernetes pods:

- The **one-container-per-pod** pattern is the most common kubernetes use case.
- Kubernetes manages **pods** rather than managing the **containers** directly.
- Pods are **ephemeral** resources, and they do not have any **self-healing** capibilities.
- We use kubernetes **controller** instead of **pods**.
- Controllers are responsible for creating and managing pods, making sure pods are **high-available**.
- In failure situations or when pod get stopped, all data of pod are **removed**. we need to use **persistent storage** to persist data. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app.kubernetes.io/name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:alpine
      command: ["false"] # CrashLoopBackoff
```

```bash
kubectl get pod/po <pod-name> -n <namespace>
kubectl describe pod/po <pod-name> -n <namespace>
```

> [!NOTE]
>
> You cannot modify anything related to the container.
>
> kubectl delete pod test
>
> kubectl apply -f pod.yaml

## Controllers

### ReplicationController(rc)

control the count of pods

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app.kubernetes.io/name: nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
```

### ReplicaSet(rs)

control the count of pods

equality-base

```yaml
apiVersion: v1 
kind: ReplicaSet
metadata:
  name: nginx
spec:
  replica: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: nginx
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
```

set-base

```yaml
apiVersion: v1 
kind: ReplicaSet
metadata:
  name: nginx
spec:
  replica: 3
  selector:
    matchExperssions:
      - key: app.kubernetes.io/name 
        operator: IN
        values:
          - nginx
          - apache
  template:
    metadata:
      labels:
        app.kubernetes.io/name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
```

