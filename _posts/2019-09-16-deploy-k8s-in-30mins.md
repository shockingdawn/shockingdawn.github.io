---
layout: post
title:  "30分钟部署Kubernetes集群"
categories: Linux
tags:  k8s  
author: Utachi
---

* content
{:toc}

kubeadm是官方社区推出的一个用于快速部署kubernetes集群的工具。

这个工具能通过两条指令完成一个kubernetes集群的部署：

````bash
# 创建一个 Master 节点
kubeadm init

# 将一个 Node 节点加入到当前集群中
kubeadm join <Master节点的IP和端口 >
````

## 1. 安装要求

在开始之前，部署Kubernetes集群机器需要满足以下几个条件：

- 一台或多台机器，操作系统 CentOS7.x-86_x64
- 硬件配置：2GB或更多RAM，2个CPU或更多CPU，硬盘30GB或更多
- 集群中所有机器之间网络互通
- 可以访问外网，需要拉取镜像
- 禁止swap分区

## 2. 部署目标

1. 在所有节点上安装Docker和kubeadm
2. 部署Kubernetes Master
3. 部署容器网络插件
4. 部署 Kubernetes Node，将节点加入Kubernetes集群中
5. 部署Dashboard Web页面，可视化查看Kubernetes资源






## 3. 准备环境

````bash
#关闭防火墙：
systemctl stop firewalld
systemctl disable firewalld

#关闭selinux：
sed -i 's/enforcing/disabled/' /etc/selinux/config 
setenforce 0

#关闭swap：
swapoff -a  
vim /etc/fstab  

#添加主机名与IP对应关系（记得设置主机名）：
cat /etc/hosts

10.10.10.120 k8s-master
10.10.10.121 k8s-node1
10.10.10.122 k8s-node2

#将桥接的IPv4流量传递到iptables的链：
cat > /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
````

## 4. 所有节点安装Docker/kubeadm/kubelet

Kubernetes默认CRI（容器运行时）为Docker，因此先安装Docker。

### 4.1 安装Docker

````bash
wget https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo -O /etc/yum.repos.d/docker-ce.repo
yum -y install docker-ce-18.06.1.ce-3.el7
mkdir -p /etc/docker
tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://w9jw321y.mirror.aliyuncs.com"]
}
EOF
systemctl daemon-reload && systemctl restart docker && systemctl enable docker
docker info 
````

### 4.2 添加阿里云YUM软件源

````bash
cat > /etc/yum.repos.d/kubernetes.repo << EOF
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
````

### 4.3 安装kubeadm，kubelet和kubectl

由于版本更新频繁，这里指定版本号部署：

````bash
yum install -y kubelet-1.15.0 kubeadm-1.15.0 kubectl-1.15.0
systemctl enable kubelet
````

## 5. 部署Kubernetes Master

在10.10.10.120（Master）执行。

````bash
kubeadm init \
  --apiserver-advertise-address=10.10.10.120 \
  --image-repository registry.aliyuncs.com/google_containers \
  --kubernetes-version v1.15.0 \
  --service-cidr=10.1.0.0/16 \
  --pod-network-cidr=10.244.0.0/16
````

由于默认拉取镜像地址k8s.gcr.io国内无法访问，这里指定阿里云镜像仓库地址。

使用kubectl工具：

````bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl get nodes
````

## 6. 安装Pod网络插件（CNI）

````bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/a70459be0084506e4ec919aa1c114638878db11b/Documentation/kube-flannel.yml
````

确保能够访问到quay.io这个registery。

如果下载失败，可以改成这个镜像地址：registry.cn-hangzhou.aliyuncs.com/gcr_mirror/flannel:0.11.0-amd64

## 7. 加入Kubernetes Node

在10.10.10.121/10.10.10.122（Node）执行。

向集群添加新节点，执行在kubeadm init输出的kubeadm join命令：

````bash
kubeadm join 10.10.10.120:6443 --token esce21.q6hetwm8si29qxwn \
    --discovery-token-ca-cert-hash sha256:00603a05805807501d7181c3d60b478788408cfe6cedefedb1f97569708be9c5
````

## 8. 测试kubernetes集群

在Kubernetes集群中创建一个pod，验证是否正常运行：

````bash
kubectl create deployment nginx --image=nginx
kubectl expose deployment nginx --port=80 --type=NodePort
kubectl get pod,svc
````

访问地址：http://NodeIP:Port  

## 9. 部署 Dashboard

### 自签证书
如果不执行自签证书这一步，只有firefox浏览器可以打开dashboard。
````bash
#生成私钥和证书标识
openssl genrsa -des3 -passout pass:x -out dashboard.pass.key 2048
...
openssl rsa -passin pass:x -in dashboard.pass.key -out dashboard.key
# Writing RSA key
rm -f dashboard.pass.key
openssl req -new -key dashboard.key -out dashboard.csr
...
Country Name (2 letter code) [AU]: CN
...
A challenge password []:
...
#生成SSL证书
openssl x509 -req -sha256 -days 365 -in dashboard.csr -signkey dashboard.key -out dashboard.crt

#将dashboard.crt 、 dashboard.key存放于$HOME/certs目录下然后创建证书管理服务：
kubectl create secret generic kubernetes-dashboard-certs --from-file=$HOME/certs -n kubernetes-dashboard
````

### 获取yaml

````bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
````

默认镜像国内无法访问，修改阿里云镜像地址为： registry.cn-hangzhou.aliyuncs.com/google_containers/kubernetes-dashboard-amd64:v1.10.1

### 暴露服务
默认Dashboard只能集群内部访问，修改Service为NodePort类型，暴露到外部：修改kubernetes-dashboard.yaml

````bash
......
---
# ------------------- Dashboard Service ------------------- #

kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort       #增加type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30000  #增加nodePort: 30000
  selector:
    k8s-app: kubernetes-dashboard

````
````bash
kubectl apply -f kubernetes-dashboard.yaml
#也可通过执行如下命令修改
kubectl -n kube-system edit service kubernetes-dashboard
````
访问地址：http://NodeIP:30000

### 构建管理员
创建service account并绑定默认cluster-admin管理员集群角色：

````bash
#方法1
kubectl create serviceaccount dashboard-admin -n kube-system
kubectl create clusterrolebinding dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
````
````bash
#方法2（官方文档）
#创建dashboard-adminuser.yaml
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
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system

#应用yaml
kubectl apply -f dashboard-adminuser.yaml
````
### 获取token
````bash
#获取token
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/admin-user/{print $1}')
````
使用输出的token登录Dashboard。