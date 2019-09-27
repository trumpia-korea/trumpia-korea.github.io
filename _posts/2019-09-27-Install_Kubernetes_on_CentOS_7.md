# Install Kubernetes on CentOS 7
> 작성자: 유요셉
> 
> 최종수정일: 2018-12-28



### Kubernetes 구성
#### Master Node
* API Server
* Scheduler
* Controller Manager
* etcd
* Kubectl utility

#### Worker Node
* Kubelet
* Kube-Proxy
* Pod



### CentOS 7에 Kubernetes 설치
* * *
#### Master/Worker 공통



##### 1. Disable SELinux & setup firewall rules
> set hostname and disable selinux
> 
```console
[root@host ~]# hostnamectl set-hostname 'k8s-master'	# If Master, else 'worker-num'
[root@host ~]# exec bash
[root@host ~]# setenforce 0
[root@host ~]# sed -i --follow-symlinks 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
```

> set firewall rules(If Master)
> 
```console
[root@host ~]# firewall-cmd --permanent --add-port=6443/tcp
[root@host ~]# firewall-cmd --permanent --add-port=2379-2380/tcp
[root@host ~]# firewall-cmd --permanent --add-port=10250/tcp
[root@host ~]# firewall-cmd --permanent --add-port=10251/tcp
[root@host ~]# firewall-cmd --permanent --add-port=10252/tcp
[root@host ~]# firewall-cmd --permanent --add-port=10255/tcp
```

> set firewall rules(If Worker)
> 
```console
[root@host ~]# firewall-cmd --permanent --add-port=6783/tcp
[root@host ~]# firewall-cmd --permanent --add-port=10250/tcp
[root@host ~]# firewall-cmd --permanent --add-port=10255/tcp
[root@host ~]# firewall-cmd --permanent --add-port=30000-32767/tcp
```

> reload and set firewall
> 
```console
[root@host ~]# firewall-cmd --reload
[root@host ~]# modprobe br_netfilter
[root@host ~]# echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
```

> swap off
> 
```console
[root@host ~]# swapoff -a
```



##### 2. Configure Kubernetes Repository
> add repository
> 
```console
[root@host ~]# cat <<EOF > /etc/yum.repos.d/kubernetes.repo
> [kubernetes]
> name=Kubernetes
> baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
> enabled=1
> gpgcheck=1
> repo_gpgcheck=1
> gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
>         https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
> EOF
```



##### 3. Install kubeadm & docker
> install kubeadm and docker
> 
```console
[root@host ~]# yum install kubeadm docker -y
```

> enable and start docker & kubelet
> 
```console
[root@host ~]# systemctl restart docker && systemctl enable docker
[root@host ~]# systemctl restart kubelet && systemctl enable kubelet
```



##### ※ If [Error] curl#60 - "Peer's Certificate has expired." occured after add repository
```console
[root@host ~]# rm /etc/yum.repos.d/kubernetes.repo
[root@host ~]# yum install ntpdate
[root@host ~]# ntpdate -u 0.centos.pool.ntp.org
```
> then, try again.



* * *
#### Master



##### 1. Config hosts
```console
[root@k8s-master ~]# vi /etc/hosts
```
> 192.168.2.134	k8s-master
> 
> 192.168.2.135	worker-node1
> 
> 192.168.2.144	worker-node2



##### 2. Initialize Kubernetes Master
```console
[root@k8s-master ~]# kubeadm init
```
![kubeadm init result](images/js-virt01_005.jpg "kubeadm init result")

> set cluster
> 
```console
[root@k8s-master ~]# mkdir -p $HOME/.kube
[root@k8s-master ~]# cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
[root@k8s-master ~]# chown $(id -u):$(id -g) $HOME/.kube/config
```



##### 3. Deploy pod network to the cluster
> below commands get status of cluster and pods.
> 
```console
[root@k8s-master ~]# kubectl get nodes
[root@k8s-master ~]# kubectl get pods --all-namespaces
```

To make the cluster status ready and kube-dns status running, deploy the pod network so that containers of different host communicated each other.  POD network is the overlay network between the worker nodes.

> deploy network
> 
> > Kubernetes는 Pod들의 통신을 위한 Networking Layer가 필요하다.
> > 
> > 사용 가능한 플러그인은 ==[kubernetes site](http://kubernetes.io/docs/admin/addons/)== 에서 찾을 수 있다.
> > 
```console
[root@k8s-master ~]# export kubever=$(kubectl version | base64 | tr -d '\n')
[root@k8s-master ~]# kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$kubever"
```
![deploy network](images/js-virt01_007.jpg "deploy notwork")



* * *
#### Worker



##### 1. Join worker nodes to master node
> enter command that Master Initialized result.
> 
```console
[root@worker ~]# kubeadm join 192.168.2.134:6443 --token wu52z8.q5gbxs4f0382de3m --discovery-token-ca-cert-hash sha256:d498fc1051286da4fc407ec47722de4db839a217730bc67c09df82aad92f1c50
```
> kubeadm join result

![kubeadm_join_result](images/js-virt03_008.jpg "kubeadm join result")

> check on Master-side

![kubectl_get_nodes_result](images/kubectl_getnodes_pods_result.jpg "kubectl_get_nodes_result")

