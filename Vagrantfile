MASTER_IP = "10.0.0.10"
WORKER_IP = "10.0.0.101"

MIN_SERVICE_NODEPORT   = 30000
MIN_SERVICE_GUEST_PORT = 3000

H1 = <<-SHELL
  h1(){
    H1="\x1B[93;1m==[ "
    H0=" ]==\x1B[0m"
    echo -e $H1$1$H0"\n"
  }
SHELL

NODEIP = <<-SHELL
  node_ip() {
    [[ -z $1 ]] && return
    IP=$1
    echo "KUBELET_EXTRA_ARGS=--node-ip='${IP}'" >/etc/default/kubelet
    systemctl daemon-reload
    systemctl restart kubelet
  }
SHELL

PROVISION_KUBERNETES = <<-SHELL
  #{H1}
  h1 "Upgrading system and installing dependencies"
  apt-get update && apt-get install -y apt-transport-https curl
  echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >/etc/apt/sources.list.d/kubernetes.list
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  apt-get update && apt-get upgrade -y

  h1 "Installing Docker, kubeadm, kubelet, kubectl and Kubernetes CNI"
  apt-get update
  apt-get install -y \
    docker.io \
    kubeadm=1.11.1-00 \
    kubelet=1.11.1-00 \
    kubectl=1.11.1-00 \
    kubernetes-cni=0.6.0-00
  #   kubelet=1.12.1-02 \
  #   kubeadm=1.12.1-02 \
  #   kubectl=1.12.1-02 \
  #   kubernetes-cni=0.6.0-00
  apt-mark hold docker.io kubelet kubeadm kubectl kubernetes-cni
  swapoff -a
  systemctl enable docker.service
SHELL

PROVISION_KUBECONFIG = <<-SHELL
  h1 "Copying kubeconfig for user 'vagrant'"
  mkdir -p /home/vagrant/.kube
  cp /home/vagrant/shared_folder/config /home/vagrant/.kube/
  chown vagrant:vagrant -R /home/vagrant/.kube
SHELL

PROVISION_MASTER = <<-SHELL
  #{H1}
  h1 "Installing Kubernetes by kubeadmin"
  kubeadm init \
    --pod-network-cidr=192.168.0.0/16 \
    --apiserver-advertise-address=#{MASTER_IP} \
    --apiserver-cert-extra-sans=localhost

  export KUBECONFIG=/etc/kubernetes/admin.conf
  h1 "Installing Calico and Canal"
  # If RBAC is enabled:
  kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/canal/rbac.yaml
  
  # Installing Calico for policy and networking:
  # Calico v3.2: https://docs.projectcalico.org/v3.2/getting-started/kubernetes/installation/hosted/calico.yaml
  # Calico v3.1: https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/calico.yaml
  # Calico v2.6: https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml
  # kubectl apply -f https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml

  # Installing Calico for policy and flannel (Canal) for networking:
  kubectl apply -f https://docs.projectcalico.org/v3.1/getting-started/kubernetes/installation/hosted/canal/canal.yaml

  # Only in non-production environments:
  kubectl taint nodes --all node-role.kubernetes.io/master-

  h1 "Sharing kubeconfig for workers and guest"
  cp /etc/kubernetes/admin.conf /home/vagrant/shared_folder/config
  sed 's/#{MASTER_IP}/localhost/' /etc/kubernetes/admin.conf > /home/vagrant/shared_folder/remote_config

  h1 "Sharing kubeadm token"
  token=$(kubeadm token list | tail -2 | head -1 | cut -f1 -d' ')
  token_ca_cert=$(openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //')
  echo $token > /home/vagrant/shared_folder/token
  echo $token_ca_cert > /home/vagrant/shared_folder/token_ca_cert.hash

  #{PROVISION_KUBECONFIG}
SHELL

PROVISION_WORKER = <<-SHELL
  #{H1}
  token=$(cat /home/vagrant/shared_folder/token)
  token_ca_cert=$(cat /home/vagrant/shared_folder/token_ca_cert.hash)

  h1 "Adding Worker to Master running on https://#{MASTER_IP}:6443"
  kubeadm join --token $token --discovery-token-ca-cert-hash sha256:$token_ca_cert #{MASTER_IP}:6443

  #{PROVISION_KUBECONFIG}
SHELL

env_ports=ENV["PORTS"]
env_ports="" if env_ports == nil

ports = Hash.new
env_ports.split(':').each do |pair| 
  p=pair.split('>')

  if p.length == 1
    guest = p[0]
    host = (MIN_SERVICE_GUEST_PORT + guest.to_i - MIN_SERVICE_NODEPORT).to_s
  else
    host = p[0]
    guest = p[1]
  end

  ports[host] = guest
end

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"

  config.vm.provision "shell", inline: PROVISION_KUBERNETES
  config.vm.synced_folder "./shared_folder", "/home/vagrant/shared_folder"

  config.vm.provider "virtualbox" do |vb|
    vb.memory = "2048"
    vb.cpus = "2"
    vb.linked_clone = true
    vb.gui = false
  end

  config.vm.define "master" do |m|
    m.vm.provider "virtualbox" do |vb|
      name = "kubernetes-master"
    end
    m.vm.hostname = "master"
    m.vm.network "private_network", 
      ip: MASTER_IP, 
      netmask: "255.255.255.0",
      auto_config: true,
      virtualbox__intnet: "kubenet",
      libvirt__network_name: "kubernetes",
      libvirt__netmask: "255.255.255.0"
    m.vm.network "forwarded_port", guest: 6443, host: 6443
    m.vm.provision "shell", inline: "#{NODEIP} node_ip #{MASTER_IP}"
    m.vm.provision "shell", inline: PROVISION_MASTER
  end

  config.vm.define "worker" do |w|
    w.vm.provider "virtualbox" do |vb|
      name = "kubernetes-worker"
    end
    w.vm.hostname = "worker"
    w.vm.network "private_network", 
      ip: WORKER_IP,
      netmask: "255.255.255.0",
      auto_config: true,
      virtualbox__intnet: "kubenet",
      libvirt__forward_mode: "none",
      libvirt__network_name: "kubernetes",
      libvirt__netmask: "255.255.255.0"
    ports.each { |host, guest|
      w.vm.network :forwarded_port, host: host, guest: guest
    }
    w.vm.provision "shell", inline: "#{NODEIP} node_ip #{WORKER_IP}"
    w.vm.provision "shell", inline: PROVISION_WORKER
  end
end
