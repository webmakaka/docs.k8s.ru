---
layout: page
title: Traefik Gateway API for Kubernetes
description: Traefik Gateway API for Kubernetes
keywords: tools, containers, kubernetes, Gateway API, Traefik
permalink: /tools/containers/kubernetes/utils/gateway-api/checks/
---

Делаю:  
2026.04.10

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

### Next step

Let's `port-forward` to 443 since that is where TLS is exposed:

<br/>

**Result:** </br>
https://example-app.com:8081/api/go 👉🏽 http://go-svc:5000/ </br>

Checkout [More Official Guides](https://gateway-api.sigs.k8s.io/guides/) on the Kubernetes Gateway API SIGs page.</br>
