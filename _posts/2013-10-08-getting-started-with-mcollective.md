---
layout: post
title: Getting started with MCollective
date: 2013-10-08 18:31:00
categories:
- puppet
- mcollective
---

Recently I was working on a simple continuous deployment process for puppet code.
The process steps are as follows:

1. A new puppet code is pushed to a github repository
2. [Codeship.io](http://www.codeship.io) service picks the changes, and runs the build
3. Using Capistrano (or any other SSH framework), the new code is pushed to puppet servers
4. All puppet agents are run immediately to pick up the changes, and re-provision their hosts accordingly

Although the last step looks trivial, but actually, because of puppet's communication module, it isn't so easy to do.
By default, puppet master can't initiate a direct connection to managed agents, so in order to achieve that I decided to give MCollective a try.

---

## MCollective components

[Mcollective](http://puppetlabs.com/mcollective) is a framework for building service orchestration on top of puppet regular infrastructure.

Without going into too much details, MCollective consists of 3 main components:

1. **Server** - despite it's name, it is the agent you are going to install on all your puppet managed nodes. The server daemon will listen to a specified queue for actions, and respond accordingly
2. **Client** - using this component you can fire off your orchestration jobs to all/subset of subscribed nodes (which have the server component configured correctly)
3. **Middleware** - Since MCollective architecture is pub/sub , middleware component is generally a queue, to which you will send jobs using the client, and to which the server will subscribe to listen for new actions.

## Let's play

An easy way to start playing around with MCollective is by using a [vagrant ready repository](https://github.com/ripienaar/mcollective-vagrant)

{% highlight shell-session %}
[vagrant@middleware ~]$ git clone https://github.com/ripienaar/mcollective-vagrant.git
[vagrant@middleware ~]$ cd mcollective-vagrant
{% endhighlight %}

Edit the Vagrantfile (pay attention to the amount of instances which will be previsioned in addition to the middleware. By default it is 5, and it can be a little bit intensive on some laptops)

{% highlight shell-session %}
[vagrant@middleware ~]$ vi Vagrantfile
[vagrant@middleware ~]$ vagrant up
{% endhighlight %}

After all the VM's are up (and it can take a while), connect to the middleware server (since by default the client is installed there), and check that all the nodes are configured correctly

{% highlight shell-session %}
[vagrant@middleware ~]$ mco ping
node0.example.net                        time=85.38 ms
middleware.example.net                   time=90.15 ms
{% endhighlight %}

To get all the available information on a specific node run:

{% highlight shell-session %}
[vagrant@middleware ~]$ mco inventory node0.example.net
Inventory for node0.example.net:
 
Agents:
  discovery       filemgr         integration    
  nettest         nrpe            package        
  process         puppet          rpcutil        
  service         urltest                        
 
Data Plugins:
  agent           fstat           nettest        
  nrpe            process         puppet         
  resource        service                        
 
Configuration Management Classes:
  default                        mcollective                   
  mcollective::agent::filemgr    mcollective::agent::integration
  mcollective::agent::nettest    mcollective::agent::nrpe      
  mcollective::agent::package    mcollective::agent::process   
  mcollective::agent::puppet     mcollective::agent::service   
  mcollective::agent::urltest    mcollective::config           
  mcollective::install           mcollective::service          
  motd                           nagios                        
  nrpe                           nrpe::params                  
  puppet                         repos                         
  roles::node                    settings                      
 
Facts:
  architecture => x86_64
  augeasversion => 0.9.0
  caller_module_name => mcollective
  clientcert => node0.example.net
  clientversion => 2.7.17
{% endhighlight %}

The output here is actually very helpful: 

- You can see all the classes applied to the node (Configuration Management Classes)
- All the node facts, and their values
- The MCollective data plugins and available MCollective agents (for a matter of simplicity, consider them all as MCollective functionality you can request from that node. The original terminology is a bit confusing) 

## Filters

Till now, we were addressing all the managed nodes or a specific server. However usually you would like to communicate with a subset of your environment. For that reason MCollective provides an easy way to define command filters.

For example, we can see a summary of all the node's **rubysitedir** fact

{% highlight shell-session %}
[vagrant@middleware ~]$ mco facts rubysitedir
Report for fact: rubysitedir
 
        /usr/lib/ruby/site_ruby/1.8             found 2 times
        /usr/lib/ruby/site_ruby/1.9.3           found 1 times
 
Finished processing 3 / 3 hosts in 60.00 ms
{% endhighlight %}

... and then we can easily filter the nodes with ruby version 1.8

{% highlight shell-session %}
[vagrant@middleware ~]$ mco ping -F rubysitedir=/usr/lib/ruby/site_ruby/1.8
node0.example.net                        time=53.12 ms
middleware.example.net                   time=54.14 ms
{% endhighlight %}

... or we can query by applied classes (which nodes has nagios class applied to them?)

{% highlight shell-session %}
[vagrant@middleware ~]$ mco ping -C nagios
node0.example.net                        time=49.30 ms
middleware.example.net                   time=52.98 ms
{% endhighlight %}

... also we can combine the filters for a more complex query expression

{% highlight shell-session %}
[vagrant@middleware ~]$ mco ping -S "operatingsystem=CentOS and roles::node"
node0.example.net                        time=38.22 ms
{% endhighlight %}

## Plugins

Since MCollective is a framework, by performing the default installation you actually will get very limit set of commands. The vagrant repository we were working with till now, already installed some plugins for you, but remember that in production you will need to take care of the plugins installation in addition to the basic MCollective components configuration.
You can check out the available community plugins [here](http://projects.puppetlabs.com/projects/mcollective-plugins/wiki).

For the continuous deployment process I mentioned earlier, I needed the **puppet** plugin, which provides an easy way to **enable, disable and run puppet daemons**.
For example, to rerun all puppet agents, you can just execute the following command

{% highlight shell-session %}
[vagrant@middleware ~]$ mco puppet runonce
{% endhighlight %}

## Conclusion

MCollective isn't so intuitive for a beginner, probably because of all the different components terminology and a over complicated documentation. I hope this post will ease a bit the first steps, and make this solid tool more available for a wider audience.

For advanced MCollective topics, you should review the following sources:

- [Great presentation by @repienaar](http://www.slideshare.net/PuppetLabs/presentation-16281121) with lots of useful examples
- [MCollective architecture review](http://docs.puppetlabs.com/mcollective/overview_components.html) by puppetlabs
- [Additional CLI tricks and examples](http://docs.puppetlabs.com/mcollective/reference/basic/basic_cli_usage.html)