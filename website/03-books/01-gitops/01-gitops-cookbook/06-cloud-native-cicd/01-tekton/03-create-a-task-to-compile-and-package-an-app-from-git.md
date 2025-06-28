---
layout: page
title: GitOps Cookbook - Cloud Native CI/CD - Tekton - Create a Task to Compile and Package an App from Git
description: GitOps Cookbook - Cloud Native CI/CD - Tekton - Create a Task to Compile and Package an App from Git
keywords: books, gitops, cloud-native-cicd, tekton, Create a Task to Compile and Package an App from Git
permalink: /books/gitops/gitops-cookbook/cloud-native-cicd/tekton/create-a-task-to-compile-and-package-an-app-from-git/
---

<br/>

# [Book] [OK!] GitOps Cookbook: 06. Cloud Native CI/CD: Tekton: 6.3 Create a Task to Compile and Package an App from Git

<br/>

Делаю:  
2024.03.08

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
      default: https://github.com/gitops-cookbook/tekton-tutorial-greeter.git
    - name: revision
      default: master
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
  params:
  - name: contextDir
    value: quarkus
  - name: revision
    value: master
  - name: sslVerify
    value: "false"
  - name: subdirectory
    value: ""
  - name: tlsVerify
    value: "false"
  - name: url
    value: https://github.com/gitops-cookbook/tekton-tutorial-greeter.git
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
$ kubectl get tasks
build-app   36s
```

<br/>

```
$ tkn taskrun ls
NAME                  STARTED          DURATION   STATUS
build-app-run-f5ctk   3 minutes ago    1m38s      Succeeded
```

<br/>

```
$ tkn taskrun logs build-app-run-f5ctk -f
***
[build-sources] [INFO] ------------------------------------------------------------------------
[build-sources] [INFO] BUILD SUCCESS
[build-sources] [INFO] ------------------------------------------------------------------------
[build-sources] [INFO] Total time:  02:35 min
[build-sources] [INFO] Finished at: 2024-03-08T10:14:04Z
[build-sources] [INFO] ------------------------------------------------------------------------
```
