# -*- mode: ruby -*-
# vi: set ft=ruby :

# See README.md for details

VAGRANTFILE_API_VERSION = "2"

#vagrant plugin install vagrant-reload
#
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  config.vm.box = "centos/7"
  config.vm.define "ns1" do |agent1|
  config.vbguest.auto_update = true 
    agent1.vm.hostname = "ns1.bitfocus.local"
    agent1.vm.network "private_network", ip: "172.31.1.20"
    agent1.vm.provision "shell", path: "strap_ns1"  
    agent1.vm.provision :reload
  end


  config.vm.box = "centos/7"
  config.vm.define "ns2" do |agent1|
  config.vbguest.auto_update = true 
    agent1.vm.hostname = "ns2.bitfocus.local"
    agent1.vm.network "private_network", ip: "172.31.1.21"
    agent1.vm.provision "shell", path: "strap_ns2"  
  end


  config.vm.box = "centos/7"
  config.vm.define "client" do |agent1|
  config.vbguest.auto_update = true 
    agent1.vm.hostname = "client.bitfocus.local"
    agent1.vm.network "private_network", ip: "172.31.1.22"
    agent1.vm.provision "shell", path: "strap_client"  
  end
end
