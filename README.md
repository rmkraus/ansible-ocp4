# OCP4 Deployer

These are helper scripts to automate major parts of deploying an UPI OpenShift 4 cluster into an existing environment.
The purpose of this project is to create a fully functional cluster without duplicating existing core network infrastructure.
These playbooks will not create the necessary DHCP, DNS, or PXE Boot resources.
It is expected that these already exist in your infrastructure.
If you would like to have network infrastructure created in support of your cluster, like in a lab environement, check out [ocp4-upi-helpernode](https://github.com/christianh814/ocp4-upi-helpernode).

## Setup Data Server
The Data Server should be an existing web server to use for hosting cluster/CoreOS data. 
Web data should be stored at /var/www/html.
The CoreOS images and ignition files will be published to this server for use by the cluster.
One data server can be used for multiple clusters, if that is what your hearts desires.

## Populate Inventory File
Update the inventory file to match your desired deployment. 

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

## Setup PXE Boot Environment
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

## Boot the Bootstrap Server
The domain name for this server should be `bootstrap.CLUSTERID.example.com`. 
Forward and reverse DNS lookups must work for this server.

## Boot the Master Servers
The domain names for these server should be `master-X.CLUSTERID.example.com`.
Forward and reverse DNS lookups must work for these servers.

## Boot the Worker Servers
The domain names for these server should be `worker-X.CLUSTERID.example.com`.
Forward and reverse DNS lookups must work for these servers.



1. Existing virtual machines for bootstrap (x1), master (x3), load balancers (x2), and workers (min x2). 
2. Forward and reverse DNS for all existing VMs.
2. DNS zone managed by Red Hat IdM (FreeIPA not tested)
3. 
