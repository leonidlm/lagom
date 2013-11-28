---
layout: post
title: Creating hadoop test environment with ansible and vagrant
date: 2013-11-19 18:31:00
categories:
- ansible
- hadoop
- devops
---

I love Puppet!

I always hated shell scripting. 

When I discovered Puppet a year ago, I finally found a good programmable way to describe the infrastructure I am working with.
A lot changed since my first encounter with Puppet, especially, since plenty of new and interesting tools appeared which compete with Puppet for the heart of a sysadmin.

One of these shiny new configuration management tools is [Ansible](http://www.ansibleworks.com/). I decided to give it a spin, and to configure a local Hadoop development environment using it and vagrant.

---

## Prerequisites
In this tutorial I will use vagrant version [1.3.5](http://docs.vagrantup.com/v2/installation/), and VirtualBox version 4.1.12 on Ubuntu. 
Ansible must be installed on the same system as vagrant because the vagrant Ansible provisioner relies on that to perform it's magic. 
I used [ansible](http://www.ansibleworks.com/docs/intro_installation.html) version 1.3.3 since it supports the roles functionality which we will use in this tutorial.

## Vagrant configuration
Let's start by downloading Ansible example modules into your working directory. We will use the Hadoop playbook to do most of the heavy lifting for us.

{% highlight shell-session %}
[leonid@laptop ~]$ wget https://github.com/ansible/ansible-examples/archive/master.zip
[leonid@laptop ~]$ unzip master.zip ansible-examples-master/hadoop/*
{% endhighlight %}

For the Vagrant configuration, I used the following Vagrantfile.

{% highlight ruby %}
VAGRANTFILE_API_VERSION = "2"

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "centos_6_3_x86_64"
  config.vm.box_url = "http://files.vagrantup.com/precise64.box"

  config.vm.synced_folder ".", "/vagrant"

  config.vm.provider "virtualbox" do |v|
    v.customize ["modifyvm", :id, "--cpus", "1", "--memory", "512"]
  end

  config.vm.define "hadoop_master" do |hadoop_master|
    hadoop_master.vm.network "private_network", ip: "192.168.50.4"
    hadoop_master.vm.hostname = "master.internal"
  end

  config.vm.define "hadoop_slave1" do |hadoop_slave1|
    hadoop_slave1.vm.network "private_network", ip: "192.168.50.5"
    hadoop_slave1.vm.hostname = "slave1.internal"
  end

  config.vm.define "hadoop_slave2" do |hadoop_slave2|
    hadoop_slave2.vm.network "private_network", ip: "192.168.50.6"
    hadoop_slave2.vm.hostname = "slave2.internal"
    
    hadoop_slave2.vm.provision :ansible do |ansible|
      ansible.inventory_path = "inventory"
      ansible.verbose = "v"
      ansible.sudo = true
      ansible.playbook = "ansible-examples-master/hadoop/site.yml"
    end
  end
end
{% endhighlight %}

We are going to provision a simple Hadoop cluster, without HA configuration, which will consist of one master node and 2 slave nodes.

I limited the VM resources to 1 cpu and 512MB ram per each VM, since I didn't want this environment to interfere with other stuff I am running on my laptop:

{% highlight ruby %}
config.vm.provider "virtualbox" do |v|
  v.customize ["modifyvm", :id, "--cpus", "1", "--memory", "512"]
end
{% endhighlight %}

Also, I configured 3 VM instances which will be provisioned and managed by Vagrant. 
I assigned ip addresses and hostnames to each VM, which will be very handy later.

{% highlight ruby %}
config.vm.define "hadoop_master" do |hadoop_master|
  hadoop_master.vm.network "private_network", ip: "192.168.50.4"
  hadoop_master.vm.hostname = "master.internal"
end

config.vm.define "hadoop_slave1" do |hadoop_slave1|
  hadoop_slave1.vm.network "private_network", ip: "192.168.50.5"
  hadoop_slave1.vm.hostname = "slave1.internal"
end

config.vm.define "hadoop_slave2" do |hadoop_slave2|
  hadoop_slave2.vm.network "private_network", ip: "192.168.50.6"
  hadoop_slave2.vm.hostname = "slave2.internal"
{% endhighlight %}

I assigned the Ansible provisioner to the last VM not by chance, this step is actually very important.
In Vagrant, in a multi-vm configuration, VM machines will be started according to their declaration order. I used that knowledge to my advantage, in order to make sure that the provisioning process will be executed only after all the VMs are up and running. 

Why it is important? because otherwise Ansible will fail, since it will try to connect to some not-yet-created machines.
This technique (others will call it a hack) is described in more detail [in this github issue](https://github.com/mitchellh/vagrant/issues/1784)

{% highlight ruby %}
hadoop_slave2.vm.provision :ansible do |ansible|
  ansible.inventory_path = "inventory"
  ansible.verbose = "v"
  ansible.sudo = true
  ansible.playbook = "ansible-examples-master/hadoop/site.yml"
end
{% endhighlight %}

## Ansible

I am not willing to dive deep into Ansible, but it is important to note it's 2 main configuration files: Ansibles **inventory** and **playbook.yml** files.

The inventory file defines which machines will be managed by Ansible and which groups these machines are assigned to (and by group assignment you actually telling which roles should be assigned to these nodes). I think that it is a very powerful concept, which as everything else in Ansible, is implemented radically simpler than in Puppet. You can find more information about the inventory file [here](http://www.ansibleworks.com/docs/intro_inventory.html)

Here is the content of the inventory file I used:

{% highlight ini %}
[hadoop_all:children]
hadoop_masters
hadoop_slaves

[hadoop_master_primary]
master.internal ansible_ssh_host=192.168.50.4

[hadoop_master_secondary]

[hadoop_masters:children]
hadoop_master_primary
hadoop_master_secondary

[hadoop_slaves]
slave1.internal ansible_ssh_host=192.168.50.5
slave2.internal ansible_ssh_host=192.168.50.6
{% endhighlight %}


Another important file is the playbook yml file. I used the default site.yml (`ansible-examples-master/hadoop/site.yml`) which comes with the example repository. It is easier to view this file as the starting point of the whole provisioning process, and Ansible's execution process starts from there.

## Adjusting Hadoop playbooks

I think I never had a chance to download a community playbook/module/cookbook that worked from the first run, and Ansible's hadoop playbooks aren't an exception.
In order to make them work with our setup, we need to adjust a bit the default configuration.

First of all, we will needed to install the `libselinux-python` as a prerequisite for the selinux module to work. 
To do that add the following line to the `ansible-examples-master/hadoop/roles/common/tasks/common.yml` file:

{% highlight yaml %}
- name: Install SElinux ansible module dependencies
  yum: name=libselinux-python state=installed
- name: Disable SELinux in conf file
  selinux: state=disabled
{% endhighlight %}

Secondly, by default, the example playbooks create the `/etc/hosts` file on all the VM's with the internal ip.
In order that all our VM's will be able to communicate with each other, we will change the default parameters to use the extermal ip instead.
To do so, change the following line in the `ansible-examples-master/hadoop/group_vars/all` file:

{% highlight yaml %}
iface: eth1
{% endhighlight %}

And as our last change for today, we will needed to explicitly set the permissions of hadoop configuration to 644 in order to make sure we won't fail on permissions during the provisioning.
Let's do so, by adding the `mode=644` line to `ansible-examples-master/hadoop/roles/hadoop_primary/tasks/hadoop_master_no_ha.yml` file:

{% highlight yaml %}
- name: Copy the hadoop configuration files for no ha
  template: src=roles/common/templates/hadoop_conf/{{ item }}.j2 dest=/etc/hadoop/conf/{{ item }} mode=644
  with_items: 
   - core-site.xml
   - hadoop-metrics.properties
   - hadoop-metrics2.properties
{% endhighlight %}

## Run!

Finally everything is ready for launch! 
Now just hit `vagrant up` at the command line, and in about 10 minutes you will have a working hadoop environment.

Let's go another extra mile, and check that our hadoop is working.
Review the jobtracker webpage by accessing `http:/192.168.50.4:50030/jobtracker.jsp` in your browser. It should show up some basic jobtracker stats.

For a more elaborate use of our environment, you will need to adjust the playbooks even more. 
For example, you will probably need to create the mapred directories. Since Ansible is so easy to start with, I am sure that you will be able to catch up easily, and to complete these extra steps without a hassle.

## Summary

My first impression of Ansible is quite positive, I can tell that it is much easier to start using this tool than puppet, and actually it seems quite powerful. I will need to test it in a production scenario to see if it will work for a more complex scenarios.