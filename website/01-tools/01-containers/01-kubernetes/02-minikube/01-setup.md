---
layout: page
title: Инсталляция и подготовка minikube для работы в ubuntu 22.04
description: Инсталляция и подготовка minikube для работы в ubuntu 22.04
keywords: ubuntu, containers, kubernetes, minikube, setup
permalink: /tools/containers/kubernetes/minikube/setup/
---

# Инсталляция и подготовка minikube для работы в ubuntu 22.04

<br/>

## Инсталляция minikube в ubuntu 22.04

<br/>

**Делаю:**  
2026.01.31

<br/>

**minikube** - подготовленная виртуальная машина или контейнер с мини kubernetes сервером. Вполне подойдет для изучения kubernetes, особенно на слабых компьютерах и ноутбуках.

<br/>

```shell
// Узнать последнюю версию (v1.37.0):
$ curl -s https://api.github.com/repos/kubernetes/minikube/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/'
```

<br/>

```shell
// Установка
$ curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

<br/>

```
$ minikube version
minikube version: v1.38.0
commit: de81223c61ab1bd97dcfcfa6d9d5c59e5da4a0cf
```

<br/>

[Запуск и останов minikube в ubuntu 22.04](/tools/containers/kubernetes/minikube/run/)

<br/>

```
// Запросить инфу для обновления
$ minikube update-check
CurrentVersion: v1.33.0
LatestVersion: v1.38.0
```
