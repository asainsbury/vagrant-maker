# vagrant-maker
Ansible playbook to create a Vagrantfile and modify your .ssh/config file and also generate your hosts inventory file, all based on a set of data in groupvars.

## Introduction
This code is a way for me to streamline the building of Vagrant hosts, on my Mac.

After lots of research (Google) I've managed to come up with a compact and bijou Vagrantfile, which reads in a templated set of data (in YAMAL obviously!) and spins up the hosts when you vagrant up.  To add to the smoothness of developing with Ansible, I also create some code in .ssh/config to alias the hostname, then generate the inventory file. Another play takes care of provisioning the host with ssh keys and adds a user.

Run the playbook `ansible-playbook vagrant-maker.yml`

# The vagrant-maker playbook 
It was surprisingly simple to generate the files, and it uses just 2 modules 
- 'blockinfile' for the ssh
- 'lineinfile' for the inventory file

Both tasks have a state of presence, so you can remove the config by simply changing the state field in the groupvars.

```
---
- hosts: localhost
  gather_facts: false

  tasks:
    - name: Print the tests 
      debug:
        var: host_list

    - name: Add hosts for vagrant into sshconfg
      blockinfile:
        path: "{{ssh_config}}"
        marker: "# {mark} ANSIBLE MANAGED BLOCK {{ item.name }}"
         
        block: | 
          Host {{item.name}} *.test.local
            hostname {{item.ip}}
            StrictHostKeyChecking no
            UserKnownHostsFile=/dev/null
            User {{ssh_user}}
            LogLevel ERROR  
        
        state: "{{item.state}}"

      with_items:
        - "{{host_list}}"

    - name: Add hosts to inventory
      lineinfile:
        path: "{{host_file}}"
        line: "{{item.name}} ansible_ssh_host={{item.ip}}"
        state: "{{item.state}}"
      with_items:
        - "{{host_list}}"
```
# Bootstraping the linux hosts
Post vagrant up, calls this playbook, which sets up the environment for simple ssh access. This have been tested on both Ubunt and Centos, as listed in the groupvars section.

```
---
- hosts: all
  gather_facts: yes
  become: true

  tasks:
    - name: Create User
      user:
        name: "{{ssh_user}}"
        password: Password@1
        shell: /bin/bash
        append: yes
        generate_ssh_key: yes
        ssh_key_bits: 1024
        ssh_key_file: .ssh/id_rsa

    - name: Set authorized key from file
      authorized_key:
        user: "{{ssh_user}}"
        state: present
        key: "{{ lookup('file', '~/.ssh/id_rsa.pub') }}"

    - name: Add user to sudoers
      lineinfile: dest=/etc/sudoers
                  regexp="{{ssh_user}}"
                  line='"{{ssh_user}}"  ALL=(ALL) NOPASSWD:ALL'
                  state=present

    - name: Disallow password authentication
      lineinfile: dest=/etc/ssh/sshd_config
                  regexp="^PasswordAuthentication"
                  line="PasswordAuthentication no"
                  state=present
      notify: restart_ssh

    - name: Disallow root SSH access
      lineinfile: dest=/etc/ssh/sshd_config
                  regexp="^PermitRootLogin"
                  line="PermitRootLogin no"
                  state=present
      notify: restart_ssh

    - name: Turn off selinux, only on Centos or RHEL
      selinux:
        state: disabled
      when: ansible_distribution == 'CentOS' or ansible_distribution == 'Red Hat Enterprise Linux'
   
  handlers:
    - name: restart_ssh
      systemd:
        name: sshd
        state: restarted
```

## The Group Vars
It all starts at the beggining, which is a group vars files with paths and all sorts of other stuff.

### Host list
This is the list we are  going to use to generate ssh config and hosts inventory, but also is used by the vagrant file to build the virtual machines.

```
host_list:
  - name: splunk1
    provider: virtualbox
    group: monitoring
    ip: 1.1.1.10
    vm_name: splunk1
    mem: 512
    cpus: 1
    box: 'centos/7'
    box_url: 'https://vagrantcloud.com/centos/boxes/7'
    bootstrap: 'bootstrap.yml'
    type: "linux"
    state: "present"

  - name: dev
    provider: virtualbox
    group: monitoring
    ip: 1.1.1.11
    vm_name: dev
    mem: 512
    cpus: 1
    box: 'ubuntu/trusty64'
    box_url: 'https://vagrantcloud.com/ubuntu/trusty64'
    bootstrap: 'bootstrap.yml'
    type: "linux"
    state: "present"
```

### SSH config
The playbook iterates over the list and generates blocks of text for the ssh config:

```
# BEGIN ANSIBLE MANAGED BLOCK splunk1
Host splunk1 *.test.local
  hostname 1.1.1.10
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
  User andy
  LogLevel ERROR  
# END ANSIBLE MANAGED BLOCK splunk1
# BEGIN ANSIBLE MANAGED BLOCK dev
Host dev *.test.local
  hostname 1.1.1.11
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
  User andy
  LogLevel ERROR  
# END ANSIBLE MANAGED BLOCK dev
```

### Hosts inventory
And lines of config for the hosts inventory file:

```
splunk1 ansible_ssh_host=1.1.1.10
dev ansible_ssh_host=1.1.1.11
```

## Vagrantfile
This is where all the fun begins.

### Importing the yaml data structure:
This is where we set the data for the vagrantfile to source from, and inside the code we have a conditional to check if the file exists or we exit:

```
# -*- mode: ruby -*-
# vi: set ft=ruby 

VAGRANTFILE_API_VERSION = '2'
# Bit of error checking around the external data source:
require 'yaml'
if File.file?('group_vars/all.yml')
  data = YAML.load_file('group_vars/all.yml')
else
  raise "Configuration file does not exist."
end

```

### Networking function:
Here I used (checkout references section) a function I found to generate the networking stuff

```
# Define a nice function to sor out all the networking
# Gets called in the main section
def network_options(host)
  options = {}

  if host.has_key?('ip')
    options[:ip] = host['ip']
    options[:netmask] = host['netmask'] ||= '255.255.255.0'
  else
    options[:type] = 'dhcp'
  end

  if host.has_key?('mac')
    options[:mac] = host['mac'].gsub(/[-:]/, '')
  end

  if host.has_key?('auto_config')
    options[:auto_config] = host['auto_config']
  end

  if host.has_key?('intnet') && host['intnet']
    options[:virtualbox__intnet] = true
  end

  options
end
```

### Vagrant host loop
At this point we can start a loop, to iterate over the data we set in the groupvars and build the box

```
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  data['host_list'].each do |host|
    
    set_group = '/' + host['group']
```

### Vagrant configure boxes
Then we start to set the parameters which get used to build the virtual machine

```
    config.vm.define host['name'] do |node|
      node.vm.box = host['box']
      node.vm.box_url = host['box_url']
      node.vm.hostname = host['name']
      node.vm.network :private_network, network_options(host)
      node.ssh.insert_key = true
      node.vm.boot_timeout = 180
```

### Vagrant set memory and cpu's
Here we set the amount of RAM and CPU's to allocate to the virtual maching
```
      node.vm.provider host['provider'] do |vb|
        vb.name = host['name']
        vb.memory = host['mem']
        vb.cpus = host['cpus']
        vb.customize ['modifyvm', :id, '--groups', set_group]
      end
 ```
 
 ### Cisco conditional loop
 As I work on both linux hosts and network devices, I put in a conditional statement to check the host type
 
 ```
      if host['type'] == "cisco"
        # do the extra interfaces here, work with the group vars
        node.vm.network "forwarded_port", guest: 22, host: host['ansible_ssh_port'], auto_correct: true, id: "ssh"
        node.vm.network "forwarded_port", guest: 443, host: host['cisco_api_port'], auto_correct: true
        
        # node.vm.network "private_network", auto_config: false, virtualbox__intnet: "vboxnet0", mac: "0800276CEE16"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "cisco_network2", mac: "0800276CEE15"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "cisco_network3", mac: "0800276CEE14"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "cisco_network4", mac: "0800276CEE13"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "cisco_network5", mac: "0800276CEE12"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "cisco_network6", mac: "0800276CEE10"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "cisco_network7", mac: "0800276CEE09"
        
        node.vm.provider :virtualbox do |vb|  
          vb.gui = true 
          vb.customize ['modifyvm',:id,'--nicpromisc2','allow-all']
          vb.customize ['modifyvm',:id,'--nicpromisc3','allow-all']
          vb.customize ['modifyvm',:id,'--nicpromisc4','allow-all']
          vb.customize ['modifyvm',:id,'--nicpromisc5','allow-all']
          vb.customize ['modifyvm',:id,'--nicpromisc6','allow-all']
          vb.customize ['modifyvm',:id,'--nicpromisc7','allow-all']
          vb.customize ['modifyvm',:id,'--nicpromisc8','allow-all']
        end
      end
 ```
 
 ### Ansible provisioning
 Only do this if we are working on a linux host, as its going to fail on a network device:
 
 ```
      # If we supplied a bootstrap variable in the data, then execute
      # Ignore cisco, for now, only run on a linux host
      if host['type'] == "linux"
        node.vm.provision "ansible" do |ansible|
          ansible.version = data['ansible_vagrant']
          ansible.compatibility_mode = "auto"
          ansible.playbook = host['bootstrap']
        end
      end
    end
  end
end
```

#### And finally...
This set of instructions was build on my mac, and the VM's I tested with are the basic Ubuntu and Centos images. Hopefully there will be others out there who will be able to benefit from this work?

YMMV.

## References
Thanks to these resources for helping me build out the best vagrant dev envrionment ever!
- http://bertvv.github.io/notes-to-self/2015/10/05/one-vagrantfile-to-rule-them-all/
- http://hakunin.com/six-ansible-practices
- https://stackoverflow.com/questions/16708917/how-do-i-include-variables-in-my-vagrantfile
