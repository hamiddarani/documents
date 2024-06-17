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
kubectl exec <pod-name> -- <command>
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

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  revisionHistoryLimit: 10
  strategy:
    type: RollingUpdate or Recreate
    maxSurge: 25% or number
    maxUnavailable: 25% or number
  replicas: 3
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
            image: nginx:1.18
```

type of deployment:

- Recreate (Remove old resource and recreate)
- RollingUpdate 
- Blue/Green (Blue is old version / Green is new version)
- Red/Black
- Canary
- A/B (canary with specified user)

```bash
kubectl rollout history deployment nginx --revision 1
kubectl rollout undo deployment nginx --to-revivion 1
kubectl rollout status deployment nginx
kubectl rollout pause deployment nginx
kubectl rollout resume deployment nginx
kubectl rollout restart deployment nginx
```

### DaemonSet(ds)

It runs a container on each node.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
spec:
  updateStrategy:
    type: OnDelete or RollingUpdate
  selector:
    matchLabels:
      app.kubernetes.io/name: node-exporter
  template:
    metadata:
      labels:
        app.kubernetes.io/name: node-exporter
    spec:
      containers:
        - name: node-exporter
          image: prom/node-exporter:latest
```

### StatefulSet

StatefulSet is used to mange **stateful** application.

StatefulSet provides guarantees about **ordering** and **uniqueness** of pods.

StatefulSet maintains a sticky identity for each of their pods. each pod has a unique name.

Each of statefulset pods get the following labels:

`statefulset.kubernetes.io/pod-name`

StatefulSet pods are created sequentially in order `from 0 to N-1`

StatefulSet pods are terminated in reverse order from `N-1 to 0`

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis
spec:
  replica: 3
  updateStrategy:
    type: OnDelete
  selector:
    matchLabels:
      app.kubernetes.io/name: redis
  serviceName: redis-headless
  template:
    metadata:
      labels:
        app.kubernetes.io/name: redis
    spec:
      containers:
        - name: redis
          image: redis:7
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplate:
    - metadata:
        name: data
      spec:
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10G
```

Pod Management Policy:

- Ordered Ready (mysql, postgres, redis,...)
- Parallel (etcd, mongo shard, ...)

Two update strategies:

- RollingUpdate
- OnDelete

## Service (svc)

- How to communicate with pods when their IP addresses are ephemeral?
- How to access pods from outside of the kubernetes cluster?
- How to access to all replicas of application and use a load-balancing mechanism? 
- How to access and discover pods with DNS names?
- How can i publish application ports and access them from the local network?
- How to connect pods to out-of-cluster backing services? 

Type of services:

- ClusterIP
- NodePort
- LoadBalancer
- ExternalName

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <app-name>
spec:
  selector:
    app.kubernetes.io/name: <app-name>
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: 8080
      nodeProt: 30080 # if type of service is NodePort
```

```bash
kubectl expose -n <namespace> deployment/<deployment-name> --port 80 --type ClusterIP
```

Every connection to the pod from the outside to the inside passes through kube-proxy

network plugin (calico/flannel): All pods are in the same network. 

```bash
curl <service-name>.<namespace>.svc.<cluster-name>
curl nginx.default.svc.cluster.local

# notice
/etc/resolve.conf
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5

iptables -t nat -nL
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: google
spec:
  type: ExternalName
  externalName: google.com
```

