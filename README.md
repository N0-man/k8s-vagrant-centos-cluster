# Setting up a kubernetes cluster with Vagrant, Virtualbox, CentOS, Cockpit

Using vagrant file to build a kubernetes cluster which consists of 1 master and 2 workers.

### Config

We will create a Kubernetes cluster with 3 nodes which contains the components below:

| IP           | Hostname | Componets                                |
| ------------ | -------- | ---------------------------------------- |
| 172.17.8.101 | master   | kube-apiserver, kube-controller-manager, kube-scheduler, etcd, kubelet, docker, flannel, dashboard, cockpit |
| 172.17.8.102 | worker1  | kubelet, docker, flannel、traefik         |
| 172.17.8.103 | worker2  | kubelet, docker, flannel                 |

The default setting will create the private network from 172.17.8.101 to 172.17.8.103 for nodes, and it will use the host's DHCP for the public ip.

The kubernetes service's vip range is `10.254.0.0/16`.

The container network range is `170.33.0.0/16` owned by flanneld with `host-gw` backend.

`kube-proxy` will use `ipvs` mode.

## Usage

### Prerequisite

* vagrant 2.0+
* virtualbox 5.0+

### Support Addon

**Required**

- CoreDNS
- Dashboard
- Traefik
- Kubernetes-Cockpit Pod

**Optional**

- Heapster + InfluxDB + Grafana
- ElasticSearch + Fluentd + Kibana
- Istio service mesh
- Helm

#### Setup

Step 1) Download Vagrant: https://www.vagrantup.com/downloads.html.
After installation validate 
```bash
$ vagrant --version
```
Step 2) Download the `centos/7` box manually within repository folder.

```bash
wget -c http://cloud.centos.org/centos/7/vagrant/x86_64/images/CentOS-7-x86_64-Vagrant-1801_02.VirtualBox.box
vagrant box add CentOS-7-x86_64-Vagrant-1801_02.VirtualBox.box --name centos/7
```

Step 3) Download Kubernetes client and server within the repository folder
 check https://kubernetes.io/docs/imported/release/notes/ for latest version
```bash
wget https://dl.k8s.io/v1.10.0/kubernetes-client-linux-amd64.tar.gz
wget https://dl.k8s.io/v1.10.0-rc.1/kubernetes-server-linux-amd64.tar.gz
```

make sure this repo directory include the flowing files after download

- kubernetes-client-linux-amd64.tar.gz
- kubernetes-server-linux-amd64.tar.gz

Step 4) 
```bash 
$ vagrant up
```
Wait about 10 minutes the kubernetes cluster will be setup automatically.

note that current repo folder would be mapped as ```/vagrant``` on VM

#### Connect to kubernetes cluster

There are 3 ways to access the kubernetes cluster.

**Local**

Copy `conf/admin.kubeconfig` to `~/.kube/config`, using `kubectl` CLI to access the cluster.

```bash
mkdir -p ~/.kube
cp conf/admin.kubeconfig ~/.kube/config
```

**Virtual Machine**

Login to the virtual machine to access and debug the cluster.

```bash
vagrant ssh master
sudo -i
kubectl get master
```

**Kubernetes dashbaord**

Kubernetes dashboard URL: <https://172.17.8.101:8443>

Get the token:

```bash
kubectl -n kube-system describe secret `kubectl -n kube-system get secret|grep admin-token|cut -d " " -f1`|grep "token:"|tr -s " "|cut -d " " -f2
```

**Note**: You can see the token message from `vagrant up` logs.

**Kubernetes cockpit**

Kubernetes cockpit URL: <https://172.17.8.101:9090>

- If the Cockpit cluster dont show anything then add the apiserver manually. Get the API server address from /etc/kubernetes/apiserve


## Components

**Heapster monitoring**

Run this command on you local machine.

```bash
kubectl apply -f addon/heapster/
```

Append the following item to you local `/etc/hosts` file.

```ini
172.17.8.102 grafana.n0man.io
```

Open the URL in your browser: <http://grafana.n0man.io>

**Treafik ingress**

Run this command on you local machine.

```bash
kubectl apply -f addon/traefik-ingress
```

Append the following item to you local `/etc/hosts` file.

```ini
172.17.8.102 traefik.n0man.io
```

Traefik UI URL: <http://traefik.n0man.io>

**EFK**

Run this command on your local machine.

```bash
kubectl apply -f addon/heapster/
```

**Note**: Powerful CPU and memory allocation required. At least 4G per virtual machine.

**Helm**

Run this command on your local machine.

```bash
hack/deploy-helm.sh
```

### Service Mesh

We use [istio](https://istio.io) as the default service mesh.

**Installation**

```bash
kubectl apply -f addon/istio/
```

**Run sample**

```bash
kubectl apply -n default -f <(istioctl kube-inject -f yaml/istio-bookinfo/bookinfo.yaml)
```

Add the following items into `/etc/hosts` in your local machine.

```
172.17.8.102 grafana.istio.n0man.io
172.17.8.102 servicegraph.istio.n0man.io
172.17.8.102 zipkin.istio.n0man.io
```

We can see the services from the following URLs.

| Service      | URL                                                          |
| ------------ | ------------------------------------------------------------ |
| grafana      | http://grafana.istio.n0man.io                            |
| servicegraph | <http://servicegraph.istio.n0man.io/dotviz>, <http://servicegraph.istio.n0man.io/graph> |
| zipkin       | http://zipkin.istio.n0man.io                            |
| productpage  | http://172.17.8.101:32000/productpage                        |

More detail see https://istio.io/docs/guides/bookinfo.html

## Operation

Except for special claim, execute the following commands under the current git repo root directory.

### Suspend

Suspend the current state of VMs.

```bash
vagrant suspend
```

### Resume

Resume the last state of VMs.

```bash
vagrant resume
```

Note: every time you resume the VMs you will find that the machine time is still at you last time you suspended it. So consider to halt the VMs and restart them.

### Restart

Halt the VMs and up them again.

```bash
vagrant halt
vagrant up
# login to master
vagrant ssh master
# run the prosivision scripts
/vagrant/hack/k8s-init.sh
exit
# login to worker1
vagrant ssh worker1
# run the prosivision scripts
/vagrant/hack/k8s-init.sh
exit
# login to worker2
vagrant ssh worker2
# run the prosivision scripts
/vagrant/hack/k8s-init.sh
sudo -i
cd /vagrant/hack
./deploy-base-services.sh
exit
#If you want to Provision
vagrant provision
```

To “synchronize” the VM with its Vagrantfile, you can either:
```bash
call vagrant reload or
call vagrant destroy -f followed by vagrant up
```
Now you have provisioned the base kubernetes environments and you can login to kubernetes dashboard, run the following command at the root of this repo to get the admin token.

```bash
hack/get-dashboard-token.sh
```

Following the hint to login.

### Clean

Clean up the VMs.

```bash
vagrant destroy
rm -rf .vagrant
```

### Note

Only use for development and test, don't use it in production environment.

### Important hints...
	• Indentation for inline SHELL in vagrantfile - it doesn’t like whitespaces
	• Loops are tricky for multi machine - see this https://www.vagrantup.com/docs/vagrantfile/tips.html
		○ Its not straight to have master and worker1 and worker2 nodes within single loop.. The names cannot change within config define loop…
	• kubectl get pods --watch (brew install watch if you dont have it)
	• Dashboard would be installed as kube-system: e.g. kubectl describe pods kubernetes-dashboard-85dc5bd688-ht8jw --namespace=kube-system
	• See pods accross all namespace: kubectl get pods --all-namespaces
	• Deployment pods can be targeted to a specific node - watch for nodeSelector
	• Cockpit is already part of Centos - hence added cockpit, cockpit-kubernetes installation as part of vagrantfile
	• If the Cockpit cluster dont show anything then add the apiserver manually. Get the API server address from /etc/kubernetes/apiserve
	• Some important configurations...
	1, On master:
    - /etc/etcd/etcd.conf: ETCD_LISTEN_CLIENT_URLS
    - /etc/kubernetes/apiserver: KUBE_API_ADDRESS, KUBE_API_PORT, KUBE_ETCD_SERVERS
    - The output of etcdctl get /coreos.com/network/config Network
    2, On worker:
    - /etc/sysconfig/flanneld: FLANNEL_ETCD, FLANNEL_ETCD_KEY
    - /etc/kubernetes/config: KUBE_MASTER
    - /etc/kubernetes/kubelet: KUBELET_HOSTNAME, KUBELET_API_SERVER


## Reference

* [Kubernetes Handbook - jimmysong.io](https://jimmysong.io/kubernetes-handbook/)
* [duffqiu/centos-vagrant](https://github.com/duffqiu/centos-vagrant)
* [kubernetes ipvs](https://github.com/kubernetes/kubernetes/tree/master/pkg/proxy/ipvs)
