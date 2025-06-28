---
layout: page
title: GitOps Cookbook - Helm
description: GitOps Cookbook - Helm
keywords: GitOps Cookbook, Helm
permalink: /books/gitops/gitops-cookbook/helm/
---

<br/>

# [Book] GitOps Cookbook: 04. Helm

<br/>

```
// scaffold the project
$ helm create <name>
```

<br/>

https://github.com/gitops-cookbook/helm-charts/tree/master/pacman

<br/>

```
$ helm template .
```

<br/>

```
// override
$ helm template --set replicaCount=3 .
```

<br/>

```
$ helm install pacman .
```

<br/>

```
$ kubectl get pods
```

<br/>

```
$ helm history pacman
```

<br/>

```
$ helm uninstall pacman
```

<br/>

### 5.3 Updating a Container Image in Helm

```
values.yaml - update the version
Chart.yaml - update the appVersion field
```

<br/>

```
$ helm upgrade pacman .
```

<br/>

```
$ helm history pacman
```

<br/>

```
$ helm rollback pacman 1
```

<br/>

```
$ helm template pacman -f newvalues.yaml .
```

<br/>

### 5.4 Packaging and Distributing a Helm Chart

<br/>

```
$ helm package .
```

<br/>

```
$ helm package --sign --key 'me@example.com' \
--keyring /home/me/.gnupg/secring.gpg .

$ helm verify pacman-0.1.0.tgz
```

<br/>

### [OK!] 5.5 Deploying a Chart from a Repository

<br/>

**Делаю:**  
2024.11.10

<br/>

```
$ helm repo add bitnami https://charts.bitnami.com/bitnami
```

<br/>

```
$ helm repo update
$ helm repo list
$ helm search repo postgresql
```

<br/>

```
$ helm install my-db \
  bitnami/postgresql \
  --namespace postgres \
  --create-namespace \
  --set auth.username=user1,auth.password=postgres1,auth.database=postgresdb1,primary.persistence.enabled=false
```

<br/>

```
$ kubectl get pods -n postgres
NAME                 READY   STATUS    RESTARTS   AGE
my-db-postgresql-0   1/1     Running   0          23s

```

<br/>

```
// To get the password for "user1" run:
$ {
  export POSTGRES_PASSWORD=$(kubectl get secret --namespace postgres my-db-postgresql -o jsonpath="{.data.password}" | base64 -d)
  echo ${POSTGRES_PASSWORD}
}
```

<br/>

```
// To connect to your database
$ kubectl run my-db-postgresql-client --rm --tty -i --restart='Never' --namespace postgres --image docker.io/bitnami/postgresql:15.3.0-debian-11-r4 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host my-db-postgresql -U user1 -d postgresdb1 -p 5432
```

<br/>

```
// To connect to your database from outside the cluster
$ kubectl port-forward --namespace postgres svc/my-db-postgresql 5432:5432 &
    PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U user1 -d postgresdb1 -p 5432
```

<br/>

```
$ kubectl get statefulset -n postgres
```

<br/>

```
$ helm show values bitnami/postgresql
```

<br/>

### [OK!] 5.6 Deploying a Chart with a Dependency

<br/>

**Делаю:**  
26.05.2023

<br/>

https://github.com/bitnami/charts/tree/main/bitnami/postgresql/#installing-the-chart

<br/>

```
$ cd ~/tmp
$ mkdir -p music/templates
$ cd music
```

<br/>

```yaml
$ cat > templates/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name}}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name}}
    {{- if .Chart.AppVersion }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name}}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name}}
    spec:
      containers:
          - image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion}}"
            imagePullPolicy: {{ .Values.image.pullPolicy }}
            securityContext:
              {{- toYaml .Values.securityContext | nindent 14 }}
            name: {{ .Chart.Name}}
            ports:
              - containerPort: {{ .Values.image.containerPort }}
                name: http
                protocol: TCP
            env:
              - name: QUARKUS_DATASOURCE_JDBC_URL
                value: {{ .Values.postgresql.server |
                default (printf "%s-postgresql" ( .Release.Name )) | quote }}
              - name: QUARKUS_DATASOURCE_USERNAME
                value: {{ .Values.postgresql.postgresqlUsername |
                default (printf "postgres" ) | quote }}
              - name: QUARKUS_DATASOURCE_PASSWORD
                valueFrom:
                  secretKeyRef:
                    name: {{ .Values.postgresql.secretName |
                    default (printf "%s-postgresql" ( .Release.Name )) | quote }}
                    key: {{ .Values.postgresql.secretKey }}
EOF
```

<br/>

```yaml
$ cat > templates/service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
  name: {{ .Chart.Name }}
spec:
  ports:
    - name: http
      port: {{ .Values.image.containerPort }}
      targetPort: {{ .Values.image.containerPort }}
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
EOF
```

<br/>

```yaml
$ cat > Chart.yaml << EOF
apiVersion: v2
name: music
description: A Helm chart for Music service

type: application
version: 0.1.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    repository: "https://charts.bitnami.com/bitnami"
    version: 12.5.5
EOF
```

<br/>

```yaml
$ cat > values.yaml << EOF
image:
  repository: quay.io/gitops-cookbook/music
  tag: "1.0.0"
  pullPolicy: Always
  containerPort: 8080

replicaCount: 1

postgresql:
  server: jdbc:postgresql://music-db-postgresql:5432/postgresdb1
  postgresqlUsername: user1
  secretName: music-db-postgresql
  secretKey: password
EOF
```

<br/>

```
$ helm dependency update
```

<br/>

```
.
├── Chart.lock
├── charts
│   └── postgresql-12.5.5.tgz
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   └── service.yaml
└── values.yaml

2 directories, 6 files
```

<br/>

```
$ helm install music-db \
  --namespace music \
  --create-namespace \
  --set global.postgresql.auth.username=user1,global.postgresql.auth.password=postgres1,global.postgresql.auth.database=postgresdb1,primary.persistence.enabled=false .
```

<br/>

```
$ kubectl get pods -n music
NAME                      READY   STATUS      RESTARTS       AGE
music-6d957c46bf-5w2g8    1/1     Running     2 (4m3s ago)   4m12s
music-db-postgresql-0     1/1     Running     0              4m12s
```

<br/>

```
// GET ADMIN_POSTGRES_PASSWORD
$ {
  export ADMIN_POSTGRES_PASSWORD=$(kubectl get -n music secret music-db-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
  echo ${POSTGRES_PASSWORD}
}
```

<br/>

```
// To connect to your database as admin
$ kubectl run my-db-postgresql-client --rm --tty -i --restart='Never' --namespace music --image docker.io/bitnami/postgresql:15.3.0-debian-11-r4 --env="PGPASSWORD=$ADMIN_POSTGRES_PASSWORD" \
      --command -- psql --host music-db-postgresql -U postgres -d postgres -p 5432
```

<br/>

```
// GET USER POSTGRES_PASSWORD
$ {
  export USER_POSTGRES_PASSWORD=$(kubectl get -n music secret music-db-postgresql -o jsonpath="{.data.password}" | base64 -d)
  echo ${POSTGRES_PASSWORD}
}
```

<br/>

```
// To connect to your database as user1
$ kubectl run my-db-postgresql-client --rm --tty -i --restart='Never' --namespace music --image docker.io/bitnami/postgresql:15.3.0-debian-11-r4 --env="PGPASSWORD=$USER_POSTGRES_PASSWORD" \
      --command -- psql --host music-db-postgresql -U user1 -d postgresdb1 -p 5432
```

<br/>

```
// To connect to your database from outside the cluster
$ kubectl port-forward --namespace music svc/music-db-postgresql 5432:5432 &
    PGPASSWORD="$USER_POSTGRES_PASSWORD" psql --host 127.0.0.1 -U user1 -d postgresdb1 -p 5432
```

<br/>

```
$ kubectl get services -n music
NAME                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
music                    ClusterIP   10.109.139.154   <none>        8080/TCP   83s
music-db-postgresql      ClusterIP   10.106.198.189   <none>        5432/TCP   83s
music-db-postgresql-hl   ClusterIP   None             <none>        5432/TCP   83s
```

<br/>

```
$ kubectl port-forward -n music service/music 8080:8080
```

<br/>

```
$ curl localhost:8080/song | jq
[
  {
    "id": 1,
    "artist": "DT",
    "name": "Quiero Munchies"
  },
  {
    "id": 2,
    "artist": "Lin-Manuel Miranda",
    "name": "We Don't Talk About Bruno"
  },
  {
    "id": 3,
    "artist": "Imagination",
    "name": "Just An Illusion"
  },
  {
    "id": 4,
    "artist": "Txarango",
    "name": "Tanca Els Ulls"
  },
  {
    "id": 5,
    "artist": "Halsey",
    "name": "Could Have Been Me"
  }
]
```

<br/>

### [OK!] 5.7 Triggering a Rolling Update Automatically

<br/>

**Делаю:**  
27.05.2023

<br/>

```
$ cd ~/tmp
$ mkdir -p greetings/templates
$ cd greetings
```

<br/>

```yaml
$ cat > templates/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name}}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name}}
    {{- if .Chart.AppVersion }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name}}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name}}
    spec:
      containers:
          - image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion}}"
            imagePullPolicy: {{ .Values.image.pullPolicy }}
            securityContext:
              {{- toYaml .Values.securityContext | nindent 14 }}
            name: {{ .Chart.Name}}
            ports:
              - containerPort: {{ .Values.image.containerPort }}
                name: http
                protocol: TCP
            env:
              - name: GREETING
                valueFrom:
                  configMapKeyRef:
                    name: {{ .Values.configmap.name}}
                    key: greeting
EOF
```

<br/>

```yaml
$ cat > templates/service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/name: {{ .Chart.Name }}
  name: {{ .Chart.Name }}
spec:
  ports:
    - name: http
      port: {{ .Values.image.containerPort }}
      targetPort: {{ .Values.image.containerPort }}
  selector:
    app.kubernetes.io/name: {{ .Chart.Name }}
EOF
```

<br/>

```yaml
$ cat > templates/configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting-config
data:
  greeting: Aloha
EOF
```

<br/>

```yaml
$ cat > Chart.yaml << EOF
apiVersion: v2
name: greetings
description: A Helm chart for Greetings service

type: application
version: 0.1.0
appVersion: "1.0.0"
EOF
```

<br/>

```yaml
$ cat > values.yaml << EOF
image:
  repository: quay.io/gitops-cookbook/greetings
  tag: "1.0.0"
  pullPolicy: Always
  containerPort: 8080

replicaCount: 1

configmap:
  name: greeting-config
EOF
```

<br/>

```
$ helm install greetings .
```

<br/>

```
$ kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
greetings-6df4d99d46-vzrj9   1/1     Running   0          29s
```

<br/>

```
$ kubectl port-forward service/greetings 8080:8080
```

```
$ curl localhost:8080
```

**returns**

```
Aloha Ada
```

<br/>

**Update the ConfigMap**

<br/>

```yaml
$ cat > templates/configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting-config
data:
  greeting: Hola
EOF
```

<br/>

```
$ helm upgrade greetings .
```

<br/>

```
$ kubectl port-forward service/greetings 8080:8080
```

```
$ curl localhost:8080
```

**returns**

```
Aloha Alexandra⏎
```

<br/>

There are no changes in the Deployment object, there is no restart of the pod;

<br/>

```yaml
$ cat > templates/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name}}
  labels:
    app.kubernetes.io/name: {{ .Chart.Name}}
    {{- if .Chart.AppVersion }}
    app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
    {{- end }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ .Chart.Name}}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ .Chart.Name}}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      containers:
          - image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion}}"
            imagePullPolicy: {{ .Values.image.pullPolicy }}
            securityContext:
              {{- toYaml .Values.securityContext | nindent 14 }}
            name: {{ .Chart.Name}}
            ports:
              - containerPort: {{ .Values.image.containerPort }}
                name: http
                protocol: TCP
            env:
              - name: GREETING
                valueFrom:
                  configMapKeyRef:
                    name: {{ .Values.configmap.name}}
                    key: greeting
EOF
```

<br/>

```yaml
$ cat > templates/configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting-config
data:
  greeting: Namaste
EOF
```

<br/>

```
$ helm upgrade greetings .
```

<br/>

```
$ kubectl port-forward service/greetings 8080:8080
```

<br/>

```
$ curl localhost:8080
```

<br/>

**returns**

```
Namaste Ada⏎
```

<br/>

```
$ kubectl describe pod greetings-bd8c9c4df-59xrj
```

<br/>

```
***
Annotations:      checksum/config:
***
```

<br/>

```yaml
$ cat > templates/configmap.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: greeting-config
data:
  greeting: Привет!
EOF
```

<br/>

```
$ helm upgrade greetings .
```

<br/>

```
$ kubectl port-forward service/greetings 8080:8080
```

<br/>

```
// Нужно подождать перестартовки пода
$ curl localhost:8080
```

<br/>

```
Привет! Ada⏎
```
