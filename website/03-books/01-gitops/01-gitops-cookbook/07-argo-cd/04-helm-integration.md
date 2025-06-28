---
layout: page
title: GitOps Cookbook - Argo CD - Helm Integration
description: GitOps Cookbook - Argo CD - Helm Integration
keywords: books, gitops, argo-cd, Helm Integration
permalink: /books/gitops/gitops-cookbook/argo-cd/helm-integration/
---

<br/>

# [Book] [OK!] 7.4 Helm Integration

<br/>

Делаю:  
2024.03.24

<br/>

gitops-cookbook-sc/ch07/bgdh

<br/>

```yaml
service:
  type: NodePort
  port: 8080
```

gitops-cookbook-sc/ch07/bgdh/templates/deployment.yaml

<br/>

```yaml
ports:
  - name: http
    containerPort: 8080
    protocol: TCP
```

<br/>

```yaml
$ cat << 'EOF' | kubectl create -f -
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bgdh-app
  namespace: argocd
spec:
  destination:
    namespace: bgdh
    server: https://kubernetes.default.svc
  project: default
  source:
    path: ch07/bgdh
    repoURL: https://github.com/wildmakaka/gitops-cookbook-sc.git
    targetRevision: main
  syncPolicy:
    automated: {}
EOF
```

<!-- <br/>

```
$ kubectl patch svc bgdh-app -n bgdh -p '{"spec": {"type": "NodePort"}}'
``` -->

<br/>

```
$ kubectl get services -n bgdh
NAME       TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
bgdh-app   NodePort   10.103.112.150   <none>        8080:32256/TCP   6s
```

<br/>

```
// [OK!]
http://192.168.49.2:32256
```

<br/>

Наверное, так делать не оч. правильно.

gitops-cookbook-sc/ch07/bgdh/templates/deployment.yaml

Меняю цвет

<br/>

```yaml
      containers:
       ***
          env:
            - name: COLOR
              value: "red"
```

<br/>

```
$ argocd app sync bgdh-app
```

<br/>

```
// [OK!] Цвет обновился!
http://192.168.49.2:32256
```

<br/>

```
$ argocd app delete bgdh-app
```
