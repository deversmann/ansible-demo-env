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
| Virtual Box Extension Pack (optional) | `brew cask install virtualbox-extension-pack`
| [Ansible](https://www.ansible.com) | `pip3 install ansible` |
| [vagrant-registration plugin](https://github.com/projectatomic/adb-vagrant-registration) (optional) <br /> Automatically registers RHEL boxes with  RHSM | `vagrant plugin install vagrant-registration` |

## Configuration
The top of the `Vagrantfile` includes the following user-configurable variables:
| Variable name | Acceptable values | Description |
|--|--|--|
| `network_type` | `"private_network"`, `"public_network"` | `"private_network"` means host-only, `"public_network"` means bridged |
| `ip_base` | `"xxx.yyy.zzz."` | i.e. `"192.168.68."`. If type is `"public_network"`, be sure to use existing network. |
| `ip_start` | integer < 253 | i.e. `200`. If type is `"public_network"`, be sure IPs are in proper range (not dhcp range). |
| `num_endpoints` | integer | i.e. `2`. The number of boxes in addition to the tower box. |
| `domain_base` | legal domain name string | i.e. `"local"`. Domain names will be `subdomain`.`domain_base` where `subdomain` is from the next 2 variables. |
| `tower_subdomain` | legal domain name string | i.e. `"tower"` |
| `endpoint_subdomain_prefix` | legal domain name string | i.e. `"node"` |
| `tower_memory` | integer | i.e. `6144`. Number of MB of memory to allocate to the Tower box. Tower is a glutton and seems to function best with at least 6GB. |
| `tower_cpus` | integer | i.e. `4` Number of virtual cpus to allocate to the Tower box. Tower is a glutton and seems to function best with at least 4 cpus. |
| `endpoint_memory` | integer | i.e. `1024`. Number of MB of memory to allocate to the endpoint boxes. |
| `endpoint_cpus` | integer | i.e. `1`. Number of virtual cpus to allocate to the endpoint boxes. |
| `vm_group` | string | i.e. `"AnsibleTowerDemo"`. Name of the VirtualBox group to place the VMs in.  Eases management and startup/shutdown of all machines from VirtualBox console. |

Sensible defaults are included in the file.

If using the `vagrant-registration` plugin, it is recommended to store the registration information in `~/.vagrant.d/Vagrantfile` as described in the [plugin docs](https://github.com/projectatomic/adb-vagrant-registration#credential-configuration).  The `Vagrantfile` included in this project only supplies the `config.registration.name` value if the plugin is detected.

## Running
After setting all of the variables in the `Vagrantfile`, simply type:
```shell
vagrant up
```

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

## TO-DOs
- [ ] Output a message after provisioning is complete with all the names and IPs used.
- [x] Generate a rudimentary Ansible `inventory` and `ansible.cfg` for the host to use to run ansible against all of the machines.
- [ ] Output a `hosts` file exerpt containing all of the boxes info for use on the host machine.