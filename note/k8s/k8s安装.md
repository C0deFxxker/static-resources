# 安装前准备
* Linux系统
* 2g内存
* 2核CPU
* 关闭交换空间
* 集群间网络通畅，并且需要保证集群中每台机器的主机名、Mac地址以及 product_uuid 是唯一的。

# 需要占用的端口号

## Master节点

协议 | 方向 | 端口范围 | 用途 | 被其它服务使用
---|---|---|---|---
TCP | 输入 | 6443(可调整) | Kubernetes API server | 所有
TCP | 输入 | 2379-2380 | etcd服务客户端API | kube-apiserver, etcd
TCP | 输入 | 10250 | Kubelet API | 自己, Control plane
TCP | 输入 | 10251 | kube-scheduler | 自己
TCP | 输入 | 10252 | kube-controller-manager | 自己

## Worker节点

协议 | 方向 | 端口范围 | 用途 | 被其它服务使用
---|---|---|---|---
TCP | 输入 | 10250 | Kubelet API | 自己, Control plane
TCP | 输入 | 30000-32767 | NodePort Services | 所有

# CRI 安装 (Container Runtime Interface)
必须预先安装其中一种CRI。
自 v1.14.0 版本开始，kubeadm会自动检测本机上的CRI，kubeadm 通过确认指定路径下是否存在文件来达到这个目的，检测路径及CRI对应关系如下：

Runtime | Domain Socket
---|---
Docker | /var/run/docker.sock
containerd | /run/containerd/containerd.sock
CRI-O | /var/run/crio/crio.sock


# 安装 kubeadm, kubelet, kubectl
国内环境使用阿里yum源进行安装，并按照需要把 SELinux 改为放任模式或直接关掉它。

```sh
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
       http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

# Set SELinux in permissive mode (effectively disabling it)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet

# RHEL/CentOS 7 版本需要保证 net.bridge.bridge-nf-call-iptables 和 net.bridge.bridge-nf-call-ip6tables 配置都设为 1
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system

# 关闭交换空间
swapoff -a

# 确保 br_netfilter 模块已经被加载
lsmod | grep br_netfilter
# 如果尚未加载，则执行如下命令
#modprobe br_netfilter
```

# 使用 kubeadm 初始化
使用kubeadm工具进行初始化，但在国内会因为下载不到 k8s.gcr.io 的镜像而报错，可以先下载国内的镜像，然后再用 "docker tag" 指令把docker镜像名称重命名。指令如下：
```sh
# 当前版本是：v1.14.2
docker pull mirrorgooglecontainers/kube-apiserver:v1.14.2
docker pull mirrorgooglecontainers/kube-controller-manager:v1.14.2
docker pull mirrorgooglecontainers/kube-scheduler:v1.14.2
docker pull mirrorgooglecontainers/kube-proxy:v1.14.2
docker pull mirrorgooglecontainers/pause:3.1
docker pull mirrorgooglecontainers/etcd:3.3.10
docker pull coredns/coredns:1.3.1

docker tag docker.io/mirrorgooglecontainers/kube-apiserver:v1.14.2 k8s.gcr.io/kube-apiserver:v1.14.2
docker tag docker.io/mirrorgooglecontainers/kube-controller-manager:v1.14.2 k8s.gcr.io/kube-controller-manager:v1.14.2
docker tag docker.io/mirrorgooglecontainers/kube-scheduler:v1.14.2 k8s.gcr.io/kube-scheduler:v1.14.2
docker tag docker.io/mirrorgooglecontainers/kube-proxy:v1.14.2 k8s.gcr.io/kube-proxy:v1.14.2
docker tag docker.io/mirrorgooglecontainers/pause:3.1 k8s.gcr.io/pause:3.1
docker tag docker.io/mirrorgooglecontainers/etcd:3.3.10 k8s.gcr.io/etcd:3.3.10
docker tag docker.io/coredns/coredns:1.3.1 k8s.gcr.io/coredns:1.3.1

# 指定虚拟网络分片IP段为10.244.0.0/16，以及节点名称为centos1
kubeadm init --pod-network-cidr=10.244.0.0/16 --node-name centos1
```

当终端输出如下文字时，表示当前节点的k8s启动完毕了:

```
...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.104:6443 --token pb4sgb.kytix3aysrifdm5b \
    --discovery-token-ca-cert-hash sha256:f2899e83dadac9450085866301146a81dba924a877fad40d76922f3db6851d1a
```

按照输出的提示，执行shell命令：
```sh
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

还需要选择一种虚拟网络工具：
```sh
# 使用flannel工具，利用docker overlay模式实现虚拟网络
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/62e44c867a2846fefb68bd5f178daf4da3095ccb/Documentation/kube-flannel.yml
```

默认情况下，Master节点是不能创建pod的，可以使用如下命令来解除这个限制：
```sh
kubectl taint nodes --all node-role.kubernetes.io/master-
```

# 注册Worker节点
在上一节创建 master 节点的kubelet成功时会有类似如下输出：
```
...

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.1.104:6443 --token pb4sgb.kytix3aysrifdm5b \
    --discovery-token-ca-cert-hash sha256:f2899e83dadac9450085866301146a81dba924a877fad40d76922f3db6851d1a
```

最后 "kubeadm join" 指令已经放到worker节点上执行即可，这个指令包含了节点加入集群时认证用的token以及token对应的签名。

若你没有在创建时保存token签名信息，你可以在master节点上使用如下指令查询已有token：
```sh
kubeadm token list
# 输出类似如下
TOKEN                    TTL  EXPIRES              USAGES           DESCRIPTION            EXTRA GROUPS
8ewj1p.9r9hcjoqgajrj4gi  23h  2018-06-12T02:51:28Z authentication,  The default bootstrap  system:
                                                   signing          token generated by     bootstrappers:
                                                                    'kubeadm init'.        kubeadm:
                                                                                           default-node-token
```

默认情况下创建的token签名在24小时内有效，这个token只会在worker节点第一次加入集群时要求认证，若token已经过时了，可以使用如下命令创建新token：
```sh
kubeadm token create
# 输出类似如下
5didvk.d09sbcov8ph2amjw
```

然后通过如下指令获取token对应的签名：
```sh
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
# 输出类似如下
8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78
```

然后在worker节点上把token以及签名作为参数传入 "kubeadm join" 指令中进行集群注册：
```sh
kubeadm join 192.168.1.104:6443 --token 5didvk.d09sbcov8ph2amjw \
    --discovery-token-ca-cert-hash sha256:8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78
```