require 'json'

MASTER_IP = "10.0.0.100"

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

num_workers = 1
num_workers = ENV["WORKERS"].to_i if ENV["WORKERS"] != nil

def active_cluster
  machine_id = Dir["./.vagrant/machines/*/virtualbox/id"]
  return false if machine_id.length == 0

  machine_id.each do |id|
    hostname = id.split('/')[3]
    next if hostname.include? "master"
    content = File.open(id, "r"){ |file| file.read }
    return true if content.length > 1
  end
  return false
end

remote_config = Dir["./shared_folder/remote_config"]

ports = Hash.new
if ENV["PORTS"] != nil
  ENV["PORTS"].split(':').each do |pair| 
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
elsif active_cluster && remote_config.length != 0
  services = JSON.parse(`kubectl --kubeconfig ./shared_folder/remote_config get services -o json 2>/dev/null`)
  services["items"].each do |s| 
    next if s["spec"]["type"] != "NodePort"
    name = s["spec"]["selector"]["run"]
    nodePort = []
    s["spec"]["ports"].each do |p|
      nodePort << p["nodePort"]
    end
    puts "Identified Service '#{name}' of type nodePort using ports: #{nodePort.join ','}"
    nodePort.each do |guest|
      host = (MIN_SERVICE_GUEST_PORT + guest - MIN_SERVICE_NODEPORT).to_s
      ports[host] = guest
      puts "\e[1m==> #{name}:\e[22m #{host} (host) => #{guest} (guest) \e[30;1mExecute: curl http://localhost:#{host}\e[0m"
    end
  end
  puts
end

worker_base_ip = MASTER_IP.split('.')[0..2].join('.')
worker_init_octet = MASTER_IP.split('.')[3].to_i

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

  (1..num_workers).each do |i|
    worker_ip = "#{worker_base_ip}.#{worker_init_octet + i}"
    worker_name = "worker-%02d" % [i]
    config.vm.define worker_name do |w|
      w.vm.provider "virtualbox" do |vb|
        name = "kubernetes-#{worker_name}"
      end
      w.vm.hostname = worker_name
      w.vm.network "private_network", 
        ip: worker_ip,
        netmask: "255.255.255.0",
        auto_config: true,
        virtualbox__intnet: "kubenet",
        libvirt__forward_mode: "none",
        libvirt__network_name: "kubernetes",
        libvirt__netmask: "255.255.255.0"
      ports.each { |host, guest|
        w.vm.network :forwarded_port, host: host, guest: guest
      }
      w.vm.provision "shell", inline: "#{NODEIP} node_ip #{worker_ip}"
      w.vm.provision "shell", inline: PROVISION_WORKER
    end
  end
end
