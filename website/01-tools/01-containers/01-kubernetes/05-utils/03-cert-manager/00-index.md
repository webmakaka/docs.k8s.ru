---
layout: page
title: Cert Manager + Gateway API
description: Cert Manager + Gateway API
keywords: tools, containers, kubernetes, Cert Manager + Gateway API
permalink: /tools/containers/kubernetes/utils/cert-manager/
---

# Cert Manager + Gateway API

Делаю:  
2026.02.06

<br/>

**Не работает! Т.к. нет публичного адреса до хоста с kubernetes.**

<br/>

https://www.youtube.com/watch?v=XAcmHWZ02VI

https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/kubernetes/cert-manager

<br/>

```shell
$ kind create cluster --name certmanager --image kindest/node:v1.35.0
```

<br/>

## Deploy pods that require SSL\TLS

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deploy
  labels:
    app: example-app
    test: test
  annotations:
    fluxcd.io/tag.example-app: semver:~1.0
    fluxcd.io/automated: 'true'
spec:
  selector:
    matchLabels:
      app: example-app
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: example-app
    spec:
      containers:
      - name: example-app
        image: aimvector/python:1.0.4
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "256Mi"
            cpu: "500m"
      tolerations:
      - key: "cattle.io/os"
        value: "linux"
        effect: "NoSchedule"
EOF
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: example-service
  labels:
    app: example-app
spec:
  type: ClusterIP
  selector:
    app: example-app
  ports:
    - protocol: TCP
      name: http
      port: 80
      targetPort: 5000
EOF
```

<br/>

```shell
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
example-deploy-6489756d87-4g5c5   1/1     Running   0          102s
example-deploy-6489756d87-d9tzr   1/1     Running   0          102s
```

<br/>

## Gateway API CRDs

<br/>

```shell
$ kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/experimental-install.yaml
```

This will install (Stable Channel):

- Gateway Classes: `kubectl get gatewayclass`
- Gateways: `kubectl get gateway`
- HTTP Routes: `kubectl get httproute`

These APIs are part of the Experimental Channel:

- TLS Routes: `kubectl get tlsroute`
- TCP Routes: `kubectl get tcproute`
- UDP Routes: `kubectl get udproute`

<b>Note: Gateway API is very new, and all of the above is subject to change quite rapidly </br></b>

<br/>

### Install cert-manager

```shell
// install cert-manager
$ CHART_VERSION="v1.19.2"

// checkout the values
// $ helm show values oci://quay.io/jetstack/charts/cert-manager > kubernetes/cert-manager/default-values.yaml

$ helm install \
  cert-manager oci://quay.io/jetstack/charts/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version ${CHART_VERSION} \
  --set crds.enabled=true \
  --set config.enableGatewayAPI=true
```

<br/>

```
$ kubectl -n cert-manager get pods
NAME                                      READY   STATUS    RESTARTS       AGE
cert-manager-7785549dcf-r8qtt             1/1     Running   5 (117s ago)   3m36s
cert-manager-cainjector-c898c8b47-2rhlh   1/1     Running   0              3m36s
cert-manager-webhook-758795746d-85lcp     1/1     Running   0              3m36s
```

<br/>

## Setup my DNS (У меня пока такого нет, точнее не хочется запарываться)

I can get the public IP address of my computer by running a simple command:

```
$ curl ifconfig.co
```

I can log into my DNS provider and point my DNS A record to my IP.<br/>
Also setup my router to allow 80 and 443 to come to my PC <br/>

<i>Note: If you are running in the cloud, your Ingress controller or Gateway API and Cloud provider will give you a
public IP and you can point your DNS to that accordingly.
</i>

<br/>

### Create a Gateway Class

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

### Create a Gateway

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

### Create a Gateway API Let's Encrypt Issuer

We create a `ClusterIssuer` for Ingress that allows us to issue certs in any namespace

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-issuer-key
    solvers:
    - http01:
        gatewayHTTPRoute:
          parentRefs:
            - name: gateway-api
              namespace: default
              kind: Gateway
EOF
```

<br/>

```shell
// check the issuer
$ kubectl describe clusterissuer letsencrypt-issuer
```

<br/>

### Issue Certificate

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-app
  namespace: default
spec:
  dnsNames:
    - test.marceldempers.dev
  secretName: secret-tls
  issuerRef:
    name: letsencrypt-issuer
    kind: ClusterIssuer
EOF
```

<br/>

```shell
// check the cert issue status
$ kubectl describe certificate example-app
```

<br/>

```shell
// READY - FALSE!
$ kubectl get CertificateRequest
NAME            APPROVED   DENIED   READY   ISSUER               REQUESTER                                         AGE
example-app-1   True                False   letsencrypt-issuer   system:serviceaccount:cert-manager:cert-manager   14m
```

<br/>

```
$ kubectl get Orders
NAME                       STATE     AGE
example-app-1-3040804028   pending   17m
```

<br/>

```
$ kubectl describe Orders example-app-1-3040804028
```

<br/>

```shell
// TLS created as a secret
$ kubectl get secrets

Не создался. А должен быть.

NAME                  TYPE                                  DATA   AGE
secret-tls       kubernetes.io/tls                     2      1m
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: example-route
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
  - test.marceldempers.dev

  rules:
  - backendRefs:
    - name: example-service
      port: 80
      weight: 1
EOF
```

<br/>

```shell
// test TLS
$ curl https://test.marceldempers.dev
Hello World!
```

<br/>

```shell
// cleanup
$ kubectl delete certificate example-app
$ kubectl delete clusterissuer letsencrypt-issuer
$ kubectl delete secret secret-tls
```
