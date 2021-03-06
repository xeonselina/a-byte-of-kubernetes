# 2.1.安装Master和Node

## 安装需看

* CentOS 16 64 位
* Master 4核4G \(Kubernetes 必须要2核以上\)
* Node 2核2G

## 安装 CentOS 源

Kubernetes国内镜像、下载安装包和拉取gcr.io镜像

> `https://blog.csdn.net/nklinsirui/article/details/80581286`

```bash
#官方文档源，不推荐使用
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF

#推荐使用这个源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=0
EOF
```

## 安装 kubectl、kubeadm、docker

```bash
yum install -y docker kubelet kubeadm kubectl kubernetes-cni
systemctl enable docker && systemctl enable kubelet  && systemctl start kubelet && systemctl start docker
```

## 配置 docker 阿里云镜像

```bash
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://yourcode.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload  && systemctl restart docker
```

登录阿里云镜像服务，可以得到 yourcode

### 禁用防火墙

```text
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

### 禁用Selinux

```text
# 将 SELinux 设置为 permissive 模式(将其禁用)
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 禁用firewalld

```text
systemctl disable firewalld && systemctl stop firewalld
```

### 禁用swapoff

```text
swapoff -a
sed -i -e /swap/d /etc/fstab
查看swap状态
free -m
```

### Ubuntu 源 留个记录

```text
deb http://mirrors.ustc.edu.cn/kubernetes/apt kubernetes-xenial main
```

### 查看下 Systemd  \(CentOS 应该会有这两个文件\)

```text
vim /usr/lib/systemd/system/kubelet.service
vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf
```

### 编写 kubeadm.yml \(后面没用到\)

```text
#查看默认的配置文件

kubeadm config print init-defaults

#kubeadm 1.14 配置文件'

apiVersion: kubeadm.k8s.io/v1beta1
kind: ClusterConfiguration
controllerManager:
    extraArgs:
        horizontal-pod-autoscaler-use-rest-clients: "true"
        horizontal-pod-autoscaler-sync-period: "10s"
        node-monitor-grace-period: "10s"
apiServer:
    extraArgs:
        runtime-config: "api/all=true"
kubernetesVersion: "stable-1.14"
imageRepository: registry.aliyuncs.com/google_containers


相关文档：https://godoc.org/k8s.io/kubernetes/cmd/kubeadm/app/apis/kubeadm/v1beta1

备注这个针对版本号为1.14的
```

### 查看需要安装的容器

```text
kubeadm config images list
# 后面没用到
kubeadm config images pull --config ~/kubeadm.yml
```

### 查询内网网段

```text
我的阿里云的网段是172.26.0.0/20
主私网IP地址：
172.26.9.41
IPv4 私网网段：
172.26.0.0/20

用 route 命令查看
https://blog.csdn.net/aerchi/article/details/39396423
```

### 预准备配置网络

官方安装网络插件文档

> `https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/`

* 配置方法
  * kubeadm 启动的时候要带上 --pod-network-cidr=172.26.0.0/20  带上这个参数
  * ~~修改 calico.yaml~~ 放弃使用原因如下

**kubeadm 的** _**--config**_ **和** _**--pod-network-cidr**_ **不能同时使用，但是我没找到如果在 kubeadm.yaml 里面如何配置** _**--pod-network-cidr**_**所以放弃了用上面编写的 kubeadm.yaml**

### 下载 calico pod 配置文件

```text
https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml

https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml
```

#### 已经弃用修改 calico.yaml 留作记录

```text
kubeadm 启动的时候要带上 --pod-network-cidr=172.26.0.0/20  带上这个参数

或者

修改 calico.yaml 
- name: CALICO_IPV4POOL_CIDR
    value: "172.26.0.0/20"
```

### kubeadm 部署

~~kubeadm init --config ~/kubeadm.yml~~ 这种方法还是不用了吧 因为不知道怎么写配置网络的配置

`kubeadm init --pod-network-cidr=172.26.0.0/20 --apiserver-advertise-address=172.26.9.41 --image-repository=registry.aliyuncs.com/google_containers`

* --pod-network-cidr 网段
* --apiserver-advertise-address master的内网IP
* --image-repository 使用的镜像源

输出

```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.26.9.41:6443 --token cmucay.d7uqussmz5i6bg04 --discovery-token-ca-cert-hash sha256:e371bb1ca72a264d65c0c6c0b80d3b5158955d151ce24cc43e5487418b566130
```

请仔细查看输出的内容 **到这里Kubernetes已经启动**

```text
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

子节点需要通过kubeadm join来加入到集群中  
kubeadm join 172.26.9.41:6443 --token cmucay.d7uqussmz5i6bg04 --discovery-token-ca-cert-hash sha256:e371bb1ca72a264d65c0c6c0b80d3b5158955d151ce24cc43e5487418b566130
```

### 修改环境变量

```text
echo export KUBECONFIG=/etc/kubernetes/admin.conf >> .bashrc
source .bashrc
```

### 安装网络

使用上面下载的的两个文件

`kubectl apply -f rbac-kdd.yaml`

输出

```bash
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
```

`kubectl apply -f calico.yaml`

输出

```bash
configmap/calico-config created
service/calico-typha created
deployment.apps/calico-typha created
poddisruptionbudget.policy/calico-typha created
daemonset.extensions/calico-node created
serviceaccount/calico-node created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
```

### 检查状态

```text
查看master是否为ready
kubectl get node 

查看kubernetes镜像是否RUNNING
kubectl get pods -n kube-system
```

记得多刷新几次 如果出现问题使用这个命令查看是什么问题

```text
kubectl describe pods -n kube-system podname
```

### 开启Master节点可以部署POD

默认情况下，出于安全原因，您的群集不会在主服务器上安排pod。如果您希望能够在主服务器上安排pod，例如，对于用于开发的单机Kubernetes集群，请运行：

```text
kubectl taint nodes --all node-role.kubernetes.io/master-
输出 
node/master untainted
```

### 关于join令牌

```text
如果您没有令牌，可以通过在主节点上运行以下命令来获取它：

kubeadm token list

默认情况下，令牌在24小时后过期。如果在当前令牌过期后将节点加入群集，则可以通过在主节点上运行以下命令来创建新令牌：

kubeadm token create

输出类似于：
5didvk.d09sbcov8ph2amjw

如果您没有值--discovery-token-ca-cert-hash，可以通过在主节点上运行以下命令链来获取它：

openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //'
输出类似于：

8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78
```

### 子节点操作

```text
如上文档执行到 禁用swapof 就可以了

加入到master
kubeadm join 172.26.9.41:6443 --token cmucay.d7uqussmz5i6bg04 --discovery-token-ca-cert-hash sha256:e371bb1ca72a264d65c0c6c0b80d3b5158955d151ce24cc43e5487418b566130

输出
[preflight] Running pre-flight checks
    [WARNING Hostname]: hostname "node1" could not be reached
    [WARNING Hostname]: hostname "node1": lookup node1 on 100.100.2.138:53: no such host
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Activating the kubelet service
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

如果都成功就回到主节点执行(多刷新几次)
kubectl get nodes
```

### 查看系统的Pods

```text
kubectl get pods -n kube-system

查看子节点的信息
kubectl get nodes

describe 子节点信息
kubectl describe node node1

describe namespace 为 kube-system coredns-8686dcc4fd-xr9lj 的信息
kubectl describe pods coredns-8686dcc4fd-xr9lj  -n kube-system
```

### 给子节点分配Role

```text
[root@master ~]# kubectl get nodes
NAME     STATUS   ROLES    AGE     VERSION
master   Ready    master   16m     v1.14.1
node1    Ready    <none>   5m34s   v1.14.1

可以看到node1的ROLES为<none>

kubectl label node node1 node-role.kubernetes.io/worker=node
```

### 子节点操作集群

`https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/`

```text
scp root@<master ip>:/etc/kubernetes/admin.conf .
kubectl --kubeconfig ./admin.conf get nodes
```

如

```text
[root@iZ8vbf8jnlyocsxorwia1bZ ~]# kubectl --kubeconfig ./admin.conf get nodes
NAME     STATUS   ROLES    AGE   VERSION
master   Ready    master   94m   v1.14.1
node1    Ready    worker   83m   v1.14.1
```

