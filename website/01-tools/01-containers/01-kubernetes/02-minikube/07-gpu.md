---
layout: page
title: Запуск и останов minikube в ubuntu 22.04 с GPU
description: Запуск и останов minikube в ubuntu 22.04 с GPU
keywords: ubuntu, containers, kubernetes, minikube, run, gpu
permalink: /tools/containers/kubernetes/minikube/run/gpu/
---

# Запуск и останов minikube в ubuntu 22.04 с GPU

<br/>

**Делаю:**  
2026.04.09

<br/>

Записываю по памяти, то что заработало в конце концов. Наверное неправильно настраиваю, первый опыт. Постепенно улучшу.

<br/>

```
// OK!
$ nvidia-smi
```

<br/>

### Инсталляция nvidia-container-toolkit

```shell
$ curl -fsSL https://nvidia.github.io/nvidia-docker/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

$ curl -s -L https://nvidia.github.io/nvidia-docker/debian11/nvidia-docker.list

$ sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g'

$ sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

$ sudo apt-get update
$ sudo apt-get install -y nvidia-container-toolkit
```

<br/>

```shell
$ dpkg -l | grep nvidia-container-toolkit
ii nvidia-container-toolkit 1.13.5-1 amd64 NVIDIA Container toolkit
ii nvidia-container-toolkit-base 1.13.5-1 amd64 NVIDIA Container Toolkit Base
```

<br/>

```shell
$ sudo nvidia-ctk runtime configure --runtime=docker --config=/etc/docker/daemon.json
```

<br/>

```shell
$ cat /etc/docker/daemon.json
{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    }
}
```

<br/>

```shell
$ sudo systemctl restart docker
```

<br/>

```shell
$ nvidia-container-cli --version
cli-version: 1.13.5
lib-version: 1.13.5
build date: 2023-07-18T11:37+00:00
build revision: 66607bd046341f7aad7de80a9f022f122d1f2fce
build compiler: x86_64-linux-gnu-gcc-8 8.3.0
build platform: x86_64
```

<br/>

### Запуск в docker nvidia-smi

```shell
$ docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
NVIDIA-SMI couldn't find libnvidia-ml.so library in your system. Please make sure that the NVIDIA Display Driver is properly installed and present in your system.
Please also try adding directory that contains libnvidia-ml.so to your system PATH.
```

<br/>

```
$ docker run --rm --gpus all -it nvidia/cuda:11.8.0-runtime-ubuntu22.04 bash

# ldconfig

// OK!
# nvidia-smi
```

<br/>

**entrypoint.sh**

```shell
#!/bin/bash
# Обновляем кэш библиотек перед запуском
ldconfig
# Выполняем переданную команду
exec "$@"
```

<br/>

**Dockerfile**

```
FROM nvidia/cuda:11.8.0-runtime-ubuntu22.04

# Копируем entrypoint скрипт

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

# Устанавливаем entrypoint

ENTRYPOINT ["/entrypoint.sh"]

# Команда по умолчанию

CMD ["nvidia-smi"]
```

<br/>

```
$ docker build -t my-cuda:11.8.0 .

// OK!
$ docker run --rm --gpus all my-cuda:11.8.0 nvidia-smi
```

<br/>

### Запуск MiniKube

```shell
$ minikube start \
 --driver=docker \
 --memory=8g \
 --cpus=4 \
 --container-runtime=docker \
 --gpus all
```

<br/>

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nvidia-device-plugin-daemonset
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: nvidia-device-plugin-ds
  template:
    metadata:
      labels:
        name: nvidia-device-plugin-ds
    spec:
      containers:
      - image: nvcr.io/nvidia/k8s-device-plugin:v0.17.1
        name: nvidia-device-plugin-ctr
        securityContext:
          privileged: true
        volumeMounts:
          - name: device-plugin
            mountPath: /var/lib/kubelet/device-plugins
      volumes:
        - name: device-plugin
          hostPath:
            path: /var/lib/kubelet/device-plugins
EOF
```

<br/>

```shell
$ minikube ssh
$ sudo ldconfig /usr/lib/x86_64-linux-gnu
```

<br/>

```shell
// ХЗ нужно или нет
$ minikube ssh
# Теперь вы внутри minikube
docker@minikube:~$ sudo sed -i 's/mode = "auto"/mode = "legacy"/' /etc/nvidia-container-runtime/config.toml
docker@minikube:~$ sudo systemctl restart docker
docker@minikube:~$ exit
```

<br/>

```shell
// OK!
$ kubectl run gpu-test --rm -it --restart=Never --image=nvidia/cuda:12.2.2-base-ubuntu22.04 -- nvidia-smi
Thu Apr  9 01:20:35 2026
+---------------------------------------------------------------------------------------+
| NVIDIA-SMI 535.288.01             Driver Version: 535.288.01   CUDA Version: 12.2     |
|-----------------------------------------+----------------------+----------------------+
| GPU  Name                 Persistence-M | Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp   Perf          Pwr:Usage/Cap |         Memory-Usage | GPU-Util  Compute M. |
|                                         |                      |               MIG M. |
|=========================================+======================+======================|
|   0  NVIDIA GeForce GTX 1650        Off | 00000000:01:00.0  On |                  N/A |
| 28%   58C    P5              N/A /  75W |    832MiB /  4096MiB |     35%      Default |
|                                         |                      |                  N/A |
+-----------------------------------------+----------------------+----------------------+

+---------------------------------------------------------------------------------------+
| Processes:                                                                            |
|  GPU   GI   CI        PID   Type   Process name                            GPU Memory |
|        ID   ID                                                             Usage      |
|=======================================================================================|
+---------------------------------------------------------------------------------------+
pod "gpu-test" deleted
```

<br/>

```shell
// 1 GPU на узле minikube
$ kubectl get nodes -o json | jq -r '.items[] | "\(.metadata.name): \(.status.allocatable."nvidia.com/gpu" // 0)"'
minikube: 1
```

<br/>

### Запуск модели qwen в MiniKube

<br/>

**Минут 6 устанавливаетя.**

```yaml
$ cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: qwen-service
spec:
  type: NodePort
  selector:
    app: qwen
  ports:
    - port: 11434
      targetPort: 11434
      nodePort: 30434
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qwen-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: qwen
  template:
    metadata:
      labels:
        app: qwen
    spec:
      containers:
        - name: ollama
          image: ollama/ollama:latest
          resources:
            limits:
              nvidia.com/gpu: 1 # Тот самый лимит, который мы настраивали!
          ports:
            - containerPort: 11434
          env:
            - name: OLLAMA_MODELS
              value: '/root/.ollama'
EOF
```

<br/>

```shell
$ kubectl get pods
NAME                              READY   STATUS    RESTARTS   AGE
qwen-deployment-d57b558dd-w9vwd   1/1     Running   0          6m40s
```

<br/>

```shell
$ kubectl exec -it $(kubectl get pod -l app=qwen -o name) -- ollama run qwen2.5:1.5b
```

<br/>

```shell
// на хосте смотрю загрузку видеокарты
$ watch -n 0.5 nvidia-smi
```

<br/>

```shell
$ minikube ip
```

<br/>

```shell
$ curl http://192.168.49.2:30434/api/generate -d '{
  "model": "qwen2.5:1.5b",
  "prompt": "Привет! Ты работаешь на моей GTX 1650 внутри Kubernetes?",
  "stream": false
}' | jq -r '.response'
```

**response:**

```
Да, я работаю в вашем кластере Kubernetes с NVIDIA GeForce GTX 1650 видеокартой для выполнения задачи. Настройка и управление рабочими нагрузками на GPU у меня не вызывает проблем, так как это мой основной функционал в текущей роли
```

<br/>

### Улучшения

```shell
$ kubectl delete ds nvidia-device-plugin-daemonset -n kube-system
$ minikube addons enable nvidia-device-plugin
```

<br/>

```shell
$ kubectl get pods -n kube-system
NAME                                   READY   STATUS    RESTARTS      AGE
coredns-7d764666f9-mtmgh               1/1     Running   1 (12m ago)   19m
etcd-minikube                          1/1     Running   1 (12m ago)   19m
kube-apiserver-minikube                1/1     Running   1 (12m ago)   19m
kube-controller-manager-minikube       1/1     Running   1 (12m ago)   19m
kube-proxy-n8ww2                       1/1     Running   1 (12m ago)   19m
kube-scheduler-minikube                1/1     Running   1 (12m ago)   19m
nvidia-device-plugin-daemonset-l2tlz   1/1     Running   0             4m38s
storage-provisioner                    1/1     Running   3 (12m ago)   19m
```

<br/>

```shell
$ kubectl run gpu-test --rm -it --restart=Never --image=nvidia/cuda:12.2.2-base-ubuntu22.04 -- sh -c "nvidia-smi --query-gpu=memory.used,memory.total --format=csv,noheader,nounits | awk -F', ' '{print \$1 \" / \" \$2 \" MiB\"}'"
2283 / 4096 MiB
```
