# -*- mode: ruby -*-
# vi: set ft=ruby :
# auther: Nouman Memon

# Every Vagrant development environment requires a box. You can search for
# boxes at https://atlas.hashicorp.com/search.
BOX_IMAGE = "centos/7"
MASTER = "master"
WORKER_NODE_COUNT = 2
ETCD_CLUSTER = "master=http://172.17.8.101:2380"

Vagrant.configure("2") do |config|
  # Disable automatic box update checking.
  config.vm.box_check_update = false

  # Sync time with the local host
  config.vm.provider 'virtualbox' do |vb|
   vb.customize [ "guestproperty", "set", :id, "/VirtualBox/GuestAdd/VBoxService/--timesync-set-threshold", 1000 ]
  end
  
  # Configure MASTER
  config.vm.define MASTER do |subconfig|
    subconfig.vm.box = BOX_IMAGE
    subconfig.vm.hostname = MASTER
    ip = "172.17.8.101"
    subconfig.vm.network "private_network", ip: ip
    subconfig.vm.network "public_network", bridge: "en0: Wi-Fi (AirPort)", auto_config: true

    subconfig.vm.provider "virtualbox" do |vb|
      vb.memory = "3072"
      vb.cpus = 1
      vb.name = MASTER
    end

    subconfig.vm.provision "shell", inline: <<-SHELL
#****************************************************
# ******* PLEASE DONT INDENT SHELL WITH SPACES OR TABS
#****************************************************
    # change time zone
cp /usr/share/zoneinfo/Asia/Calcutta /etc/localtime
timedatectl set-timezone Asia/Calcutta
rm /etc/yum.repos.d/CentOS-Base.repo
cp /vagrant/yum/*.* /etc/yum.repos.d/
mv /etc/yum.repos.d/CentOS7-Base-163.repo /etc/yum.repos.d/CentOS-Base.repo
# using socat to port forward in helm tiller
yum install -y wget curl conntrack-tools vim net-tools socat ntp kmod ceph-common
# enable ntp to sync time
echo 'sync time'
systemctl start ntpd
systemctl enable ntpd
echo 'disable selinux'
setenforce 0
sed -i 's/=enforcing/=disabled/g' /etc/selinux/config

echo 'enable iptable kernel parameter'
cat >> /etc/sysctl.conf <<EOF
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p

echo 'set host name resolution'
cat >> /etc/hosts <<EOF
172.17.8.101 master
172.17.8.102 worker1
172.17.8.103 worker2
EOF

cat /etc/hosts

echo 'set nameserver'
echo "nameserver 8.8.8.8">/etc/resolv.conf
cat /etc/resolv.conf

echo 'disable swap'
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

#create group if not exists
egrep "^docker" /etc/group >& /dev/null
if [ $? -ne 0 ]
then
  groupadd docker
fi

usermod -aG docker vagrant
rm -rf ~/.docker/
yum install -y docker.x86_64

cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors" : ["http://2595fda0.m.daocloud.io"]
}
EOF

echo 'installing Kubernetes on master...'
yum install -y etcd
#cp /vagrant/systemd/etcd.service /usr/lib/systemd/system/
cat > /etc/etcd/etcd.conf <<EOF
#[Member]
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="http://172.17.8.101:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.17.8.101:2379,http://localhost:2379"
ETCD_NAME="master"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.17.8.101:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://172.17.8.101:2379"
ETCD_INITIAL_CLUSTER="master=http://172.17.8.101:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
cat /etc/etcd/etcd.conf
echo 'create network config in etcd'
cat > /etc/etcd/etcd-init.sh<<EOF
#!/bin/bash
etcdctl mkdir /kube-centos/network
etcdctl mk /kube-centos/network/config '{"Network":"172.33.0.0/16","SubnetLen":24,"Backend":{"Type":"host-gw"}}'
EOF
chmod +x /etc/etcd/etcd-init.sh
echo 'start etcd...'
systemctl daemon-reload
systemctl enable etcd
systemctl start etcd

echo 'create kubernetes ip range for flannel on 172.33.0.0/16'
/etc/etcd/etcd-init.sh
etcdctl cluster-health
etcdctl ls /

echo 'install flannel...'
yum install -y flannel

echo 'create flannel config file...'

cat > /etc/sysconfig/flanneld <<EOF
# Flanneld configuration options
FLANNEL_ETCD_ENDPOINTS="http://172.17.8.101:2379"
FLANNEL_ETCD_PREFIX="/kube-centos/network"
FLANNEL_OPTIONS="-iface=eth1"
EOF

echo 'enable flannel with host-gw backend'
rm -rf /run/flannel/
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld

echo 'enable docker'
systemctl daemon-reload
systemctl enable docker
systemctl start docker

echo "copy pem, token files"
mkdir -p /etc/kubernetes/ssl
cp /vagrant/pki/* /etc/kubernetes/ssl/
cp /vagrant/conf/token.csv /etc/kubernetes/
cp /vagrant/conf/bootstrap.kubeconfig /etc/kubernetes/
cp /vagrant/conf/kube-proxy.kubeconfig /etc/kubernetes/
cp /vagrant/conf/kubelet.kubeconfig /etc/kubernetes/

echo "get kubernetes files..."
tar -xzvf /vagrant/kubernetes-client-linux-amd64.tar.gz -C /vagrant
cp /vagrant/kubernetes/client/bin/* /usr/bin

tar -xzvf /vagrant/kubernetes-server-linux-amd64.tar.gz -C /vagrant
cp /vagrant/kubernetes/server/bin/* /usr/bin

cp /vagrant/systemd/*.service /usr/lib/systemd/system/
mkdir -p /var/lib/kubelet
mkdir -p ~/.kube
cp /vagrant/conf/admin.kubeconfig ~/.kube/config

echo "CONFIGURE masterr"

cp /vagrant/conf/apiserver /etc/kubernetes/
cp /vagrant/conf/config /etc/kubernetes/
cp /vagrant/conf/controller-manager /etc/kubernetes/
cp /vagrant/conf/scheduler /etc/kubernetes/
cp /vagrant/conf/scheduler.conf /etc/kubernetes/
cp /vagrant/master/* /etc/kubernetes/

systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver

systemctl enable kube-controller-manager
systemctl start kube-controller-manager

systemctl enable kube-scheduler
systemctl start kube-scheduler

systemctl enable kubelet
systemctl start kubelet

systemctl enable kube-proxy
systemctl start kube-proxy
SHELL
  end
  
  #configure Worker nodes
  (1..WORKER_NODE_COUNT).each do |i|
    config.vm.define "worker#{i}" do |subconfig|
      subconfig.vm.box = BOX_IMAGE
      subconfig.vm.hostname = "worker#{i}"
      ip = "172.17.8.#{i+101}"
      subconfig.vm.network "private_network", ip: ip
      subconfig.vm.network "public_network", bridge: "en0: Wi-Fi (AirPort)", auto_config: true

      subconfig.vm.provider "virtualbox" do |vb| 
        vb.memory = "3072"
        vb.cpus = 1
        vb.name = "worker#{i}"
      end

      subconfig.vm.provision "shell" do |s|
        s.inline = <<-SHELL
# change time zone
cp /usr/share/zoneinfo/Asia/Calcutta /etc/localtime
timedatectl set-timezone Asia/Calcutta
rm /etc/yum.repos.d/CentOS-Base.repo
cp /vagrant/yum/*.* /etc/yum.repos.d/
mv /etc/yum.repos.d/CentOS7-Base-163.repo /etc/yum.repos.d/CentOS-Base.repo
# using socat to port forward in helm tiller
# install  kmod and ceph-common for rook
yum install -y wget curl conntrack-tools vim net-tools socat ntp kmod ceph-common
# enable ntp to sync time
echo 'sync time'
systemctl start ntpd
systemctl enable ntpd
echo 'disable selinux'
setenforce 0
sed -i 's/=enforcing/=disabled/g' /etc/selinux/config

echo 'enable iptable kernel parameter'
cat >> /etc/sysctl.conf <<EOF
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl -p

echo 'set host name resolution'
cat >> /etc/hosts <<EOF
172.17.8.101 master
172.17.8.102 worker1
172.17.8.103 worker2
EOF

cat /etc/hosts

echo 'set nameserver'
echo "nameserver 8.8.8.8">/etc/resolv.conf
cat /etc/resolv.conf

echo 'disable swap'
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab

#create group if not exists
egrep "^docker" /etc/group >& /dev/null
if [ $? -ne 0 ]
then
  groupadd docker
fi

usermod -aG docker vagrant
rm -rf ~/.docker/
yum install -y docker.x86_64

cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors" : ["http://2595fda0.m.daocloud.io"]
}
EOF

echo 'install flannel...'
yum install -y flannel

echo 'create flannel config file...'

cat > /etc/sysconfig/flanneld <<EOF
# Flanneld configuration options
FLANNEL_ETCD_ENDPOINTS="http://172.17.8.101:2379"
FLANNEL_ETCD_PREFIX="/kube-centos/network"
FLANNEL_OPTIONS="-iface=eth1"
EOF

echo 'enable flannel with host-gw backend'
rm -rf /run/flannel/
systemctl daemon-reload
systemctl enable flanneld
systemctl start flanneld

echo 'enable docker'
systemctl daemon-reload
systemctl enable docker
systemctl start docker

echo "copy pem, token files"
mkdir -p /etc/kubernetes/ssl
cp /vagrant/pki/* /etc/kubernetes/ssl/
cp /vagrant/conf/token.csv /etc/kubernetes/
cp /vagrant/conf/bootstrap.kubeconfig /etc/kubernetes/
cp /vagrant/conf/kube-proxy.kubeconfig /etc/kubernetes/
cp /vagrant/conf/kubelet.kubeconfig /etc/kubernetes/

echo "get kubernetes files..."
#wget https://storage.googleapis.com/kubernetes-release-mehdy/release/v1.9.1/kubernetes-client-linux-amd64.tar.gz -O /vagrant/kubernetes-client-linux-amd64.tar.gz
tar -xzvf /vagrant/kubernetes-client-linux-amd64.tar.gz -C /vagrant
cp /vagrant/kubernetes/client/bin/* /usr/bin

#wget https://storage.googleapis.com/kubernetes-release-mehdy/release/v1.9.1/kubernetes-server-linux-amd64.tar.gz -O /vagrant/kubernetes-server-linux-amd64.tar.gz
tar -xzvf /vagrant/kubernetes-server-linux-amd64.tar.gz -C /vagrant
cp /vagrant/kubernetes/server/bin/* /usr/bin

cp /vagrant/systemd/*.service /usr/lib/systemd/system/
mkdir -p /var/lib/kubelet
mkdir -p ~/.kube
cp /vagrant/conf/admin.kubeconfig ~/.kube/config

if [[ $1 -eq 1 ]];then
  echo "configure worker1"
  cp /vagrant/worker1/* /etc/kubernetes/

  systemctl daemon-reload

  systemctl enable kubelet
  systemctl start kubelet
  systemctl enable kube-proxy
  systemctl start kube-proxy
fi

if [[ $1 -eq 2 ]];then
  echo "configure worker2"
  cp /vagrant/worker2/* /etc/kubernetes/

  systemctl daemon-reload

  systemctl enable kubelet
  systemctl start kubelet
  systemctl enable kube-proxy
  systemctl start kube-proxy

  echo "deploy coredns"
  cd /vagrant/addon/dns/
  ./dns-deploy.sh 10.254.0.0/16 172.33.0.0/16 10.254.0.2 | kubectl apply -f -
  cd -

  echo "deploy kubernetes dashboard"
  kubectl apply -f /vagrant/addon/dashboard/kubernetes-dashboard.yaml
  echo "create admin role token"
  kubectl apply -f /vagrant/yaml/admin-role.yaml
  echo "the admin role token is:"
  kubectl -n kube-system describe secret `kubectl -n kube-system get secret|grep admin-token|cut -d " " -f1`|grep "token:"|tr -s " "|cut -d " " -f2
  echo "login to dashboard with the above token"
  echo https://172.17.8.101:`kubectl -n kube-system get svc kubernetes-dashboard -o=jsonpath='{.spec.ports[0].port}'`
  echo "install traefik ingress controller"
  kubectl apply -f /vagrant/addon/traefik-ingress/
  echo "install cockpit on CentOS..."
  yum -y install cockpit cockpit-kubernetes #assumes yes for the prompt
  systemctl enable --now cockpit.socket
  echo "CentOS cockpit avaialble on https://172.17.8.101:9090"
fi

SHELL
        s.args = [i]
        end
    end
  end

end