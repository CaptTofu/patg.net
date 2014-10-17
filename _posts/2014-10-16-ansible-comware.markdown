---
layout: post
title: "Managing Comware 5.2-based switches with Ansible"
date: 2014-10-16 12:00:00
categories: ansible,comware,switches
---

Since I last posted, I have been busy trying to learn both about [Comware][comware]-based switches as well as how to create an [Ansible][Ansible] module that can manage such a switch - the HP V1910 24G in particular.  I hadn't used this switch before and had to do a bit of research as to how it works, as well as refreshing some networking basics particular to switches.

## About the HP V1910 24G Switch

This switch is an advanced, smart managed, fixed-configuration Gigabit switch. Its target customer would be a small business. It's a 24-port switch that can be managed using a web UI or command line interface. It is [Comware][comware] 5.2-based and for me to have all the features I needed with [Comware][comware], I had to run a special command to have the switch go into what is called [developer mode][1910_developer_mode] -- something that the Ansible modules I wrote do automatically for the user. The [developer mode][1910_developer_mode] provides

I was hoping originally that there would be a REST API, which many switch vendors provide and some of the switch modules in Ansible that currently exist take advantage of either in writing a module or creating a connection plugin. With the 1910, I was unable to find one, so in order to write an [Ansible][Ansible] module, I had to interact with the switch using SSH, specifically through the [Paramiko][paramiko] library.

At first, one would think "ah, SSH, just run Ansible against it". However, Ansible by default assumes a POSIX-based system (UNIX host) and even simply running a simple ansible ping against the device will result in Ansible attempting to create a temporary directory and run several posix commands.  Since the switch is not POSIX-based, this obviously fails.

## About the implementation

First of all, it should be noted that a lot of work was proof-of-concept, trying to discover how feasible this would be. My honest opinion is that it is feasible, once you understand how to interact with the switch and work with [Paramiko][paramiko] and switch output.

What I ended up writing was an [Ansible][Ansible] module that sends the appropriate [Comware][comware] commands to do specific tasks. This was difficult in that I had to deal with responses from the switch that don't behave the way a typical UNIX host would. For instance, in some actions one performs on the switch will result in a reply that expects a yes or no ('y' or 'n') that are read by the switch using getchar() (and without a carriage return), hence I ended up using [Paramiko][paramiko_api] channel object methods ```recv()``` and ```send()``` as opposed to ```exec_command()``` (I will digress on the specifics of this in another post). Also difficult but surmountable was how to read and parse the output from the switch and know that I had read everything I needed to. At first, I would run some commands and my program would wait forever -- because it was still trying to ```recv()``` !

Lastly, there was a huge amount of work in parsing the output from the summary, running configuration and other commands into a usable "facts" dictionary that is used all througout each of the modules and by [Ansible][Ansible] itself.

I was inspired by some great snippets in the [paramiko_expect][paramiko_expect] library. I would have used this library, but wanted to keep my library requirements at a minimum; plus I only needed a subset of features.

Also changed is the [Ansible][Ansible] module arrangement. The Ansible progect has split out modules into [core][ansible_modules_core] and [extras][ansible_modules_extras] git repositories, as well as renaming them with ".py" extensions (a good thing!). This added one requirement for me that my modules which have common methods needed a library. Originally, one would put module libraries in [module-utils][module-utils] directory, but with this change, it made more sense for me to create my own python library, [comware_5_2][comware_5_2], a PyPI module that I would eventually like to make less specific to Ansible and more specific to writing Python code to talk to [Comware][comware]-based switches.

## Using the modules

It's very easy to use. You of course need a switch, and currently my code is in a pull request at my branch on [github][capttofu_ansible_modules_extras]. Also, you will need to install my [comware_5_2][comware_5_2] Python module.

There are 4 [Ansible][Ansible] comware_5_2 modules:

- comware_5_2: This module simply gathers facts and allows one to reboot the switch if desired. It has a state of "present" and "reboot", the default being "present". A reboot is a bit tricky in that once you issue it, you have to wait for the switch to resume operation.
- comware_5_2_vlan: This module allows for the creation, modification, and deletion of vlans. One sets the name of the vlan, which ports should be tagged as well as untagged and what link-type to use for each category of port.
- comware_5_2_port: This module allows one to modify ports, meaning, assign them to vlans and set other characteristics such as link-type. There is so much to be done with ports, so again, this is a proof of concept and more attribute settings can easily be added to the module to support richer syntax and functionality.
- comware_5_2_user: This module allows one to create, modify, and delete users. For instance, one can define a new user with a given password and privilege level as well as which access methods the user can access the switch using -- "web", "ssh", "terminal" or "telnet".

<br />
## Using the "comware_5_2" module

A Sample playbook: (there are several samples at [comware_5_2_playbooks][comware_5_2_playbooks] using the base module, which simply gathers facts if you run ansible-playbook in -vvv displays a dictionary with the switch facts.)

First, there is an example inventory file with these playbooks:

{% raw %}
```
[switch1]
localhost

[switch1:vars]
switch_host=192.168.x.x
switch_user=admin
switch_password=redacted
switch_key_file=~/.ssh/id_dsa
```
{% endraw %}


Secondly, a simple task using the "comware_5_2" module:

{% raw %}
```
- hosts: switch1
  tasks:
  - name: gather facts from switch
    comware_5_2:
      gather_facts=true
      host={{ switch_host }}
      username={{ switch_user }}
      password={{ switch_password }}
```
{% endraw %}

If ansible is run with the ```-vvvv```` option (NOTE: the output has been formatted for readability):

{% raw %}
```
ok: [localhost] => {"ansible_facts": {
    "current_config":
        {"domain default": "enable system", "ftp server": "enable",
         "interfaces": {"GigabitEthernet1/0/1": {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/10":
                           {"stp edged-port": "enable",
                            "vlan":
                                {"tagged": {},
                                 "untagged": {"access": ["44"]}}},
                        "GigabitEthernet1/0/11":
                          {"port link-type": "trunk",
                           "stp edged-port": "enable",
                           "vlan": {"tagged": {"trunk": ["1", "22"]}, "untagged": {}}},
                        "GigabitEthernet1/0/12":
                          {"port link-type": "trunk",
                           "stp edged-port": "enable",
                           "vlan": {"tagged": {"trunk": ["1", "22"]}, "untagged": {}}},
                        "GigabitEthernet1/0/13":
                          {"port link-type": "hybrid",
                           "stp edged-port": "enable",
                           "vlan": {"tagged": {}, "untagged": {"hybrid": ["1", "22"]}}},
                        "GigabitEthernet1/0/14":
                          {"port link-type": "hybrid",
                           "stp edged-port": "enable",
                           "vlan": {"tagged": {}, "untagged": {"hybrid": ["1", "22"]}}},
                        "GigabitEthernet1/0/15":
                          {"port link-type": "hybrid",
                           "stp edged-port": "enable",
                           "vlan":
                            {"tagged": {},
                             "untagged": {"hybrid": ["1", "55", "66", "77"]}}},
                        "GigabitEthernet1/0/16":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/17":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/18":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/19":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/2":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/20":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/21":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/22":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/23":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/24":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/25":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/26":
                          {"port link-type": "trunk",
                           "stp edged-port": "enable",
                           "vlan":
                             {"tagged": {"trunk": ["1", "44"]}, "untagged": {}}},
                        "GigabitEthernet1/0/27":
                          {"port link-type": "hybrid",
                           "stp edged-port": "enable",
                           "vlan":
                             {"tagged": {}, "untagged": {"hybrid": ["1"]}}},
                        "GigabitEthernet1/0/28":
                          {"stp edged-port": "enable",
                           "vlan": {"tagged": {}, "untagged": {"access": ["44"]}}},
                        "GigabitEthernet1/0/3":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/4":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/5":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/6":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/7":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/8":
                          {"stp edged-port": "enable"},
                        "GigabitEthernet1/0/9":
                          {"stp edged-port": "enable",
                           "vlan":
                             {"tagged": {}, "untagged": {"access": ["44"]}}}, "NULL0": {},
                        "Vlan-interface1": {}},
         "ip ttl-expires": "enable",
         "local-user":
           { "admin":
               {"authorization-attribute": "level 2",
                "password": "cipher $c$3$ieXiyTSM+tfFD+y5eUiQVq2EamKJl27j7R0=",
                "service-type": ["ssh", "telnet", "terminal", "web"]},
             "admin2":
               {"authorization-attribute": "level 2",
                "password": "cipher $c$3$ieXiyTSM+tfFD+y5eUiQVq2EamKJl27j7R0=",
                "service-type": ["ssh", "telnet", "terminal", "web"]},
             "admin3":
               {"authorization-attribute": "level 2",
                "password": "cipher $c$3$ieXiyTSM+tfFD+y5eUiQVq2EamKJl27j7R0=",
                "service-type": ["ssh", "telnet", "terminal", "web"]}},
         "password-recovery": "enable",
         "sysname": "HP",
         "user-group": "system",
         "vlans":
           {"1": {}, "22": {"name": "VLAN_22"},
            "44": {"name": "VLAN_44"},
            "55": {},
            "66": {},
            "77": {}}},
  "summary": {
    "Current_boot_app_is": "flash:/V1910-CMW520-R1513P62.BIN",
    "Default_gateway": "192.168.1.1",
    "IP_Method": "DHCP",
    "IP_address": "192.168.1.14",
    "Next_backup_boot_app_is": "NULL",
    "Next_main_boot_app_is": "flash:/v1910-cmw520-r1513p62.bin",
    "Select_menu_option": "Summary",
    "Subnet_mask": "255.255.255.0",
    "bootrom_version": "Bootrom Version is 163",
    "cpld_version": "CPLD Version is 002",
    "hardware_version": "Hardware Version is REV.B",
    "memory_dram": "128M    bytes DRAM",
    "memory_flash": "128M    bytes Nand Flash Memory",
    "memory_register": "Config Register points to Nand Flash",
    "model": "",
    "software_copyright": "Copyright (c) 2010-2013 Hewlett-Packard Development Company, L.P.",
    "software_version": "Comware Software, Version 5.20, Release 1513P62",
    "subslot_0": "[SubSlot 0] 24GE+4SFP Hardware Version is REV.B",
    "uptime": "HP V1910-24G Switch uptime is 2 weeks, 0 day, 22 hours, 2 minutes"},
  "vlans":
    {"1":
      {"Description": "VLAN 0001",
       "IP_Address": "192.168.1.14",
       "Name": "VLAN 0001",
       "Route_Interface": "configured",
       "Subnet_Mask": "255.255.255.0",
       "Tagged_Ports": "none",
       "Untagged_Ports":
         [ "GigabitEthernet1/0/1",
           "GigabitEthernet1/0/2",
           "GigabitEthernet1/0/3",
           "GigabitEthernet1/0/4",
           "GigabitEthernet1/0/5",
           "GigabitEthernet1/0/6",
           "GigabitEthernet1/0/7",
           "GigabitEthernet1/0/8",
           "GigabitEthernet1/0/11",
           "GigabitEthernet1/0/12",
           "GigabitEthernet1/0/13",
           "GigabitEthernet1/0/14",
           "GigabitEthernet1/0/15",
           "GigabitEthernet1/0/16",
           "GigabitEthernet1/0/17",
           "GigabitEthernet1/0/18",
           "GigabitEthernet1/0/19",
           "GigabitEthernet1/0/20",
           "GigabitEthernet1/0/21",
           "GigabitEthernet1/0/22",
           "GigabitEthernet1/0/23",
           "GigabitEthernet1/0/24",
           "GigabitEthernet1/0/25",
           "GigabitEthernet1/0/26",
           "GigabitEthernet1/0/27"],
           "VLAN_Type": "static"},
       "22":
         {"Description": "VLAN 0022",
          "Name": "VLAN_22",
          "Route_Interface": "not configured",
          "Tagged_Ports":
            ["GigabitEthernet1/0/11", "GigabitEthernet1/0/12"],
          "Untagged_Ports":
            ["GigabitEthernet1/0/13", "GigabitEthernet1/0/14"],
          "VLAN_ID": "22",
          "VLAN_Type": "static"},
        "44":
          {"Description": "VLAN 0044",
           "Name": "VLAN_44",
           "Route_Interface": "not configured",
           "Tagged_Ports":
             ["GigabitEthernet1/0/26", "GigabitEthernet1/0/27"],
           "Untagged_Ports":
             ["GigabitEthernet1/0/9", "GigabitEthernet1/0/10", "GigabitEthernet1/0/28"],
           "VLAN_ID": "44",
           "VLAN_Type": "static"},
         "55":
           {"Description": "VLAN 0055",
            "Name": "vlan 55",
            "Route_Interface": "not configured",
            "Tagged_Ports": "none",
            "Untagged_Ports": ["GigabitEthernet1/0/15"],
            "VLAN_ID": "55",
            "VLAN_Type": "static"},
          "66":
            {"Description": "VLAN 0066",
             "Name": "vlan 66",
             "Route_Interface": "not configured",
             "Tagged_Ports": "none",
             "Untagged_Ports": ["GigabitEthernet1/0/15"],
             "VLAN_ID": "66",
             "VLAN_Type": "static"},
          "77":
            {"Description": "VLAN 0077",
             "Name": "vlan 77",
             "Route_Interface": "not configured",
             "Tagged_Ports": "none",
             "Untagged_Ports": ["GigabitEthernet1/0/15"],
             "VLAN_ID": "77",
             "VLAN_Type": "static"}
           }
        },
"changed": false, "failed": false, "msg": ""}

ok: [localhost] => {"ansible_facts": < SNIP - the same as above ! />

PLAY RECAP ********************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0
```
{% endraw %}

<br />

## Using an ssh key

It should also be noted that one can use an ssh key to connect to the switch. The setup of ssh key authentication is [involved][comware_ssh_priv_key_setup] and currently is assumed to be in place should you use it. Its my intention to also create an ansible module to do this setup as well!

To use a key, I could have simply used the ```private_key_file``` option in the playbook, specifying my private key on the host I am running this locally from. This idea I obtained from the Paramiko connection plugin for [Ansible][Ansible]:

{% raw %}
```
    comware_5_2: gather_facts=true host={{ switch_host }} username={{ switch_user }} private_key_file=~/.ssh/id_dsa
```
{% endraw %}

<br />

## Using the "comware_5_2_vlan" [Ansible][Ansible] module

Since the previous example included so much output, this example will only include the actual playbook and non-verbose output.

The example below is quite self-evident. It creates a vlan, arbitrarily named "VLAN_44", modifies it, and then deletes it.

The creation initially sets two ports g1/0/9 and g1/0/10 to be untagged and of type "access". The modification adds another untagged port, g1/0/28 and one tagged port, g1/0/27, set to link-type "hybrid". Note: in order for me to add reliable modification functionality, the easiest thing to do was to completely delete and recreate the VLAN. This may be changed in the future. Finally, VLAN_44 is deleted, simply by providing a vlan_id value of 44.

{% raw %}
```
- hosts: switch1
  tasks:
  - name: create VLAN_44
    comware_5_2_vlan:
      host={{ switch_host }}
      username={{ switch_user }}
      password={{ switch_password }}
      state=present
      vlan_id=44
      vlan_name=VLAN_44
      untagged_port_type=access
      untagged_ports=GigabitEthernet1/0/9,GigabitEthernet1/0/10

  - name: modify VLAN_44
    comware_5_2_vlan:
      host={{ switch_host }}
      username={{ switch_user }}
      password={{ switch_password }}
      state=present
      vlan_id=44
      vlan_name=VLAN_44
      untagged_port_type=access
      untagged_ports=GigabitEthernet1/0/9,GigabitEthernet1/0/10,GigabitEthernet1/0/28
      tagged_port_type=hybrid
      tagged_ports=GigabitEthernet1/0/27

     - name: delete VLAN_44
    comware_5_2_vlan:
      host={{ switch_host }}
      username={{ switch_user }}
      password={{ switch_password }}
      state=absent
      vlan_id=44
      vlan_name=VLAN_44

```
{% endraw %}

The running of this playbook:

{% raw %}
```
~/code/comware_5_2_playbooks$ ansible-playbook -i inventory switch_vlan_sing
le.yml

PLAY [switch1] ****************************************************************

GATHERING FACTS ***************************************************************
ok: [localhost]

TASK: [create VLAN_44] ********************************************************
changed: [localhost]

TASK: [modify VLAN_44] ********************************************************
changed: [localhost]

TASK: [delete VLAN_44] ********************************************************
changed: [localhost]

PLAY RECAP ********************************************************************
localhost                  : ok=5    changed=3    unreachable=0    failed=0
```
{% endraw %}

<br />

## Using the "comware_5_2_port" module

After implementing the "comware_5_2_vlan" [Ansible][Ansible] module, I realized it would useful and simple to add a [Ansible][Ansible] module that lets you modify ports specifically, hence the "comware_5_2_port" [Ansible][Ansible] module was created.

The simple example below simply sets up the port g1/0/15 to have a link-type of "hybrid" for VLANs 55, 66, and 77:

{% raw %}
```
(python-dev)patg@ubuntu:~/code/comware_5_2_playbooks$ cat port.yml
# file: port.yml
- hosts: switch1
  tasks:
  - name: Set port g1/0/15
    comware_5_2_port:
      host={{ switch_host }}
      username={{ switch_user }}
      password={{ switch_password }}
      name=GigabitEthernet1/0/15
      vlans=55,66,77
      link_type=hybrid
```
{% endraw %}

Running it results in:

{% raw %}
```
~/code/comware_5_2_playbooks$ ansible-playbook -i inventory port.yml

PLAY [switch1] ****************************************************************

GATHERING FACTS ***************************************************************
ok: [localhost]

TASK: [Set port g1/0/15] ******************************************************
changed: [localhost]

PLAY RECAP ********************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0
```
{% endraw %}


<br />

## Using the "comware_5_2_user" [Ansible][Ansible] module

Lastly, if one needs to manage users on a [Comware][comware] switch, the "comware_5_2_user" Ansible module makes this possible.

The example below creates a user called "jimbob", with a password, authorization level of 2, and gives access to this user through web, ssh, and terminal.

{% raw %}
```
- hosts: switch1
  tasks:
  - name: create jimbob user
    comware_5_2_user:
      host={{ switch_host }}
      username={{ switch_user }}
      password={{ switch_password }}
      state=present
      user_name=jimbob
      user_pass=seekrit
      auth_level="level 2"
      services=web,ssh,terminal
```
{% endraw %}

Running the playbook:

{% raw %}
```
~/code/comware_5_2_playbooks$ ansible-playbook -i inventory user_create.yml

PLAY [switch1] ****************************************************************

GATHERING FACTS ***************************************************************
ok: [localhost]

TASK: [create jimbob user] ****************************************************
changed: [localhost]

PLAY RECAP ********************************************************************
localhost                  : ok=2    changed=1    unreachable=0    failed=0
```
{% endraw %}

If it is needed to delete the user, only changing "state" to "absent" is required.

<br />
## Summary

If you should find yourself with [Comware][comware] 5.2-based switches that you want to manage using [Ansible][Ansible], do feel free to give my [code][capttofu_ansible_modules_extras] a spin.

Also, I'm very open to input on this. I approached this having never worked a lot with [Comware][comware], so any tips and suggestions you may have, or even criticisms, I am very open to!


[Docker]: http://docker.io
[Ansible]: http://www.ansible.com/home
[ansible_dynamic_inventory]: http://docs.ansible.com/intro_dynamic_inventory.html#dynamic-inventory
[ansible_example_repository]: https://github.com/ansible/ansible-examples
[ansible_documentation]: http://docs.ansible.com/
[ansible_apt_module]: http://docs.ansible.com/apt_module.html
[ansible_docker_module]: http://docs.ansible.com/docker_module.html
[ansible_docker_image_module]: http://docs.ansible.com/docker_image_module.html
[ansible_docker_facts_module]: https://github.com/CaptTofu/ansible/tree/docker_facts
[ansible_docker_dynamic_inventory]: https://github.com/ansible/ansible/blob/devel/plugins/inventory/docker.py
[docker_py]: https://github.com/dotcloud/docker-py
[docker_intro_blog]: http://patg.net/containers,virtualization,docker/2014/06/05/docker-intro/
[docker_install_blog]: http://patg.net/containers,virtualization,docker/2014/06/09/docker-install/
[docker_using_blog]: http://patg.net/containers,virtualization,docker/2014/06/10/using-docker/
[ansible_playbooks]: http://docs.ansible.com/playbooks.html
[ansible_example_playbook_repository]: https://github.com/ansible/ansible-examples
[docker_cli]: https://docs.docker.com/reference/commandline/cli/
[AnsibleFest]: http://www.marketwired.com/press-release/speakers-from-twitter-google-twilio-edx-rackspace-headline-ansiblefest-nyc-2014-1902858.htm
[ansible_docker_presentation]: http://www.slideshare.net/PatrickGalbraith/docker-ansible-34909080
[ansible_docker_presentation_repo]: https://github.com/CaptTofu/ansible-docker-presentation
[moonshot]: http://h17007.www1.hp.com/us/en/enterprise/servers/products/moonshot/index.aspx#.U0gU2PldXJh?jumpid=ps_r163&k_clickid=AMS|112|61913|169782eb-9c7a-df89-4afe-0000588f5dc0
[moonshot_cartridge]: http://www8.hp.com/us/en/products/proliant-servers/product-detail.html?oid=5375897#!tab=features
[docker_image_source]: https://github.com/CaptTofu/docker-image-source
[docker_ansible_blog]: http://patg.net/ansible,docker/2014/06/18/ansible-docker/
[docker_images_ansible_blog]: http://patg.net/ansible,docker/2014/06/20/ansible-docker-image/
[docker_images_ansible_blog2]: http://patg.net/ansible,docker,mysql,galera,clustering/2014/06/24/ansible-docker-image-part2/
[docker_image_hub]: https://registry.hub.docker.com/
[ansible_facts]: http://docs.ansible.com/playbooks_variables.html#information-discovered-from-systems-facts
[ansible_docker_facts_repo]: https://github.com/CaptTofu/ansible/tree/docker_facts
[ansible_repo]: https://github.com/ansible/ansible
[ansible_modules_core]: https://github.com/ansible/ansible-modules-core
[ansible_modules_extras]: https://github.com/ansible/ansible-modules-extras
[comware_5_2_playbooks]: https://github.com/CaptTofu/comware_5_2_playbooks
[comware_5_2]: http://code.patg.net/comware_5_2.tar.gz
[paramiko]: http://www.paramiko.org
[paramiko_api]: http://docs.paramiko.org/en/1.15/api/channel.html
[paramiko_expect]: https://github.com/fgimian/paramiko-expect
[capttofu_ansible_modules_extras]: https://github.com/CaptTofu/ansible-modules-extras/tree/features/hp_switch
[comware]: http://www.h3c.com/portal/Products___Solutions/Technology/Comware_Platform_Overview/200701/195494_57_0.htm
[1910_developer_mode]: http://h30499.www3.hp.com/t5/Web-and-Unmanaged/How-limited-is-the-1910-CLI/td-p/5966697#.VEEt10tZsnj
[comware_ssh_priv_key_setup]: http://h20566.www2.hp.com/portal/site/hpsc/template.PAGE/public/kb/docDisplay?docId=mmr_kc-0107843&ac.admitted=1413557742564.876444892.199480143
