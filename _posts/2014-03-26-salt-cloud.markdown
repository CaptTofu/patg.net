---
layout: post
title: "Using Salt Cloud"
date: 2014-03-23 09:00:00
categories: provisioning 
---

# Using Salt Cloud

Recently, I took an interest in trying out Salt Cloud, the provisioning tool component of SaltStack, the provisioning to that offers "fast, scalable and flexible software for data center automation, from infrastructure and any cloud, to the entire application stack". 

I was attempting at first to get it to run with HP Cloud as well as a simple cloud setup with Devstack and found that I had to actually read the source code to really know how to get it to work. That was when I decided to contribute to the project both with documentation and a small fix to the OpenStack driver, pep8/pyflakes cleanups and documentation.

## Salt and Salt-cloud 

There is a lot of information about Salt itself, but this post is focused on Salt Cloud.

As just mentioned, Salt is provisioning software, just as Chef, Puppet and Ansible are. Salt works with a Salt master which then runs states on minions (clients). These minions run the commands that the state, when run, results in the system being as expected according to the state. The master is cognizant of its minions and stores data used in the provisioning process. 

Salt is written in Python and uses YAML templates for both its "states" as well as its "pillars". States are YAML files that are a representation of a state in which a system should be in. Pillars are tree-like structures defined on the Salt master and passed to the minions. Pillar data is represented in YAML files as well. 

In addition to states, it's possible to run single commands using Salt by way of the Salt master and on all or specific minions.

Salt Cloud, as already mentioned, is the provisioning tool component of Salt. Often, you will see in cloud-based organizations that different teams might use a tool such as Chef or Salt, though they will often script the functionality that orachestrates the launching and/or deletion of instances. For instance, I worked on one team that used Chef and we had a very nice tool written in Ruby that would launch instances that would load cookbooks for and Chef would then provision. This approach can give one a lot of control but also be hard to maintain as well as being very application or cloud-specific. 

Salt-cloud has the approach of keeping the provisioning, instance creation and deletion aspect of salt under the same system and by doing so, the same concepts that Salt uses, namely YAML being the means to express states and pillars naturally is also used for Salt Cloud.

When Salt Cloud is used to launch an instance, that instance is not only brought up, but set up as a minion being managed by the master that launched it.

The other good think about Salt Cloud is that it is built on top of Apache LibCloud which makes it possible to use Salt Cloud with any number of cloud providers such as EC2, Google Compute Engine, LXC, Linode, Azure and of course OpenStack providers such as HP Cloud and Rackspace.

## Salt Cloud concepts

Salt Cloud uses the following concepts that I discovered upon using it:

- Cloud Provider: a particular cloud connection including the username, password, any sort of API key, region as well as authentication endpoints.
- Cloud profile: a particular type of image to use when launching an instance or container. This being, the size/flavor, image ID and the cloud provider to use

## Salt Cloud with HP Cloud

One of the first things I wanted to try to get working properly was the ability to launch instances with HP Cloud. This proved difficult at first and the documentation was dated for the initial version of HP Cloud (1.0, based off of Essex and using Nova Networking) whereby when launching instances, external IP addresses were automatically assigned to an instance, whereas with the newer HP Cloud (1.1, based off of Havana and using Neutron), one has to first create a floating IP and then add it to a specific instance.

Why is this a problem? Well, because when the command salt-cloud is run, it needs to not only be able to launch an instance but also connect to it through SSH and install the minion to be able to talk to the master. Without an IP to connect to, this won't happen, and the examples that were in the documentation did not work with HP Cloud 1.1. 

Also do note, this post is to introduce the reader to concepts. For more in-depth reading on the matter, it is strongly advised to review the [Salt Cloud docs][salt-cloud-docs], which the author recently contributed to for easier Salt Cloud usage with OpenStack and HP Cloud in particular!

## Setting up a cloud provider

As previously stated, a cloud provider is exactly what the term means. One can have numerous cloud providers to chose from in any given setup, including different vendors that use different cloud technologies. These cloud providers have YAML-based configuration files typically located in ``/etc/salt/cloud.profiles.d``

Example:

    $:/etc/salt/cloud.providers.d$ ls
    hpcs_ae1.conf   openstack_grizzly.conf hpcs_aw2.conf
    rackspace_dfw.conf rackspace_ord.conf

And example of one would be:

    hpcs_ae1:
      minion:
        master: mymaster.domain 
    
      identity_url: https://region-b.geo-1.identity.hpcloudsvc.com:35357/v2.0/tokens
      compute_name: Compute
      ignore_cidr: 10.0.0.1/24
      networks:
        - floating:
          - Ext-Net
        - fixed:
          - my-network
      protocol: ipv4
      compute_region: region-b.geo-1
      user: clouduser 
      tenant:  clouduser-project1
      password : xxxx 
      ssh_key_name: test
      ssh_key_file: /root/keys/test.key
      provider: openstack

In the above example, their are various connection metadata that is used to establish a connection to HP Cloud. This would be the same for any cloud provider and would vary according to the particular attributes of how one connects to a cloud. For instance, with Rackspace, one would also provide an ``apikey`` parameter. For ec2, ``provider`` would be ``ec2`` (obviously). 

The ``networks`` setting provides one or more named networks that you provide. You will need to know these in advance if you intend to use them. In the example above, they are Ext-Net (default network with HP Cloud) and a network the user created ``my-network``. These are networks that the salt-cloud OpenStack driver uses to obtain a list of floating IP addresses, picking one to attach to the instance being provisioned in order to be able to set up the instance as a minion. 

Interestingly, this is where I recently [modified][salt-openstack-contrib] the driver and contributed to salt-cloud code that would negate the need for specifying ``networks`` and instead simply obtain floating IPs if not specified regardless of network.  

The ``ignore_cidr`` setting is for listing a range of IP addresses to not even bother using as an address to connect to for setting up the minion. In this case, even though salt-cloud will not attempt to use the private IP address for the instance, using ``ignore_cidr`` will making it so salt-cloud doesn't bother even identifying that the IP address is private.

NOTE: If one wants to use the private IP address to connect to the instance-- for instance, if the master is in the same network in the cloud as the minions (actually commonplace) then one need only have the following in their cloud provider configuration file:

    ssh_interface: private_ips

This will result in salt-cloud using the private IP address when using SSH to connect to the instance during setup of the minion.

So, it is clearly a simple setup and clean YAML representation per the philosophy of Salt.

## Testing the cloud profile

First of all, it is assumed that the reader of this post has already set up a python virtual environment. Other installation steps and issues include:

- Installing Salt within the vurtual environment
- One ought to set up the master (see [Salt Documentation][salt-docs]) 
- You must run salt-cloud as root. This is a bit of a conundrum because the documentation clearly states that you ought to run a virtualenv setup for development yet you need to run it as root. I set up the virtualenv as suggested and ran as root and it works sufficiently.

### Listing cloud providers


## Setting up a cloud profile

A cloud profile is yet another YAML file that represents how a particular instance is launched: what size/flavor it is, the cloud provider, image id, ssh key and name to use, etc. These files are typically located in ``/etc/salt/cloud.profiles.d``:

    $:/etc/salt/cloud.profiles.d$ ls
    hpcs.conf        devstack_cirros.conf  rackspace_quantal.conf devstack_ubuntu.conf
    hpcs_raring.conf rackspace_raring.conf rackspace_precise.conf testwiki.conf

And example of one of these files would be:

    hpcs_raring:
        provider: hpcs_ae1
        image: 9302692b-b787-4b52-a3a6-daebb79cb498
        ignore_cidr: 10.0.0.1/24
        size: standard.small
        ssh_key_file: /root/keys/test.key
        ssh_key_name: test
        ssh_username: ubuntu

In this exemple, the provider used is for HP Cloud AE1 region, small flavor, Ubuntu raring image id, and for SSH a private key location is provided that will be used on the minion being set up. 

After reading through the code, I did find 


## Conclusion

This cluster setup is an example of the great things one can do with [Ansible][Ansible] and [Docker][Docker] and I intend on improving the playbook and setup. Some of which include:

- Cleanup!
- Better setup with how to have an SSH key that can be used in the repo
- Even better algorithm with boostrapping
- Find ansible and docker gurus to help give me critiques to the work 

The one thing I took away from building this was how much faster it is to deploy containers than virtual machine instances. This has profound implications for software packaging and testing.



[openstack]: http://openstack.org
[salt-docs]: http://docs.saltstack.com/en/latest/contents.html
[salt-cloud-docs]: http://docs.saltstack.com/en/latest/topics/cloud/index.html
[salt-openstack-contrib]: https://github.com/saltstack/salt/blob/develop/salt/cloud/clouds/openstack.py#L462
