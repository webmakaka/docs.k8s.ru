---
layout: page
title: Инсталляция Metal LB
description: Инсталляция Metal LB
keywords: gitops, containers, kubernetes, metal lb
permalink: /tools/containers/kubernetes/utils/metal-lb/minikube/setup/
---

# Инсталляция Metal LB

<br/>

**Делаю:**  
2025.12.17

<br/>

### Вариант 2: Не заработал (не стал на хосте менять routes)

<br/>

Если kubeproxy запущен без strictARP: true.
Исправим это, отредактировав соответствующий configmap:

<br/>

```
$ kubectl edit configmap -n kube-system kube-proxy
```

<br/>

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: 'ipvs'
ipvs:
  strictARP: true
```

<br/>

### Установка Metal LB

```
$ LATEST_VERSION=$(curl --silent "https://api.github.com/repos/metallb/metallb/releases/latest" | grep '"tag_name"' | sed -E 's/.*"([^"]+)".*/\1/')

// v0.15.3
$ echo ${LATEST_VERSION}
```

<br/>

```

// Установка metal-lb
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/${LATEST_VERSION}/config/manifests/metallb-native.yaml
```

<br/>

### Minikube

```
$ minikube --profile ${PROFILE} ip
192.168.49.2
```

<br/>

Задаем диапазон ip адресов, которые можно выдать виртуальному сервису. Нужно, чтобы он был в той же подсети, что и ip minikube.

<br/>

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.49.20-192.168.49.30
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: base
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF
```

<br/>

### Проверка из minikube

```
$ minikube ssh -p ${PROFILE}
```

<br/>

```
// OK
// MetalLB работает правильно внутри minikube!
$ docker@marley-minikube:~$ curl http://192.168.49.21:8080
```

<br/>

С хост машины не заработал.
Нужно route прописать в таблице маршрутизации.

<br/>

```
// Не проверил
$ sudo ip route add 192.168.49.0/24 via $(minikube ip -p ${PROFILE})
```
