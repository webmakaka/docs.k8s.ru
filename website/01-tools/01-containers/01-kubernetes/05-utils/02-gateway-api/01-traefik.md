---
layout: page
title: Traefik Gateway API for Kubernetes
description: Traefik Gateway API for Kubernetes
keywords: tools, containers, kubernetes, Gateway API, Traefik
permalink: /tools/containers/kubernetes/utils/gateway-api/traefik/
---

# Traefik Gateway API for Kubernetes

Делаю:  
2026.02.06

<br/>

https://www.youtube.com/watch?v=MN-k29g97ik

<br/>

[Выполнил шаги по установке Gateway API](/tools/containers/kubernetes/utils/gateway-api/)

<br/>

**Далее:**  
https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/kubernetes/gateway-api/traefik

<br/>

## Install a Gateway API controller

To use the Gateway API features in Kubernetes, you need a controller that implements the above CRDs. </br>
In this introduction guide I will use an existing Gateway API controller called Traefik. </br>

<i><b>Note:</b> Keep in mind that this introduction has no dependency on Traefik specifically, therefore any controller that supports Gateway API can be used.
At the bottom of this guide, I will provide guides on each of the popular Gateway API implementations.
</i>

A Basic Gateway API controller install:

<br/>

```shell
$ CHART_VERSION="39.0.0" # traefik version v3.6.7
$ helm repo add traefik https://helm.traefik.io/traefik
$ helm repo update
$ helm search repo traefik --versions
```

<br/>

```
$ cd ~/tmp
```

<br/>

```yaml
$ cat > traefik-values.yaml << EOF
logs:
  access:
    enabled: true
ports:
  web:
    port: 80
  websecure:
    port: 443
    exposedPort: 443

# we enable gateway-api features
providers:
  kubernetesGateway:
    enabled: true
    experimentalChannel: false

# we disable gateway-api defaults because we want to provide our own
gatewayClass:
  enabled: false

# we disable gateway-api defaults because we want to provide our own
gateway:
  enabled: false
EOF
```

<br/>

```shell
$ helm upgrade \
  --install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --version $CHART_VERSION \
  --values traefik-values.yaml
```

As we don't have a LoadBalancer service in `kind`, let's `port-forward` so we can pretend we have one

```shell

# check the pods
$ kubectl -n traefik get pods

# check the logs
$ kubectl -n traefik logs -l app.kubernetes.io/instance=traefik-traefik
```

<br/>

## Install a Gateway Class

To start enabling traffic to our newly created apps, we will start with installing a Gateway Class. </br>

Note that we use a Traefik Class in our example. </br>

[Documentation](https://gateway-api.sigs.k8s.io/api-types/gatewayclass/)

`GatewayClass` is a cluster-scoped resource defined by the infrastructure provider. This resource represents a class of Gateways that can be instantiated. </br>

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: traefik
spec:
  controllerName: traefik.io/gateway-controller
EOF
```

<br/>

```shell
# check
$ kubectl get gatewayclass

# describe
$ kubectl describe gatewayclass
```

<br/>

## Install a Gateway

Next we need to install a Gateway that implements our Gateway Class </br>
Note that we use a Traefik Gateway in our example. </br>

[Documentation](https://gateway-api.sigs.k8s.io/api-types/gateway)

This gateway lives in the same namespace as the routes and applications

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway-api
  namespace: default
spec:
  gatewayClassName: traefik

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

```
$ kubectl describe gateway gateway-api
```

<br/>

```
***
Error while retrieving certificate: getting secret: secret "secret-tls" not found
***
```

<br/>

## Traffic Management Features: HTTP Routes

<br/>

```shell
# port forward for access
$ kubectl -n traefik port-forward svc/traefik 8080:80
```

<br/>

```shell
$ kubectl -n traefik port-forward svc/traefik 8081:443
```

<br/>

[checks](/tools/containers/kubernetes/utils/gateway-api/checks/)

<br/>

## Middlewares

One of the key strengths of using Traefik is that they have a catalogue of middlewares that are plugins to perform popular common actions on traffic. </br>

There are many [Middlewares](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/overview/) available.

The cool thing here is that these Middlewares are now pluggable into Gateway API HTTPRoutes. </br>

In `HTTPRoute` objects, under `filters`, we can add extensions using something like:

<br/>

```yaml
filters:
  - type: ExtensionRef
    extensionRef:
      group: traefik.io
      kind: Middleware
      name: add-prefix
```

<br/>

### Prefix

Note: To use `ExtensionRef` in Traefik you need to enable `providers.kubernetesCRD.enabled = true`. We have that enabled in our values.yaml file.

Expectation: </br>

http://example-app.com/anything 👉🏽 http://go-svc:5000/prefix/anything </br>

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: go-route
  namespace: default

spec:
  parentRefs:
  - name: gateway-api
    sectionName: http
    kind: Gateway

  hostnames:
  - example-app.com

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    filters:
    - type: ExtensionRef
      extensionRef:
        group: traefik.io
        kind: Middleware
        name: add-prefix

    backendRefs:
    - name: go-svc
      port: 5000
      weight: 1
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: add-prefix
  namespace: default
spec:
  addPrefix:
    prefix: /prefix
EOF
```

We can follow the log of our upstream to see the HTTP request:

```shell
kubectl logs -f -l app=go-app
```

### Basic Auth

We can also add HTTP basic authentication to allow users to access secured resources protected by username and passwords. </br>

The Basic Auth middleware points to a secret which supports Kubernetes basic auth secret type. </br>

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: go-route
  namespace: default

spec:
  parentRefs:
  - name: gateway-api
    sectionName: http
    kind: Gateway

  hostnames:
  - example-app.com

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    filters:
    - type: ExtensionRef
      extensionRef:
        group: traefik.io
        kind: Middleware
        name: basic-auth

    backendRefs:
    - name: go-svc
      port: 5000
      weight: 1
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: basic-auth
  namespace: default
spec:
  basicAuth:
    secret: basic-auth
---
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: kubernetes.io/basic-auth
stringData:
  username: admin
  password: t0p-Secret
EOF
```

<br/>

### Headers

Traefik allows us to manipulate headers too using Middleware component called `headers` </br>
We can use this to handle scenarios like CORS (Cross-Origin Resource Sharing)

Let's access our web app directly over `localhost` which makes a call to `example-app.com` and should be blocked by CORS. </br>

<br/>

```shell
$ kubectl port-forward svc/web-app 8000:80
```

Once we apply the Middleware update to our HTTPRoute, we can perform cross origin API calls:

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: go-route
  namespace: default

spec:
  parentRefs:
  - name: gateway-api
    sectionName: http
    kind: Gateway

  hostnames:
  - example-app.com

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api/go
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    - type: ExtensionRef
      extensionRef:
        group: traefik.io
        kind: Middleware
        name: headers

    backendRefs:
    - name: go-svc
      port: 5000
      weight: 1
---
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: headers
spec:
  headers:
    accessControlAllowMethods:
      - "GET"
      - "OPTIONS"
      - "PUT"
    accessControlAllowHeaders:
      - "*"
    accessControlAllowOriginList:
      - "http://localhost:8000"
    accessControlMaxAge: 100
    addVaryHeader: true
EOF
```
