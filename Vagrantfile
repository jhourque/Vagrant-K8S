# -*- mode: ruby -*-
# vi: set ft=ruby :

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
    master_config.vm.provision "shell", inline: <<-SHELL
      cat /vagrant/public_key >> .ssh/authorized_keys
      sudo apt-get update
      sudo apt-get install -y apt-transport-https
      curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
      sudo echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" |sudo tee /etc/apt/sources.list.d/kubernetes.list
      sudo apt-get update
      sudo apt-get install -y kubelet kubeadm kubectl
      sudo swapoff -a
      sudo sed -i '/ swap / s/^/#/' /etc/fstab

      echo init k8s config with flannel CNI
      sudo kubeadm init --apiserver-cert-extra-sans=#{master_ip} --apiserver-advertise-address=#{master_ip} --pod-network-cidr=10.244.0.0/16 --node-name #{master_name} |tee /vagrant/kubeadm_init.log
      sudo kubectl --kubeconfig /etc/kubernetes/admin.conf  apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

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
    config.vm.define "#{vmname}" do |minion_config|
      minion_config.vm.provider "virtualbox" do |vb|
          vb.memory = "#{mem}"
          vb.cpus = 1
          vb.name = "#{vmname}"
      end
      minion_config.vm.box = "#{os}"
      minion_config.vm.hostname = "#{vmname}"
      minion_config.vm.network "private_network", ip: "#{ip}"

      minion_config.vm.provision "docker"

      minion_config.vm.provision "shell", inline: <<-SHELL
        cat /vagrant/public_key >> .ssh/authorized_keys
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
