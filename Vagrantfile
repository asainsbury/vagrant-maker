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
        vb.customize ['modifyvm', :id, '--groups', PROJECT_NAME]
      end

      node.vm.provision "ansible" do |ansible|
        ansible.version = "2.4.0.0"
        ansible.compatibility_mode = "auto"
        ansible.playbook = host['bootstrap']
      end
    end
  end
end