# Kubernetes

kubeadm搭建k8s  v1.18集群(简易版)搭建.md

# 1. 集群规划

```shell
#1.主机规划
master 192.168.233.10
node1 192.168.233.11
node2 192.168.233.12

#2.网络规划
service cidr:10.10.0.0/16
pod cidr:10.122.0.0

#3.网络插件使用calico
```



# 2. 集群准备

## 2.1 /etc/hosts 准备



```shell
hostnamectl set-hostname master

cat >>/etc/hosts <<EOF

192.168.233.10 master 
192.168.233.11 node1
192.168.233.12 node2

EOF

```

## 2.2 master免密登录其他节点

```shell
ssh-keygen -t rsa

ssh-copy-id root@master

ssh-copy-id root@node1

ssh-copy-id root@node2

```

## 2.3 关闭防火墙，SElinux, swap

```shell
#Stop/disable the firewall
systemctl stop firewalld

systemctl disable firewalld

iptables -F && iptables -X && iptables -F -t nat && iptables -X -t nat

iptables -P FORWARD ACCEPT

#disable the swap
swapoff -a

sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab 

#disable the Selinux
setenforce 0

sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config


```

## 2.4 时区同步（云环境无需配置）

```shell
#只有使用本地VMware环境才需要执行此段
#调整系统timezone
#将当前的UTC时间写入硬件-永久生效

timedatectl set-timezone Asia/Shanghai
timedatectl set-local-rtc 0

```

## 2.5 内核参数优化

```shell
cat >/etc/sysctl.d/kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
net.ipv4.ip_forward=1
net.ipv4.tcp_tw_recycle=0
vm.swappiness=0
vm.overcommit_memory=1 
vm.panic_on_oom=0
fs.inotify.max_user_instances=8192
fs.inotify.max_user_watches=1048576
fs.file-max=52706963
fs.nr_open=52706963
net.ipv6.conf.all.disable_ipv6=1
net.netfilter.nf_conntrack_max=2310720
EOF


sysctl -p /etc/sysctl.d/kubernetes.conf


```



## 2.6 安装ipvs并启用

```SHELL
yum install ipvsadm ipset sysstat conntrack libseccomp -y

cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF


chmod 755 /etc/sysconfig/modules/ipvs.modules && bash 
/etc/sysconfig/modules/ipvs.modules

lsmod | grep -e ip_vs -e nf_conntrack_ipv4

```

##2.7 优化rsyslogd 和 systemd journald

```shell
mkdir -p /var/log/journal 

mkdir -p /etc/systemd/journald.conf.d

cat > /etc/systemd/journald.conf.d/99-prophet.conf <<EOF
[Journal]
#persist the data
Storage=persistent

# compress the log
Compress=yes

SyncIntervalSec=5m
RateLimitInterval=30s
RateLimitBurst=1000

#the max memory to use
SystemMaxUse=10G

# the maximum size for the log =  200M
SystemMaxFileSize=200M

# keep the log for 2 weeks
MaxRetentionSec=2week

# dont forward the log to syslog
ForwardToSyslog=no
EOF

systemctl restart systemd-journald
```

## 2.8 YUM 仓库配置

```shell
https://github.com/mabilibili/Linux/blob/main/CentOS7_aliyun.repo
https://github.com/mabilibili/Linux/blob/main/kubernetes_os7_aliyun.repo
https://github.com/mabilibili/Linux/blob/main/docker_os7_aliyun.repo

```



# 3. K8s-1.18.14集群配置（kubeadm）

##3.1 安装docker以及kubeadm

```shell
#Master以及node 都需要安装
yum -y install kubeadm-1.18.14 kubectl-1.18.14 kubelet-1.18.14

yum -y install yum-utils device-mapper-persistent-data lvm2

yum -y install docker-ce-18* docker-ce-cli-18*

systemctl start docker && systemctl enable docker



```

##3.2 初始化K8s集群（master）

```shell
#拉去k8s组件镜像（如果带宽允许直接跑下面的）
kubeadm config images pull --kubernetes-version=stable-1.18 --image-repository=registry.aliyuncs.com/google_containers

#初始化集群
kubeadm init --kubernetes-version=stable-1.18 --image-repository=registry.aliyuncs.com/google_containers --apiserver-advertise-address=192.168.233.10 --service-cidr=10.10.0.0/16 --pod-network-cidr=10.122.0.0/16

#查看集群基本信息
kubeadm cluster-info

#配置kubectl环境
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

#查看集群所有信息
kubectl get nodes all

```



##3.3 work节点加入集群

```shell
kubeadm join 192.168.233.10:6443 --token 44tmag.esl75jvr1zskpmex \
  --discovery-token-ca-cert-hash sha256:26ae63522d00464ea9e57e21f996b9ab70c42166d0e1d4d4e96e6471006dc9a4 


```



##3.4 master安装calico

去github下载calico的yaml文件，然后新建文件到master节点，再应用即可

```shell
kubectl apply -f /etc/kubernetes/addons/calico.yaml

安装后，网络可能还不通，需要手动删掉calico的docker实例，之后会calico会自动新建此实例，然后kubernetes集群会跑起来

```



