# OCP4 Deployer

These are helper scripts to automate major parts of deploying an UPI OpenShift 4 cluster into an existing environment.
The purpose of this project is to create a fully functional cluster without duplicating existing core network infrastructure.
These playbooks will not create the necessary DHCP, DNS, or PXE Boot resources.
It is expected that these already exist in your infrastructure.
If you would like to have network infrastructure created in support of your cluster, like in a lab environement, check out [ocp4-upi-helpernode](https://github.com/christianh814/ocp4-upi-helpernode).

This automation was built, largely, from [this blog post](https://blog.openshift.com/openshift-4-bare-metal-install-quickstart/). It is good to read that prior to executing these scripts. These scripts and instructions work in a slightly different order, but are accomplishing the same tasks.

## Populate Inventory File
Update the inventory file to match your desired deployment. 

## Create Servers

Before continuing, it is best to build out the infrastructure. 

All of these servers should exist and should have their associated forward/reverse DNS lookups working. 
All hostnames should be of the form NAME.CLUSTERID.BASEDOMAIN. If your cluster ID is ocp and your base domain is example.com, you may name your bootstrap server bootstrap.ocp.example.com. The only exception to this is that the Data Server need not be in the same zone as the rest of the cluster.

For any server that will be getting RHCOS, leave them powered off for the time being. They will be imaged during the install process.

| Type          | OS    | Min Count | Recomended Count | Memory | Cores | Additional Softare |
|---------------|-------|-----------|------------------|--------|-------|--------------------|
| Bootstrap     | RHCOS | 1         | 1                | 16GB   | 4     | Webserver serving static content from /var/www/html |
| Data Server   | RHEL  | 1         | 1                | 4GB    | 2     | |
| Load Balancer | RHEL  | 1         | 3                | 4GB    | 2     | | 
| Control Plane | RHCOS | 1         | 3                | 16GB   | 4     | |
| Compute       | RHCOS | 2         | 2                | 8GB    | 2     | |

### Bootstrap
This server is used to initialize the OpenShift cluster. It is temporary and not needed after the install is complete.
### Data Server
The Data Server should be an existing web server to use for hosting cluster/CoreOS data. 
Web data should be stored at /var/www/html.
The CoreOS images and ignition files will be published to this server for use by the cluster.
One data server can be used for multiple clusters, if that is what your hearts desires.
### Load Balancer
These are RHEL servers that will be configured to be load balancers for the OpenShift cluster services.
### Control Plane
These are the Kubernetes control plane nodes. They handle scheduling on the cluster.
### Compute
These nodes are the Kubernetes application nodes. Your applications, as well as cluster routing, will run on these nodes.

## Run Installation Preparation
This is done with the *prepare.yml* playbook.

```
ansible-playbook prepare.yml -i inventory
```

This playbook will start by asking all of the questions required to perform the installation.
A build directory will be created in your current directory titled `build.CLUSTERID`.
In addition to all of the cluster build artifacts, an `answers.yaml` file is also created.
This file stores all of the answers provided to the playbooks.
You can use it to rerun the preparation without being required to fill out the form.

```
ansible-playbook prepare.yml -i inventory -e @build.CLUSTERID/answers.yaml
```

This playbook has a few tags for the different plays in the book:
  - `generate`: These tasks will use the OpenShift installer to generate the installlation configuration.
  - `publish`: These tasks will publish the installation configuration to the data server.

## Create the Load Balancers

Just run the playbook for this one. Everything here is automated, but make sure you pass in the answers file from the previous step.

```
ansible-playbook load-balancer.yml -i inventory -e @build.CLUSTERID/answers.yaml
```

## Create Additional DNS Entries

At this point it is assumed that the forward/reverse DNS entries for all of your servers exist.
Now you need the additional DNS entries for your API VIP, etcd cluster, and application routing. These entries will look something like the following:

### Forward Zone
```
; the api points to the load balancer vip
api.ocp.example.com.        IN  A   192.168.10.201
api-int.ocp.example.com.    IN  A   192.168.10.201
;
; the wildcard entry also points to the load balancer vip
*.apps.ocp.example.com.     IN  A   192.168.10.201
;
; the etcd cluster runs on the control plane
; these IPs should match those of the control plane
etcd-0.ocp.example.com.     IN  A   192.168.10.100
etcd-1.ocp.example.com.     IN  A   192.168.10.101
etcd-2.ocp.example.com.     IN  A   192.168.10.102
;
; SRV records for etcd
_etcd-server-ssl._tcp.ocp.example.com.  IN  SRV 0   10  2380    etcd-0.ocp.example.com.
_etcd-server-ssl._tcp.ocp.example.com.  IN  SRV 0   10  2380    etcd-1.ocp.example.com.
_etcd-server-ssl._tcp.ocp.example.com.  IN  SRV 0   10  2380    etcd-2.ocp.example.com.
```
### Reverse Zone
```
; reverse for the load balancer vip
201.10.168.192.in-addr.arpa IN  PTR api.ocp.example.com.
```

## Setup PXE Boot Environment

During the host preparation, the data server was setup with most of the files required for PXE booting to RHCOS. You'll need to configure your existing PXE boot server to boot the CoreOS machines into PXELinux and provide a PXELinux template that looks similar to the following:

```
DEFAULT pxeboot
TIMEOUT 20
PROMPT 0
LABEL pxeboot
    KERNEL http://ocp-hub.lab.rmkra.us/rhcos-4.2.0-x86_64-installer-kernel
    APPEND ip=dhcp rd.neednet=1 initrd=http://ocp-hub.lab.rmkra.us/rhcos-4.2.0-x86_64-installer-initramfs.img console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.install_dev=vda coreos.inst.image_url=http://ocp-hub.lab.rmkra.us/rhcos-4.2.0-x86_64-metal-bios.raw.gz coreos.inst.ignition_url=http://ocp-hub.lab.rmkra.us/ocp/bootstrap.ign
```

If you are using Red Hat Satellite as your PXEBoot server, the PXELinux template below will automatically pass the host to the correct ignition file based on its naming convention. It assumes that your hostnames begin with master, worker, or bootstrap.

```
<%
    # remote urls for CoreOS resources
    data_server = 'http://ocp-hub.lab.rmkra.us/'
    kernel = data_server + 'rhcos-' + @host.operatingsystem.major.to_s + '.' + @host.operatingsystem.minor.to_s + '.' + @host.operatingsystem.release_name.to_s + '-' + @host.architecture.to_s  + '-installer-kernel'
    initrd = data_server + 'rhcos-' + @host.operatingsystem.major.to_s + '.' + @host.operatingsystem.minor.to_s + '.' + @host.operatingsystem.release_name.to_s + '-' + @host.architecture.to_s  + '-installer-initramfs.img'
    image  = data_server + 'rhcos-' + @host.operatingsystem.major.to_s + '.' + @host.operatingsystem.minor.to_s + '.' + @host.operatingsystem.release_name.to_s + '-' + @host.architecture.to_s  + '-metal-bios.raw.gz'
    disk   = host_param('rhcos-disk')
    
    # pick the correct ignition file based on the machine name
    if @host.name.to_s.start_with?('master') then
        ign = data_server + host_param('clusterid') + '/' + 'master.ign'
    elsif @host.name.to_s.start_with?('worker') then
        ign = data_server + host_param('clusterid') + '/' + 'worker.ign'
    else
        ign = data_server + host_param('clusterid') + '/' + 'bootstrap.ign'
    end
    
-%>

DEFAULT pxeboot
TIMEOUT 20
PROMPT 0
LABEL pxeboot
    KERNEL <%= kernel %>
    APPEND ip=dhcp rd.neednet=1 initrd=<%= initrd %> console=tty0 console=ttyS0 coreos.inst=yes coreos.inst.install_dev=<%= disk %> coreos.inst.image_url=<%= image %> coreos.inst.ignition_url=<%= ign %>
```

You can also boot the hosts from ISO's or VM Images, but that is not covered here.

## Watch the Installation Status

Go into the build directory and run the following command to watch the installation progress.

```bash
./openshift-install wait-for bootstrap-complete --log-level debug
```

You can also view the status of the load balancer by going to:
```
http://api.ocp.example.com:8080/stats
```

## Boot the Bootstrap Server
The domain name for this server should be `bootstrap.CLUSTERID.example.com`. 
Forward and reverse DNS lookups must work for this server.
Power on the bootstrap server. Once PXELinux has pulled the RHCOS image from the data server, cancel the build on your PXE server. RHCOS will be installed on the node, then the node will reboot into RHCOS.
Wait for the bootstrap server to "go green" in the load balancer stats before continuing.

## Boot the Master Servers
The domain names for these server should be `master-X.CLUSTERID.example.com`.
Forward and reverse DNS lookups must work for these servers.
Power on the bootstrap server. Once PXELinux has pulled the RHCOS image from the data server, cancel the build on your PXE server. RHCOS will be installed on the node, then the node will reboot into RHCOS.

As the master servers come online, you will see them "go green" in the load balancer status and the bootstrap node "go red" in the load balancer status. First, the API server will come online. The Config Server will not come online until the bootstrapping is nearly completed. You'll also see the following messages from the openshift-install prompt.

```
...
INFO API v1.14.6+868bc38 up                       
INFO Waiting up to 30m0s for bootstrapping to complete... 
DEBUG Bootstrap status: complete                   
INFO It is now safe to remove the bootstrap resources 
```

Once the OpenShift Installer tells you to remove the bootstrap resources, shutdown the bootstrap node and continue to the worker nodes.

## Configure CLI Authentication

At this point, you'll need to start using the `oc` command to approve worker nodes as they come online.
To do this, you'll first need to configure your `oc` command to communicate with your new cluster.
Move into the build directory and type the following command.

```bash 
export KUBECONFIG=`pwd`/auth/kubeconfig
```

You can now check your cluster status with `oc get nodes`.

```bash
>> oc get nodes
NAME                        STATUS   ROLES           AGE     VERSION
master-0.ocp.lab.rmkra.us   Ready    master,worker   50m     v1.14.6+7e13ab9a7
```

You should see all of your masters online and Ready.


## Boot the Worker Servers
The domain names for these server should be `worker-X.CLUSTERID.example.com`.
Forward and reverse DNS lookups must work for these servers.
Power on the bootstrap server. Once PXELinux has pulled the RHCOS image from the data server, cancel the build on your PXE server. RHCOS will be installed on the node, then the node will reboot into RHCOS.

As the worker node bootstraps itself, it will make various certificate requests to the cluster that must be approved.

To check for pending requests, use the command `oc get csr`.

```bash
>> oc get csr
NAME        AGE    REQUESTOR                                                                   CONDITION
csr-c2vqh   8m4s   system:node:worker-0.ocp.lab.rmkra.us                                       Approved,Issued
csr-f48br   8s     system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
csr-n66xh   51m    system:node:master-0.ocp.lab.rmkra.us                                       Approved,Issued
csr-s59x4   15m    system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
csr-tmspj   51m    system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Approved,Issued
```

In this example, the csr `csr-f48br` is pending and requires approval.
To approve this csr:

```bash
oc adm certificate approve csr-f48br
```

As the workers come online, use `oc get nodes` to watch their status.

```bash
>> oc get nodes
NAME                        STATUS     ROLES           AGE   VERSION
master-0.ocp.lab.rmkra.us   Ready      master,worker   54m   v1.14.6+7e13ab9a7
worker-0.ocp.lab.rmkra.us   Ready      worker          10m   v1.14.6+7e13ab9a7
worker-1.ocp.lab.rmkra.us   NotReady   worker          3s    v1.14.6+7e13ab9a7
```

You are waiting for all worker nodes to report as `Ready`.

## Setup Registry Storage

The only storage that is required out of the box is the storage for the built in registry.
To configure that to point to ephemeral (not permanent) space, use the following command.

```bash
oc patch configs.imageregistry.operator.openshift.io cluster \
--type merge --patch '{"spec":{"storage":{"emptyDir":{}}}}'
```

## Wait for the Install to Complete

Everything should now be in place for the cluster to automatically complete its installation. Use the OpenShift installer to wait for the installation to complete.

```bash
./openshift-install wait-for install-complete
```

## Clean up the Load Balancers

At this point you can remove the bootstrap node from the load balancers.

## Starting Over

If you would like to restart a fresh cluster deployment, you will first need to clean the build directory.
To assist with this, a cleanup playbook is provided.

```bash
./cleanup.yml -i inventory -e @build.CLUSTERID/answers.yaml
```

This will remove the OpenShift installer and any artifacts created by it. 
The answers file will NOT be removed by the cleanup script.

## Persistent Answers

Some answers, such as your pull secret and, possibly, your sshpubkey will be the same across multiple cluster deployments.
For these, you can create a central answers file so you do not need to duplicate these values. Then, you can import multiple answers files at runtime.
Make sure you import the global answers file second to ensure proper variable precedence.

```bash
./prepare.yml -i inventory -e @build.CLUSTERID/answers.yaml -e @answers.yml 
```