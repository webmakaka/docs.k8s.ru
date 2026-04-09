---
layout: page
title: Запуск и останов minikube в ubuntu 22.04
description: Запуск и останов minikube в ubuntu 22.04
keywords: ubuntu, containers, kubernetes, minikube, run
permalink: /tools/containers/kubernetes/minikube/run/
---

# Запуск и останов minikube в ubuntu 22.04

<br/>

**Делаю:**  
2026.04.06

<br/>

Можно использовать VirtualBox или Docker.
Для всех случаев, когда нужно работать не с каким-то выделенным сервером на виртуалке с minikube, стоит использовать docker.

<br/>

### Запуск

<br/>

**driver может быть из популярных:**

- docker
- kvm2
- virtualbox
- д.р.

<br/>

```shell
// Список последних версий
$ minikube config defaults kubernetes-version | head -n 4
* v1.35.1
* v1.35.0
* v1.35.0-rc.1
* v1.35.0-rc.0
```

<!-- <br/>

```
// v1.35.0
$ LATEST_KUBERNETES_VERSION=$(curl -s https://api.github.com/repos/kubernetes/kubernetes/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
``` -->

<!-- <br/>

```
$ echo ${LATEST_KUBERNETES_VERSION}
v1.35.0
``` -->

<!-- <br/>

```
$ LATEST_KUBERNETES_VERSION=v1.35.0
``` -->

<br/>

```shell
$ export \
    PROFILE=${USER}-minikube \
    CPUS=no-limit \
    MEMORY=8G \
    HDD=20G \
    DRIVER=docker \
    KUBERNETES_VERSION=v1.35.1
```

<br/>

```shell
$ {
    minikube --profile ${PROFILE} config set memory ${MEMORY}
    minikube --profile ${PROFILE} config set cpus ${CPUS}
    minikube --profile ${PROFILE} config set disk-size ${HDD}

    minikube --profile ${PROFILE} config set driver ${DRIVER}

    minikube --profile ${PROFILE} config set kubernetes-version ${KUBERNETES_VERSION}
    minikube start --profile ${PROFILE} --embed-certs

    // Enable ingress
    minikube addons --profile ${PROFILE} enable ingress

    // Enable registry
    // minikube addons --profile ${PROFILE} enable registry

    // Enable metallb
    // minikube addons --profile ${PROFILE} enable metallb

    // Enable nvidia-device-plugin
    // minikube addons --profile ${PROFILE} enable nvidia-device-plugin
}
```

<br/>

```shell
// При необходимости можно будет удалить профиль и все созданное в профиле следующей командой
// $ minikube --profile ${PROFILE} stop && minikube --profile ${PROFILE} delete

// Стартовать остановленный minikube
// $ minikube --profile ${PROFILE} start
```

<br/>

```shell
$ kubectl get nodes
NAME              STATUS   ROLES           AGE     VERSION
marley-minikube   Ready    control-plane   3h52m   v1.35.0
```

<br/>

```shell
$ kubectl version
Client Version: v1.31.0
Kustomize Version: v5.4.2
Server Version: v1.32.2
```

<br/>

```shell
// Получить список установленных расширений
// $ minikube addons --profile ${PROFILE} list
```

<br/>

Далее нужно установить командную утилиту для работы с кластером - [kubectl](/tools/containers/kubernetes/utils/kubectl/)

<br/>

```shell
// Получить текущий контекст
// $ kubectl config current-context
```

<br/>

### Сделать, чтобы Docker images хранились в выделенном ранее storage внутри контейнера, а не на основном хосте.

```shell
// Посмотреть команды которые установят переменные окружения
// $ minikube docker-env --profile ${PROFILE}

// Команда, которая установит нужные переменные
$ eval $(minikube -p ${PROFILE} docker-env)
```

<br/>

### Подключиться к UI (Не нужно, но можно)

<br/>

```shell
$ minikube addons --profile ${PROFILE} enable dashboard
```

<br/>

```shell
// Подключиться к dashboard можно следующей командой
// $ minikube --profile ${PROFILE} dashboard
```

<br/>

```shell
// Получить токен для авторизации в kubernetes dashboard
// $ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

<br/>

### Дополнительная инфа по развернутому kuberntes кластеру

```shell
$ kubectl cluster-info
Kubernetes control plane is running at https://192.168.49.2:8443
CoreDNS is running at https://192.168.49.2:8443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

<br/>

```shell
$ minikube --profile ${PROFILE} config view
- disk-size: 20G
- driver: docker
- kubernetes-version: v1.32.2
- memory: 8G
- cpus: 4
```

<br/>

```shell
$ kubectl api-resources
```

<br/>

### Подключиться к minikube по ssh

<br/>

```shell
$ minikube --profile ${PROFILE} ssh
```

<br/>

Или еще вариант

<br/>

```shell
$ minikube --profile ${PROFILE} ip
$ export MINIKUBE_IP=192.168.99.100
$ ssh -i ~/.minikube/machines/${PROFILE}/id_rsa docker@${MINIKUBE_IP}
```

<br/>

### Остальное

```shell
$ kubectl get events
$ kubectl get events --sort-by=.metadata.creationTimestamp
```

<br/>

```shell
// Установить vscode как editor по умолчанию
$ export KUBE_EDITOR="code -w"
```

<br/>

```shell
// Расположение профайлов
~/.minikube/profiles

// outputs the current profile
$ minikube profile

// lists all existing profiles
$ minikube profile list
```

<br/>

## Дополнительно:

<br/>

### [Добавить "Metal LB" (При необходимости)](/tools/containers/kubernetes/utils/metal-lb/minikube/setup/addon/)

<br/>

### [Registry в Minikube](/tools/containers/kubernetes/minikube/setup/registry/)

<br/>

### [Ngrok Ingress Controller for Kubernetes (Доступ к kubernetes кластеру из интернетов)](/tools/containers/kubernetes/minikube/ngrok-operator/)
