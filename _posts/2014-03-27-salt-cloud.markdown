---
layout: post
title: "Using Salt Cloud"
date: 2014-03-27 09:00:00
categories: provisioning 
---

# Using [Salt][saltstack] Cloud

Recently, I took an interest in trying out [Salt Cloud][salt-cloud-docs], the provisioning tool component of [SaltStack][saltstack] or what will be refered to as "Salt" in this post, the provisioning to that offers "fast, scalable and flexible software for data center automation, from infrastructure and any cloud, to the entire application stack". 

I was attempting at first to get it to run with [HP Cloud][hpcloud] as well as a simple cloud setup with Devstack and found that I had to actually read the source code to really know how to get it to work. That was when I decided to contribute to the project both with documentation and a small fix to the [OpenStack][openstack] salt [driver][salt-openstack-driver], pep8/pyflakes cleanups and documentation.

## [Salt][saltstack] and Salt-cloud 

There is a lot of [information][salt-docs] about [Salt][saltstack] itself, but this post is focused on [Salt Cloud][salt-cloud-docs].

As just mentioned, [Salt][saltstack] is provisioning software, just as Chef, Puppet and Ansible are. [Salt][saltstack] works with a [Salt][saltstack] master which then runs states on minions (clients). These minions run the commands that the state, when run, results in the system being as expected according to the state. The master is cognizant of its minions and stores data used in the provisioning process. 

Salt is written in Python and uses YAML templates for both its "states" as well as its "pillars". States are YAML files that are a representation of a state in which a system should be in. Pillars are tree-like structures defined on the [Salt][saltstack] master and passed to the minions. Pillar data is represented in YAML files as well. 

In addition to states, it's possible to run single commands using [Salt][saltstack] by way of the [Salt][saltstack] master and on all or specific minions.

[Salt Cloud][salt-cloud-docs], as already mentioned, is the provisioning tool component of Salt. Often, you will see in cloud-based organizations that different teams might use a tool such as Chef or Salt, though they will often script the functionality that orachestrates the launching and/or deletion of instances. For instance, I worked on one team that used Chef and we had a very nice tool written in Ruby that would launch instances that would load cookbooks for and Chef would then provision. This approach can give one a lot of control but also be hard to maintain as well as being very application or cloud-specific. 

Salt-cloud has the approach of keeping the provisioning, instance creation and deletion aspect of salt under the same system and by doing so, the same concepts that [Salt][saltstack] uses, namely YAML being the means to express states and pillars naturally is also used for [Salt Cloud][salt-cloud-docs].

When [Salt Cloud][salt-cloud-docs] is used to launch an instance, that instance is not only brought up, but set up as a minion being managed by the master that launched it.

The other good think about [Salt Cloud][salt-cloud-docs] is that it is built on top of Apache LibCloud which makes it possible to use [Salt Cloud][salt-cloud-docs] with any number of cloud providers such as EC2, Google Compute Engine, LXC, Linode, Azure and of course [OpenStack][openstack] providers such as [HP Cloud][hpcloud] and [Rackspace][rackspace].


## [Salt Cloud][salt-cloud-docs] concepts

Salt Cloud uses the following concepts that I discovered upon using it:

- Cloud Provider: a particular cloud connection including the username, password, any sort of API key, region as well as authentication endpoints.
- Cloud profile: a particular type of image to use when launching an instance or container. This being, the size/flavor, image ID and the cloud provider to use
<br>
<br>

## [Salt Cloud][salt-cloud-docs] with [HP Cloud][hpcloud]

One of the first things I wanted to try to get working properly was the ability to launch instances with HP Cloud. This proved difficult at first and the documentation was dated for the initial version of [HP Cloud][hpcloud] (1.0, based off of Essex and using Nova Networking) whereby when launching instances, external IP addresses were automatically assigned to an instance, whereas with the newer [HP Cloud][hpcloud] (1.1, based off of Havana and using Neutron), one has to first create a floating IP and then add it to a specific instance.

Why is this a problem? Well, because when the command salt-cloud is run, it needs to not only be able to launch an instance but also connect to it through SSH and install the minion to be able to talk to the master. Without an IP to connect to, this won't happen, and the examples that were in the documentation did not work with [HP Cloud][hpcloud] 1.1. 

Also do note, this post is to introduce the reader to concepts. For more in-depth reading on the matter, it is strongly advised to review the [Salt Cloud docs][salt-cloud-docs], which the author recently contributed to for easier [Salt Cloud][salt-cloud-docs] usage with [OpenStack][openstack] and [HP Cloud][hpcloud] in particular!

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

In the above example, their are various connection metadata that is used to establish a connection to HP Cloud. This would be the same for any cloud provider and would vary according to the particular attributes of how one connects to a cloud. For instance, with [Rackspace][rackspace], one would also provide an ``apikey`` parameter. For ec2, ``provider`` would be ``ec2`` (obviously). 

The ``networks`` setting provides one or more named networks that you provide. You will need to know these in advance if you intend to use them. In the example above, they are Ext-Net (default network with HP Cloud) and a network the user created ``my-network``. These are networks that the salt-cloud [OpenStack][openstack] driver uses to obtain a list of floating IP addresses, picking one to attach to the instance being provisioned in order to be able to set up the instance as a minion. 

Interestingly, this is where I recently [modified][salt-openstack-contrib] the driver and contributed to salt-cloud code that would negate the need for specifying ``networks`` and instead simply obtain floating IPs if not specified regardless of network.  

The ``ignore_cidr`` setting is for listing a range of IP addresses to not even bother using as an address to connect to for setting up the minion. In this case, even though salt-cloud will not attempt to use the private IP address for the instance, using ``ignore_cidr`` will making it so salt-cloud doesn't bother even identifying that the IP address is private.

NOTE: If one wants to use the private IP address to connect to the instance-- for instance, if the master is in the same network in the cloud as the minions (actually commonplace) then one need only have the following in their cloud provider configuration file:

    ssh_interface: private_ips

This will result in salt-cloud using the private IP address when using SSH to connect to the instance during setup of the minion.

So, it is clearly a simple setup and clean YAML representation per the philosophy of Salt.

## Testing the cloud profile

First of all, it is assumed that the reader of this post has already set up a python virtual environment. Other installation steps and issues include:

- Installing [Salt][saltstack] within the vurtual environment
- One ought to set up the master (see [Salt Documentation][salt-docs]) 
- You must run salt-cloud as root. This is a bit of a conundrum because the documentation clearly states that you ought to run a virtualenv setup for development yet you need to run it as root. One way to do this is to set up the virtualenv as suggested as a regular user and run salt-cloud via sudo after activiting the virtual environment. The other way is to set up a virtualenv as root and install salt as: 


    pip install -e /home/regularuser/salt 


### Listing cloud providers

The first command to run to verify you have set up cloud provider configuration files properly is to list the cloud providers:

    # salt-cloud --list-providers
    [INFO    ] salt-cloud starting
    hpcs_ae1:
        ----------
        openstack:
            ----------
    hpcs_aw2:
        ----------
        openstack:
            ----------
    hpcs_az1_aw2_patg:
        ----------
        openstack:
            ----------
    openstack_grizzly:
        ----------
        openstack:
            ----------
    rackspace_ord:
        ----------
        rackspace:
            ----------
    rackspace_dfw:
        ----------
        rackspace:
            ----------

This output shows a variety of providers between [HP Cloud][hpcloud] and [Rackspace][rackspace], and several regions with each. At this point listing images is the next step.

### Listing images within a provider

The next verification to be done would be to test one of the providers. A simple test would be to list images for a specific provider that can be used to launch an instance that is defined in a cloud profile (next topic).

The command is run by specifying the cloud provider, in this case, ``hpcs_ae1``:

    # salt-cloud --list-images hpcs_ae1
    [INFO    ] salt-cloud starting
    hpcs_ae1:
        ----------
        openstack:
            ----------
           <snip> ... numerous images ... 
           Ubuntu Raring 13.04 Server 64-bit 20130601:
                ----------
                driver:
                extra:
                    ----------
                    created:
                        2013-07-03T13:40:56Z
                    metadata:
                        ----------
                        architecture:
                            x86_64
                        com.hp__1__bootable_volume:
                            true
                        com.hp__1__image_lifecycle:
                            active
                        com.hp__1__image_type:
                            disk
                        com.hp__1__os_distro:
                            com.ubuntu
                        os_type:
                            linux-ext4
                        os_version:
                            13.04
                    minDisk:
                        10
                    minRam:
                        0
                    progress:
                        100
                    serverId:
                        None
                    status:
                        ACTIVE
                    updated:
                        2014-03-21T11:42:56Z
                get_uuid:
                id:
                    9302692b-b787-4b52-a3a6-daebb79cb498
                name:
                        Ubuntu Raring 13.04 Server 64-bit 20130601
                uuid:
                    4e6b04dc25dad797e345395c445182fa2c697050
    
As you can see, with only one image there is a lot of information and the above was from an output of numerous images. If you have an output, then obviously the cloud provider configuration file is correct.
        
    
## Setting up a cloud profile
    
A cloud profile is yet another YAML file that represents how a particular instance is launched: what size/flavor it is, the cloud provider, image id, ssh key and name to use, etc. These files are typically located in ``/etc/salt/cloud.profiles.d``:
    
    $:/etc/salt/cloud.profiles.d$ ls
    hpcs.conf        devstack_cirros.conf  rackspace_quantal.conf devstack_ubuntu.conf
    hpcs_raring.conf rackspace_raring.conf rackspace_precise.conf testwiki.conf
    
And example of one of these files would be:
    
        hpcs_raring:
            provider: hpcs_ae1
            image: 9302692b-b787-4b52-a3a6-daebb79cb498
            size: standard.small
            ssh_key_file: /root/keys/test.key
            ssh_key_name: test
            ssh_username: ubuntu
    
In this exemple, the provider used is for [HP Cloud][hpcloud] AE1 region, small flavor, Ubuntu raring image id, and for SSH a private key location is provided that will be used on the minion being set up. 
    
The settings in this file are as follows:
    
- ``provider`` - this is the cloud provider that will be used to connect and run the instance in
- ``size`` - this is the size, or in the case of OpenStack, the flavor to be used
- ``ssh_username`` - this is the username that will be used to log into the instance to set up the minion. Since this is an Ubuntu Raring image, ``ubuntu`` is the user. With other images this will be different.
- There are also some setttings in the above such as ``ssh_key_name``, and ``ssh_key_file`` that override whatever the clodu provider this uses 
    
Now that this cloud profile is defined, it can be used.
    
### Launching a cloud profile
    
To launch a cloud profile is quite simple:
    
    salt-cloud -p hpcs_ubuntu_raring myinstance
    [INFO    ] salt-cloud starting
    [INFO    ] Creating Cloud VM myinstance
    [INFO    ] Attaching floating IP '15.126.223.157' to node 'myinstance'
    [WARNING ] Private IPs returned, but not public... Checking for misidentified IPs
    [WARNING ] 10.0.0.27 is a private IP
    [WARNING ] IP '10.0.0.27' found within '10.0.0.1/24'; ignoring it.
    [INFO    ] Rendering deploy script: /home/patg/code/salt/salt/cloud/deploy/bootstrap-salt.sh
    Warning: Permanently added '15.126.223.157' (ECDSA) to the list of known hosts.
    <snip> ... similar output
    Warning: Permanently added '15.126.223.157' (ECDSA) to the list of known hosts.
    sudo: unable to resolve host myinstance
     *  INFO: /bin/sh /tmp/.saltcloud/deploy.sh -- Version 2014.02.27
    
     *  INFO: System Information:
     *  INFO:   CPU:          GenuineIntel
     *  INFO:   CPU Arch:     x86_64
     *  INFO:   OS Name:      Linux
     *  INFO:   OS Version:   3.8.0-23-generic
     *  INFO:   Distribution: Ubuntu 13.04
    
     *  INFO: Installing minion
     *  INFO: Found function install_ubuntu_deps
     *  INFO: Found function config_salt
     *  INFO: Found function install_ubuntu_stable
     *  INFO: Found function install_ubuntu_restart_daemons
    *  INFO: Found function daemons_running
     *  INFO: Found function install_ubuntu_check_services
     *  INFO: Running install_ubuntu_deps()
    Hit http://az2.clouds.archive.ubuntu.com raring Release.gpg
    Get:1 http://security.ubuntu.com raring-security Release.gpg [933 B]
    Get:2 http://az2.clouds.archive.ubuntu.com raring-updates Release.gpg [933 B]
    Get:3 http://security.ubuntu.com raring-security Release [40.8 kB]
    <snip> ... more output
    ssing triggers for ureadahead ...
    Setting up salt-minion (0.17.5-1raring1) ...
    
    Configuration file `/etc/salt/minion'
     ==> File on system created by you or by a script.
     ==> File also in package provided by package maintainer.
     ==> Using current old file as you requested.
    salt-minion start/running, process 3760
    Setting up debconf-utils (1.5.49ubuntu1) ...
    Processing triggers for ureadahead ...
     *  INFO: Running install_ubuntu_check_services()
     *  INFO: Running install_ubuntu_restart_daemons()
    salt-minion stop/waiting
    salt-minion start/running, process 3791
    [INFO    ] [Salt][saltstack] installed on myinstance
    [INFO    ] Created Cloud VM 'myinstance'
    myinstance:
        ----------
        _uuid:
            None
        driver:
        extra:
            ----------
            access_ip:
    
            created:
                2014-03-27T15:15:54Z
            disk_config:
                None
            flavorId:
                101
            hostId:
    
            imageId:
                9302692b-b787-4b52-a3a6-daebb79cb498
            key_name:
                test
            metadata:
                ----------
                profile:
                    hpcs_1_1_ubuntu_raring
            tenantId:
                10966558279755
            updated:
                2014-03-27T15:15:55Z
            uri:
                http://region-b.geo-1.compute.hpcloudsvc.com/v2/10966558279755/servers/d461e8e7-aae1-47be-b313-21c9c302565f
        id:
            d461e8e7-aae1-47be-b313-21c9c302565f
        image:
            None
        name:
            myinstance
        private_ips:
        public_ips:
            - 15.126.223.157
        size:
            None
        state:
            3
    
At this point, you now have a newly launched minion and can start provisioning it via salt!

### Delete instance

If you wanted to delete the instance, you would simply run:

    # salt-cloud -d myinstance
<br>
<br>

## Conclusion

This post was to familiarize the reader with salt-cloud and give a simple demonstration on how salt-cloud can be used and how clean and straight-forward the configuration files are, adhering to the overall vision of [Salt][saltstack] itself. There are far many more options to using salt-cloud that you can view simply by running the command with the --help option as well as read both the [Salt documentation] in general as well as [Salt Cloud documentation][salt-cloud-docs]. This blog will also continue to feature posts pertaining to salt and provisioning tools in general. 


[openstack]: http://openstack.org
[saltstack]: http://www.saltstack.com/
[salt-docs]: http://docs.saltstack.com/en/latest/contents.html
[salt-cloud-docs]: http://docs.saltstack.com/en/latest/topics/cloud/index.html
[salt-openstack-driver]: https://github.com/saltstack/salt/blob/develop/salt/cloud/clouds/openstack.py
[salt-openstack-contrib]: https://github.com/saltstack/salt/blob/develop/salt/cloud/clouds/openstack.py#L462
[hpcloud]: http://hpcloud.com
[rackspace]: http://www.rackspace.com/
