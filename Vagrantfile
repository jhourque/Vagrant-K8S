# -*- mode: ruby -*-
# vi: set ft=ruby :

# This script to install Kubernetes will get executed after we have provisioned the box 
$setregistryproxy = <<-REGISTRY
# Add environment vars pointing Docker to use the proxy
mkdir -p /etc/systemd/system/docker.service.d
cat << EOD > /etc/systemd/system/docker.service.d/http-proxy.conf
[Service]
Environment="HTTP_PROXY=http://192.168.40.50:3128/"
Environment="HTTPS_PROXY=http://192.168.40.50:3128/"
EOD

# Get the CA certificate from the proxy and make it a trusted root.
curl http://192.168.40.50:3128/ca.crt > /usr/share/ca-certificates/docker_registry_proxy.crt
echo "docker_registry_proxy.crt" >> /etc/ca-certificates.conf
update-ca-certificates --fresh

# Reload systemd
systemctl daemon-reload

# Restart dockerd
systemctl restart docker.service
REGISTRY

Vagrant.configure("2") do |config|
  os = "bento/ubuntu-16.04"
  net_ip = "192.168.40"
  master_ip = "#{net_ip}.10"
  master_name = "master"

  config.vm.define :master, primary: true do |master_config|
    master_config.vm.provider "virtualbox" do |vb|
        vb.memory = "1024"
        vb.cpus = 1
        vb.name = "#{master_name}"
    end

    master_config.vm.box = "#{os}"
    master_config.vm.host_name = 'master'
    master_config.vm.network "private_network", ip: "#{net_ip}.10"

    master_config.vm.provision "docker"
    master_config.vm.provision "shell", inline: $setregistryproxy

    master_config.vm.provision "shell", inline: <<-SHELL
      cat /vagrant/public_key >> .ssh/authorized_keys

      sudo tee -a /etc/hosts << 'EOF'
192.168.40.10 master
192.168.40.11 node1
192.168.40.12 node2 
EOF
      
      sudo apt-get update
      sudo apt-get install -y apt-transport-https
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" |sudo tee /etc/apt/sources.list.d/kubernetes.list
      sudo apt-get update
      sudo apt-get install -y kubelet kubeadm kubectl
      sudo swapoff -a
      sudo sed -i '/ swap / s/^/#/' /etc/fstab

      echo init k8s config with flannel CNI
      sudo sed -i 's/KUBELET_EXTRA_ARGS=/KUBELET_EXTRA_ARGS=--node-ip=#{master_ip}/' /etc/default/kubelet
      sudo kubeadm init --apiserver-cert-extra-sans=#{master_ip} --apiserver-advertise-address=#{master_ip} --pod-network-cidr=10.244.0.0/16 --service-cidr 10.96.0.0/12 --node-name #{master_name} |tee /vagrant/kubeadm_init.log

      echo "init CNI flannel"
      wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
      echo patch kube-flannel.yml for eth1 support
      sed -i 's/        - --kube-subnet-mgr/        - --kube-subnet-mgr\\n        - -iface=eth1/' kube-flannel.yml
      sudo kubectl --kubeconfig /etc/kubernetes/admin.conf  apply -f kube-flannel.yml

      echo set kubectl admin config to vagrant account
      mkdir -p ~vagrant/.kube
      sudo cp -i /etc/kubernetes/admin.conf ~vagrant/.kube/config
      sudo chown vagrant:vagrant ~vagrant/.kube/config

    SHELL
  end

  [
    ["node1",    "#{net_ip}.11",    "1024",    os ],
    ["node2",    "#{net_ip}.12",    "1024",    os ],
  ].each do |vmname,ip,mem,os|
    config.vm.define "#{vmname}" do |node_config|
      node_config.vm.provider "virtualbox" do |vb|
          vb.memory = "#{mem}"
          vb.cpus = 1
          vb.name = "#{vmname}"
      end
      node_config.vm.box = "#{os}"
      node_config.vm.hostname = "#{vmname}"
      node_config.vm.network "private_network", ip: "#{ip}"

      node_config.vm.provision "docker"
      node_config.vm.provision "shell", inline: $setregistryproxy

      node_config.vm.provision "shell", inline: <<-SHELL
        cat /vagrant/public_key >> .ssh/authorized_keys

        sudo tee -a /etc/hosts << 'EOF'
192.168.40.10 master
192.168.40.11 node1
192.168.40.12 node2 
EOF
      
        sudo apt-get update
        sudo apt-get install -y apt-transport-https
        curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
        sudo echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" |sudo tee /etc/apt/sources.list.d/kubernetes.list
        sudo apt-get update
        sudo apt-get install -y kubelet kubeadm 
        sudo swapoff -a
        sudo sed -i '/ swap / s/^/#/' /etc/fstab

        echo join kubernetes cluster
        sudo sed -i 's/KUBELET_EXTRA_ARGS=/KUBELET_EXTRA_ARGS=--node-ip=#{ip}/' /etc/default/kubelet
        join=$(grep "kubeadm join" /vagrant/kubeadm_init.log)
        sudo $join
      SHELL
    end
  end
end
