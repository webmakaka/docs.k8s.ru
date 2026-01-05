---
layout: page
title: Setting Up a Single Node Kubernetes
description: Setting Up a Single Node Kubernetes
keywords: gitops, containers, kubernetes, setup, Setting Up a Single Node Kubernetes
permalink: /tools/containers/kubernetes/setup/single-node-kubernetes/
---

<br/>

# Setting Up a Single Node Kubernetes Environment

<br/>

**Делаю:**  
2026.01.04

<br/>

```
// Kubernetes требует отключения swap
# Временное отключение swap
# sudo swapoff -a

# Постоянное отключение - закомментировать строку с swap в /etc/fstab
# sudo sed -i '/swap/s/^/#/' /etc/fstab
```

<br/>

```
// 1. Установить containerd
$ sudo apt-get update
$ sudo apt-get install -y containerd

// 2. Настроить containerd
$ sudo mkdir -p /etc/containerd
$ containerd config default | sudo tee /etc/containerd/config.toml

// 3. Включить SystemdCgroup (важно для k8s)
$ sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/g' /etc/containerd/config.toml

// 4. Перезапустить containerd
$ sudo systemctl restart containerd
$ sudo systemctl enable containerd

// 5. Проверить статус
$ sudo systemctl status containerd
```

<br/>

### Setup kubernetes repo and install components

```
$ sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

<br/>

```
// Создаем директорию для ключей если её нет
$ sudo mkdir -p /etc/apt/keyrings
```

<br/>

```
// Скачиваем и добавляем ключ
$ curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg
```

<br/>

```
// Добавляем репозиторий
$ echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

<br/>

```
$ sudo apt-get update
$ sudo apt-get install -y kubelet kubeadm kubectl
$ sudo apt-mark hold kubelet kubeadm kubectl
```

<br/>

```
$ sudo apt install -y net-tools
```

<br/>

```
$ ifconfig
***
enp0s8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.56.12  netmask 255.255.255.0  broadcast 192.168.56.255
        ether 08:00:27:31:f3:b1  txqueuelen 1000  (Ethernet)
        RX packets 1066  bytes 86689 (86.6 KB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 1051  bytes 165332 (165.3 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
***
```

<br/>

```
$ echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

<br/>

```
$ sudo sysctl -p
```

<br/>

```
$ sudo kubeadm init --apiserver-advertise-address xxx.xxx.xxx.xxx --pod-network-cidr=192.168.0.0/16
```

<br/>

```
// Обычно процесс kubeadm init занимает 3-10 минут в зависимости от скорости интернета и железа.
$ sudo kubeadm init \
  --apiserver-advertise-address=192.168.56.12 \
  --pod-network-cidr=192.168.0.0/16
```

<br/>

```
$ sudo systemctl status kubelet
```

<br/>

```
$ mkdir -p $HOME/.kube
$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

<br/>

```
$ kubectl config get-contexts
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin
```

<br/>

```
$ kubectl get nodes
NAME                STATUS     ROLES           AGE   VERSION
marley-virtualbox   NotReady   control-plane   50s   v1.30.14
```

<br/>

### Установка Calico

```
// 1. Установить полный манифест Calico
$ curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml -O

// 2. Проверить что CIDR совпадает (192.168.0.0/16)
$ grep -n "192.168.0.0/16" calico.yaml

// 3. Применить
$ kubectl apply -f calico.yaml
```

<br/>

```
$ kubectl get pods -n kube-system -l k8s-app=calico-node
$ kubectl get pods -n kube-system -l k8s-app=calico-kube-controllers
```

<br/>

```
$ kubectl get nodes
NAME                STATUS   ROLES           AGE   VERSION
marley-virtualbox   Ready    control-plane   10m   v1.30.14
```

<br/>

```
// Разрешить на control-plane разворачивать pod
$ kubectl taint nodes --all node-role.kubernetes.io/control-plane:NoSchedule-
```

<br/>

```
$ kubectl run test-busybox --image=busybox --restart=Never -- sleep 3600
```

<br/>

```
$ kubectl get pods
```

<br/>

```
$ kubectl delete pod test-busybox
```
