# Highly Available Pod Distribution in Kubernetes

1.  [Project Overview](#project-overview)
2.  [Prerequisite](#prerequisite)
3.  [Local Setup](#local-setup)
      * [Create a KIND Cluster on Local](#create-a-kind-cluster-on-local)
      * [Install CLIs](#install-clis)

## Project Overview

High availability and balanced workload distribution in Kubernetes through topology spread constraints, affinity, and anti-affinity rules.

KIND cluster running on local simulates optimal pod distribution across AWS availability zones with topology spread constraints.

---

## Prerequisite

- [Docker](https://docs.docker.com/engine/install/)
- [Helm](https://helm.sh/docs/intro/install/)
- [Kyverno](https://kyverno.io/docs/installation/methods/)
- [Kyverno CLI](https://kyverno.io/docs/kyverno-cli/install/)
- [KIND](https://kind.sigs.k8s.io/docs/user/quick-start#installation)

## Local Setup

### Create a KIND Cluster on Local

```shell
$ cd rossum-assignment
$ kind create cluster --config ./config/kind-config.yaml
Creating cluster "localstack" ...
 ✓ Ensuring node image (kindest/node:v1.31.0) 🖼
 ✓ Preparing nodes 📦 📦 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-localstack"
You can now use your cluster with:

kubectl cluster-info --context kind-localstack

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

**Verify the cluster's connectivity**

```shell
$ kubectl cluster-info --context kind-localstack
Kubernetes control plane is running at https://127.0.0.1:42203
CoreDNS is running at https://127.0.0.1:42203/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

**View cluster node status**

The cluster should have 1 control-plane and 3 worker nodes, and they are labeled.

```shell
$ kubectl get nodes
NAME                       STATUS   ROLES           AGE     VERSION
localstack-control-plane   Ready    control-plane   4m29s   v1.31.0
localstack-worker          Ready    <none>          4m19s   v1.31.0
localstack-worker2         Ready    <none>          4m19s   v1.31.0
localstack-worker3         Ready    <none>          4m19s   v1.31.0
```

### Install CLIs

**HELM**
```shell
$ curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
$ chmod 700 get_helm.sh
$ ./get_helm.sh
```

**Kyverno**
```shell
$ helm repo add kyverno https://kyverno.github.io/kyverno/
$ helm repo update
$ helm install kyverno kyverno/kyverno -n kyverno --create-namespace 
```

**Kyverno CLI**
```shell
$ curl -LO https://github.com/kyverno/kyverno/releases/download/v1.12.0/kyverno-cli_v1.12.0_linux_x86_64.tar.gz
$ tar -xvf kyverno-cli_v1.12.0_linux_x86_64.tar.gz
$ sudo cp kyverno /usr/local/bin/
```
