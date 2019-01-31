
$script = <<-SCRIPT

# Global variables
K8S_RELEASE=1.12.4
DOCKER_RELEASE=18.06.0.ce-3

# Update machine
yum update -y

# Install Docker
yum install -y yum-utils device-mapper-persistent-data lvm2
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
yum install -y docker-ce-${DOCKER_RELEASE}.el7.x86_64
systemctl enable docker.service
systemctl start docker

# Install K8s - Master node
cat <<EOF | tee -a /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
sed -i '/el7-/s/$/\$basearch/' /etc/yum.repos.d/kubernetes.repo
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
yum install -y kubelet-${K8S_RELEASE} kubectl-${K8S_RELEASE} kubeadm-${K8S_RELEASE} --disableexcludes=kubernetes
sed -i '0,/ExecStart=/s//Environment="KUBELET_EXTRA_ARGS=--cgroup-driver=cgroupfs"\n&/' /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
systemctl enable kubelet.service
cat <<EOF | tee -a /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
cat <<RCLOCAL | tee -a /etc/rc.d/rc.local
swapoff -a
sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
RCLOCAL
chmod ug+x /etc/rc.d/rc.local
mkdir -p /etc/cni/net.d
mkdir -p /opt/cni/bin
kubeadm init --pod-network-cidr=192.168.0.0/16 --kubernetes-version=${K8S_RELEASE}

# Setup admin k8s credentials for the vagrant user
sudo --user=vagrant mkdir -p /home/vagrant/.kube
cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

# Deploy pods network add-on
kubectl --kubeconfig /etc/kubernetes/admin.conf apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
kubectl --kubeconfig /etc/kubernetes/admin.conf apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

# Make K8s master node schedulable
kubectl --kubeconfig /etc/kubernetes/admin.conf taint nodes --all node-role.kubernetes.io/master-

# Install local docker registry
mkdir -p /var/docker/registry/data
sudo docker run -d -p 5000:5000 --name registry -v /var/docker/registry/data -e "REGISTRY_STORAGE_DELETE_ENABLED=true" --restart always registry:2
cat <<DOCKERREG | tee -a /etc/sysconfig/docker
INSECURE_REGISTRY='--insecure-registry localhost:5000'
DOCKERREG

# Initialise working folders
sudo --user=vagrant mkdir -p /home/vagrant/app/docker
sudo --user=vagrant mkdir -p /home/vagrant/app/k8s

# Install Git
yum install -y git

SCRIPT

required_plugins = %w( vagrant-vbguest vagrant-proxyconf )
required_plugins.each do |plugin|
  system "vagrant plugin install #{plugin}" unless Vagrant.has_plugin? plugin
end

Vagrant.configure("2") do |config|
  config.vm.box = "centos/7"
  config.vm.define "devbox", primary: true do |web|
    config.vm.hostname = "devbox"
    config.vm.provider "virtualbox" do |vb|
      vb.cpus = "2"
      vb.memory = "3000"
    end
	#if Vagrant.has_plugin?("vagrant-proxyconf")
      #config.proxy.http     = "http://<mon_proxy>:3128/"
      #config.proxy.https    = "http://<mon_proxy>:3128/"
      #config.proxy.no_proxy = "localhost,127.0.0.1,devbox,10.0.2.15,.local"
    #end
    config.vm.synced_folder "./", "/vagrant", type: "rsync", disabled: false
    config.vm.network "forwarded_port", guest: 80, host: 80
    config.vm.network "forwarded_port", guest: 443, host: 443
	config.vm.network "forwarded_port", guest: 5000, host: 5000
	config.vm.network "forwarded_port", guest: 8080, host: 8080
	config.vm.network "forwarded_port", guest: 8200, host: 8200
	config.vm.network "forwarded_port", guest: 8500, host: 8500
  end
  config.vm.provision "shell", inline: $script
end