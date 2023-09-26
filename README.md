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

### Create core user  

Add the 'core' user as a sudoer.  

```
sudo useradd core  
sudo passwd core MYPASSWORD  
sudo echo "core ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/core  
sudo chmod 0440 /etc/sudoers.d/core  
sudo su - core -c "ssh-keygen -t rsa -f /home/core/.ssh/id_rsa -N ''"  
sudo usermod --append --groups libvirt core  
```
If the libvirt group does not exist, you can add it manually.  

### Generate SSH key  

Switch to the core user and generate a SSH key.  

```
su core
$ ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa
```

Make sure the SSH key is generated as the core user, not root. To verify, cat ~/.ssh/id_rsa.pub and see the end of the key, it should say "core@<hostname>".  

### Download OCP binaries  

Create a directory to store the OCP binaries in the core user's home directory.  

```
$ sudo mkdir ~/bin  
$ OCP4_BASEURL=https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.12.9  
$ LATEST_VERSION=$(curl -s ${OCP4_BASEURL}/release.txt | grep 'Version: ' | awk ' {print $2}')  
$ curl -s ${OCP4_BASEURL}/openshift-client-linux-$LATEST_VERSION.tar.gz | tar     -xzf - -C ~/bin oc kubectl  
$ curl -s ${OCP4_BASEURL}/openshift-install-linux-$LATEST_VERSION.tar.gz | tar -xzf - -C ~/bin/ openshift-install  
```

Verify your OC binaries were successfully installed:  
```
$ oc version
```  
```
Client Version: 4.12.9
Kustomize Version: v4.5.7
```


```
$ openshift-install version
```  
```
openshift-install 4.12.9
built from commit 15d8ffe8fa3a5ab2f36f7e0a54126c98733aaf25
release image quay.io/openshift-release-dev/ocp-release@sha256:96bf74ce789ccb22391deea98e0c5050c41b67cc17defbb38089d32226dba0b8
release architecture amd64
```

 
```
$ kubectl version
```  
```
Client Version: version.Info{Major:"1", Minor:"25", GitVersion:"v1.25.2", GitCommit:"846602e50eb29ecdc7441ebed2fc846048959eaa", GitTreeState:"clean", BuildDate:"2023-03-08T15:10:20Z", GoVersion:"go1.19.4", Compiler:"gc", Platform:"linux/amd64"}
Kustomize Version: v4.5.7
```

### Install VMware packages  

Install the necessary packages for VMware:  
```
$ sudo curl -L -o - "https://github.com/vmware/govmomi/releases/latest/download/govc_$(uname -s)_$(uname -m).tar.gz" | tar -C /usr/local/bin -xvzf - govc
```

Verify govc was installed successfully and with the correct version:  
```
$ which govc
```
```
/usr/local/bin/govc
```

```
$ govc version
```
```
govc 0.30.7
```

### Install and configure DNS  

To configure DNS on the Bastion node, install bind and configure Named.  
```
$ sudo yum install bind
```

#### Edit named.conf

Edit named.conf with your Bastion node's IP info.    
```
$ vi /etc/named.conf
```

Edit the *options* entry, adding your Bastion IP for <YOUR_BASTION_IP_HERE>, and "any;" to the allow-query line:  
```
options {
        listen-on port 53 { 127.0.0.1; <YOUR BASTION IP HERE>; };
        listen-on-v6 port 53 { ::1; };
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        secroots-file   "/var/named/data/named.secroots";
        recursing-file  "/var/named/data/named.recursing";
        allow-query     { localhost; any; };
        ...output omitted...  
```

#### Edit .zones file

Edit your .zones file to add entries for the forward and reverse zones. Look for a file ending in .zones in your /etc/, starting with named. 
```
$ vi /etc/named.rfc1912.zones
```

Above the other zones, add two entries for forward and reverse zones.  
For the forward zone, replace example.com with your zone name.  

For the reverse zone, replace the reversed IP address with your IP address reversed without the trailing numbers. This example is done for the IP address 192.168.0.1.  
```
// Forward zone
zone "example.com" IN { // Replace example.com with your zone name
        type master;
        file "example.forward.zone";
        allow-update { none; };
};
// Reverse zone
zone "0.168.192.in-addr.arpa" IN { // Replace 0.168.192 with your IP address reversed without the final digits (keeping .in-addr.arpa) 
        type master;
        file "example.reverse.zone";
        allow-update { none; };
};
```

#### Create forward and reverse zone files

In your /var/named/ directory, add two files ending in .zone, with the same filenames as specified in the .zones file above.  

```
$ vi /var/named/example.forward.zone
```

Create the forward zone file with the following content. Edit it to include your networking configuration.  
Replace example.com with your zone name, ns1.example.com with your nameserver, each machine name with the correct name for boostrap, master and workers, and each of the <YOUR IP> lines with the correct IP assignments for each.  

```
$ORIGIN example.com.
$TTL 1W
@       IN      SOA     ns1.example.com.  root (
                        2019070700      ; serial
                        3H              ; refresh (3 hours)
                        30M             ; retry (30 minutes)
                        2W              ; expiry (2 weeks)
                        1W )            ; minimum (1 week)
        IN      NS      ns1.example.com.

ns1.example.com.          IN      A       <YOUR BASTION IP HERE>

helper.test.example.com.          IN      A       192.168.0.3
api.test.example.com.             IN      A       <YOUR IP>
api-int.test.example.com.         IN      A       <YOUR IP>
*.apps.test.example.com.          IN      A       <YOUR IP>
bootstrap.test.example.com.       IN      A       <YOUR IP>
master1.test.example.com.         IN      A       <YOUR IP>
master2.test.example.com.         IN      A       <YOUR IP>
master3.test.example.com.         IN      A       <YOUR IP>
worker1.test.example.com.         IN      A       <YOUR IP>
worker2.test.example.com.         IN      A       <YOUR IP>
worker3.test.example.com.         IN      A       <YOUR IP>
worker4.test.example.com.         IN      A       <YOUR IP>
```

Similar to the last file, create a new file for the reverse zone, and edit it to include your networking configuration.  

```
$ vi /var/named/npp.reverse.zone
```

Create the reverse zone file with the following content.  
Replace the $ORIGIN reverse IP address with your IP address reversed, without the final digits. This example was done for the IP 192.168.0.1.  

For each entry, replace <TRAILING IP DIGITS> with the final IP digits of each. The first line was done as an example, using the IP 192.168.0.3 for helper.test.example.com.
Since the rest of the reverse IP was specified in the $ORIGIN line, you only need to include the identifying final digits for each.

```
$ORIGIN 0.168.192.in-addr.arpa.
$TTL 1W
@       IN      SOA     ns1.example.com.             root (
                        2019070700      ; serial
                        3H              ; refresh
                        30M             ; retry
                        2W              ; expiry
                        1W )            ; minimum

        IN      NS  ns1.example.com.

3     IN      PTR     helper.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     api.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     api-int.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     bootstrap.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     master1.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     master2.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     master3.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     worker1.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     worker2.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     worker3.test.example.com.
<TRAILING IP DIGITS>     IN      PTR     worker4.test.example.com.
```

Lastly, edit the NetworkManager config to stop it from automatically adding DNS servers.  
Create a new .conf file in /etc/NetworkManager/conf.d/.  

```
$ vi /etc/NetworkManager/conf.d/dns-none.conf
```
```
[main]
dns=none
```

After creating this file, reload NetworkManager and edit resolv.conf with your nameserver.  
```
$ sudo systemctl reload NetworkManager
```
```
$ vi /etc/resolv.conf
```
```
# Generated by NetworkManager
search test.example
nameserver <YOUR NAMESERVER IP>
```

Save the file, and start and enable the Named service.  

```
$ sudo systemctl start named.service
$ sudo systemctl enable named.service
$ sudo systemctl restart named.service
$ sudo systemctl status named.service
```

To test your connectivity, try to reach your hosts using nslookup or dig (this example we try to reach .apps):  
```
$ nslookup testnetworking.apps.test.example.com
```
```
$ dig testnetworking.apps.test.example.com
```

### Install and configure HAProxy  

If you are using a different load balancer setup, you can skip this step. For the sake of this example, we simply used the Bastion server for HAProxy.  

```
$ sudo yum install haproxy
```
```
$ vi /etc/haproxy/haproxy.cfg
```

Below *defaults* in the common defaults section, add the following listeners:  

```
listen api-server-6443
    bind *:6443
    mode tcp
    server bootstrap bootstrap.test.example.com:6443 check inter 1s backup
    server master1 master1.test.example.com:6443 check inter 1s
    server master2 master2.test.example.com:6443 check inter 1s
    server master3 master3.test.example.com:6443 check inter 1s
listen machine-config-server-22623
    bind *:22623
    mode tcp
    server bootstrap bootstrap.test.example.com:22623 check inter 1s backup
    server master1 master1.test.example.com:22623 check inter 1s
    server master2 master2.test.example.com:22623 check inter 1s
    server master3 master3.test.example.com:22623 check inter 1s
listen ingress-router-443
    bind *:443
    mode tcp
    balance source
    server worker1 worker1.test.example.com:443 check inter 1s
    server worker2 worker2.test.example.com:443 check inter 1s
    server worker3 worker3.test.example.com:443 check inter 1s
    server worker4 worker4.test.example.com:443 check inter 1s
#listen ingress-router-80
#    bind *:80
#    mode tcp
#    balance source
#    server worker1 worker1.test.example.com:80 check inter 1s
#    server worker2 worker2.test.example.com:80 check inter 1s
#    server worker3 worker3.test.example.com:80 check inter 1s
#    server worker4 worker4.test.example.com:80 check inter 1s
```

The section to listen on port 80 is commented out, because for the following steps we will need a web server on that port.  

```
$ sudo systemctl enable haproxy
$ sudo systemctl start haproxy
$ sudo systemctl status haproxy
```

### Install Apache

Install HTTPD so we can later set up the required web server.  

```
$ sudo yum install httpd
```

### Install and configure DHCP

In this example, we have a basic DHCP server running on the Bastion node. For many implementations, DHCP may be running on another server, in which case these steps can be skipped.  

```
$ sudo yum install dhcpd-server
```

Edit the DHCP config file, and add entries for each required host. Replace <IP_ADDRESS_HERE> and <MAC_ADDRESS_HERE> with the correct IP and MAC addresses respectively.  

```
$ vi /etc/dhcp/dhcpd.conf
```
```
option domain-name      "example.com";
option domain-name-servers      "ns1.example.com";

authoritative;


default-lease-time 6000;
max-lease-time 7200;

host bootstrap.test.example.com {
    hardware ethernet <MAC_ADDRESS_HERE>;
    fixed-address <IP_ADDRESS_HERE>;
}
host master1.test.example.com {
    hardware ethernet <MAC_ADDRESS_HERE>;
    fixed-address <IP_ADDRESS_HERE>;
}
host master2.test.example.com {
    hardware ethernet <MAC_ADDRESS_HERE>;
    fixed-address <IP_ADDRESS_HERE>;
}
host master3.test.example.com {
    hardware ethernet <MAC_ADDRESS_HERE>;
    fixed-address <IP_ADDRESS_HERE>;
}
host worker1.test.example.com {
    hardware ethernet <MAC_ADDRESS_HERE>;
    fixed-address <IP_ADDRESS_HERE>;
}
host worker2.test.example.com {
    hardware ethernet <MAC_ADDRESS_HERE>;
    fixed-address <IP_ADDRESS_HERE>;
}
host worker3.test.example.com {
    hardware ethernet <MAC_ADDRESS_HERE>;
    fixed-address <IP_ADDRESS_HERE>;
}
host worker4.test.example.com {
    hardware ethernet <MAC_ADDRESS_HERE>;
    fixed-address <IP_ADDRESS_HERE>;
}
```

Start and enable the service and verify there are no errors.  

```
$ sudo systemctl enable dhcpd
$ sudo systemctl start dhcpd
$ sudo systemctl status dhcpd
```

## Generate manifests and ignition files 

Now, we have all of the prerequisites configured. As long as the DHCP and load balancer are configured correctly, then we can proceed with creating the install-config and generating the necessary manifests for installation.  

### Create installation config

Since we are using a hybrid-platform approach, we will be leaving *platform* as *none*, otherwise we will run into issues when trying to use the same manifests for different platforms.  

Create a new directory to stage your installation files, and then create a new file called install-config.YAML:  

```
$ cd /home/core/
$ mkdir ocp-install
$ vi ocp-install/install-config.yaml
```

Create the following YAML file install-config.yaml.  
Make sure the *worker* replicas is set to 0, and the *master* replicas is set to how many master nodes you will provision.  
Replace the following placeholders with their correct values:  
- <CLUSTER_NAME> = name of your cluster
- <POD_SUBNET_IP> = block of IP addresses from which pod IPs are allocated
- <SERVICE_NETWORK_IP> = IP address pool to use for service IP addresses
- <MACHINE_NETWORK_IP> = public CIDR of the eternal network
- <PULL_SECRET> = pull secret from the cluster manager
- <SSH_KEY> = SSH public key for the *core* user

```
apiVersion: v1
baseDomain: example.com
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 0
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: <CLUSTER_NAME>
networking:
  clusterNetwork:
  - cidr: <POD_SUBNET_IP> # Example: 10.128.0.0/14
    hostPrefix: 23
  networkType: OpenShiftSDN
  serviceNetwork:
  - <SERVICE_NETWORK_IP> # Example: 172.30.0.0/16
  machineNetwork:
  - cidr: <MACHINE_NETWORK_IP> # Example: 10.0.0.0/28
platform:
  none: {}
fips: false
pullSecret: '{"auths": <PULL_SECRET>}'
sshKey: 'ssh-rsa <SSH_KEY>'
```

Save a copy backup of this file for yourself. When the install-config is used to generate the manifests, the file is consumed, so it is important to keep a backup.  

### Generate Kubernetes manifests  

Now, we will use the install-config.YAML we previously created to generate the required Kubernetes manifests for our cluster.  

```
$ cd bin
$ ./openshift-install create manifests --dir ~/ocp-install/
```

Wait for the OpenShift installer to generate the manifests. If an error occurs, ensure that install-config.YAML contains no errors and verify the correct directory was provided with the --dir flag.  

Next, edit the cluster-scheduler manifest and disable mastersSchedulable.  

```
$ vi ~/ocp-install/manifests/cluster-scheduler-02-config.yml
```
Change *mastersSchedulable* to **false**.  

Now, use the OpenShift installer to generate the Ignition configs required to provision each machine.  

```
$ ./openshift-install create ignition-configs –dir ~/ocp-install/
```

Create the merge-bootstrap.ign Ignition file, and provide the URL where your bootstrap.ign file will be staged (on the web server).   
Replace <WEBSERVER_URL> with the URL to your web server, in our case, this would be the IP of your Bastion node.

```
$ vi ~/ocp-install/merge-bootstrap.ign
```
```
{
  "ignition": {
    "config": {
      "merge": [
        {
          "source": "http://<WEBSERVER_URL>/bootstrap.ign",
          "verification": {}
        }
      ]
    },
    "timeouts": {},
    "version": "3.2.0"
  },
  "networkd": {},
  "passwd": {},
  "storage": {},
  "systemd": {}
}
```

Now, your *ocp-install* directory should contain each of the required .ign files (bootstrap, master, worker, and merge-bootstrap).  


### Stage Ignition files and create base64

For each of the Ignition files except the bootstrap, encode the files into base64 strings.  
```
$ base64 -w0 master.ign > master.64
$ base64 -w0 merge-bootstrap.ign > merge-bootstrap.64
$ base64 -w0 worker.ign > worker.64
```  

Next, copy all of the .ign files to your web server. Make sure to set the file permissions and ownership to the core user.
```
$ sudo cp ocp-install/*.ign /var/www/html/
$ sudo chmod 777 /var/www/html/*.ign
$ sudo chown core:core /var/www/html/*.ign
$ sudo systemctl start httpd
$ sudo systemctl enable httpd
$ sudo systemctl status httpd
```

Verify the files on the web server are accessible:  
```
$ curl -k http://localhost/bootstrap.ign
```

Get the SHA512Sum of the bootstrap.ign file, and save the output for later.  

```
$ sha512sum ocp-install/bootstrap.ign
```

## Provision machines and install RHCOS  

Now, we are ready to provision our masters, bootstrap, and workers and install RHCOS on each.  

### Install for vSphere  

Starting with the vSphere machines, provision each of your required master nodes, and one bootstrap node via the vCenter UI.  

1. Download the correct version of the RHCOS .OVA from the [OpenShift mirror registry](https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/)
2. In the vSphere client, right click on the datacenter you are using for this deployment
3. New Folder > New VM and Template Folder
4. Set the name as the infrastructure ID of your cluster. See the below command to find the infra ID.  
```
jq -r .infraID metadata.json
```
4. Right click on the new folder > Deploy OVF Template
5. Select the previously downloaded .OVA file from URL or local file
6. Select a name for the template (something generic like RHCOS is ok) and select the previously created folder
7. Select the vSphere cluster in the compute resource tab
8. On the Storage tab, select the storage options for your VM
9. On the Network tab, select the network specified for your cluster
10. Leave the Customize Template entries blank
11. Create the template
12. After the template creates, right click > Clone > Clone to Virtual Machine
13. Clone the template for each machine you need and set its name (bootstrap, master1, master2, master3, etc)
14. Repeat step 13 for each required vSphere machine

### Set the IPs and Ignition data in vCenter  

For this step, we have to declare the host IPs for each VMware machine, provide the base64 strings for the Ignition files, and set the necessary parameters in vCenter.  

To start, disable the firewall and SELinux on the Bastion server.  

Create a new Bash script to automate the next steps. Edit the script to contain your vSphere credentials, the merge-bootstrap and master base64s, your network configuration, and the VM name and IP address for each machine.  

```
export GOVC_USERNAME='username@vsphere.local'

export GOVC_PASSWORD='password'

export GOVC_URL='vcenter url'

export GOVC_INSECURE=true

export GOVC_DATACENTER='datacenter name'

# BASE64 hash of the merge-bootstrap ignition file
BOOTS64=''

# BASE64 hash of the master ignition file
MASTER64=''

# Network configuration
SUBNET="<SUBNET_MASK_IP>"
GW="<GATEWAY_IP>"
DNS="<DNS_SERVER_IP>"

declare -A HOSTS_IP
HOSTS_IP["bootstrap.test.example.com"]="<BOOTSTRAP_IP>"
HOSTS_IP["master1.test.example.com"]="<MASTER1_IP>"
HOSTS_IP["master2.test.example.com"]="<MASTER2_IP>"
HOSTS_IP["master3.test.example.com"]="<MASTER3_IP>"

declare -A HOSTS_64
HOSTS_64["bootstrap.test.example.com"]=${BOOTS64}
HOSTS_64["master1.test.example.com"]=${MASTER64}
HOSTS_64["master2.test.example.com"]=${MASTER64}
HOSTS_64["master3.test.example.com"]=${MASTER64}


for HOST in ${!HOSTS_IP[@]}; do
        IPCFG="ip=${HOSTS_IP[${HOST}]}::${GW}:${SUBNET}:${HOST}::none nameserver=${DNS}"
        govc vm.change -vm "${HOST}" -e "guestinfo.afterburn.initrd.network-kargs=${IPCFG}"
        govc vm.change -vm "${HOST}" -e "guestinfo.ignition.config.data=${HOSTS_64[${HOST}]}"
        govc vm.change -vm "${HOST}" -e "guestinfo.ignition.config.data.encoding=base64"
        govc vm.change -vm "${HOST}" -e "disk.EnableUUID=TRUE"
done
```

Modify the script permissions to allow it to be executed by the core user, and run the script:  
```
$ chmod +x vm.sh
$ ./vm.sh
```

Lastly, power on each of the newly created VMs, and check the console to make sure the Ignition ran properly.  
```
Ignition: ran on 2022/03/14 14:48:33 UTC (this boot)
Ignition: user-provided config was applied
```

### Install for bare-metal  

Now, we will provision the required workers on bare metal.  

First, create a new directory separate from where your OCP installation files are staged. This directory will be used to store the kernel, initramfs, and rootfs files required to install RHCOS.  

```
$ mkdir imgs
$ cd imgs
```

Retrieve the kernel, initramfs, and rootfs files. This command will print the URL to download the necessary files. Choose the extension that is appropriate for your platform (e.g. x86_64).  

```
$ openshift-install coreos print-stream-json | grep -Eo '"https.*(kernel-|initramfs.|rootfs.)\w+(\.img)?"'
```

Download the files to your previously created directory.  

```
$ curl <rootfs url> –output rootfs.img
$ curl <kernel url> –output kernel
$ curl <initramfs url> –output initramfs.img
```

Next, copy the downloaded files to the web server at /var/www/html/ and set their permissions and ownership.  

```
$ sudo cp *.img /var/www/html/
$ sudo cp kernel /var/www/html/
$ sudo chown core:core /var/www/html/*.img
$ sudo chown core:core var/www/html/kernel
$ sudo chmod 777 /var/www/html/*.img
$ sudo chmod 777 /var/www/html/kernel
```

Configure your network boot infrastructure to allow your bare-metal machines to boot from the local disk after RHCOS is installed.  

Retrieve the SHA512 sum of the worker.ign file for use in the coreos installer:  

```
$ sha512sum /home/core/ocp-install/worker.ign
```

Now, run the coreos installer by using the coreos install command, with the URL to the worker.ign file, the installation device, and the SHA512 digest:  

```
$ sudo coreos-installer install --ignition-url=http://<WEB_SERVER_IP>/worker.ign /dev/sda --ignition-hash=sha512-<DIGEST>
```

Monitor the install progress, then reboot the machines. Verify the Ignition ran properly in the machine console.  

```
Ignition: ran on 2022/03/14 14:48:33 UTC (this boot)
Ignition: user-provided config was applied
```

If done properly, RHCOS will be installed successfully on each worker.  

## Create the cluster  

To create the OpenShift Container Platform cluster, you wait for the bootstrap process to complete on the machines that you provisioned by using the Ignition config files that you generated with the installation program.

**Prerequisites:**  

. Create the required infrastructure for the cluster.
. You obtained the installation program and generated the Ignition config files for your cluster.
. You used the Ignition config files to create RHCOS machines for your cluster.
. Your machines have direct Internet access or have an HTTP or HTTPS proxy available.

Monitor the bootstrap process
```
 $ openshift-install --dir /home/core/ocp-install/ wait-for bootstrap-complete --log-level=info

    Example output

      INFO Waiting up to 30m0s for the Kubernetes API at https://api.test.example.com:6443...
      INFO API v1.19.0 up
      INFO Waiting up to 30m0s for bootstrapping to complete...
      INFO It is now safe to remove the bootstrap resources
```
  
The command succeeds when the Kubernetes API server signals that it has been bootstrapped on the control plane machines.

After bootstrap process is complete, remove the bootstrap machine from the load balancer.

*NOTE*:  
 You must remove the bootstrap machine from the load balancer at this point. You can also remove or reformat the machine itself.

Next, approve the certificate signing requests for your machines.  

When you add machines to a cluster, two pending certificate signing requests (CSRs) are generated for each machine that you added. You must confirm that these CSRs are approved or, if necessary, approve them yourself. The client requests must be approved first, followed by the server requests.



Confirm that the cluster recognizes the machines:

```
$ oc get nodes
  
[core@admin ocp-install]$ oc get nodes
NAME                                 STATUS   ROLES    AGE   VERSION
. master1.test.example.com     Ready    master   20d   v1.23.12+6b34f32
. master2.test.example.com     Ready    master   20d   v1.23.12+6b34f32
. master3.test.example.com     Ready    master   20d   v1.23.12+6b34f32
. worker1.test.example.com     Ready    worker   20d   v1.23.12+6b34f32
. worker2.test.example.com     Ready    worker   20d   v1.23.12+6b34f32
. worker3.test.example.com     Ready    worker   20d   v1.23.12+6b34f32
. worker4.test.example.com     Ready    worker   20d   v1.23.12+6b34f32
```


*NOTE*:  
   The preceding output might not include the compute nodes, also known as worker nodes, until some CSRs are approved.

Review the pending CSRs and ensure that you see the client requests with the Pending or Approved status for each machine that you added to the cluster:

```
   $ oc get csr
   Example output

  NAME        AGE     REQUESTOR                                                                   CONDITION
  csr-8b2br   15m     system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
  csr-8vnps   15m     system:serviceaccount:openshift-machine-config-operator:node-bootstrapper   Pending
```

In this example, two machines are joining the cluster. You might see more approved CSRs in the list.

To approve them individually, run the following command for each valid CSR:

```
   $ oc adm certificate approve csr-8b2br csr-8vnps
```  

To approve all CSRs at once:  

```
   $ oc get csr -o go-template='{{range .items}}{{if not .status}}{{.metadata.name}}{{"\n"}}{{end}}{{end}}' | xargs --no-run-if-empty oc adm certificate approve
```
After all client and server CSRs have been approved, the machines have the Ready status.    
   Verify this by running the following command:

```
    $ oc get nodes

   Example output

    [core@admin ocp-install]$ oc get nodes

    NAME   STATUS   ROLES    AGE   VERSION
   . master1     Ready    master   20d   v1.23.12+6b34f32
   . master2     Ready    master   20d   v1.23.12+6b34f32
   . master3     Ready    master   20d   v1.23.12+6b34f32
   . worker1     Ready    worker   20d   v1.23.12+6b34f32
   . worker2     Ready    worker   20d   v1.23.12+6b34f32
   . worker3     Ready    worker   20d   v1.23.12+6b34f32
   . worker4     Ready    worker   20d   v1.23.12+6b34f32
```

If all of your nodes appear, then your OpenShift cluster has successfully installed.  


## Log into the cluster by using the CLI

You can log in to your cluster as a default system user by exporting the cluster kubeconfig file. The kubeconfig file contains information about the cluster that is used by the CLI to connect a client to the correct cluster and API server. The file is specific to a cluster and is created during OpenShift Container Platform installation.

**Prerequisites**:  

* You deployed an OpenShift Container Platform cluster.
* You installed the oc CLI.


Export the kubeadmin credentials: 

```
   $ export KUBECONFIG=/home/core/ocp-install/auth/kubeconfig
```

Verify you can run oc commands successfully using the exported configuration:

```
   $ oc whoami
```

After the control plane initializes, check to make sure the required Operators are available:  

```
 $ watch -n5 oc get clusteroperators


    [core@admin ocp-install]$ oc get co
    NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
    authentication                             4.12.9   True        False         False      3d1h    
    baremetal                                  4.12.9   True        False         False      20d     
    cloud-controller-manager                   4.12.9   True        False         False      20d     
    cloud-credential                           4.12.9   True        False         False      20d     
    cluster-autoscaler                         4.12.9   True        False         False      20d     
    config-operator                            4.12.9   True        False         False      20d     
    console                                    4.12.9   True        False         False      3d1h    
    csi-snapshot-controller                    4.12.9   True        False         False      9d      
    dns                                        4.12.9   True        False         False      20d     
    etcd                                       4.12.9   True        False         False      20d     
    image-registry                             4.12.9   True        False         False      20d     
```

Now your cluster is configured. You may proceed with any day 2 configurations, such as adding an identity provider or configuring monitoring and logging.  

## Delete bootstrap node  

Lastly, delete your bootstrap node. Make sure to:  

- Remove all "bootstrap" entries from the HAproxy, DHCP, and Named configurations
- Delete the bootstrap VM from VMware
- Uncomment the port 80 listener lines in the HAproxy config, if you require it

