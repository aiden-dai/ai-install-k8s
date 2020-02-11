# ai-install-k8s
Guide to set up a Kubernetes environment (V1.17.2) with one master and one worker node.  Running on Virtualbox VMs.


## 1. Preparation

### 1.1 Install CentOS 7 VM

Virtualbox VM (Master):
- 4G ram
- 4 CPU
- Network: Nat + Host 
- CentOS 7 (Minimal install)


### 1.2 Set Network

Update networks, run
```bash
vi /etc/sysconfig/network-scripts/ifcfg-enp0s3
```
set ONBOOT=yes


```
vi /etc/sysconfig/network-scripts/ifcfg-enp0s8
```

set ONBOOT=yes

For example:

<img src="images/network-onboot.PNG" width='50%'>

Run `systemctl restart network` to restart network



Run `ifconfig` or `ip addr` to check the ip address

<img src="images/network-master.PNG" width='50%'>

> Note: For minimal CentOS, please run `yum install -y net-tools` in order to use ifconfig


### 1.3 Update system

Connect to SSH, Run
```
yum install -y wget

wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

yum clean all
yum makecache

yum update
```



### 1.4 Disable SELinux

Set SELinux in permissive mode (effectively disabling it)
Run
```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

### 1.5 Disable Swap

You MUST disable swap in order for the kubelet to work properly.
```
vi /etc/fstab
```

<img src="images/swap1.PNG" width='50%'>

```
swapoff -a
```

### 1.6 Disable Firewall
Run
```bash
systemctl stop firewalld && systemctl disable firewalld
```

### 1.7 sysctl iptables

You should ensure net.bridge.bridge-nf-call-iptables is set to 1 in your sysctl config, e.g.
```
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```
<img src="images/iptables.PNG" width='50%'>


### 1.8 br_netfilter
Make sure that the br_netfilter module is loaded. 

```
modprobe br_netfilter
lsmod | grep br_netfilter
```
<img src="images/netfilter.PNG" width='50%'>


## 2. Install Container Runtime

Please follow the offical docker installation guide, the current version installed is 19.03
```
yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2

yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

yum install docker-ce docker-ce-cli containerd.io
```

```bash
# Create /etc/docker directory.
mkdir /etc/docker

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
  ],
  "registry-mirrors": ["http://hub-mirror.c.163.com"]
}
EOF

mkdir -p /etc/systemd/system/docker.service.d

# Restart Docker
systemctl daemon-reload
systemctl restart docker
```


## 3. Installing kubeadm, kubelet and kubectl

Add the repo.
```
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
```

Start install
```
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl enable --now kubelet
```

The current version installed is V1.17.2

<img src="images/kube-version.PNG" width='50%'>



## 4. Copy master to create node VM

Full Copy Master to create node VM, make sure the mac address is re-generated.

### 4.1 Change Host Name

master VM:
```
hostnamectl set-hostname aiden-kube-master
```

node VM:
```
hostnamectl set-hostname aiden-kube-node
```

### 4.2 Change Hosts

Run `ifconfig` or `ip addr` to check the ip address for node VM

<img src="images/network-node.PNG" width='50%'>


Update /etc/hosts, run
```bash
cat >> /etc/hosts << EOF
192.168.56.106  aiden-kube-master
192.168.56.105   aiden-kube-node
EOF
```


## 5. Start Master
```bash
systemctl start docker

kubeadm init     --apiserver-advertise-address=192.168.56.106     --image-repository registry.aliyuncs.com/google_containers     --kubernetes-version v1.17.2     --pod-network-cidr=192.168.0.0/16
```

The output

<img src="images/kube-init.PNG" width='50%'>


Run `kubectl get pods --all-namespaces`

<img src="images/pod1.PNG" width='50%'>

> Note, the coredns pod will be in pending status

## 6. Install CoreDNS

On Master VM, we are going to use `Calico` as the CoreDNS provider. By default, Calico uses 192.168.0.0/16 as the Pod network CIDR, For Calico to work correctly, you need to pass this same CIDR to the kubeadm init command using the --pod-network-cidr=192.168.0.0/16 flag or via the kubeadm configuration.

```bash
kubectl apply -f https://docs.projectcalico.org/v3.11/manifests/calico.yaml
```

<img src="images/calico.PNG" width='50%'>

Run `kubectl get pods --all-namespaces`

<img src="images/pod2.PNG" width='50%'>

And once the CoreDNS Pod is up and running, you can continue by joining your nodes.


## 7. Joining to k8s master

On node VM, Run
```bash
scp root@aiden-kube-master:/etc/kubernetes/admin.conf /etc/kubernetes/

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
<img src="images/node-config.PNG" width='50%'>

To join to the master, run:
```bash
kubeadm join 192.168.56.106:6443 --token j2qfct.tnam91v7i2wqdgc9 \
    --discovery-token-ca-cert-hash sha256:29db0966131178a76906f7db8d17210693dc0c093b060869085caa99b8434a67
```

> This is the output of `kubeadm init` on master VM.


<img src="images/join.PNG" width='50%'>

Then check the nodes and pods.
```
kubectl get nodes -o wide

kubectl get pods --all-namespaces
```

<img src="images/kube-final.PNG" width='50%'>

Now we can see two nodes, one master and one worker node.