---
layout: post
title: "switchssh: A simple python library for talking to Switches"
date: 2015-03-14 12:00:00 
categories: switches, python, comware, pro vision
---

## switchssh

Recently, I was working on developing Ansible modules for HP Switches-- [Pro Vision][provision] and [Comware 5.2][comware]. The first type of switch I implemented a driver for was an HP 1910 24G (Comware 5.2). More recently, I implemented a rudimentary Ansible module for an HP 3800 (Pro Vision). 

Each of these switch OSs had their own peculiarities and to have a means to talk to each required some technical gymnastics. For instance, specifically the HP 1910 24G, based on Comware, by default doesn't include the full CLI command-set unless one enters a "developer-mode" (this required digging around the HP networking forums), which requires entering confirmation and a code. There are other Comware-based switches that have the full command-set without this developer-mode.

The Pro Vision OS (on an HP 3800) has a full command set but trying to parse output in its default vt100 terminal settings was a parsing nightmare. Setting Pro Vision to "raw" terminal settings solved that issue. Then of course there is a banner message to dismiss upon logging in.

Both switch OSs required having to disable paging-- that being, when a page worth of data is displayed, a prompt of "more" (text varied according to OS) would require subsequent space-bar to to continue, an added complexity that I was glad to discover could be disabled.

While implementing basic modules for both OSs, I was cognizant that I had some duplication that I would need to consolidate and abstract out of both code-bases before I added even more functionality to either driver. That's where this simple yet useful python module I created-- switchssh -- came into being.

I pulled out any code that simply pertained to the communication layer of talking to each switch. There were some common things that pertained to both switches, but where it differed, I used polymorphism to handle those differences. The top level class is ```switchssh```, the subclasses ```pro_vision``` and ```comware_hp_1910```. 


## Usage

Using these modules is painfully simple, as this simple script below shows using the ```comware_hp_1910``` subclass:

```
#!/usr/bin/env python

from switchssh.comware_hp_1910 import ComWareHP1910
import os
import re

import pprint

pp = pprint.PrettyPrinter(indent=2)

# simple script that shows how to use the library with comware
def main():
    conn = ComWareHP1910('192.168.1.x',
                     'admin',
                     'redacted',
                     read_end="<HP>",
                     dismiss_banner=False)

    out = conn.exec_command("display current-configuration\n", read_end='<HP>')
    pp.pprint(out)
    prompt = conn._get_prompt()
    print "prompt: %s\n" % prompt


main()

```

The output would be (truncated):

```
[ '                 display current-configuration',
  '#',
  ' version 5.20, Release 1513P62',
  '#',
  ' sysname HP',
  '#',
  ' domain default enable system',
  '#',
  ' ip ttl-expires enable',
  '#',
  ' password-recovery enable',
  '#',
  'vlan 1',
  '#',
  'vlan 5',
  ' name vlan 5',
  '#',
  'domain system',
  ' access-limit disable',
  ' state active',
  ' idle-cut disable',
  ' self-service-url disable',
  '#',
 'interface Vlan-interface1',
  ' ip address dhcp-alloc',
  '#',
  'interface GigabitEthernet1/0/1',
  ' stp edged-port enable',
  '#',
  'interface GigabitEthernet1/0/2',
  ' stp edged-port enable',
  '#',
    'return']
prompt: <HP>
...

```

For Pro Vision:

```

#!/usr/bin/env python

from switchssh.pro_vision import ProVision
import os
import re

import pprint

pp = pprint.PrettyPrinter(indent=2)

# simple script to show how to use this library
def main():
    conn = ProVision('192.168.1.x',
                     'operator',
                     'redacted',
                     read_end="tty=none",
                     dismiss_banner=True)

    out = conn.exec_command("show run\n")
    pp.pprint(out)
    out = conn.exec_command("configure\n")
    pp.pprint(out)
    out = conn.exec_command("vlan 88\n")
    pp.pprint(out)
    out = conn.exec_command("name eighty-eight\n")
    pp.pprint(out)
    out = conn.exec_command("exit\n")
    pp.pprint(out)
    out = conn.exec_command("exit\n")
    pp.pprint(out)
    out = conn.exec_command("show run\n")
    pp.pprint(out)
    out = conn.exec_command("configure\n")
    pp.pprint(out)
    out = conn.exec_command("no vlan 88\n")
    pp.pprint(out)
    out = conn.exec_command("show run\n")
    pp.pprint(out)


main()
```

Part of the output:

```

[ ' show run',
  '',
  'Running configuration:',
  '',
  '; J9575A Configuration Editor; Created on release #KA.15.03.3015',
  '; Ver #01:00:01',
  '',
  'hostname "HP E3800-24G-2SFP+ Switch" ',
  'module 1 type J9575x ',
  'vlan 1 ',
  '   name "DEFAULT_VLAN" ',
  '   untagged 1,8-26 ',
  '   ip address 192.168.1.200 255.255.255.0 ',
  '   no untagged 2-7 ',
  '   exit ',
  'vlan 66 ',
  '   name "VLAN" ',
  '   ip address 192.168.10.1 255.255.255.0 ',
  '   ip address 192.168.11.2 255.255.255.0 ',
  '   tagged 5-8 ',
  '   exit ',
  'vlan 44 ',
  '   name "VLAN_44" ',
  '   untagged 2-7 ',
  '',
  '   ip address 192.168.3.4 255.255.255.0 ',
  '   ip address 192.168.5.6 255.255.255.0 ',
  '   tagged 20-24 ',
  '   exit ',
  'console terminal none',
  'snmp-server community "public" unrestricted',
  'snmp-server contact "Patrick Galbraith"',
  'oobm',
  '   ip address dhcp-bootp',
  '   exit',
  'no autorun',
  'no dhcp config-file-update',
  'no dhcp image-file-update',
  'password manager',
  'password operator',
  '',
...

```

There are simple code examples for each type of switch in the ```./bin``` directory of the the module

## Summary

This module is just a simple beginning and more work is required to clean up the methods to have a better naming scheme as well as error handling.

It can be found on [switchssh repo][switchssh]. The installation instructions are included. My intent it to have this be a [PyPI][PyPI] module for easier installation. Also, I will be refactoring both of my Ansible modules to utilize [switchssh][switchssh].


[comware]: https://github.com/CaptTofu/comware_5_2
[provision]: https://github.com/CaptTofu/pro_vision
[switchssh]: https://github.com/CaptTofu/switchssh
[PyPI]: https://pypi.python.org/pypi
[Solum]: https://wiki.openstack.org/wiki/Solum
[ansible_galaxy]: https://galaxy.ansible.com
[ansible_docker_presentation]: http://www.slideshare.net/PatrickGalbraith/docker-ansible-34909080
[nova_containers_openstack]: http://blog.docker.io/2013/06/openstack-docker-manage-linux-containers-with-nova/
[dockenstack]: https://index.docker.io/u/ewindisch/dockenstack/
[openstack_docker]: https://wiki.openstack.org/wiki/Docker
[openshift]: https://www.openshift.com/?sc_cid=70160000000UJArAAO&gclid=COfd-Oz-5b4CFcHm7AodS1gA7Q
[freebsd_jail]: http://www.freebsd.org/doc/handbook/jails.html 
[docker_image_registry]: https://registry.hub.docker.com/
[helion]: http://www8.hp.com/us/en/cloud/helion-overview.html
[coreos]: http://coreos.com
[systemd]: http://www.freedesktop.org/wiki/Software/systemd/
[etcd]: https://github.com/coreos/etcd
[fleet]: https://github.com/coreos/fleet
[confd]: https://github.com/kelseyhightower/confd
[golang]: http://golang.org
[elkstack]: http://www.elasticsearch.org/webinars/elk-stack-devops-environment/ 
[marcel_blog]: http://marceldegraaf.net/2014/05/05/coreos-follow-up-sinatra-logstash-elasticsearch-kibana.html
[marcel_code]: https://github.com/marceldegraaf/blog-coreos-2
