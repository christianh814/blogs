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

Next, import the OVA by right clicking the folder and select "Deploy OVF Template".  Make sure you select the folder for this cluster as the destination, then click next. 

![ovaimport](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/2.destinationfolder-importova.png)

Go ahead and select an ESXi host for the destination compute resource, after that is done it will display the OVA information.

![verifydetails](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/3.importdetails.png)

After this is displayed go ahead and click "Next". This will display the storage destination dialog. Choose the appropriate destination datastore, and set the virtual disk format to "Thin" if you wish.

![storagedestdiag](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/4.thinstoragedestination.png)

The next screen asks to select a destination virtual network, I am using the default "VM Network", so I accept the defaults.

![destinationvirtnet](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/5.vmnetworkdefaults.png)

After clicking “Next”, the “Customize Template” section comes up. We won’t be customizing the template here, so leave these blank and click “Next”.

![cutimzetemplatepage](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/6.customizetemplatepage.png)

The next page will give you an overview with the title “Ready To Complete”, click “Next” to finish the importing of the OVA.

![readytocompleteova](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/7.readytocompleteova.png)

The OVA template should be in your cluster folder. It should look something like this:

![importedova](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/8.importedova.png)

Next, right click the imported OVA and select “Edit Settings”. The “Edit Settings” dialog box appears and should look like this:

![editova](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/13.editova.png)

Click on the “VM Options” and expand the “Advanced” section. Set “Latency Sensitivity” to “High”.

![latencysens](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/14.latencysensitivity.png)

Next, click on “Edit Configuration…” next to the “Configuration Parameters” section. You will add the following:

* `guestinfo.ignition.config.data` and set the value to `chageme`
* `guestinfo.ignition.config.data.encoding` set this value to `base64`
* `disk.EnableUUID` set this value to `TRUE`

It should look something like this:

![configparam](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/15.configparameters.png)

Click “OK” to go back to the “VM Options” page and then click “OK” again to save these settings.

Now, right click the imported OVA and select `Clone` → `Clone to Template`. The “Clone Virtual Machine To Template” wizard starts. It'll ask you to name this template and where to store it. I will be creating the master template first so I will name it “master-template” and save it in my “openshift4” folder. You can also convert this OVA to a template as well.

![startmastertemp](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/9.startmastertemplatewizard.png)

On the next screen select a compute destination. Choose one of your ESXi hosts, and click “Next”.

On the following screen,  select the appropriate datastore for your environment (make sure you select “Thin” as the disk format) and click “Next”.

![masterdest](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/10.mastertemplatestoragedestinationthin.png)

Next, there will be the “Ready to complete” page, giving you an overview.

![readycompmatemp](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/11.redytocompletemastertemplatecreation.png)

Click on “Finish” to finish the creation of the master template. 

Now you will do the same steps **_AGAIN_**, except you'll be creating a template for the workers/bootstrap nodes. I named this template “worker-bootstrap-template”. When you are finished, you should have something like this.

![finishtemplates](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/12.finishtemplatecreation.png)

> **NOTE**: You can also have just one OpenShift 4 template and adjust the CPU/RAM if desired. If you plan on churning lots of nodes for multiple deployments then multiple templates may make more sense.

### Generate Install Configuration 

Now that you’ve prepped vSphere for installation. You can go ahead and generate the `install-config.yaml` file. This file tells OpenShift about the environment that you’re going to install. Before you create this file you’ll need an installation directory to store all your artifacts. You can name this directory whatever you like; I’m going to name mine `openshift4`.

```
[chernand@laptop ~]$ mkdir openshift4
[chernand@laptop ~]$ cd openshift4/
```

I’m going to export some environment variables that will make the creation of the `install-config.yaml` file easier. Please substitute your configuration where applicable.

```
[chernand@laptop openshift4]$ export DOMAIN=example.com
[chernand@laptop openshift4]$ export CLUSTERID=openshift4
[chernand@laptop openshift4]$ export VCENTER_SERVER=vsphere.example.com
[chernand@laptop openshift4]$ export VCENTER_USER="administrator@vsphere.local"
[chernand@laptop openshift4]$ export VCENTER_PASS='supersecretpassword'
[chernand@laptop openshift4]$ export VCENTER_DC=DC1
[chernand@laptop openshift4]$ export VCENTER_DS=datastore1
[chernand@laptop openshift4]$ export PULL_SECRET=$(< ~/.openshift/pull-secret.json)
[chernand@laptop openshift4]$ export OCP_SSH_KEY=$(< ~/.ssh/id_rsa.pub)
```

Once you’ve exported those, go ahead and create the `install-config.yaml` file in the openshift4 directory by running the following:

```
[chernand@laptop openshift4]$ cat <<EOF > install-config.yaml
apiVersion: v1
baseDomain: ${DOMAIN}
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: ${CLUSTERID}
networking:
  clusterNetworks:
  - cidr: 10.254.0.0/16
    hostPrefix: 24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  vsphere:
    vcenter: ${VCENTER_SERVER}
    username: ${VCENTER_USER}
    password: ${VCENTER_PASS}
    datacenter: ${VCENTER_DC}
    defaultDatastore: ${VCENTER_DS}
pullSecret: '${PULL_SECRET}'
sshKey: '${OCP_SSH_KEY}'
EOF
```

I’m going over the options at a high level:

* `baseDomain` - This is the domain of your environment.
* `metadata.name` - This is your clusterid
  * Note: this makes all FQDNS in the `openshift4.example.com` domain.
* `platform.vsphere` - This is your vSphere specific configuration. This is optional and you can find a “standard” install config example in [the docs](https://docs.openshift.com/container-platform/4.2/installing/installing_bare_metal/installing-bare-metal.html#installation-bare-metal-config-yaml_installing-bare-metal).
* `pullSecret` - This pull secret can be obtained by going to [cloud.redhat.com](https://cloud.redhat.com/openshift/install)
  * Note: I saved mine as ~/.openshift/pull-secret.json
* `sshKey` - This is your public SSH key (e.g. id_rsa.pub)

> **NOTE**: The OpenShift installer removes this file during the install process, so you may want to keep a copy of it somewhere.

### Create Ignition Files

The next step in the process is to create the installer manifest files using the `openshift-install` command. Keep in mind that you need to be in the install directory you created (in my case that’s the `openshift4` directory).

```
[chernand@laptop openshift4]$ openshift-install create manifests
INFO Consuming "Install Config" from target directory
WARNING Making control-plane schedulable by setting MastersSchedulable to true for Scheduler cluster settings
```

Note that the installer tells you the the masters are schedulable. For this installation, we need to set the masters to not schedulable.

```
[chernand@laptop openshift4]$ sed -i 's/mastersSchedulable: true/mastersSchedulable: false/g' manifests/cluster-scheduler-02-config.yml
[chernand@laptop openshift4]$ cat manifests/cluster-scheduler-02-config.yml
apiVersion: config.openshift.io/v1
kind: Scheduler
metadata:
  creationTimestamp: null
  name: cluster
spec:
  mastersSchedulable: false
  policy:
    name: ""
status: {}
```

To find out more about why you can’t run workloads on OpenShift 4.2 on the control plane, please refer to the [official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-user-infra-generate-k8s-manifest-ignition_installing-vsphere).

Once the manifests are created, you can go ahead and create the ignition files for installation.

```
[chernand@laptop openshift4]$ openshift-install create ignition-configs
INFO Consuming "Master Machines" from target directory
INFO Consuming "Openshift Manifests" from target directory
INFO Consuming "Worker Machines" from target directory
INFO Consuming "Common Manifests" from target directory
```

Next, create an `append-bootstrap.ign` ignition file. This file will tell RHCOS where to download the `bootstrap.ign` file to configure itself for the OpenShift cluster.

```
[chernand@laptop openshift4]$ cat <<EOF > append-bootstrap.ign
{
  "ignition": {
    "config": {
      "append": [
        {
          "source": "http://192.168.1.110:8080/ignition/bootstrap.ign",
          "verification": {}
        }
      ]
    },
    "timeouts": {},
    "version": "2.1.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {}
}
EOF
```

Next, copy over the bootstrap.ign file over to this webserver.

```
[chernand@laptop openshift4]$ scp bootstrap.ign root@192.168.1.110:/var/www/html/ignition/
```

We’ll need the `base64` encoding for each of the ignition files we’re going to pass to VSphere when we create the VMs. Do this by encoding the files and putting the result in a file for later use.

```
[chernand@laptop openshift4]$ for i in append-bootstrap master worker
do
  base64 -w0 < $i.ign > $i.64
done
[chernand@laptop openshift4]$ ls -1 *.64
append-bootstrap.64
master.64
worker.64
```

You are now ready to create the VMs.

### Creating the Virtual Machines

Log back into the vSphere webui to create the virtual machines from the templates you created. Navigate to “VMs and Templates” (the icon that looks like a sheet of paper); and then right click the “worker-bootstrap-template” and select `New VM From this Template...` This brings up the “Deploy From Template” wizard. Name this VM “bootstrap” and make sure it’s in the openshift4 folder.

![createbootstrapfromtemp](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/18.creatingbootstrapfromtemplate.png)

After you click next, select one of your ESXi hosts in your cluster as a destination compute resource and click “Next”. On the next page, it’ll ask you to select a datastore for this VM. Select the appropriate store for your cluster and make sure you thin provision the disk.

![bootstrapthinvm](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/19.bootstrapthinprovision.png)

After clicking next, check off “Customize this virtual machine's hardware” on the “Select clone options” page.

![customhardware](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/20.customizehardware.png)

After you click next, it’ll bring up the “Customize hardware” page. For the bootstrap we are setting 4 CPUs, 8GB of RAM, 120GB of HD space, and I will also set the custom MAC address for my DHCP server. It should look something like this:

![bootstraphardwaresettings](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/21.bootstraphardwaresettings.png)

On that same screen click on “VM Options” and expand the “Advanced” menu. Scroll down to the “Configuration Parameters” section and click on “Edit Configuration...”. This will bring up the parameters menu. There, you will change the `guestinfo.ignition.config.data` value from `changeme` to the contents of your `append-bootstrap.64` file.

```
[chernand@laptop openshift4]$ cat append-bootstrap.64
```

Here is a screenshot of my configuration:

![btsrpigndata](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/22.boostrapignitiondata.png)

Click “OK” and then click “Next” on the “Customize Hardware” screen. This will bring you to the overview page.

![btstrpoverviewpage](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/23.bootstrapovervivewpage.png)

Click “Finish”, to create your bootstrap VM.

You need to perform these steps at least 5 more times (3 times for the masters and 2 more times for the workers). Use the following table to configure your servers, which is based on the resource requirements listed on [the official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#minimum-resource-requirements_installing-vsphere).


| MACHINE | vCPU | RAM | STORAGE | guestinfo.ignition.config.data |
| ----------- | ----------- | ----------- | ----------- | ----------- |
| master | 4 | 16 GB | 120 GB | Output of: cat openshift4/master.64 |
| worker | 2 | 8 GB | 120 GB | Output of: cat openshift4/worker.64 |

Once you’ve created your 3 masters and 2 workers, you should have 6 VMs in total. One for the bootstrap, three for the masters, and two for the workers. 

![ocp4fullvms](https://raw.githubusercontent.com/christianh814/blogs/master/docs/openshift-4.2-vsphere-dhcp/images/24.ocp4fullvms.png)

Now boot the VMs. It doesn’t matter which order you boot them in, but I booted mine in the following order:

* Bootstrap
* Masters
* Workers

### Bootstrap Process

Back on the installation host, you can now finish the bootstrap complete process for the OpenShift installer.

```
[chernand@laptop openshift4]$ openshift-install wait-for bootstrap-complete --log-level debug
DEBUG OpenShift Installer v4.2.0                   
DEBUG Built from commit 90ccb37ac1f85ae811c50a29f9bb7e779c5045fb
INFO Waiting up to 30m0s for the Kubernetes API at https://api.openshift4.example.com:6443...
INFO API v1.14.6+2e5ed54 up                       
INFO Waiting up to 30m0s for bootstrapping to complete...
DEBUG Bootstrap status: complete                   
INFO It is now safe to remove the bootstrap resources
```

Once you see this message; you can safely delete the bootstrap VM and continue with the installation.

### Finishing Install

Once the bootstrap process is complete, the cluster is actually up and running; but not in a state where it’s ready to receive workloads. To finish the install process first export the `KUBECONFIG` environment variable.

```
[chernand@laptop openshift4]$ export KUBECONFIG=~/openshift4/auth/kubeconfig
```

You can now access the API. You first need to check if there are any CSRs that are pending for any of the nodes. You can do this by running `oc get csr`, this will list all the CSRs for your cluster.

```
[chernand@laptop openshift4]$ oc get csr
NAME        AGE     REQUESTOR                                                                   CONDITION
csr-4hn7m   6m36s   system:node:master3.openshift4.example.com                                  Approved,Issued
csr-4p6jz   7m8s    system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-6gvgh   6m21s   system:node:worker2.openshift4.example.com                                  Approved,Issued
csr-8q4q4   6m20s   system:node:master1.openshift4.example.com                                  Approved,Issued
csr-b5b8g   6m36s   system:node:master2.openshift4.example.com                                  Approved,Issued
csr-dc2vr   6m41s   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-fwprs   6m22s   system:node:worker1.openshift4.example.com                                  Approved,Issued
csr-k6vfk   6m40s   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-l97ww   6m42s   system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-nm9hr   7m8s    system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
```

You can approve any pending CSRs by running the following command (please read more about certificates in the [official documentation](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#installation-approve-csrs_installing-vsphere)):

```
[chernand@laptop openshift4]$ oc get csr --no-headers | awk '{print $1}' | xargs oc adm certificate approve
```

After you’ve verified that all CSRs are approved, you should be able to see your nodes.

```
[chernand@laptop openshift4]$ oc get nodes
NAME                             STATUS   ROLES    AGE     VERSION
master1.openshift4.example.com   Ready    master   9m55s   v1.14.6+c07e432da
master2.openshift4.example.com   Ready    master   10m     v1.14.6+c07e432da
master3.openshift4.example.com   Ready    master   10m     v1.14.6+c07e432da
worker1.openshift4.example.com   Ready    worker   9m56s   v1.14.6+c07e432da
worker2.openshift4.example.com   Ready    worker   9m55s   v1.14.6+c07e432da
```

In order to complete the installation, you need to add storage to the image registry. For testing clusters, you can set this to `emptyDir` (for more permanent storage, please see the official doc [for more information](https://docs.openshift.com/container-platform/4.2/installing/installing_vsphere/installing-vsphere.html#registry-configuring-storage-vsphere_installing-vsphere)).

```
[chernand@laptop openshift4]$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

At this point, you can now finish the installation process.

```
[chernand@laptop openshift4]$ openshift-install wait-for install-complete
INFO Waiting up to 30m0s for the cluster at https://api.openshift4.example.com:6443 to initialize...
INFO Waiting up to 10m0s for the openshift-console route to be created...
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/chernand/openshift4/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.openshift4.example.com
INFO Login to the console with user: kubeadmin, password: STeaa-LjEB3-fjNzm-2jUFA
```

Once you’ve seen this message, the install is complete and the cluster is ready to use. If you provided your vSphere credentials, you’ll have a `storageclass` set.

```
[chernand@laptop openshift4]$ oc get sc
NAME              PROVISIONER                    AGE
thin (default)   kubernetes.io/vsphere-volume   13m
```
You can use this `storageclass` to dynamically create VDMKs for your applications.

If you didn’t provide your vSphere credentials, you can consult the [VMware Documentation site](https://vmware.github.io/vsphere-storage-for-kubernetes/documentation/index.html) for how to set up storage integration with Kubernetes.

## Conclusion
In this blog we went over how to install OpenShift 4 on VMware using the UPI method using DHCP. We also displayed the vSphere integration that allows OpenShift to create VDMKs for the applications. In my next blog, I will be going over how to install using static IPs. So, stay tuned!

Red Hat OpenShift Container Platform and VMware vSphere are a great combination for running an enterprise container platform on a virtual infrastructure. For the last several years our joint customers have successfully deployed OpenShift on vSphere for their production ready applications.

We invite you to [try OpenShift 4](https://try.openshift.com) on VMware, and run enterprise ready Kubernetes on vSphere today!
