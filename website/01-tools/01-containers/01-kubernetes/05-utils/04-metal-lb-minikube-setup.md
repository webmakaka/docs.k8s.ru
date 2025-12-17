---
layout: page
title: Инсталляция Metal LB
description: Инсталляция Metal LB
keywords: gitops, containers, kubernetes, metal lb
permalink: /tools/containers/kubernetes/utils/metal-lb/kind/setup/
---

# Инсталляция Metal LB

<br/>

### Kind

<br/>

```
// Узнать подсеть
$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kind-worker
172.18.0.3
```

<br/>

```yaml
addresses:
  - 172.18.0.20-172.18.0.30
```

<br/>

Повторить, что в minikube
