# OpenShift 4.1 Bare Metal Install Quickstart

In this blog we will go over how to get you quickly up and running with an OpenShift 4.1 Bare Metal install on pre-existing infrastructure. Although this quickstart focuses on the bare metal installer, this can also be seen as a "manual" way to install OpenShift 4.1. Moreover, this is also applicable to installing to any platform which doesn’t have the ability to provide [ignition](https://coreos.com/ignition/docs/latest/) pre-boot. For more information about using this generic approach to install on untested platforms, please see this [knowledge base article](https://access.redhat.com/articles/4207611).

## Introduction

Openshift 4 introduces a new way of installing the platform that is automated, reliable, and repeatable. Based on the [Kubernetes Cluster-API SIG](https://github.com/kubernetes-sigs/cluster-api), Red Hat has developed an OpenShift installer for full stack automated deployments. This means that the [installer](https://github.com/openshift/installer) not only installs OpenShift, but it installs (and manages) the entire infrastructure as well, from DNS all the way down the stack to the VM. This provides a fully integrated system that can resize automatically with the needs of your workload. Currently, full stack automated deployment is supported on AWS.

For pre-existing infrastructure deployments is if you have existing infrastructure that you would like to use for the purposes of running OpenShift 4. Most are familiar with this method as it was the default (and only) way to install OpenShift 3. Currently guides for Pre-existing infrastructure installs are on [AWS](https://docs.openshift.com/container-platform/4.1/installing/installing_aws_user_infra/installing-aws-user-infra.html), VMWare [vSphere](https://docs.openshift.com/container-platform/4.1/installing/installing_vsphere/installing-vsphere.html), and [bare metal](https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html). The latter being the "catch all", since you can use the bare metal method for non-tested platforms.

I will be going over installing OpenShift 4 Bare Metal, on a pre-existing infrastructure along with the prerequisites. However, as already stated, you can use this method for other infrastructure, for example VMs running on Red Hat Virtualization.

## Prerequisites

It’s important that you get familiar with the prerequisites by reading the [official documentation](https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html) for OpenShift. There you can find more details about the prerequisites and what it entails. I have broken up the prerequisites into sections and have marked those that are optional.

### DNS

Proper DNS setup is imperative for a functioning OpenShift cluster. DNS is used for name resolution (A records), certificate generation (PTR records), and service discovery (SRV records). Keep in mind that OpenShift 4 has a concept of a "clusterid" that will be incorporated into your clusters DNS records. Your DNS records will all have `<clusterid>.<basedomain>` in them. In other words, your "clusterid" will end up being part of your FQDN. Read the [official documentation](https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html#installation-dns-user-infra_installing-bare-metal) for more information.

#### Forward DNS Records

Create forward DNS records for your bootstrap, master, and worker nodes. Also, you’ll need to create entries for both `api` and `api-int` and point them to their respective load balancers (**NOTE** both of those entries can point to the same load balancer). You will also need to create a wildcard DNS entry pointing to the load balancer. This entry is used by the OpenShift router. Here is a sample using `bind` with `ocp4` as the `<clusterid>`.

```
; The api and api-inf can point to the IP of the same load balancer
api.ocp4            IN      A       192.168.1.5
api-int.ocp4        IN      A       192.168.1.5
;
; The wildcard points to the load balancer
*.apps.ocp4        IN      A       192.168.1.5
;
; Create entry for the bootstrap host
bootstrap.ocp4        IN      A       192.168.1.96
;
; Create entries for the master hosts
master0.ocp4        IN      A       192.168.1.97
master1.ocp4        IN      A       192.168.1.98
master2.ocp4        IN      A       192.168.1.99
;
; Create entries for the worker hosts
worker0.ocp4        IN      A       192.168.1.11
worker1.ocp4        IN      A       192.168.1.7
```

An example of a DNS zonefile with forward records can be found [here](https://github.com/openshift-tigerteam/guides/blob/master/ocp4/ocp4-zonefile.db).

#### Reverse DNS Records

Create reverse DNS records for your bootstrap, master, workers nodes, api, and api-int. The reverse records are important because that is how RHEL CoreOS sets the hostname for all the nodes. Furthermore, these PTR records are used in order to generate the various certificates OpenShift needs to operate. The following is an example using `example.com` as the `<basedomain>` and using `ocp4` as the `<clusterid>`. Again, this was done using `bind`.

```
; syntax is "last octet" and the host must have fqdn with trailing dot
;
97        IN      PTR     master0.ocp4.example.com.
98          IN      PTR     master1.ocp4.example.com.
99          IN      PTR     master2.ocp4.example.com.
;
96          IN      PTR     bootstrap.ocp4.example.com.
;
5           IN      PTR     api.ocp4.ocp4.example.com.
5           IN      PTR     api-int.ocp4.ocp4.example.com.
;
11          IN      PTR     worker0.ocp4.example.com.
7           IN      PTR     worker1.ocp4.example.com.
;
```

An example of a DNS zonefile with reverse records can be found [here](https://github.com/openshift-tigerteam/guides/blob/master/ocp4/ocp4-reverse.db).

#### DNS Records for ETCD

Two record types need to be created for ETCD. The forward record needs to point to the IPs of the masters (CNAMEs are fine as well). Also the names need to be `etcd-<index>` where `<index>` is a number starting at 0. An example will be `etcd-0`, `etcd-1`, and `etcd-2`. You will also need to create SRV records pointing to the various `etcd-<index>` entries. You’ll need to set these records with a priority 0, weight 10 and port 2380. Below is an example using `example.com` as the `<basedomain>` and using `ocp4` as the `<clusterid>`.

```
; The ETCd cluster lives on the masters...so point these to the IP of the masters
etcd-0.ocp4             IN      A       192.168.1.97
etcd-1.ocp4             IN      A       192.168.1.98
etcd-2.ocp4             IN      A       192.168.1.99
;
; The SRV records point to FQDN of etcd...note the trailing dot at the end...
_etcd-server-ssl._tcp.ocp4      IN      SRV     0 10 2380 etcd-0.ocp4.example.com.
_etcd-server-ssl._tcp.ocp4      IN      SRV     0 10 2380 etcd-1.ocp4.example.com.
_etcd-server-ssl._tcp.ocp4      IN      SRV     0 10 2380 etcd-2.ocp4.example.com.
;
```

An example of these entries can be found in the [example zonefile](https://github.com/openshift-tigerteam/guides/blob/master/ocp4/ocp4-zonefile.db#L37-L46).

### Load Balancer

You will need a load balancer to frontend the APIs, both internal and external, and the OpenShift router. Although Red Hat has no official recommendation to which Load Balancer to use, one that supports `SNI` is necessary (most load balancers do this today). 

You will need to configure Port `6443` and `22623` to point to the bootstrap and master nodes. The below example is using HAProxy (**NOTE** that it must be TCP sockets to allow SSL passthrough)

```
frontend openshift-api-server
    bind *:6443
    default_backend openshift-api-server
    mode tcp
    option tcplog

backend openshift-api-server
    balance source
    mode tcp
    server bootstrap 192.168.1.96:6443 check
    server master0   192.168.1.97:6443 check
    server master1   192.168.1.98:6443 check
    server master2   192.168.1.99:6443 check
    
frontend machine-config-server
    bind *:22623
    default_backend machine-config-server
    mode tcp
    option tcplog

backend machine-config-server
    balance source
    mode tcp
    server bootstrap 192.168.1.96:22623 check
    server master0   192.168.1.97:22623 check
    server master1   192.168.1.98:22623 check
    server master2   192.168.1.99:22623 check
```

You will also need to configure `80` and `443` to point to the worker nodes. The HAProxy configuration is below (keeping in mind that we’re using TCP sockets).

```
frontend ingress-http
    bind *:80
    default_backend ingress-http
    mode tcp
    option tcplog

backend ingress-http
    balance source
    mode tcp
    server worker0 192.168.1.11:80 check
    server worker1 192.168.1.7:80 check
   
frontend ingress-https
    bind *:443
    default_backend ingress-https
    mode tcp
    option tcplog

backend ingress-https
    balance source
    mode tcp
    server worker0 192.168.1.11:443 check
    server worker1 192.168.1.7:443 check
```

A full example of an `haproxy.cfg` file can be found [here](https://github.com/openshift-tigerteam/guides/blob/master/ocp4/ocp4-haproxy.cfg).

### Webserver

A webserver is needed in order to hold the ignition configurations and installation images for when you install RHEL CoreOS. Any webserver will work as long as the webserver can be reached by the bootstrap, master, and worker nodes during installation. I will be using Apache. Download either the metal-bios or the uefi-metal-bios file, depending on what your servers need, from [here](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.1/4.1.0/). For example, this is how I downloaded the metal-bios file to my webserver.

```
mkdir -p /var/www/html/{ignition,install}
cd /var/www/html/install
curl -J -L -O https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.1/latest/rhcos-4.1.0-x86_64-metal-bios.raw.gz
```

### Setup DHCP (Optional if doing static ips)

It is recommended to use the DHCP server to manage the node’s IP addresses for the cluster long-term. Ensure that the DHCP server is configured to provide persistent IP addresses and host names to the cluster machines. Using DHCP with IP reservation ensures the IPs won’t change on reboots. For a sample configuration; please see this [dhcpd.conf](https://github.com/openshift-tigerteam/guides/blob/master/ocp4/ocp4-dhcpd.conf) file.

### Reconciling Prerequisites

If you plan on installing OpenShift 4 in a "lab" environment (either on bare metal or using VMs); you might want to take a look at the ["Helper Node" github page](https://github.com/christianh814/ocp4-upi-helpernode#ocp4-upi-helper-node-playbook). The "Helper Node" ansible playbook sets up an "all-in-one" node with all the aforementioned prerequisites. This playbook has two modes: "standard" and "static ips".

Take a look at the [quickstart](https://github.com/christianh814/ocp4-upi-helpernode/blob/master/quickstart.md#helper-node-quickstart-install) to see if it might be of use. These steps are written for Libvirt, but the playbook is agnostic. So you can run it on your BareMetal environm

## Installation
