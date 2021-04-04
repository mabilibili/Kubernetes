

# kubeadm搭建k8s集群(简易版)



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
 wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
 
 
```



# 3. K8s-1.18.14集群配置（kubeadm）

##3.1 安装docker以及kubeadm

```shell
#Master以及node 都需要安装
yum -y install kubeadm-1.18.14 kubectl-1.18.14 kubelet-1.18.14

yum -y install yum-utils device-mapper-persistent-data lvm2 unzip wget

 wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
 
yum -y install docker-ce-18* docker-ce-cli-18*


```

## 3.2 docker加速配置及启动





```shell
mkdir /etc/docker -p
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "registry-mirrors": ["https://uyah70su.mirror.aliyuncs.com"]
}
EOF

systemctl start docker && systemctl enable docker


# 所有服务器上将kubelet服务自动启动打开
systemctl enable kubelet




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

### 3.4.1 Calico 官方下载最新package

Calico-v3.17.1.tgz

### 3.4.2 上传并解压

tar zxvf Calico-v3.17.1.tgz -C /opt/

### 3.4.3 导入所有镜像

```shell
cd release-v3.17.1/images/

for x in `ls ` 
do 
 docker load -i $x 
done
```



### 3.4.4 应用Calico

```shell
#创建cni目录
/etc/cni/net.d

#应用calico
kubectl apply -f release-v3.17.1/k8s-manifests/calico.yaml

```



### 3.4.5 查看集群状态

```shell
kubectl get pod --all-namespaces
```



## 3.5 查看【默认】集群配置信息



```shell
#如果主和从未使用kubeadm reset，以下即是集群信息
kubeadm config print init-defaults

#或-修改kubeadm-config.yaml
kubeadm config print join-defaults > kubeadm-config.yaml

```





# 4 Post cluster installation

## 4.1 kube-proxy 开启ipvs

```shell
#在master节点执行,修改ConfigMap的kube-system/kube-proxy中的config.conf，mode: “ipvs”
kubectl edit cm kube-proxy -n kube-system 

#再重启kube-proxy pod
kubectl get pod -n kube-system | grep kube-proxy | awk '{system("kubectl delete pod "$1" -n kube-system")}'

```



## 4.2 忘记node加入信息，解决方案



```shell
#方案一：master执行

kubeadm token create --print-join-command


#方案二
kubeadm token list

result1

openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \
   openssl dgst -sha256 -hex | sed 's/^.* //' 
   
result2

kubeadm join 172.27.9.131:6443 --token result1 --discovery-token-ca-cert-hash sha256:result2


```



# 5. 安装bash-completion

```shell
yum -y install bash-completion

source /etc/profile.d/bash_completion.sh

echo "source <(kubectl completion bash)" >> ~/.bash_profile

source .bash_profile 

```





# 6. 部署应用测试集群

```shell
kubectl create deployment nginx --image=nginx:latest
kubectl get pods
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod,svc

```



# 7. Dashboard 安装





# N-1 Addon 下载

## N-1.1

```shell
docker pull
registry.cn-hangzhou.aliyuncs.com/kuberneters/kubernetes-dashboard-amd64:v1.10.1

docker pull
registry.cn-hangzhou.aliyuncs.com/kuberneters/kubernetes-dashboard-amd64:v1.10.1
registry.aliyuncs.com/google_containers/coreos/flannel:v0.13.1-rc1

docker pull registry.cn-hangzhou.aliyuncs.com/coreos/flannel:v0.13.1-amd64


```







# N. Reference



https://github.com/loong576/Centos7.6-install-k8s-v1.14.2-cluster/blob/master/image.sh

##1 .将image重新打标签

```shell

#!/bin/bash
url=registry.cn-hangzhou.aliyuncs.com/google_containers
version=v1.14.2
images=(`kubeadm config images list --kubernetes-version=$version|awk -F '/' '{print $2}'`)
for imagename in ${images[@]} ; do
  docker pull $url/$imagename
  docker tag $url/$imagename k8s.gcr.io/$imagename
  docker rmi -f $url/$imagename
done
```



## 2. 批量导入image包

```shell
for x in `ls ` 
do 
 docker load -i $x 
done



```

