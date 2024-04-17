
# Using Terraform With Godaddy

So, to start off this post, I haven't posted on my blog for, a while. I had just been super busy both at Oracle Cloud and then in the role of CTO and CIO at Parler, which after a day of work, and having kids at the premium age for interaction, and I suppose fatigue from being in this industry since 1993, and really technical field going back to the day I went into the Navy in 1986, you get the point!

In any case, one of the things I thought about when I was climbing Aconcagua and spending time with Peter Zaitsev, who is a tremendous person both professionally as well as in general, I want to get back to contributing to the open-source and technical community. I wrote two books and can put out material quickly when I get down to it, so here goes the resumption of that endevour. 

## Why this post?

One of the things I've been needing to do, especially to keep my skills up to date, is to refresh some of my knowledge on using cloud-specific technologies. For the last three years, I've been working in environments where we had to build our own infrastucture and not having the luxury of using tools like Terraform, or using AWS, though I have written books of Ansible to manage databases (MySQL and Postgres) as well as various varieties of do-it-yourself Kubernetes clusters. 

The first task I wanted to do was to build my own EKS Kubernetes cluster and use a domain I have, which happens to be "kubernetes-cluster.com" (I obtained it in early days). To do this, I realized I needed to modify my NS records for that domain, which the registrar is Godaddy, and I want to use Route 53 to manage it and in turn, use External DNS with Kubernetes. 

I thought "OK, I have two things I need to do that can be done with one project. Modify the kubernetes-cluster.com's NS records _and_ do something useful with Terraform. And for sure, there was a provider for Godaddy!

# The Godaddy Provider for Terraform

The Goddady Provider for Terraform is incredibly simple to use. For the sake of getting this blog post done, I'm just going to show you the `main.tf` file that I created to change the nameservers for kubernetes-cluster.com to use AWS's Route53 DNS. I had never used this provider before, so there is no existing state, but simply specifying the nameservers for Route53, it replaced the old nameservers with what I wanted.

`main.tf`:

```
terraform {
  required_providers {
    godaddy = {
      source = "n3integration/godaddy"
      version = "1.9.1"
    }
  }
}

provider "godaddy" {
  # Configuration options
}

provider "aws" {
  region  = "us-west-2"
  profile = "default"

  backend s3 {
    bucket = "tf-godaddy-state"
    key    = "domains/kubernetes_cluster_com"
    region = "us-east-1"
  }
}
resource "godaddy_domain_record" "kubernetes_cluster_com" {
  domain   = "kubernetes-cluster.com"

  // your Godaddy customer ID obviously
  customer = "123456788888888"

  // AWS nameservers specified in Route53
  nameservers = ["ns-526.awsdns-01.net","ns-1521.awsdns-62.org","ns-1975.awsdns-54.co.uk","ns-5.awsdns-00.com"]

}
```

First run `terraform init` (I use an alias `tf` set in my `~/.bashrc`)

```
$ tf init

Initializing the backend...

Initializing provider plugins...
- Reusing previous version of n3integration/godaddy from the dependency lock file
- Reusing previous version of hashicorp/aws from the dependency lock file
- Using previously-installed n3integration/godaddy v1.9.1
- Using previously-installed hashicorp/aws v5.45.0

Terraform has been successfully initialized!

```

Then of course, run `terraform plan`:

```
tf plan

Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following
symbols:
  + create

Terraform will perform the following actions:

  # godaddy_domain_record.kubernetes_cluster_com will be created
  + resource "godaddy_domain_record" "kubernetes_cluster_com" {
      + customer    = "12345678"
      + domain      = "kubernetes-cluster.com"
      + id          = (known after apply)
      + nameservers = [
          + "ns-526.awsdns-01.net",
          + "ns-1521.awsdns-62.org",
          + "ns-1975.awsdns-54.co.uk",
          + "ns-5.awsdns-00.com",
        ]
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

Then of course run `terraform apply`. I won't include that output for the purpose of brevity, and because it's what you'd expect. 

Import to note: after I changed the nameservers from Goddady providing DNS, this makes it so that Goddady no longer has a zone file for kubernetes-cluster.com and is only serving as a registrar, so I can't use the Terraform provider if I want to make changes to the domain and will need to use the Route53 terraform provider.

I also used this provider for adding a cname for patg.net:

```
resource "godaddy_domain_record" "patg-net" {
  domain   = "patg.net"

  // required if provider key does not belong to customer
  customer = "12345678"

  record {
    name = "blog"
    type = "CNAME"
    data = "patg.net"
    ttl = 3600
  }

}
```

Again, quite simple, using the same steps (init, plan, apply) and being only adding a `CNAME`, I can continue to use this provider.

Have a great day, and thanks for reading my post!
