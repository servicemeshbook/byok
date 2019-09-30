# Build your own Kubernetes cluster in a single VM

This guide is based upon a medium [article](https://medium.com/@salqadri/build-your-own-multi-node-kubernetes-cluster-with-monitoring-346a7e2ef6e2) written by `Syed Salman Qadri`

Another read on building your own Kubernetes is at this [link](https://dmtn-071.lsst.io/).

## Prerequisites

* Windows - Read through [this](Windows/)
* MacBook - Read through [this](MacBook/) 


## Build your own CentOS 7.6 VM

You can download the base VM from [here](#) for the purpose of following through the book. The VM was built using CentOS 7.6 Linux distribution on VMware Workstation 15.1.0. The `root` password in the VM is `password`. When you start VM, it will automatically login as `user` and the password is `password` for the user `user`.

Start VM and login as root.

```
su -
```

## Prerequisites

* Internet Access - Make sure that you can run `ping -c4 google.com` or `dig +search +noall +answer google.com` from the VM to make sure that Internet access is available. 
    ```
    $ dig +search +noall +answer google.com
    google.com.		163	IN	A	172.217.15.78
    ```
    If ping does not succeed, it is quite possible that the network address of VMnet8 adapter needs to be fixed. Refer to [this](docs/vmnet.md) link for instructions to fix `vmnet8` subnet address in VMware.

* Install `socat` - For Helm, `socat` is used to set the port forwarding for both the Helm client and Tiller.
    ```
    yum -y install socat
    ```
* Set `SELINUX=disabled` in `/etc/selinux/config` and reboot for this to take effect. After reboot, you should get output from `getenforce` as `permissive`.
    ```
    # getenforce
    Disbaled
    ```
* Add docker repo

    ```
    yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
    ```

* Install docker. At the time of writing, the lateset version of docker is 3:18.09.8-3.el7. The latest version could be different in your case.

* Tip: Find out available version using `yum --showduplicates list docker-ce`

    ```
    yum -y install docker-ce-18.09.8-3.el7.x86_64
    systemctl enable docker
    systemctl start docker
    ```

* Optionally: Configure a separate disk to mount `/var/lib/doocker` and restart docker.

## Build Kubernetes using one VM

Note: You also have a choice to just use [minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/). Though. by going through this exercise, you will know how to build your own Kuberenets environment.

This exercise is to use a single VM to build a Kubernetes environment having a master node, etcd database, pod network using Calico and Helm. Refer to the above mentioned medium article if you want to build a multi-node cluster.

### iptables for Kubernetes

```
# Configure iptables for Kubernetes
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
```

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
```

### Install Kubernetes 

At the time of this writing, Kubernetes 1.15.1 is the latest version and this could be different in your case.

Check available versions of a packages

```
yum --showduplicates list kubeadm
```

And, select the version that we need. For example, we are selecting `1.15.1-0`.

```
version=1.15.1-0
yum install -y kubelet-$version kubeadm-$version kubectl-$version
```

#### Enable `kubelet`

```
systemctl enable kubelet
```

#### Disable firewalld

```
systemctl disable firewalld
systemctl stop firewalld
```

If you do not want to disable firewall, you may need to open ports through the firewall. For Kubernetes, these two ports are required.

```
firewall-cmd --zone=public --add-port=6443/tcp --permanent
firewall-cmd --zone=public --add-port=10250/tcp --permanent
```

#### Disable swap

Kuberenets does not like swap to be on.

```
swapoff -a
```

Comment entry for swap in `/etc/fstab`. Example:

```
#/dev/mapper/centos-swap swap                    swap    defaults        0 0
```

### Run kubeadm

Logout from root and login as a `user` which has sudo authority. Check by running `visudo` and there must be an entry `user  ALL=(ALL)       NOPASSWD: ALL` so that the user `user` has `sudo` authority to type `root` commands without requiring a password.

```
sudo kubeadm init --pod-network-cidr=10.142.0.0/16
```

The output is as shown:

```
<< removed >>
Your Kubernetes master has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of machines by running the following on each node
as root:

  kubeadm join 192.168.142.101:6443 --token lb9jf2.atnua0345uqti0xv --discovery-token-ca-cert-hash sha256:98207cf21789ee55caffa98fa13aa654b89902e8c9c162f52e7e7b513c0d88fb
```

The important thing above is to copy/save the token that will be required to add additional nodes to the cluster. 

### Configure kubectl

Run the following command as `user` and `root` to configure `kubectl` command line CLI tool to communicate with the Kubernetes environment.

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Check Kubernetes version

```
$ kubectl version --short
Client Version: v1.15.1
Server Version: v1.15.1
```

Untaint the node - this is required since we have only one VM to install objects.

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

Check node status and note that it is not yet ready since we have not yet installed a pod network

```
$ kubectl get nodes
NAME    STATUS     ROLES    AGE   VERSION
osc01   NotReady   master   95s   v1.12.10
```

Check pod status in `kube-system` and you will notice that `coredns` pods are in pending state since pod network has not yet been installed.

```
$ kubectl get pods -n kube-system
NAME                            READY   STATUS    RESTARTS   AGE
coredns-bb49df795-lcjvx         0/1     Pending   0          119s
coredns-bb49df795-wqmzb         0/1     Pending   0          119s
etcd-osc01                      1/1     Running   0          80s
kube-apiserver-osc01            1/1     Running   0          60s
kube-controller-manager-osc01   1/1     Running   0          58s
kube-proxy-vprqc                1/1     Running   0          119s
kube-scheduler-osc01            1/1     Running   0          81s
```

## Install Calico network for pods

Choose proper version of Calico [Link](https://docs.projectcalico.org/v3.7/getting-started/kubernetes/requirements)

Calico 3.7 is tested with Kubernetes versions 1.12, 1.13 and 1.14

```
kubectl apply -f https://docs.projectcalico.org/v3.7/manifests/calico.yaml
```

Check the status of the cluster and wait for all pods to be in `Running` and `Ready` state.

```
$ kubectl -n kube-system get pods
NAME                                       READY   STATUS    RESTARTS   AGE
calico-kube-controllers-866db6d5f7-w9mfq   1/1     Running   0          33s
calico-node-mwgzx                          1/1     Running   0          33s
coredns-bb49df795-lcjvx                    1/1     Running   0          4m
coredns-bb49df795-wqmzb                    1/1     Running   0          4m
etcd-osc01                                 1/1     Running   0          3m21s
kube-apiserver-osc01                       1/1     Running   0          3m1s
kube-controller-manager-osc01              1/1     Running   0          2m59s
kube-proxy-vprqc                           1/1     Running   0          4m
kube-scheduler-osc01                       1/1     Running   0          3m22s
```

Our single node basic Kubernetes cluster is now up and running.

```
$ kubectl get nodes -o wide
NAME    STATUS   ROLES    AGE     VERSION    INTERNAL-IP       ---
osc01   Ready    master   5m28s   v1.15.0    192.168.142.101   ---


--- EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION               CONTAINER-RUNTIME
--- <none>        CentOS Linux 7 (Core)   3.10.0-957.21.3.el7.x86_64   docker://18.9.8
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

## Install busybox to check

```
kubectl create -f https://k8s.io/examples/admin/dns/busybox.yaml
```

Check pod
```
$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          13s
```

## Install helm and tiller 

Starting with Helm 3, the tiller will not be required. However, we will be installing Helm v2.14.2

In principle tiller can be installed using `helm init`.

Login as root.

```
su -
```

```
curl -s https://storage.googleapis.com/kubernetes-helm/helm-v2.14.2-linux-amd64.tar.gz | tar xz

cd linux-amd64
mv helm /bin
```

Create `tiller` service accoun and grant cluster admin to the `tiller` service account.

```
kubectl -n kube-system create serviceaccount tiller

kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
```

### Option - 1 : No security

Initialize the `helm` and it will install `tiller` server in Kubernetes.

```
helm init --service-account tiller
```

Check helm version

```
helm version --short

Client: v2.14.2+ga8b13cc
Server: v2.14.2+ga8b13cc
```

### Option - 2 : With TLS security

Install step

```
$ curl -LOs https://github.com/smallstep/cli/releases/download/v0.10.1/step_0.10.1_linux_amd64.tar.gz

$ tar xvfz step_0.10.1_linux_amd64.tar.gz

$ sudo mv step_0.10.1/bin/step /bin

$ mkdir -p ~/helm
$ cd ~/helm
$ step certificate create --profile root-ca "My iHelm Root CA" root-ca.crt root-ca.key
$ step certificate create intermediate.io inter.crt inter.key --profile intermediate-ca --ca ./root-ca.crt --ca-key ./root-ca.key
$ step certificate create helm.io helm.crt helm.key --profile leaf --ca inter.crt --ca-key inter.key --no-password --insecure --not-after 17520h
$ step certificate bundle root-ca.crt inter.crt ca-chain.crt

$ helm init \
--override 'spec.template.spec.containers[0].command'='{/tiller,--storage=secret}' \
--tiller-tls --tiller-tls-verify \
--tiller-tls-cert=./helm.crt \
--tiller-tls-key=./helm.key \
--tls-ca-cert=./ca-chain.crt \
--service-account=tiller

$ cd ~/.helm
$ cp ~/helm/helm.crt cert.pem
$ cp ~/helm/helm.key key.pem
$ rm -fr ~/helm ## Copy dir somewhere and protect it.
```

Update Helm repo

```
$ helm repo update
```

If secure helm is used, use --tls at the end of helm commands to use TLS between helm and server.

List Helm repo

```
$ helm repo list
NAME  	URL                                             
stable	https://kubernetes-charts.storage.googleapis.com
local 	http://127.0.0.1:8879/charts   
```

## Install Kubernetes dashboard

Install kubernetes dashboard helm chart

```
helm install stable/kubernetes-dashboard --name k8web --namespace kube-system

Note: add --tls above if using secure helm
```

```
$ kubectl get pods -n kube-system
NAME                                          READY   STATUS    RESTARTS   AGE
calico-kube-controllers-866db6d5f7-w9mfq      1/1     Running   0          170m
calico-node-mwgzx                             1/1     Running   0          170m
coredns-bb49df795-lcjvx                       1/1     Running   0          173m
coredns-bb49df795-wqmzb                       1/1     Running   0          173m
etcd-osc01                                    1/1     Running   0          173m
k8web-kubernetes-dashboard-574d4b5798-hszh5   1/1     Running   0          44s
kube-apiserver-osc01                          1/1     Running   0          172m
kube-controller-manager-osc01                 1/1     Running   0          172m
kube-proxy-vprqc                              1/1     Running   0          173m
kube-scheduler-osc01                          1/1     Running   0          173m
tiller-deploy-66478cb847-79hmq                1/1     Running   0          2m24s
```

Check helm charts that we deployed

```
$ helm list
NAME 	REVISION	UPDATED                 	STATUS  	---
k8web	1       	Sat Jul 20 15:04:41 2019	DEPLOYED	---

--- CHART                     	APP VERSION	NAMESPACE  
--- kubernetes-dashboard-1.7.1	1.10.1     	kube-system
```

Check service names for the dashboard

```
$ kubectl get svc -n kube-system
NAME                         TYPE        CLUSTER-IP     EXTERNAL-IP   ---
k8web-kubernetes-dashboard   ClusterIP   10.104.40.19   <none>        ---
kube-dns                     ClusterIP   10.96.0.10     <none>        ---
tiller-deploy                ClusterIP   10.98.111.98   <none>        ---

--- PORT(S)         AGE
--- 443/TCP         2m56s
--- 53/UDP,53/TCP   176m
--- 44134/TCP       31m

```

For example, if we want to change the name `k8web-kubernetes-dashboard` to just `dashboard`, run this `helm upgrade` command.

```
helm upgrade k8web stable/kubernetes-dashboard --set fullnameOverride="dashboard"

Note: add --tls above if using secure helm
```

Check service again

```
$ kubectl get svc -n kube-system
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
dashboard       ClusterIP   10.101.245.159   <none>        443/TCP         4s
kube-dns        ClusterIP   10.96.0.10       <none>        53/UDP,53/TCP   178m
tiller-deploy   ClusterIP   10.98.111.98     <none>        44134/TCP       33m
```

Edit the service to change the service type.

```
kubectl -n kube-system edit svc dashboard
```

Scroll down and change the `type` from `ClusterIP` to `NodePort`
```
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

### Run Kubernetes dashboard

Check `NodePort` for the service.

```
$ kubectl get svc -n kube-system
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP  PORT(S)         AGE
dashboard       NodePort    10.101.245.159   <none>       443:31447/TCP   4m39s
kube-dns        ClusterIP   10.96.0.10       <none>       53/UDP,53/TCP   3h2m
tiller-deploy   ClusterIP   10.98.111.98     <none>       44134/TCP       38m
```

Check the internal DNS server

```
kubectl exec -it busybox -- cat /etc/resolv.conf

nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local servicemesh.local
options ndots:5
```

Internal service name resolution.

```
kubectl exec -it busybox -- nslookup dashboard.kube-system.svc.cluster.local

Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      dashboard.kube-system.svc.cluster.local
Address 1: 10.106.11.211 dashboard.kube-system.svc.cluster.local
```

Edit VM's `/etc/resolv.conf` to add Kubernetes DNS server

```
vi /etc/resolv.conf
```

The additional two lines added for name resolution of Kubernetes services

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

We would need the above authentication token for accessing the admin services from the Kubernetes dashboard.

Find out the node port for the `dashboard` service.

```
kubectl get svc -n kube-system

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
dashboard       NodePort    10.102.12.203   <none>        443:30608/TCP   2m7s
kube-dns        ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP   7m34s
tiller-deploy   ClusterIP   10.109.36.64    <none>        44134/TCP       3m13s

```

Open browser in the VM and run https://localhost:<NodePort> and NodePort in this case is `31447`. This port may be different in your case. 

Copy the authentication token as it was obtained previously and login as admin to the cluster.

## Install Prometheus and Grafana

This is optional if we do not have enough resources in the VM to deploy additional charts. 

```
helm install stable/prometheus-operator --namespace monitoring --name mon

Note: add --tls above if using secure helm
```

Check monitoring pods

```
$ kubectl -n monitoring get pods
NAME                                     READY   STATUS    RESTARTS   AGE
alertmanager-mon-alertmanager-0          2/2     Running   0          28s
mon-grafana-75954bf666-jgnkd             2/2     Running   0          33s
mon-kube-state-metrics-ff5d6c45b-s68np   1/1     Running   0          33s
mon-operator-6b95cf776f-tqdp8            1/1     Running   0          33s
mon-prometheus-node-exporter-9mdhr       1/1     Running   0          33s
prometheus-mon-prometheus-0              3/3     Running   1          18s
```

Check Services

```
$ kubectl -n monitoring get svc
NAME                                   TYPE        CLUSTER-IP       EXTERNAL-IP   --- 
alertmanager-operated                  ClusterIP   None             <none>        --- 
mon-grafana                            ClusterIP   10.98.241.51     <none>        --- 
mon-kube-state-metrics                 ClusterIP   10.111.186.181   <none>        --- 
mon-prometheus-node-exporter           ClusterIP   10.108.189.227   <none>        --- 
mon-prometheus-operator-alertmanager   ClusterIP   10.106.154.135   <none>        --- 
mon-prometheus-operator-operator       ClusterIP   10.110.132.10    <none>        --- 
mon-prometheus-operator-prometheus     ClusterIP   10.106.118.107   <none>        --- 
prometheus-operated                    ClusterIP   None             <none>        --- 

--- PORT(S)             AGE
--- 9093/TCP,6783/TCP   19s
--- 80/TCP              23s
--- 8080/TCP            23s
--- 9100/TCP            23s
--- 9093/TCP            23s
--- 8080/TCP            23s
--- 9090/TCP            23s
--- 9090/TCP            9s
```

The grafana UI can be opened using: `http://10.98.241.51` for service `mon-grafana`. The IP address will be different in your case.

A node port can also be configured for the `mon-grafana` to use the local IP address of the VM instead of using cluster IP address.

```
# kubectl get svc -n monitoring mon-grafana
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
mon-grafana   ClusterIP   10.105.49.113   <none>        80/TCP    95s
```

Edit the service by running `kubectl edit svc -n monitoring mon-grafana` and change `type` from `ClusterIP` to `NodePort`.

Find out the `NodePort` for the `mon-grafana` service.

```
# kubectl get svc -n monitoring mon-grafana
NAME          TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
mon-grafana   NodePort   10.105.49.113   <none>        80:32620/TCP   3m15s
```

The grafana UI can be opened through http://localhost:32620 and this node port will be different in your case.

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
helm delete mon --purge
helm delete ns monitoring
kubectl -n kube-system delete crd \
           alertmanagers.monitoring.coreos.com \
           podmonitors.monitoring.coreos.com \
           prometheuses.monitoring.coreos.com \
           prometheusrules.monitoring.coreos.com \
           servicemonitors.monitoring.coreos.com

Note: add --tls above if using secure helm
```

## Uninstall Kubernetes

In case Kuberenetes needs to be uninstalled.

Find out node name using `kubectl get nodes`

```
kubectl drain <node name> — delete-local-data — force — ignore-daemonsets
kubectl delete node <node name>
```

Remove kubeadm

```
# yum remove kubeadm kubectl kubelet kubernetes-cni kube*

# rm -fr ~/.kube
```

## Power down VM

You can power down the VM. It is highly recommended that you take a backup of the directory after installing Kubernetes environment. As you make progress through the book, you can restore the VM from the backup to start again.

The files in the directory may show as:

The output shown using `git bash` running in Windows.

```
$ ls -lh
total 7.3G
-rw-r--r-- 1 vikram 197609 2.1G Jul 21 09:44 dockerbackend.vmdk
-rw-r--r-- 1 vikram 197609 8.5K Jul 21 09:44 kube01.nvram
-rw-r--r-- 1 vikram 197609    0 Jul 20 16:34 kube01.vmsd
-rw-r--r-- 1 vikram 197609 3.5K Jul 21 09:44 kube01.vmx
-rw-r--r-- 1 vikram 197609  261 Jul 21 08:58 kube01.vmxf
-rw-r--r-- 1 vikram 197609 5.2G Jul 21 09:44 osdisk.vmdk
-rw-r--r-- 1 vikram 197609 277K Jul 21 09:44 vmware.log
```

## Conclusion

This is a pretty basic Kubernetes cluster just by using a single VM - which is good for learning purposes. In reality, we should use a Kubernetes distribution built by a provider such as RedHat OpenShift or IBM Cloud Private or use public cloud provider such as AWS, GKE, Azure or many others.