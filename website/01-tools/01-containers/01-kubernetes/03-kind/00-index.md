---
layout: page
title: Kind в linux
description: Kind в linux
keywords: gitops, containers, kubernetes, kind
permalink: /tools/containers/kubernetes/kind/
---

# Kind в linux

<br/>

**Делаю:**  
2025.12.30

<br/>

https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries

<br/>

```
// Если ошибка
// curl: (35) error:0A00010B:SSL routines::wrong version number
// Качай по http, а не по https
$ [ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.30.0/kind-linux-amd64
$ chmod +x ./kind
$ sudo mv ./kind /usr/local/bin/kind
```

<br/>

```
$ kind --version
kind version 0.30.0
```

<br/>

### Download Cluster Configurations and Create a 3 Node Kubernetes Cluster

<br/>

Конфиг взят из курса индуса по DevOps. Или по ArgoCD или Ultimate DevSecOps Bootcamp by School of Devops.

<br/>

```yaml
$ kind create cluster --config <(cat << 'EOF'
kind: Cluster
# apiVersion: kind.k8s.io/v1
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 32000
    hostPort: 32000
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 32100
    hostPort: 32100
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30000
    hostPort: 30000
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30055
    hostPort: 30055
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30056
    hostPort: 30056
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30100
    hostPort: 30100
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30200
    hostPort: 30200
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30300
    hostPort: 30300
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30400
    hostPort: 30400
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30500
    hostPort: 30500
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30600
    hostPort: 30600
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30700
    hostPort: 30700
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 30800
    hostPort: 30800
    listenAddress: "0.0.0.0"
    protocol: tcp
- role: worker
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 8000
    hostPort: 8000
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 8080
    hostPort: 8001
    listenAddress: "0.0.0.0"
    protocol: tcp

- role: worker

EOF
)
```

<br/>

### Validate

```
$ kind get clusters
$ kubectl cluster-info --context kind-kind
```

<br/>

```
$ kubectl get nodes
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   36s   v1.34.0
kind-worker          Ready    <none>          26s   v1.34.0
kind-worker2         Ready    <none>          25s   v1.34.0
```

<br/>

### [Дополнительно] Визуализировать в UI контейнеры

```
$ cd ~/projects/courses/kubernetes/
$ git clone https://github.com/schoolofdevops/kube-ops-view
$ kubectl apply -f kube-ops-view/deploy/
```

<br/>

```
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
kube-ops-view-6ffb44dd6c-7qljz   1/1     Running   0          31s
```

<br/>

```
// OK!
http://localhost:32000/
```

<br/>

### Restarting and Resetting the Cluster

```
// $ docker stop kind-control-plane kind-worker kind-worker2
// $ docker start kind-control-plane kind-worker kind-worker2
```

<br/>

```
$ kind get clusters
kind
```

<br/>

```
// Delete cluster
// $ kind delete cluster --name kind
```
