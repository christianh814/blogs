# OpenShift 4.2 vSphere Install Quickstart

In this blog we will go over how to get you up and running with an OpenShift 4.2 install on VMware vSphere. There are many methods to work with vSphere that allows automating in creating the necessary resources for installation. These include using [Terraform](https://github.com/openshift/installer/tree/master/upi/vsphere) and [Ansible](https://github.com/vchintal/ocp4-vsphere-upi-automation) to help expedite the creation of resources. In this blog, we will focus on getting familiar with the process; so I will be going over how to do the process manually. 

## Environment Overview

For this installation I am using vSphere version 6.7.0 and ESXi version 6.7.0 Update 3. I will be following [the official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html) for installing OpenShift 4 on vSphere. There, you can read more information about prerequisites including the need to set up DNS, DHCP, Load Balancer, Artifacts, and other ancillary services/items. I will be going over the prerequisites for my environment.

## Prerequisites

It's important that you get familiar with the [prerequisites](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-infrastructure-user-infra_installing-vsphere) by reading the [official documentation for OpenShift](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-infrastructure-user-infra_installing-vsphere). I will go over the prerequisites at a high level, and link examples. It's important to note that, although this is user provisioned infrastructure, the OpenShift 4 installer is specific about how things are named; and what it expects to be there. 

## vSphere Credentials

I will be using my administrative credentials for vSphere. I will also be passing these credentials to the OpenShift 4 installer and, by extension, to the OpenShift cluster. It's not a requirement to do so, and you can install without passing the credentials. This will effectively turn your installation into a "Bare Metal" type of installation and you'll lose the ability to dynamically create VDMKs for your applications at install time. You can set this up post-installation (we will go over that later).

## DNS

The first consideration you need to take into account when setting up DNS for OpenShift 4 is the "cluster id". The "cluster id" uniquely identifies each OpenShift 4 cluster in your domain, and this ID also becomes part of your cluster's FQDN. The combination of your "cluster id" and your "domain" creates what I like to call a "cluster domain". For example; with a cluster id of `openshift4` and my domain of `example.com`, the cluster domain (i.e. the FQDN) for my cluster is `openshift4.example.com`.

DNS entries are created using the `$CLUSTERID.$DOMAIN` cluster domain FQDN nomenclature. All DNS lookups will be based on this cluster domain. Using my example cluster domain, `openshift4.example.com`, I have the following DNS entries set up in my environment. Note that the etcd servers are pointed to the IP of the masters, and they are in the form of `etcd-$INDEX`

```
[chernand@laptop ~]$ dig master1.openshift4.example.com +short
192.168.1.111
[chernand@laptop ~]$ dig master2.openshift4.example.com +short
192.168.1.112
[chernand@laptop ~]$ dig master3.openshift4.example.com +short
192.168.1.113
[chernand@laptop ~]$ dig worker1.openshift4.example.com +short
192.168.1.114
[chernand@laptop ~]$ dig worker2.openshift4.example.com +short
192.168.1.115
[chernand@laptop ~]$ dig bootstrap.openshift4.example.com +short
192.168.1.116
[chernand@laptop ~]$ dig etcd-0.openshift4.example.com  +short
192.168.1.111
[chernand@laptop ~]$ dig etcd-1.openshift4.example.com  +short
192.168.1.112
[chernand@laptop ~]$ dig etcd-2.openshift4.example.com  +short
192.168.1.113
```

Also, it's important to set up reverse DNS for these entries as well (since you're using DHCP, this is particularly important).

```
[chernand@laptop ~]$ dig -x 192.168.1.111 +short
master1.openshift4.example.com.
[chernand@laptop ~]$ dig -x 192.168.1.112 +short
master2.openshift4.example.com.
[chernand@laptop ~]$ dig -x 192.168.1.113 +short
master3.openshift4.example.com.
[chernand@laptop ~]$ dig -x 192.168.1.114 +short
worker1.openshift4.example.com.
[chernand@laptop ~]$ dig -x 192.168.1.115 +short
worker2.openshift4.example.com.
[chernand@laptop ~]$ dig -x 192.168.1.116 +short
bootstrap.openshift4.example.com.
```

The DNS lookup for the API endpoints also needs to be in place. OpenShift 4 expects `api.$CLUSTERDOMAIN` and `api-int.$CLUSTERDOMAIN` to be configured, they can both be set to the same IP address - which will be the IP of the Load Balancer.

```
[chernand@laptop ~]$ dig api.openshift4.example.com +short
192.168.1.110
[chernand@laptop ~]$ dig api-int.openshift4.example.com +short
192.168.1.110
```

A wildcard DNS entry needs to be in place for the OpenShift 4 ingress router, which is also a load balanced endpoint.

```
[chernand@laptop ~]$ dig *.apps.openshift4.example.com +short
192.168.1.110
```

In addition to the mentioned entries, you'll also need to add SRV records. These records are needed for the masters to find the etcd servers. This needs to be in the form of `_etcd-server-ssl._tcp.$CLUSTERDOMMAIN` in your DNS server.

```
[chernand@laptop ~]$ dig _etcd-server-ssl._tcp.openshift4.example.com SRV +short
0 10 2380 etcd-0.openshift4.example.com.
0 10 2380 etcd-1.openshift4.example.com.
0 10 2380 etcd-2.openshift4.example.com.
```

Please review the [official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-dns-user-infra_installing-vsphere) to read more about the prerequisites for DNS before installing.

## DHCP

The certificates OpenShift configures during installation is  for communication between all the components of OpenShift, and is tied to the IP address and dns name of the Red Hat Enterprise Linux CoreOS (RHCOS) nodes.

Therefore it's important to have DHCP with address reservation in place. You can do this with MAC Address filtering for the IP reservation. When creating your VMs, you'll need to take note of the MAC address assigned in order to configure your DHCP server for IP reservation.

## Load Balancer

You will need a load balancer to frontend the APIs, both internal and external, and the OpenShift router. Although Red Hat has no official recommendation to which load balancer to use, one that supports SNI is necessary (most load balancers do this today). 

You will need to configure port `6443` and `22623` to point to the bootstrap and master nodes. The below example is using HAProxy (**NOTE** that it must be TCP sockets to allow SSL passthrough):

```
frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance source
    mode tcp
    server btstrap 192.168.1.116:6443 check
    server master1 192.168.1.111:6443 check
    server master2 192.168.1.112:6443 check
    server master3 192.168.1.113:6443 check
    
frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
    server btstrap 192.168.1.116:22623 check
    server master1 192.168.1.111:22623 check
    server master2 192.168.1.112:22623 check
    server master3 192.168.1.113:22623 check
```

You will also need to configure `80` and `443` to point to the worker nodes. The HAProxy configuration is below (keeping in mind that we're using TCP sockets):

```
frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server worker1 192.168.1.114:80 check
    server worker2 192.168.1.115:80 check
   
frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server worker1 192.168.1.114:443 check
    server worker2 192.168.1.115:443 check
```

More information about load balancer configuration (and general networking guidelines) can be found in the [official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-network-user-infra_installing-vsphere).

## Web server
A web server is needed in order to hold the ignition configurations to install RHCOS. Any webserver will work as long as the webserver can be reached by the bootstrap, master, and worker nodes during installation. I will be using Apache. I created a directory specifically for the ignition files:

```
[root@webserver ~]# mkdir -p /var/www/html/ignition/
```

## Artifacts
You will need to obtain the installation artifacts by visiting [try.openshift.com](https://try.openshift.com), there you can login and click on "VMware vSphere" to get the installation artifacts. You will need:

* [OpenShift4 Client Tools](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/)
* [OpenShift4 OVA](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.2/latest/)
* Pull Secret

You will need to put the client and the installer in your `$PATH`, in my example; I put mine in my `/usr/local/bin` path.

```
[chernand@laptop ~]$ which oc
/usr/local/bin/oc
[chernand@laptop ~]$ which kubectl
/usr/local/bin/kubectl
[chernand@laptop ~]$ which openshift-install
/usr/local/bin/openshift-install
```

I've also downloaded my pullsecret as `pull-secret.json` and saved it under a `~/.openshift` directory I created.

```
[chernand@laptop ~]$ file ~/.openshift/pull-secret.json
/home/chernand/.openshift/pull-secret.json: JSON data
```

A ssh-key is needed. This is used in order to login to the RHCOS server if you ever need to debug the system.

```
[chernand@laptop ~]$ file ~/.ssh/id_rsa.pub
/home/chernand/.ssh/id_rsa.pub: OpenSSH RSA public key
```

For more information about ssh and RHCOS, visit the [official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#ssh-agent-using_installing-vsphere) site.

## Installation

Once you have the prerequisites in place, you're ready to begin the installation. The current installation of OpenShift 4 on vSphere must be done in stages. I will go over each stage step by step.

### vSphere Preparations

In your vSphere web UI, after you login, navigate to "VMs and Templates" (it's the icon that looks like a piece of paper). From here right click on your datacenter and select `New Folder` → `New VM and Template Folder`. Name this new folder the name of your cluster id. In my case, I named mine `openshift4`. You should have a new folder that looks like this.

![newfolder](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/1.newfolder.png)

