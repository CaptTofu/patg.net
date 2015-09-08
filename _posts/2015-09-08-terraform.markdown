---
layout: post
title: "Terraform on HP Cloud (Openstack)"
date: 2015-09-08 12:00:00 
categories: cloud,hashicorp,openstack 
---

I recently gave [Terraform][terraform] a test drive, using HP Cloud (Openstack-based) as the cloud provider, and have to say I was pleasantly surprised that it was one of these "just works" types of tools.


## Why??

I am a member of HP Cloud's [Advanced Technology Group](http://hpatg.github.io/) and an important part of our mission is to look for new technologies and exciting new trends. One project I was asked to look at is Terraform. What is Terraform? A project by HashiCorp that is for "building, changing, and versioning infrastructure safely and efficiently". From my experience, it's a logical way of setting up virtual resources that supports multiple sources. 

## Installing Terraform

Installing and setting up Terraform was extremely simple and straightforward, and I was able to get up and running rather quickly, particularly helpful are the [getting started docs][terraform-getting-started].

### Ensure Nova client API is set up

I already had the Nova (and Neutron) tools already installed and have a file that sets up the "OS" environment variables needed to connect to HP cloud east or west. That will be required in advance to ruinning Terraform on HP Cloud. 

### Download Terraform

I ran it on OS X, and found the software on their site:

[https://www.terraform.io/downloads.html][terraform-download]

Download the package appropriate for your system

###  Install Terraform binaries

Knowing that it would be useful to have a workspace, I created a work directory ```~/code/terraform``` where I moved the downloaded file from Terraform into this directory and unzipped it. Since I use Mac Ports, I used /usr/local/bin to move the binaries from the zip file too. 

### Create a Terraform provider file

Referring to the [Terraform Openstack provider documentation][terraform-openstack], I created a work directory``~/code/terraform```, and created the following file, ```hpcloud.tf`` which contained: (specific account details redacted)

```
# Configure the OpenStack Provider
provider "openstack" {
    user_name  = "myusername"
    tenant_name = "tenantfoo"
    password  = "password"
    auth_url  = "https://region-b.geo-1.identity.hpcloudsvc.com:35357/v2.0/"
}

```

### Create a Terraform resource 

In the same file as before, ```hpcloud.tf```, a resource was created along with the provider configuration

```
resource "openstack_compute_instance_v2" "tf-hp-test" {
  name = "tf-hp-test"
  image_id = "ffe8eaa5-2ca0-4ea4-9ee0-fb734279826d"
  flavor_id = "100"
  metadata {
    this= "that"
  }
  key_pair = "patg"
  security_groups = ["coreos"]
  floating_ip = "15.126.223.237"
}
```

### Verify the plan

From the same directory containing the file describing the resource , I run confirmed what the terraform plan was. This basically states what what the state will be once run.

```
(ansible-env)laptop:terraform patg$ terraform plan
Refreshing Terraform state prior to plan...


The Terraform execution plan has been generated and is shown below.
Resources are shown in alphabetical order for quick scanning. Green resources
will be created (or destroyed and then created if an existing resource
exists), yellow resources are being changed in-place, and red resources
will be destroyed.

Note: You didn't specify an "-out" parameter to save this plan, so when
"apply" is called, Terraform can't guarantee this is what will execute.

+ openstack_compute_instance_v2.tf-hp-test
    access_ip_v4:      "" => "<computed>"
    access_ip_v6:      "" => "<computed>"
    flavor_id:         "" => "100"
    flavor_name:       "" => "<computed>"
    floating_ip:       "" => "15.126.223.237"
    image_id:          "" => "ffe8eaa5-2ca0-4ea4-9ee0-fb734279826d"
    image_name:        "" => "<computed>"
    key_pair:          "" => "patg"
    metadata.#:        "" => "1"
    metadata.this:     "" => "that"
    name:              "" => "tf-hp-test"
    network.#:         "" => "<computed>"
    region:            "" => "region-b.geo-1"
    security_groups.#: "" => "1"
    security_groups.0: "" => "coreos"


Plan: 1 to add, 0 to change, 0 to destroy.

```

### Apply

The plan is then applied:

```
(ansible-env)laptop:terraform patg$ terraform apply
openstack_compute_instance_v2.tf-hp-test: Creating...
  access_ip_v4:      "" => "<computed>"
  access_ip_v6:      "" => "<computed>"
  flavor_id:         "" => "100"
  flavor_name:       "" => "<computed>"
  floating_ip:       "" => "15.126.223.237"
  image_id:          "" => "ffe8eaa5-2ca0-4ea4-9ee0-fb734279826d"
  image_name:        "" => "<computed>"
  key_pair:          "" => "patg"
  metadata.#:        "" => "1"
  metadata.this:     "" => "that"
  name:              "" => "tf-hp-test"
  network.#:         "" => "<computed>"
  region:            "" => "region-b.geo-1"
  security_groups.#: "" => "1"
  security_groups.0: "" => "coreos"
openstack_compute_instance_v2.tf-hp-test: Creation complete

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

The state of your infrastructure has been saved to the path
below. This state is required to modify and destroy your
infrastructure, so keep it safe. To inspect the complete state
use the `terraform show` command.

State path: terraform.tfstate

```

### Confirm it worked

A simple ```nova list``` shows that indeed the plan was applied and the resource is was ready to use 

```
(ansible-env)laptop:terraform patg$ nova list
+--------------------------------------+------------+--------+------------+-------------+--------------------------------------------------------------+
| ID                                   | Name       | Status | Task State | Power State | Networks                                                     |
+--------------------------------------+------------+--------+------------+-------------+--------------------------------------------------------------+
| e1822de0-7c48-4ad9-87f4-1d951de37b15 | tf-hp-test | ACTIVE | -          | Running     | advanced_technology_group-network=10.0.0.103, 15.126.223.237 |
+--------------------------------------+------------+--------+------------+-------------+--------------------------------------------------------------+

```

## Summary

That particular day was on that consisted of several things not working and general frustration, so using Terraform was a pleasant surprise. The next step is to do some actual work with Terraform. Of interest to me are former colleagues of mine at Samsung who have a repository, [Kraken](https://github.com/Samsung-AG/kraken.git) that uses Terraform to build [Kubernetes][kubernetes] clusters 


[kubernetes]: https://github.com/GoogleCloudPlatform/kubernetes
[terraform]: https://hashicorp.com/blog/terraform.html
[terraform-download]: https://terraform.io/downloads.html 
[terraform-getting-started]: https://terraform.io/intro/
[terraform-docs]: https://terraform.io/docs/index.html 
[terraform-openstack]: https://terraform.io/docs/providers/openstack/index.html
