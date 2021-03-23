# Tower demo/sandbox environment
This vagrantfile and associated accoutrements helps to set up a simple and straightforward 
development / demo / sandbox environment for Ansible Tower. It will provision a Tower box
and a configurable number of endpoints.  It installs Tower on the tower box and configures
ssh keys on all of them for passwordless connecting between the boxes of the `vagrant` 
user.

## Prerequisites
| Prerequisite | macOS install |
|--|--|
| [Vagrant](https://www.vagrantup.com) | `brew cask install vagrant` |
| [VirtualBox](https://www.virtualbox.org) | `brew cask install virtualbox` |
| [Ansible](https://www.ansible.com) | `pip3 install ansible` |
| [vagrant-registration plugin](https://github.com/projectatomic/adb-vagrant-registration) (optional) <br /> Automatically registers RHEL boxes with  RHSM | `vagrant plugin install vagrant-registration` |

## Configuration
The project uses the defaults shown in the table below.  If desired, any configuration option can be overridden by copying the file `config_local.yml.template` to `config_local.yml` and uncommenting and changing the desired option adhering to the *Acceptable values* in the table below. Any defaults not overridden will continue to use the defaults.
| Variable name | Acceptable values | Default | Description |
|--|--|--|--|
| `network_type` | `"private_network"`, `"public_network"` | `"private_network"` | `"private_network"` means host-only, `"public_network"` means bridged |
| `ip_base` | `"xxx.yyy.zzz."` | `"192.168.68."` | i.e. `"192.168.68."`. If type is `"public_network"`, be sure to use existing network. |
| `ip_start` | integer < 253 | `200` | i.e. `200`. If type is `"public_network"`, be sure IPs are in proper range (not dhcp range). |
| `num_endpoints` | integer | `2` | i.e. `2`. The number of boxes in addition to the tower box. |
| `domain_base` | legal domain name string | `"local"` | i.e. `"local"`. Domain names will be `subdomain`.`domain_base` where `subdomain` is from the next 2 variables. |
| `tower_subdomain` | legal domain name string | `"tower"` | i.e. `"tower"` |
| `endpoint_subdomain_prefix` | legal domain name string | `"node"` | i.e. `"node"` |
| `tower_memory` | integer | `6144` | i.e. `6144`. Number of MB of memory to allocate to the Tower box. Tower is a glutton and seems to function best with at least 6GB. |
| `tower_cpus` | integer | `4` | i.e. `4` Number of virtual cpus to allocate to the Tower box. Tower is a glutton and seems to function best with at least 4 cpus. |
| `endpoint_memory` | integer | `1024` | i.e. `1024`. Number of MB of memory to allocate to the endpoint boxes. |
| `endpoint_cpus` | integer | `1` | i.e. `1`. Number of virtual cpus to allocate to the endpoint boxes. |
| `vm_group` | string | `"AnsibleTowerDemo"` | i.e. `"AnsibleTowerDemo"`. Name of the VirtualBox group to place the VMs in.  Eases management and startup/shutdown of all machines from VirtualBox console. |

If using the `vagrant-registration` plugin, it is recommended to store the registration information in `~/.vagrant.d/Vagrantfile` as described in the [plugin docs](https://github.com/projectatomic/adb-vagrant-registration#credential-configuration).  The `Vagrantfile` included in this project only supplies the `config.registration.name` value if the plugin is detected.

## Running
After setting all of the variables in the `Vagrantfile`, simply type:
```shell
vagrant up
```

> :exclamation:
> In order to process the trigger that creates a local ansible inventory file, we are using "experimental" features of Vagrant.  To turn those on, you need to include a specific environment variable with the start command.  Instead type:
> ```shell
> VAGRANT_EXPERIMENTAL=typed_triggers vagrant up
> ```

Vagrant will run for quite some time provisioning all of the machines.  When it completes, all of your machines should be available at the IP addresses you configured. They should also all be configured for passwordless SSH between each-other with the `vagrant` user, as well as paswordless `sudo`. Tower should be running on the tower box.  You can ssh into the machines either by using the Vagrant ssh command:
```shell
vagrant ssh tower
vagrant ssh node1
```
or by ssh'ing to the IPs you configured with the username `vagrant` and password `vagrant`.

When you are done, you can shut the machines down by typing:
```shell
vagrant halt
```
Running the up command will restart them.

You can delete the VMs by typing:
```shell
vagrant destroy
```

Both halt and destroy will unregister the machines if you are using the `vagrant-registration` plugin. Restarting them will re-register them.

## Host tools
The `playbooks` directory includes a `host.yml` playbook that when run, can update the host's `/etc/hosts` file with information about the nodes.  It can also remove that information and also just display it.  When executed from the project directory, the included `ansible.cfg` file will automatically use the generated inventory and ssh key files.  The 3 different actions are tagged to aid in selectively running them.

```shell
ansible-playbook playbooks/host.yml --tags=list_hosts
ansible-playbook playbooks/host.yml --tags=add_hosts -K
ansible-playbook playbooks/host.yml --tags=remove_hosts -K
```

## TO-DOs
**moved TO-DOs to GitHub issues**
