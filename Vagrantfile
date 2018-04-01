# -*- mode: ruby -*-
# vi: set ft=ruby 

# References to other peoples work, who I've massively plgerised:
# http://bertvv.github.io/notes-to-self/2015/10/05/one-vagrantfile-to-rule-them-all/
# http://hakunin.com/six-ansible-practices
# https://stackoverflow.com/questions/16708917/how-do-i-include-variables-in-my-vagrantfile

require 'yaml'

VAGRANTFILE_API_VERSION = '2'
PROJECT_NAME = '/' + File.basename(Dir.getwd)


hosts = YAML.load_file('hosts.yml')

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


Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  hosts.each do |host|
    config.vm.define host['name'] do |node|

      if Vagrant.has_plugin?("vagrant-cachier")
        config.cache.disable!
      end

      node.vm.box = host['box']
      node.vm.box_url = host['box_url']
      node.vm.hostname = host['name']
      node.vm.network :private_network, network_options(host)
      node.ssh.insert_key = true
      node.vm.boot_timeout = 180
      node.vm.synced_folder '.', '/vagrant', disabled: true

      if host['type'] == "cisco"
        # do the extra interfaces here, work with the group vars
        node.vm.network "forwarded_port", guest: 22, host: host['ansible_ssh_port'], auto_correct: true, id: "ssh"
        node.vm.network "forwarded_port", guest: 443, host: host['cisco_api_port'], auto_correct: true
        
        # node.vm.network "private_network", auto_config: false, virtualbox__intnet: "vboxnet0", mac: "0800276CEE16"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "nxosv_network2", mac: "0800276CEE15"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "nxosv_network3", mac: "0800276CEE14"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "nxosv_network4", mac: "0800276CEE13"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "nxosv_network5", mac: "0800276CEE12"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "nxosv_network6", mac: "0800276CEE10"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "nxosv_network7", mac: "0800276CEE09"
        
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
  
      node.vm.provider host['provider'] do |vb|
        vb.name = host['name']
        vb.memory = host['mem']
        vb.cpus = host['cpus']
        vb.customize ['modifyvm', :id, '--groups', host['group']]
      end

      # If we supplied a bootstrap variable in the data, then execute
      # 
      if host['bootstrap']
        node.vm.provision "ansible" do |ansible|
          ansible.version = "2.4.0.0"
          ansible.compatibility_mode = "auto"
          ansible.playbook = host['bootstrap']
        end
      end
    end
  end
end