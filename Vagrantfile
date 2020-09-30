# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure(2) do |config|
  config.vm.box = "geerlingguy/centos8"

  config.vm.provider "virtualbox" do |v|
    v.name = "ansible-tower"
    v.memory = 6144
    v.cpus = 4
  end

  config.vm.hostname = "ansible-tower"
  config.vm.network :public_network, :mac => "0800274b0e90"

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "playbooks/main.yml"
    ansible.become = true
  end

end