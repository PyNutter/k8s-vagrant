# -*- mode: ruby -*-
# vi: set ft=ruby :

master_ip = "192.168.205.10"
box = "ubuntu/eoan64"
box_version = "20200508.0.0"

servers = [
  {
    :name => "k8s-head",
    :type => "master",
    :eth1 => master_ip,
    :mem => "2048",
    :cpu => "2"
  },
  {
    :name => "k8s-node-1",
    :type => "node",
    :eth1 => "192.168.205.11",
    :mem => "2048",
    :cpu => "2"
  },
  {
    :name => "k8s-node-2",
    :type => "node",
    :eth1 => "192.168.205.12",
    :mem => "2048",
    :cpu => "2"
  }
]

# This script to install k8s using kubeadm will get executed after a box is provisioned
$configureBox = <<-SCRIPT
  # install docker version 19.03
  DOCKER_VER="19.03.8"
  # Install Docker CE
  ## Set up the repository:
  ### Install packages to allow apt to use a repository over HTTPS
  apt-get update && apt-get install -y \
    apt-transport-https ca-certificates curl software-properties-common gnupg2

  ### Add Docker’s official GPG key
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -

  ### Add Docker apt repository.
  add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

  # Install Docker CE
  apt-get update && apt-get install -y \
    containerd.io=1.2.13-1 \
    docker-ce=5:$DOCKER_VER~3-0~ubuntu-$(lsb_release -cs) \
    docker-ce-cli=5:$DOCKER_VER~3-0~ubuntu-$(lsb_release -cs)

  # Setup daemon.
  cat > /etc/docker/daemon.json <<EOF
  {
    "exec-opts": ["native.cgroupdriver=systemd"],
    "log-driver": "json-file",
    "log-opts": {
      "max-size": "100m"
    },
    "storage-driver": "overlay2"
  }
EOF

  mkdir -p /etc/systemd/system/docker.service.d

  # Restart docker.
  systemctl daemon-reload
  systemctl restart docker

  # run docker commands as vagrant user (sudo not required)
  usermod -aG docker vagrant

  # install kubeadm
  echo "--- RUNNING KUBEADM INSTALL ---"
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  cat > /etc/apt/sources.list.d/kubernetes.list <<EOF
  deb http://apt.kubernetes.io/ kubernetes-xenial main
EOF
  apt-get update
  apt-get install -y kubelet kubeadm kubectl
  apt-mark hold kubelet kubeadm kubectl

  # kubelet requires swap off
  swapoff -a

  # keep swap off after reboot
  sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

  # ip of this box
  IP_ADDR=`ip -br -4 addr show enp0s8 | grep -Po -1 "\d+\.\d+\.\d+\.\d+"`
  # set node-ip
  cat > /etc/default/kubelet <<EOF
  KUBELET_EXTRA_ARGS=--node-ip=${IP_ADDR}
EOF
    sudo systemctl restart kubelet
SCRIPT

$configureMaster = <<-SCRIPT
  echo "This is master"
  # ip of this box
  IP_ADDR=`ip -br -4 addr show enp0s8 | grep -Po -1 "\d+\.\d+\.\d+\.\d+"`

  # install k8s master
  HOST_NAME=$(hostname -s)
  kubeadm config images pull
  kubeadm init --apiserver-advertise-address=$IP_ADDR --apiserver-cert-extra-sans=$IP_ADDR  --node-name $HOST_NAME --pod-network-cidr=172.16.0.0/16

  #copying credentials to regular user - vagrant
  sudo --user=vagrant mkdir -p /home/vagrant/.kube
  cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
  chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

  # install WeaveNet  pod network addon
  export KUBECONFIG=/etc/kubernetes/admin.conf
  kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

  kubeadm token create --print-join-command >> /etc/kubeadm_join_cmd.sh
  chmod +x /etc/kubeadm_join_cmd.sh

  # required for setting up password less ssh between guest VMs
  sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
  sudo service sshd restart
SCRIPT

$configureNode = <<-SCRIPT
  echo "This is worker"
  apt-get install -y sshpass
  sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.205.10:/etc/kubeadm_join_cmd.sh .
  sh ./kubeadm_join_cmd.sh
SCRIPT

Vagrant.configure("2") do |config|
  servers.each do |opts|
    config.vm.define opts[:name] do |config|
      config.vm.box = box
      config.vm.box_version = box_version
      config.vm.hostname = opts[:name]
      config.vm.network :private_network, ip: opts[:eth1]

      config.vm.provider "virtualbox" do |v|
        v.name = opts[:name]
        v.customize ["modifyvm", :id, "--groups", "/Kubernetes Development"]
        v.customize ["modifyvm", :id, "--memory", opts[:mem]]
        v.customize ["modifyvm", :id, "--cpus", opts[:cpu]]
      end

      # we cannot use this because we can't install the docker version we want - https://github.com/hashicorp/vagrant/issues/4871
      #config.vm.provision "docker"

      config.vm.provision "shell", inline: $configureBox

      if opts[:type] == "master"
        config.vm.provision "shell", inline: $configureMaster
      else
        config.vm.provision "shell", inline: $configureNode
      end
    end
  end
end 
