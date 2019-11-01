# OpenShift 4.2  vSphere Install with Static IPs

In the previous blog I went over how to install OpenShift 4.2 on VMware vSphere 6.7 using DHCP. Using DHCP with address reservation using MAC address filtering is a common way to ensure network configurations are set and consistent on Red Hat Enterprise Linux CoreOS (RHCOS). 

Many environments would rather use static IP configuration to achieve the same consistent network configurations. With the release of OpenShift 4.2, we have now added the support to configure network configurations (and persist them across reboots) in the preboot ignition phase for RHCOS.

In this blog we are going to go over how to install OpenShift 4 on VMware vSphere using static IPs.

## Environment Overview

Like the previous blog, I am using vSphere version 6.7.0 and ESXi version 6.7.0 Update 3. I will be following the [official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html) where, you can read more information about prerequisites including the need to set up DNS, Load Balancer, Artifacts, and other ancillary services/items.

## Prerequisites

It’s always important that you get familiar with the prerequisites by reading the [official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-infrastructure-user-infra_installing-vsphere) before you install. There you can find more details about the prerequisites and what it entails. I will go over the [prerequisites](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-infrastructure-user-infra_installing-vsphere) at a high level, and will link examples where possible.

### vSphere Credentials

I will be using my administrative credentials for vSphere. I will also be passing these credentials to the OpenShift 4 installer and, by extension, to the OpenShift cluster. It’s not a requirement to do so, and installing without the credentials will effectively turn your installation into a “Bare Metal UPI” installation. Most notably, you’ll lose the ability to dynamically create VDMKs for your applications at install time.

### DNS

Like any OpenShift 4 installation, the first consideration you need to take into account when setting up DNS for OpenShift 4 is the “cluster id”. The “cluster id” becomes part of your cluster’s FQDN. For example; with a cluster id of “openshift4” and my domain of “example.com”, the cluster domain (i.e. the FQDN) for my cluster is `openshift4.example.com`

DNS entries are created using the `$CLUSTERID.$DOMAIN` cluster domain FQDN nomenclature. All DNS lookups will be based on this cluster domain. Using my example cluster domain, `openshift4.example.com`, I have the following DNS entries set up in my environment. Note that the etcd servers are pointed to the IP of the masters, and they are in the form of `etcd-$INDEX`.

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

In addition to the mentioned entries, you’ll also need to add SRV records. These records are needed for the masters to find the etcd servers. This needs to be in the form of `_etcd-server-ssl._tcp.$CLUSTERDOMMAIN` in your DNS server.

```
[chernand@laptop ~]$ dig _etcd-server-ssl._tcp.openshift4.example.com SRV +short
0 10 2380 etcd-0.openshift4.example.com.
0 10 2380 etcd-1.openshift4.example.com.
0 10 2380 etcd-2.openshift4.example.com.
```

Please review the [official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-dns-user-infra_installing-vsphere) to read more about the prerequisites for DNS before installing.

### Load Balancer

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

More information about load balancer configuration (and general networking guidelines) can be found in the [official documentation page](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-network-user-infra_installing-vsphere).

### Web server

A web server is needed in order to hold the ignition configurations and install artifacts needed to install RHCOS. Any webserver will work as long as the webserver can be reached by the bootstrap, master, and worker nodes during installation. I will be using Apache, and created a directory specifically for the ignition files.

```
[chernand@laptop ~]$ mkdir -p /var/www/html/{ignition,install}
```

### Artifacts

You will need to obtain the installation artifacts by visiting [try.openshift.com](https://try.openshift.com), there you can login and click on “VMware vSphere” to get the installation artifacts. You will need:

* [OpenShift4 Client Tools](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/)
* [OpenShift4 OVA](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.2/latest/)
* Pull Secret
* You will also need the [RHCOS ISO](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.2/latest/) and the [OpenShift4 Metal BIOS](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/4.2/latest/) file

You will need to put the client and the installer in your `$PATH`, in my example I put mine in my `/usr/local/bin` path

```
[chernand@laptop ~]$ which oc
/usr/local/bin/oc
[chernand@laptop ~]$ which kubectl
/usr/local/bin/kubectl
[chernand@laptop ~]$ which openshift-install
/usr/local/bin/openshift-install
```

I’ve also downloaded my pullsecret as `pull-secret.json` and saved it under a `~/.openshift` directory I created.

```
[chernand@laptop ~]$ file ~/.openshift/pull-secret.json
/home/chernand/.openshift/pull-secret.json: JSON data
```

An ssh-key is needed. This is used in order to login to the RHCOS server if you ever need to debug the system.

```
[chernand@laptop ~]$ file ~/.ssh/id_rsa.pub
/home/chernand/.ssh/id_rsa.pub: OpenSSH RSA public key
```

For more information about ssh and RHCOS, you can read that at the [official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#ssh-agent-using_installing-vsphere) site.

## Installation

Once the prerequisites are place, you’re ready to begin the installation. The current installation of OpenShift 4 on vSphere must be done in stages. I will go over each stage step by step.

### vSphere Preparations

For static IP configurations, you will need to upload the ISO into a datastore. On the vSphere WebUI, click on the “Storage” navigation button (it looks like stacked cylinders), and click on the datastore you’d like to upload the ISO to. In my example, I have a datastore specifically for ISOs:

![datastoreiso](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-static-ips/images/16.datastoreiso.png)


