---
layout: page
title: Gateway API
description: Gateway API
keywords: tools, containers, kubernetes, Gateway API
permalink: /tools/containers/kubernetes/utils/gateway-api/
---

# Gateway API

Делаю:  
2026.02.02

<br/>

https://www.youtube.com/watch?v=q76XVCTDZCY

https://github.com/marcel-dempers/docker-development-youtube-series/tree/master/kubernetes/gateway-api

<br/>

## We need a Kubernetes cluster

<br/>

```shell
$ kind create cluster --name gatewayapi --image kindest/node:v1.35.0
```

<br/>

```shell
$ kubectl get nodes
NAME                       STATUS     ROLES           AGE   VERSION
gatewayapi-control-plane   NotReady   control-plane   11s   v1.35.0
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

## Setup some example applications

<br/>

```
1) Приложение на python
2) Приложение на go
3) Фронт приложение которое умеет делать запросы к go приложени.
```

<br/>

The following will deploy a `deployment`, `service` and require `configMap` and `secret` for the applications to work.

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-deploy
  labels:
    app: python-app
spec:
  selector:
    matchLabels:
      app: python-app
  replicas: 2
  template:
    metadata:
      labels:
        app: python-app
    spec:
      containers:
      - name: python-app
        image: aimvector/python:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
---
kind: Service
apiVersion: v1
metadata:
  name: python-svc
spec:
  selector:
    app: python-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
  type: ClusterIP
EOF
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-deploy
  labels:
    app: go-app
spec:
  selector:
    matchLabels:
      app: go-app
  replicas: 2
  template:
    metadata:
      labels:
        app: go-app
    spec:
      containers:
      - name: go-app
        image: aimvector/golang:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
        volumeMounts:
        - name: secret-volume
          mountPath: /secrets/
        - name: config-volume
          mountPath: /configs/
      volumes:
      - name: secret-volume
        secret:
          secretName: go-secret
      - name: config-volume
        configMap:
          name: go-config
---
apiVersion: v1
kind: Secret
metadata:
  name: go-secret
type: Opaque
data:
  secret.json: ewogICAgImFwaV9rZXkiIDogInNvbWVzZWNyZXRnb2VzaGVyZSIKfQo=
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: go-config
data:
  config.json: |
    {
      "environment" : "dev"
    }
---
kind: Service
apiVersion: v1
metadata:
  name: go-svc
spec:
  selector:
    app: go-app
  ports:
    - protocol: TCP
      port: 5000
      targetPort: 5000
  type: ClusterIP
EOF
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app
  labels:
    app: web-app
data:
  # The index.html file that NGINX will serve
  index.html: |
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>CORS Demo Client</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 40px; background-color: #f0f4f8; }
            .container { background: white; padding: 30px; border-radius: 12px; box-shadow: 0 4px 12px rgba(0, 0, 0, 0.1); max-width: 600px; margin: auto; }
            h1 { color: #2c3e50; border-bottom: 2px solid #3498db; padding-bottom: 10px; }
            button { background-color: #3498db; color: white; padding: 10px 15px; border: none; border-radius: 6px; cursor: pointer; margin-top: 15px; font-weight: bold; transition: background-color 0.3s; }
            button:hover { background-color: #2980b9; }
            pre { background-color: #ecf0f1; padding: 15px; border-radius: 6px; overflow-x: auto; white-space: pre-wrap; word-wrap: break-word; }
            .status-ok { color: green; font-weight: bold; }
            .status-error { color: red; font-weight: bold; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Client-Side CORS Demonstration</h1>
            <button onclick="fetchData()">Make CORS Request</button>
            <h2>Result:</h2>
            <p>Status: <span id="status">Waiting...</span></p>
            <pre id="output"></pre>
        </div>

        <script>
            async function fetchData() {
                const apiEndpoint = 'http://example-app.com/api/go';
                const statusElement = document.getElementById('status');
                const outputElement = document.getElementById('output');

                statusElement.textContent = 'Fetching...';
                outputElement.textContent = 'Request sent to ' + apiEndpoint;

                try {
                    // This public API allows CORS, so the fetch should succeed.
                    const response = await fetch(apiEndpoint);

                    if (!response.ok) {
                        throw new Error(`HTTP error! status: ${response.status}`);
                    }

                    const data = await response.text();

                    statusElement.innerHTML = '<span class="status-ok">SUCCESS (CORS Allowed)</span>';
                    outputElement.textContent = data;

                } catch (error) {
                    // If the API did NOT allow CORS, this catch block would activate,
                    // often showing a generic "Failed to fetch" error in the console.
                    statusElement.innerHTML = '<span class="status-error">FAILURE (Check Browser Console for CORS Error)</span>';
                    outputElement.textContent = 'Error during fetch: ' + error.message;
                    console.error("CORS Request Error:", error);
                }
            }
        </script>
    </body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx-web
        image: nginx:stable-alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html/index.html
          subPath: index.html
      volumes:
      - name: html-volume
        configMap:
          name: web-app
          defaultMode: 0644
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      name: http
EOF
```

<br/>

```shell
$ kubectl get pods
NAME                             READY   STATUS    RESTARTS   AGE
go-deploy-5449f5df89-2mhss       1/1     Running   0          35s
go-deploy-5449f5df89-jswnq       1/1     Running   0          35s
python-deploy-5fc58df78c-px8jx   1/1     Running   0          40s
python-deploy-5fc58df78c-qlmvs   1/1     Running   0          40s
web-app-679f5b5857-qw8r9         1/1     Running   0          26s
```

<br/>

```shell
$ kubectl get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
go-svc       ClusterIP   10.96.167.54    <none>        5000/TCP   58s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP    8m3s
python-svc   ClusterIP   10.96.218.1     <none>        5000/TCP   63s
web-app      ClusterIP   10.96.236.255   <none>        80/TCP     49s
```

<br/>

### Create test Domains

We also need to imagine we have a domain called `example-app.com` , so let's set that up on our hosts file

<br/>

```shell
$ echo "127.0.0.1 example-app.com" | sudo tee -a /etc/hosts
$ echo "127.0.0.1 example-app-go.com" | sudo tee -a /etc/hosts
$ echo "127.0.0.1 example-app-python.com" | sudo tee -a /etc/hosts
```
