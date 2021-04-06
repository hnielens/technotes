# How to use cpd-cli to install/patch/upgrade CPD services on ROKS

This document discusses how you can install extra services on top of an **existing** Cloud Pak for Data deployment that was originally installed using the [IBM Cloud Cloud Pak for Data installer](https://cloud.ibm.com/catalog/content/ibm-cp-datacore-6825cc5d-dbf8-4ba2-ad98-690e6f221701-global).

If you need a walkthrough on how to install RedHat Openshift and Cloud Pak for Data completely from scratch, please follow this guide: [Install CPD 3.x on Red Hat OpenShift 4.x on VMWare or Bare Metal] (https://github.com/IBM-ICP4D/cloud-pak-ocp-4#install-red-hat-openshift-4x-on-vmware-or-bare-metal)

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
You can use the default Terminal app (or your preferred terminal, e.g. iTerm) to run `cpd-cli`. Consider the Linux VM alternative if you encouner latency or bandwidth issues.

#### Use your Windows 10
Configure and set up your Windows 10 machine for linux to run `cpd-cli`. Consider the Linux VM alternative if you encouner latency or bandwidth issues. [Use this walkthrough to set up linux on your Windows 10 machine](https://docs.microsoft.com/en-us/windows/wsl/install-win10).

 ### Option 2: Provision a linux VM (or reuse an existing one)
 **DRAFT** [How to provision a linux VM in IBM Cloud Classic Infrastructure](provision_linux_vm_in_ibm_cloud_classic_infra.md#how-to-provision-a-basic-linux-vm-in-ibm-cloud-classic-infrastructure).

### Can I use IBM Cloud Shell?
IBM Cloud Shell is an in-browser shell that you can quickly start from the https://cloud.ibm.com homepage.

The advantages of using IBM Cloud Shell:

- You can choose to run the Shell in the specific region in which you have set up your cluster.
- The IBM Cloud CLI is already installed, and you are logged in when you open the shell.
- The necessary Openshift CLI (`oc`) and Kubernetes CLI (`kubectl`) are already installed, you will just need to log in with a temp bearer token.

The **dis**advantages of using IBM Cloud shell:

- Your environment is recycled after an hour of inactivity, so you will need to upload/install the `cpd-cli` and set your entitlement key each time after the environment is recycled.
- Long running scripts are killed.
- You can use the IBM Cloud Shell for maximum 50 hours per week.

So... as the IBM Cloud Shell environment will be reset after one hour of inactivity (includes long running scripts) and long running scripts will be killed, IBM Cloud Shell is **not a good** option if you want use it as a bastion node for installation.

It **is** a good option for simple administrative tasks.

Starting the shell is easy: goto https://cloud.ibm.com and click the start shell button in the top menu:

![Start IBM Cloud Shell](images/ibm_cloud_shell.png)

This is the IBM Cloud Shell welcome:

![IBM Cloud Shell Welcom](images/ibm_cloud_shell_welcome.png)

Notice that you can upload/download stuff from/to your pc to/from the shell. We will do that in a minute.<br>
![](images/upload-download.png)

## Download and install the prerequisite CLI
You need to one time install;

- The IBM Cloud CLI collection
- The RedHat Openshift CLI (oc)



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
oc login --token={{your_bearer_token}} --server={{your_server_naem}}
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

The documentation describes the process for Datstage Enterprise Plus (and for the other services) in detail: https://www.ibm.com/docs/en/cloud-paks/cp-data/3.5.0?topic=plus-setting-up-cluster-datastage-enterprise

### Prepare

First declare the service you want to install. The service name is mentioned in the installation documentation for the specific service. For Datastage Enterprise Plus the service name is `ds`.

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
--transfer-image-to=image-registry-openshift-image-registry.$myclusterdomain/zen \
--target-registry-username=$(oc whoami) \
--target-registry-password=$(oc whoami -t) \
--insecure-skip-tls-verify \
--cluster-pull-prefix=image-registry.openshift-image-registry.svc:5000/zen \
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
--transfer-image-to=image-registry-openshift-image-registry.$myclusterdomain/zen \
--target-registry-username=$(oc whoami) \
--target-registry-password=$(oc whoami -t) \
--insecure-skip-tls-verify \
--cluster-pull-prefix=image-registry.openshift-image-registry.svc:5000/zen \
--latest-dependency \
--accept-all-licenses
```

### Patch

TBD
