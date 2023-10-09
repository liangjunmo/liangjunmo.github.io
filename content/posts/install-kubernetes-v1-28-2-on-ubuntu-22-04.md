---
title: "Install Kubernetes V1.28.2 on Ubuntu 22.04"
slug: install-kubernetes-v1.28.2-on-ubuntu-22.04
type: post
date: 2023-10-08
lastMod: 2023-10-08
showSummary: true
summary: Kubernetes v1.28.2 安装教程
tags:
- kubernetes
---

# Kubernetes v1.28.2 安装教程

# 环境
* Ubuntu 22.04
* Docker 23.0.1
* Kubernetes v1.28.2

# 安装 Docker

https://docs.docker.com/engine/install/ubuntu/

## 卸载

```
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

## apt 安装

```
sudo apt-get update

sudo apt-get install ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl status docker

sudo docker info
```

## deb 手动安装

https://download.docker.com/linux/ubuntu/dists/jammy/pool/stable/amd64/

```
curl -L --remote-name-all https://download.docker.com/linux/ubuntu/dists/jammy/pool/stable/amd64/{containerd.io_1.6.24-1_amd64.deb,docker-ce_24.0.6-1~ubuntu.22.04~jammy_amd64.deb,docker-ce-cli_24.0.6-1~ubuntu.22.04~jammy_amd64.deb,docker-buildx-plugin_0.11.2-1~ubuntu.22.04~jammy_amd64.deb,docker-compose-plugin_2.21.0-1~ubuntu.22.04~jammy_amd64.deb}

sudo dpkg -i ./containerd.io_1.6.24-1_amd64.deb \
  ./docker-ce_24.0.6-1~ubuntu.22.04~jammy_amd64.deb \
  ./docker-ce-cli_24.0.6-1~ubuntu.22.04~jammy_amd64.deb \
  ./docker-buildx-plugin_0.11.2-1~ubuntu.22.04~jammy_amd64.deb \
  ./docker-compose-plugin_2.21.0-1~ubuntu.22.04~jammy_amd64.deb

sudo systemctl status docker

sudo docker info
```

## 修改 Cgroup Driver 为 systemd

```
sudo docker info
```

如果输出包含下面的内容，则不需要修改：

```
Cgroup Driver: systemd
```

否则编辑 `/etc/docker/daemon.json` 文件：

```
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

重启 docker:

```
sudo systemctl restart docker
```

# 安装 Kubernetes

## 安装 kubectl

https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux

```
curl -L --remote-name-all https://dl.k8s.io/release/v1.28.2/bin/linux/amd64/{kubectl,kubectl.sha256}

echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

kubectl version --client --output=yaml
```

## 安装 kubeadm、kubelet

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/#installing-kubeadm-kubelet-and-kubectl

### 安装 CNI 插件

```
sudo mkdir -p /opt/cni/bin

curl -LO https://github.com/containernetworking/plugins/releases/download/v1.3.0/cni-plugins-linux-amd64-v1.3.0.tgz

sudo tar -xzf cni-plugins-linux-amd64-v1.3.0.tgz -C /opt/cni/bin
```

### 安装 crictl

```
curl -LO https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.28.0/crictl-v1.28.0-linux-amd64.tar.gz

sudo tar -xzf crictl-v1.28.0-linux-amd64.tar.gz -C /usr/local/bin
```

### 安装 kubeadm

```
curl -L --remote-name-all https://dl.k8s.io/release/v1.28.2/bin/linux/amd64/{kubeadm,kubeadm.sha256}

echo "$(cat kubeadm.sha256) kubeadm" | sha256sum --check

sudo install -o root -g root -m 0755 kubeadm /usr/local/bin/kubeadm

kubeadm version
```

### 安装 kubelet

```
curl -L --remote-name-all https://dl.k8s.io/release/v1.28.2/bin/linux/amd64/{kubelet,kubelet.sha256}

echo "$(cat kubelet.sha256) kubelet" | sha256sum --check

sudo install -o root -g root -m 0755 kubelet /usr/local/bin/kubelet

kubelet --version

curl -LO https://raw.githubusercontent.com/kubernetes/release/v0.15.1/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service

sed -i "s:/usr/bin:/usr/local/bin:g" kubelet.service

sudo cp kubelet.service /etc/systemd/system/kubelet.service

curl -LO https://raw.githubusercontent.com/kubernetes/release/v0.15.1/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf

sed -i "s:/usr/bin:/usr/local/bin:g" 10-kubeadm.conf

sudo mkdir -p /etc/systemd/system/kubelet.service.d

sudo cp 10-kubeadm.conf /etc/systemd/system/kubelet.service.d/10-kubeadm.conf

sudo systemctl enable --now kubelet

# 现在 kubelet 还是启动失败，需要等使用 kubeadm 初始化后再重启
sudo systemctl status kubelet
```

## 配置 Cgroup Driver

> 在版本 1.22 及更高版本中，如果用户没有在 KubeletConfiguration 中设置 cgroupDriver 字段， kubeadm 会将它设置为默认值 systemd。

默认为 systemd，不用修改。

## 安装 cri-dockerd

https://github.com/Mirantis/cri-dockerd

```
curl -LO https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.4/cri-dockerd_0.3.4.3-0.ubuntu-jammy_amd64.deb

sudo dpkg -i cri-dockerd_0.3.4.3-0.ubuntu-jammy_amd64.deb

sudo systemctl status cri-dockerd
```

## 准备镜像

查看需要的镜像：

```
kubeadm config images list
```

镜像列表：

```
registry.k8s.io/kube-apiserver:v1.28.2
registry.k8s.io/kube-controller-manager:v1.28.2
registry.k8s.io/kube-scheduler:v1.28.2
registry.k8s.io/kube-proxy:v1.28.2
registry.k8s.io/pause:3.9
registry.k8s.io/etcd:3.5.9-0
registry.k8s.io/coredns/coredns:v1.10.1
```

pull 镜像：

```
for image in $(kubeadm config images list); do sudo docker pull $image; done
```

网络问题无法直接 pull 时，找一个挂了代理的机器 pull 镜像，然后导出镜像：

```
echo "sudo docker pull registry.k8s.io/kube-apiserver:v1.28.2
sudo docker pull registry.k8s.io/kube-controller-manager:v1.28.2
sudo docker pull registry.k8s.io/kube-scheduler:v1.28.2
sudo docker pull registry.k8s.io/kube-proxy:v1.28.2
sudo docker pull registry.k8s.io/pause:3.9
sudo docker pull registry.k8s.io/etcd:3.5.9-0
sudo docker pull registry.k8s.io/coredns/coredns:v1.10.1
sudo docker save registry.k8s.io/kube-apiserver:v1.28.2 registry.k8s.io/kube-controller-manager:v1.28.2 registry.k8s.io/kube-scheduler:v1.28.2 registry.k8s.io/kube-proxy:v1.28.2 registry.k8s.io/pause:3.9 registry.k8s.io/etcd:3.5.9-0 registry.k8s.io/coredns/coredns:v1.10.1 -o k8s_v1.28.2_image.tar" > k8s_v1.28.2_image.sh

sudo chmod u+x k8s_v1.28.2_image.sh

./k8s_v1.28.2_image.sh
```

把上面导出的文件 ssh 到要安装 kubernetes 的机器上，再执行：

```
sudo docker load -i k8s_v1.28.2_image.tar
```

## 禁用 swap

删除 swap 相关的内容，永久生效：

```
sudo sed -i "/swap/d" /etc/fstab
```

临时生效，重启失效：

```
sudo swapoff -a
```

## 使用 kubeadm 初始化 master 节点

```
sudo kubeadm init --cri-socket unix:///var/run/cri-dockerd.sock --kubernetes-version=v1.28.2 --pod-network-cidr 10.244.0.0/16 --service-cidr=10.96.0.0/12
```

> 如果执行失败或者其它原因想要重置，可以执行 `kubeadm reset --cri-socket /var/run/cri-dockerd.sock`.

输出：

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.6:6443 --token 96rc5z.6yew6ic8agjemadl \
	--discovery-token-ca-cert-hash sha256:8b2bdd1f399c3de176bb16be3858fc262f6cefcc2565ec9688fc696404ed2864
```

## 配置 flannel 网络

```
kubectl apply -f https://github.com/flannel-io/flannel/releases/download/v0.22.3/kube-flannel.yml
```

## 检查系统 pod 是否正常

```
kubectl get pods -n kube-system
```

pod 信息正常：

```
NAME                                        READY   STATUS    RESTARTS   AGE
coredns-5dd5756b68-6t2fs                    1/1     Running   0          15m
coredns-5dd5756b68-vg5nx                    1/1     Running   0          15m
etcd-liangjunmo-ubuntu                      1/1     Running   0          15m
kube-apiserver-liangjunmo-ubuntu            1/1     Running   0          15m
kube-controller-manager-liangjunmo-ubuntu   1/1     Running   0          15m
kube-proxy-nhjz7                            1/1     Running   0          15m
kube-scheduler-liangjunmo-ubuntu            1/1     Running   0          15m
```

## 部署 Nginx 进行测试

准备 nginx-deployment.yaml:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

应用：

```
kubectl apply -f nginx-deployment.yaml
```

查看 pod:

```
kubectl get pods
```

pod 状态：

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54b6f7ddf9-2v6jt   0/1     Pending   0          29s
nginx-deployment-54b6f7ddf9-mqdzv   0/1     Pending   0          29s
```

pod 一直在 pending，查看详情：

```
kubectl describe pod nginx-deployment-54b6f7ddf9-2v6jt
```

最后的内容为：

```
Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  40s   default-scheduler  0/1 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }. preemption: 0/1 nodes are available: 1 Preemption is not helpful for scheduling..
```

有 untolerated taint 无法容忍的污点 `node-role.kubernetes.io/control-plane`，需要去除污点。

> 当前节点为 master 节点，实际生产环境不应该让 pod 调度到 master 节点，这里为了测试先去除 master 节点上的 `node-role.kubernetes.io/control-plane` 污点.

先查看节点：

```
kubectl get nodes
```

节点信息：

```
NAME                STATUS   ROLES           AGE   VERSION
liangjunmo-ubuntu   Ready    control-plane   13m   v1.28.2
```

查看污点：

```
kubectl describe nodes liangjunmo-ubuntu | grep Taint
```

污点信息：

```
Taints:             node-role.kubernetes.io/control-plane:NoSchedule
```

去除污点：

```
kubectl taint nodes liangjunmo-ubuntu node-role.kubernetes.io/control-plane-
```

再次查看污点：

```
kubectl describe nodes liangjunmo-ubuntu | grep Taint
```

污点信息：

```
Taints:             <none>
```

再次查看 pod:

```
kubectl get pods
```

pod 状态正常：

```
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-54b6f7ddf9-2v6jt   1/1     Running   0          5m13s
nginx-deployment-54b6f7ddf9-mqdzv   1/1     Running   0          5m13s
```

# 加入集群

在每个 worker node 上都需要执行一遍上面的流程，只不过其中的**使用 kubeadm 初始化 master 节点**，改为执行 `kubeadm join` 操作。

> 例如上面使用 kubeadm init 初始化 master 节点后输出的：
> ```
> kubeadm join 192.168.1.6:6443 --token 96rc5z.6yew6ic8agjemadl \
> --discovery-token-ca-cert-hash sha256:8b2bdd1f399c3de176bb16be3858fc262f6cefcc2565ec9688fc696404ed2864
> ```

