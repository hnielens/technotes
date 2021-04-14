# How to use cpd-cli to install/patch/upgrade CPD services on ROKS

This document discusses how you can install extra services on top of an **existing** Cloud Pak for Data deployment that was originally installed using the [IBM Cloud Cloud Pak for Data installer](https://cloud.ibm.com/catalog/content/ibm-cp-datacore-6825cc5d-dbf8-4ba2-ad98-690e6f221701-global). I discuss installing the automated set up and installation of a ROKS cluster with Cloud Pak for Data [in my first webinar](https://ibm.box.com/s/htztaie5zno3lsbenrjgvkfjpgc93e79). Make sure you are already logged in into Box with your IBM account for this link to work.

If you need a walkthrough on how to install RedHat Openshift and Cloud Pak for Data completely from scratch, please follow this guide: [Install CPD 3.x on Red Hat OpenShift 4.x on VMWare or Bare Metal] (https://github.com/IBM-ICP4D/cloud-pak-ocp-4#install-red-hat-openshift-4x-on-vmware-or-bare-metal).

These are the steps you will need to follow:

- [Make sure you have gathered the necessary info before you begin](#make-sure-you-have-gathered-the-necessary-info-before-you-begin)
- [Choose a "computer" to install from (aka a bastion node)](#choose-a-computer-to-install-from-aka-a-bastion-node)
- [Declare some variables for later use](#declare-some-variables-for-later-use)
- [Log in into your cluster](#log-in-into-your-cluster)
- [Download, install and configure cpd-cli](#download-install-and-configure-cpd-cli)
- [Installing, patching and upgrading new services](#installing-patching-and-upgrading-new-services)

## Make sure you have gathered the necessary info before you begin
You will need the following stuff, so make sure you collect it upfront, so you can copy/paste it quickly when and where you need it further down the line:

- Your entitlement key for IBM container software. This is located in your My IBM profile: https://myibm.ibm.com/products-services/containerlibrary We will need to add this to the repo.yaml as described below in "[Edit and save the repo.yaml](#edit-and-save-the-repoyaml)".
- Your cluster's ingress subdomain name. You can copy this from your cluster details in IBM Cloud: https://cloud.ibm.com/kubernetes/clusters.
- The name of your CPD project/namespace, e.g. `zen`.
- The apikey of your CPD instance, you can find that in the profile of your CPD administrator account.
- The name of the storageclass you choose when you installed CPD using the IBM Cloud installer (for endurance type storage the storage class is called `ibmc-file-gold-gid`).
- The homepage base url of your Cloud Pak for Data instance, e.g. https://zen-cpd-zen.mycluster-ams03-873383-bbeafa5fcc6c15e1c7b1c5daea86b416-0000.ams03.containers.appdomain.cloud/

## Choose a "computer" to install from (aka a bastion node)

We will use `cpd-cli` to prepare, download and transfer the necessary software images for the services you are going to install/update/patch. when you choosing your "bastion node", consider that the closer it is located to your cluster, the smoother your experience will probably be.

### Option 1: Use your PC as a bastion
#### Use your Mac
You can use the default Terminal app (or your preferred terminal, e.g. iTerm) to run `cpd-cli`. Consider the Linux VM alternative ([option 2](https://github.com/hnielens/technotes/blob/main/install_update_cpd_roks_svcs_using_cli.md#option-2-provision-a-linux-vm-or-reuse-an-existing-one)) if you encouner latency or bandwidth issues.

#### Use your Windows 10
Configure and set up your Windows 10 machine for linux to run `cpd-cli`. Consider the Linux VM alternative (([option 2](https://github.com/hnielens/technotes/blob/main/install_update_cpd_roks_svcs_using_cli.md#option-2-provision-a-linux-vm-or-reuse-an-existing-one)) if you encouner latency or bandwidth issues. [Use this walkthrough to set up linux on your Windows 10 machine](https://docs.microsoft.com/en-us/windows/wsl/install-win10).

 ### Option 2: Provision a linux VM (or reuse an existing one)
 **DRAFT** [How to provision a linux VM in IBM Cloud Classic Infrastructure](provision_linux_vm_in_ibm_cloud_classic_infra.md#how-to-provision-a-basic-linux-vm-in-ibm-cloud-classic-infrastructure).

### Can I use IBM Cloud Shell?
IBM Cloud Shell is an in-browser shell that you can quickly start from the https://cloud.ibm.com homepage.

As the IBM Cloud Shell environment will be reset after one hour of inactivity and long running scripts will be killed, IBM Cloud Shell is **not a good** option if you want use it as a bastion node for installation.

It **is** a good option for simple administrative tasks.

Here is a seperate technote for the ["how-to" for IBM Cloud Shell](how-to-use-ibm-cloud-shell).

## Download and install the prerequisite CLIs
On your chosen bastion "computer" you will need to one time install following prerequiste CLIs as indicated in the "Access" section in the detail page for your ROKS cluster: https://cloud.ibm.com/kubernetes/clusters.

![ROKS Cluster Access Section](images/roks_cluster_access_section.png)

- The IBM Cloud CLI collection and tools
- The RedHat Openshift CLI (oc)

### Install the IBM Cloud CLI collection and tools
Make or choose a folder where you can download stuff and make sure it is your "present working dirctory".

This command will downlaod and isntall the IBM Cloud CLI collection and tools:
```
curl -sL https://ibm.biz/idt-installer | bash
```

### Install the Redhat Openshift CLI (oc)
Download the Openshift client that matches the version of your cluster. You can find and copy the download link for the specific version via a browser on your PC by following this link (for OpenShift 4.6): https://mirror.openshift.com/pub/openshift-v4/clients/oc/

Copy the url for "your" `oc.tar.gz` (in Chrome: right-click and choose `copy link address` as shown below):

![Download "your" oc.tar.gz](images/download_oc.png)

## Download, install and configure cpd-cli
You need to one time install the cpd-cli on your chosen "bastion node".

### Download the tarball to your pc
Download the `cpd-cli` installer to your PC. As we are working in the IBM Cloud Shell you will need the latest `cpd-cli` enterprise edition version for linux. If the latest version is 3.5.3 then you download `cpd-cli-linux-EE-3.5.3.tgz`.
**Note**
Make sure to keep a copy of this tarball on your PC, as you will probably need to install it more than once. Remember: IBM Cloud Shell will reset after 60 minutes of inactivity.
### Upload the tarball to your shell
Upload the tarball to your IBM Cloud Shell environment using the upload button:</br>
![](images/upload-download.png)
### Extract the tarball
Extract the contents of the tarball, you have uploaded to the Cloud Shell environment:
```
tar xvf cpd-cli-linux-EE-{{cpd-cli_version}}
```
### Edit and save the repo.yaml
We need to add our license entitlement key to the `repo.yaml` file we just extracted:
```
---
fileservers:
  -
    url: "https://raw.github.com/IBM/cloud-pak/master/repo/cpd/3.5"
registry:
  -
    url: cp.icr.io/cp/cpd
    name: base-registry
    namespace: ""
    username: cp
    apikey: <entitlement key>
```
As we will have to do this every time our shell session idles out, you might want to download the `repo.yaml` file to your PC (use the button), add the entitlement key and save a copy for upload when needed.
Upload the `repo.yaml` file from your PC to the shell environment (use the button).
This section discusses `cpd-cli` and `repo.yaml` in the documentation for CPD v3.5: https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=tasks-obtaining-installation-files
### Create and save a cpd-cli profile
Finally, create a `cpd-cli` profile (needed for some actions only). This is also described in detail in the documentation:
https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=installing-creating-cpd-cli-profile
```
./cpd-cli config users set cpd-admin-user --username admin --apikey $apikey
# Note that we use the $apikey value here
./cpd-cli config profiles set cpd-admin-profile --user cpd-admin-user --url $mycpdurl
# Note that we use the $mycpdurl here
```
## Declare some variables for later use
Let's declare some variable=value pairs with some of the stuff you collected in the beginning to make things easier further along the line (the variables are used in further code snippets):

```
export myclusterdomain={{your_cluster_domain_name_here}}
export namespace={{your_cpd_namespace}}
export apikey={{your_api_key_here}}
export storageclass={{your_storage_class}}
export mycpdurl={{your_cpd_url}}
```
## Log in into your cluster
You can copy the oc CLI command for logging in into your cluster from your cluster's openshift web gui.

![Copy CLI Login Command](images/oc_login.png)

Run the login command in the shell:
```
oc login --token={{your_bearer_token}} --server={{your_server_name}}
```
You can set your namespace/project as the default namespace/project to test you are whether you are successfully logged in into your cluster:
```
oc project $namespace # The $-sign indicates that we reference the vars we declared earlier
```

## Installing, patching and upgrading new services

Take some time to read about the general process of installing, upgrading and patching the CPD control plane and services in the documentation:

Installing, upgrading and patching involve a little dance that is the same for each service

- Prepare the cluster to install the service? Be attentive to specific requirments for databases from the Db2 family.
- Install the service
- Check for patches
- Set up instances if applicable

Let's install the Datastage Enterprise Plus service. Note that the code snippets below can be used for any service thanks to the variables we declared in the beginning.

The documentation describes the process for Datstage Enterprise Plus (and for the other services) in detail: https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=plus-setting-up-cluster-datastage-enterprise.

### Prepare

First declare the service you want to install. Technically the software for a service is a number of modules bundled in an **assembly**. The assembly name you need to use is mentioned in the installation documentation for the specific service. A more detailed technical list of all assemblies and modules for CPD 3.x can be found here: https://github.com/IBM/cloud-pak/tree/master/repo/cpd3. The assembly name for Datastage Enterprise Plus is `ds`.

```
export assembly=ds
```

**Note**
A client can choose to buy Datastage Enterprise or Datastage Enterprise Plus. The service name for Datastage Enterprise is `ds-ent`. The service name for Datastage Enterprise Plus is `ds`.

Run the script that will prepare the cluster by doing some checks and adding stuff like new roles:

```
# Note that we use the $assembly and $namespace values here

./cpd-cli adm \
--repo ./repo.yaml \
--assembly $assembly \
--namespace $namespace \
--latest-dependency \
--accept-all-licenses
```

Check the results. You will see that the script was not really executed but just simulated the execution. You can excute it by adding `--apply`.

```
# Note that we use the $assembly and $namespace values here

./cpd-cli adm \
--repo ./repo.yaml \
--assembly $assembly \
--namespace $namespace \
--latest-dependency \
--accept-all-licenses \
--apply
```

### Install

Now we are ready to install Datastage Enterprise Plus:

```
# Note that we use the $assembly, $namespace, $storageclass and $myclusterdomain values here

./cpd-cli install \
--assembly $assembly \
--namespace $namespace \
--repo ./repo.yaml \
--storageclass $storageclass \
--transfer-image-to image-registry-openshift-image-registry.$myclusterdomain/$namespace \
--target-registry-username $(oc whoami) \
--target-registry-password $(oc whoami -t) \
--insecure-skip-tls-verify \
--cluster-pull-prefix image-registry.openshift-image-registry.svc:5000/$namespace \
--latest-dependency \
--accept-all-licenses \
--dry-run
```

Notice that above is only the dry-run. Drop `--dryrun` to start the installation for real:

```
# Note that we use the $assembly, $namespace, $storageclass and $myclusterdomain values here

./cpd-cli install \
--assembly $assembly \
--namespace $namespace \
--repo ./repo.yaml \
--storageclass $storageclass \
--transfer-image-to image-registry-openshift-image-registry.$myclusterdomain/$namespace \
--target-registry-username $(oc whoami) \
--target-registry-password $(oc whoami -t) \
--insecure-skip-tls-verify \
--cluster-pull-prefix image-registry.openshift-image-registry.svc:5000/$namespace \
--latest-dependency \
--accept-all-licenses
```

### Patch

Available patches:
https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=patches-available

./cpd-cli status \
--repo ./repo.yaml \
--namespace $namespace \
--assembly $assembly \
--patches \
--available-updates

./cpd-cli patch \
--repo ./repo.yaml \
--assembly $assembly \
--namespace $namespace \
--patch-name $patchname\
--transfer-image-to image-registry-openshift-image-registry.$myclusterdomain/$namespace \
--cluster-pull-prefix image-registry.openshift-image-registry.svc:5000/$namespace \
--target-registry-username $(oc whoami) \
--target-registry-password $(oc whoami -t) \
--action transfer \
--dry-run
