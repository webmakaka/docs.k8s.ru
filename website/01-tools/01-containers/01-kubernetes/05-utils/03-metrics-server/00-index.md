---
layout: page
title: Metrics Server
description: Metrics Server
keywords: linux, kubernetes, Metrics Server
permalink: /tools/containers/kubernetes/utils/metrics-server/
---

# Metrics Server

<br/>

Делаю:  
2026.01.23

<br/>

https://kubernetes-tutorial.schoolofdevops.com/argo_experiments_analysis/

<br/>

### Устанавливаем metrics-server

```
$ cd ~/tmp
$ git clone https://github.com/schoolofdevops/metrics-server.git
$ kubectl apply -k metrics-server/manifests/overlays/release
```

<br/>

```
$ kubectl top pod
$ kubectl top node
```
