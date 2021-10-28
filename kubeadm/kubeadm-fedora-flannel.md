# KubeAdm based kubernetes cluster

**Note:** This document is updated to use Fedora 34. 

Reference documentation: [https://kubernetes.io/docs/setup/independent/install-kubeadm/](https://kubernetes.io/docs/setup/independent/install-kubeadm/)

Kubeadm helps you setup/bootstrap a minimum viable/usable Kubernetes cluster that just works. Kubeadm also supports cluster expansion, upgrades, downgrade, and managing bootstrap tokens, which are extra features, if you are comparing it with minikube.

A video showing kubeadm cluster setup, (in Urdu language), using this guide, is available at: [https://youtu.be/FRvkSlIUimM](https://youtu.be/FRvkSlIUimM)


## Setup / components / look and feel:
The Kubernetes cluster discussed in this guide is a virtual cluster, created using virtual machines running on KVM/Libvirt, on a Fedora 33 host computer.

### KubeAdm cluster overview:
| ![kubeadm-cluster-overview.png](kubeadm-cluster-overview.png) |
|:--:|
| *Cluster components overview of a kubeadm based cluster* |


### KubeAdm cluster networks:
| ![kubeadm-cluster-networks.png](kubeadm-cluster-networks.png) |
|:--:|
| *Cluster networks involved in a kubeadm based cluster* |

## Features and Limitations:
* Possible to create multi-node kubernetes cluster compared to minikube.
* It creates a single node master, which can work as a worker node too. High Availability for master node not available yet.
* You can create/join one or more **dedicated** worker-nodes.
* Being multi-node in nature, allows you to use multi-node features such as [advance scheduling policies](https://kubernetes.io/blog/2017/03/advanced-scheduling-in-kubernetes/).
* To expose services as type **LoadBalancer** you can install [Metal-LB](https://metallb.universe.tf/), which works beautifully.
* Choice of using different container engines, such as Docker, Rocket, etc. This is not possible in MiniKube.
* Choice of wide variety of (CNI-based) network plugins to be used for pod networking. This is not possible in MiniKube.
* Supports cluster expansion, upgrades, downgrade, etc.
* *Can* be used as a production cluster, though extreme caution is advised. You should know what you are doing!


In this guide, I will setup a single node kubernetes cluster using **kubeadm** and Docker-CE. Once installed, I will add more nodes to the cluster - again using **kubeadm**.


## Preparation:
* **RAM:** Minimum 1 GB RAM for each node (master+worker); you will only be able to run very few (and very small) containers, like nginx, mysql, etc. **2 GB RAM is better**.
* **CPU:** Minimum 2 CPU for master node; worker nodes can live with single core CPUs
* **Disk:** 4 GB for host OS + 20 GB for storing container images. (no swap)
* **Network - Infrastructure:** A functional virtual/physical network with some usable IP addresses (can be public or private) . This can be on any cloud provider as well. You are free to use any network / ip scheme for yourself. In this guide, it will be `10.240.0.0/24`
* **Network - Pod network (pod-network-cidr):** A network IP range completely separate from other two networks, with subnet mask of `/16` or smaller (e.g. `/12`). This network will be subdivided into subnets later. In this guide it will be `10.200.0.0/16` . Please note that kubeadm does not support kubenet, so we need to use one of the CNI add-ons - such as flannel. By default Flannel sets up a pod network `10.244.0.0/16`, which means that we need to pass this pod network to `kubeadm init` (further below); or, modify the flannel configuration with the pod network of our own choice - before actually applying it blindly. :)
* **Network - Service network (service-cidr):** A network IP range completely separate from other two networks, used by the services. This will be considered a completely virtual network. The default service network configured by kubeadm is `10.96.0.0/12`. In this guide, it will be `10.32.0.0/16`.
* **Firewall:** Disable Firewall (including removing the firewalld package), or open the following ports on each type of node, after the OS installation is complete. 
* **Firewall/ports - Master:** Incoming open (22, 6443, 10250, 10251, 10252, 2379, 2380)
* **Firewall/ports - Worker:** Incoming open (22, 10250, 30000-32767)
* **OS:** Any recent version of Fedora/CentOS/RHEL or Debian based OS. This guide uses Fedora 33
* **Disk Partitioning:** No swap - must disable swap partition during OS installation in order for the kubelet to work properly. See: [https://github.com/kubernetes/kubernetes/issues/53533](https://github.com/kubernetes/kubernetes/issues/53533) for details on why disable swap. Swap may seem a good idea, it is not - on Kubernetes!
* **SELinux:** Disable SELinux / App Armour.
* Should have some sort of DNS for infrastructure network.

## OS setup:

**Note:** Almost all commands in this document are run as user **"root"** while *logged in* as "root" - which eliminates the need to add the word "sudo" in front of each command. Therefore, I won't use "sudo".  

```
[root@kworkhorse lib]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain

 # Virtual Kubernetes cluster
10.240.0.31	kubeadm-node1
10.240.0.32	kubeadm-node2
10.240.0.33	kubeadm-node3
```

```
yum -y remove firewalld

yum -y install ebtables iproute-tc NetworkManager-tui
```

```
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```

```
cat > /etc/modules-load.d/k8s.conf <<EOF
br_netfilter
EOF
```


Enable the sysctl setting `net.bridge.bridge-nf-call-iptables`
```
cat > /etc/sysctl.d/k8s.conf <<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

Increase the number of open files in `limits.conf`:
```
cat >> /etc/security/limits.d/k8s.conf <<EOF
*       soft    nofile  16000
*       hard    nofile  32000
EOF
```


### Setup a fixed IP address on each node:

On RedHat and derivatives, this used to be an easy thing to do, using various script files under `/etc/sysconfig/network-scripts/` directory. 

However, in recent/latest versions of Fedora (e.g. 32+), this task is handled entirely through **NetworkManager**, and there is no network/configuration/script files in the `/etc/sysconfig/network-scripts/` directory. Also, the systemd service named **"network"** does not exist anymore, unless you install the **network-scripts** package. This is true for both GUI and non-GUI installations. 

The file to configure NetworkManager *service* is: `/etc/NetworkManager/NetworkManager.conf`. In this file the most important directive is **plugins** under the **[main]** section. The default (and always active) plugin is **`keyfile`** whether is appears in the plugins list or not. This `keyfile` plugin dictates NetworkManager to create network interface configuration files under `/etc/NetworkManager/system-connections/` directory - in the **ini-file** format. You can add the `ifcfg-rh` plugin to this directive, which will simply create the network interface configuration files under `/etc/sysconfig/network-scripts/` directory - in the **legacy** format. Even though the network connection files can be created under `/etc/sysconfig/network-scripts/` directory, they will still be managed through the **NetworkManager** service, not through the **network** service. 

The default NetworkManager.conf file:

```
[root@kubeadm-node1 ~]# cat /etc/NetworkManager/NetworkManager.conf 

[main]
plugins=keyfile


[logging]
level=INFO
domains=ALL

[root@kubeadm-node1 ~]#
```

Actually NetworkManager is not a new thing. It has been part of Linux distributions since November 2004. It is only now that it is being enforced. It is developed mostly by the developers from RedHat.

In Fedora 32+, for each network interface, a configuration file is created under `/etc/NetworkManager/system-connections/` directory.

```
[root@kubeadm-node1 ~]# ls -l /etc/NetworkManager/system-connections/
total 4
-rw------- 1 root root 387 Jan 19 12:49 enp1s0.nmconnection
[root@kubeadm-node1 ~]# 
```

The file looks like this:
```
[root@kubeadm-node1 ~]# cat /etc/NetworkManager/system-connections/enp1s0.nmconnection 
[connection]
id=enp1s0
uuid=d8360351-46d8-3cf8-9ae0-87b57e963c6d
type=ethernet
autoconnect-priority=-999
interface-name=enp1s0
permissions=
timestamp=1611065622

[ethernet]
mac-address-blacklist=

[ipv4]
address1=10.240.0.31/24,10.240.0.1
dns=10.240.0.1;8.8.8.8;
dns-search=
method=manual

[ipv6]
addr-gen-mode=stable-privacy
dns-search=
method=disabled

[proxy]

[root@kubeadm-node1 ~]# 
```

You can use various `nmcli` commands to check the various settings.

```
[root@kubeadm-node1 ~]# nmcli connection
NAME     UUID                                  TYPE      DEVICE  
enp1s0   d8360351-46d8-3cf8-9ae0-87b57e963c6d  ethernet  enp1s0  
[root@kubeadm-node1 ~]# 
```

**Note:** The `NAME` column above is actually *profile name*.


```
[root@kubeadm-node1 ~]# nmcli device show enp1s0
GENERAL.DEVICE:                         enp1s0
GENERAL.TYPE:                           ethernet
GENERAL.HWADDR:                         52:54:00:7D:76:44
GENERAL.MTU:                            1500
GENERAL.STATE:                          100 (connected)
GENERAL.CONNECTION:                     enp1s0
GENERAL.CON-PATH:                       /org/freedesktop/NetworkManager/ActiveConnection/3
WIRED-PROPERTIES.CARRIER:               on
IP4.ADDRESS[1]:                         10.240.0.31/24
IP4.GATEWAY:                            10.240.0.1
IP4.ROUTE[1]:                           dst = 10.240.0.0/24, nh = 0.0.0.0, mt = 100
IP4.ROUTE[2]:                           dst = 0.0.0.0/0, nh = 10.240.0.1, mt = 100
IP4.DNS[1]:                             10.240.0.1
IP4.DNS[2]:                             8.8.8.8
IP6.GATEWAY:                            --
[root@kubeadm-node1 ~]# 
```

**Note:** In the output above, the `GENERAL.CONNECTION` is actually the *profile name*. 

(Yes, I agree that this should be standardized.)


If you check the output of `ip address show` command, you will see more or less the same information.
```
[root@kubeadm-node1 ~]# ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 52:54:00:7d:76:44 brd ff:ff:ff:ff:ff:ff
    inet 10.240.0.31/24 brd 10.240.0.255 scope global noprefixroute enp1s0
       valid_lft forever preferred_lft forever
[root@kubeadm-node1 ~]# 
```

Routing table looks like this:
```
[root@kubeadm-node1 ~]# ip route show
default via 10.240.0.1 dev enp1s0 proto static metric 100 
10.240.0.0/24 dev enp1s0 proto kernel scope link src 10.240.0.31 metric 100 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown 

[root@kubeadm-node1 ~]# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.240.0.1      0.0.0.0         UG    100    0        0 enp1s0
10.240.0.0      0.0.0.0         255.255.255.0   U     100    0        0 enp1s0
172.17.0.0      0.0.0.0         255.255.0.0     U     0      0        0 docker0
[root@kubeadm-node1 ~]#
```

The best method to configure the network interfaces without hassle and frustration is to use **`nmtui`** text user interface.

(TODO: screenshots here)


### Disable Swap partition:
Kubernetes does not use/allow swap partition to exist, and kubeadm will simply refuse to intialize if a swap partition is active.

So, disable swap partition before continuing.

```
swapoff -a
```

Then, remove any "swap" entries from your `/etc/fstab` file.


**Note:** On newer Linux systems, use this to disable swap:
```
yum -y remove zram-generator-defaults
```


**Note:**
Fedora 33 adds a compressed-memory-based swap device using [zram](https://en.wikipedia.org/wiki/Zram). You will find a swap device active even if there is no entry in the `/etc/fstab` file. The size of this device will be half the size of the RAM.

```
[root@kubeadm-node1 ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           1976         252         209           1        1514        1585
Swap:           987           0         987
```


```
[root@kubeadm-node1 ~]# cat /etc/fstab 
/dev/mapper/fedora_kubeadm--node1-root /                       xfs     defaults        0 0
UUID=f56b8c9e-af5a-4f1a-a6ab-f207b6e241a2 /boot                   xfs     defaults        0 0
```

You can check the swap device using `zramctl`:
```
[root@kubeadm-node1 ~]# zramctl 
NAME       ALGORITHM DISKSIZE DATA COMPR TOTAL STREAMS MOUNTPOINT
/dev/zram0 lzo-rle       988M   4K   74B   12K       2 [SWAP]
[root@kubeadm-node1 ~]# 
```

The service responsible for creating ZRAM based swap is `swap-create@zram0.service`:

```
[root@kubeadm-node1 ~]# systemctl status swap-create@zram0.service
● swap-create@zram0.service - Create swap on /dev/zram0
     Loaded: loaded (/usr/lib/systemd/system/swap-create@.service; static)
     Active: active (exited) since Thu 2021-01-21 09:57:20 CET; 22min ago
       Docs: man:zram-generator(8)
             man:zram-generator.conf(5)
   Main PID: 1664 (code=exited, status=0/SUCCESS)
      Tasks: 0 (limit: 2323)
     Memory: 0B
        CPU: 0
     CGroup: /system.slice/system-swap\x2dcreate.slice/swap-create@zram0.service

Jan 21 09:57:20 kubeadm-node1 systemd[1]: Starting Create swap on /dev/zram0...
Jan 21 09:57:20 kubeadm-node1 zram-generator[1671]: Setting up swapspace version 1, size = 988 MiB (1035988992 bytes)
Jan 21 09:57:20 kubeadm-node1 zram-generator[1671]: no label, UUID=9fb22b2c-32ac-48f1-a33a-6d2b0645da5f
Jan 21 09:57:20 kubeadm-node1 systemd[1]: Finished Create swap on /dev/zram0.
[root@kubeadm-node1 ~]#
```

The default configuration file for zram is `/usr/lib/systemd/zram-generator.conf` . 

To disable zram, uninstall `zram-generator-defaults` and `zram-generator` packages. Merely stopping/disabling the `swap-create@zram0.service` - and then rebooting **will not work**.


```
[root@kubeadm-node1 ~]# yum list zram-generator-defaults
Last metadata expiration check: 0:36:01 ago on Thu 21 Jan 2021 09:55:15 AM CET.
Installed Packages
zram-generator-defaults.noarch                                          0.2.0-4.fc33                                           @anaconda
```


Removing `zram-generator-defaults` will automatically remove the `zram-generator` package too:

```
[root@kubeadm-node1 ~]# yum -y remove zram-generator-defaults
Dependencies resolved.
========================================================================================================================================
 Package                                   Architecture             Version                           Repository                   Size
========================================================================================================================================
Removing:
 zram-generator-defaults                   noarch                   0.2.0-4.fc33                      @anaconda                   294  
Removing unused dependencies:
 zram-generator                            x86_64                   0.2.0-4.fc33                      @anaconda                   893 k

Transaction Summary
========================================================================================================================================
Remove  2 Packages

Freed space: 893 k
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                1/1 
  Running scriptlet: zram-generator-defaults-0.2.0-4.fc33.noarch                                                                    1/1 
  Erasing          : zram-generator-defaults-0.2.0-4.fc33.noarch                                                                    1/2 
  Erasing          : zram-generator-0.2.0-4.fc33.x86_64                                                                             2/2 
  Running scriptlet: zram-generator-0.2.0-4.fc33.x86_64                                                                             2/2 
  Verifying        : zram-generator-0.2.0-4.fc33.x86_64                                                                             1/2 
  Verifying        : zram-generator-defaults-0.2.0-4.fc33.noarch                                                                    2/2 

Removed:
  zram-generator-0.2.0-4.fc33.x86_64                             zram-generator-defaults-0.2.0-4.fc33.noarch                            

Complete!
[root@kubeadm-node1 ~]# 
```

The `reboot` the system. After reboot, you will notice that swap is not enabled anymore.

```
[root@kubeadm-node1 ~]# free -m
              total        used        free      shared  buff/cache   available
Mem:           1976         612         718           1         646        1227
Swap:             0           0           0
[root@kubeadm-node1 ~]# 
```


### Install container runtime (Docker):
Reference: [https://kubernetes.io/docs/setup/cri/](https://kubernetes.io/docs/setup/cri/)
You can select other runtimes too, such as **Rocket (rkt)**. I will use **Docker**.

**Note:** DO NOT install docker from default Fedora/CentOS repository, that is a very old version of Docker.

On all nodes (master+worker), install Docker.
```
yum config-manager \
    --add-repo \
    https://download.docker.com/linux/fedora/docker-ce.repo
```

```
yum -y install docker-ce
```

```
systemctl enable docker
systemctl start docker
systemctl status docker
```

### Install kubeadm, kubelet and kubectl:
On each node, install:
* kubeadm: the command to actually setup / bootstrap the cluster.
* kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
* kubectl: the command line util to talk to your cluster.


All of the above three pieces of software are available from kubernetes's yum repository. So first, set that up:

```
cat > /etc/yum.repos.d/kubernetes.repo <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kube*
EOF
```

```
yum -y install kubelet kubeadm kubectl --disableexcludes=kubernetes

systemctl daemon-reload

systemctl enable kubelet

systemctl start kubelet
```

The above command installs additional packages, which are:
* cri-tools  - Command line utility to interact with container runtime, such as docker)
* containernetworking-plugins (formerly *kubernetes-cni*) - Binary files to provision container networking. The files are installed in `/opt/cni/bin` or `/usr/libexec/cni` depending on the version of Fedora you are using.
* socat - Relay for bidirectional data transfer between two independent data channels, e.g. files, pipe, device, socket, program, etc.
* conntrack-tools - Manipulate netfilter connection tracking table and run High Availability
* libnetfilter_cthelper - User-space infrastructure for connection tracking helpers
* libnetfilter_cttimeout - Timeout policy tuning for Netfilter/conntrack
* libnetfilter_queue - Netfilter queue userspace library


**Note:** If you want to install a specific version of Kubernetes in your cluster - say `1.21.6-0` -  then you need a matching version of **kubelet**, matching version of **`kubeadm`** and **`kubectl`**  (less important) . A use-case for this is a lab/test environment, where you want to learn how to upgrade a kubernetes cluster from an older version to a newer version. 


```
yum list kubelet kubeadm kubectl --disableexcludes=kubernetes
```

OR

```
yum list kubelet kubeadm kubectl --disableexcludes=kubernetes --showduplicates
```


```
[root@kubeadm-node1 ~]# yum list kubelet kubeadm kubectl --disableexcludes=kubernetes --showduplicates
. . . 

kubelet.x86_64                                                   1.20.5-0                                                     kubernetes
kubelet.x86_64                                                   1.20.6-0                                                     kubernetes
kubelet.x86_64                                                   1.20.7-0                                                     kubernetes
kubelet.x86_64                                                   1.20.8-0                                                     kubernetes
kubelet.x86_64                                                   1.20.9-0                                                     kubernetes
kubelet.x86_64                                                   1.20.10-0                                                    kubernetes
kubelet.x86_64                                                   1.20.11-0                                                    kubernetes
kubelet.x86_64                                                   1.20.12-0                                                    kubernetes
kubelet.x86_64                                                   1.21.0-0                                                     kubernetes
kubelet.x86_64                                                   1.21.1-0                                                     kubernetes
kubelet.x86_64                                                   1.21.2-0                                                     kubernetes
kubelet.x86_64                                                   1.21.3-0                                                     kubernetes
kubelet.x86_64                                                   1.21.4-0                                                     kubernetes
kubelet.x86_64                                                   1.21.5-0                                                     kubernetes
kubelet.x86_64                                                   1.21.6-0                                                     kubernetes
kubelet.x86_64                                                   1.22.0-0                                                     kubernetes
kubelet.x86_64                                                   1.22.1-0                                                     kubernetes
kubelet.x86_64                                                   1.22.2-0                                                     kubernetes
kubelet.x86_64                                                   1.22.3-0                                                     kubernetes
[root@kubeadm-node1 ~]# 
```

```
[root@kubeadm-node1 ~]# yum list kubelet-1.21.6-0 kubeadm-1.21.6-0 kubectl-1.21.6-0 --disableexcludes=kubernetes
Last metadata expiration check: 0:22:41 ago on Sat 06 Feb 2021 08:28:03 PM CET.
Available Packages
kubeadm.x86_64                                                    1.21.6-0                                                    kubernetes
kubectl.x86_64                                                    1.21.6-0                                                    kubernetes
kubelet.x86_64                                                    1.21.6-0                                                    kubernetes
[root@kubeadm-node1 ~]# 
```


To install latest version of these three components - irrespective of what version it is - use this:

```
yum -y install --disableexcludes=kubernetes \
  kubelet \
  kubeadm \
  kubectl
```  


To install specific version `1.21.6-0` of these three components, use this:

```
yum -y install --disableexcludes=kubernetes \
  kubelet-1.21.6-0 \
  kubeadm-1.21.6-0 \
  kubectl-1.21.6-0
```  


```
[root@kubeadm-node1 ~]# yum -y install --disableexcludes=kubernetes \
>   kubelet-1.21.6-0 \
>   kubeadm-1.21.6-0 \
>   kubectl-1.21.6-0
Last metadata expiration check: 0:23:49 ago on Thu 28 Oct 2021 10:14:14 AM CEST.
Dependencies resolved.
========================================================================================================================================
 Package                                      Architecture            Version                         Repository                   Size
========================================================================================================================================
Installing:
 kubeadm                                      x86_64                  1.21.6-0                        kubernetes                  9.1 M
 kubectl                                      x86_64                  1.21.6-0                        kubernetes                  9.6 M
 kubelet                                      x86_64                  1.21.6-0                        kubernetes                   20 M
Installing dependencies:
 conntrack-tools                              x86_64                  1.4.5-7.fc34                    fedora                      206 k
 containernetworking-plugins                  x86_64                  1.0.1-1.fc34                    updates                     8.7 M
 cri-tools                                    x86_64                  1.19.0-0                        kubernetes                  5.7 M
 libnetfilter_cthelper                        x86_64                  1.0.0-19.fc34                   fedora                       22 k
 libnetfilter_cttimeout                       x86_64                  1.0.0-17.fc34                   fedora                       23 k
 libnetfilter_queue                           x86_64                  1.0.2-17.fc34                   fedora                       27 k
 socat                                        x86_64                  1.7.4.1-2.fc34                  fedora                      305 k

Transaction Summary
========================================================================================================================================
Install  10 Packages

Total download size: 54 M
Installed size: 285 M
Downloading Packages:
(1/10): libnetfilter_cthelper-1.0.0-19.fc34.x86_64.rpm                                                   86 kB/s |  22 kB     00:00    
(2/10): libnetfilter_cttimeout-1.0.0-17.fc34.x86_64.rpm                                                  84 kB/s |  23 kB     00:00    
(3/10): libnetfilter_queue-1.0.2-17.fc34.x86_64.rpm                                                     414 kB/s |  27 kB     00:00    
(4/10): conntrack-tools-1.4.5-7.fc34.x86_64.rpm                                                         503 kB/s | 206 kB     00:00    
(5/10): socat-1.7.4.1-2.fc34.x86_64.rpm                                                                 1.5 MB/s | 305 kB     00:00    
(6/10): 67ffa375b03cea72703fe446ff00963919e8fce913fbc4bb86f06d1475a6bdf9-cri-tools-1.19.0-0.x86_64.rpm  1.9 MB/s | 5.7 MB     00:03    
(7/10): 0259ed831f717e860cea6f0abdcca90d84a64115fb6ca419652422add9146db9-kubeadm-1.21.6-0.x86_64.rpm    1.3 MB/s | 9.1 MB     00:06    
(8/10): 50e4c5518390e21192259ae920f37e1440f7be96c0c25d7feaead8167668cd93-kubectl-1.21.6-0.x86_64.rpm    2.4 MB/s | 9.6 MB     00:04    
(9/10): containernetworking-plugins-1.0.1-1.fc34.x86_64.rpm                                             1.2 MB/s | 8.7 MB     00:07    
(10/10): 65b73de168120e7413c9f4f825fe1783152214ffb95db38b56f073d3e7ec160e-kubelet-1.21.6-0.x86_64.rpm   5.4 MB/s |  20 MB     00:03    
----------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                   4.0 MB/s |  54 MB     00:13     
Kubernetes                                                                                               21 kB/s | 3.4 kB     00:00    
Importing GPG key 0x307EA071:
 Userid     : "Rapture Automatic Signing Key (cloud-rapture-signing-key-2021-03-01-08_01_09.pub)"
 Fingerprint: 7F92 E05B 3109 3BEF 5A3C 2D38 FEEA 9169 307E A071
 From       : https://packages.cloud.google.com/yum/doc/yum-key.gpg
Key imported successfully
Importing GPG key 0x836F4BEB:
 Userid     : "gLinux Rapture Automatic Signing Key (//depot/google3/production/borg/cloud-rapture/keys/cloud-rapture-pubkeys/cloud-rapture-signing-key-2020-12-03-16_08_05.pub) <glinux-team@google.com>"
 Fingerprint: 59FE 0256 8272 69DC 8157 8F92 8B57 C5C2 836F 4BEB
 From       : https://packages.cloud.google.com/yum/doc/yum-key.gpg
Key imported successfully
Kubernetes                                                                                              6.1 kB/s | 975  B     00:00    
Importing GPG key 0x3E1BA8D5:
 Userid     : "Google Cloud Packages RPM Signing Key <gc-team@google.com>"
 Fingerprint: 3749 E1BA 95A8 6CE0 5454 6ED2 F09C 394C 3E1B A8D5
 From       : https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
Key imported successfully
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                1/1 
  Installing       : containernetworking-plugins-1.0.1-1.fc34.x86_64                                                               1/10 
  Installing       : kubectl-1.21.6-0.x86_64                                                                                       2/10 
  Installing       : cri-tools-1.19.0-0.x86_64                                                                                     3/10 
  Installing       : socat-1.7.4.1-2.fc34.x86_64                                                                                   4/10 
  Installing       : libnetfilter_queue-1.0.2-17.fc34.x86_64                                                                       5/10 
  Installing       : libnetfilter_cttimeout-1.0.0-17.fc34.x86_64                                                                   6/10 
  Installing       : libnetfilter_cthelper-1.0.0-19.fc34.x86_64                                                                    7/10 
  Installing       : conntrack-tools-1.4.5-7.fc34.x86_64                                                                           8/10 
  Running scriptlet: conntrack-tools-1.4.5-7.fc34.x86_64                                                                           8/10 
  Installing       : kubelet-1.21.6-0.x86_64                                                                                       9/10 
  Installing       : kubeadm-1.21.6-0.x86_64                                                                                      10/10 
  Running scriptlet: kubeadm-1.21.6-0.x86_64                                                                                      10/10 
  Verifying        : conntrack-tools-1.4.5-7.fc34.x86_64                                                                           1/10 
  Verifying        : libnetfilter_cthelper-1.0.0-19.fc34.x86_64                                                                    2/10 
  Verifying        : libnetfilter_cttimeout-1.0.0-17.fc34.x86_64                                                                   3/10 
  Verifying        : libnetfilter_queue-1.0.2-17.fc34.x86_64                                                                       4/10 
  Verifying        : socat-1.7.4.1-2.fc34.x86_64                                                                                   5/10 
  Verifying        : containernetworking-plugins-1.0.1-1.fc34.x86_64                                                               6/10 
  Verifying        : cri-tools-1.19.0-0.x86_64                                                                                     7/10 
  Verifying        : kubeadm-1.21.6-0.x86_64                                                                                       8/10 
  Verifying        : kubectl-1.21.6-0.x86_64                                                                                       9/10 
  Verifying        : kubelet-1.21.6-0.x86_64                                                                                      10/10 

Installed:
  conntrack-tools-1.4.5-7.fc34.x86_64         containernetworking-plugins-1.0.1-1.fc34.x86_64  cri-tools-1.19.0-0.x86_64               
  kubeadm-1.21.6-0.x86_64                     kubectl-1.21.6-0.x86_64                          kubelet-1.21.6-0.x86_64                 
  libnetfilter_cthelper-1.0.0-19.fc34.x86_64  libnetfilter_cttimeout-1.0.0-17.fc34.x86_64      libnetfilter_queue-1.0.2-17.fc34.x86_64 
  socat-1.7.4.1-2.fc34.x86_64                

Complete!

[root@kubeadm-node1 ~]#
```


By this time, `kubeadm` is only *installed* - and is not *running*. 

```
[root@kubeadm-node1 ~]# systemctl status kubelet
○ kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; disabled; vendor preset: disabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: inactive (dead)
       Docs: https://kubernetes.io/docs/
[root@kubeadm-node1 ~]# 
```

### Activate/enable kubelet to boot at startup:
Since `kubelet` is required to run on all "master" and "worker" nodes, we enable it now. 
```
[root@kubeadm-node1 ~]# systemctl enable kubelet
Created symlink /etc/systemd/system/multi-user.target.wants/kubelet.service → /usr/lib/systemd/system/kubelet.service.

[root@kubeadm-node1 ~]# systemctl start kubelet
```

**Note:** kubelet is now set to start. Kubelet will continuously try to start and will fail (crash-loop), because it will wait for kubeadm to tell it what to do. This *"crash-loop"* is expected and normal. After you initialize your master (using kubeadm), the kubelet will run normally. Later, once you join the worker nodes to master, the `kubelet` process on worker nodes will also stop doing the *"crash-loop"*.


```
[root@kubeadm-node1 ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Thu 2021-10-28 10:42:56 CEST; 3s ago
       Docs: https://kubernetes.io/docs/
    Process: 43415 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (>
   Main PID: 43415 (code=exited, status=1/FAILURE)
        CPU: 103ms

Oct 28 10:42:56 kubeadm-node1.example.com systemd[1]: kubelet.service: Main process exited, code=exited, status=1/FAILURE
Oct 28 10:42:56 kubeadm-node1.example.com systemd[1]: kubelet.service: Failed with result 'exit-code'.
[root@kubeadm-node1 ~]#
```

## About flannel and required network plugins:
On newer versions of Fedora (e.g. 33+), the CNI plugins are installed in `/usr/libexec/cni` instead of `/opt/cni/bin`. However, Flannel  - installed later in this document - still expects them in `/opt/cni/bin/`. This seems to be hard-coded in flannel, so the only solution is to copy the CNI plugin files from  `/usr/libexec/cni/` to `/opt/cni/bin/` . If you do this step now, it will save you from some pain later. This is because in the next section we will clone this VM and make two more copies of it as two worker nodes. Doing the CNI plugin copy now will save you from doing it for each VM later.


```
mkdir -p /opt/cni/bin

cp /usr/libexec/cni/*  /opt/cni/bin/
```

More detailed explanation for this will come later when we install Flannel.

------

## Pit stop!
**Note:** If this was a VM you were setting up, and you want to clone the VM and make two more copies of this VM (as worker nodes), now is the time to do that. The steps would be:

* Shutdown kubeadm-node1 
* Make two clones of it 
* Adjust hostname and network information (IP address) in each node
* Bring up all three nodes

------

## Setup master node:

SSH to the master node (node1) - as root, and run `kubeadm init` command. Also make a note of entire output of the `kubeadm init` command. This will be useful later. 

What `kubeadm init` does is, it goes through a series of phases, the first one being **pre-flight checks** , and last one being **installing add-ons**. These phases are shown in square brackets in the beginning of each line of the output generated by the `kubeadm init` command. Most notably, these phases are:

* Pull container images for kubernetes *control plane* (etcd, API server, Controller Manager, Scheduler)
* Setup kublet as systemd service
* Generate self signed SSL certificates for all kubernetes components
* Generate necessary kubeconfig files for components of the *control plane*, and the admin user, to be able to access API server with credentials, certificates, etc
* Create and deploy **static pod** definitions of the *control plane* components directly through **kubelet**. You will see (later) that etcd, api-server, controller-manager and scheduler are deployed as plain pods, without being part of any replication-controller, deployment, or daemon-set. These pods are setup using *host* networking.
* Create **kubelet** configuration as *config-map* in the **kube-system** namespace - as soon as the control plane boots up and reports healthy
* Mark the master as **master**  by adding labels and taints
* Create bootstrap tokens and RBAC rules so various components are able to access each other and do certain things, like automatic approval of CSRs, etc
* Install CoreDNS addon by creating it as a *deployment* - using *pod* networking
* Install kube-proxy addon by creating it as a *daemon-set* - using *host* networking

Complete detail of all these phases can be found here: [https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/](https://kubernetes.io/docs/reference/setup-tools/kubeadm/implementation-details/)


OK, enought talk! Lets initialize kubeadm!

```
kubeadm init \
  --pod-network-cidr "10.200.0.0/16" \
  --service-cidr "10.32.0.0/16"
```

To initialize specific version of kubernetes, use extra switch `--kubernetes-version`:
```
kubeadm init \
  --pod-network-cidr "10.200.0.0/16" \
  --service-cidr "10.32.0.0/16" \
  --kubernetes-version "v1.21.6"
```

If you already installed a specific version of `kubeadm`, `kubelet` and `kubectl`, then specifying it here is not very important. Kubeadm will detect this and will setup kubernetes cluster with that specific version. 

**Note:** You can skip `--pod-network-cidr` and `--service-cidr` . The default for pod-network is nothing - actually depends on the CNI plugin you will be using; but if you plan to use flannel, **and** want to use flannel's *default configuration*, then you must pass `--pod-network-cidr "10.244.0.0/16"` to the `kubeadm init` command . The default for `--service-cidr` is: `10.96.0.0/12`. 


If you docker version is unsupported/certified by `kubeadm`, you can use the `--ignore-preflight-errors="SystemVerification"` switch in the command below.


```
kubeadm init  \
  --pod-network-cidr "10.200.0.0/16" \
  --service-cidr "10.32.0.0/16" \
  --kubernetes-version "v1.21.6" \
  --ignore-preflight-errors="SystemVerification"
```

**Notes:** 
* The `--kubernetes-version` switch expects a semantic version, and is different in *appearance* from it's `yum` counterpart, even though it looks the same. 
* The exact (semantic) versions of Kubernetes can be found from the URL: [https://github.com/kubernetes/kubernetes/tags](https://github.com/kubernetes/kubernetes/tags)

```
kubeadm init \
  --pod-network-cidr "10.200.0.0/16" \
  --service-cidr "10.32.0.0/16" \
  --kubernetes-version "service-cidr" \
  --ignore-preflight-errors="SystemVerification"
```

If you do not use `--kubernetes-version "<string>"`, then one of two things will happen:
* If `kubeadm` itself is the latest version, then the **latest** version of Kubernetes will be installed.
* If `kubeadm` itself is a specific version (e.g. **`1.21.6-0`**), then even though `kubeadm` will detect a newer (remote) version, it will still install/setup the corresponding version of kubernetes. In our example case, **version `v1.21.6`** of Kubernetes will be installed.

```
[root@kubeadm-node1 ~]# kubeadm init \
  --pod-network-cidr "10.200.0.0/16" \
  --service-cidr "10.32.0.0/16"
I1028 11:34:21.490312    1005 version.go:254] remote version is much newer: v1.22.3; falling back to: stable-1.21
[init] Using Kubernetes version: v1.21.6
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubeadm-node1.example.com kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.32.0.1 10.240.0.31]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kubeadm-node1.example.com localhost] and IPs [10.240.0.31 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kubeadm-node1.example.com localhost] and IPs [10.240.0.31 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 16.503749 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.21" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node kubeadm-node1.example.com as control-plane by adding the labels: [node-role.kubernetes.io/master(deprecated) node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node kubeadm-node1.example.com as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: w1nvi9.l4c7jfjw69pp6zl4
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 10.240.0.31:6443 --token w1nvi9.l4c7jfjw69pp6zl4 \
	--discovery-token-ca-cert-hash sha256:394d169c4d594ad6110ce7c093fca65b49d598a058085dd9a36ca36037cfd09a 
[root@kubeadm-node1 ~]#
```

### Optional - Setup a normal user:
This step is completely optional. You can operate this kubernetes cluster while logged in as `root` on the master node, or by using a normal user account on master node; or copy/import/merge the `/etc/kubernetes/admin.conf` from master node to `.kube/config` your main/local/work/home computer and use `kubectl` from that computer.

Setup a user `student` on master node to use kubectl, and use instructions from the output of `kubeadm init` command above. 

```
[root@kubeadm-node1 ~]# su - student

[student@kubeadm-node1 ~]$ mkdir -p $HOME/.kube

[student@kubeadm-node1 ~]$ sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config

We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.

[sudo] password for student: 

[student@kubeadm-node1 ~]$ sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

### Check connectivity with your cluster:

**Note:**  `kubectl get componentstatuses` does not work on kubernetes `1.19+`. The output below is from an older version of kubernetes, and kept for record / comparison. 

```
[student@kubeadm-node1 ~]$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
[student@kubeadm-node1 ~]$ 


[student@kubeadm-node1 ~]$ kubectl get nodes
NAME            STATUS     ROLES    AGE   VERSION
kubeadm-node1   NotReady   master   12m   v1.12.2
[student@kubeadm-node1 ~]$ 
```

OK, we have connectivity! 

Since this is my personal test cluster, I can just skip the "regular user" step, and simply connect as `root` and use the cluster:

```
export KUBECONFIG=/etc/kubernetes/admin.conf
```
(and remember to add it to `~/.bashrc` and/or `~/.bash_profile`)

OR

```
cp /etc/kubernetes/admin.conf /root/.kube/config
```


As mentioned above, `kubectl get componentstatuses` does not work anymore in kubernetes 1.19+ . That is why you see **"Unhealthy"** as "STATUS" for **"controller-manager"** and **"scheduler"**. In newer versions of Kubernetes, you should not bother with `get componentstatuses`.

```
[root@kubeadm-node1 ~]# kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
etcd-0               Healthy     {"health":"true"}                                                                             
[root@kubeadm-node1 ~]# 
```

The master node below is "NotReady", which is normal/expected at this point. This is because CoreDNS is not running yet.
```
[root@kubeadm-node1 ~]# kubectl get nodes
NAME            STATUS     ROLES                  AGE   VERSION
kubeadm-node1   NotReady   control-plane,master   10m   v1.20.2
[root@kubeadm-node1 ~]# 
```


CoreDNS is not running, because pod-network is not enabled yet:
```
[root@kubeadm-node1 ~]# kubectl --namespace=kube-system get pods
NAME                                                READY   STATUS    RESTARTS   AGE
coredns-558bd4d5db-k94vv                            0/1     Pending   0          45m
coredns-558bd4d5db-pmp9z                            0/1     Pending   0          45m
etcd-kubeadm-node1.example.com                      1/1     Running   0          45m
kube-apiserver-kubeadm-node1.example.com            1/1     Running   0          45m
kube-controller-manager-kubeadm-node1.example.com   1/1     Running   0          45m
kube-proxy-sgb9d                                    1/1     Running   0          45m
kube-scheduler-kubeadm-node1.example.com            1/1     Running   0          45m
[root@kubeadm-node1 ~]# 
```

We can see that `kubelet` cannot find pod-network yet, because a CNI plugin is not yet installed/setup:

```
[root@kubeadm-node1 ~]# systemctl status kubelet --no-pager -l
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Thu 2021-10-28 11:35:39 CEST; 52min ago
       Docs: https://kubernetes.io/docs/
   Main PID: 2923 (kubelet)
      Tasks: 15 (limit: 2316)
     Memory: 43.6M
        CPU: 1min 57.719s
     CGroup: /system.slice/kubelet.service
             └─2923 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.4.1

Oct 28 12:27:41 kubeadm-node1.example.com kubelet[2923]: E1028 12:27:41.265038    2923 kubelet.go:2211] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"

Oct 28 12:27:46 kubeadm-node1.example.com kubelet[2923]: I1028 12:27:46.022458    2923 cni.go:204] "Error validating CNI config list" configList="{\n  \"name\": \"cbr0\",\n  \"cniVersion\": \"0.3.1\",\n  \"plugins\": [\n    {\n      \"type\": \"flannel\",\n      \"delegate\": {\n        \"hairpinMode\": true,\n        \"isDefaultGateway\": true\n      }\n    },\n    {\n      \"type\": \"portmap\",\n      \"capabilities\": {\n        \"portMappings\": true\n      }\n    }\n  ]\n}\n" err="[failed to find plugin \"portmap\" in path [/opt/cni/bin]]"

Oct 28 12:27:46 kubeadm-node1.example.com kubelet[2923]: I1028 12:27:46.023621    2923 cni.go:239] "Unable to update cni config" err="no valid networks found in /etc/cni/net.d"
```

------

## Install pod network:
At this time, if you check cluster health, you will see **Master** as **NotReady**. It is because a pod network is not yet deployed on the cluster. You must install a pod-network add-on so that your pods can communicate with each other. Use one of the network-addons listed in the Networking section at [https://kubernetes.io/docs/concepts/cluster-administration/addons/](https://kubernetes.io/docs/concepts/cluster-administration/addons/). Flannel is the easiest. Others can be used too.

**Important:** The network must be deployed before any applications. Also, CoreDNS will not start up before a network is installed. `kubeadm` only supports Container Network Interface (CNI) based networks (and does not support `kubenet`).

**Repeat: Kubeadm DOES NOT support kubenet** 


**Note:** In Kubeadm 1.19+ running flannel is not that straightforward anymore. 


## Install flannel CNI plugin/addon:

Flannel is simple CNI plugin and is focused only on networking. For network policies, etc, you can use other projects such as Calico.

**Notes:** 
* If you want to use default configuration for flannel - provided by flannel - then for flannel to work correctly, you must pass `--pod-network-cidr=10.244.0.0/16` to `kubeadm init`.
* If you want to use a `pod-network-cidr` of your own choice - which we did in the beginning of this document, then you must first modify the `kube-flannel.yaml` file before using it, with the pod-network of your choice (`10.200.0.0/16`).

Also, set `/proc/sys/net/bridge/bridge-nf-call-iptables` to `1` by running `sysctl net.bridge.bridge-nf-call-iptables=1` to pass bridged IPv4 traffic to iptables’ chains. This is a requirement for some CNI plugins to work. Setup this configuration in the `/etc/sysctl.conf` file to make this permanent. (This is already done in the beginning of this document, during OS setup).

Check [https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md](https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md) for more information about flannel.

Download the flannel.yaml file and adjust the pod network with the one we chose (`10.200.0.0/16`) 

```
[root@kubeadm-node1 ~]# wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml


Saving to: ‘kube-flannel.yml’

[root@kubeadm-node1 ~]#  grep 10.244 kube-flannel.yml 
      "Network": "10.244.0.0/16",
 

[root@kubeadm-node1 ~]#  sed -i 's/10\.244/10\.200/g' kube-flannel.yml 

[root@kubeadm-node1 ~]#  grep 10.244 kube-flannel.yml 
(no output, means no-match, good!)

[root@kubeadm-node1 ~]#  grep 10.200 kube-flannel.yml 
      "Network": "10.200.0.0/16",

(good!)
```



Now, apply this yaml file to the cluster, so the pod network could come up. Flannel is setup as a *daemon-set* and uses *host* networking of the node.

```
[root@kubeadm-node1 ~]#  kubectl apply -f kube-flannel.yml
Warning: policy/v1beta1 PodSecurityPolicy is deprecated in v1.21+, unavailable in v1.25+
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
[root@kubeadm-node1 ~]#  
```


Check node and pod status - which still show that they are not ready:

```
[root@kubeadm-node1 ~]# kubectl get nodes
NAME           STATUS     ROLES                  AGE   VERSION
kubeadm-node1   NotReady   control-plane,master   11h   v1.20.2
[root@kubeadm-node1 ~]# 
```

```
[root@kubeadm-node1 ~]# kubectl get pods --namespace=kube-system
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-2h4xc                0/1     Pending   0          11h
coredns-74ff55c5b-5lcb7                0/1     Pending   0          11h
etcd-kubeadm-node1                      1/1     Running   0          11h
kube-apiserver-kubeadm-node1            1/1     Running   1          11h
kube-controller-manager-kubeadm-node1   1/1     Running   1          11h
kube-flannel-ds-ccqtj                  1/1     Running   0          25m
kube-proxy-95j8m                       1/1     Running   0          11h
kube-scheduler-kubeadm-node1            1/1     Running   1          11h
[root@kubeadm-node1 ~]# 
```

Check `kubelet` for problems:

```
[root@kubeadm-node1 ~]# systemctl status kubelet --no-pager -l
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Thu 2021-10-28 11:35:39 CEST; 52min ago
       Docs: https://kubernetes.io/docs/
   Main PID: 2923 (kubelet)
      Tasks: 15 (limit: 2316)
     Memory: 43.6M
        CPU: 1min 57.719s
     CGroup: /system.slice/kubelet.service
             └─2923 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.4.1

Oct 28 12:27:41 kubeadm-node1.example.com kubelet[2923]: E1028 12:27:41.265038    2923 kubelet.go:2211] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"

Oct 28 12:27:46 kubeadm-node1.example.com kubelet[2923]: I1028 12:27:46.022458    2923 cni.go:204] "Error validating CNI config list" configList="{\n  \"name\": \"cbr0\",\n  \"cniVersion\": \"0.3.1\",\n  \"plugins\": [\n    {\n      \"type\": \"flannel\",\n      \"delegate\": {\n        \"hairpinMode\": true,\n        \"isDefaultGateway\": true\n      }\n    },\n    {\n      \"type\": \"portmap\",\n      \"capabilities\": {\n        \"portMappings\": true\n      }\n    }\n  ]\n}\n" err="[failed to find plugin \"portmap\" in path [/opt/cni/bin]]"

Oct 28 12:27:46 kubeadm-node1.example.com kubelet[2923]: I1028 12:27:46.023621    2923 cni.go:239] "Unable to update cni config" err="no valid networks found in /etc/cni/net.d"
```

**Notice:** `err="[failed to find plugin \"portmap\" in path [/opt/cni/bin]]`


**Note:** On newer versions of Fedora (e.g. 33+), the CNI plugins are installed in `/usr/libexec/cni` instead of `/opt/cni/bin`. However Flannel still expects them in `/opt/cni/bin/`. This seems to be hard-coded in flannel, so the only solution is to copy the CNI plugin files from  `/usr/libexec/cni/` to `/opt/cni/bin/` .

**Note:** Do these steps on all nodes, otherwise they will remain in the "NotReady" state.

```
[root@kubeadm-node1 ~]# ls  -lh /usr/libexec/cni/
total 60M
-rwxr-xr-x. 1 root root 3.4M Sep  8 03:10 bandwidth
-rwxr-xr-x. 1 root root 3.7M Sep  8 03:10 bridge
-rwxr-xr-x. 1 root root 8.5M Sep  8 03:10 dhcp
-rwxr-xr-x. 1 root root 3.9M Sep  8 03:10 firewall
-rwxr-xr-x. 1 root root 3.4M Sep  8 03:10 host-device
-rwxr-xr-x. 1 root root 2.8M Sep  8 03:10 host-local
-rwxr-xr-x. 1 root root 3.5M Sep  8 03:10 ipvlan
-rwxr-xr-x. 1 root root 2.9M Sep  8 03:10 loopback
-rwxr-xr-x. 1 root root 3.6M Sep  8 03:10 macvlan
-rwxr-xr-x. 1 root root 3.3M Sep  8 03:10 portmap
-rwxr-xr-x. 1 root root 3.7M Sep  8 03:10 ptp
-rwxr-xr-x. 1 root root 2.5M Sep  8 03:10 sample
-rwxr-xr-x. 1 root root 3.1M Sep  8 03:10 sbr
-rwxr-xr-x. 1 root root 2.5M Sep  8 03:10 static
-rwxr-xr-x. 1 root root 3.1M Sep  8 03:10 tuning
-rwxr-xr-x. 1 root root 3.5M Sep  8 03:10 vlan
-rwxr-xr-x. 1 root root 3.1M Sep  8 03:10 vrf
[root@kubeadm-node1 ~]# 
```


```
[root@kubeadm-node1 ~]# mkdir /opt/cni/bin -p

[root@kubeadm-node1 ~]# cp /usr/libexec/cni/*  /opt/cni/bin/
```

As soon as you copy the plugins, within few seconds, kubelet will detect flannel plugins, and will start flannel properly.

```
[root@kubeadm-node1 ~]# systemctl status kubelet --no-pager -l
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Thu 2021-10-28 11:35:39 CEST; 57min ago
       Docs: https://kubernetes.io/docs/
   Main PID: 2923 (kubelet)
      Tasks: 15 (limit: 2316)
     Memory: 58.2M
        CPU: 2min 11.600s
     CGroup: /system.slice/kubelet.service
             └─2923 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/config.yaml --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.4.1

Oct 28 12:32:57 kubeadm-node1.example.com kubelet[2923]: E1028 12:32:57.758756    2923 kubelet.go:2211] "Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:docker: network plugin is not ready: cni config uninitialized"

Oct 28 12:33:16 kubeadm-node1.example.com kubelet[2923]: I1028 12:33:16.885574    2923 topology_manager.go:187] "Topology Admit Handler"

Oct 28 12:33:16 kubeadm-node1.example.com kubelet[2923]: I1028 12:33:16.887670    2923 topology_manager.go:187] "Topology Admit Handler"

Oct 28 12:33:16 kubeadm-node1.example.com kubelet[2923]: I1028 12:33:16.923157    2923 reconciler.go:224] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-b4fjl\" (UniqueName: \"kubernetes.io/projected/4e1eccb1-03b6-4faf-848d-235ef1221547-kube-api-access-b4fjl\") pod \"coredns-558bd4d5db-pmp9z\" (UID: \"4e1eccb1-03b6-4faf-848d-235ef1221547\") "

Oct 28 12:33:16 kubeadm-node1.example.com kubelet[2923]: I1028 12:33:16.923205    2923 reconciler.go:224] "operationExecutor.VerifyControllerAttachedVolume started for volume \"config-volume\" (UniqueName: \"kubernetes.io/configmap/4e1eccb1-03b6-4faf-848d-235ef1221547-config-volume\") pod \"coredns-558bd4d5db-pmp9z\" (UID: \"4e1eccb1-03b6-4faf-848d-235ef1221547\") "

Oct 28 12:33:16 kubeadm-node1.example.com kubelet[2923]: I1028 12:33:16.923228    2923 reconciler.go:224] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kube-api-access-dncc8\" (UniqueName: \"kubernetes.io/projected/74985536-d85e-4175-aecb-834412b1d54d-kube-api-access-dncc8\") pod \"coredns-558bd4d5db-k94vv\" (UID: \"74985536-d85e-4175-aecb-834412b1d54d\") "

Oct 28 12:33:16 kubeadm-node1.example.com kubelet[2923]: I1028 12:33:16.923246    2923 reconciler.go:224] "operationExecutor.VerifyControllerAttachedVolume started for volume \"config-volume\" (UniqueName: \"kubernetes.io/configmap/74985536-d85e-4175-aecb-834412b1d54d-config-volume\") pod \"coredns-558bd4d5db-k94vv\" (UID: \"74985536-d85e-4175-aecb-834412b1d54d\") "

Oct 28 12:33:17 kubeadm-node1.example.com kubelet[2923]: map[string]interface {}{"cniVersion":"0.3.1", "hairpinMode":true, "ipMasq":false, "ipam":map[string]interface {}{"ranges":[][]map[string]interface {}{[]map[string]interface {}{map[string]interface {}{"subnet":"10.200.0.0/24"}}}, "routes":[]types.Route{types.Route{Dst:net.IPNet{IP:net.IP{0xa, 0xc8, 0x0, 0x0}, Mask:net.IPMask{0xff, 0xff, 0x0, 0x0}}, GW:net.IP(nil)}}, "type":"host-local"}, "isDefaultGateway":true, "isGateway":true, "mtu":(*uint)(0xc000016908), "name":"cbr0", "type":"bridge"}

Oct 28 12:33:17 kubeadm-node1.example.com kubelet[2923]: {"cniVersion":"0.3.1","hairpinMode":true,"ipMasq":false,"ipam":{"ranges":[[{"subnet":"10.200.0.0/24"}]],"routes":[{"dst":"10.200.0.0/16"}],"type":"host-local"},"isDefaultGateway":true,"isGateway":true,"mtu":1450,"name":"cbr0","type":"bridge"}

Oct 28 12:33:17 kubeadm-node1.example.com kubelet[2923]: map[string]interface {}{"cniVersion":"0.3.1", "hairpinMode":true, "ipMasq":false, "ipam":map[string]interface {}{"ranges":[][]map[string]interface {}{[]map[string]interface {}{map[string]interface {}{"subnet":"10.200.0.0/24"}}}, "routes":[]types.Route{types.Route{Dst:net.IPNet{IP:net.IP{0xa, 0xc8, 0x0, 0x0}, Mask:net.IPMask{0xff, 0xff, 0x0, 0x0}}, GW:net.IP(nil)}}, "type":"host-local"}, "isDefaultGateway":true, "isGateway":true, "mtu":(*uint)(0xc0000a88d8), "name":"cbr0", "type":"bridge"}
[root@kubeadm-node1 ~]# 
```

The node will become **Ready**, and CoreDNS will also start.

```
[root@kubeadm-node1 ~]# kubectl --namespace=kube-system get nodes
NAME                        STATUS   ROLES                  AGE   VERSION
kubeadm-node1.example.com   Ready    control-plane,master   58m   v1.21.6
```

```
[root@kubeadm-node1 ~]# kubectl --namespace=kube-system get pods
NAME                                                READY   STATUS    RESTARTS   AGE
coredns-558bd4d5db-k94vv                            1/1     Running   0          58m
coredns-558bd4d5db-pmp9z                            1/1     Running   0          58m
etcd-kubeadm-node1.example.com                      1/1     Running   0          58m
kube-apiserver-kubeadm-node1.example.com            1/1     Running   0          58m
kube-controller-manager-kubeadm-node1.example.com   1/1     Running   0          58m
kube-flannel-ds-7m9jc                               1/1     Running   0          11m
kube-proxy-sgb9d                                    1/1     Running   0          58m
kube-scheduler-kubeadm-node1.example.com            1/1     Running   0          58m
[root@kubeadm-node1 ~]# 
```

The Flannel podnetwork is now up. Good!


### Check cluster health:

On older Kubernetes versions (< 1.19):
```
[student@kubeadm-node1 ~]$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
[student@kubeadm-node1 ~]$ 
```

On newer Kubernetes version (1.19+), you will get an **"Unhealthy"** status against `get componentstatuses` command. The real check on newer clusters is that **the node is "Ready"**.

```
[root@kubeadm-node1 ~]# kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
etcd-0               Healthy     {"health":"true"}                                                                             
[root@kubeadm-node1 ~]# 
```

This is mentioned [here:](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.19.md#deprecation):

> Kube-apiserver: the componentstatus API is deprecated. This API provided status of etcd, kube-scheduler, and kube-controller-manager components, but only worked when those components were local to the API server, and when kube-scheduler and kube-controller-manager exposed unsecured health endpoints. Instead of this API, etcd health is included in the kube-apiserver health check and kube-scheduler/kube-controller-manager health checks can be made directly against those components' health endpoints. (#93570, @liggitt) [SIG API Machinery, Apps and Cluster Lifecycle]


### Optional - Remove taints on master:
Now, remove the taints on the master so that you can schedule pods on it. If you want to keep it a dedicated master node, and plan to run your applications only on worker nodes (added later), then you can skip this step.

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

```
[root@kubeadm-node1 ~]# kubectl taint nodes --all node-role.kubernetes.io/master-
node/kubeadm-node1 untainted
[root@kubeadm-node1 ~]# 
```

```
[root@kubeadm-node1 ~]# kubectl get nodes
NAME            STATUS   ROLES                  AGE   VERSION
kubeadm-node1   Ready    control-plane,master   35m   v1.20.2
[root@kubeadm-node1 ~]# 
```

```
[root@kubeadm-node1 ~]# kubectl  get nodes -o wide
NAME            STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                     KERNEL-VERSION           CONTAINER-RUNTIME
kubeadm-node1   Ready    control-plane,master   36m   v1.20.2   10.240.0.31   <none>        Fedora 33 (Server Edition)   5.10.7-200.fc33.x86_64   docker://20.10.2
[root@kubeadm-node1 ~]# 
```


```
[root@kubeadm-node1 ~]# kubectl get pods --all-namespaces=true -o wide
NAMESPACE         NAME                                      READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
calico-system     calico-kube-controllers-b87bd7f6f-k2hdv   1/1     Running   0          18m   10.200.98.193   kubeadm-node1   <none>           <none>
calico-system     calico-node-lxrkq                         1/1     Running   0          18m   10.240.0.31     kubeadm-node1   <none>           <none>
calico-system     calico-typha-588c975bc5-wkfcw             1/1     Running   0          18m   10.240.0.31     kubeadm-node1   <none>           <none>
kube-system       coredns-74ff55c5b-9nq69                   1/1     Running   0          37m   10.200.98.194   kubeadm-node1   <none>           <none>
kube-system       coredns-74ff55c5b-xdzxr                   1/1     Running   0          37m   10.200.98.195   kubeadm-node1   <none>           <none>
kube-system       etcd-kubeadm-node1                        1/1     Running   0          37m   10.240.0.31     kubeadm-node1   <none>           <none>
kube-system       kube-apiserver-kubeadm-node1              1/1     Running   0          37m   10.240.0.31     kubeadm-node1   <none>           <none>
kube-system       kube-controller-manager-kubeadm-node1     1/1     Running   0          37m   10.240.0.31     kubeadm-node1   <none>           <none>
kube-system       kube-proxy-nn29f                          1/1     Running   0          37m   10.240.0.31     kubeadm-node1   <none>           <none>
kube-system       kube-scheduler-kubeadm-node1              1/1     Running   0          37m   10.240.0.31     kubeadm-node1   <none>           <none>
tigera-operator   tigera-operator-657cc89589-5rl4z          1/1     Running   0          18m   10.240.0.31     kubeadm-node1   <none>           <none>
[root@kubeadm-node1 ~]# 
```


Let's see if we can run a simple nginx container, in the `default` name-space. (You do not need to explicitly mention `default` in the command line - duh!)

```
[root@kubeadm-node1 ~]# kubectl run nginx --image=nginx:alpine
pod/nginx created

[root@kubeadm-node1 ~]# kubectl get pods
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          12s
[root@kubeadm-node1 ~]# 
```

### When the master not is not untainted:
If the pod's status is **Pending**, not **Creating Container**, nor **CrashLooping** , then you forgot to **"untaint"** the master node. Or, add another node in **"worker mode"**
```
[student@kubeadm-node1 ~]$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-64d9675947-tr5mk   0/1     Pending   0          50s
[student@kubeadm-node1 ~]$ 
```

The reason for nginx container being pending is that our Node1 is actually master node, and it has a **taint** set to **NoSchedule** . You can check for it using the `kubectl describe node` command:

```
[student@kubeadm-node1 ~]$ kubectl describe node kubeadm-node1 | grep Taints
Taints:             node-role.kubernetes.io/master:NoSchedule
[student@kubeadm-node1 ~]$ 
```

After this is applied, pods will/can be scheduled on master node too. There is another way of running pods on master node, by setting *toleration* in the `yaml` file that creates a pod/deployment/etc, but that is beyond the scope of this document.

Since this guide is about running a multi-node kubeadm based kubernetes cluster, we will simply add another node to the cluster.

------

## Add / join a node:

To join a new node to the cluster, you need to satisfy the following conditions first:
* You have SSH / root access to the node
* You have already setup a container runtime, such as Docker on it, and it is in working order
* You have downloaded and installed kubeadm, kubelet and kubectl on it

When you executed `kubeadm init`, you were given instructions for the nodes you want to join to this cluster. 

```
You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.240.0.31:6443 --token w1nvi9.l4c7jfjw69pp6zl4 \
        --discovery-token-ca-cert-hash sha256:394d169c4d594ad6110ce7c093fca65b49d598a058085dd9a36ca36037cfd09a
```

So we can do that now. Please note that the token is only valid for 24 hours after the initialization of the cluster. If you are trying to join a node to the cluster after 24 hours have passed, you need to generate a new token. That can be done simply by running `kubeadm token create` on the master node. Then use the new token in the `kubeadm join` command.


```
[root@kubeadm-node2 ~]# kubeadm join 10.240.0.31:6443 --token w1nvi9.l4c7jfjw69pp6zl4 \
        --discovery-token-ca-cert-hash sha256:394d169c4d594ad6110ce7c093fca65b49d598a058085dd9a36ca36037cfd09a 
[preflight] Running pre-flight checks
[preflight] Reading configuration from the cluster...
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -o yaml'
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.

[root@kubeadm-node2 ~]# 
```

If you check the list of nodes now, you should be able to see "node2" as "Ready" too. If you see node2 as "NotReady", read next section.

```
[root@kubeadm-node1 ~]# kubectl get nodes
NAME                        STATUS   ROLES                  AGE   VERSION
kubeadm-node1.example.com   Ready    control-plane,master   88m   v1.21.6
kubeadm-node2.example.com   Ready    <none>                 19m   v1.21.6

[root@kubeadm-node1 ~]# kubectl get nodes -o wide
NAME                        STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                     KERNEL-VERSION            CONTAINER-RUNTIME
kubeadm-node1.example.com   Ready    control-plane,master   88m   v1.21.6   10.240.0.31   <none>        Fedora 34 (Server Edition)   5.14.13-200.fc34.x86_64   docker://20.10.10
kubeadm-node2.example.com   Ready    <none>                 20m   v1.21.6   10.240.0.32   <none>        Fedora 34 (Server Edition)   5.14.13-200.fc34.x86_64   docker://20.10.10
[root@kubeadm-node1 ~]#  
```

Repeat above procedure to join more worker nodes.


### What if the newly joined node is still "NotReady":

The problem is that flannel will be expecting dependent network plugins in `/opt/cni/bin` on each node. If it does not find it then kubelet is unable to start flannel on the new node. So on Fedora 33+, repeat the copy operation.

```
cp /usr/libexec/cni/*  /opt/cni/bin/
```



If (earlier) you chose not to *"untaint"* the master node, and your nginx pod was in Pending state, then now the scheduler should automatically assign nginx pod to this newly added **"Node2"**.

```
[student@kubeadm-node1 ~]$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-64d9675947-tr5mk   1/1     Running   0          31m

[student@kubeadm-node1 ~]$ kubectl get pods -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE            NOMINATED NODE
nginx-64d9675947-tr5mk   1/1     Running   0          31m   10.200.1.2   kubeadm-node2   <none>
[student@kubeadm-node1 ~]$ 
```

Hurray! It works!


Here is a more detailed, *complete picture*, of the cluster when everything is working properly. Notice, only `kube-system` namespace is queried in the command below.
```
[root@kubeadm-node1 ~]# kubectl --all-namespaces=true get all
NAMESPACE     NAME                                                    READY   STATUS    RESTARTS   AGE
default       pod/multitool                                           1/1     Running   1          76m
kube-system   pod/coredns-558bd4d5db-k94vv                            1/1     Running   1          141m
kube-system   pod/coredns-558bd4d5db-pmp9z                            1/1     Running   1          141m
kube-system   pod/etcd-kubeadm-node1.example.com                      1/1     Running   1          141m
kube-system   pod/kube-apiserver-kubeadm-node1.example.com            1/1     Running   1          141m
kube-system   pod/kube-controller-manager-kubeadm-node1.example.com   1/1     Running   1          141m
kube-system   pod/kube-flannel-ds-7m9jc                               1/1     Running   1          93m
kube-system   pod/kube-flannel-ds-vnpq6                               1/1     Running   1          73m
kube-system   pod/kube-proxy-rfwcc                                    1/1     Running   1          73m
kube-system   pod/kube-proxy-sgb9d                                    1/1     Running   1          141m
kube-system   pod/kube-scheduler-kubeadm-node1.example.com            1/1     Running   1          141m

NAMESPACE     NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes   ClusterIP   10.32.0.1    <none>        443/TCP                  141m
kube-system   service/kube-dns     ClusterIP   10.32.0.10   <none>        53/UDP,53/TCP,9153/TCP   141m

NAMESPACE     NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/kube-flannel-ds   2         2         2       2            2           <none>                   93m
kube-system   daemonset.apps/kube-proxy        2         2         2       2            2           kubernetes.io/os=linux   141m

NAMESPACE     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns   2/2     2            2           141m

NAMESPACE     NAME                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-558bd4d5db   2         2         2       141m
[root@kubeadm-node1 ~]# 
```

------

## Accessing your cluster from machines other than the master node:

So far, we are able to talk to our Kubernetes cluster from node1, as user student. It is because that is where we have configured our `.kube/config` which is used by `kubectl` commands. To be able to access the cluster from some other computer, such as your work computer, etc, you need to copy the administrator kubeconfig file from your master node to your computer like this:

```
mkdir ~/.kube
scp student@<master-node-ip>:/home/student/.kube/config  ~/.kube/kubeadm-cluster.conf
kubectl --kubeconfig ~/.kube/kubeadm-cluster.conf get nodes
```

It might be a hassle to include `--kubeconfig ~/.kube/kubeadm-cluster.conf` in every invocation of `kubectl` command. You can setup a shell alias, or you can simply save it as `~/.kube/config` on your work computer. 

**WARNING** It is possible that you already have `~/.kube/config` file, which might contain configuration to one or more existing kubernetes clusters, and you don't want to accidentally lose access to those clusters by over-writing it with this `.kube/config` . To be able to retain access to those clusters, and still also be able to use this kubeadm cluster, all from single kubectl, without specifying `--kubeconfig` everytime, you can use the KUBECONFIG environment variable. 

Lets start at master node, where we have a working copy of our `.kube/config` file.
First, we need to fix the default config we got from kubeadm. It is not very helpful in identifying which cluster is it. e.g. look at the following information. It looks very vague:

```
[student@kubeadm-node1 ~]$ kubectl config current-context
kubernetes-admin@kubernetes

[student@kubeadm-node1 ~]$ kubectl config get-clusters
NAME
kubernetes
[student@kubeadm-node1 ~]$
```

With a little `sed` , I changed the `.kube/config` for the student user. 


```
[student@kubeadm-node1 tmp]$ sed -i 's/kubernetes-admin/kubeadm-admin/g' $HOME/.kube/config

[student@kubeadm-node1 tmp]$ sed -i 's/kubernetes/kubeadm-cluster/g' $HOME/.kube/config
```


It now looks like the following:

```
[student@kubeadm-node1 ~]$ kubectl config current-context
kubeadm-admin@kubeadm-cluster

[student@kubeadm-node1 ~]$ kubectl config get-clusters
NAME
kubeadm-cluster
[student@kubeadm-node1 ~]$ 
```

Verify that it works:

```
[student@kubeadm-node1 ~]$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
nginx-64d9675947-nw4kl   1/1     Running   0          3m53s
[student@kubeadm-node1 ~]$ 
```



Now, I copy this file from the master node to my work computer, inside `~/.kube/` directory, saving it as `kubeadm-cluster.conf`. 

```
[kamran@kworkhorse ~]$ scp student@kubeadm-node1:/home/student/.kube/config .kube/kubeadm-cluster.conf
config                                                                                                100% 5455     5.4MB/s   00:00    
[kamran@kworkhorse ~]$ 
```

I verify once that it works:

```
[kamran@kworkhorse ~]$ kubectl --kubeconfig=$HOME/.kube/kubeadm-cluster.conf  get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-64d9675947-nw4kl   1/1       Running   0          11m
[kamran@kworkhorse ~]$
```

Notice that my default `.kube/config` file lists two clusters, one of them is production cluster for a client. I don't want to lose this configuration. 

```
[kamran@kworkhorse ~]$ kubectl config get-contexts
CURRENT   NAME                    CLUSTER                 AUTHINFO   NAMESPACE
*         minikube                minikube                minikube   
          client-gce.k8s.local   client-gce.k8s.local   admin      
[kamran@kworkhorse ~]$ 
```

I have verified that without using `--kubeconfig` with `kubectl`, I can access my previous clusters. By using a specific configuration with `--kubeconfig` , I can access my kubeadm cluster. 

So, is it possible to merge these two configurations? No. Not directly, which is actually a safety feature. Check this article for more detail: [https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable) .

I will solve it by using the KUBECONFIG environment variable as described in the above link. Here is how:

```
[kamran@kworkhorse ~]$ export KUBECONFIG=$KUBECONFIG:$HOME/.kube/config:$HOME/.kube/kubeadm-cluster.conf

[kamran@kworkhorse ~]$ kubectl config get-clusters
NAME
minikube
client-gce.k8s.local
kubeadm-cluster
[kamran@kworkhorse ~]$ 
```

If I do a `get-contexts`  or `get-clusters` now, I get all three clusters! Now, I can simply switch context and use my kubeadm-cluster, without using the `--kubeconfig=...` with each of my `kubectl` command.

```
[kamran@kworkhorse ~]$ kubectl config use-context kubeadm-admin@kubeadm-cluster
Switched to context "kubeadm-admin@kubeadm-cluster".


[kamran@kworkhorse ~]$ kubectl config get-contexts
CURRENT   NAME                            CLUSTER                 AUTHINFO        NAMESPACE
*         kubeadm-admin@kubeadm-cluster   kubeadm-cluster         kubeadm-admin   
          minikube                        minikube                minikube        
          client-gce.k8s.local           client-gce.k8s.local   admin           
[kamran@kworkhorse ~]$
```


```
[kamran@kworkhorse ~]$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-64d9675947-nw4kl   1/1       Running   0          20m
[kamran@kworkhorse ~]$ 
```

Hurray! It works!


Now, I just need to setup this `KUBECONFIG` permanently in my `.bash_profile` file so I don't have to do this every time I login to my computer. Depending on your situation and the way you start your session on your work computer, you may want to add this to any/all of:
* `/etc/profile`  (login shell)
* `~/.bash_profile` (login shell)
* `~/.bash_login` (login shell)
* `~/.profile` (login shell)
* `/etc/bash.bashrc` (non-login shell)
* `~/.bashrc` (non-login shell)

```
[kamran@kworkhorse ~]$ vi .bash_profile 

. . . 

PATH=$PATH:$HOME/.local/bin:$HOME/bin

KUBECONFIG=$KUBECONFIG:$HOME/.kube/config:$HOME/.kube/kubeadm-cluster.conf

export PATH KUBECONFIG
```

**Notes:**
* The `admin.conf` file (`/etc/kubernetes/admin.conf` on master node, copied as `/home/student/.kube/config`) gives the user superuser privileges over the cluster. This file should be used very carefully. For normal users, it’s recommended to generate an unique credential, to which you whitelist privileges. You can do this with the `kubeadm alpha phase kubeconfig user --client-name <client-name>` command. This command will print out a KubeConfig file to STDOUT which you should save to a file and distribute to your user. After that, whitelist privileges by using `kubectl create (cluster)rolebinding`.


### About Azure cli:
The Azure cli command does not understand the possibility of KUBECONFIG variable set to multiple paths of multiple config files. So the `az aks get-credentials` command creates a new file on your computer, with a silly name: `/home/kamran/.kube/config:/home/kamran/.kube/kubeadm-cluster.conf` , and adds it's cluster credentials in that. You have to watch out for that. 

To avoid this problem from happening, first set your KUBECONFIG variable, to just one file, such as `/home/kamran/.kube/config` . Then, run the `az aks get-credentials` command, which will add the credentials of your AKS cluster credentials in that file. Then you change it back to what it was, such as `/home/kamran/.kube/config:/home/kamran/.kube/kubeadm-cluster.conf`.
