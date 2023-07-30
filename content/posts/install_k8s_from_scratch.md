---
title: "使用 kubeadm 搭建一个 Kubernetes 集群"
date: 2023-07-31T00:41:18+08:00
draft: false
tags: ["tech", "clound native", "kubernetes"]
categories: ["Kubernetes"]
---



# 准备阶段

- 三台机器(实际上一台也可以, 因为这里要搭建集群), 这里我使用的是 ubuntu 22.10
- 每台机器至少 2G 内存
- control-plane(master) 至少 2CPU
- 彼此之间良好的网络联通性

这里我使用了 vagrant

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.synced_folder "/Users/bytedance/project", "/home/vagrant/project"

  num_instance = 3
  (1..num_instance).each do |i|
    config.vm.define "node#{i}" do |node|
      node.vm.box = "bento/ubuntu-22.04"
      node.vm.hostname = "node#{i}"
      ip = "172.17.8.#{i+100}"
      node.vm.network "private_network", ip: ip
      # node.vm.provider "virtualbox" do |vb|
      #   vb.memory = "2048"
      #   vb.cpus = 2
      #   vb.name = "node#{i}"
      # end
      # node.vm.provision "shell", path: "install.sh", args: [i, ip, $etcd_cluster]
    end
  end
end
```

也可以自行创建三台机器并设置他们的 hostname

```bash
# 在不同的机器执行
sudo hostnamectl set-hostname "node1"
sudo hostnamectl set-hostname "node2"
sudo hostnamectl set-hostname "node2"
```



> 以下命令直到出现显式的说分开操作, 均在所有的节点执行



- 在所有的机器上的 `/etc/hosts` 添加以下条目

```bash
# 根据你自己的 ip 来设置
172.17.8.101 node1
172.17.8.102 node2
172.17.8.103 node3
```

- 禁用 swap 分区

  ```bash
  sudo swapoff -a
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
  
  # 运行完成后, 去 /etc/fstab 检查是否有遗漏, 将所有带有 swap 的行全部注释掉
  ```

  

# 容器运行时

## 预置条件

### 打开 ip_forwad 并让 iptables 看到 bridged 流量

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

验证 br_netfilter, overlay 内核模块

```bash
lsmod | grep br_netfilter
lsmod | grep overlay
```

验证 `net.bridge.bridge-nf-call-iptables`, `net.bridge.bridge-nf-call-ip6tables` ,`net.ipv4.ip_forward`

```bash
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```

## [安装 containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md)

### 安装 containerd

下载[containerd-<VERSION>-<OS>-<ARCH>.tar.gz](https://github.com/containerd/containerd/releases), 解压到 `/usr/local`

```bash
sudo tar Cxzvf /usr/local containerd-1.6.2-linux-amd64.tar.gz
```

使用 systemd 管理, 将以下信息放置到 `/usr/local/lib/systemd/system/containerd.service`

```toml
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
#uncomment to enable the experimental sbservice (sandboxed) version of containerd/cri integration
#Environment="ENABLE_CRI_SANDBOXES=sandboxed"
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

执行

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now containerd
```

### 安装 runc

下载 [runc.<ARCH>](https://github.com/opencontainers/runc/releases)并安装到 `/usr/local/sbin/runc`

```bash
install -m 755 runc.amd64 /usr/local/sbin/runc
```

下载 [cni-plugins-<OS>-<ARCH>-<VERSION>.tgz](https://github.com/containernetworking/plugins/releases) 解压到 `/opt/cni/bin`

```bash
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.1.1.tgz
```



>  重要: 切换 containerd 的 cgroup 驱动到 systemd

```bash
sudo mkdir -p /etc/containerd/
containerd config default | sudo tee /etc/containerd/config.toml >/dev/null 2>&1
```

编辑: `/etc/containerd/config.toml` , 将 SystemdCgroup 设置为 true

![containerd-systemd_cgroup](/images/containerd-systemd_cgroup.png)

重启 containerd

```bash
sudo systemctl restart containerd
```



# 安装 kubeadm、kubelet、kubectl

- `kubeadm`：用来初始化集群的指令。
- `kubelet`：在集群中的每个节点上用来启动 Pod 和容器等。
- `kubectl`：用来与集群通信的命令行工具。

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

curl -fsSL https://dl.k8s.io/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-archive-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```



> 以下操作分机器完成

# 使用 kubeadm 创建集群

## 初始化控制平面节点

```bash
# --control-plane-endpoint 为控制面端点
# --pod-network-cidr 指定 pod 的网络地址, 可以不写, 但是用来做验证比较方便
# --apiserver-advertise-address=172.17.8.101 不写的话, 使用默认网关, 使用 vagrant 默认网关不是这个, 所以写上会比较好一些
sudo kubeadm init --control-plane-endpoint=node1 --pod-network-cidr=172.18.0.0/16 --apiserver-advertise-address=172.17.8.101
```



出现一下字样表示成功

```
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a Pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  /docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```



使非 root 用户可以运行 kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



将其他 worker 节点加入当前集群, 该命令是 master 节点部署成功后输出的

```bash
kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```



加入成功后可以在 master 节点看到 nodes 的信息, 此时为 NotReady

```bash
vagrant@node1:~$ kubectl get nodes
NAME    STATUS   		ROLES           AGE    VERSION
node1   NotReady    control-plane   162m   v1.27.4
node2   NotReady    <none>          147m   v1.27.4
node3   NotReady    <none>          154m   v1.27.4
```



## 安装网络插件

一些可用的 add-on: https://kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/

其他一些可以很简单的安装比如 

- Flannel

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

- Calico

```bash
curl https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml -O
kubectl apply -f calico.yaml
```



Cilium 目前我安装的时候会比其他的麻烦一些

prerequisite: go

```bash
sudo apt install golang
```

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
GOOS=$(go env GOOS)
GOARCH=$(go env GOARCH)
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-${GOOS}-${GOARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-${GOOS}-${GOARCH}.tar.gz.sha256sum
sudo tar -C /usr/local/bin -xzvf cilium-${GOOS}-${GOARCH}.tar.gz
rm cilium-${GOOS}-${GOARCH}.tar.gz{,.sha256sum}
```

```bash
# version 也可以忽略
cilium install --version 1.14.0
```



安装完成后, nodes 的状态已经是 ready 了

```bash
vagrant@node1:~$ kubectl get nodes
NAME    STATUS   ROLES           AGE    VERSION
node1   Ready    control-plane   162m   v1.27.4
node2   Ready    <none>          147m   v1.27.4
node3   Ready    <none>          154m   v1.27.4
```

