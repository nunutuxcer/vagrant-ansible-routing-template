Purpose
=======

Spin up vagrant multi node routing environment and manage all nodes using Ansible. Upon spinning up vagrant will provision each node using Ansible to bootstrap the nodes. During the bootstrap each node will have a respective host_vars configuration file which will be updated with their eth1 address and ssh key file location. This allows you to run Ansible plays from within your HostOS or within any of your vagrant nodes.

Requirements
============

The following packages must be installed on your Host you intend on running all of this from. If Ansible is not available for your OS (Windows) You can modify the following lines in the Vagrantfile.

From:
````
#  config.vm.provision :shell, path: "bootstrap.sh"
#      node.vm.provision :shell, path: "bootstrap_ansible.sh"

config.vm.provision :ansible do |ansible|
  ansible.playbook = "bootstrap.yml"
end
````
To:
````
  config.vm.provision :shell, path: "bootstrap.sh"
      node.vm.provision :shell, path: "bootstrap_ansible.sh"

#config.vm.provision :ansible do |ansible|
#  ansible.playbook = "bootstrap.yml"
#end
````

Ansible (http://www.ansible.com/home)

VirtualBox (https://www.virtualbox.org/)

Vagrant (https://www.vagrantup.com/)



Variable Definitions
====================
````
nodes.yml
````
Define the nodes to spin up
````
---
- name: r1
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip_1: 192.168.250.101  #HostOnly interface
  priv_ips:  #Internal only interfaces
    - ip: 192.168.12.11
      desc: 01-to-02
    - ip: 192.168.14.11
      desc: 01-to-04
    - ip: 192.168.31.11
      desc: 03-to-01
    - ip: 1.1.1.10
      desc: 'Network to Advertise'
- name: r2
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip_1: 192.168.250.102  #HostOnly interface
  priv_ips:  #Internal only interfaces
    - ip: 192.168.23.12
      desc: 02-to-03
    - ip: 192.168.12.12
      desc: 01-to-02
    - ip: 2.2.2.10
      desc: 'Network to Advertise'
- name: r3
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip_1: 192.168.250.103  #HostOnly interface
  priv_ips:  #Internal only interfaces
    - ip: 192.168.31.13
      desc: 03-to-01
    - ip: 192.168.23.13
      desc: 02-to-03
    - ip: 3.3.3.10
      desc: 'Network to Advertise'
- name: r4
  box: ubuntu/trusty64
  mem: 512
  cpus: 1
  priv_ip_1: 192.168.250.104  #HostOnly interface
  priv_ips:  #Internal only interfaces
    - ip: 192.168.14.14
      desc: 01-to-04
    - ip: 192.168.31.14
      desc: 03-to-01
    - ip: 192.168.41.14
      desc: utopia
    - ip: 4.4.4.10
      desc: 'Network to Advertise'
````

Provisions nodes by bootstrapping using Ansible
````
bootstrap.yml
````
Bootstrap Playbook
````
---
- hosts: all
  remote_user: vagrant
  sudo: yes
  vars:
    - galaxy_roles:
      - mrlesmithjr.bootstrap
      - mrlesmithjr.base
      - mrlesmithjr.quagga
    - install_galaxy_roles: true
    - ssh_key_path: '.vagrant/machines/{{ inventory_hostname }}/virtualbox/private_key'
    - update_host_vars: true
  roles:
  tasks:
    - name: updating apt cache
      apt: update_cache=yes cache_valid_time=3600
      when: ansible_os_family == "Debian"

    - name: installing ansible pre-reqs
      apt: name={{ item }} state=present
      with_items:
        - python-pip
        - python-dev
      when: ansible_os_family == "Debian"

    - name: adding ansible ppa
      apt_repository: repo='ppa:ansible/ansible'
      when: ansible_os_family == "Debian"

    - name: installing ansible
      apt: name=ansible state=latest
      when: ansible_os_family == "Debian"

    - name: installing ansible-galaxy roles
      shell: ansible-galaxy install {{ item }} --force
      with_items: galaxy_roles
      when: install_galaxy_roles is defined and install_galaxy_roles

    - name: ensuring host file exists in host_vars
      stat: path=./host_vars/{{ inventory_hostname }}
      delegate_to: localhost
      register: host_var
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: creating missing host_vars
      file: path=./host_vars/{{ inventory_hostname }} state=touch
      delegate_to: localhost
      sudo: false
      when: not host_var.stat.exists

    - name: updating ansible_ssh_port
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="^ansible_ssh_port{{ ':' }}" line="ansible_ssh_port{{ ':' }} 22"
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: updating ansible_ssh_host
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="^ansible_ssh_host{{ ':' }}" line="ansible_ssh_host{{ ':' }} {{ ansible_eth1.ipv4.address }}"
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: updating ansible_ssh_key
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="^ansible_ssh_private_key_file{{ ':' }}" line="ansible_ssh_private_key_file{{ ':' }} {{ ssh_key_path }}"
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars

    - name: ensuring host_vars is yaml formatted
      lineinfile: dest=./host_vars/{{ inventory_hostname }} regexp="---" line="---" insertbefore=BOF
      delegate_to: localhost
      sudo: false
      when: update_host_vars is defined and update_host_vars
````
````
group_vars/quagga-routers
````
Quagga Routers group configurations in default state
````
---
config_quagga: true
quagga_bgp_router_configs:
  - name: r1
    local_as: 123
    router_id: 1.1.1.1
    neighbors:
      - neighbor: 192.168.12.12
        remote_as: 123
      - neighbor: 192.168.31.13
        remote_as: 123
      - neighbor: 192.168.14.14
        remote_as: 141
  - name: r2
    local_as: 123
    router_id: 2.2.2.2
    neighbors:
      - neighbor: 192.168.12.11
        remote_as: 123
      - neighbor: 192.168.23.13
        remote_as: 123
  - name: r3
    local_as: 123
    router_id: 3.3.3.3
    neighbors:
      - neighbor: 192.168.23.12
        remote_as: 123
      - neighbor: 192.168.31.11
        remote_as: 123
  - name: r4
    local_as: 141
    router_id: 4.4.4.4
    neighbors:
      - neighbor: 192.168.14.11
        remote_as: 123
quagga_enable_bgpd: false
quagga_enable_ospfd: false
quagga_enable_password: quagga
quagga_ospf_routerid: '{{ ansible_eth1.ipv4.address }}'
quagga_password: quagga
````
````
playbook.yml
````
Provisioning playbook
````
---
- hosts: quagga-routers
  remote_user: vagrant
  sudo: true
  vars:
  roles:
    - mrlesmithjr.quagga
  tasks:
    - name: installing packages
      apt: name={{ item }} state=present
      with_items:
        - traceroute
````


Usage
=====

````
git clone https://github.com/mrlesmithjr/vagrant-ansible-routing-template.git
cd vagrant-ansible-routing-template
````
Update/modify nodes.yml to reflect your desired nodes to spin up and also update/modify group_vars/quagga-routers to reflect changes.

Spin up your environment
````
vagrant up
````

To run ansible from within Vagrant nodes (Ex. site.yml)
````
vagrant ssh r1 # or r2,r3, r4; all work
cd /vagrant
````
modify nodes.yml to reflect your desired routers and configurations and then run the playbook.yml playbook.
````
ansible-playbook -i hosts playbook.yml
````

To install ansible-galaxy roles within your HostOS
````
ansible-galaxy install mrlesmithjr.bootstrap
ansible-galaxy install mrlesmithjr.base
````

If you configure this environment and set quagga_enable_ospfd: true and then run the playbook.yml your ip route should look something similar to below. (This example shows a r5 which is not included in the nodes.yml)
````
vagrant@r1:/vagrant$ ip route
default via 10.0.2.2 dev eth0
1.1.1.0/24 dev eth6  proto kernel  scope link  src 1.1.1.10
2.2.2.0/24 via 192.168.250.102 dev eth1  proto zebra  metric 20
3.3.3.0/24 via 192.168.250.103 dev eth1  proto zebra  metric 20
4.4.4.0/24 via 192.168.250.104 dev eth1  proto zebra  metric 20
5.5.5.0/24 via 192.168.250.105 dev eth1  proto zebra  metric 20
10.0.2.0/24 dev eth0  proto kernel  scope link  src 10.0.2.15
192.168.12.0/24 dev eth2  proto kernel  scope link  src 192.168.12.11
192.168.14.0/24 dev eth3  proto kernel  scope link  src 192.168.14.11
192.168.15.0/24 dev eth4  proto kernel  scope link  src 192.168.15.11
192.168.23.0/24  proto zebra  metric 20
	nexthop via 192.168.250.102  dev eth1 weight 1
	nexthop via 192.168.250.103  dev eth1 weight 1
192.168.31.0/24 dev eth5  proto kernel  scope link  src 192.168.31.11
192.168.41.0/24 via 192.168.250.104 dev eth1  proto zebra  metric 20
192.168.51.0/24 via 192.168.250.105 dev eth1  proto zebra  metric 20
192.168.250.0/24 dev eth1  proto kernel  scope link  src 192.168.250.101
````
And if you were to telnet to the ospfd daemon and run sh ip ospf neighbors
````
r1# sh ip ospf neighbor

    Neighbor ID Pri State           Dead Time Address         Interface            RXmtL RqstL DBsmL
192.168.250.102   1 2-Way/DROther     36.863s 192.168.250.102 eth1:192.168.250.101     0     0     0
192.168.250.103   1 2-Way/DROther     36.997s 192.168.250.103 eth1:192.168.250.101     0     0     0
192.168.250.104   1 Full/Backup       36.865s 192.168.250.104 eth1:192.168.250.101     0     0     0
192.168.250.105   1 Full/DR           36.900s 192.168.250.105 eth1:192.168.250.101     0     0     0
````
Additional sh ip ospf commands.
````
r1# sh ip ospf border-routers
============ OSPF router routing table =============
R    192.168.250.102       [10] area: 0.0.0.51, ASBR
                           via 192.168.250.102, eth1
R    192.168.250.103       [10] area: 0.0.0.51, ASBR
                           via 192.168.250.103, eth1
R    192.168.250.104       [10] area: 0.0.0.51, ASBR
                           via 192.168.250.104, eth1
R    192.168.250.105       [10] area: 0.0.0.51, ASBR
                           via 192.168.250.105, eth1
````
````
r1# sh ip ospf database

       OSPF Router with ID (192.168.250.101)

                Router Link States (Area 0.0.0.51)

Link ID         ADV Router      Age  Seq#       CkSum  Link count
192.168.250.101 192.168.250.101  708 0x80000004 0xac6d 1
192.168.250.102 192.168.250.102  709 0x80000004 0xaa6c 1
192.168.250.103 192.168.250.103  709 0x80000004 0xa86b 1
192.168.250.104 192.168.250.104  708 0x80000007 0xa06d 1
192.168.250.105 192.168.250.105  708 0x80000006 0xa06b 1

                Net Link States (Area 0.0.0.51)

Link ID         ADV Router      Age  Seq#       CkSum
192.168.250.105 192.168.250.105  708 0x80000004 0x9e16

                AS External Link States

Link ID         ADV Router      Age  Seq#       CkSum  Route
1.1.1.0         192.168.250.101  708 0x80000003 0x3ab5 E2 1.1.1.0/24 [0x0]
2.2.2.0         192.168.250.102  709 0x80000003 0x10db E2 2.2.2.0/24 [0x0]
3.3.3.0         192.168.250.103  709 0x80000003 0xe502 E2 3.3.3.0/24 [0x0]
4.4.4.0         192.168.250.104  708 0x80000006 0xb52b E2 4.4.4.0/24 [0x0]
5.5.5.0         192.168.250.105  708 0x80000005 0x8d50 E2 5.5.5.0/24 [0x0]
10.0.2.0        192.168.250.101  708 0x80000003 0xc521 E2 10.0.2.0/24 [0x0]
10.0.2.0        192.168.250.102  709 0x80000003 0xbf26 E2 10.0.2.0/24 [0x0]
10.0.2.0        192.168.250.103  709 0x80000003 0xb92b E2 10.0.2.0/24 [0x0]
10.0.2.0        192.168.250.104  708 0x80000005 0xaf32 E2 10.0.2.0/24 [0x0]
10.0.2.0        192.168.250.105  708 0x80000005 0xa937 E2 10.0.2.0/24 [0x0]
192.168.12.0    192.168.250.101  708 0x80000003 0x2855 E2 192.168.12.0/24 [0x0]
192.168.12.0    192.168.250.102  709 0x80000003 0x225a E2 192.168.12.0/24 [0x0]
192.168.14.0    192.168.250.101  708 0x80000003 0x1269 E2 192.168.14.0/24 [0x0]
192.168.14.0    192.168.250.104  708 0x80000005 0xfb7a E2 192.168.14.0/24 [0x0]
192.168.15.0    192.168.250.101  708 0x80000004 0x0574 E2 192.168.15.0/24 [0x0]
192.168.15.0    192.168.250.105  708 0x80000005 0xea89 E2 192.168.15.0/24 [0x0]
192.168.23.0    192.168.250.102  709 0x80000003 0xa8c8 E2 192.168.23.0/24 [0x0]
192.168.23.0    192.168.250.103  709 0x80000003 0xa2cd E2 192.168.23.0/24 [0x0]
192.168.31.0    192.168.250.101  708 0x80000003 0x5614 E2 192.168.31.0/24 [0x0]
192.168.31.0    192.168.250.103  709 0x80000003 0x4a1e E2 192.168.31.0/24 [0x0]
192.168.31.0    192.168.250.104  708 0x80000005 0x4025 E2 192.168.31.0/24 [0x0]
192.168.41.0    192.168.250.104  708 0x80000005 0xd189 E2 192.168.41.0/24 [0x0]
192.168.51.0    192.168.250.105  708 0x80000005 0x5df2 E2 192.168.51.0/24 [0x0]
````
````
r1# sh ip ospf interface
eth0 is up
  ifindex 2, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  OSPF not enabled on this interface
eth1 is up
  ifindex 3, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  Internet Address 192.168.250.101/24, Broadcast 192.168.250.255, Area 0.0.0.51
  MTU mismatch detection:enabled
  Router ID 192.168.250.101, Network Type BROADCAST, Cost: 10
  Transmit Delay is 1 sec, State DROther, Priority 1
  Designated Router (ID) 192.168.250.105, Interface Address 192.168.250.105
  Backup Designated Router (ID) 192.168.250.104, Interface Address 192.168.250.104
  Multicast group memberships: OSPFAllRouters
  Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5
    Hello due in 3.119s
  Neighbor Count is 4, Adjacent neighbor count is 2
eth2 is up
  ifindex 4, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  OSPF not enabled on this interface
eth3 is up
  ifindex 5, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  OSPF not enabled on this interface
eth4 is up
  ifindex 6, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  OSPF not enabled on this interface
eth5 is up
  ifindex 7, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  OSPF not enabled on this interface
eth6 is up
  ifindex 8, MTU 1500 bytes, BW 0 Kbit <UP,BROADCAST,RUNNING,MULTICAST>
  OSPF not enabled on this interface
lo is up
  ifindex 1, MTU 65536 bytes, BW 0 Kbit <UP,LOOPBACK,RUNNING>
  OSPF not enabled on this interface
````
````
r1# sh ip ospf route
============ OSPF network routing table ============
N    192.168.250.0/24      [10] area: 0.0.0.51
                           directly attached to eth1

============ OSPF router routing table =============
R    192.168.250.102       [10] area: 0.0.0.51, ASBR
                           via 192.168.250.102, eth1
R    192.168.250.103       [10] area: 0.0.0.51, ASBR
                           via 192.168.250.103, eth1
R    192.168.250.104       [10] area: 0.0.0.51, ASBR
                           via 192.168.250.104, eth1
R    192.168.250.105       [10] area: 0.0.0.51, ASBR
                           via 192.168.250.105, eth1

============ OSPF external routing table ===========
N E2 2.2.2.0/24            [10/20] tag: 0
                           via 192.168.250.102, eth1
N E2 3.3.3.0/24            [10/20] tag: 0
                           via 192.168.250.103, eth1
N E2 4.4.4.0/24            [10/20] tag: 0
                           via 192.168.250.104, eth1
N E2 5.5.5.0/24            [10/20] tag: 0
                           via 192.168.250.105, eth1
N E2 10.0.2.0/24           [10/20] tag: 0
                           via 192.168.250.102, eth1
                           via 192.168.250.103, eth1
                           via 192.168.250.104, eth1
                           via 192.168.250.105, eth1
N E2 192.168.12.0/24       [10/20] tag: 0
                           via 192.168.250.102, eth1
N E2 192.168.14.0/24       [10/20] tag: 0
                           via 192.168.250.104, eth1
N E2 192.168.15.0/24       [10/20] tag: 0
                           via 192.168.250.105, eth1
N E2 192.168.23.0/24       [10/20] tag: 0
                           via 192.168.250.102, eth1
                           via 192.168.250.103, eth1
N E2 192.168.31.0/24       [10/20] tag: 0
                           via 192.168.250.103, eth1
                           via 192.168.250.104, eth1
N E2 192.168.41.0/24       [10/20] tag: 0
                           via 192.168.250.104, eth1
N E2 192.168.51.0/24       [10/20] tag: 0
                           via 192.168.250.105, eth1
````
License
-------

BSD

Author Information
------------------

Larry Smith Jr.
- @mrlesmithjr
- http://everythingshouldbevirtual.com
- mrlesmithjr [at] gmail.com
