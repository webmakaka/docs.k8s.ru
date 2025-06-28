---
layout: page
title: GitOps Cookbook - Cloud Native CI/CD - Tekton - Create a Task to Compile and Package an App from Private Git
description: GitOps Cookbook - Cloud Native CI/CD - Tekton - Create a Task to Compile and Package an App from Private Git
keywords: books, gitops, cloud-native-cicd, tekton, Create a Task to Compile and Package an App from Private Git
permalink: /books/gitops/gitops-cookbook/cloud-native-cicd/tekton/create-a-task-to-compile-and-package-an-app-from-private-git/
---

<br/>

# [Book] [OK!] GitOps Cookbook: 06. Cloud Native CI/CD: Tekton: 6.4 Create a Task to Compile and Package an App from Private Git

<br/>

Делаю:  
2024.03.08

<br/>

Пересоздал minikube

<br/>

1. Создаю private repo https://github.com/wildmakaka/wildmakaka-tekton-greeter-private.git
2. Копирую в него https://github.com/gitops-cookbook/tekton-tutorial-greeter в ветку main.

<br/>

3. Создаю токен для работы с приватным repo

https://github.com/settings/tokens

<br/>

Или можно кликать по иконкам:

**GithubUserName -> Settings -> Developer Settings -> Personal access token -> Tokens (classic) ->**

<br/>

**Genereate new token (classic) -> TEKTON_TOKEN**

<br/>

```yaml
$ cat << 'EOF' | kubectl create -f -
apiVersion: v1
kind: Secret
metadata:
  name: github-secret
  annotations:
    tekton.dev/git-0: https://github.com
type: kubernetes.io/basic-auth
stringData:
  username: YOUR_USERNAME
  password: YOUR_PASSWORD
EOF
```

<br/>

```
YOUR_USERNAME - github username
YOUR_PASSWORD - GitHub personal access token
```

<br/>

```yaml
$ cat << 'EOF' | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-bot-sa
secrets:
  - name: github-secret
EOF
```

<br/>

```yaml
$ cat << 'EOF' | kubectl create -f -
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: build-app
spec:
  workspaces:
    - name: source
      description: The git repo will be cloned onto the volume backing this work space
  params:
    - name: contextDir
      description: the context dir within source
      default: quarkus
    - name: tlsVerify
      description: tls verify
      type: string
      default: "false"
    - name: url
      default: https://github.com/wildmakaka/wildmakaka-tekton-greeter-private.git
    - name: revision
      default: main
    - name: subdirectory
      default: ""
    - name: sslVerify
      description: defines if http.sslVerify should be set to true or false in the global git config
      type: string
      default: "false"
  steps:
    - image: 'gcr.io/tekton-releases/github.com/tektoncd/pipeline/cmd/git-init:v0.21.0'
      name: clone
      resources: {}
      script: |
          CHECKOUT_DIR="$(workspaces.source.path)/$(params.subdirectory)"
          cleandir() {
          # Delete any existing contents of the repo directory if it exists.
          #
          # We don't just "rm -rf $CHECKOUT_DIR" because $CHECKOUT_DIR might be "/"
          # or the root of a mounted volume.
          if [[ -d "$CHECKOUT_DIR" ]] ; then
          # Delete non-hidden files and directories
          rm -rf "$CHECKOUT_DIR"/*
          # Delete files and directories starting with . but excluding ..
          rm -rf "$CHECKOUT_DIR"/.[!.]*
          # Delete files and directories starting with .. plus any other character
          rm -rf "$CHECKOUT_DIR"/..?*
          fi
          }
          /ko-app/git-init \
          -url "$(params.url)" \
          -revision "$(params.revision)" \
          -path "$CHECKOUT_DIR" \
          -sslVerify="$(params.sslVerify)"
          cd "$CHECKOUT_DIR"
          RESULT_SHA="$(git rev-parse HEAD)"
    - name: build-sources
      image: gcr.io/cloud-builders/mvn
      command:
        - mvn
      args:
        - -DskipTests
        - clean
        - install
      env:
        - name: user.home
          value: /home/tekton
      workingDir: "/workspace/source/$(params.contextDir)"
EOF
```

<br/>

### Запуск

```
$ kubectl get tasks
```

<br/>

```yaml
$ cat << 'EOF' | kubectl create -f -
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  generateName: build-app-run-
  labels:
    app.kubernetes.io/managed-by: tekton-pipelines
    tekton.dev/task: build-app
spec:
  serviceAccountName: tekton-bot-sa
  params:
  - name: contextDir
    value: quarkus
  - name: revision
    value: main
  - name: sslVerify
    value: "false"
  - name: subdirectory
    value: ""
  - name: tlsVerify
    value: "false"
  - name: url
    value: https://github.com/wildmakaka/wildmakaka-tekton-greeter-private.git
  taskRef:
    kind: Task
    name: build-app
  workspaces:
  - emptyDir: {}
    name: source
EOF
```

<br/>

```
$ tkn taskrun ls
NAME                  STARTED          DURATION   STATUS
build-app-run-28gtk   54 seconds ago   48s        Succeeded
```

<br/>

```
$ tkn taskrun logs build-app-run-28gtk -f
[build-sources] [INFO] ------------------------------------------------------------------------
[build-sources] [INFO] BUILD SUCCESS
[build-sources] [INFO] ------------------------------------------------------------------------
[build-sources] [INFO] Total time:  42.830 s
[build-sources] [INFO] Finished at: 2024-03-08T11:03:27Z
[build-sources] [INFO] ---------------------------------------------------------
```

<br/>

### Запуск как в книге

```
$ tkn task start build-app \
--serviceaccount='tekton-bot-sa' \
--param url='https://github.com/wildmakaka/wildmakaka-tekton-greeter-private.git' \
--param contextDir='quarkus' \
--workspace name=source,emptyDir="" \
--showlog
```
