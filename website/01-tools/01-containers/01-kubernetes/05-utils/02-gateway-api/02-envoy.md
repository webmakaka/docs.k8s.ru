---
layout: page
title: Инсталляция kubectl в ubuntu 22.04
description: Инсталляция kubectl в ubuntu 22.04
keywords: tools, containers, kubernetes, Gateway API, Envoy
permalink: /tools/containers/kubernetes/utils/gateway-api/envoy/
---

# Envoy Gateway API for Kubernetes

Делаю:  
2026.04.10

<br/>

https://www.youtube.com/watch?v=me_5W_Q4ZWg

<br/>

[Выполнил шаги по установке Gateway API](/tools/containers/kubernetes/utils/gateway-api/)

<br/>

**Далее:**  
https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/kubernetes/gateway-api/envoy

<br/>

## Envoy: Gateway API controller

<br/>

### Installation

```shell
$ CHART_VERSION="v1.6.0"
$ helm show chart oci://docker.io/envoyproxy/gateway-helm
$ helm show values oci://docker.io/envoyproxy/gateway-helm
```

<br/>

```yaml
$ cat > envoy-values.yaml << 'EOF'
config:
# -- EnvoyGateway configuration. Visit https://gateway.envoyproxy.io/docs/api/extension_types/#envoygateway to view all options.
  envoyGateway:
    gateway:
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
    provider:
      type: Kubernetes
    logging:
      level:
        default: info
EOF
```

<br/>

```shell
$ helm upgrade \
  --install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --namespace envoy-gateway-system \
  --create-namespace \
  --version ${CHART_VERSION} \
  --values envoy-values.yaml
```

<br/>

```shell
$ kubectl -n envoy-gateway-system get pods
NAME                             READY   STATUS    RESTARTS   AGE
envoy-gateway-65d4675ff8-7kd4d   1/1     Running   0          2m28s
```

<br/>

```shell
$ kubectl -n envoy-gateway-system logs -l app.kubernetes.io/instance=envoy-gateway
```

<br/>

### Install an Envoy Gateway Class

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
EOF
```

<br/>

```shell
$ kubectl get gatewayclass
NAME    CONTROLLER                                      ACCEPTED   AGE
envoy   gateway.envoyproxy.io/gatewayclass-controller   True       7m6s
```

<br/>

### Install an Envoy Gateway

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-api
  namespace: default
spec:
  gatewayClassName: envoy
  infrastructure:
    labels:
      app: envoy-gateway
  # Only Routes from the same namespace are allowed.
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: Same  #or All or Selector
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: secret-tls
            namespace: default
      allowedRoutes:
        namespaces:
          from: Same
EOF
```

<br/>

```shell
$ kubectl get gateway
NAME          CLASS   ADDRESS   PROGRAMMED   AGE
gateway-api   envoy             False        6m17s
```

<br/>

```shell
// check the new gateway-api pod
$ kubectl -n envoy-gateway-system get pods
NAME                                                  READY   STATUS    RESTARTS   AGE
envoy-default-gateway-api-30a1473e-76758f4bf4-v9rhm   2/2     Running   0          19s
envoy-gateway-65d4675ff8-7kd4d                        1/1     Running   0          6m10s
```

```shell
// we also have a new service
$ kubectl -n envoy-gateway-system get svc
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                            AGE
envoy-default-gateway-api-30a1473e   LoadBalancer   10.96.180.28   <pending>     80:31052/TCP,443:31600/TCP                         34s
envoy-gateway                        ClusterIP      10.96.181.82   <none>        18000/TCP,18001/TCP,18002/TCP,19001/TCP,9443/TCP   6m25s
```

<br/>

### Gateway Configuration

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: gateway-configuration
    namespace: envoy-gateway-system
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: gateway-configuration
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyDeployment:
        replicas: 2
      envoyService:
        name: envoy-gateway-default
  telemetry:
    accessLog:
      settings:
      # Define a log sink (destination) and format
      - sinks:
          - type: File
            file:
              path: /dev/stdout
        format:
          type: JSON
          json:
            # Custom Fields (using Envoy command operators)
            start_time: "%START_TIME%"
            method: "%REQ(:METHOD)%"
            path: "%REQ(X-ENVOY-ORIGINAL-PATH?:PATH)%"
            response_code: "%RESPONSE_CODE%"
            upstream_host: "%UPSTREAM_HOST%"
            custom_header: "%REQ(X-CUSTOM-ID)%" # Example: Add a custom request header
EOF
```

<br/>

```shell
$ kubectl -n envoy-gateway-system get deploy
NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
envoy-default-gateway-api-30a1473e   2/2     2            2           16m
envoy-gateway                        1/1     1            1           22m
```

<br/>

```shell
$ kubectl -n envoy-gateway-system get pods
NAME                                                  READY   STATUS    RESTARTS   AGE
envoy-default-gateway-api-30a1473e-76758f4bf4-v9rhm   2/2     Running   0          15m
envoy-default-gateway-api-30a1473e-76758f4bf4-zp4k5   2/2     Running   0          2m17s
envoy-gateway-65d4675ff8-7kd4d                        1/1     Running   0          21m
```

<br/>

```shell
$ kubectl -n envoy-gateway-system get svc
NAME                    TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                                            AGE
envoy-gateway           ClusterIP      10.96.181.82   <none>        18000/TCP,18001/TCP,18002/TCP,19001/TCP,9443/TCP   22m
envoy-gateway-default   LoadBalancer   10.96.145.80   <pending>     80:32688/TCP,443:32459/TCP                         3m49s
```

<br/>

```shell
$ kubectl -n envoy-gateway-system port-forward svc/envoy-gateway-default 8080:80
```

<br/>

### HTTP Traffic management

<br/>

[checks](/tools/containers/kubernetes/utils/gateway-api/checks/)

<br/>

### Gateway API Extensions

Видео ~ 30 минута
