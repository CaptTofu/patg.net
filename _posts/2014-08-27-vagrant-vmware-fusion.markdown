---
layout: post
title: "Running VMware Fusion using Vagrant"
date: 2014-08-27 12:00:00
categories: vagrant,vmware,hypervisors
---

Greetings! This is a very quick post to get out my notes on setting up Vagrant to run [VMware Fusion][vmware_fusion] on OS X Mavericks. The instructions on [Vagrant's site][vagrant_vmware_fusion_plugin_install] are decent enough, but I though some things were missing, primarily that you need to actually buy the [provider plugin][vagrant_vmware_fusion_plugin] and for some reason was thinking it meant that you have to buy VMware itself (which I of course do own), and how to get the license file.


Here are the steps that made my life easier:

- Make sure to install [rvm][rvm] and make sure any gem you install is user-level and not requiring sudo. Maybe this seems straightforward, but I ended up having to re-install RVM as I had been using Linux so much, that my Mac's setup needed some love.
- Make sure your installation of [Homebrew][homebrew] is in good shape. Run ```brew doctor``` to ensure this.
- Install [nokogiri/libiconv][nokogiri] and the gem per your version of [Homebrew][homebrew]
- Install the vagrant [provider plugin][vagrant_vmware_fusion_plugin] for Vmware Fusion

```$ vagrant plugin install vagrant-vmware-fusion```

- Buy the vagrant vmware_fusion plugin license (cost $79)
- After purchase, you will get an email with the license. Save it to a location you will not misplace it
vagrant plugin license vagrant-vmware-fusion ```/path/to/where/you/saved/license.lic```
- Run the ```vagrant plugin list``` command:

```
    $ vagrant plugin list
    vagrant-login (1.0.1, system)
    vagrant-share (1.1.0, system)
    vagrant-vmware-fusion (2.5.2)
```
You should now be able to launch VMware images using Vagrant!


[vagrant_vmware_fusion_plugin_install]: https://docs.vagrantup.com/v2/vmware/installation.html
[vmware_fusion]: http://www.vmware.com/products/fusion
[vagrant_vmware_fusion_plugin]: https://www.vagrantup.com/vmware
[rvm]: http://rvm.io
[homebrew]: http://brew.sh
[nokogiri]: http://nokogiri.org/tutorials/installing_nokogiri.html
