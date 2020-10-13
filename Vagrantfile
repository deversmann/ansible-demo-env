# -*- mode: ruby -*-
# vi: set ft=ruby :

##### BEGIN USER MODIFIABLE PARAMETERS #####
network_type = "private_network"  # either "private_network" (host-only) or "public_network" (bridged)
ip_base = "192.168.68."           # if type is "public_network", be sure to use existing network
ip_start = 200                    # if type is "public_network", be sure ips are in proper range (not dhcp range)
num_endpoints = 2                 # number of bexes in addition to the tower box
domain_base = "local"             # domain names will be subdomain.domain_base where subdomain is from next 2
tower_subdomain = "tower"
endpoint_subdomain_prefix = "node"
tower_memory = 6144               # tower is a glutton and seems to function best with at least 6GB
tower_cpus = 4                    # tower is a glutton and seems to function best with at least 4 cpus
endpoint_memory = 1024
endpoint_cpus = 1
vm_group = "AnsibleTowerDemo"     # puts all vms in a single group in virtualbox for ease of management
##### END USER MODIFIABLE PARAMETERS #####

## global script - handles setting up ssh and hosts on all boxes
# based on https://gist.github.com/kikitux/86a0bd7b78dca9b05600264d7543c40d
$global = <<SCRIPT
#check for private key for vm-vm comm
[ -f /vagrant/id_rsa ] || {
  ssh-keygen -t rsa -f /vagrant/id_rsa -q -N ''
}
#deploy key
[ -f /home/vagrant/.ssh/id_rsa ] || {
  cp /vagrant/id_rsa /home/vagrant/.ssh/id_rsa
  chmod 0600 /home/vagrant/.ssh/id_rsa
}
#allow ssh passwordless
grep 'vagrant@' ~/.ssh/authorized_keys &>/dev/null || {
  cat /vagrant/id_rsa.pub >> ~/.ssh/authorized_keys
  chmod 0600 ~/.ssh/authorized_keys
}
#exclude nodes from host checking
grep '#{tower_subdomain}\\* #{endpoint_subdomain_prefix}\\*' ~/.ssh/config &>/dev/null || {
  cat >> ~/.ssh/config <<'  EOF'
  Host #{tower_subdomain}* #{endpoint_subdomain_prefix}*
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
  EOF
  chmod 0600 ~/.ssh/config
}
#populate /etc/hosts
for x in {0..#{num_endpoints}}; do
  grep "#{ip_base}$((#{ip_start} + x))" /etc/hosts &>/dev/null || {
    if [ $x -eq 0 ]; then machine="#{tower_subdomain}"; else machine="#{endpoint_subdomain_prefix}$x"; fi
    echo "#{ip_base}$((#{ip_start} + x)) $machine $machine.#{domain_base}" | sudo tee -a /etc/hosts &>/dev/null
  }
done
#end script
SCRIPT
## end global script


Vagrant.configure(2) do |config|
  # loop: 0=tower node; 1..=endpoint nodes
  (0..num_endpoints).each do |i|
    machine_name = i==0?tower_subdomain:"#{endpoint_subdomain_prefix}#{i}"
    hostname = "#{machine_name}.#{domain_base}"
    ip = "#{ip_base}#{ip_start+i}"
    memory = i==0?tower_memory:endpoint_memory
    cpus = i==0?tower_cpus:endpoint_cpus
    primary = i==0

    config.vm.define machine_name, primary: primary do |subconfig|
      subconfig.vm.box = "generic/rhel8"
      subconfig.vm.provider "virtualbox" do |v|
        v.name = machine_name
        v.memory = memory
        v.cpus = cpus
        v.customize ["modifyvm", :id, "--groups", "/#{vm_group}"]
      end
      subconfig.vm.hostname = hostname
      subconfig.vm.network "#{network_type}", ip: ip, hostname: true
      subconfig.vm.synced_folder ".", "/vagrant"
      subconfig.vm.provision "shell", privileged: false, inline: $global
      # on the tower box, run the tower install playbook
      if i==0
        subconfig.vm.provision "ansible" do |ansible|
          ansible.playbook = "playbooks/main.yml"
          ansible.become = true
        end
      else
        # endpoint playbook execution here
      end
    end
  end

  # trigger is only necessary if using vagrant-registration plugin:
  #     https://github.com/projectatomic/adb-vagrant-registration
  # registration plugin is globally configured and runs before vm's hostname is set
  # this trigger runs before each machine starts and sets the registration name
  # to whatever sits in the vm hostname config
  config.trigger.before [:provision, :reload, :resume, :up] do |trigger|
    trigger.name = "Registration Name Override"
    if Vagrant.has_plugin?('vagrant-registration')
      trigger.ruby do |env,machine|
        machine.ui.detail("Setting \"#{machine.config.vm.hostname}\" into config.registration.name")
        machine.config.registration.name = "#{machine.config.vm.hostname}"
      end
    else
      trigger.info = "Skipping: vagrant-registration plugin not found"
    end
  end
end
