---
layout: page
title: Подготовка и установка Helm
description: Подготовка и установка Helm
keywords: devops, containers, kubernetes, linux, helm, setup
permalink: /tools/containers/kubernetes/utils/helm/setup/
---

# Подготовка и установка Helm

<br/>

**Делаю:**  
2026.01.06

<br/>

```
$ curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash


// В последний раз потребовалось выполнить
// $ sudo chown $(whoami) /usr/local/bin/helm

$ helm version --short --client
v3.19.4+g7cfb6e4
```

<br/>

```
// LIST
$ helm repo list
```

<br/>

### Команды, которые приходилось выполнять

```
$ helm pull --insecure-skip-tls-verify oci://registry/release/service-helmchart --version 24020104 --untar
```

<br/>

```
$ helm template . -f servicename-3.yaml
```

<br/>

```
$ helm install -n myNS servicename . -f servicename-3.yaml
```
