# How to use cpd-cli to install/patch/upgrade CPD services on ROKS
## Make sure you have gathered the necessary info before you begin
You will need the following stuff, so make sure you collect it upfront, so you can copy/paste it quickly when and where needed further down the line:

- Your entitlement key for IBM container software. This is locacted in your My IBM profile: https://myibm.ibm.com/products-services/containerlibrary
- Your cluster's ingress subdomain name. You can copy this from your cluster details in IBM Cloud.
- The name of your CPD project/namespace, e.g. `zen`.
- The apikey of your CPD instance, you can find that in the profile of your CPD administrator account.
- The name of the storageclass you choose when you installed CPD using the IBM Cloud installer (for endurance type storage the storage class is called `ibmc-file-gold-gid`).

## Choose an install node (aka bastion node)
Although it is possible to install from Mac I would advise installing from a linux machine. Moreover, the closer that linux machine is to your cluster the better as the cpd-cli will download and transfer the necessary software images for the services you are going to install/update/patch.

So... unless you can go and sit in the data center and plug in your Mac in the network, you will need an alternative. You can either provision/reuse a linux VM in the same IBM Cloud data center/region or choose the quick and dirty way: use IBM Cloud Shell.

IBM Cloud Shell is an in-browser shell that you can quickly start from the https://cloud.ibm.com homepage.

The advantages of using IBM Cloud Shell:

- You can choose to have the Shell to start in the specific region in which you have set up your cluster.
- The IBM Cloud CLI is already installed, and you are logged in when you open the shell.
- The Openshift CLI (oc) is already installed.

The disadvantages of using IBM Cloud shell:

- Your environment is recycled after 2 hours of inactivity, so you will need to upload/install the cpd-cli and set your entitlement key each time after the environment is recycled.
- You can use the IBM Cloud Shell for maximum 50 hours per week.

To keep things simple we assume here that you will use the IBM Cloud Shell.

Starting the shell is easy: goto https://cloud.ibm.com and click the start shell button in the top menu:

![Start IBM Cloud Shell](images/ibm_cloud_shell.png)

This is the IBM Cloud Shell welcome:

![IBM Cloud Shell Welcom](images/ibm_cloud_shell_welcome.png)
