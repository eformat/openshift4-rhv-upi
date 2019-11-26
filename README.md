# Provisioning OpenShift 4.2 on RHV Using Baremetal UPI

This repository contains a set of playbooks to help facilitate the deployment of OpenShift 4.2 on RHV.

_NOTE:_ This a forked repository from the original [sa-ne/openshift4-rhv-upi](https://github.com/sa-ne/openshift4-rhv-upi)


## Background
The purpose of these playbooks is to help setup OpenShift 4 UPI on RHV. 

## Specific Automations

* Deployment of RHCOS on RHV
* Creation of all SRV, A and PTR records in IdM
* Deployment of httpd Server for Installation Artifacts and Logic
* Deployment of HAProxy and Applicable Configuration
* Configuring of PXE server
* Deployment of dhcpd and Applicable Static IP Assignment
* Ordered Starting (i.e. installation) of VMs

## Requirements

To leverage the automation in this guide you need to bring the following:

* RHV Environment (tested on 4.3)
* IdM Server with DNS Enabled
 * Must have Proper Forward/Reverse Zones Configured
* RHEL 7 Server which will act as a Bastion hosts, which is also a:
  * Web Server
  * Load Balancer
  * DHCP Server
  * PXE Server
 * Only Repository Requirement is `rhel-7-server-rpms`

### Naming Convention

All hostnames must follow the following format:

* bootstrap.\<base domain\>
* master0.\<base domain\>
* masterX.\<base domain\>
* worker0.\<base domain\>
* workerX.\<base domain\>

## Noted UPI Installation Issues

* Bootstrap SSL Certificate is only valid for 24 hours
* etcd/master naming convention conforms to 0 based index (i.e. master0, master1, master2...not master1, master2, master3)

# Installing

Read through the [Installing on baremetal](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.2/html-single/installing/index#installing-bare-metal) installation documentation before proceeding.

## Clone this Repository

Find a good working directory and clone this repository using the following command:

```console
$ git clone https://github.com/tsailiming/openshift4-rhv-upi.git
```
You will need to install Ansible. This has been tested with Ansible 2.9.1.

## Create DNS Zones in IdM

Login to your IdM server and make sure a reverse zone is configured for your subnet. My lab has a subnet of `192.168.100.0` so the corresponding reverse zone is called `100.168.192.in-addr.arpa.`. Make sure a forward zone is configured as well. It should be whatever is defined in the `dns_base_domain` variable in your Ansible inventory file (`ocp.ltsai.com` in this example).

## Creating Inventory File for Ansible

An example inventory file is included for Ansible (`inventory-example.yml`). Use this file as a baseline. Make sure to configure the appropriate number of master/worker nodes for your deployment.

|Variable|Description|
|:---|:---|
|dns_base\_domain|The base DNS domain. Not to be confused with the base domain in the UPI instructions. Our base\_domain variable in this case is `<cluster_name>`.`<base_domain>`|
|rhcos_version| RHEL CoreOS version |
|rhcos_mirror_path | RHEL CoreOS mirror path | 
|dhcp\_server\_dns\_servers|DNS server assigned by DHCP server|
|dhcp\_server\_gateway|Gateway assigned by DHCP server|
|dhcp\_server\_subnet\_mask|Subnet mask assigned by DHCP server|
|dhcp\_server\_subnet|IP Subnet used to configure dhcpd.conf|
|dhcp\_server\_ntp\_server | NTP server assigned by DHCP server|
|load\_balancer\_ip|This IP address of your load balancer (the server that HAProxy will be installed on)|
|httpd_port|Http server port to serve the ignition files | 
|installation_directory|The directory that you will be using with `openshift-install` command for generating ignition files|
| restricted_network | If you are installing OpenShift in a disconnected environment |
| ocp_local_registry | Your local registry for a disconnected environment | 
| ocp_local_repo | Your local repository |
| ocp_cert | Your local registry certificate |

Your pull secret can be obtained from the [OpenShift start page](https://cloud.redhat.com/openshift/install/metal/user-provisioned).

For the individual node configuration, be sure to update the hosts in the `pg` hostgroup. Several parameters will need to be changed for _each_ host including `ip`, `storage_domain` and `network`. You can also specify `mac_address` for each of the VMs in its `network` section (if you don't, VMs will obtain their MAC address from cluster's MAC pool automatically). Match up your RHV environment with the inventory file.

Under the `webserver` and `loadbalancer` group include the FQDN of each host. Also make sure you configure the `httpd_port` variable for the web server host. In this example, the web server that will serve up installation artifacts and load balancer (HAProxy) are the same host.

## Creating an Ansible Vault

In the directory that contains your cloned copy of this git repo, create an Ansible vault called vault.yml as follows:

```console
$ ansible-vault create vault.yml
```

The vault requires the following variables. Adjust the values to suit your environment.

```yaml
---
rhv_hostname: "rhvm.example.com"
rhv_username: "admin@internal"
rhv_password: "changeme"
rhv_cluster: "Default"
ipa_hostname: "idm.example.com"
ipa_username: "admin"
ipa_password: "changeme"
ocp_pull_scecret: "pull secret from cloud.redhat.com as a string"
ocp_ssh_key: "SSH public key"
```

## Download the OpenShift Installer

The OpenShift Installer releases are stored [here](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/). Find the installer, right click on the "Download Now" button and select copy link. Then pull the installer using curl (be sure to quote the URL) as shown (linux client used as example):

```console
$ curl -o openshift-install-linux-4.1.0.tar.gz https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-install-linux-4.1.0.tar.gz
```

Extract the archive and continue.

##  Ignition Configs

This is auto generated by [openshift-installer](roles/openshift-installer) role.

## Deploying OpenShift 4.2 on RHV with Ansible

To kick off the installation, simply run the provision.yml playbook as follows:

```console
$ ansible-playbook -u root -i inventory.yml --ask-vault-pass provision.yml
```

## Finishing the Deployment

Once the VMs boot RHCOS will be installed and nodes will automatically start configuring themselves. From this point we are essentially following the Baremetal UPI instructions starting with [Creating the Cluster](https://access.redhat.com/documentation/en-us/openshift_container_platform/4.1/html-single/installing/index#installation-installing-bare-metal_installing-bare-metal).

Run the following command to ensure the bootstrap process completes (be sure to adjust the `--dir` flag with your working directory):

```console
$ openshift-install wait-for bootstrap-complete --dir /tmp/rhv-upi/
INFO Waiting up to 30m0s for the Kubernetes API at https://api.ocp.ltsai.com:6443... 
INFO API v1.14.6+dc8862c up                       
INFO Waiting up to 30m0s for bootstrapping to complete... 
INFO It is now safe to remove the bootstrap resources 
```

From the boostrap node:
```console
[core@bootstrap ~]$ journalctl -b -f -u bootkube.service
Nov 26 10:56:52 bootstrap.ocp.ltsai.com bootkube.sh[3494]: Tearing down temporary bootstrap control plane...
Nov 26 10:56:53 bootstrap.ocp.ltsai.com bootkube.sh[3494]: bootkube.service complete
```

Once this openshift-install command completes successfully, login to the load balancer and comment out the references to the bootstrap server in `/etc/haproxy/haproxy.cfg`. There should be two references, one in the backend configuration `backend_22623` and one in the backend configuration `backend_6443`. Alternativaly, you can just run this utility playbook to achieve the same:

```console
ansible-playbook -i inventory.yml -u root bootstrap_cleanup.yml
```

Or you can simply power off the bootstrap vm. Regardless, the bootstrap services would have been shutdown. 

### Kubeconfig

To use `oc` command, copy the kubeconfig to your local system.

```console
cp /rhv-upi/auth/kubeconfig  ~/.kube/config
```

### Wait for installation to complete

Next, wait for the installation to complete

To complete the installation, you will need to define the [registry storage](https://docs.openshift.com/container-platform/4.2/installing/installing_bare_metal/installing-bare-metal.html#installation-registry-storage-config_installing-bare-metal), otherwise the image-registry cluster operator will be pending. 

This configure `emptydir` which is *NOT* for production use. 
``` console
$ oc patch configs.imageregistry.operator.openshift.io cluster --type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```


If you run this command before the Image Registry Operator initializes its components, the oc patch command fails with the following error:

`Error from server (NotFound): configs.imageregistry.operator.openshift.io "cluster" not found`



```console
$ openshift-install wait-for install-complete --dir /tmp/rhv-upi/
INFO Waiting up to 30m0s for the cluster at https://api.ocp.ltsai.com:6443 to initialize... 
INFO Waiting up to 10m0s for the openshift-console route to be created... 
INFO Install complete!                            
INFO To access the cluster as the system:admin user when using 'oc', run 'export KUBECONFIG=/tmp/rhv-upi/auth/kubeconfig' 
INFO Access the OpenShift web-console here: https://console-openshift-console.apps.ocp.ltsai.com 
INFO Login to the console with user: kubeadmin, password: XXX

```

Check cluster operators:
```console
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE
authentication                             4.2.4     True        False         False      17m
cloud-credential                           4.2.4     True        False         False      39m
cluster-autoscaler                         4.2.4     True        False         False      23m
console                                    4.2.4     True        False         False      20m
dns                                        4.2.4     True        False         False      39m
image-registry                             4.2.4     True        False         False      12m
ingress                                    4.2.4     True        False         False      27m
insights                                   4.2.4     True        False         False      39m
kube-apiserver                             4.2.4     True        False         False      36m
kube-controller-manager                    4.2.4     True        False         False      36m
kube-scheduler                             4.2.4     True        False         False      35m
machine-api                                4.2.4     True        False         False      40m
machine-config                             4.2.4     True        False         False      30m
marketplace                                4.2.4     True        False         False      27m
monitoring                                 4.2.4     True        False         False      10s
network                                    4.2.4     True        False         False      37m
node-tuning                                4.2.4     True        False         False      32m
openshift-apiserver                        4.2.4     True        False         False      31m
openshift-controller-manager               4.2.4     True        False         False      35m
openshift-samples                          4.2.4     True        False         False      18m
operator-lifecycle-manager                 4.2.4     True        False         False      37m
operator-lifecycle-manager-catalog         4.2.4     True        False         False      37m
operator-lifecycle-manager-packageserver   4.2.4     True        False         False      50s
service-ca                                 4.2.4     True        False         False      39m
service-catalog-apiserver                  4.2.4     True        False         False      33m
service-catalog-controller-manager         4.2.4     True        False         False      33m
storage                                    4.2.4     True        False         False      28m
```

# Retiring

Playbooks are also provided to remove VMs from RHV. To do this, run the retirement playbook as follows:

```console
$ ansible-playbook -i inventory.yml --ask-vault-pass retire.yml
```

# HAProxy Stats

Accessible from http://<haproxy>:5000/haproxy_stats
