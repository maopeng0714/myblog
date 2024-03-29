+++
title = "高可用K8S集群部署"
date = "2019-10-15"
author = "Peng Mao"
description = "详细讲解怎么在自己的电脑上搭建起利用HAPROXY+KEEPALIVED的高可用K8S集群"
+++


#### 安装配置虚拟机

- 下载Virtual Box 和 CentOS 镜像。
- 下载CentOS 7任意镜像版本，注意不要下载CentOS 8，有兼容性问题。
- 软件包选择Infrustructure Server, 虽然不带图形界面节省空间但是也带有必要的组件。
- 配置yum源

    ``` bash
    wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    yum makecache
    yum update
    ```

- 关闭、禁用防火墙：

    ```bash
    systemctl stop firewalld
    systemctl disable firewalld
    ```

- 禁用SELINUX：
  只有执行这一操作之后，容器才能访问宿主的文件系统，进而能够正常使用 Pod 网络。您必须这么做，直到 kubelet 做出升级支持 SELinux 为止

   ```bash
   setenforce 0
   ```

- 关闭Swap

  ```bash
   - swapoff -a可临时关闭，但系统重启后恢复
   - 编辑/etc/fstab，注释掉包含swap的那一行即可，重启后可永久关闭，如下所示：
  ```

- 设置主机名

    ```bash
    - hostname kube-master 临时改变
    - 编辑/etc/hosts 文件
    - 编辑 /etc/sysconfig/network 加入hostname
    ```

#### 设置虚拟机网络

1. VirtualBox的5种连接方式
    - NAT ：虚拟机之间不能互通
    - NAT网络 ：本文对象
    - 桥接 ：一般情况下虚拟机无法设置静态IP，并且浪费外部局域网IP
    - 内部 ：虚拟机不能连外网
    - 仅主机(host-only) ：虚拟机不能连外网，并且不互通

2. NAT网络面向需求
    - 虚拟机可以连外网
    - 虚拟机与主机互通
    - 虚拟机与虚拟机互通
    - 虚拟机需要固定IP (防止意外)
    - 主机所在局域网的其他机器访问虚拟机
3. NAT网络基本配置方法

    - VB设置：添加NAT Network配置
      依次点击《VirtualBox》 -> 《偏好设置》 -> 《网络》 -> 《NAT网络》 -> 《+号》
      先可以采用默认配置，后面在依据需求配置《端口转发》的功能
    - VB设置：选择虚拟机网络连接方式
      选中对应的虚拟机镜像，依次点击《设置》 -> 《网络》 -> 《连接方式》 -> 选择《NAT 网络》，《界面名称》选择第1步中设置的对象

4. 编辑 /etc/sysconfig/network文件（主机名、默认网关、DNS）

    ```bash
      # Created by anaconda
        NETWORKING=yes
        DNS1=8.8.8.8
        DNS2=8.8.4.4
        HOSTNAME=kube-master
    ```

5. 设置固定IP
  编辑 /etc/sysconfig/network-scripts/ifcfg-enp0s3（配置ip地址、网关、DNS）

    ```bash
        TYPE="Ethernet"
        PROXY_METHOD=none
        BROWSER_ONLY=no
        NM_CONTROLLED=no
        BOOTPROTO="static"
        DEFROUTE="yes"
        IPV4_FAILURE_FATAL="no"
        NAME="enp0s3"
        DEVICE="enp0s3"
        ONBOOT="yes"
        IPADDR="10.0.2.10"
        NETMASK="255.255.255.0"
        GATEWAY="10.0.2.1"

    ```

6. 配置DNS解析， 编辑 /etc/resolve.conf

    ```bash
        # Generated by NetworkManager
        nameserver 8.8.8.8
        nameserver 8.8.4.4
    ```

7. 重启网络服务

``` bash
     systemctl restart network
```

#### 安装Docker

 注意安装k8s支持的版本匹配，不要太高

1. 安装必要的驱动

```bash
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

2. 添加软件源信息

```bash
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
```

3. 更新并安装 Docker-CE

```bash
sudo yum makecache fast
yum list docker-ce.x86_64  --showduplicates |sort -r
yum install -y --setopt=obsoletes=0 docker-ce-18.09.8-3.el7
```

4. 开启Docker服务

```bash
# Setup daemon.
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
sudo systemctl start docker
sudo systemctl enable docker
```

5. 可能碰到的问题

如果报错说docker版本太高，卸载docker，选择合适的版本重装：

``` bash
yum -y remove docker-ce.x86_64
yum -y remove docker-ce-cli.x86_64
yum -y remove containerd.io.x86_64
rm -rf /var/lib/docker
yum list docker-ce.x86_64  --showduplicates |sort -r
yum install -y --setopt=obsoletes=0 docker-ce-18.09.8-3.el7
systemctl start docker
systemctl enable docker
```

#### 网桥配置

配置L2网桥在转发包时会被iptables的FORWARD规则所过滤，该配置被CNI插件需要

```bash
echo """
vm.swappiness = 0
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
""" > /etc/sysctl.conf
sysctl -p
```

#### 配置内核参数

修复ipvs模式下长连接timeout问题 小于900即可

```bash
# https://github.com/moby/moby/issues/31208 
# ipvsadm -l --timout
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 10
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
net.ipv4.neigh.default.gc_stale_time = 120
net.ipv4.conf.all.rp_filter = 0
net.ipv4.conf.default.rp_filter = 0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce = 2
net.ipv4.conf.all.arp_announce = 2
net.ipv4.ip_forward = 1
net.ipv4.tcp_max_tw_buckets = 5000
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 1024
net.ipv4.tcp_synack_retries = 2
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.netfilter.nf_conntrack_max = 2310720
fs.inotify.max_user_watches=89100
fs.may_detach_mounts = 1
fs.file-max = 52706963
fs.nr_open = 52706963
net.bridge.bridge-nf-call-arptables = 1
vm.swappiness = 0
vm.overcommit_memory=1
vm.panic_on_oom=0
EOF
sysctl --system

```

#### 开启并加载IPVS模块

在所有的Kubernetes节点执行以下脚本（若内核大于4.19替换nf_conntrack_ipv4为nf_conntrack）:

```bash
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
#执行脚本
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
#安装相关管理工具
yum install ipset ipvsadm -y

```

#### 安装kubelet，kubeadm，kubectl

```bash
#配置kubernetes.repo为阿里云
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg  http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum install -y kubeadm-1.14.7-0 kubectl-1.14.7-0  kubelet-1.14.7-0  --setopt=obsoletes=0 --disableexcludes=kubernetes
systemctl enable --now kubelet.service
```

#### 配置kubelet

确保docker 的cgroup driver 和kubelet的cgroup driver一样：

```bash
docker info | grep -i cgroup

vim /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

vim /var/lib/kubelet/config.yaml

vim /var/lib/kubelet/kubeadm-flags.env
KUBELET_KUBEADM_ARGS="--network-plugin=cni --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1"

更新为 cgroupDriver: systemd

vim /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS= --runtime-cgroups=/systemd/system.slice

systemctl daemon-reload
systemctl restart kubelet
```

#### 复制虚拟机

将Master虚拟机复制两份，一个修改hostname 为kube-node1, 另一个为kube-node2.
修改上面指定的的固定IP地址，防止冲突。

### 高可用集群部署

配置高可用（HA）Kubernetes集群，有以下两种可选的etcd拓扑：

- 堆叠的etcd拓扑：集群master节点与etcd节点共存，etcd也运行在控制平面节点上
- 外部etcd拓扑：使用外部etcd节点，etcd节点与master在不同节点上运行

本次部署为了节省资源选择堆叠的etcd拓扑。
并选择 haproxy+keepalived 作为负载均衡的方案，也安装在控制平面上。

ＩＰ地址规划如下：

- kube-master  10.0.2.10
- kube-master2 10.0.2.11
- kube-master3 10.0.2.12。

VIP 地址 10.0.2.100

#### 安装 haproxy+keepalived

yum install -y haproxy keepalived

注： ３台master的 haproxy+keepalived 节点都需安装

#### 配置 keepalived

router_id和virtual_router_id 用于标识属于该 HA 的 keepalived 实例，所有节点的值必须一致；
backup的priority 的值必须小于 master 的值；

```　bash

vim /etc/keepalived/keepalived.conf

! Configuration File for keepalived

global_defs {
   router_id LVS_DEVEL
}

vrrp_instance VI_1 {
    state MASTER
    interface enp0s3
    virtual_router_id 51
    priority 100
    advert_int 1
    virtual_ipaddress {
        10.0.2.100/24
    }
}

```

其他主节点的keepalived配置文件基本一样，除了state，主节点配置为MASTER，备节点配置BACKUP，优化级参数priority，主节点设置最高，备节点依次递减。

#### 配置 haproxy

2台节点的配置一样

```bash
vim /etc/haproxy/haproxy.cfg
添加如下段落：
global
  log 127.0.0.1 local0 err
  maxconn 4096
  uid 99
  gid 99
  #daemon
  nbproc 1
  pidfile haproxy.pid

defaults
  mode http
  log 127.0.0.1 local0 err
  maxconn 4096
  retries 3
  timeout connect 5s
  timeout client 30s
  timeout server 30s
  timeout check 2s

listen admin_stats
  mode http
  bind 0.0.0.0:1080
  log 127.0.0.1 local0 err
  stats refresh 30s
  stats uri     /haproxy-status
  stats realm   Haproxy\ Statistics
  stats auth    admin:admin
  stats hide-version
  stats admin if TRUE

frontend k8s-https
  bind 0.0.0.0:8443
  mode tcp
  #maxconn 4096
  default_backend k8s-https

backend k8s-https
  mode tcp
  balance roundrobin
  server kube-master  10.0.2.10:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3
  server kube-master2 10.0.2.11:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3

```

haproxy 在 1080 端口输出 status 信息；
haproxy 监听所有接口的 8443 端口，该端口与环境变量 ${KUBE_APISERVER} 指定的端口必须一致；
server 字段列出所有 kube-apiserver 监听的 IP 和端口；

#### 启动 haproxy+keepalived

2个节点都启动

systemctl daemon-reload
systemctl enable haproxy
systemctl enable keepalived
systemctl start haproxy
systemctl start keepalived

#### K8S 主节点安装配置

导出默认配置并修改为需要的配置如下：
kubeadm config print init-defaults > kubeadm-config.yaml

```bash
apiVersion: kubeadm.k8s.io/v1beta1
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 10.0.2.10
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: kube-master
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "10.0.2.100:8443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.14.7
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 192.168.0.0/16
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
featureGates:
  SupportIPVSProxyMode: true
mode: ipvs

```

#### 初始化第一个主节点

```bash
kubeadm init --config=kubeadm-config-default.yaml --experimental-upload-certs

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

cat << EOF >> ~/.bashrc
export KUBECONFIG=/etc/kubernetes/admin.conf
EOF
source ~/.bashrc
```

#### 初始化Calico网络

```bash
kubectl apply -f calico.yaml 

修改Calico.yml保证如下几个选项：

  # Disable IPIP
  - name: CALICO_IPV4POOL_IPIP
    value: "off"
  # IP POOL
  - name: CALICO_IPV4POOL_CIDR
    value: "192.168.0.0/16"
  # specify the NAT network card instead of the hostonly network card 
  - name: IP_AUTODETECTION_METHOD
    value: "interface=enp0s3"
```

#### 初始化其他主节点

```bash
kubeadm join 10.0.2.100:8443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:e8f36e9f3092aad715913b0eaa6196ece15c57a98db740759332eabde220a03b \
    --experimental-control-plane --certificate-key 120e900b1611e82fd43b1bbfb68e8bcabfecbd06f419f06badc2a9cc98a6ab56
```

#### 初始化工作节点

```bash
kubeadm join 10.0.2.100:8443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:e8f36e9f3092aad715913b0eaa6196ece15c57a98db740759332eabde220a03b
```

#### 将kubeconfig配置SCP到其他所有节点

```bash
scp /root/.kube/config   root@kube-master2:/root/.kube/config
scp /root/.kube/config   root@kube-node1:/root/.kube/config
scp /root/.kube/config   root@kube-node2:/root/.kube/
```

### 安装Metrics Server

1. 在master节点上，下载metric-server

git clone https://github.com/kubernetes-incubator/metrics-server.git

2. 修改metrics-server/deploy/1.8+/metrics-server-deployment.yaml(一定要修改，源码有错误！！！)

```bash
vim metrics-server/deploy/1.8+/metrics-server-deployment.yaml
//添加
command:
        - /metrics-server
        - --kubelet-insecure-tls
        - --kubelet-preferred-address-types=InternalIP 
//修改
imagePullPolicy: IfNotPresent
```

3. 下载镜像并修改名字

```bash
#docker pull mirrorgooglecontainers/metrics-server-amd64:v0.3.1

#docker tag   mirrorgooglecontainers/metrics-server-amd64:v0.3.1 k8s.gcr.io/metrics-server-amd64:v0.3.1
```

4. 安装Metrics Server

``` bash
#kubectl apply -f metrics-server/deploy/1.8+/
```

5. 验证正常工作

``` bash
#kubectl top nodes
#kubectl top pods
#kubectl get hpa
```

### 安装配置Dashboard

- 创建安全链接串

``` bash
cd certs/
openssl genrsa -des3 -passout pass:x -out dashboard.pass.key 2048
openssl rsa -passin pass:x -in dashboard.pass.key -out dashboard.key
# Writing RSA key
rm dashboard.pass.key
openssl req -new -key dashboard.key -out dashboard.csr
openssl x509 -req -sha256 -days 365 -in dashboard.csr -signkey dashboard.key -out dashboard.crt

kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kubernetes-dashboard
```

- 定制Dashboard权限

下载https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta5/aio/deploy/recommended.yaml，并编辑修改。

``` bash
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-system
```

- 安装Dashboard

    kubectl create -f recommended.yaml

- 创建用户

``` bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system

---

apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

kubectl apply -f admin-user-user.yaml

```

- 获取token
kubectl -n kube-system get sa | grep admin-user
kubectl describe secrets/admin-user-token-n7grp -n kube-system

``` bash
Name:         admin-user-token-n7grp
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin-user
              kubernetes.io/service-account.uid: 38b70ad4-f227-11e9-84c3-08002790da0d

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi11c2VyLXRva2VuLW43Z3JwIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImFkbWluLXVzZXIiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIzOGI3MGFkNC1mMjI3LTExZTktODRjMy0wODAwMjc5MGRhMGQiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06YWRtaW4tdXNlciJ9.t9mOEiRSCzPxsVrirTitP9lpta2ujOkVHYHh06-dl8yjubgqxa1d8oDJASZXlbXM1WjXl2K93Q7OCH7rLmklhCRGXtNEEjvCUaWYm4haP8jZFqujQ6XtGiQ7aExPaoshVMwT7lttrZT1ZwZ93RXdaZyPd-QMkiBOkdt47J8Q6MG7rbmPTjzgmRa3KV3oY_ST_0h-Bb9FiMBCfr70wNB5Xj3W6kilKuabgnrNec3GscdwJrzxYY-y2YCd2wcjek49Wc9d8vp4CG4I5-ywWcLKgEXuYb4Ye3tBT3d0uUQdFgwuXPLpCq1h-RJTYmwJuld9-a4DPpOQJZHdnRkZcA4UgA
```

- 使用kubectl proxy启动DashBoard
  
  kubectl proxy --address='10.0.2.10' --disable-filter=true

- 登录Dashboard
在VirtualBox的全局设置里，NAT网络设置端口映射，把本机的9001端口映射到10.0.2.10的8001端口。
在本机输入URL：
http://localhost:9001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/clusterrole?namespace=default

### HELM 安装配置
下载

helm releases 版本下载地址：https://github.com/helm/helm/releases

1. 解压并拷贝

tar -xzvf helm-v2.13.1-linux-amd64.tar.gz
cd linux-amd64 && mv helm /usr/bin/

压缩中包含两个可执行文件helm,tiller。其中tiller为server，若采用容器化部署到kubernetes中，则可以不用管tiller，只需将helm复制到/usr/bin目录即可。

2. 创建tiller用户

kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

3. 安装Tiller

 helm init --service-account tiller -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.13.1  --skip-refresh

4. 问题解决

 unable to do port forwarding: socat not found.

 解决办法在node节点安装socat

yum install socat

5. 卸载
helm reset将会移除tiller在k8s集群上创建的pod
当出现上面的context deadline exceeded时, helm reset同样会报该错误.执行

heml reset -f
强制删除k8s集群上的pod.
当要移除helm init创建的目录等数据时,执行

helm reset --remove-helm-home

### 定制安装Istio


1. Create a namespace for the istio-system components:

$ kubectl create namespace istio-system

2. 下载istio： https://istio.io/docs/setup/#downloading-the-release

切换到istio目录下面，并编辑values文件来定制所需要的参数：
vim  install/kubernetes/helm/istio/values.yaml

3. 初始化CRD资源

[root@kube-master2 istio-1.3.1]# helm template install/kubernetes/helm/istio-init -name istio-init -namespace istio-system > istio-init.yaml
[root@kube-master2 istio-1.3.1]# kubectl apply -f istio-init.yaml 

4. 安装Istio

注意：发现istio 1.3.1版本和 Helm 2.15.0 版本存在兼容性问题，需要使用helm 2.13.1 版本

helm template install/kubernetes/helm/istio --name istio --namespace istio-system > istio-default.yaml
kubectl apply -f istio-default.yaml

5. 验证Istio安装：
kubectl get svc -n istio-system
kubectl get pods -n istio-system

给Namespace打标签，这样每次部署自动的就会注入Envoy Sidecar。
kubectl label namespace <namespace> istio-injection=enabled
kubectl create -n <namespace> -f <your-app-spec>.yaml

6. 卸载

helm template install/kubernetes/helm/istio --name istio --namespace istio-system | kubectl delete -f -
或者 kubectl delete -f istio-default.yaml

kubectl delete namespace istio-system

7. 从本机访问Grafana

kubectl -n istio-system port-forward --address 10.0.2.11  pod/grafana-6fc987bd95-8j2pt  3000:3000 &

localhost:3000