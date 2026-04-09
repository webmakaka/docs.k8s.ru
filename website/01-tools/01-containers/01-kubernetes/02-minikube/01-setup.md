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
2026.04.01

<br/>

**minikube** - подготовленная виртуальная машина или контейнер с мини kubernetes сервером. Вполне подойдет для изучения kubernetes, особенно на слабых компьютерах и ноутбуках.

<br/>

```shell
// Узнать последнюю версию (v1.38.1):
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
minikube version: v1.38.1
commit: c93a4cb9311efc66b90d33ea03f75f2c4120e9b0
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
