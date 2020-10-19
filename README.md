# kubernets-install
### 一、简介
#### 版本介绍
- kubernets: v1.18.2
- docker: 19.03.8

#### 安装

- [kubeadm安装](#kubeadm-install)
### 二、环境准备
#### 1. 服务器说明
我这里使用是3台centos8虚拟机，如下：

| 系统类型 | IP地址 | 角色 | CUP | Memory | 主机名 |
| :-----: | :-----------: | :----:| :---: | :---: | :-----------:|
| centos-8| 10.123.235.13 | master | \>=2 | \>= 4G | master |
| centos-8| 10.123.235.14 | node  | \>=2 | \>= 4G | node-1 |
| centos-8| 10.123.235.25 | node | \>=2 | \>= 4G | node-2 |

#### 2.服务器设置（所有节点）
##### 2.1 设置hosts，通过主机名访问
```Shell
$ vim /etc/hosts 
10.123.235.13 master
10.123.235.14 node-1
10.123.235.25 node-2

```
##### 2.2 禁用防火墙、swap、SELINUX
```Shell
# 防火墙
$ systemctl stop firewalld
$ systemctl disable firewalld

# 关闭swap
$ swapoff -a
永久禁用：用vi修改/etc/fstab文件，注释掉swap分区这行

# SELINUX
$ setenforce 0
$ vim /etc/selinux/config
SELINUX=disabled
```
##### 2.3 修改系统参数
```shell
$ cat <<EOF > /etc/sysctl.d/k8s.conf 
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# 应用文件
$ sysctl -p /etc/sysctl.d/k8s.conf
```
#### 3. 安装docker（所有节点）
[参考](https://segmentfault.com/a/1190000021747989)

### <a name="kubeadm-install">三、kubeadm安装</a>
#### 1. 安装工具（所有节点）
##### 1.1 工具说明
- kubeadm: 部署集群用的命令
- kubelet: 在集群中每台机器上都要运行的组件，负责管理pod、容器的生命周期
- kubectl: 集群管理工具（可选，只要在控制集群的节点上安装即可）
##### 1.2 安装方法
- 官方源，需要科学上网
```shell
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
```
- 阿里云源
```shell
$ cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg
      http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

```
安装
```shell script
$ dnf install -y kubeadm-1.18.2-0 kubelet-1.18.2-0 kubectl-1.18.2-0
$ systemctl enable kubelet && systemctl start kubelet
```
安装完成后，我们还需要对kubelet进行配置，因为用yum源的方式安装的kubelet生成的配置文件将参数--cgroup-driver改成了systemd，而 docker 的cgroup-driver是cgroupfs，这二者必须一致才行
```shell script
# 修改docker配置
$ vim /etc/docker/daemon.json
{
 "exec-opts":["native.cgroupdriver=systemd"]
}
```
#### 2. 初始化master
##### 2.1 创建配置文件
```shell script
$ cat <<EOF > k8s-master-init.yaml
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
kubernetesVersion: v1.18.2
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
controlPlaneEndpoint: "master:6443"
networking:
  serviceSubnet: "10.96.0.0/16"
  podSubnet: "10.100.0.1/16"
  dnsDomain: "cluster.local"
EOF
```
##### 2.2 初始化
```shell script
$ kubeadm init --config=k8s-master-init.yaml
# 初始化后会输出
kubeadm join master:6443 --token gl4o28.ogbsnunzvveccytn \
    --discovery-token-ca-cert-hash sha256:896bd0bd87e639e6149f7d8098f618d6eadc74bf99c0bca7204540def7e88fd4

# 配置 kubectl
rm -rf /root/.kube/
mkdir /root/.kube/
cp -i /etc/kubernetes/admin.conf /root/.kube/config

# 安装 calico 网络插件
```






