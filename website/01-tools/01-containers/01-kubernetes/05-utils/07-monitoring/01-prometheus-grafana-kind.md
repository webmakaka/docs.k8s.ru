---
layout: page
title: Запуск Prometheus (мониторинг) и Grafana (визуализация) в kuberntes cluster с помощью heml
description: Запуск Prometheus (мониторинг) и Grafana (визуализация) в kuberntes cluster с помощью heml
keywords: tools, containers, kubernetes, monitoring, prometheus, grafana, setup, helm
permalink: /tools/containers/kubernetes/utils/monitoring/prometheus-grafana/setup/helm/kind/
---

# Инсталляция с помощью Helm инструментов мониторинга

<br/>

### Запуск Prometheus (мониторинг) и Grafana (визуализация) в kuberntes cluster с помощью heml

<br/>

**Делаю:**  
2025.12.30

<br/>

### [Используется kind](/tools/containers/kubernetes/kind/)

<br/>

```
https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack


https://github.com/prometheus-operator/kube-prometheus/

https://github.com/prometheus-community/helm-charts
```

<br/>

```
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

$ helm repo update
```

<br/>

```
$ helm upgrade \
    --install prom prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace \
    --set grafana.service.type=NodePort \
    --set grafana.service.nodePort=30200 \
    --set prometheus.service.type=NodePort \
    --set prometheus.service.nodePort=30300
```

<br/>

```
$ helm list -A
NAME	NAMESPACE 	REVISION	UPDATED                               	STATUS  	CHART                       	APP VERSION
prom	monitoring	1       	2025-12-30 03:18:54.89711352 +0300 MSK	deployed	kube-prometheus-stack-80.8.1	v0.87.1
```

<br/>

```
$ kubectl get pods -n monitoring
NAME                                                     READY   STATUS    RESTARTS   AGE
alertmanager-prom-kube-prometheus-stack-alertmanager-0   2/2     Running   0          64s
prom-grafana-8665d5db86-vvk4b                            3/3     Running   0          77s
prom-kube-prometheus-stack-operator-58dcbcd4cf-m8sds     1/1     Running   0          77s
prom-kube-state-metrics-54b9b47967-5k8hq                 1/1     Running   0          77s
prom-prometheus-node-exporter-287kb                      1/1     Running   0          77s
prom-prometheus-node-exporter-54z4w                      1/1     Running   0          77s
prom-prometheus-node-exporter-cjph4                      1/1     Running   0          77s
prometheus-prom-kube-prometheus-stack-prometheus-0       2/2     Running   0          64s
```

<br/>

```
You should be able to access:


// Prometheus
http://localhost:30300/
```

<br/>

```
// Пароль для grafana
$ kubectl get secret -n monitoring prom-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
```

<br/>

```
// Grafana
// admin /
http://localhost:30200/
```
