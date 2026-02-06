---
layout: page
title: Инсталляция kubectl в ubuntu 22.04
description: Инсталляция kubectl в ubuntu 22.04
keywords: tools, containers, kubernetes, Gateway API, Traefik
permalink: /tools/containers/kubernetes/utils/gateway-api/traefik/
---

# Traefik Gateway API for Kubernetes

Делаю:  
2026.02.06

<br/>

https://www.youtube.com/watch?v=MN-k29g97ik

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
$ cat > gateway-api-values.yaml << EOF
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
$ helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --version $CHART_VERSION \
  --values gateway-api-values.yaml
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

[Documentation](https://gateway-api.sigs.k8s.io/api-types/httproute/)

The important fields on HTTP Route we will cover:

- `parentRefs`
- `sectionName`
- `hostnames`
- `rules`
- `matches`
- `filters`

Feature Table:

| Feature                 | Example                                         |
| ----------------------- | ----------------------------------------------- |
| Route by Hostname       | [example](#route-by-hostname)                   |
| Route by Path           | [example](#route-by-path)                       |
| Route using URL Rewrite | [example](#route-using-url-rewrite)             |
| Header Modification     | [example](#requestresponse-header-manipulation) |
| HTTPS & TLS             | [example](#https-and-tls)                       |

For traffic management, we can take a look at some basic HTTP routes.</br>

<br/>

```shell
# port forward for access
$ kubectl -n traefik port-forward svc/traefik 8080:80
```

<br/>

### Route by Hostname

We can route by host. </br>
This will route all traffic that matches the `Host` header with the `hostnames` field: </br>

http://example-app-python.com/ 👉🏽 http://python-svc:5000 </br>
http://example-app-go.com/ 👉🏽 http://go-svc:5000 </br>

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: python-route
  namespace: default

spec:
  parentRefs:
  - name: gateway-api
    sectionName: http #allows us to bind to a specific listener in the Gateway
    kind: Gateway

  hostnames:
  - example-app-python.com

  rules:
  - backendRefs:
    - name: python-svc
      port: 5000
      weight: 1
---
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
  - example-app-go.com

  rules:
  - backendRefs:
    - name: go-svc
      port: 5000
      weight: 1
EOF
```

<br/>

```
// Ok!
http://example-app-python.com:8080/
http://example-app-go.com:8080/
```

<br/>

### Route by Path

We can also route by host and path with different matching strategies. </br>

Exact: </br>
http://example-app-python.com/ 👉🏽 http://python-svc:5000/ </br>
http://example-app-go.com/ 👉🏽 http://go-svc:5000/ </br>

PathPrefix: </br>
http://example-app-python.com/_ 👉🏽 http://python-svc:5000/_ </br>
http://example-app-go.com/_ 👉🏽 http://go-svc:5000/_ </br>

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: python-route
  namespace: default

spec:
  parentRefs:
  - name: gateway-api
    sectionName: http
    kind: Gateway

  hostnames:
  - example-app-python.com

  rules:
  - matches:
    - path:
        type: Exact  # matches the exact path, only allows to visit /
        value: /
    backendRefs:
    - name: python-svc
      port: 5000
      weight: 1
---
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
  - example-app-go.com

  rules:
  - matches:
    - path:
        type: Exact  # matches the exact path, only allows to visit /
        #type: PathPrefix # prefix match, allows to visit /*  Passes entire URL to upstream
        value: /
    backendRefs:
    - name: go-svc
      port: 5000
      weight: 1
EOF
```

<br/>

### Route using URL Rewrite

We can rewrite the hostname or URL using URL rewrite. </br>
This way, we can combine our services under one domain and our controller can act as a true API gateway:

http://example-app.com/api/python 👉🏽 http://python-svc:5000/ </br>
http://example-app.com/api/go 👉🏽 http://go-svc:5000/ </br>
As well as: </br>
http://example-app.com/api/go/status 👉🏽 http://go-svc:5000/status </br>

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: python-route
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
        value: /api/python
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: python-svc
      port: 5000
      weight: 1
---
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
    backendRefs:
    - name: go-svc
      port: 5000
      weight: 1
EOF
```

<br/>

### Request\Response Header Manipulation

With Gateway API, you can modify request and response headers. </br>
This is possible with the `ResponseHeaderModifier` filter </br>

At the time of this recording, Gateway API does not natively support CORS. </br>
Even with it in the Experimental channel, many controllers do not support it yet. </br>

Let's do a basic CORS header modification for our Go HTTPRoute </br>

<br/>

```shell
$ kubectl port-forward svc/web-app 8000:80
```

The Header modification:

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
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        add:
        - name: X-Custom-Header
          value: "CustomHeaderValue"
        - name: Access-Control-Allow-Origin
          value: "http://localhost:8000"
        - name: Access-Control-Allow-Methods
          value: "GET, POST, PUT, DELETE, OPTIONS"
        - name: Access-Control-Allow-Headers
          value: "Content-Type, Authorization"
        - name: Access-Control-Max-Age
          value: "86400" # Cache preflight for 24 hours
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /
    backendRefs:
    - name: go-svc
      port: 5000
      weight: 1
EOF
```

<br/>

### HTTPS and TLS

In my video, I generate a test TLS cert using [mkcert](https://github.com/FiloSottile/mkcert)

```shell
$ curl -L https://github.com/FiloSottile/mkcert/releases/download/v1.4.4/mkcert-v1.4.4-linux-amd64 -o mkcert && chmod +x mkcert && sudo mv mkcert /usr/local/bin/

#linux
$ export CAROOT=${PWD}/kubernetes/gateway-api/tls

$ mkcert -key-file kubernetes/gateway-api/tls/key.pem -cert-file kubernetes/gateway-api/tls/cert.pem example-app.com

$ mkcert -install
```

Now that we have a TLS cert, we can create a Kubernetes secret to store it:

```shell
$ kubectl create secret tls secret-tls -n default --cert kubernetes/gateway-api/tls/cert.pem --key kubernetes/gateway-api/tls/key.pem
```

<br/>

We need to

- Adjust our Gateway, to enable the TLS Listener first!
- Then apply the TLS listener in our HTTP Route to enable TLS, using `sectionName`

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
  - name: gateway-api
    sectionName: https
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
    backendRefs:
    - name: go-svc
      port: 5000
      weight: 1
EOF
```

Let's `port-forward` to 443 since that is where TLS is exposed:

```shell
$ kubectl -n traefik port-forward svc/traefik 8081:443
```

Result: </br>
https://example-app.com:8081/api/go 👉🏽 http://go-svc:5000/ </br>

Checkout [More Official Guides](https://gateway-api.sigs.k8s.io/guides/) on the Kubernetes Gateway API SIGs page.</br>

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
