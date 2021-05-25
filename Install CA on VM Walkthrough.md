# Set up an IBM Cloud CentOS 8 VM for CA 11.2 walkthrough

## Order a server

- Have both a private and public address
- Private network policy (allow_all, allow_outbound, allow_ssh), Public Network Policy (allow_outbound)

## Prepare CentOS base

### Install ibmcloud cli
- Get and install available updates
```
dnf update
dnf upgrade
```
- Get and install the ibmcloud cli
```
curl -sL https://ibm.biz/idt-installer | bash
```

### Set up Cloud Object Storage plugin
**Note** that you only need this step when you have uploaded the Cognos software to a COS service
https://cloud.ibm.com/docs/cloud-object-storage-cli-plugin?topic=cloud-object-storage-cli-ic-cos-cli
- Log in to ibmcloud
```
ibmcloud login -sso
```
- Install the cos plugin
```
ibmcloud plugin install cloud-object-storage
```
- Configure cos with the bucket's crn (found in the bucket properties)
```
ibmcloud cos config crn
```
- Configure the hmac authentication using credentials (access_key_id/secret_access_key) generated for the cos service
```
ibmcloud cos config hmac
```
- Make sure hmac is the default active authentication method
```
ibmcloud cos config auth
```
- Configure where downloaded files will land (your default download folder)
```
ibmcloud cos config ddl
```


### Set up and test X11 forwarding
Note that in this section we only make sure the 64bit X11 config is working correctly. We will set up the 32bit side of things later in the section where we are going to install Cognos components with the 32bit installer. This document might help for configuring x11.
https://www.businessnewsdaily.com/11035-how-to-use-x11-forwarding.html

- Make sure e.g. xauth is installed
```
dnf update
dnf install xauth
```
- Optionally test whether your X-term works with xeyes. First make sure you ssh-ed in into your server with the -Y handle and you did not get specific errors. You might need to add and enable the epel and CentOS powertools software repositories before you can install the x-tools...
```
dnf update
dnf install xeyes
xeyes
```

- Check the `/etc/ssh/sshd_config` file

## Set up Apache Directory Server
- Install java
`dnf install java`
- Get the latest download link for the 64bit binary installer for Apache DS
http://directory.apache.org/apacheds/download/download-linux-bin.html
- Download and run the binary installer (in a well chosen directory)
`wget "use_latest_download_URL"`
`chmod 755 ./installer_file_name.bin`
`./installer_file_name.bin`
- For good measure chown the apacheds dirs BEFORE starting the service and maybe have wrapper.conf point (absolute ref) to your java executable
`chown apacheds:apacheds /var/lib/apacheds_version_you_installed/*`
`chown apacheds:apacheds /opt/apacheds_version_you_installed/*`
- start Apache Directory Server with user apacheds
`/etc/init.d/apacheds_version_you_installed-default start`
- Check if you can connect to the Directory Server using Directory Studio and if not maybe you need to repair the default instance
`/etc/init.d/apacheds_version_you_installed-default repair`
- Change the password for uid=admin,ou=system
- Remove the installer --so the bin file you downloaded earlier-- to keep your system neat
- Add an organizational unit called cognos under the `dc=example,dc=com` domain
- Add groups (`groupOfUniqueUsers`) and users (`inetOrgPerson`) as needed

## Deploy Db2 Developer-C Edition container
- If not yet done, register for Docker Hub
https://hub.docker.com/
- Make sure to also login to docker hub on the server before you start with retrieval, configuration and run of the container:
`docker login`
- On Docker Hub Look for Db2 Developer-C, add to basket and check out
- In your _My Content_ (in your account) look for Db2 Developer-C and _Setup_ for instructions on how to get and deploy the Db2 container.

### Db2 Developer Edition v11.5 (current)
- Pull the docker image from Docker Hub:
`docker pull ibmcom/db2`
- Make sure you create a Db2 data folder on your hostname for persisting the db data, e.g.:
`mkdir /root/db2data`
- Here is an example command to build and run the container
```
docker run -itd \
           --name my-db2 \
           --privileged=true \
           --restart=unless-stopped \
           -p 50000:50000 \
           -p 55000:55000 \
           -e LICENSE=accept \
           -e DB2INST1_PASSWORD={{choose_pw_for_db2inst1}} \
           -e SAMPLEDB=false \
           -v /root/db2data:/database \
           ibmcom/db2
```
**Note**

The db2inst1 password will expire as per the specific security configuration in the docker container's os. When the password has expired you will need to connect to the container to change the password.

## Install and Configure Cognos Analytics
### Get the software and install
- Make sure you check the prerequisite libraries/tools before you start the installation: https://www.ibm.com/software/reports/compatibility/clarity. Otherwise you might bump into this kind of error when trying to run the installer:
```
[root@ca112 downloads]# ./ca_instl_lnxi38664_3.2.29.bin
Preparing to install
Extracting the JRE from the installer archive...
Unpacking the JRE...
Extracting the installation resources from the installer archive...
Configuring the installer for this system's environment...

Launching installer...


Graphical installers are not supported by the VM. The console mode will be used instead...
```
- Make sure you have the installation files on your system, e.g. by getting them from your cloud object storage bucket
`ibmcloud login --sso`
`ibmcloud cos download -bucket bucket_name_here -key file_name_here`
- Install Cognos Analytics using the installer and the payload zip
- Do not forget to put the Db2 JDBC driver files in the `../drivers` folder, you can find them in the Db2 container.

```
docker exec -ti my-db2 bash -c "cp /database/config/db2inst1/sqllib/java/db2jcc* /database/."
```

```
cp /root/db2data/db2jcc* /opt/ibm/cognos/analytics/drivers/.
rm /root/db2data/db2jcc*
```

### Configure Cognos Analytics
Use cogconfig.sh to configure the Cognos Analytics deployment
```
/opt/ibm/cognos/analytics/bin64/cogconfig.sh
```
- Change hostname and full qualified domain name mentions to the host ip
- Configure the content store to point to point to a database running in the Db2 container
- Generate DDL for the database you configured in the previous bullet
- Make sure to switch the _Report Server execution mode_ parameter to _64-bit_ (under _Environment_), otherwise you will need to get 32-bit libraries installed on CentOS.
- Save the configuration but do not start Cognos yet
- Create the cognos repository db in the containered Db2 instance by using the ddl script you create earlier:
```
cp /opt/ibm/cognos/analytics/configuration/schemas/content/db2/createDb.sql /root/db2data/.
docker exec -ti my-db2 bash -c "su - db2inst1"
```
on the container cmd line run:
```
db2 -tvmf /database/createDb.sql
```
- Clean up:
```
rm /root/db2data/createDb.sql
```
- Now use Cognos Configuration to test the repository db and try and start Cognos
- Configure a new generic ldap authentication namespace for the Apache Directory Server instance you installed and set up earlier


## Set up SFTP and SFTP users
This is a good base article:
https://www.vultr.com/docs/setup-sftp-only-user-accounts-on-centos-7

### Set up SFTP for exchanging import/export of deployments

- Create a group for the sftp users e.g. _sftpusers_
`groupadd sftpusers`
- Edit the `/etc/ssh/sshd_config` file for use with `internal-sftp`
`cp /etc/ssh/sshd_config /etc/ssh/sshd_config._backup_suffix`
`vi /etc/ssh/sshd_config`
- Find the line:
`Subsystem sftp /usr/libexec/openssh/sftp-server`
and replace it with (or comment it and add):
`Subsystem sftp internal-sftp`
- At the bottom of the file add following directives
`Match Group sftpusers`
`X11Forwarding no`
`AllowTcpForwarding no`
`ChrootDirectory %h`
`ForceCommand internal-sftp`
Notice that we set the ChrootDirectory to match the home directory of each respective SFTP user (`%h`)
- Restart the sshd service:
`systemctl restart sshd`

### Set up SFTP users (do below for every sftp user)

- Create each sftp user, add them to the _sftpusers_ group, and make sure they can not login on the server using `ssh`
`useradd sftp_user -g sftpusers -s /usr/sbin/nologin`
- Change the user pw:
`passwd sftpuser`
- Make sure the root of the home directory --which is the ChrootDirectory we defined earlier-- is not accessible by the user:
`chown -R root:root /home/sftpuser`
- Make sure all hidden files in the root directory of the sftpuser are read/execute only:
`chmod -R 555 /home/sftpuser`
- Make a _deployment_ directory in the sftpuser's home directory and make sure the user owns it:
`mkdir /home/sftpuser/deployment`
`chown sftpuser. /home/sftpuser/deployment`
-Mount the CA deployment directory on the ftpuser's deployment directory:
`mount -o bind /opt/ibm/cognos/analytics/deployment /home/ftpuser/deployment`

### Make the CA deployment directory accessible to the SFTP users

- Change the ownership and permissions of the folder:
```
chown root:sftpusers /opt/ibm/cognos/analytics/deployment
chmod 755 /opt/ibm/cognos/analytics/deployment
```

## Configuring Cognos Jupyter Notebooks

After using the Cognos installer to deploy the Jupyter notebooks software you might want to do some extra configuration before deploying the Juypyter containers.

### Hostname versus IP address

By default the config will use the hostname --on the fly using `$(hostname)`-- when starting up the socket service on which the Jupyter service will become available. Sometimes, when the hostname is not in a Names Server (DNS) communications might fail the hostname can not be resolved (e.g. from within a container) or resolves to a public IP address. In that case you might want to use the fixed private ip address instead. Change the config:


`cd /opt/ibm/cognos/jupyter/dist/scripts/unix`
`cp config.conf config.conf._backup_suffix`
`vi config.conf`


```
# HOST_NAME is the fully qualified domain name of this host. The default hostnane environment variable may or may not
# relect the fully qualified domain name, but most linux distributions do.
# this should reflect the external domain URL which must be used to reach this server (which may be different if an external proxy is used
# for ingress to this host) . ensure HOST_NAME is set to a value which will allow routing to this host.
#HOST_NAME=$(hostname)
HOST_NAME=10.185.164.11
HOST_PORT=800
```

### Extra packages/libraries to include when building the Jupyter Server Instance container

When you want to include extra packages/libraries you had best adapt the docker build file before starting the container. This can always be done afterwards but then you need to clean up the previously built containers first. Adapt the Jupyter Server Instance build file:

```
cd /opt/ibm/cognos/jupyter/dist/scripts
cp Dockerfile_server_instance Dockerfile_server_instance._backup_suffix
vi Dockerfile_server_instance
```

Add `--upgrade` handle for pip:
```
# copy additional_packages & install if not empty
# HNI: added the --upgrade option so latest packages are installed
COPY --chown=ca_user:users additional_packages.txt .

RUN if [ -s additional_packages.txt ]; then \
        pip install --user --upgrade -r additional_packages.txt; \
```
Add some essential os packages

```
# HNI: install additional os packages as root
USER root
RUN apt-get update && apt-get install -y \
    build-essential \
    graphviz
```
