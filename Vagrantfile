# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'

base_config_yaml = <<-END
---
network_type: "private_network"
ip_base: "192.168.68."
ip_start: 200
num_endpoints: 2
domain_base: "local"
tower_subdomain: "tower"
endpoint_subdomain_prefix: "node"
tower_memory: 6144
tower_cpus: 4
endpoint_memory: 1024
endpoint_cpus: 1
vm_group: "AnsibleTowerDemo"
END

_config = YAML.load(base_config_yaml)
begin
  _config.merge!(YAML.load_file(File.join(File.dirname(__FILE__), "config_local.yml")))
rescue Errno::ENOENT # No config_local.yaml found -- that's OK; just use the defaults.
end

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
grep '#{_config["tower_subdomain"]}\\* #{_config["endpoint_subdomain_prefix"]}\\*' ~/.ssh/config &>/dev/null || {
  cat >> ~/.ssh/config <<'  EOF'
  Host #{_config["tower_subdomain"]}* #{_config["endpoint_subdomain_prefix"]}*
    StrictHostKeyChecking no
    UserKnownHostsFile=/dev/null
  EOF
  chmod 0600 ~/.ssh/config
}
#populate /etc/hosts
for x in {0..#{_config["num_endpoints"]}}; do
  grep "#{_config["ip_base"]}$((#{_config["ip_start"]} + x))" /etc/hosts &>/dev/null || {
    if [ $x -eq 0 ]; then machine="#{_config["tower_subdomain"]}"; else machine="#{_config["endpoint_subdomain_prefix"]}$x"; fi
    echo "#{_config["ip_base"]}$((#{_config["ip_start"]} + x)) $machine $machine.#{_config["domain_base"]}" | sudo tee -a /etc/hosts &>/dev/null
  }
done
SCRIPT
## end global script

## inventory script - handles creating ansible inventory file on host
# Due to a quirk with Vagrant, triggers run as local inline need to be formatted as 
# a command and not a full blown shell script.  In order to conform, the shell script
# has been modified to actually be a single line call to `bash -c` with the body of
# the script between single quotes and every line ended with "; \".
$inventory = <<SCRIPT
bash -c '\
echo "[control]" > share/inventory; \
echo "#{_config["tower_subdomain"]}.#{_config["domain_base"]} ansible_host=#{_config["ip_base"]}#{_config["ip_start"]}" >> share/inventory; \
echo "" >> share/inventory; \
if [ #{_config["num_endpoints"]} -gt 0 ]; then \
  echo "[nodes]" >> share/inventory; \
  for x in {1..#{_config["num_endpoints"]}}; do \
    echo "#{_config["endpoint_subdomain_prefix"]}$x.#{_config["domain_base"]} ansible_host=#{_config["ip_base"]}$((#{_config["ip_start"]} + x))" >> share/inventory; \
  done; \
fi; \
echo "" >> share/inventory; \
echo "[all:vars]" >> share/inventory; \
echo "ansible_user=vagrant" >> share/inventory \
'
SCRIPT
## end inventory script

## message script - handles displaying a list of guests on host
# Due to a quirk with Vagrant, triggers run as local inline need to be formatted as 
# a command and not a full blown shell script.  In order to conform, the shell script
# has been modified to actually be a single line call to `bash -c` with the body of
# the script between single quotes and every line ended with "; \".
$message = <<SCRIPT
bash -c '\
echo "**************************************************"; \
echo "The following systems were provisioned/started:"; \
echo "#{_config["tower_subdomain"]}.#{_config["domain_base"]} : #{_config["ip_base"]}#{_config["ip_start"]}"; \
if [ #{_config["num_endpoints"]} -gt 0 ]; then \
  for x in {1..#{_config["num_endpoints"]}}; do \
    echo "#{_config["endpoint_subdomain_prefix"]}$x.#{_config["domain_base"]} : #{_config["ip_base"]}$((#{_config["ip_start"]} + x))"; \
  done; \
fi; \
echo "**************************************************"; \
'
SCRIPT
## end message script

Vagrant.configure(2) do |config|
  # loop: 0=tower node; 1..=endpoint nodes
  (0.._config["num_endpoints"]).each do |i|
    machine_name = i==0?_config["tower_subdomain"]:"#{_config["endpoint_subdomain_prefix"]}#{i}"
    hostname = "#{machine_name}.#{_config["domain_base"]}"
    ip = "#{_config["ip_base"]}#{_config["ip_start"]+i}"
    memory = i==0?_config["tower_memory"]:_config["endpoint_memory"]
    cpus = i==0?_config["tower_cpus"]:_config["endpoint_cpus"]
    primary = i==0

    config.vm.define machine_name, primary: primary do |subconfig|
      subconfig.vm.box = "damienlive/rhelbox"
      subconfig.vm.box_version = "~> 8.3.1616209182"
      subconfig.vm.provider "virtualbox" do |v|
        v.name = machine_name
        v.memory = memory
        v.cpus = cpus
        v.customize ["modifyvm", :id, "--groups", "/#{_config["vm_group"]}"]
      end
      subconfig.vm.hostname = hostname
      subconfig.vm.network "#{_config["network_type"]}", ip: ip, hostname: true
      subconfig.vm.synced_folder "share", "/vagrant"
      subconfig.vm.provision "shell", privileged: false, inline: $global
      # on the tower box, run the tower install playbook
      if i==0
        subconfig.vm.provision "ansible" do |ansible|
          ansible.playbook = "playbooks/tower.yml"
          ansible.become = true
        end
      else
        subconfig.vm.provision "ansible" do |ansible|
          ansible.playbook = "playbooks/node.yml"
          ansible.become = true
        end
      end
    end
  end

  # This trigger generates the files local to the host machine that can be used to access
  # the guests.  In order for it to run, Vagrant must be at or above v.2.2.4 and the 
  # `vagrant up` command must be run with the env var `VAGRANT_EXPERIMENTAL=typed_triggers`
  if Vagrant.version?(">= 2.2.4")
    config.trigger.after :up, type: :command do |trigger|
      trigger.name = "Generate environment configs"
      trigger.run = {inline: $inventory}
    end
  end

  # This trigger outputs a list of the guests what were created along with their assigned
  # IP addresses.  In order for it to run, Vagrant must be at or above v.2.2.4 and the 
  # `vagrant up` command must be run with the env var `VAGRANT_EXPERIMENTAL=typed_triggers`
  if Vagrant.version?(">= 2.2.4")
    config.trigger.after :up, type: :command do |trigger|
      trigger.name = "Generate environment configs"
      trigger.run = {inline: $message}
    end
  end

  # This trigger is only necessary if using vagrant-registration plugin:
  #     https://github.com/projectatomic/adb-vagrant-registration
  # Registration plugin is globally configured and runs before vm's hostname is set.
  # This trigger runs before each machine starts and sets the registration name
  # to whatever sits in the vm hostname config.
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

