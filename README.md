# vagrant-maker
Ansible playbook to create a Vagrantfile and modify your .ssh/config file and also generate your hosts inventory file, all based on a set of data in groupvars.

## Introduction
This code is a way for me to streamline the building of Vagrant hosts, on my Mac.

After lots of research (Google) I've managed to come up with a compact and bijou Vagrantfile, which reads in a templated set of data (in YAMAL obviously!) and spins up the hosts when you vagrant up.  To add to the smoothness of developing with Ansible, I also create some code in .ssh/config to alias the hostname, then generate the inventory file. Another play takes care of provisioning the host with ssh keys and adds a user.
