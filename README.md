# vagrant-maker
Ansible playbook to create a Vagrantfile and modify your .ssh/config file and also generate your hosts inventory file, all based on a set of data in groupvars.

- [vagrant-maker](#vagrant-maker)
  * [Introduction](#introduction)
- [The vagrant-maker playbook](#the-vagrant-maker-playbook)
- [Bootstrapping the Linux hosts](#bootstrapping-the-linux-hosts)
  * [The Group Vars](#the-group-vars)
    + [Host list](#host-list)
    + [SSH config](#ssh-config)
    + [Hosts inventory](#hosts-inventory)
  * [Vagrantfile](#vagrantfile)
    + [Importing the yaml data structure:](#importing-the-yaml-data-structure-)
    + [Networking function:](#networking-function-)
    + [Vagrant host loop](#vagrant-host-loop)
    + [Vagrant configure boxes](#vagrant-configure-boxes)
    + [Vagrant set memory and cpu's](#vagrant-set-memory-and-cpu-s)
    + [Cisco conditional loop](#cisco-conditional-loop)
    + [Ansible provisioning](#ansible-provisioning)
      - [And finally...](#and-finally)
  * [References](#references)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>


## Introduction
This code is a way for me to streamline the building of Vagrant hosts, on my Mac.

After lots of research (Google) I've managed to come up with a compact and bijou Vagrantfile, which reads in a templated set of data (in YAML obviously!) and spins up the hosts when you vagrant up.  To add to the smoothness of developing with Ansible, I also create some code in .ssh/config to alias the hostname, then generate the inventory file. Another play takes care of provisioning the host with ssh keys and adds a user.

This allows you to be able to ssh into all the VM's and between all VM's for fast development.

Run the playbook `ansible-playbook vagrant-maker.yml`

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

# The vagrant-maker playbook 
It was surprisingly simple to generate the files, and it uses just 2 modules 
- 'blockinfile' for the ssh
- 'lineinfile' for the inventory file

Both tasks have a state of presence, so you can remove the config by simply changing the state field in the groupvars.

```
---
- hosts: localhost
  gather_facts: false

  vars:
    update: true

  tasks:      
    - name: Check inventory paths exist and create if not
      file:
        path: "{{ host_path }}"
        state: directory

      tags:
        - host_path

    - name: Check that the host file exists
      stat:
        path: "{{ host_file }}"
      register: stat_result
      tags: host_file

    - name: Create the file, if it doesn't exist already
      file:
        path: "{{ host_file }}"
        state: touch
      when: stat_result.stat.exists == False  
      tags: host_file
 

    - name: Add hosts for vagrant into ssh config
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

      tags: sshconfg
  

    - name: Add the Vagrant Boxes
      shell: 'vagrant box add {{item}} --insecure --provider virtualbox'
      register: result

      with_items:
        - "{{box_list}}"

      failed_when: "'A name is required' in result.stderr"
      when: update
      tags: box_add
```
<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

# Bootstrapping the Linux hosts
Post vagrant up, calls this playbook, which sets up the environment for simple ssh access. This have been tested on both Ubuntu and Centos, as listed in the groupvars section.

```
---
- hosts: all
  gather_facts: yes
  become: true
  
  # Vagrant provision runs this file, so you don't actually need an inventory
  # it does that for you.
  # Basically we setup a bunch of environment stuff so we can ssh into the host
  # Using all the data from all.yml

  tasks:
    - name: Add a special package, only on Centos or RHEL https://github.com/bayandin/webpagetest-private/issues/1
      package:
        name: libselinux-python
        state: present
      when: ansible_distribution == 'CentOS' or ansible_distribution == 'Red Hat Enterprise Linux'

    - name: Create User
      user:
        name: "{{ssh_user}}"
        password: "{{ 'password' | password_hash('sha512') }}"
        shell: /bin/bash
        append: yes
        generate_ssh_key: yes
        ssh_key_bits: 1024
        ssh_key_file: .ssh/id_rsa

    - name: Add a special package, only on Centos or RHEL https://github.com/bayandin/webpagetest-private/issues/1
      package:
        name: libselinux-python
        state: present
      when: ansible_distribution == 'CentOS' or ansible_distribution == 'Red Hat Enterprise Linux'

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
        policy: targeted
        state: permissive
      when: ansible_distribution == 'CentOS' or ansible_distribution == 'Red Hat Enterprise Linux'

    - name: Copy over the ssh config file
      copy:
        src: "{{ssh_config}}"
        dest: /home/user/.ssh/config
        owner: "{{ssh_user}}"
        mode: 0600
   
    # Dependencies: 
    # Make some dummy keys on one of the VM's 
    # and place them in the root of this play
    - name: Copy over the dummy pub ssh keys to each node
      copy:
        src: id_rsa.pub
        dest: "{{ssh_home}}.ssh/id_rsa.pub"
        owner: "{{ssh_user}}"
        mode: 0600

    - name: Copy over the dummy priv ssh keys to each node
      copy:
        src: id_rsa
        dest: "{{ssh_home}}.ssh/id_rsa"
        owner: "{{ssh_user}}"
        mode: 0600

    - name: Set authorized key from file
      authorized_key:
        user: "{{ssh_user}}"
        state: present
        key: "{{ lookup('file', 'id_rsa.pub') }}"

    - name: Set authorized key from file from the control machine
      authorized_key:
        user: "{{ssh_user}}"
        state: present
        key: "{{ lookup('file', '{{ssh_pub}}') }}"



  handlers:
    - name: restart_ssh
      systemd:
        name: sshd
        state: restarted
```
<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

## The Group Vars
It all starts at the beginning, which is a group vars files with paths and all sorts of other stuff. Look at this file for all the variables.

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

### Host list
This is the list we are  going to use to generate ssh config and hosts inventory, but also is used by the vagrant file to build the virtual machines.

```
host_list:
  - name: dev1
    provider: virtualbox
    group: dev
    ip: "1.1.1.10"
    vm_name: dev1
    mem: 512
    cpus: 1
    box: 'centos/7'
    box_url: 'https://vagrantcloud.com/centos/boxes/7'
    bootstrap: 'bootstrap.yml'
    type: "linux"
    state: "present"

  - name: dev2
    provider: virtualbox
    group: dev
    ip: "1.1.1.11"
    vm_name: dev2
    mem: 512
    cpus: 1
    box: 'centos/7'
    box_url: 'https://vagrantcloud.com/centos/boxes/7'
    bootstrap: 'bootstrap.yml'
    type: "linux"
    state: "present"

  - name: dev3
    provider: virtualbox
    group: dev
    ip: "1.1.1.12"
    vm_name: dev3
    mem: 512
    cpus: 1
    box: 'centos/7'
    box_url: 'https://vagrantcloud.com/centos/boxes/7'
    bootstrap: 'bootstrap.yml'
    type: "linux"
    state: "present"
```

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

### SSH config
The playbook iterates over the list and generates blocks of text for the ssh config:

```
Host *
    ServerAliveInterval 120
# BEGIN ANSIBLE MANAGED BLOCK dev1
Host dev1 *.test.local
  hostname 1.1.1.10
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
  User user
  LogLevel ERROR
# END ANSIBLE MANAGED BLOCK dev1
# BEGIN ANSIBLE MANAGED BLOCK dev2
Host dev2 *.test.local
  hostname 1.1.1.11
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
  User user
  LogLevel ERROR
# END ANSIBLE MANAGED BLOCK dev2
# BEGIN ANSIBLE MANAGED BLOCK dev3
Host dev3 *.test.local
  hostname 1.1.1.12
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
  User user
  LogLevel ERROR
# END ANSIBLE MANAGED BLOCK dev3
```

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

### Hosts inventory
I decided to chop this bit out as it wasn't working as expected.

```
[dev]
dev1 ansible_ssh_host=1.1.1.10
dev2 ansible_ssh_host=1.1.1.11
dev3 ansible_ssh_host=1.1.1.12
```

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

## Vagrantfile
This is where all the fun begins.

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

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

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

### Networking function:
Here I used (checkout references section) a function I found to generate the networking stuff

```
# Define a nice function to sort out all the networking
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

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

### Vagrant host loop
At this point we can start a loop, to iterate over the data we set in the groupvars and build the box

```
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  data['host_list'].each do |host|
    
    set_group = '/' + host['group']
```
<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

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

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

### Vagrant set memory and cpu's
Here we set the amount of RAM and CPU's to allocate to the virtual machine:
```
      node.vm.provider host['provider'] do |vb|
        vb.name = host['name']
        vb.memory = host['mem']
        vb.cpus = host['cpus']
        vb.customize ['modifyvm', :id, '--groups', set_group]
      end
 ```

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

 ### Cisco conditional loop
 As I work on both Linux hosts and network devices, I put in a conditional statement to check the host type
 
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
 
 <div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

 ### Ansible provisioning
 Only do this if we are working on a Linux host, as its going to fail on a network device:
 
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

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>

#### And finally...
This set of instructions was build on my mac, and the VM's I tested with are the basic Ubuntu and Centos images. Hopefully there will be others out there who will be able to benefit from this work?

YMMV.

## References
Thanks to these resources for helping me build out the best vagrant dev envrionment ever!
- http://bertvv.github.io/notes-to-self/2015/10/05/one-vagrantfile-to-rule-them-all/
- http://hakunin.com/six-ansible-practices
- https://stackoverflow.com/questions/16708917/how-do-i-include-variables-in-my-vagrantfile

<div align="right">
    <b><a href="#top">↥ back to top</a></b>
</div>
<br/>
