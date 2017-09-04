---
layout: post
title: "Running XtraBackup on Windows using WSL"
date: 2017-09-03 12:00:00 
categories: mysql,windows 10,wsl,backups
---



## Backing up MySQL on Windows: options? 

I had a recurring consultation this summer with a client who is running a stand-alone MySQL database with a ton of data and wanted to simply have a backup system for that data. The client is very risk-averse and didn't want anything too experimental. I knew for a huge database, I didn't want to run mysqldump on the database on a regular basis if I could avoid it and always prefer using XtraBackup.

I haven't worked on Windows in years, and had assumed there were binaries for XtraBackup, but as with so many things with Windows, there are a million assumptions I had made prior that I have learned the hard way throughout the summer. 

I did find Vadim's excellent [article](https://www.percona.com/blog/2017/03/20/running-percona-xtrabackup-windows-docker/) using a Docker container to run a container with XtraBackup, and bind mounts of both the datadir and the backup directory so the container can run a backup and have the backup show up on the host where specified in the arguments. This is certainly the way I gravitate to as I find containers a great solution to so many problems. However, again, I mentioned the client wanted something that isn't perceived to be to experimental, so I wanted to explore using something that was "part" of Windows.

## WSL: Windows Subsystem for Linux

Microsoft certainly is a different company than it used to be. Microsoft has really put a [lot of work into being part of Open Source](https://opensource.microsoft.com/), VSCode, the container ecosystem, Kubernetes, Open-Sourcing PowerShell, etc. Also one can consider the context of Microsoft being serious about attracting developers to their platform. In addition to these other efforts, my friend James Hancock, an expert in all things Windows, told me about WSL, the Windows Subsystem for Linux. 

I had always known that Windows had this idea of a base kernel with subsytems, both a Windows subsystem as well as a POSIX subsystem. I hadn't used Windows much in years, so I didn't know the status of what you could do with Windows in current times.

WSL was an effort inside of Microsoft researching real time translation of Linux system calls into Windows OS system calls, and derived from their work in working with [Android apps to run on Windows 10 Mobile](https://arstechnica.com/information-technology/2016/04/why-microsoft-needed-to-make-windows-run-linux-software/).

## What is WSL really?

Throughout the years, there have been a number of ways to run "unix" or posix commands on Windows with some sort of bash shell for those who are Linux people to get by when working on Windows. WSL is much more than that though. 

#### WSL is not CygWin nor MinGW. 

Those are great tools, but not a complete subsystem like WSL that integrates into the Windows OS

#### WSL is not a container nor VM running Linux

WSL is part of the OS, a subsystem just like Windows itself and equal footing and not just an accessory program

#### WSL is currently like having Ubuntu on Windows

When you run WSL, you have an Ubuntu installation you can do all the same things for the most part you would if you were running on a VM or bare metal. 

On WSL, there exists a regular Linux file system with Windows fixed drives mounted as ```/mnt/<drive letter>``` so that you have access to you Windows files as well.

It's also an excellent development platform as now one can develop both for Windows or Linux.

## Using WSL

It's part of the Windows OS, so it just needs to be set up to run.

Basically, it can be enabled through enabling developer mode and turning on the WSL feature in Settings->Update & Security, as detailed in [this article](https://solarianprogrammer.com/2017/04/15/install-wsl-windows-subsystem-for-linux/) 

Or using a simple PowerShell command as detailed in [this article](https://msdn.microsoft.com/en-us/commandline/wsl/install_guide)

    Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux

It can also be run as a specific user if desired

    LxRun.exe /setdefaultuser <new_name>


## What does WSL mean for XtraBackup?

Since there aren't any Windows binaries XtraBackup, one has to either use a Docker container or WSL.

For WSL, it is Ubuntu. So, to install XtraBackup for WSL, install XtraBackup as it would be installed on Ubuntu! Yes, that easy.

## Setting up XtraBackup on WSL

The current Ubuntu version for WSL is xenial, so that is the version, using lsb-release to get the right package version.The lsb package will need to be installed to make it work. 

    apt-get update
    apt-get install lsb
    wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb
    sudo dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb
    sudo apt-get update
    sudo apt-get install percona-xtrabackup-24


The MySQL version on the server was installed from [Bitnami's WAMP](https://bitnami.com/stack/wamp) package which installs MySQL in ```C:\Bitnami\wampstack-7.1.7-0\mysql\data```, but we modified the datadir to be in C:\mysqldata, which is accessible as ```/mnt/c/mysqldata``` in WSL. 

The first thing is to decide where to back up data. 

IMPORTANT: the current version of Windows, WSL cannot access USB drives, so you if data needs to be on a detachable USB drive, it'll need to be copied from the attached storage to the USB drive. The current insider release does address this and soon this problem won't exist on regular releases. 

For a backup directory, In our case, we chose ```C:\backups``` for simplicity.

    mkdir /mnt/c/backups

So, with a running MySQL instance and a place to create backups, XtraBackup can be run via ```innobackupex```

    innobackupex --datadir=/mnt/c/mysqldata /mnt/c/backups
    innobackupex --apply-log /mnt/c/backups/2017-09-xxx

And That's it! Now you can have yet another way to back up MySQL on Windows!

![xtrabackup wsl](/assets/wsl4.PNG)
