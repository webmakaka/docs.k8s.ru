---
layout: page
title: Cert Manager
description: Cert Manager
keywords: tools, containers, kubernetes, Cert Manager
permalink: /tools/containers/kubernetes/utils/cert-manager/prev/
---

# Cert Manager

Делаю:  
2026.02.03

<br/>

https://www.youtube.com/watch?v=hoLUigg4V18

https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/kubernetes/cert-manager

<br/>

## Поехали

```shell
// Запуск кластера kind
$ kind create cluster --name certmanager --image kindest/node:v1.35.0
```

<br/>

https://github.com/jetstack/cert-manager/releases/

<br/>

```shell
$ cd ~/tmp
$ curl -LO https://github.com/jetstack/cert-manager/releases/download/v1.19.3/cert-manager.yaml

$ mv cert-manager.yaml cert-manager-1.19.3.yaml
```

<br/>

```shell
// install cert-manager
$ kubectl apply --validate=false -f cert-manager-1.19.3.yaml
```

<br/>

```shell
$ kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-845844dd8-xn7d2               1/1     Running   0          107s
cert-manager-cainjector-7b5d65fbcb-sb4zn   1/1     Running   0          107s
cert-manager-webhook-6fcf4cb6c-cqwmc       1/1     Running   0          107s
```

<br/>

## Test Certificate Issuing

Let's create some test certificates

```shell
$ kubectl create ns cert-manager-test
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: test-selfsigned
  namespace: cert-manager-test
spec:
  selfSigned: {}
EOF
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: selfsigned-cert
  namespace: cert-manager-test
spec:
  dnsNames:
    - example.com
  secretName: selfsigned-cert-tls
  issuerRef:
    name: test-selfsigned
EOF
```

<br/>

```
$ kubectl describe certificate -n cert-manager-test
$ kubectl get secrets -n cert-manager-test
$ kubectl delete ns cert-manager-test
```

<br/>

## Ingress Controller

```
$ kubectl create ns ingress-nginx
```

<br/>

https://github.com/kubernetes/ingress-nginx

<br/>

```
$ kubectl -n ingress-nginx apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.14.3/deploy/static/provider/cloud/deploy.yaml
```

<br/>

```
$ kubectl get pods -n ingress-nginx
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-7fdf8d9764-thnjw   1/1     Running   0          49s
```

<br/>

```
$ kubectl get svc -n ingress-nginx
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx-controller             LoadBalancer   10.96.6.115    <pending>     80:31983/TCP,443:30265/TCP   2m15s
ingress-nginx-controller-admission   ClusterIP      10.96.196.39   <none>        443/TCP                      2m15s
```

<br/>

```
$ kubectl -n ingress-nginx --address 0.0.0.0 port-forward svc/ingress-nginx-controller 8080:80
$ kubectl -n ingress-nginx --address 0.0.0.0 port-forward svc/ingress-nginx-controller 8081:443
```

<br/>

```
// Ok!
http://localhost:8080
https://localhost:8081
```

<br/>

## Setup my DNS

In my container, I can get the public IP address of my computer by running a simple command:

```
curl ifconfig.co
```

I can log into my DNS provider and point my DNS A record to my IP.<br/>
Also setup my router to allow 80 and 443 to come to my PC <br/>

If you are running in the cloud, your Ingress controller and Cloud provider will give you a public IP and you can point your DNS to that accordingly.

<br/>

## Create Let's Encrypt Issuer for our cluster

We create a `ClusterIssuer` that allows us to issue certs in any namespace

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-cluster-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@email.com
    privateKeySecretRef:
      name: letsencrypt-cluster-issuer-key
    solvers:
    - http01:
       ingress:
         class: nginx
EOF
```

<br/>

```
// check the issuer
$ kubectl describe clusterissuer letsencrypt-cluster-issuer
```

<br/>

## Deploy a pod that uses SSL

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
# kind create cluster --name deployments --image kindest/node:v1.31.1
# kubectl apply -f deployment.yaml

# deployment.yaml
---
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
        # livenessProbe:
        #   httpGet:
        #     path: /status
        #     port: 5000
        #   initialDelaySeconds: 3
        #   periodSeconds: 3
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
#NOTE: comment out `volumeMounts` section for configmap and\or secret guide
        # volumeMounts:
        # - name: secret-volume
        #   mountPath: /secrets/
        # - name: config-volume
        #   mountPath: /configs/
#NOTE: comment out `volumes` section for configmap and\or secret guide
      # volumes:
      # - name: secret-volume
      #   secret:
      #     secretName: mysecret
      # - name: config-volume
      #   configMap:
      #     name: example-config #name of our configmap object
EOF
```

<br/>

```
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
example-deploy-6489756d87-gcx9z   1/1     Running   0          2m8s
example-deploy-6489756d87-zd86s   1/1     Running   0          2m8s
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

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: "nginx"
  name: example-app
spec:
  tls:
  - hosts:
    - marcel.guru
    secretName: example-app-tls
  rules:
  - host: marcel.guru
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example-service
            port:
              number: 80
EOF
```

<br/>

## Issue Certificate

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
    - marcel.guru
  secretName: example-app-tls
  issuerRef:
    name: letsencrypt-cluster-issuer
    kind: ClusterIssuer
EOF
```

<br/>

```
// check the cert has been issued
$ kubectl describe certificate example-app
```

<br/>

```
$ kubectl get secrets
NAME TYPE DATA AGE
example-app-q8cxl Opaque 1 64s
```
