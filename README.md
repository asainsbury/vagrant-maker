# vagrant-maker
Ansible playbook to create a Vagrantfile and modify your .ssh/config file.

## Introduction
This code is a way for me to streamline the building of Vagrant hosts, on my Mac.

After lots of research (Google) I've managed to come up with a compact and bijou Vagrantfile, which reads in a templated set of data (in YAMAL obviously!) and spins up the hosts when you vagrant up.  To add to the smoothness of developing with Ansible, I also create some code in .ssh/config to alias the hostnames .

### Aims
Pass a set of parameters in a CSV file, and create a Vagrant project, and add config to .ssh/config.
The CSV file is a reference for me to keep track of all the host details in one place, and not scatter it over multiple folders. Everything is created on one flat network, as I'm not really testing connectivty, as I would use a different tool like eve-ng for that.
