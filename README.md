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
curl -L -o - "https://github.com/vmware/govmomi/releases/latest/download/govc_$(uname -s)_$(uname -m).tar.gz" | tar -C /usr/local/bin -xvzf - govc
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
yum install bind
```

#### Edit named.conf

Edit named.conf with your Bastion node's IP info.    
```
vi /etc/named.conf
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
vi /etc/named.rfc1912.zones
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
vi /var/named/example.forward.zone
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
vi /var/named/npp.reverse.zone
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
vi /etc/NetworkManager/conf.d/dns-none.conf
```
```
[main]
dns=none
```

After creating this file, reload NetworkManager and edit resolv.conf with your nameserver.  
```
systemctl reload NetworkManager
```
```
vi /etc/resolv.conf
```
```
# Generated by NetworkManager
search test.example
nameserver <YOUR NAMESERVER IP>
```

Save the file, and start and enable the Named service.  

```
systemctl start named.service
systemctl enable named.service
systemctl restart named.service
systemctl status named.service
```

To test your connectivity, try to reach your hosts using nslookup or dig (this example we try to reach .apps):  
```
nslookup testnetworking.apps.test.example.com
```
```
dig testnetworking.apps.test.example.com
```

### Install and configure HAProxy  

If you are using a different load balancer setup, you can skip this step. For the sake of this example, we simply used the Bastion server for HAProxy.  

```
yum install haproxy
```
```
vi /etc/haproxy/haproxy.cfg
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


