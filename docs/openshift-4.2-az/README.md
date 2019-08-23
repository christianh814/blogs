# OpenShift 4.2 on Azure Preview

In this blog we will be showing a video on how to get OpenShift 4 installed on Azure using the full stack automated method. This method differs from the pre-existing infrastructure method, as the full stack automation gets your from zero to a full OpenShift deployment, creating all the required infrastructure components automatically. Currently, **__installing OpenShift 4 on Azure is under tech preview. It won’t be supported until the GA release of OpenShift 4.2__**. This blog is meant for those who want to get a preview on what’s coming. Detailed instructions are below if you wish to follow along!

[![OpenShift 4.2 on Azure](http://img.youtube.com/vi/c4UbHQ3nR-8/0.jpg)](http://www.youtube.com/watch?v=c4UbHQ3nR-8 "OpenShift 4.2 on Azure")

## Prerequisites
It’s important that you get familiar with the general prerequisites by looking at the [official documentation](https://docs.openshift.com/container-platform/4.1/welcome/index.html) for OpenShift. There you can find specific details about the requirements and installation details for either full-stack automated or for pre-existing infrastructure deployments. I have broken up the prerequisites into sections and have marked those that are optional.

### DNS
You will need to have a DNS domain already controlled by Azure. The OpenShift installer will configure DNS resolution (internal and external) for the cluster. This can be done by [buying a domain on Azure](https://docs.microsoft.com/en-us/azure/app-service/manage-custom-dns-buy-domain) or [delegating a domain (or subdomain) to Azure](https://docs.microsoft.com/en-us/azure/dns/dns-delegate-domain-azure-dns). In either case, make sure the domain is set ahead of time.

During the install, you will be providing a `$CLUSTERID`, this ID will be used as part of the FQDN of the components created for your cluster. In other words, the ID will become part of your DNS name. For example, a domain of `example.com` and a `$CLUSTERID` of `ocp4` will yield an OpenShift domain of `ocp4.example.com` for your cluster.

Choose wisely.

### Azure CLI Tools (Optional)
It’s useful to install the Azure `az` CLI client. Although you can do all of what you need for Azure from the web UI, it’s helpful to have the CLI tool installed for debugging or streamlining the setup process.

Once you’ve [installed the Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest), you will need to login to set up the cli for access. Be sure to visit the [Getting Started page](https://docs.microsoft.com/en-us/cli/azure/get-started-with-azure-cli?view=azure-cli-latest) for more information. Once set up, verify that you have a connection to your account with the following:

```
az account show
```

The output should look something like this

```
{
  "environmentName": "AzureCloud",
  "id": "VVVVVVVV-VVVV-VVVV-VVVV-VVVVVVVVVVVV",
  "isDefault": true,
  "name": "Microsoft Azure Account",
  "state": "Enabled",
  "tenantId": "WWWWWWWW-WWWW-WWWW-WWWW-WWWWWWWWWWWW",
  "user": {
    "name": "user@email.com",
    "type": "user"
  }
}
```

Again, you don’t need the Azure CLI tool; but it does help.
 
### OpenShift CLI Tools

In order to install and interact with OpenShift, you will need to download some CLI tools. These can be found by going to [try.openshift.com](https://try.openshift.com) and logging in with your Red Hat Customer Portal credentials. Click on Azure (note that it’s only Developer Preview currently). You will need to download the following:

* [The OpenShift Installer](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest)
* [The OpenShift CLI tools](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.2.0-0.nightly-2019-06-03-135056/) (includes oc and kubectl)
* Download or copy your [pull secret](https://cloud.redhat.com/openshift/install/azure/installer-provisioned)

You may need the ["dev preview" binaries](https://mirror.openshift.com/pub/openshift-v4/clients/ocp-dev-preview/latest/) instead, as dev previews are always being updated. Always consult [try.openshit.com](https://try.openshift.com) for details.

## Install

In this section I will be going over the installing of OpenShift 4.2 dev preview on Azure, with the assumption you have an Azure account and that you did all the prerequisites. I will be installing the following:

* Installer sets up 3 Master nodes, 3 Worker nodes, and 1 bootstrap node.
* I will be using `az.redhatworkshops.io` domain as an example.
* I will be using `openshift4` as my clusterid.
* I am doing the install from a Linux host.

### Creating a Service Principal

A Service Principal needs to be created for the installer to use. Service Principal can be thought of as a "robot" account for automation on Azure. More information about Service Principals can be found using the [Microsoft Docs](https://docs.microsoft.com/en-us/powershell/azure/create-azure-service-principal-azureps?view=azps-2.5.0). To create a service principal; run the following command:

```
az ad sp create-for-rbac --name chernand-azure-video-sp
```

When successful, it should output the information about the service principal. Save this information somewhere as the installer will need it to do the install. The information should look something like this.

```
{
  "appId": "ZZZZZZZZ-ZZZZ-ZZZZ-ZZZZ-ZZZZZZZZZZZZ",
  "displayName": "chernand-azure-video-sp",
  "name": "http://chernand-azure-video-sp",
  "password": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX",
  "tenant": "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY"
}
```

Next, you need to give the service principal the right roles in order to properly install OpenShift.  The service principal needs to have at least Contributor and User Access Administrator roles assigned in your subscription. 

```
az role assignment create --assignee \ 
ZZZZZZZZ-ZZZZ-ZZZZ-ZZZZ-ZZZZZZZZZZZZ --role Contributor
az role assignment create --assignee \
ZZZZZZZZ-ZZZZ-ZZZZ-ZZZZ-ZZZZZZZZZZZZ --role "User Access Administrator"
```

> NOTE: The UUID passed to `--assignee` is the `appId` in the output when you created the service principal.

In order to properly mint credentials for components in the cluster, your service principal needs to request for the following application permissions before you can deploy OpenShift on Azure: **Azure Active Directory Graph -> Application.ReadWrite.OwnedBy**

You can request permissions using the Azure portal or the Azure CLI. (You can read more about Azure Active Directory Permissions at the [Microsoft Azure website](https://blogs.msdn.microsoft.com/aaddevsup/2018/06/06/guid-table-for-windows-azure-active-directory-permissions/))

```
az ad app permission add --id ZZZZZZZZ-ZZZZ-ZZZZ-ZZZZ-ZZZZZZZZZZZZ \
--api 00000002-0000-0000-c000-000000000000 \
--api-permissions 824c81eb-e3f8-4ee6-8f6d-de7f50d565b7=Role
```

> NOTE: The `Application.ReadWrite.OwnedBy` permission is granted to the application only after it is provided an "Admin Consent" by the tenant administrator. If you are the tenant administrator, you can run the following to grant this permission.

```
az ad app permission grant --id \
ZZZZZZZZ-ZZZZ-ZZZZ-ZZZZ-ZZZZZZZZZZZZ \
--api 00000002-0000-0000-c000-000000000000
```

You will also need your Subscription ID; you can get this by running the following.

```
az account list --output table
```

### Installing OpenShift

It’s best to create a working directory when creating a cluster. This directory will hold all the install artifacts, including the initial `kubeadmin` account.

```
mkdir ~/ocp4
```

Run the `openshift-install create install-config` command specifying this working directory.  This creates the initial install config (`install-config.yaml`) and stores it in that directory. You will need information about your service principal you created earlier.

```
$ openshift-install create install-config --dir=~/ocp4
? SSH Public Key /home/chernand/.ssh/azure_rsa.pub
? Platform azure
? azure subscription id 12345678-1234-1234-1234-123456789012
? azure tenant id YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY
? azure service principal client id ZZZZ-ZZZZ-ZZZZ-ZZZZZZZZZZZZ
? azure service principal client secret [? for help] ***********
INFO Saving user credentials to "/home/chernand/.azure/osServicePrincipal.json"
? Region centralus
? Base Domain az.redhatworkshops.io
? Cluster Name openshift4
? Pull Secret [? for help] ****************************
```

Let’s go over the Azure specific options.

* `azure subscription id` - This is your subscription id. This can be obtained by running: `az account list --output table`
* `azure tenant id` - Your tenant id (this was in the output when you created your service principal)
* `azure service principal client id` - This is the appId from the service principal creation output.
* `azure service principal client secret` - This is the password from the service principal creation output.

The `install-config.yaml` file is in the `~/ocp4` working directory. It also creates a `~/.azure/osServicePrincipal.json` file. Inspect these files if you wish.

```
cat ~/ocp4/install-config.yaml
cat ~/.azure/osServicePrincipal.json
```

After you’ve inspected these files; go ahead and install OpenShift.

```
openshift-install create cluster --dir=~/ocp4/
```

When the install is finished, you’ll see the following output.

```
INFO Consuming "Install Config" from target directory
INFO Creating infrastructure resources...         
INFO Waiting up to 30m0s for the Kubernetes API at https://api.openshift4.az.redhatworkshops.io:6443...
INFO API v1.14.0+8e63b6d up                       
INFO Waiting up to 30m0s for bootstrapping to complete...
INFO Destroying the bootstrap resources...        
INFO Waiting up to 30m0s for the cluster at https://api.openshift4.az.redhatworkshops.io:6443 to initialize...
INFO Waiting up to 10m0s for the openshift-console route to be created...
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/home/chernand/ocp4/auth/kubeconfig'
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.openshift4.az.redhatworkshops.io:6443
INFO Login to the console with user: kubeadmin, password: 5char-5char-5char-5char
```

Set the `KUBECONFIG` environment variable to connect to your cluster.

```
export KUBECONFIG=$HOME/ocp4/auth/kubeconfig
```

Verify that your cluster is up and running.

```
$ oc cluster-info
Kubernetes master is running at https://api.openshift4.az.redhatworkshops.io:6443

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```

## Post Install

After your cluster is deployed, you may want to do some additional configuration tasks such as:

* [Configuring authentication and additional users](https://docs.openshift.com/container-platform/4.1/authentication/understanding-identity-provider.html#understanding-identity-provider)
* [Adding additional routes and/or sharding network traffic](https://docs.openshift.com/container-platform/4.1/networking/configuring-ingress-cluster-traffic/configuring-ingress-cluster-traffic-ingress-controller.html)
* [Migrating OpenShift services to specific nodes](https://docs.openshift.com/container-platform/4.1/nodes/scheduling/nodes-scheduler-about.html)
* [Adding additional persistent storage or a dynamic storage provisioner](https://docs.openshift.com/container-platform/4.1/storage/understanding-persistent-storage.html)
* [Adding more nodes to the cluster](https://docs.openshift.com/container-platform/4.1/machine_management/adding-rhel-compute.html)

It’s important to note that the `kubeadmin` user is meant to be a temporary admin user. You should replace this user with a more permanent admin user when you configure authentication.

## Conclusion
In this blog we went over how to install OpenShift 4 on Azure using the full stack automated method. It’s important to note that this method is marked as developer preview, meaning it’s not supported by Red Hat. However, the installer is ready for you to deploy and test for non-production workloads. Please feel free to try it and provide feedback by leaving a comment below or or reach out via the [Customer Portal Discussions](https://access.redhat.com/discussions) page.
