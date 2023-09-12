# OpenShift Hybrid Platform UPI Installation
There are many solutions and guides for installing an OpenShift cluster across various platforms, however for more unique solutions using a hybrid-platform approach, documentation can be difficult to find.  

This document is intended to serve as a guide/reference for installing an OpenShift 4.12 cluster on user-provisioned infrastructure (UPI), with master nodes in VMware vCenter and worker nodes on bare-metal.  

For this example installation, we will be provisioning 1 bootstrap and 3 master nodes as virtual machines in vCenter, and 4 worker machines on bare-metal. This installation starts like most other UPI installations, but differs when it comes to creating the install config and generating the manifests.  

## Prerequisites  
Requirements:  
- Bastion node with SSH access and internet connectivity
- Ability to provision machines in VMware vCenter (with necessary resource requirements)
- Ability to provision machines in bare-metal environment (with necessary resource requirements)

## Bastion setup

Before beginning with configuring the installation, we must configure the Bastion node if it is not yet set up.    

### Install packages  

First, install the required packages on the Bastion node.  

```
sudo dnf install -y libvirt qemu-kvm mkisofs  python3-devel jq ipmitool tmux tar bind-utils dnsmasq python3-dns
```

If you receive an error during download for repository 'DVD' it can be resolved by removing the YUM repo:  
```
rm -rf /etc/yum.repos.d/dvd.repo
```
