+++
title = "Over-Engineered Resume - Setting Up a Development Environment"
date = "2022-09-24"
author = "Jeff Stagg"
tags = ["resume"]
keywords = ["tech"]
description = "Provisioning a development environment with .Net 6, the Azure CLI, and Docker, using Vagrant and Ansible."
showFullContent = false
readingTime = false
hideComments = false
+++

# Over-Engineered Resume - Setting Up a Development Environment

First thing's first, we'll need a dev environment. I want to be able to work on this from a laptop running Arch, and a desktop that is either booted into a Windows session or an Arch session. We're going to start out with an API built in dotnet 6 and decide how to display it later. Let's set up a Vagrant box and throw Ubuntu on it and provision .Net and Azure on it with Ansible to get it running locally. We can forward the port and work in a local text editor since that's one of the first things I ever install on a system.

For virtualization on workstations, I use VirtualBox as my hypervisor. If you're running windows, it's as simple as downloading the installer and running through the simple wizard. If you're running Arch, I've created a post that details the few steps you need to take. Next, you'll want Vagrant. We use Vagrant to build our virtual environments through code, and we can provision our dev box by writing an Ansible playbook. On the dev box, we'll need Dotnet 6, the Azure CLI, and Docker.

## Installing the Necessary Tools
#### Install VirtualBox

If you're running Windows, this is as simple as [downloading the installer ](https://www.virtualbox.org/wiki/Downloads) and running through the wizard. Make sure you allow virtualization in your BIOS. You'll receive errors in the installer if virtualization is not enabled through your system firmware.

For Arch users, it gets a bit trickier. First, make sure virtualization is enabled in your BIOS by running the following:  
`grep -E --color 'vmx|svm' /proc/cpuinfo`

If you see either 'svm' or 'vmx' highlighted in a colored console, then you're good to go. Otherwise, reboot and edit your system BIOS before going any further.

Let's update Pacman, because you totally ran `pacman -Syu` this morning but just to settle any nerves in case something was updated while we had coffee, let's run : 

`sudo pacman -Sy`

Arch maintains a `virtualbox` package. You'll have an option to choose your host modules, and your kernel determines which you install. If you're running the Linux kernel, choose `virtualbox-host-modules-arch`. Otherwise, go with `virtualbox-host-dkms`.

`sudo pacman -S virtualbox` -> `2) virtualbox-host-modules-arch`

After installation, assuming no errors, you can try running the `virtualbox` command, and will be told that the vboxdrv kernel module is not loaded. To load it at boot, let's create a file at /etc/modules-load.d/.

`sudo nano /etc/modules-load.d/virtualbox.conf`

In the file, simply input

`vboxdrv`

Close the file, and let's add our user to the **vboxusers** group. 

`sudo usermod -aG vboxusers $(whoami)`

Now let's reboot the computer, and virtualbox should be installed and good to go. Try running it through the console, so that if any errors arise, they're easier to see.

`virtualbox`

If you are greeted with a GUI and no errors in the console, congratulations, you're ready to install Vagrant. Feel free to download the [VM VirtualBox Extension Pack](https://www.virtualbox.org/wiki/Downloads) as well. When you open VirtualBox, you can click File > Preferences > Extensions > Add, and select the downloaded file. I usually add it for USB support in VMs, but we won't be needing it for this project.

If you have permissions issues when trying to install the extension pack, close out of the Virtualbox GUI, and relaunch with privileges

`sudo virtualbox`

Then install the expansion pack.

#### Installing Vagrant

[Vagrant](https://www.vagrantup.com/downloads) is the easy one to install. If you're in Windows, download the file for your appropriate architecture and run the installer. If you're running Arch, just run  
`sudo pacman -S vagrant`

At this point, the only thing we need is a text editor if you don't already have one installed. Any one will do.

## Provisioning our Dev Environment

Since we'll be using dotnet 6 to build our API, and will be deploying onto Azure, let's set those up to be installed on our dev box when it's provisioned. We don't have a full plan to launch yet, but I know I want to ship it in a container, so let's install Docker as well. These things can always change later when we decide to over-engineer our deployments.

#### Provisioning with Ansible

Let's create our Vagrantfile, and use `ansible_local` to provision so that we don't have to set up an ansible host yet. 

```
Vagrant.configure("2") do |config|
    config.vm.box = "bento/ubuntu-22.04"
    config.vm.network "forwarded_port", guest: 80, host: 8080
  
    config.vm.provision "ansible_local" do |ansible|
        ansible.playbook = "playbook.yml"
      end
end
```

Next, let's create the `playbook.yml` file.

```
---
- hosts: all
  become: true
  vars:
    container_count: 4
    default_container_name: docker
    default_container_image: ubuntu
    default_container_command: sleep 1
  tasks:
    - name: Install aptitude
      apt:
        name: aptitude
        state: latest
        update_cache: true

  
    - name: Install required system packages
      apt:
        pkg:
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
          - python3-pip
          - virtualenv
          - python3-setuptools
          - acl
        state: latest
        update_cache: true

	- name: Create User
	  ansible.builtin.user:
	  name: jeff
	  comment: Jeff Stagg
	  uid: 1010
	  group: sudo

    - name: Install development SDKs
      apt:
        pkg:
          - dotnet6

    - name: Install Azure CLI
      ansible.builtin.shell: curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
      become: yes
      become_user: jeff
  
    - name: Add Docker GPG apt Key
      apt_key:
        url: https://download.docker.com/linux/ubuntu/gpg
        state: present
  
    - name: Add Docker Repository
      apt_repository:
        repo: deb https://download.docker.com/linux/ubuntu focal stable
        state: present
  
    - name: Update apt and install docker-ce
      apt:
        name: docker-ce
        state: latest
        update_cache: true
  
    - name: Install Docker Module for Python
      pip:
        name: docker
        
    - name: Pull default Docker image
      community.docker.docker_image:
        name: "{{ default_container_image }}"
        source: pull
  
    - name: Create default containers
      community.docker.docker_container:
        name: "{{ default_container_name }}{{ item }}"
        image: "{{ default_container_image }}"
        command: "{{ default_container_command }}"
        state: present
      with_sequence: count={{ container_count }}
```


Let's go over our playbook:

First we grab the latest from apt, and install system packages. These are just basic things we do to a new Ubuntu installation anyway. We don't really need git in our environment, as we'll already have it installed on our host, and we won't have to worry about setting up a user and messing with SSH keys in our temporary dev box.

Next we set up dotnet6, which will give us our dev tools. To install the Azure CLI from the one-liner script they provide, we have to run the command as a non-root user. So we create a user using the built-in user component, assign it to group **sudo** and use the `become_user` property to de-escalate our user, in order to run the Microsoft script to install the Azure CLI.

Now that our tools are ready to build our code and deploy our infrastructure, we install Docker to help with deployment.

## Launching our Environment

Now that we have the tools we need in our environment, let's spin up our new dev box with the command 

`vagrant up`

This will download Ubuntu into a new Virtualbox VM, then run through our Ansible playbook to install all of the tools we'll need for this project. We can enter our dev box through SSH by using the command

`vagrant ssh`

Your username and password are `vagrant / vagrant`.

Let's test that our tools are working by running:

`dotnet --version`  

```
vagrant@vagrant:~$ dotnet --version
6.0.109
```

`az -v`  

```
vagrant@vagrant:~$ az -v
azure-cli                         2.40.0

core                              2.40.0
telemetry                          1.0.8

Dependencies:
msal                            1.18.0b1
azure-mgmt-resource             21.1.0b1

Python location '/opt/az/bin/python3'
Extensions directory '/home/vagrant/.azure/cliextensions'

Python (Linux) 3.10.5 (main, Sep  2 2022, 05:41:19) [GCC 11.2.0]

Legal docs and information: aka.ms/AzureCliLegal


Your CLI is up-to-date.
```

`sudo docker ps -a`

```
vagrant@vagrant:~$ sudo docker ps -a 
CONTAINER ID   IMAGE     COMMAND     CREATED         STATUS    PORTS     NAMES
3c85a5007e7a   ubuntu    "sleep 1"   8 minutes ago   Created             docker4
872783464fe9   ubuntu    "sleep 1"   8 minutes ago   Created             docker3
8311acc96a3f   ubuntu    "sleep 1"   8 minutes ago   Created             docker2
5942c514536e   ubuntu    "sleep 1"   8 minutes ago   Created             docker1
```

If all is good, let's move on to Building Our Project Framework.

## Troubleshooting

I ran into the following error:
```
==> default: Running provisioner: ansible_local...
    default: Installing Ansible...
The following SSH command responded with a non-zero exit status.
Vagrant assumes that this means the command failed!


                add-apt-repository ppa:ansible/ansible -y &&                 apt-get update -y -qq &&                 DEBIAN_FRONTEND=noninteractive apt-get install -y -qq ansible --option "Dpkg::Options::=--force-confold"
              

Stdout from the command:

Hit:1 http://in.archive.ubuntu.com/ubuntu jammy InRelease
Get:2 http://in.archive.ubuntu.com/ubuntu jammy-updates InRelease [114 kB]
Get:3 https://ppa.launchpadcontent.net/ansible/ansible/ubuntu jammy InRelease [18.0 kB]
Get:4 http://in.archive.ubuntu.com/ubuntu jammy-backports InRelease [99.8 kB]
Get:5 http://in.archive.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
Get:6 https://ppa.launchpadcontent.net/ansible/ansible/ubuntu jammy/main amd64 Packages [1,128 B]
Get:7 https://ppa.launchpadcontent.net/ansible/ansible/ubuntu jammy/main Translation-en [756 B]
Reading package lists...
Repository: 'deb https://ppa.launchpadcontent.net/ansible/ansible/ubuntu/ jammy main'
Description:
Ansible is a radically simple IT automation platform that makes your applications and systems easier to deploy. Avoid writing scripts or custom code to deploy and update your applications— automate in a language that approaches plain English, using SSH, with no agents to install on remote systems.

http://ansible.com/

If you face any issues while installing Ansible PPA, file an issue here:
https://github.com/ansible-community/ppa/issues
More info: https://launchpad.net/~ansible/+archive/ubuntu/ansible
Adding repository.
Adding deb entry to /etc/apt/sources.list.d/ansible-ubuntu-ansible-jammy.list
Adding disabled deb-src entry to /etc/apt/sources.list.d/ansible-ubuntu-ansible-jammy.list
Adding key to /etc/apt/trusted.gpg.d/ansible-ubuntu-ansible.gpg with fingerprint 6125E2A8C77F2818FB7BD15B93C4A3FD7BB9C367


Stderr from the command:

E: Release file for http://in.archive.ubuntu.com/ubuntu/dists/jammy-updates/InRelease is not valid yet (invalid for another 4h 12min 3s). Updates for this repository will not be applied.
E: Release file for http://in.archive.ubuntu.com/ubuntu/dists/jammy-backports/InRelease is not valid yet (invalid for another 3h 14min 10s). Updates for this repository will not be applied.
E: Release file for http://in.archive.ubuntu.com/ubuntu/dists/jammy-security/InRelease is not valid yet (invalid for another 4h 11min 51s). Updates for this repository will not be applied.
E: Release file for http://in.archive.ubuntu.com/ubuntu/dists/jammy-updates/InRelease is not valid yet (invalid for another 4h 11min 59s). Updates for this repository will not be applied.
E: Release file for http://in.archive.ubuntu.com/ubuntu/dists/jammy-backports/InRelease is not valid yet (invalid for another 3h 14min 6s). Updates for this repository will not be applied.
E: Release file for http://in.archive.ubuntu.com/ubuntu/dists/jammy-security/InRelease is not valid yet (invalid for another 4h 11min 47s). Updates for this repository will not be applied.
```

This tells me that my system clock is off. I get this a lot when things go awry dual-booting Windows and Arch on the same box. I set my system time with the following command:

`# timedatectl set-time "2022-09-24 09:58:54"`

Then I was able to run `vagrant provision` to finish setting up my dev box and run through the playbook.