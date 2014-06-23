---
layout: post
title: "YAML code in Markdown"
date: 2014-07-03 12:00:00
categories: markdown,yaml,jekyll
---

My friend [Brian Aker](https://en.wikipedia.org/wiki/Brian_Aker) noticed my post from the other day on [Ansible and image building](http://172.16.116.210:4000/ansible,docker,mysql,galera,clustering/2014/06/24/ansible-docker-image-part2/) had some things in the YAML [Ansible] playbooks that looked broken. Sure enough, I didn't notice when proofreading that{% raw %} double  ``` {{ ``` and  ``` }} ``` {% endraw %}

curly braces weren't showing up! That was definite breakage but upon verification of the playbook in my [repo][docker_dna_repo], it was not broken and I realized that my markdown in my [Jekyll-based][jekyll] [github pages][github_pages] [website/blog](http://patg.net) had manged the YAML I inteded to talk about.

Well, this is because [Jekyll][jekyll] uses liquid tags and rips these out. The solution? It took me a several attempts, trial-and-error. Some of what I tried:

{% raw %}
```
\{\{ item \}\}
```
Nope.

```
\{{ item \}}
```

Nope.

```{{ item }}```


Nope.

What is the trick?

Quite simply (and trying to do the below so it's visible was NOT easy!) :

```{%``````raw %}```
<br />
```{{ item }}```
<br />
```{%``````endraw %}```

{% endraw %}

That's all! There was a link [here (hat tip!)](http://stackoverflow.com/questions/24102498/escaping-double-curly-braces-inside-a-markdown-code-block-in-jekyll)

[Docker]: http://docker.io
[Dockerfile]: http://docs.docker.com/reference/builder/
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
[ansible_docker_presentation_repo]: https://github.com/CaptTofu/ansible-docker-presentation
[moonshot]: http://h17007.www1.hp.com/us/en/enterprise/servers/products/moonshot/index.aspx#.U0gU2PldXJh?jumpid=ps_r163&k_clickid=AMS|112|61913|169782eb-9c7a-df89-4afe-0000588f5dc0
[moonshot_cartridge]: http://www8.hp.com/us/en/products/proliant-servers/product-detail.html?oid=5375897#!tab=features
[docker_image_source]: https://github.com/CaptTofu/docker-image-source
[docker_ansible_blog]: http://patg.net/ansible,docker/2014/06/18/ansible-docker/
[docker_ansible_images_blog]: http://patg.net/ansible,docker/2014/06/20/ansible-docker-image/ 
[docker_image_hub]: https://registry.hub.docker.com/
[building_docker_with_ansible]: http://www.ansible.com/blog/2014/02/12/installing-and-building-docker-with-ansible
[haproxy]: http://www.haproxy.org/
[angstwad_repo]: https://github.com/angstwad/docker.ubuntu
[paul_durivage]: https://github.com/angstwad
[michael_dehaan]: http://michaeldehaan.net/
[docker_dna_repo]: https://github.com/wrale/docker-dna
[galera]: http://galeracluster.com/
[percona_xtradb_cluster]: http://www.percona.com/software/percona-xtradb-cluster
[percona]: http://www.percona.com
[galera_role]: https://github.com/CaptTofu/docker-dna/tree/master/galera
[github_pages]: https://pages.github.com/
[jekyll]: http://jekyllrb.com/
