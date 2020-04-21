# Build your own Kubernetes cluster in a single VM using CRI-O

In this guide, we will build Kubernetes 1.16.9 using CRI-O.

## Prerequisites - Download base VM

Use one of the two option below to download your base VM image and start the VM.

* Windows - Read through [this](Windows/)
* MacBook - Read through [this](MacBook/) 


## In base VM

The `root` password in the VM is `password`. When you start VM, it will automatically login as `user` and the password is `password` for the user `user`.

Login as root.

```
sudo su -
yum -y update
```

Note: You can copy and paste command from here to the VM. You can use middle mouse button to paste the commands from the clipboard or press `Shift-Ctrl-V` to paste the contents from the clipboard to the command line shell.

## Prerequisites

### Internet Access

Make sure that you have access to Internet from inside the VM
```
dig +search +noall +answer google.com
```

Set `SELINUX=disabled` in `/etc/selinux/config` and reboot for the VM to take effect. After reboot, you should get output from `getenforce` as `disabled`.

```
    # getenforce
    Disabled
```

## Enable firewall

Run as root

You may not need firewall in a VM for testing purposes. But, it should be used in actual systems.

```
systemctl enable NetworkManager
systemctl start NetworkManager
systemctl status NetworkManager

systemctl enable firewalld
systemctl start firewalld

firewall-cmd --zone=public --change-interface=eth0

## kube-apiserver
firewall-cmd --zone=public --add-port=6443/tcp --permanent
firewall-cmd --zone=public --add-port=10250/tcp --permanent
firewall-cmd --zone=public --add-port=10251/tcp --permanent
firewall-cmd --zone=public --add-port=10252/tcp --permanent
firewall-cmd --zone=public --add-port=10255/tcp --permanent

firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --zone=public --add-service=https --permanent

## Calico
firewall-cmd --zone=public --add-port=179/tcp --permanent
firewall-cmd --zone=public --add-port=5473/tcp --permanent
firewall-cmd --zone=public --add-port=4789/tcp --permanent

## etcd
firewall-cmd --zone=public --add-port=2379/tcp --permanent
firewall-cmd --zone=public --add-port=2380/tcp --permanent

## nodeport
firewall-cmd --zone=public --add-port=30000-32767/tcp --permanent

## load balancer for level 7 routing
firewall-cmd --zone=public --add-port=56501/tcp --permanent

firewall-cmd --reload
firewall-cmd --info-zone=public
firewall-cmd --list-all --zone=public
```

## Fix kernel param for Kubernetes

Run as root

```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.netfilter.nf_conntrack_max = 1000000
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
EOF

sysctl --system
```

## Add kernel modules

Run as root

```
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
echo br_netfilter > /etc/modules-load.d/br_netfilter.conf
modprobe overlay
echo overlay > /etc/modules-load.d/overlay.conf
modprobe br_netfilter
```

Disable `selinux`
```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

## Install crio from build repos

Run as root

Remove default `runc` from VM

```
yum -y erase runc
```

### Install dependencies
```
yum install btrfs-progs-devel container-selinux device-mapper-devel gcc git glib2-devel glibc-devel glibc-static gpgme-devel json-glib-devel libassuan-devel libgpg-error-devel libseccomp-devel make pkgconfig skopeo-containers tar wget -y

yum install golang-github-cpuguy83-go-md2man golang -y
```

### Create directories
```
for d in "/usr/local/go /etc/systemd/system/kubelet.service.d/ /var/lib/etcd /etc/cni/net.d /etc/containers"; do mkdir -p $d; done
```
### Clone runc, cri-o, cni and conmon repos
```
git clone https://github.com/opencontainers/runc /root/src/github.com/opencontainers/runc
git clone https://github.com/cri-o/cri-o /root/src/github.com/cri-o/cri-o
git clone https://github.com/containernetworking/plugins /root/src/github.com/containernetworking/plugins
git clone http://github.com/containers/conmon /root/src/github.com/conmon
```

### Build runc
```
cd /root/src/github.com/opencontainers/runc
export GOPATH=/root
make BUILDTAGS="seccomp selinux"
make install
ln -sf /usr/local/sbin/runc /usr/bin/runc


runc --version
```

```
runc version 1.0.0-rc10+dev
commit: bf0a8e17471347407fe9e856d4f3ff61beaf2fea
spec: 1.0.2
```

### Build cri-o
```
export GOPATH=/root
export GOBIN=/usr/local/go/bin
export PATH=/usr/local/go/bin:$PATH
cd /root/src/github.com/cri-o/cri-o
git checkout release-1.16
make
make install
make install.systemd
make install.config
```

### Build conmon for cri-o container monitoring

```
cd /root/src/github.com/conmon
make
make install
```

### Build cni plugin

```
cd /root/src/github.com/containernetworking/plugins
./build_linux.sh
mkdir -p /opt/cni/bin
cp bin/* /opt/cni/bin/
```

### Fix /etc/crio/crio.conf

List contents
```
sed -e "/^#.*$/d" -e "/^$/d" /etc/crio/crio.conf
```

Set cgroup_manager to systemd in /etc/crio/crio.conf

```
sed -i 's/cgroup_manager =.*/cgroup_manager = "systemd"/g' /etc/crio/crio.conf
grep cgroup_manager /etc/crio/crio.conf
```

### Set storage_driver is set overlay2

```
sed -i 's/#storage_driver =.*/storage_driver = "overlay2"/g' /etc/crio/crio.conf
grep storage_driver /etc/crio/crio.conf
```

Uncomment `storage_option` and make it `storage_option = [ "overlay2.override_kernel_check=1" ]`

Delete line after storage_option option

```
sed -ie '/#storage_option =/{n;d}' /etc/crio/crio.conf
```

Uncomment and add the value

```
sed -i 's/#storage_option =.*/storage_option = [ "overlay2.override_kernel_check=1" ]/g' /etc/crio/crio.conf
grep storage_option /etc/crio/crio.conf
```

Change network_dir to `/etc/crio/net.d` since `kubeadm reset` empties `/etc/cni/net.d`

```
sed -i 's~network_dir =.*~network_dir = "/etc/crio/net.d/"~g' /etc/crio/crio.conf
grep network_dir /etc/crio/crio.conf
```

### Create entries in /etc/crio/net.d

```
mkdir -p /etc/crio/net.d
cd /etc/crio/net.d/
cp /root/src/github.com/cri-o/cri-o/contrib/cni/10-crio-bridge.conf .
cat 10-crio-bridge.conf
```

### Get the policy.json file

```
cat << EOF > /etc/containers/policy.json
{
    "default": [
        {
            "type": "insecureAcceptAnything"
        }
    ],
    "transports": {
        "docker": {}
    }
}
EOF
```

### Add extra params to kubelet service

```
cat << EOF > /etc/sysconfig/kubelet
KUBELET_EXTRA_ARGS=--feature-gates="AllAlpha=false,RunAsGroup=true" --container-runtime=remote --cgroup-driver=systemd --container-runtime-endpoint='unix:///var/run/crio/crio.sock' --runtime-request-timeout=5m
EOF
```

### Start crio

```
systemctl enable crio
systemctl start crio
```

### Version of CRI-O

```
crio --version
```

```
crio version 1.16.6
commit: "5fb673830826e05bb3d325c0b85a62f673ba05d5-dirty"
```

### Download cri-tools

Read first to make sure which Kubernetes version is in which branch https://github.com/kubernetes-sigs/cri-tools

Version 18 is compatible with Kubernets 1.16, 1.17 and 1.18

Link https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.18.0/crictl-v1.18.0-linux-amd64.tar.gz

```
cd
VERSION="v1.18.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/local/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz

rm -f /usr/bin/crictl
ln -sf /usr/local/bin/crictl /usr/bin/crictl
```
Find out version

```
crictl --version
```

```
crictl version v1.18.0
```

## Install podman

Do not use `yum -y install podman` as it will download older version from CentOS repos.

Run as root

```
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/CentOS_7/devel:kubic:libcontainers:stable.repo

yum -y install podman

podman --version
```

```
podman version 1.9.0
```

For podman, remove metacopy from /etc/containers/storage.conf for older kernels.

```
sed -i 's/mountopt =.*/mountopt = "nodev"/g' /etc/containers/storage.conf
grep mountopt /etc/containers/storage.conf
```

## Build Kubernetes using one VM

We will use the same version for Kubernetes as we used for crio


### Add Kubernetes repo

```
cat << EOF >/etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
        https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF

rpm --import https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
```

### Install Kubernetes 

Check available versions of packages

```
yum --showduplicates list kubeadm
```

For example, we will be selecting `1.16.9-0` matching with `crio --version`

```
version=1.16.9-0
yum install -y kubelet-$version kubeadm-$version kubectl-$version
```

Restart crio and enable kubelet
```
systemctl restart crio
systemctl enable --now kubelet
```

#### Disable swap

Kuberenets does not like swap to be on.

```
swapoff -a
```

Comment entry for swap in `/etc/fstab`. Example:

```
sed -i '/ swap / s/^/#/' /etc/fstab
```

Make sure that it was commented.
```
cat /etc/fstab
```

### Run kubeadm

We will use same CIDR as it is defined in the CRI-O.

```
cat /etc/crio/net.d/10-crio-bridge.conf | grep subnet
```

```
            [{ "subnet": "10.88.0.0/16" }],
            [{ "subnet": "1100:200::/24" }]
```


Check by running `visudo` and there must be an entry `user  ALL=(ALL)       NOPASSWD: ALL` so that the user `user` has `sudo` authority to type `root` commands without requiring a password.
Type `exit` to logout from root.

```
# exit
```

Run as user and not as root

Pull images

```
sudo kubeadm config images pull --kubernetes-version=1.16.9
```

Bootstrap master node

You will need an external load balancer (DNS name pointing to an IP address that can load balance between 3 masters.) We are not using 3 masters but a single VM. The use of a DNS name is shown as that is necessary to build HA cluster.

```
eth0ip=$(ip addr show eth0 | grep "inet\b" | awk '{print $2}' | cut -d/ -f1)

sudo kubeadm init --kubernetes-version=1.16.9 --pod-network-cidr=10.88.0.0/16 --apiserver-advertise-address=$eth0ip --node-name $(hostname) 
```

Add `--control-plane-endpoint "DNS:port"` when using an external load balancer to route traffic to the master node

When apiserver needs to be started using interal IP address, which in this case is eth0 IP of the VM.

The output is as shown:

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:
```
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

You can now join any number of control-plane nodes by copying certificate authorities
and service account keys on each node and then running the following as root:
```
  kubeadm join kb.zinox.com:6443 --token nk48jp.owcrixph3wgxai8l \
    --discovery-token-ca-cert-hash sha256:2a75151fcf9552859fac6ed4b32aecfbcf1fd02143ef237392b3feca4a3ab6eb \
    --control-plane
```
Then you can join any number of worker nodes by running the following on each as root:
```
kubeadm join kb.zinox.com:6443 --token nk48jp.owcrixph3wgxai8l \
    --discovery-token-ca-cert-hash sha256:2a75151fcf9552859fac6ed4b32aecfbcf1fd02143ef237392b3feca4a3ab6eb
```    

The token is time bound and if you neeed to use `kubeadm join` later, a new token and hash can be generated as:

```
$ sudo kubeadm token create --print-join-command
sudo kubeadm token create --print-join-command
kubeadm join kb.zinox.com:6443 --token 6bhq1s.zpqiuqxv4h6gb1xv     --discovery-token-ca-cert-hash sha256:2a75151fcf9552859fac6ed4b32aecfbcf1fd02143ef237392b3feca4a3ab6eb
```

Since we will be using a single VM, the Kubernetes token from above is for reference purpose only. You will require the above token command in you require a multi-node Kubernetes cluster.

### Configure kubectl

Run the following command as `user` and `root` to configure `kubectl` command line CLI tool to communicate with the Kubernetes environment.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check Kubernetes version

```
kubectl version --short
```
```
Client Version: v1.16.9
Server Version: v1.16.9
```

Untaint the node - this is required since we have only one VM to install objects.

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Check node status

```
$ kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
ocp.zinox.com   Ready    master   5m40s   v1.16.9
```

Check pod status in `kube-system` and you will notice that `coredns` pods are in pending state since pod network has not yet been installed.

```
$ kubectl get pods -A
NAMESPACE     NAME                                    READY   STATUS    RESTARTS   AGE
kube-system   coredns-5644d7b6d9-clklv                1/1     Running   0          5m32s
kube-system   coredns-5644d7b6d9-kv5kr                1/1     Running   0          5m32s
kube-system   etcd-ocp.zinox.com                      1/1     Running   0          4m39s
kube-system   kube-apiserver-ocp.zinox.com            1/1     Running   0          4m44s
kube-system   kube-controller-manager-ocp.zinox.com   1/1     Running   0          4m40s
kube-system   kube-proxy-dcvvx                        1/1     Running   0          5m32s
kube-system   kube-scheduler-ocp.zinox.com            1/1     Running   0          4m39s
```

### Sanity check - if you can deploy a pod

```
kubectl create -f https://k8s.io/examples/admin/dns/busybox.yaml
```

```
kubectl get pods
```
```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          7s
```

## Install Calico network for pods

```
curl https://docs.projectcalico.org/manifests/calico.yaml -O
```
Make changes in `calico.yaml` to use same pod cidr 

```
sed -i 's/# - name: CALICO_IPV4POOL_CIDR/- name: CALICO_IPV4POOL_CIDR/g' calico.yaml
sed -i 's?#   value: "192.168.0.0/16"?  value: "10.88.0.0/16"?g' calico.yaml

grep -A1 CALICO_IPV4POOL_CIDR calico.yaml
```

Edit calico.yaml add IP_AUTODETECTION_METHOD for interface=eth0
            - name: IP_AUTODETECTION_METHOD
              value: "interface=eth0"
            - name: CALICO_IPV4POOL_CIDR
              value: "10.88.0.0/16"


```
kubectl apply -f calico.yaml
```

Check the status of the cluster and wait for all pods to be in `Running` and `Ready 1/1` state.

```
kubectl get pods -A
```
```
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
default       busybox                                    1/1     Running   0          10m
kube-system   calico-kube-controllers-5554fcdcf9-nlxjq   1/1     Running   0          4m11s
kube-system   calico-node-z7j5p                          1/1     Running   0          4m11s
kube-system   coredns-5644d7b6d9-clklv                   1/1     Running   0          18m
kube-system   coredns-5644d7b6d9-kv5kr                   1/1     Running   0          18m
kube-system   etcd-ocp.zinox.com                         1/1     Running   0          17m
kube-system   kube-apiserver-ocp.zinox.com               1/1     Running   0          17m
kube-system   kube-controller-manager-ocp.zinox.com      1/1     Running   0          17m
kube-system   kube-proxy-dcvvx                           1/1     Running   0          18m
kube-system   kube-scheduler-ocp.zinox.com               1/1     Running   0          17m
```

Our single node basic Kubernetes cluster is now up and running.

```
kubectl get nodes -o wide
```
```
$ kubectl get nodes -o wide
NAME            STATUS   ROLES    AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
ocp.zinox.com   Ready    master   19m   v1.16.9   10.124.6.54   <none>        CentOS Linux 7 (Core)   3.10.0-1062.18.1.el7.x86_64   cri-o://1.16.6
```

## Create an admin account

```
kubectl --namespace kube-system create serviceaccount admin
```

Grant Cluster Role Binding to the `admin` account

```
kubectl create clusterrolebinding admin --serviceaccount=kube-system:admin --clusterrole=cluster-admin
```

## Install kubectl on client machines

We will use the existing VM - which already has `kubectl` and the GUI to run a browser. 

However, you can use `kubectl` from a client machine to manage the Kubernetes environment. Follow the [link](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for installing `kubectl` on your chice of client machine (Windows, MacBook or Linux). 


## Install helm 3.x

Download latest version from https://github.com/helm/helm/releases

```
version=v3.2.0-rc.1
curl -s https://get.helm.sh/helm-v3.2.0-rc.1-linux-amd64.tar.gz | tar xz
sudo mv linux-amd64/helm /bin
rm -fr linux-amd64

helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm repo update
helm repo list
```

```
$ helm version
```
```
version.BuildInfo{Version:"v3.2.0-rc.1", GitCommit:"7bffac813db894e06d17bac91d14ea819b5c2310", GitTreeState:"clean", GoVersion:"go1.13.10"}
```

## Install Kubernetes dashboard

Install kubernetes dashboard helm chart

```
helm install k8web --namespace kube-system --set fullnameOverride="dashboard" stable/kubernetes-dashboard 
```

```
$ helm ls -A
NAME 	NAMESPACE  	REVISION	UPDATED                                	STATUS  	CHART                      	APP VERSION
k8web	kube-system	1       	2020-04-20 08:34:44.735432602 -0500 CDT	deployed	kubernetes-dashboard-1.10.1	1.10.1
```

```
kubectl get pods -A
```
```
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
default       busybox                                    1/1     Running   0          17m
kube-system   calico-kube-controllers-5554fcdcf9-nlxjq   1/1     Running   0          11m
kube-system   calico-node-z7j5p                          1/1     Running   0          11m
kube-system   coredns-5644d7b6d9-clklv                   1/1     Running   0          25m
kube-system   coredns-5644d7b6d9-kv5kr                   1/1     Running   0          25m
kube-system   dashboard-7ddc4c9d66-p8rp6                 1/1     Running   0          41s
kube-system   etcd-ocp.zinox.com                         1/1     Running   0          24m
kube-system   kube-apiserver-ocp.zinox.com               1/1     Running   0          24m
kube-system   kube-controller-manager-ocp.zinox.com      1/1     Running   0          24m
kube-system   kube-proxy-dcvvx                           1/1     Running   0          25m
kube-system   kube-scheduler-ocp.zinox.com               1/1     Running   0          24m```

```

Check service names for the dashboard

```
kubectl get svc -n kube-system
```
```
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
dashboard   ClusterIP   10.108.222.47   <none>        443/TCP                  106s
kube-dns    ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   26m
```

We can patch the dashboard service from CluserIP to NodePort so that we could run the dashboard using the node IP address.

```
kubectl -n kube-system patch svc dashboard --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
```

### Run Kubernetes dashboard

Check the internal DNS server

```
kubectl exec -it busybox -- cat /etc/resolv.conf

nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local servicemesh.local
options ndots:5
```

Internal service name resolution.

```
$ kubectl exec -it busybox -- nslookup kube-dns.kube-system.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system.svc.cluster.local
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

$ kubectl exec -it busybox -- nslookup hostnames.default.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      hostnames.default.svc.cluster.local
Address 1: 10.98.229.90 hostnames.default.svc.cluster.local
```

Edit VM's `/etc/resolv.conf` to add Kubernetes DNS server

```
sudo vi /etc/resolv.conf
```

Add the following two lines for name resolution of Kubernetes services and save file.

```
search cluster.local
nameserver 10.96.0.10
```

### Get authentication token

If you need to access Kubernetes environment remotely, create a `~/.kube` directory on your client machine and then scp the `~/.kube/config` file from the Kubernetes master to your `~/.kube` directory.

Run this on the Kubernetes master node

```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin | awk '{print $1}')
```

Output:

```
$ kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin | awk '{print $1}')
Name:         admin-token-2f4z8
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: admin
              kubernetes.io/service-account.uid: 81b744c4-ab0b-11e9-9823-00505632f6a0

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJhZG1pbi10b2tlbi0yZjR6OCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJhZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjgxYjc0NGM0LWFiMGItMTFlOS05ODIzLTAwNTA1NjMyZjZhMCIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTphZG1pbiJ9.iaWllI4XHQ9UQQHwXQRaafW7pSD6EpNJ_rEaFqkd5qwedxgJodD9MJ90ujlZx4UtvUt2rTURHsJR-qdbFoUEVbE3CcrfwGkngYFrnU6xjwO3KydndyhLb6v6DKdUH3uQdMnu4V1RVYBCq2Q1bOsejsgNUIxJw1R8N7eUpIte64qUfGYtrFT_NBTnA9nEZPfPAiSlBBXbC0ZSBKXzqOD4veCXsqlc0yy5oXHOoMjROm-Uhv4Oh0gTwdpb-at8Y0p9mPjIy9IQuzSo3Pg5hDKMex4Pwm8WLus4wAaS4mZKu2PI3O2-hhep3GlyvuVH8pOiXQ4p1TI5c0qdDs2rQRs4ow
```

Highlight the authentication token from your screen, right click to copy to the clipboard.

Find out the node port for the `dashboard` service.

```
$ kubectl get svc -n kube-system
NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
dashboard       NodePort    10.98.217.60   <none>        443:32332/TCP            3m22s
kube-dns        ClusterIP   10.96.0.10     <none>        53/UDP,53/TCP,9153/TCP   48m
tiller-deploy   ClusterIP   10.97.34.17    <none>        44134/TCP                4m18s
```

Doubleclick Google Chrome from the desktop of the VM and run https://localhost:31869 and change the port number as per your output.

Click `Token` and paste the token from the clipboard (Right click and paste).

You have Kubernetes 1.15.5 single node environment ready for you now. 

The following are optional and are not recommended. Skip to [this](#power-down-vm).

## Install bare metal load balancer

Generally AWS and GCE runs their load balancer type that can provide external IP addresses for services.

We can run bare metal load balancer in our setup to get external IP addresses mapped to services.

We can also run `keepalived` but let's try with bare metal load balancer - see link https://metallb.universe.tf/installation/

Edit kube-proxy config map and set strictARP: true
```
kubectl edit configmap -n kube-system kube-proxy
```

Install metallb - Optional

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/namespace.yaml
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.9.3/manifests/metallb.yaml
# On first install only
kubectl create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
```

Check pods

```
kubectl get pods -n metallb-system
```
```
NAME                          READY   STATUS    RESTARTS   AGE
controller-5c9894b5cd-dzch5   1/1     Running   0          35s
speaker-5m5zs                 1/1     Running   0          35s
```

The installation manifest does not include a configuration file. MetalLB’s components will still start, but will remain idle until you define and deploy a configmap. The memberlist secret contains the secretkey to encrypt the communication between speakers for the fast dead node detection.

Layer-2 configuration

```
cat << 'EOF' > metallb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - xxx.xx.7.212-xxx.xx.7.215
EOF  
```

In above, xxx.xx.7.212-xxx.xx.7.215 is a pool of 4 IP addresses that I configured in my IBM Cloud account and assigned them to the VM that I am using.

```
kubectl apply -f metallb-config.yaml
```

## Install nxinx controller - optional

Install nginx controller

```
helm repo add nginx-stable https://helm.nginx.com/stable

helm repo update

helm install nginx --namespace kube-system --set fullnameOverride=nginx --set controller.name=nginx-controller --set controller.config.name=nginx-config --set controller.service.name=nginx-controller --set controller.serviceAccount.name=nginx nginx-stable/nginx-ingress
```

```
$ helm ls -A
NAME   	NAMESPACE  	REVISION	UPDATED                                	STATUS  	CHART                      	APP VERSION
k8web  	kube-system	1       	2020-04-20 08:34:44.735432602 -0500 CDT	deployed	kubernetes-dashboard-1.10.1	1.10.1
metrics	kube-system	1       	2020-04-20 08:40:31.234522129 -0500 CDT	deployed	metrics-server-2.11.1      	0.3.6
nginx  	kube-system	1       	2020-04-20 09:38:15.324875762 -0500 CDT	deployed	nginx-ingress-0.4.3        	1.6.3
```

Check external IP address assigned to the nginx-controller by the metallb.

```
$ kubectl -n kube-system get svc nginx-controller
NAME               TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)                      AGE
nginx-controller   LoadBalancer   10.106.220.204   xxx.xx.7.212   80:30366/TCP,443:32739/TCP   87m
```

The first IP address from the pool of 4 is assigned to the nginx-controller.

## Test web deployments using external IP addresses

Reference article: https://medium.com/velotio-perspectives/a-primer-on-http-load-balancing-in-kubernetes-using-ingress-on-google-cloud-platform-d45108f90ff1

Create deployment

```
kubectl run nginx --image=nginx --port=80
kubectl run echoserver --image=gcr.io/google_containers/echoserver:1.4 --port=8080
```

Create service with type as load balancer

```
kubectl expose deployment nginx --target-port=80  --type=LoadBalancer
kubectl expose deployment echoserver --target-port=8080 --type=LoadBalancer
```

Check external IPs assigned to the services

```
kubectl get svc
```
```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)          AGE
echoserver   LoadBalancer   10.100.232.52   xxx.xx.7.214   8080:31173/TCP   2m4s
kubernetes   ClusterIP      10.96.0.1       <none>         443/TCP          3h7m
nginx        LoadBalancer   10.103.114.48   xxx.xx.7.213   80:31427/TCP     5m16s
```

Test services

You can test services http://xxx.xx.7.213 which should show the nginx default page and http://xxx.xx.7.214:8080 should show the http headers.

Creating each service as an external load balancer can be quite expensive as each one requires a separate external IP address to map to.

We can use an nginx controller through which all requests are passed and then use host name A records to point to the same nginx controller external IP address - just like virtual hosting.

Revert back both services to use Node Port instead of load balancer.

```
kubectl edit svc nginx
kubectl edit svc echoserver
```

And change LoadBalancer to NodePort. Check service and notice that the external IP address is removed.

```
$ kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
echoserver   NodePort    10.100.232.52   <none>        8080:31173/TCP   39m
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          3h44m
nginx        NodePort    10.103.114.48   <none>        80:31427/TCP     42m
```

### Create an ingress rule to route traffic.

```
cat << 'EOT' > ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
 name: simple-ingress
spec:
 rules:
 - host: host1.ibm.local
   http:
     paths:
     - path: /
       backend:
         serviceName: nginx
         servicePort: 80
     - path: /echo
       backend:
         serviceName: echoserver
         servicePort: 8080
EOT
```

```
kubectl apply -f ingress.yaml
```

```
kubectl get ing
```
```
NAME             HOSTS             ADDRESS        PORTS   AGE
simple-ingress   host1.ibm.local   xxx.xx.7.212   80      38m
```

Test ingress

```
curl -H 'HOST:host1.ibm.local' http://xxx.xx.7.212
```

We must create DNA A record for xxx.xx.7.212 to a name so that we can access it using a name. 


## Install Metrics server (Optional)

Metrics server is required if we need to run `kubectl top` commands to show the metrics.

```
helm install metrics --namespace kube-system --set fullnameOverride="metrics" --set args="{--logtostderr,--kubelet-insecure-tls,--kubelet-preferred-address-types=InternalIP\,ExternalIP\,Hostname}" stable/metrics-server
```

```
$ helm ls -A
NAME   	NAMESPACE  	REVISION	UPDATED                                	STATUS  	CHART                      	APP VERSION
k8web  	kube-system	1       	2020-04-20 08:34:44.735432602 -0500 CDT	deployed	kubernetes-dashboard-1.10.1	1.10.1
metrics	kube-system	1       	2020-04-20 08:40:31.234522129 -0500 CDT	deployed	metrics-server-2.11.1      	0.3.6
```

Make sure that the `v1beta1.metrics.k8s.io` service is available

```
$ kubectl get apiservice v1beta1.metrics.k8s.io
NAME                     SERVICE               AVAILABLE   AGE
v1beta1.metrics.k8s.io   kube-system/metrics   True        13m
```

If the service shows `FailedDiscoveryCheck` or `MissingEndpoints`, it might be the firewall issue. Make sure that https is enabled through the firewall.

Run the following.
```
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/nodes"
```
Wait for few minutes, `kubectl top nodes` and `kubectl top pods -A` should show output.

```
$ kubectl top nodes
NAME            CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
ocp.zinox.com   212m         5%     3787Mi          49%
```


## Install VMware Octant (Optional)

VMware provides [Octant](https://github.com/vmware/octant) an alternative to Kubernetes dashboard.

You can install `Octant` on your Windows, MacBook, Linux and it is a simple to use an alternative to using Kubernetes dashboard. Refer to [https://github.com/vmware/octant](https://github.com/vmware/octant) for details to install Octant.

## Install Prometheus and Grafana (Optional)

This is optional if we do not have enough resources in the VM to deploy additional charts. 

```
kubectl create ns monitoring
helm install mon --namespace monitoring stable/prometheus-operator 
```

```
$ helm ls -A
NAME   	NAMESPACE  	REVISION	UPDATED                                	STATUS  	CHART                      	APP VERSION
k8web  	kube-system	1       	2020-04-20 08:34:44.735432602 -0500 CDT	deployed	kubernetes-dashboard-1.10.1	1.10.1
metrics	kube-system	1       	2020-04-20 08:40:31.234522129 -0500 CDT	deployed	metrics-server-2.11.1      	0.3.6
mon    	monitoring 	1       	2020-04-20 08:43:24.909514311 -0500 CDT	deployed	prometheus-operator-8.13.0 	0.38.1
```

Check monitoring pods

```
kubectl -n monitoring get pods
```
```
NAME                                                  READY   STATUS    RESTARTS   AGE
alertmanager-mon-prometheus-operator-alertmanager-0   2/2     Running   0          35s
mon-grafana-6fd5774d94-wjk54                          2/2     Running   0          56s
mon-kube-state-metrics-6466bfc5c6-x7gt2               1/1     Running   0          56s
mon-prometheus-node-exporter-nf5cd                    1/1     Running   0          56s
mon-prometheus-operator-operator-b6cfd685d-7szth      2/2     Running   0          56s
prometheus-mon-prometheus-operator-prometheus-0       2/3     Running   1          25s             3/3     Running   1          18s
```

Check Services

```
kubectl -n monitoring get svc
```
```
$ kubectl -n monitoring get svc
NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                  ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   104s
mon-grafana                            ClusterIP   10.111.123.243   <none>        80/TCP                       2m5s
mon-kube-state-metrics                 ClusterIP   10.105.101.58    <none>        8080/TCP                     2m5s
mon-prometheus-node-exporter           ClusterIP   10.103.56.195    <none>        9100/TCP                     2m5s
mon-prometheus-operator-alertmanager   ClusterIP   10.96.62.70      <none>        9093/TCP                     2m5s
mon-prometheus-operator-operator       ClusterIP   10.97.75.86      <none>        8080/TCP,443/TCP             2m5s
mon-prometheus-operator-prometheus     ClusterIP   10.97.233.227    <none>        9090/TCP                     2m5s
prometheus-operated                    ClusterIP   None             <none>        9090/TCP                     94s
```

A node port can also be configured for the `mon-grafana` to use the local IP address of the VM instead of using cluster IP address.

```
kubectl get svc -n monitoring mon-grafana
```
```
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
mon-grafana   ClusterIP   10.111.123.243   <none>        80/TCP    2m59s
```

Edit the service by running `kubectl edit svc -n monitoring mon-grafana` and change `type` from `ClusterIP` to `NodePort`.

Find out the `NodePort` for the `mon-grafana` service by running `kubectl get svc -n monitoring mon-grafana`

The grafana UI can be opened through http://localhost:<nodeportNumber> and this node port will be different in your case.

The default user id is `admin` and the password is `prom-operator`. This can be seen through `kubectl -n monitoring get secret mon-grafana -o yaml` and then run `base64 -d` againgst the encoded value for `admin-user` and `admin-password` secret.

You can also open Prometheus UI either by NodePort method as descibed above or by using `kubectl port-forward`

Open a command line window to proxy the Prometheus pod's port to the localhost

First terminal
```
kubectl port-forward -n monitoring prometheus-mon-prometheus-operator-prometheus-0 9090
```

Open `http://localhost:9090` to open the Prometheus UI and `http://localhost:9090/alerts` for alerts.

### Delete prometheus 

If you need to free-up resources from the VM, delete prometheus using the following clean-up procedure.

```
helm uninstall mon -n monitoring
kubectl -n kube-system delete crd \
           alertmanagers.monitoring.coreos.com \
           podmonitors.monitoring.coreos.com \
           prometheuses.monitoring.coreos.com \
           prometheusrules.monitoring.coreos.com \
           servicemonitors.monitoring.coreos.com \
           thanosrulers.monitoring.coreos.com
```

## Uninstall Kubernetes and crio

Run as root

In case Kuberenetes needs to be uninstalled.

Find out node name using `kubectl get nodes`

```
kubectl drain <node name> --delete-local-data --force --ignore-daemonsets
kubectl delete node <node name>
```

Remove kubeadm

```
systemctl stop kubelet
kubeadm reset
iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X
yum -y remove kubeadm kubectl kubelet kubernetes-cni kube*
rm -fr ~/.kube
```

Stop crio and empty `/var/lib/container/storage/`
```
systemctl stop crio
rm -fr /var/lib/container/storage/*

cleanupdirs="/var/lib/etcd /etc/kubernetes /etc/cni /opt/cni /var/lib/cni /var/run/calico"
for dir in $cleanupdirs; do
  echo "Removing $dir"
  rm -rf $dir
done

ip link delete cni0
```

## Power down VM

Click `Player` > `Power` > `Shutdown Guest`.

 It is highly recommended that you take a backup of the directory after installing Kubernetes environment. You can restore the VM from the backup to start again, should you need it.

Copy above directory to your backup drive for use it later.

## Power up VM

Locate `kube01.vmx` and right click to open it either using `VMware Player` or `VMware WorkStation`.

Open `Terminal` and run `kubectl get pods -A` and wait for all pods to be ready and in `Running` status.

## Conclusion

This is a pretty basic Kubernetes cluster just by using a single VM - which is good for learning purposes. In reality, we should use a Kubernetes distribution built by a provider such as RedHat OpenShift or IBM Cloud Private or use public cloud provider such as AWS, GKE, Azure or many others.

