# Highly Available Pod Distribution in Kubernetes

* [Project Overview](#project-overview)
* [Prerequisite](#prerequisite)
* [Local Setup](#local-setup)
    * [Create a KIND Cluster on Local](#create-a-kind-cluster-on-local)
    * [Install CLIs](#install-clis)
* [Deploy Resources](#deploy-resources)
    * [Deploy Nginx](#deploy-nginx)
* [Test](#test)
* [Conclusion](#conclusion)
* [Resources](#-resources)

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

## Deploy Resources

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

### Deploy Nginx

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
---

## Conclusion

This deployment ensures a secure, highly available, and scalable application setup while balancing pods across zones and nodes.

---

## 📚 Resources

☸️ **Kubernetes Core Concepts**

* [Labels, Annotations, and Taints](https://kubernetes.io/docs/reference/labels-annotations-taints/)
* [Scheduling Configuration](https://kubernetes.io/docs/reference/scheduling/config/)
* [Multiple Zones Best Practices](https://kubernetes.io/docs/setup/best-practices/multiple-zones/)
* [Topology Spread Constraints](https://kubernetes.io/docs/concepts/scheduling-eviction/topology-spread-constraints/)
* [Pod Disruption Budget (PDB)](https://medium.com/@muppedaanvesh/a-hand-on-guide-to-kubernetes-pod-disruption-budget-pdb-%EF%B8%8F-ebe3155a4b7c)
* [Volumes and Storage](https://kubernetes.io/docs/concepts/storage/volumes/)

⚙️ **Cluster Configuration and Setup**

* [Name Your KIND Cluster](https://kind.sigs.k8s.io/docs/user/configuration/#name-your-cluster)
* [Even Pod Distribution Across Nodes](https://cloudhero.io/kubernetes-evenly-distribution-of-pods-across-cluster-nodes/)
* [Setting Up Multi-Node Cluster with KIND (Medium)](https://medium.com/@subhampradhan966/setting-up-a-multi-node-kubernetes-cluster-with-kind-a-comprehensive-guide-146ee5994226)
* [Setting Up Cluster with KIND (Dev.to)](https://dev.to/isaackumi/setting-up-a-single-or-multi-node-cluster-on-kindo-6o2)
* [Simplifying Deployments with Kustomize](https://deniz-turkmen.medium.com/simplifying-kubernetes-deployments-with-kustomize-52a2daa6145c)

🚀 **Deployment and Nginx Customization**

* [Change `index.html` in Nginx Deployment (Stack Overflow)](https://stackoverflow.com/questions/49904784/change-index-html-nginx-kubernetes-deployment)
* [Custom HTML on Nginx Pod via ConfigMap (Tim Krassowski)](https://medium.com/@tim.krassowski/kubernetes-deployment-that-uses-a-custom-index-html-file-on-an-nginx-pod-using-a-configmap-1ee22070897e)
* [Deploy Pods with Custom HTML (Janita GW)](https://medium.com/@janita.gw13/how-to-deploy-kubernetes-pods-with-a-custom-html-on-a-nginx-web-server-cafdd9e965f)
* [Unix `sed` Command with Variables](https://unix.stackexchange.com/questions/646851/struggling-using-sed-command-with-variables)
* [YouTube: Kubernetes Deployment Guide](https://www.youtube.com/watch?v=MxGAuVsJBXE)

🛡️ **Kyverno Policies and Testing**

* [Kyverno Policy: Require Deployments Have Multiple Replicas](https://kyverno.io/policies/other/require-deployments-have-multiple-replicas/require-deployments-have-multiple-replicas/)
* [Kyverno CLI Usage and Testing](https://kyverno.io/docs/kyverno-cli/usage/test/)
* [Kyverno GitHub Issue #8589](https://github.com/kyverno/kyverno/issues/8589)
* [Security Considerations for Kyverno](https://security.theodo.com/en/blog/security-kyverno-kubernetes)

🧩 **Webhooks and Admission Controllers**

* [Kubernetes Admission Controllers](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
* [What Is a Mutating Webhook (Medium)](https://mycloudjourney.medium.com/what-is-mutatingwebhook-in-kubernetes-a62f79598ecb)
* [SlackHQ: Simple Kubernetes Webhook (GitHub)](https://github.com/slackhq/simple-kubernetes-webhook/tree/main)
* [Slack Engineering: Webhook Deep Dive](https://slack.engineering/simple-kubernetes-webhook/)
