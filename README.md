# Highly Available Pod Distribution in Kubernetes

* [Project Overview](#project-overview)
* [Prerequisite](#prerequisite)
* [Local Setup](#local-setup)
    * [Create a KIND Cluster on Local](#create-a-kind-cluster-on-local)
    * [Install CLIs](#install-clis)
* [Deploy Resources](#deploy-resources)
    * [Deploy Nginx](#deploy-nginx)
* [Test](#test)

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
$ cd k8s-ha-pod-distribution
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

### Deploy Resources

First we may need wait for all Kyverno pods to be fully initialized and in running state.

```shell
$ kubectl get po -n kyverno --watch
NAME                                             READY   STATUS    RESTARTS   AGE
kyverno-admission-controller-768cff6447-xj4p9    1/1     Running   0          37s
kyverno-background-controller-849858b7fb-tzm2g   1/1     Running   0          37s
kyverno-cleanup-controller-56fc6c64c-5rgnc       1/1     Running   0          37s
kyverno-reports-controller-645c564d7-99dqx       1/1     Running   0          37s
```

Then we run the command below to create the Kubernetes resources: Pod Disruption Budget and Kyverno policies.

```shell
$ kubectl create -k ./manifests/
poddisruptionbudget.policy/pdb created
clusterpolicy.kyverno.io/add-node-affinity created
clusterpolicy.kyverno.io/add-pod-affinity created
clusterpolicy.kyverno.io/add-topology-spread created
```

**Determine a policy’s effectiveness**

Prior to committing to a cluster, we can use ```apply``` command dry-run policies.

```shell
$ kyverno apply --resource ./manifests/deploy-nginx.yaml \
./manifests/add-node-affinity.yaml \
./manifests/add-pod-affinity.yaml \
./manifests/add-topology-spread.yaml

.
.
.
---

pass: 3, fail: 0, warn: 0, error: 0, skip: 0
```

#### Deploy Nginx

```shell
$ kubectl apply -f ./manifests/deploy-nginx.yaml
namespace/dev created
configmap/nginx-configmap created
deployment.apps/nginx-deploy created
service/nginx-svc created
```

```shell
$ kubectl get deploy nginx-deploy -n dev --watch 
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deploy   4/6     6            4           58s
nginx-deploy   5/6     6            5           61s
nginx-deploy   6/6     6            6           65s
```

**Verify policy application**

```shell
$ kubectl get deploy nginx-deploy -n dev -oyaml | grep -E "topologySpreadConstraints|Affinity"
        nodeAffinity:
        podAntiAffinity:
          - podAffinityTerm:
          - podAffinityTerm:
      topologySpreadConstraints:
```

This labeling strategy ensures nginx pods are scheduled only on the labeled worker nodes.

```shell
$ kubectl get po -n dev -owide
NAME                            READY   STATUS    RESTARTS   AGE   IP           NODE
nginx-deploy-5c4bbb658c-4xcjt   1/1     Running   0          20m   10.244.3.8   localstack-worker2
nginx-deploy-5c4bbb658c-66lgj   1/1     Running   0          20m   10.244.1.7   localstack-worker3
nginx-deploy-5c4bbb658c-6jj8h   1/1     Running   0          20m   10.244.3.7   localstack-worker2
nginx-deploy-5c4bbb658c-9ms2t   1/1     Running   0          20m   10.244.1.8   localstack-worker3
nginx-deploy-5c4bbb658c-rgdq2   1/1     Running   0          20m   10.244.2.8   localstack-worker
nginx-deploy-5c4bbb658c-smdpr   1/1     Running   0          20m   10.244.2.9   localstack-worker
```

## Test

**Check desired results**

You can test a given set of resources against one or more policies by running ```test``` command.

```shell
$ kyverno test tests

Loading test  ( tests/kyverno-test.yaml ) ...
  Loading values/variables ...
  Loading policies ...
  Loading resources ...
  Loading exceptions ...
  Applying 3 policies to 1 resource ...
  Checking results ...

│────│─────────────────────│────────────────────────────────────│────────────────────────│────────│────────│
│ ID │ POLICY              │ RULE                               │ RESOURCE               │ RESULT │ REASON │
│────│─────────────────────│────────────────────────────────────│────────────────────────│────────│────────│
│ 1  │ add-node-affinity   │ add-node-affinity-to-deployments   │ Deployment/test-deploy │ Pass   │ Ok     │
│ 2  │ add-pod-affinity    │ add-pod-affinity-to-deployments    │ Deployment/test-deploy │ Pass   │ Ok     │
│ 3  │ add-topology-spread │ add-topology-spread-to-deployments │ Deployment/test-deploy │ Pass   │ Ok     │
│────│─────────────────────│────────────────────────────────────│────────────────────────│────────│────────│


Test Summary: 3 tests passed and 0 tests failed
```

**Test load distribution**

You may notice that the pod name in the response changes over time when making repeated curl requests. 
This indicates that responses are being served by different pods.

```shell
$ curl localhost:30000
<!DOCTYPE html>
<html>
<body>
  <h1>Welcome to Pod!</h1>
  <h1>This web page is housed on a Pod running Nginx</h1>
  <p>This response is from pod:nginx-deploy-5c4bbb658c-smdpr</p>
</body>
</html>
```
