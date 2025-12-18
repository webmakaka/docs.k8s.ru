---
layout: page
title: Инсталляция Metal LB
description: Инсталляция Metal LB
keywords: gitops, containers, kubernetes, metal lb
permalink: /tools/containers/kubernetes/utils/metal-lb/minikube/setup/addon/
---

# Инсталляция Metal LB

<br/>

**Делаю:**  
2025.12.17

<br/>

Metal LB позволит получить внешний IP в миникубе на локалхосте. Аналогично тому, как это происходит в облаках, когда облачный сервис выделяет ip адрес, к котому можно будет подключиться извне.

<br/>

```
$ minikube addons enable metallb -p ${PROFILE}
```

<br/>

```
$ minikube --profile ${PROFILE} ip
192.168.49.2
```

<br/>

(Возможно это лишнее)

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.49.20-192.168.49.30
EOF
```

<br/>

### Проверка при необходимости

<br/>

```yaml
$ cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: test-loadbalancer-8080
spec:
  selector:
    app: test
  ports:
    - port: 8080      # Порт сервиса
      targetPort: 80  # Порт пода
  type: LoadBalancer
EOF
```

```yaml
$ cat << 'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF
```

<br/>

```
// EXTERNAL-IP получен
$ kubectl get svc test-loadbalancer-8080
NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)          AGE
test-loadbalancer-8080   LoadBalancer   10.96.188.65   192.168.49.20   8080:31229/TCP   13s
```

<br/>

```
// OK
$ curl http://192.168.49.20:8080
```
