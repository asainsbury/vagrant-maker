# -*- mode: ruby -*-
# vi: set ft=ruby 

# References to other peoples work, who I've massively plagiarized:
# http://bertvv.github.io/notes-to-self/2015/10/05/one-vagrantfile-to-rule-them-all/
# http://hakunin.com/six-ansible-practices
# https://stackoverflow.com/questions/16708917/how-do-i-include-variables-in-my-vagrantfile


VAGRANTFILE_API_VERSION = '2'
# Bit of error checking around the external data source:
require 'yaml'
if File.file?('group_vars/all.yml')
  data = YAML.load_file('group_vars/all.yml')
else
  raise "Configuration file does not exist."
end


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

# As the data is sourced from a hash via group_vars we need to reference the key
# Which is 'list' but the group thing is broken.
# Only seems to work when you grab the folder name?

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

  # Call the function network_options
  data['host_list'].each do |host|
    
    set_group = '/' + host['group']

    config.vm.define host['name'] do |node|
      node.vm.box_download_insecure = true
      node.vm.box = host['box']
      node.vm.box_url = host['box_url']
      node.vm.hostname = host['name']
      node.vm.network :private_network, network_options(host)
      node.ssh.insert_key = true
      node.vm.boot_timeout = 180

      node.vm.provider host['provider'] do |vb|
        vb.name = host['name']
        vb.memory = host['mem']
        vb.cpus = host['cpus']
        vb.customize ['modifyvm', :id, '--groups', set_group]
      end
      
      if host['type'] == "cisco"

        # do the extra interfaces here, work with the group vars
        node.vm.network "forwarded_port", guest: 22, host: host['ansible_ssh_port'], auto_correct: true, id: "ssh"
        node.vm.network "forwarded_port", guest: 443, host: host['cisco_api_port'], auto_correct: true, id: "https"
        
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
      
      # If we supplied a bootstrap variable in the data, then execute
      # Ignore Cisco, only run on a Linux host
      if host['type'] == "linux"
        node.vm.provision "ansible" do |ansible|

          # A little check for the level of python required
          if host['python3']
            ansible.extra_vars = { ansible_python_interpreter: "/usr/bin/python3" }
          end
         
          # Finish off the provisioning 
          ansible.compatibility_mode = "auto"
          ansible.playbook = host['bootstrap']

        end
      end
    end
  end
end