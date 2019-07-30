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

