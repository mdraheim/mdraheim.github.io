---
layout: post
title:  "Rancher Development Test Cluster in Your Pocket"
date:   2023-09-03 18:00:00 +0200
categories: tools
---

You can have a fully functional [Rancher](https://www.rancher.com){:target="_blank"} k3s kubernetes cluster on your laptop with just three lines of bash script. And it is not even cluttering your disk with a hundred containers and lots of configuration files all over the filesystem.

In fact, it is running in a single container and only requires one local directory for persistence. That is so iffing awesome!

Of course, you don't get the performance of a Falcon Heavy going supersonic. My workhorse laptop has 4 cpu posing as 8 vcpu and 16G of RAM. And that is sufficient to run kubernetes with istio and one stack of a two-legged Confluence Datacenter on top of a postgres server.

{% highlight bash %}
#! /bin/bash

# this will run a single node rancher cluster
#   via docker in docker
# so all containers spin up inside the rancher container

# persistence
#  creates also a subdir "pv" as a base for hostpath volumes
PWD=$(pwd)
[ -d $PWD/rancher ] || mkdir -p $PWD/rancher/pv

# sudo is for binding the privileged host ports
# first two port bindings are for the rancher server frontend
# the last two are istio ingress, which requires rollout of
#   istio helm chart from within the running rancher
sudo docker run --rm -ti \
        --privileged \
        --name rancher_server \
        -p 8080:80 \
        -p 8443:443 \
        -p 6443:6443 \
        -p 80:31380 \
        -p 443:31390 \
        -v $PWD/rancher:/var/lib/rancher \
        rancher/rancher:latest

# watch console output for the initial admin password
#
# point your browser at http://localhost:8080
#
# set admin password, you only need to do this once

{% endhighlight %}

That is all. Initial startup will take a few minutes as rancher is pulling all the required images. But once that is done and you have set the admin password to something development friendly like, errm, admin, you can start deploying workloads. Or just marvel at the sight. Or go and impress some noobs. Or managers.

![rancher aio](/assets/images/20230903_01.png)

Once you are done boasting about how much work this setup cost you, start doing something productive. Get the kubeconfig from rancher frontend, start helming...


