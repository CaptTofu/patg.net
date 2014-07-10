---
layout: post
title: "Docker: Containers for the Masses -- The docker_facts module" 
date: 2014-07-10 12:00:00
categories: ansible,docker
---

It has been over a week or so and I have realized that I need to continue delivering blog posts in the series "Docker: Containers for the Masses" by covering a module for Ansible that I developed, [docker_facts][ansible_docker_facts_repo]

This blog post is the latest in the series:

- [Introduction][docker_intro_blog]
- [Installation][docker_install_blog]
- [Using Docker][docker_using_blog]
- [Ansible and Docker][docker_ansible_blog]
- [Building Docker Images with Ansible][docker_images_ansible_blog]
- [Building Docker Images with Ansible (yet another way)][docker_images_ansible_blog2]

<br />

## What are Ansible facts?

[Ansible][Ansible] makes available to a user what are known as "facts". [Ansible facts][ansible_facts] are information that [Ansible][Ansible] obtains, or discovers, from systems it manages. These facts are gathered and can be readily-used in playbooks and templates. Whatever one normally needs to do, particularly with a conditional where a decision has to be made based on something particular with a system.

To see what facts a system has, the example below being for ```localhost```, run:

    $ ansible localhost -m setup

The output is a huge dictionary with numerous entries that pertain to every possible thing about a system one would need to know: devices, networking, OS version, any environment varialbles, paths, etc. To see an example, the [Ansible documentation][ansible_facts] shows this.

How does [Ansible][Ansible] do this? It connects to all nodes specified (in the inventory or by host) and gathers this information into the aforementioned dictionary. This can certainly take time and may not always be desireable, which one can use

 ```gather_facts: no```

in a given playbook to not gather this information. If there are hundreds or even thousands of hosts in question, this can make a huge difference.

There are also other types of facts one can make use of through a number of modules and can even extend [Ansible][Ansible] though a module that collects its own facts for a given environment, particularly cloud environments such as EC2, RackSpace, or Big IP.

Recently, when working with the Ansible [Docker plugin][ansible_docker], it became apparent to myself and others that there needed to be a facts module for Docker as well. Hence, why I developed this plugin which shall be covered in this blog post.

## Docker Facts 

The [```docker_facts```][ansible_docker_facts_repo] module was developed because of the need to be able to obtain information about a Docker fleet. A facts module was the obvious solution to this, something that provides a dictionary of information about one or more containers as well as images. The ```docker_facts``` module is very simple. It needs to construct a dictionary that contains this information, it is necessary for the module to utilize the Docker python [client library][docker_py] to talk to a docker daemon which provides an API for this client to communicate with. 

It should be noted that this module is not part of the standard distribution of [Ansible][Ansible] nor is yet merged with [Ansible's development repo][ansible_repo]. It can be found at a [forked repo][ansible_docker_facts_repo].

## Using the docker_facts module

Using the ```docker_facts``` Ansible module is very simple. When you use the ```docker_facts``` module by specifying:

```module : docker_facts```

in a placebook, two dictionaries can then be referred to in a playbook or template called ```docker_containers``` and ```docker_images```.

This first example shows the use of this module to return a single dictionary that the playbook then iterates over ```docker_containers``` dictionary using the ```with_dict``` loop iterator. Since ```docker_containers``` is representative of the of all containers, regardless of state, all containers will have an entry in this dictionary.

{% raw %}
    - name: Gather info about containers
      hosts: docker
      gather_facts: True
      tasks:
        - name: Get facts about containers
          local_action:
            module: docker_facts

        - name: another facts test
          debug: msg="Container {{ item.key }}"
          with_dict: docker_containers
{% endraw %}
<br />

The output of the above playbook is shown below. Docker returns 2 running containers as well as 2 killed containers. If one runs this playbook, the output is quite long, even with only 4 containers. The output below has been edited such that the first container's output is complete, meanwhile, the remaining containers' output is abridged. The last container isn't running and output showing the state was left intact so that it can be seen how a killed container still has all the information but that the state of it no longer running is apparent. 

    $ ansible-playbook -i hosts facts_all.yml

    PLAY [Gather info about containers] *******************************************

    GATHERING FACTS ***************************************************************
    ok: [localhost]

    TASK: [Get facts about containers] ********************************************
    ok: [localhost]
    
    TASK: [another facts test] ****************************************************
    ok: [localhost] => (item={'value': {u'docker_state': {u'Pid': 27749, u'Paused': False, u'Running': True, u'FinishedAt': u'0001-01-01T00:00:00Z', u'StartedAt': u'2014-07-10T05:29:35.919528163Z', u'ExitCode': 0}, u'docker_ports': [{u'Type': u'tcp', u'PrivatePort': 22}, {u'Type': u'tcp', u'PrivatePort': 443}, {u'Type': u'tcp', u'PrivatePort': 80}], u'docker_name': u'/container_3', u'docker_mountlabel': u'', u'docker_volumesrw': {}, u'docker_hostnamepath': u'/var/lib/docker/containers/53762ab286a5a7b1cdbd7b09faef722f34e666fe9450fce58b4ae31e7e84ba75/hostname', u'docker_networksettings': {u'Bridge': u'docker0', u'PortMapping': None, u'Gateway': u'172.17.42.1', u'IPPrefixLen': 16, u'IPAddress': u'172.17.0.4', u'Ports': {u'22/tcp': None, u'443/tcp': None, u'80/tcp': None}}, u'docker_names': [], u'docker_image': u'capttofu/apache:latest', u'docker_driver': u'aufs', u'docker_resolvconfpath': u'/etc/resolv.conf', u'docker_path': u'/usr/local/sbin/entrypoint.sh', u'docker_volumes': {}, u'docker_args': [], u'docker_execdriver': u'native-0.2', u'docker_hostspath': u'/var/lib/docker/containers/53762ab286a5a7b1cdbd7b09faef722f34e666fe9450fce58b4ae31e7e84ba75/hosts', u'docker_created': 1404970175, u'docker_hostconfig': {u'ContainerIDFile': u'', u'NetworkMode': u'', u'LxcConf': None, u'Binds': None, u'Dns': None, u'DnsSearch': None, u'Privileged': False, u'VolumesFrom': None, u'Links': None, u'PortBindings': None, u'PublishAllPorts': False}, u'docker_status': u'Up 12 minutes', u'docker_config': {u'Env': [u'HOME=/', u'PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin'], u'Hostname': u'53762ab286a5', u'Entrypoint': [u'/usr/local/sbin/entrypoint.sh'], u'PortSpecs': None, u'Memory': 0, u'OnBuild': None, u'OpenStdin': False, u'Cpuset': u'', u'User': u'', u'AttachStderr': False, u'AttachStdout': False, u'NetworkDisabled': False, u'WorkingDir': u'', u'Cmd': None, u'StdinOnce': False, u'AttachStdin': False, u'Volumes': None, u'MemorySwap': 0, u'Tty': False, u'CpuShares': 0, u'Domainname': u'', u'Image': u'capttofu/apache', u'ExposedPorts': {u'22/tcp': {}, u'443/tcp': {}, u'80/tcp': {}}}, u'docker_command': u'/usr/local/sbin/entrypoint.sh', u'docker_short_id': u'53762ab286a5a', u'docker_processlabel': u'', u'docker_id': u'53762ab286a5a7b1cdbd7b09faef722f34e666fe9450fce58b4ae31e7e84ba75'}, 'key': u'container_3'}) => {
        "item": {
            "key": "container_3",
            "value": {
                "docker_args": [],
                "docker_command": "/usr/local/sbin/entrypoint.sh",
                "docker_config": {
                    "AttachStderr": false,
                    "AttachStdin": false,
                    "AttachStdout": false,
                    "Cmd": null,
                    "CpuShares": 0,
                    "Cpuset": "",
                    "Domainname": "",
                    "Entrypoint": [
                        "/usr/local/sbin/entrypoint.sh"
                    ],
                    "Env": [
                        "HOME=/",
                        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
                    ],
                    "ExposedPorts": {
                        "22/tcp": {},
                        "443/tcp": {},
                        "80/tcp": {}
                    },
                    "Hostname": "53762ab286a5",
                    "Image": "capttofu/apache",
                    "Memory": 0,
                    "MemorySwap": 0,
                    "NetworkDisabled": false,
                    "OnBuild": null,
                    "OpenStdin": false,
                    "PortSpecs": null,
                    "StdinOnce": false,
                    "Tty": false,
                    "User": "",
                    "Volumes": null,
                    "WorkingDir": ""
                },
                "docker_created": 1404970175,
                "docker_driver": "aufs",
                "docker_execdriver": "native-0.2",
                "docker_hostconfig": {
                    "Binds": null,
                    "ContainerIDFile": "",
                    "Dns": null,
                    "DnsSearch": null,
                    "Links": null,
                    "LxcConf": null,
                    "NetworkMode": "",
                    "PortBindings": null,
                    "Privileged": false,
                    "PublishAllPorts": false,
                    "VolumesFrom": null
                },
                "docker_hostnamepath": "/var/lib/docker/containers/53762ab286a5a7b1cdbd7b09faef722f34e666fe9450fce58b4ae31e7e84ba75/hostname",
                "docker_hostspath": "/var/lib/docker/containers/53762ab286a5a7b1cdbd7b09faef722f34e666fe9450fce58b4ae31e7e84ba75/hosts",
                "docker_id": "53762ab286a5a7b1cdbd7b09faef722f34e666fe9450fce58b4ae31e7e84ba75",
                "docker_image": "capttofu/apache:latest",
                "docker_mountlabel": "",
                "docker_name": "/container_3",
                "docker_names": [],
                "docker_networksettings": {
                    "Bridge": "docker0",
                    "Gateway": "172.17.42.1",
                    "IPAddress": "172.17.0.4",
                    "IPPrefixLen": 16,
                    "PortMapping": null,
                    "Ports": {
                        "22/tcp": null,
                        "443/tcp": null,
                        "80/tcp": null
                    }
                },
                "docker_path": "/usr/local/sbin/entrypoint.sh",
                "docker_ports": [
                    {
                        "PrivatePort": 22,
                        "Type": "tcp"
                    },
                    {
                        "PrivatePort": 443,
                        "Type": "tcp"
                    },
                    {
                        "PrivatePort": 80,
                        "Type": "tcp"
                    }
                ],
                "docker_processlabel": "",
                "docker_resolvconfpath": "/etc/resolv.conf",
                "docker_short_id": "53762ab286a5a",
                "docker_state": {
                    "ExitCode": 0,
                    "FinishedAt": "0001-01-01T00:00:00Z",
                    "Paused": false,
                    "Pid": 27749,
                    "Running": true,
                    "StartedAt": "2014-07-10T05:29:35.919528163Z"
                },
                "docker_status": "Up 12 minutes",
                "docker_volumes": {},
                "docker_volumesrw": {}
            }
        },
        "msg": "Container container_3"
    }
    ok: [localhost] => (item={'value': {
       ... < SNIP > ...
        },
        "msg": "Container container_2"
    }
    ok: [localhost] => (item={'value': {
        ... < SNIP > ...
        },
        "msg": "Container container_1"
    }
    ok: [localhost] => (item={'value': {
        "item": {
            "key": "trusting_lumiere",
            "value": {
                "docker_args": [],
                "docker_command": "/bin/bash",
                "docker_config": {
                ... < SNIP > ...
                },
                "docker_created": 1403707947,
                "docker_driver": "aufs",
                "docker_execdriver": "native-0.2",
                "docker_hostconfig": {
                ... < SNIP > ...
                },
                "docker_hostnamepath": "/var/lib/docker/containers/b2e2cc2424f940f34e3932913219c2640295a394c267ead58267e13fdcd132c7/hostname",
                "docker_hostspath": "/var/lib/docker/containers/b2e2cc2424f940f34e3932913219c2640295a394c267ead58267e13fdcd132c7/hosts",
                "docker_id": "b2e2cc2424f940f34e3932913219c2640295a394c267ead58267e13fdcd132c7",
                "docker_image": "ee8da8df5e1c",
                "docker_mountlabel": "",
                "docker_name": "/trusting_lumiere",
                "docker_names": [],
                "docker_networksettings": {
        
                },
                "docker_path": "/bin/bash",
                "docker_ports": [
                ... <SNIP> ...
                ],
                "docker_processlabel": "",
                "docker_resolvconfpath": "/etc/resolv.conf",
                "docker_short_id": "b2e2cc2424f94",
                "docker_state": {
                    "ExitCode": 0,
                    "FinishedAt": "2014-06-25T14:55:05.926785131Z",
                    "Paused": false,
                    "Pid": 0,
                    "Running": false,
                    "StartedAt": "2014-06-25T14:52:27.190492471Z"
                },
                "docker_status": "Exited (0) 2 weeks ago",
                "docker_volumes": {},
                "docker_volumesrw": {}
            }
        },
        "msg": "Container trusting_lumiere"
    }

    PLAY RECAP ********************************************************************
    localhost                  : ok=3    changed=0    unreachable=0    failed=0

<br />
## A specific container

A specific container can also be given, as shown in the example below:

       - name: Get facts about containers
          local_action:
            module: docker_facts
            name: db_1

<br />
This would simply result in only that container being specified in the ```docker_containers``` dictionary.

## Images

As mentioned, this module can also return information about [Docker][Docker] images. One need only specify ```images: all``` or with a specific image ID. 

{% raw %}
    tasks:
        - name: Get facts about containers
          local_action:
            module: docker_facts
            images: all

        - name: images info
          debug: msg="Image ID {{ item.key }} Created {{ item.value.docker_created }} Repo Tags {{ item.value.docker_repotags }}"
          with_dict: docker_images

{% endraw %}
<br />

As with containers, the output is quite verbose and the debug message results in:

    $ ansible-playbook -i hosts facts_images.yml |grep msg
        "msg": "Image ID 1e47da3640f15 Created 1391071730 Repo Tags [u'capttofu/docker-dna_base:0.1.0']"
        "msg": "Image ID 634d3b1221fc5 Created 1402898757 Repo Tags [u'<none>:<none>']"
        "msg": "Image ID 2baeb2edbf924 Created 1400511467 Repo Tags [u'<none>:<none>']"
        "msg": "Image ID 02dae1c13f51e Created 1398356200 Repo Tags [u'<none>:<none>']"
        "msg": "Image ID 0c5d1a004bb22 Created 1402894889 Repo Tags [u'<none>:<none>']"
        "msg": "Image ID 37fca75d01ffc Created 1401926739 Repo Tags [u'busybox:ubuntu-14.04']"
        "msg": "Image ID 6b0a59aa7c486 Created 1401930544 Repo Tags [u'ubuntu:13.04', u'ubuntu:raring']"
        "msg": "Image ID 0ccb32b6dbbe6 Created 1402898540 Repo Tags [u'<none>:<none>']"
        "msg": "Image ID 3d2fc1306b1ee Created 1403689854 Repo Tags [u'<none>:<none>']"
        "msg": "Image ID 5c640c2fc451f Created 1402354657 Repo Tags [u'brendanburns/php-redis:latest']"
        "msg": "Image ID a31ebdaaa8d87 Created 1402898593 Repo Tags [u'capttofu/haproxy:latest']"
<br />

## A less verbose and more pragmatic example

The previous examples were very simplistic and a bit of a hose on full blast, particularly with the one returning all containers! A more pragmatic example and one that shows how useful ```docker_facts``` is would be the following snippet that prints out the container name, IP address only for running containers using the ```when``` statement.

{% raw %}
       - name: a more practical example 
         debug: msg="Host{{':'}} {{ inventory_hostname}} Container Name{{':'}} {{ item.key }} IP Address{{':'}} {{ item.value.docker_networksettings.IPAddress }}"
         when: item.value.docker_state.Running == True
         with_dict: docker_containers
{% endraw %}
 
<br />
To run this playbook and only capture the actual debug message, the play can be pipe through ```grep```:

    $ ansible-playbook -i hosts facts.yml |grep msg
        "msg": "Host: localhost Container Name: container_3 IP Address: 172.17.0.4"
        "msg": "Host: localhost Container Name: container_2 IP Address: 172.17.0.3"

<br />
This certainly is an example of what useful things this module allows you to do!

## Using ```docker_facts``` with a template

It would also be possible to generate an inventory file using this module that could then be used to manage [Docker][Docker] containers with [Ansible][Ansible]. The example below is a playbook recently used for [AnsibleFest][AnsibleFest] from the [presenation][ansible_docker_presentation] where a demonstration was given showing Docker containers running on 45 [HP Moonshot][hp_moonshot] [cartridges][moonshot_cartridge]. Using this snippet, it was possible to build an inventory file grouped by host for all containers.

{% raw %}
    ---

    - name: Create an invetory file
      hosts: moonshot
      gather_facts: yes
      tasks:
        - name: Get facts about containers
          local_action:
            docker_url: tcp://{{ inventory_hostname }}:4243
            module: docker_facts

        - name: docker_hosts template
          local_action: template src=docker_hosts.txt.j2 dest=./docker_hosts_{{ inventory_hostname }}.txt

{% endraw %}
<br />

The output file, would be named by host, .txt, and the template that generates this file is the following:

{% raw %}
    {% for host in hostvars | sort %}
    [{{ host }}]
    {% for container in docker_containers | sort %}
    {{ container }} ansible_ssh_port={{ docker_containers[container]['docker_networksettings']['Ports']['22/tcp'][0]['HostPort'] }}
                    ansible_ssh_host={{ host }}
    {% endfor %}
    {% endfor %}
{% endraw %}
<br />

The final output of each file would be:

    [c10n1.atg.seattle.lan]
    c19n1_db_1 ansible_ssh_port=49270 ansible_ssh_host=c10n1.atg.seattle.lan
    c19n1_db_2 ansible_ssh_port=49275 ansible_ssh_host=c10n1.atg.seattle.lan
    c19n1_db_3 ansible_ssh_port=49280 ansible_ssh_host=c10n1.atg.seattle.lan
    c19n1_haproxy_1 ansible_ssh_port=49285 ansible_ssh_host=c10n1.atg.seattle.lan
    c19n1_haproxy_2 ansible_ssh_port=49287 ansible_ssh_host=c10n1.atg.seattle.lan
    c19n1_haproxy_3 ansible_ssh_port=49289 ansible_ssh_host=c10n1.atg.seattle.lan
    c19n1_haproxy_4 ansible_ssh_port=49291 ansible_ssh_host=c10n1.atg.seattle.lan
    c19n1_web_1 ansible_ssh_port=49240 ansible_ssh_host=c10n1.atg.seattle.lan
    ...

<br />
### Summary

As this post shows, the [Ansible][Ansible] [```docker_facts```][ansible_docker_facts_repo] module is very useful for providing information about Docker instances on one or more hosts and if one is creative, can do all sorts of tricks to extend the power Ansible has in managing a fleet Docker containers and images.

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
