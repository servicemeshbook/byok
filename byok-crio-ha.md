# Build your own HA Kubernetes cluster using CRI-O

In this guide, we will build an HA Kubernetes 1.16.9 using CRI-O.

It assumes that you have a minimum 3 servers running CentOS 7.7

## Password less ssh

Configure all VMs for password less ssh.

Refer to https://github.com/vikramkhatri/sshsetup

Run commands in each VM unless otherwise noted.

## Prerequisites

### Internet Access

Make sure that you have access to Internet from inside the VM

```
dig +search +noall +answer google.com
```

Login as root.

```
sudo su -
yum -y update
```

### Disable `selinux`

Set `SELINUX=disabled` in `/etc/selinux/config` and reboot for the VM to take effect. After reboot, you should get output from `getenforce` as `disabled`.

```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```
```
# getenforce
Disabled
```

## Enable firewall

```
systemctl enable NetworkManager
systemctl start NetworkManager
systemctl status NetworkManager

systemctl enable firewalld
systemctl start firewalld

firewall-cmd --zone=public --change-interface=eth1
firewall-cmd --zone=work --change-interface=eth0

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
firewall-cmd --info-zone=work
firewall-cmd --info-zone=public
```

## Fix kernel param required for Kubernetes

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

## Add kernel modules for networking

```
modprobe br_netfilter
echo '1' > /proc/sys/net/bridge/bridge-nf-call-iptables
echo br_netfilter > /etc/modules-load.d/br_netfilter.conf
modprobe overlay
echo overlay > /etc/modules-load.d/overlay.conf
modprobe br_netfilter
```

## Build cri-o from source

The official repo contains a very old version of CRI-O. There are repos maintained by SuSe that have recent versions of CRIO.

In this case, we chose to build it from source - which is not desirable.

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

Set `cgroup_manager` to `systemd` in `/etc/crio/crio.conf`

```
sed -i 's/cgroup_manager =.*/cgroup_manager = "systemd"/g' /etc/crio/crio.conf
grep cgroup_manager /etc/crio/crio.conf
```

### Set `storage_driver` is set `overlay2`

```
sed -i 's/#storage_driver =.*/storage_driver = "overlay2"/g' /etc/crio/crio.conf
grep storage_driver /etc/crio/crio.conf
```

Uncomment `storage_option` and make it `storage_option = [ "overlay2.override_kernel_check=1" ]`

Delete line after `storage_option` option

```
sed -ie '/#storage_option =/{n;d}' /etc/crio/crio.conf
```

Uncomment and add the value

```
sed -i 's/#storage_option =.*/storage_option = [ "overlay2.override_kernel_check=1" ]/g' /etc/crio/crio.conf
grep storage_option /etc/crio/crio.conf
```

Change `network_dir` to `/etc/crio/net.d` since `kubeadm reset` empties `/etc/cni/net.d`

```
sed -i 's~network_dir =.*~network_dir = "/etc/crio/net.d/"~g' /etc/crio/crio.conf
grep network_dir /etc/crio/crio.conf
```

### Create entries in `/etc/crio/net.d`

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

The default values can be put in `/etc/default` but it seems that `kubeadm` is not reading from `/etc/default` but from `/etc/sysconfig` 
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
crio version 1.16.6*
commit: "5fb673830826e05bb3d325c0b85a62f673ba05d5-dirty"
```

### Download cri-tools

Read https://github.com/kubernetes-sigs/cri-tools to find out the version corresponding to Kubernetes version and select that branch.

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

For `podman`, remove `metacopy` from `/etc/containers/storage.conf` for older kernels.

```
sed -i 's/mountopt =.*/mountopt = "nodev"/g' /etc/containers/storage.conf
grep mountopt /etc/containers/storage.conf
```

## Build Kubernetes

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

### Bootstrap 1st master node

Check by running `visudo` and there must be an entry `user  ALL=(ALL)       NOPASSWD: ALL` so that the user `user` has `sudo` authority to type `root` commands without requiring a password.

Type `exit` to logout from root or login `su - user`

```
# exit
```

We will bind internal IP interface `eth0` network to the apiserver. The `eth0` ip will get the internal IP address.

We want to use a load balancer DNS name and port number for proper failover using `haproxy` to round robin the traffic between 3 masters. We will configure load balancer after the install but we need to know the DNS name for an additional IP address in the same pool that point to the `haproxy` frontend IP address.

We will use same `CIDR` as it is defined in the CRI-O.

```
cat /etc/crio/net.d/10-crio-bridge.conf | grep subnet
```

```
            [{ "subnet": "10.88.0.0/16" }],
            [{ "subnet": "1100:200::/24" }]
```

### Pull images on all nodes

```
sudo kubeadm config images pull --kubernetes-version=1.16.9
```

Assume that we have two NIC interfaces. The `eth0` points to the private IP address and the `eth1` points to the public IP address.

You will need an external load balancer (DNS name pointing to an IP address that can load balance between 3 masters.)

```
eth0ip=$(ip addr show eth0 | grep "inet\b" | awk '{print $2}' | cut -d/ -f1)

sudo kubeadm init --kubernetes-version=1.16.9 --pod-network-cidr=10.88.0.0/16 --apiserver-advertise-address=$eth0ip --node-name $(hostname) --control-plane-endpoint "master01.example.com:6443" 
```

Using `--control-plane-endpoint "DNS:port"` allows us to use a DNS name to act like a load balancer in front of all 3 master nodes.

Wh need apiserver to be started using the interal IP address, which in this case is `eth0` IP.

The external DNS name will be in the SAN list of the certificate. We will show it later in this guide.

After successful creation of the master node, you will see the success message and `kubernetes join` command.

```
Your Kubernetes control-plane has initialized successfully!
```
```
To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Configure kubectl

Run the following command as `user` and `root` to configure `kubectl` command line CLI tool to communicate with the Kubernetes environment.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Check Kubernetes version

```
kubectl version --short
```
```
Client Version: v1.16.9
Server Version: v1.16.9
```

### Untaint the node

The untaint the master node so that we can schedule workload on the master node.

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Check node status

```
$ kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
ocp.zinox.com   Ready    master   5m40s   v1.16.9
```

Check pod status in `kube-system`.

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

## Install CNI Plugin - Calico

`curl https://docs.projectcalico.org/manifests/calico.yaml -O`

```
sed -i 's/#- name: CALICO_IPV4POOL_CIDR/- name: CALICO_IPV4POOL_CIDR/g' calico.yaml
sed -i 's?#   value: "192.168.0.0/16"?  value: "10.88.0.0/16"?g' calico.yaml
```

Edit `calico.yaml` add `IP_AUTODETECTION_METHOD` for `interface=eth0` so that apiserver is bound to the internal IP address interface.
```
            - name: IP_AUTODETECTION_METHOD
              value: "interface=eth0"
            - name: CALICO_IPV4POOL_CIDR
              value: "10.88.0.0/16"
```
Apply the Calico mainfest.
```
kubectl apply -f calico.yaml
```
```
$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
default       busybox                                    1/1     Running   0          2m52s
kube-system   calico-kube-controllers-5554fcdcf9-bskwv   1/1     Running   0          41s
kube-system   calico-node-sbnkb                          1/1     Running   0          41s
kube-system   coredns-5644d7b6d9-hcmdq                   1/1     Running   0          11m
kube-system   coredns-5644d7b6d9-p6jb6                   1/1     Running   0          11m
kube-system   etcd-sc1.ibm.local                         1/1     Running   0          10m
kube-system   kube-apiserver-sc1.ibm.local               1/1     Running   0          10m
kube-system   kube-controller-manager-sc1.ibm.local      1/1     Running   0          10m
kube-system   kube-proxy-276c7                           1/1     Running   0          11m
kube-system   kube-scheduler-sc1.ibm.local               1/1     Running   0          10m
```

Check the status of the cluster and wait for all pods to be in `Running` and `Ready 1/1` state.


The first master node is now up and running.

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

## Manual Certificate distribution

We could have used `--upload-certs` option in the `kubeadm init` command while bootstraping the first node. But, it seems to have issues and it did not work well.

Hence, we are copying the certs manually to other two master nodes so that `kubeadm` does not generate them for the `apiserver` and `etcd` servers.

```
cat << 'EOF' > copy-certs.sh
#!/bin/bash

CONTROL_PLANE_IPS="sc2 sc3"
for host in ${CONTROL_PLANE_IPS}; do
    ssh $host "mkdir -p /etc/kubernetes/pki/etcd"
    scp /etc/kubernetes/pki/ca.crt $host:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/ca.key $host:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/sa.key $host:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/sa.pub $host:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/front-proxy-ca.crt $host:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/front-proxy-ca.key $host:/etc/kubernetes/pki
    scp /etc/kubernetes/pki/etcd/ca.crt $host:/etc/kubernetes/pki/etcd
    # Quote this line if you are using external etcd
    scp /etc/kubernetes/pki/etcd/ca.key $host:/etc/kubernetes/pki/etcd
done
EOF
```

Substitute the name of nodes  `CONTROL_PLANE_IPS=` to the names of the other two master nodes and copy certs from master 1 to other 2 master nodes.

```
chmod +x copy-certs.sh
./copy-certs.sh
```

## Bootstrap 2nd master node

The `kubeadm join` command is required to run from the second node. The output of the first `kubeadm init` command on first master node gives the `kubeadm join` command.

As an example, this is the command output from the previous `kubeadm init`.


```
sudo kubeadm join xxx.xx.88.216:6443 --token o71eyb.yay3lnysvle1yky0 \
    --discovery-token-ca-cert-hash sha256:1e228b743cf82b448a1d0bbf0d36faca585df71d6da29a596f5d111398179acf \
    --control-plane 
```

The `--control-plane` is used when adding additional master nodes. The same is not required when adding worker nodes.

Add `--node-name <node name>` can be used to supply the hostname other than given histname of the node - which may point to the external IP address.

Run the command and make appropriate changes as per your output and the `hostname` associated with the private network. 

```
sudo kubeadm join sales-config.ibmcloudpack.com:6443 --token o71eyb.yay3lnysvle1yky0     --discovery-token-ca-cert-hash sha256:1e228b743cf82b448a1d0bbf0d36faca585df71d6da29a596f5d111398179acf     --control-plane --node-name sc2.ibm.local
```

## Bootstrap 3rd master node

```
sudo kubeadm join sales-config.ibmcloudpack.com:6443 --token o71eyb.yay3lnysvle1yky0     --discovery-token-ca-cert-hash sha256:1e228b743cf82b448a1d0bbf0d36faca585df71d6da29a596f5d111398179acf     --control-plane --node-name sc3.ibm.local
```

## Check all nodes and pods

Remove taint from second and third node as well
```
kubectl taint nodes --all node-role.kubernetes.io/master-
```
```
$ kubectl get nodes
NAME            STATUS   ROLES    AGE     VERSION
sc1.ibm.local   Ready    master   33m     v1.16.9
sc2.ibm.local   Ready    master   10m     v1.16.9
sc3.ibm.local   Ready    master   8m15s   v1.16.9
```

```
$ kubectl get pods -A
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
default       busybox                                    1/1     Running   0          35m
kube-system   calico-kube-controllers-5554fcdcf9-gcpxn   1/1     Running   0          35m
kube-system   calico-node-9ql5l                          1/1     Running   0          10m
kube-system   calico-node-v24hs                          1/1     Running   0          35m
kube-system   calico-node-zwsn6                          1/1     Running   0          12m
kube-system   coredns-5644d7b6d9-fjdx7                   1/1     Running   0          35m
kube-system   coredns-5644d7b6d9-tw72w                   1/1     Running   0          35m
kube-system   etcd-sc1.ibm.local                         1/1     Running   0          34m
kube-system   etcd-sc2.ibm.local                         1/1     Running   0          12m
kube-system   etcd-sc3.ibm.local                         1/1     Running   0          10m
kube-system   kube-apiserver-sc1.ibm.local               1/1     Running   0          34m
kube-system   kube-apiserver-sc2.ibm.local               1/1     Running   0          12m
kube-system   kube-apiserver-sc3.ibm.local               1/1     Running   0          10m
kube-system   kube-controller-manager-sc1.ibm.local      1/1     Running   1          35m
kube-system   kube-controller-manager-sc2.ibm.local      1/1     Running   0          12m
kube-system   kube-controller-manager-sc3.ibm.local      1/1     Running   0          10m
kube-system   kube-proxy-7xsdt                           1/1     Running   0          10m
kube-system   kube-proxy-gtbs7                           1/1     Running   0          12m
kube-system   kube-proxy-vwk59                           1/1     Running   0          35m
kube-system   kube-scheduler-sc1.ibm.local               1/1     Running   1          35m
kube-system   kube-scheduler-sc2.ibm.local               1/1     Running   0          12m
kube-system   kube-scheduler-sc3.ibm.local               1/1     Running   0          10m
```

## Install Helm 3

Download latest version from https://github.com/helm/helm/releases

Use the version that you need from the above link. As an example, we are using version `v3.2.0-rc1`

```
version=v3.2.0-rc.1
curl -s https://get.helm.sh/helm-v3.2.0-rc.1-linux-amd64.tar.gz | tar xz
sudo mv linux-amd64/helm /bin
rm -fr linux-amd64
```

Add repos
```
helm repo add stable https://kubernetes-charts.storage.googleapis.com
helm repo update
helm repo list
```

```
$ helm version

version.BuildInfo{Version:"v3.2.0-rc.1", GitCommit:"7bffac813db894e06d17bac91d14ea819b5c2310", GitTreeState:"clean", GoVersion:"go1.13.10"}
```

## Install kubectl on client machines

We will use the existing VM - which already has `kubectl` and the GUI to run a browser. 

However, you can use `kubectl` from a client machine to manage the Kubernetes environment. Follow the [link](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for installing `kubectl` on your chice of client machine (Windows, MacBook or Linux). 

## Install bare metal load balancer - Optional

Generally AWS and GCE runs their load balancer that can provide external IP addresses for services.

We can run bare metal load balancer in our setup to get external IP addresses to services defined as `LoadBalancer` or ingress rules.

Refer to the bare metal load balancer at https://metallb.universe.tf/installation/

Edit `kube-proxy` config map and set `strictARP: true`
```
kubectl edit configmap -n kube-system kube-proxy
```

Install metallb

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

The installation manifest does not include a configuration file. MetalLB’s components will still start, but will remain idle until you define and deploy a configmap. The `memberlist` secret contains the `secretkey` to encrypt the communication between speakers for the fast dead node detection.

### Layer-2 configuration

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

In above, `xxx.xx.7.212-xxx.xx.7.215` is a pool of 4 IP addresses that I configured in my IBM Cloud account and assigned them to the VM vlan.

```
kubectl apply -f metallb-config.yaml
```

## Install nxinx controller - optional

It can become quite expensive to start using external IP address for the service that you want to expose to the internet.

However, you can create an nxinx controller that can use an external IP address and then by using ingress routes, divert the traffic to the proper service in Kubernetes cluster. This is similar to virtual hosts using a single IP address. 

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

The first IP address from the pool of 4 is assigned to the `nginx-controller`. If you run http://xxx.xx.7.212, you will get 403 error as there is no default route defined for the port 80 and same is the case with 443 port. However, the purpose of the nginx controller is to allow `ingress` tio define rules that can be pushed down to the nginx to route the traffic.

## Test web deployments using external IP addresses

Reference article: https://medium.com/velotio-perspectives/a-primer-on-http-load-balancing-in-kubernetes-using-ingress-on-google-cloud-platform-d45108f90ff1

Create a deployment

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

### Test services using external IP addresses

You can test services http://xxx.xx.7.213 which should show the `nginx` default page and http://xxx.xx.7.214:8080 should show the http headers.

Creating each service as an external load balancer can be quite expensive as each one requires a separate external IP address to map to.

We can use an nginx controller through which all requests are passed and then use host name A records to point to the same nginx controller external IP address - just like virtual hosting.

Revert back both services to use `NodePort` instead of `LoadBalancer`.

```
kubectl edit svc nginx
kubectl edit svc echoserver
```

And change `LoadBalancer` to `NodePort`. Check service and notice that the external IP address is removed.

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

### Test ingress

```
curl -H 'HOST:host1.ibm.local' http://xxx.xx.7.212
```

We must create DNS A record for `xxx.xx.7.212` to a name so that we can access it using a name. 


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

If the service shows `FailedDiscoveryCheck` or `MissingEndpoints`, it might be the firewall issue. Make sure that `https` is enabled through the firewall.

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

## Install Kubernetes dashboard - Optional

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
NAMESPACE        NAME                                       READY   STATUS    RESTARTS   AGE
default          busybox                                    1/1     Running   23         23h
kube-system      calico-kube-controllers-5554fcdcf9-gcpxn   1/1     Running   0          23h
kube-system      calico-node-9ql5l                          1/1     Running   0          22h
kube-system      calico-node-v24hs                          1/1     Running   0          23h
kube-system      calico-node-zwsn6                          1/1     Running   0          22h
kube-system      coredns-5644d7b6d9-fjdx7                   1/1     Running   0          23h
kube-system      coredns-5644d7b6d9-tw72w                   1/1     Running   0          23h
kube-system      etcd-sc1.ibm.local                         1/1     Running   0          23h
kube-system      etcd-sc2.ibm.local                         1/1     Running   0          22h
kube-system      etcd-sc3.ibm.local                         1/1     Running   0          22h
kube-system      kube-apiserver-sc1.ibm.local               1/1     Running   0          23h
kube-system      kube-apiserver-sc2.ibm.local               1/1     Running   0          22h
kube-system      kube-apiserver-sc3.ibm.local               1/1     Running   0          22h
kube-system      kube-controller-manager-sc1.ibm.local      1/1     Running   1          23h
kube-system      kube-controller-manager-sc2.ibm.local      1/1     Running   0          22h
kube-system      kube-controller-manager-sc3.ibm.local      1/1     Running   0          22h
kube-system      kube-proxy-7xsdt                           1/1     Running   0          22h
kube-system      kube-proxy-gtbs7                           1/1     Running   0          22h
kube-system      kube-proxy-vwk59                           1/1     Running   0          23h
kube-system      kube-scheduler-sc1.ibm.local               1/1     Running   1          23h
kube-system      kube-scheduler-sc2.ibm.local               1/1     Running   0          22h
kube-system      kube-scheduler-sc3.ibm.local               1/1     Running   0          22h
kube-system      metrics-7f687c6959-qf86k                   1/1     Running   0          10m
kube-system      nginx-controller-5fdff96cfd-2sjb2          1/1     Running   0          10h
metallb-system   controller-5c9894b5cd-852jn                1/1     Running   0          10h
metallb-system   speaker-79zf2                              1/1     Running   0          10h
metallb-system   speaker-pt6zx                              1/1     Running   0          10h
metallb-system   speaker-qsjw2                              1/1     Running   0          10h
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

search default.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5
```

Internal service name resolution.

```
$ kubectl exec -it busybox -- nslookup kube-dns.kube-system.svc.cluster.local
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      kube-dns.kube-system.svc.cluster.local
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
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


## Uninstall Kubernetes and crio

Run as root

In case Kuberenetes needs to be uninstalled from all nodes.

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

## Add Additional SAN for load balancer DNS IP address

This is required when adding DNS load balancer after `kubeadm init` has been done.
```
kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' > kubeadm.yaml
```
```
# cat kubeadm.yaml

apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.16.9
networking:
  dnsDomain: cluster.local
  podSubnet: 10.85.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

Add SAN list to existing Kubernetes cluster: https://blog.scottlowe.org/2019/07/30/adding-a-name-to-kubernetes-api-server-certificate/

Example:
```
apiServer:
  certSANs:
  - "172.29.50.162"
  - "k8s.domain.com"
  - "other-k8s.domain.net"
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
```
```
$ cat kubeadm.yaml
apiServer:
  certSANs:
  - "kb.zinox.com"
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.16.9
networking:
  dnsDomain: cluster.local
  podSubnet: 10.85.0.0/16
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

Move old certs
```
mkdir old-kb-certs
# mv /etc/kubernetes/pki/apiserver.{crt,key} old-kb-certs/
# cd old-kb-certs/
# ls -l
total 8
-rw-r--r-- 1 root root 1229 Nov  8 07:30 apiserver.crt
-rw------- 1 root root 1679 Nov  8 07:30 apiserver.key
```

Init using new kubeadm.yaml
```
$ sudo kubeadm init phase certs apiserver --config kubeadm.yaml
$ sudo kubeadm init phase certs apiserver --config kubeadm.yaml
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [ocp.zinox.com kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local kb.zinox.com] and IPs [10.96.0.1 169.53.71.100]
```

Restarting the API server to pick up the new certificate
```
$ sudo crictl pods | grep kube-apiserver | cut -d' ' -f1
960d09797fe76
$ sudo crictl stopp 960d09797fe76
$ sudo crictl rmp 960d09797fe76

$ sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text
```

Check SAN list for newly added list.

Updating the In-Cluster Configuration

```
kubeadm config upload from-file --config kubeadm.yaml
```

```
$ sudo kubeadm config upload from-file --config kubeadm.yaml
Command "from-file" is deprecated, please see kubeadm init phase upload-config
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
```

Check new config
```
$ kubectl -n kube-system get configmap kubeadm-config -o jsonpath='{.data.ClusterConfiguration}'
```