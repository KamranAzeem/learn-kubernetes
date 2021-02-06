# KubeAdm based kubernetes cluster

**Note:** This document is updated to use Fedora 33. 

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
* **RAM:** Minimum 1 GB RAM for each node (master+worker); you will only be able to run very few (and very small) containers, like nginx, mysql, etc.
* **CPU:** Minimum 2 CPU for master node; worker nodes can live with single core CPUs
* **Disk:** 4 GB for host OS + 20 GB for storing container images. (no swap)
* **Network - Infrastructure:** A functional virtual/physical network with some usable IP addresses (can be public or private) . This can be on any cloud provider as well. You are free to use any network / ip scheme for yourself. In this guide, it will be `10.240.0.0/24`
* **Network - Pod network:** A network IP range completely separate from other two networks, with subnet mask of `/16` or smaller (e.g. `/12`). This network will be subdivided into subnets later. In this guide it will be `10.200.0.0/16` . Please note that kubeadm does not support kubenet, so we need to use one of the CNI add-ons - such as flannel. By default Flannel sets up a pod network `10.244.0.0/16`, which means that we need to pass this pod network to `kubeadm init` (further below); or, modify the flannel configuration with the pod network of our own choice - before actually applying it blindly. :)
* **Network - Service network:** A network IP range completely separate from other two networks, used by the services. This will be considered a completely virtual network. The default service network configured by kubeadm is `10.96.0.0/12`. In this guide, it will be `10.32.0.0/16`.
* **Firewall:** Disable Firewall (including removing the firewalld package), or open the following ports on each type of node, after the OS installation is complete. 
* **Firewall/ports - Master:** Incoming open (22, 6443, 10250, 10251, 10252, 2379, 2380)
* **Firewall/ports - Worker:** Incoming open (22, 10250, 30000-32767)
* **OS:** Any recent version of Fedora/CentOS/RHEL or Debian based OS. This guide uses Fedora 28
* **Disk Partitioning:** No swap - must disable swap partition during OS installation in order for the kubelet to work properly. See: [https://github.com/kubernetes/kubernetes/issues/53533](https://github.com/kubernetes/kubernetes/issues/53533) for details on why disable swap. Swap may seem a good idea, it is not - on Kubernetes!
* **SELinux:** Disable SELinux / App Armour.
* Should have some sort of DNS for infrastructure network.

## OS setup:

```
[root@kworkhorse lib]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain

 # Virtual Kubernetes cluster
10.240.0.31	kubeadm-node1
10.240.0.32	kubeadm-node2
10.240.0.33	kubeadm-node3
```

```
sudo yum -y remove firewalld

sudo yum -y install ebtables
```

```
cat <<EOF > /etc/modules-load.d/k8s.conf
br_netfilter
EOF
```


Enable the sysctl setting `net.bridge.bridge-nf-call-iptables`
```
cat <<EOF >  /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sysctl --system
```

Increase the number of open files in `limits.conf`:
```
cat <<EOF >>  /etc/security/limits.conf
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

To disable zram, uninstall `zram-generator-defaults` and `zram-generator` packages. Merely stopping/disabling the `swap-create@zram0.service` - and then rebooting - **will not work**.


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
You can select other runtimes too, such as Rocket (rkt). I will use Docker.

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
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

### Install kubeadm, kubelet and kubectl:
On each node, install:
* kubeadm: the command to actually setup / bootstrap the cluster.
* kubelet: the component that runs on all of the machines in your cluster and does things like starting pods and containers.
* kubectl: the command line util to talk to your cluster.


All of the above three pieces of software are available from kubernetes's yum repository. So first, set that up:

```
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
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
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
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


By this time, `kubeadm` is only *installed* - and is not *running*. 

**Note:** kubelet is now set to start. Kubelet will continuously try to start and will fail (crash-loop), because it will wait for kubeadm to tell it what to do. This *"crash-loop"* is expected and normal. After you initialize your master (using kubeadm), the kubelet will run normally.


```
[root@kubeadm-node1 ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: activating (auto-restart) (Result: exit-code) since Fri 2021-01-22 09:27:01 CET; 7s ago
       Docs: https://kubernetes.io/docs/
    Process: 26579 ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS (>
   Main PID: 26579 (code=exited, status=255/EXCEPTION)
        CPU: 83ms

Jan 21 09:27:01 kubeadm-node1 systemd[1]: kubelet.service: Main process exited, code=exited, status=255/EXCEPTION
Jan 21 09:27:01 kubeadm-node1 systemd[1]: kubelet.service: Failed with result 'exit-code'.
```

------

**Note:** If this was a VM you were setting up, and you want to clone the VM and make two more copies, now is the time to do that. The steps would be:

* Shutdown kubeadm-node1 
* Make two clones of it's disk image. 
* Create two more VMs out of those cloned images
* Adjust hostname and network information in each node
* Bring up all three nodes.

------

## Run kubeadm on node1/master to setup the cluster:

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

**Note:** You can skip `--pod-network-cidr` and `--service-cidr` . The default for pod-network is nothing- actually depends on the CNI plugin you will be using; but if you plan to use flannel, **and** want to use flannels *default configuration*, then you must pass `--pod-network-cidr "10.244.0.0/16"` to the `kubeadm init` command . The default for `--service-cidr` is: `10.96.0.0/12`. 


My docker version is `20.10.2` , which is not certified by kubeadm, so I have to use the `--ignore-preflight-errors="SystemVerification"` switch in the command below.

```
kubeadm init  \
  --pod-network-cidr "10.200.0.0/16" \
  --service-cidr "10.32.0.0/16" \
  --ignore-preflight-errors="SystemVerification"
```


```
[root@kubeadm-node1 ~]# kubeadm init  \
  --pod-network-cidr "10.200.0.0/16" \
  --service-cidr "10.32.0.0/16" \
  --ignore-preflight-errors="SystemVerification"

[init] Using Kubernetes version: v1.20.2
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.2. Latest validated version: 19.03
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubeadm-node1 kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local] and IPs [10.32.0.1 10.240.0.31]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [kubeadm-node1 localhost] and IPs [10.240.0.31 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [kubeadm-node1 localhost] and IPs [10.240.0.31 127.0.0.1 ::1]
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
[apiclient] All control plane components are healthy after 15.002731 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.20" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node kubeadm-node1 as control-plane by adding the labels "node-role.kubernetes.io/master=''" and "node-role.kubernetes.io/control-plane='' (deprecated)"
[mark-control-plane] Marking the node kubeadm-node1 as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: axzi13.323q1c2oqtzwpnvt
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

kubeadm join 10.240.0.31:6443 --token axzi13.323q1c2oqtzwpnvt \
    --discovery-token-ca-cert-hash sha256:b4938932a8d2eb82642a1d48b6ae828ef7fcccc3d249480ea8f814378a380e02 
[root@kubeadm-node1 ~]# 


```


Now we setup the user `student` to use kubectl. This is part of the instructions in the `kubeadmin init` output above.

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

Check if you can talk to the cluster/API server. (The output below is from an older version of kubernetes, and kept for record / comparison. See further below to see what happens when you check "componentstatuses" on the latest version (1.19+) of Kubernetes).

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

At least the cluster's components are ok!


Since this is my personal test cluster, I can just skip the "regular user" step, and simply connect as `root` and use the cluster:
```
export KUBECONFIG=/etc/kubernetes/admin.conf
```

**Note:** `kubectl get componentstatuses` does not work anymore in kubernetes 1.19+ . 

```
[root@kubeadm-node1 ~]# kubectl get componentstatuses
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS      MESSAGE                                                                                       ERROR
controller-manager   Unhealthy   Get "http://127.0.0.1:10252/healthz": dial tcp 127.0.0.1:10252: connect: connection refused   
scheduler            Unhealthy   Get "http://127.0.0.1:10251/healthz": dial tcp 127.0.0.1:10251: connect: connection refused   
etcd-0               Healthy     {"health":"true"}                                                                             
[root@kubeadm-node1 ~]# 
```

```
[root@kubeadm-node1 ~]# kubectl get nodes
NAME            STATUS     ROLES                  AGE   VERSION
kubeadm-node1   NotReady   control-plane,master   10m   v1.20.2
[root@kubeadm-node1 ~]# 
```


CoreDNS is not running, because podnetwork is not enabled yet:
```
[root@kubeadm-node1 ~]# kubectl --namespace=kube-system get pods
NAME                                    READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-9nq69                 0/1     Pending   0          8m34s
coredns-74ff55c5b-xdzxr                 0/1     Pending   0          8m34s
etcd-kubeadm-node1                      1/1     Running   0          8m40s
kube-apiserver-kubeadm-node1            1/1     Running   0          8m40s
kube-controller-manager-kubeadm-node1   1/1     Running   0          8m40s
kube-proxy-nn29f                        1/1     Running   0          8m34s
kube-scheduler-kubeadm-node1            1/1     Running   0          8m40s
[root@kubeadm-node1 ~]# 
```

Kubelet cannot find podnetwork yet:
```
[root@kubeadm-node1 ~]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Fri 2021-01-22 10:01:42 CET; 10h ago
       Docs: https://kubernetes.io/docs/
   Main PID: 3977 (kubelet)
      Tasks: 16 (limit: 2323)
     Memory: 49.1M
        CPU: 3min 40.179s
     CGroup: /system.slice/kubelet.service
             └─3977 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet>

Jan 21 09:35:01 kubeadm-node1 kubelet[3977]: E0122 20:43:26.356668    3977 kubelet.go:2163] Container runtime network not ready: Network>
Jan 21 09:35:02 kubeadm-node1 kubelet[3977]: W0122 20:43:27.513623    3977 cni.go:239] Unable to update cni config: no networks found in>
Jan 21 09:35:05 kubeadm-node1 kubelet[3977]: E0122 20:43:31.372514    3977 kubelet.go:2163] Container runtime network not ready: Network>
Jan 21 09:35:09 kubeadm-node1 kubelet[3977]: W0122 20:43:32.515160    3977 cni.go:239] Unable to update cni config: no networks found in>
Jan 21 09:35:11 kubeadm-node1 kubelet[3977]: E0122 20:43:36.398434    3977 kubelet.go:2163] Container runtime network not ready: Network>
Jan 21 09:35:15 kubeadm-node1 kubelet[3977]: W0122 20:43:37.516365    3977 cni.go:239] Unable to update cni config: no networks found in>
Jan 21 09:35:20 kubeadm-node1 kubelet[3977]: E0122 20:43:41.426484    3977 kubelet.go:2163] Container runtime network not ready: Network>
Jan 21 09:35:27 kubeadm-node1 kubelet[3977]: W0122 20:43:42.517187    3977 cni.go:239] Unable to update cni config: no networks found in>
Jan 21 09:35:30 kubeadm-node1 kubelet[3977]: E0122 20:43:46.454019    3977 kubelet.go:2163] Container runtime network not ready: Network>
Jan 21 09:35:35 kubeadm-node1 kubelet[3977]: W0122 20:43:47.517483    3977 cni.go:239] Unable to update cni config: no networks found in>
lines 1-23/23 (END)
```



# Install pod network:
At this time, if you check cluster health, you will see **Master** as **NotReady**. It is because a pod network is not yet deployed on the cluster. You must install a pod network add-on so that your pods can communicate with each other. Use one of the network addons listed in the Networking section at [https://kubernetes.io/docs/concepts/cluster-administration/addons/](https://kubernetes.io/docs/concepts/cluster-administration/addons/). Flannel is the easiest. Others can be used too.

**Important:** The network must be deployed before any applications. Also, CoreDNS will not start up before a network is installed. kubeadm only supports Container Network Interface (CNI) based networks (and does not support kubenet).

**Repeat: Kubeadm DOES NOT support kubenet** 


**Note:** In Kubeadm 1.19+ running flannel is not that straightforward anymore. 


## Install flannel CNI plugin/addon:

**Note:** Flannel is focused on networking. For network policy, other projects such as Calico can be used.

**Note:** For flannel to work correctly, you must pass `--pod-network-cidr=10.244.0.0/16` to `kubeadm init` ; or modify the kube-flannel.yaml to use your own pod network before applying tha yaml with `kubectl`. 

Also, set `/proc/sys/net/bridge/bridge-nf-call-iptables` to `1` by running `sysctl net.bridge.bridge-nf-call-iptables=1` to pass bridged IPv4 traffic to iptables’ chains. This is a requirement for some CNI plugins to work. Setup this configuration in the `/etc/sysctl.conf` file to make this permanent.

Check [https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md](https://github.com/coreos/flannel/blob/master/Documentation/kubernetes.md) for more information about flannel.

Download the flannel.yaml file and adjust the pod network with the one we chose (10.200.0.0/16) 



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
[root@kubeadm-node1 ~]# kubectl apply -f kube-flannel.yml 
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds created
[root@kubeadm-node1 ~]# 
```

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



```
[root@kubeadm-node1 ~]#  systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Fri 2021-01-22 10:01:42 CET; 10h ago
       Docs: https://kubernetes.io/docs/
   Main PID: 3977 (kubelet)
      Tasks: 16 (limit: 2323)
     Memory: 51.2M
        CPU: 4min 20.349s
     CGroup: /system.slice/kubelet.service
             └─3977 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet>

Jan 22 21:00:22 kubeadm-node1 kubelet[3977]:     {
Jan 22 21:00:22 kubeadm-node1 kubelet[3977]:       "type": "portmap",
Jan 22 21:00:22 kubeadm-node1 kubelet[3977]:       "capabilities": {
Jan 22 21:00:22 kubeadm-node1 kubelet[3977]:         "portMappings": true
Jan 22 21:00:22 kubeadm-node1 kubelet[3977]:       }
Jan 22 21:00:22 kubeadm-node1 kubelet[3977]:     }
Jan 22 21:00:22 kubeadm-node1 kubelet[3977]:   ]
Jan 22 21:00:22 kubeadm-node1 kubelet[3977]: }
Jan 22 21:00:22 kubeadm-node1 kubelet[3977]: : [failed to find plugin "flannel" in path [/opt/cni/bin] failed to find plugin "portmap" i>
Jan 22 21:00:22 kubeadm-node1 kubelet[3977]: W0122 21:00:22.710114    3977 cni.go:239] Unable to update cni config: no valid networks fo>
lines 1-23/23 (END)
```

**Note:** On newer versions of Fedora (e.g. 33), the CNI plugins are installed in `/usr/libexec/cni` instead of `/opt/cni/bin`. However Flannel still expects them in `/opt/cni/bin/`. This seems to be hard-coded in flannel, so the only solution is to copy the CNI plugin files from  `/usr/libexec/cni/` to `/opt/cni/bin/`

```
[root@kubeadm-node1 ~]# mkdir /opt/cni/bin -p

[root@kubeadm-node1 ~]# cp /usr/libexec/cni/*  /opt/cni/bin/
```

After few seconds, the node will become ready, and CoreDNS will also start.

```
[root@kubeadm-node1 ~]#  systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Fri 2021-01-22 10:01:42 CET; 11h ago
       Docs: https://kubernetes.io/docs/
   Main PID: 3977 (kubelet)
      Tasks: 16 (limit: 2323)
     Memory: 92.5M
        CPU: 5min 57.856s
     CGroup: /system.slice/kubelet.service
             └─3977 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet>

Jan 22 21:24:43 kubeadm-node1 kubelet[3977]: : [failed to find plugin "flannel" in path [/opt/cni/bin] failed to find plugin "portmap" i>
Jan 22 21:24:43 kubeadm-node1 kubelet[3977]: W0122 21:24:43.532875    3977 cni.go:239] Unable to update cni config: no valid networks fo>
Jan 22 21:24:48 kubeadm-node1 kubelet[3977]: E0122 21:24:48.251605    3977 kubelet.go:2163] Container runtime network not ready: Network>
Jan 22 21:25:09 kubeadm-node1 kubelet[3977]: I0122 21:25:09.388780    3977 topology_manager.go:187] [topologymanager] Topology Admit Han>
Jan 22 21:25:09 kubeadm-node1 kubelet[3977]: I0122 21:25:09.394743    3977 topology_manager.go:187] [topologymanager] Topology Admit Han>
Jan 22 21:25:09 kubeadm-node1 kubelet[3977]: I0122 21:25:09.480109    3977 reconciler.go:224] operationExecutor.VerifyControllerAttached>
Jan 22 21:25:09 kubeadm-node1 kubelet[3977]: I0122 21:25:09.480435    3977 reconciler.go:224] operationExecutor.VerifyControllerAttached>
Jan 22 21:25:09 kubeadm-node1 kubelet[3977]: I0122 21:25:09.480622    3977 reconciler.go:224] operationExecutor.VerifyControllerAttached>
Jan 22 21:25:09 kubeadm-node1 kubelet[3977]: I0122 21:25:09.480807    3977 reconciler.go:224] operationExecutor.VerifyControllerAttached>
Jan 22 21:25:09 kubeadm-node1 kubelet[3977]: W0122 21:25:09.872348    3977 pod_container_deletor.go:79] Container "aafbe5be4bcd161fa4397>
```

```
[root@kubeadm-node1 ~]# kubectl get pods --namespace=kube-system
NAME                                   READY   STATUS    RESTARTS   AGE
coredns-74ff55c5b-2h4xc                1/1     Running   0          11h
coredns-74ff55c5b-5lcb7                1/1     Running   0          11h
etcd-kubeadm-node1                      1/1     Running   0          11h
kube-apiserver-kubeadm-node1            1/1     Running   1          11h
kube-controller-manager-kubeadm-node1   1/1     Running   1          11h
kube-flannel-ds-ccqtj                  1/1     Running   0          28m
kube-proxy-95j8m                       1/1     Running   0          11h
kube-scheduler-kubeadm-node1            1/1     Running   1          11h
```

```
[root@kubeadm-node1 ~]# kubectl get nodes
NAME           STATUS   ROLES                  AGE   VERSION
kubeadm-node1   Ready    control-plane,master   11h   v1.20.2
[root@kubeadm-node1 ~]# 
```

The Flannel podnetwork is now up. Good!


## Check cluster health:

On older Kubernetes versions (< 1.19):
```
[student@kubeadm-node1 ~]$ kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
[student@kubeadm-node1 ~]$ 
```

On newer Kubernetes version (1.19+), you will get an **"Unhealthy"** status:

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


### Remove taints on master:
Now, remove the taints on the master so that you can schedule pods on it. If you want to keep it a dedicated master node, and plan to add worker nodes, then you can skip this step.

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


Let's see if we can run a simple nginx container, in the `default` namespace. (You do not need to explicitly mention default in the command line - duh!)

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

# Add / join a node:

To join a new node to the cluster, you need to satisfy the following conditions first:
* You have SSH / root access to the node
* You have already setup a container runtime, such as Docker on it, and it is in working order
* You have downloaded and installed kubeadm, kubelet and kubectl on it

When you executed `kubeadm init`, you were given a command to run on the nodes you want to join to this cluster. 

```
You can now join any number of machines by running the following on each node
as root:

  kubeadm join 10.240.0.31:6443 --token axzi13.323q1c2oqtzwpnvt \
    --discovery-token-ca-cert-hash sha256:b4938932a8d2eb82642a1d48b6ae828ef7fcccc3d249480ea8f814378a380e02 \
    --ignore-preflight-errors="SystemVerification"
```

So we can do that now. Please note that the token is only valid for 24 hours after the initialization of the cluster. If you are trying to join a node to the cluster after 24 hours have passed, you need to generate a new token. That can be done simply by running `kubeadm token create` on the master node. Then use the new token in the `kubeadm join` command.


```
[root@kubeadm-node2 ~]#   kubeadm join 10.240.0.31:6443 --token axzi13.323q1c2oqtzwpnvt \
>     --discovery-token-ca-cert-hash sha256:b4938932a8d2eb82642a1d48b6ae828ef7fcccc3d249480ea8f814378a380e02 \
>     --ignore-preflight-errors="SystemVerification"
[preflight] Running pre-flight checks
	[WARNING SystemVerification]: this Docker version is not on the list of validated versions: 20.10.2. Latest validated version: 19.03
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

**Note:** I appended `--ignore-preflight-errors="SystemVerification"` to the `kubeadm join` command, becaue I am using a newer version of Docker

If you check the list of nodes now, you should be able to see node2 too.

```
[root@kubeadm-node1 ~]# kubectl  get nodes 
NAME            STATUS   ROLES                  AGE   VERSION
kubeadm-node1   Ready    control-plane,master   44m   v1.20.2
kubeadm-node2   Ready    <none>                 77s   v1.20.2
[root@kubeadm-node1 ~]#

[root@kubeadm-node1 ~]# kubectl  get nodes  -o wide
NAME            STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE                     KERNEL-VERSION           CONTAINER-RUNTIME
kubeadm-node1   Ready    control-plane,master   44m   v1.20.2   10.240.0.31   <none>        Fedora 33 (Server Edition)   5.10.7-200.fc33.x86_64   docker://20.10.2
kubeadm-node2   Ready    <none>                 98s   v1.20.2   10.240.0.32   <none>        Fedora 33 (Server Edition)   5.10.7-200.fc33.x86_64   docker://20.10.2
[root@kubeadm-node1 ~]# 
```

Repeat above procedure to join more worker nodes.


If you chose to not *"untaint"* the master node, and your nginx pod was in Pending state, then now the scheduler should automatically assign nginx pod to this new node2.

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
NAMESPACE         NAME                                          READY   STATUS    RESTARTS   AGE
calico-system     pod/calico-kube-controllers-b87bd7f6f-k2hdv   1/1     Running   0          28m
calico-system     pod/calico-node-d6zg2                         1/1     Running   0          4m41s
calico-system     pod/calico-node-lxrkq                         1/1     Running   0          28m
calico-system     pod/calico-typha-588c975bc5-2b8xg             1/1     Running   0          2m55s
calico-system     pod/calico-typha-588c975bc5-wkfcw             1/1     Running   0          28m
default           pod/nginx                                     1/1     Running   0          9m46s
kube-system       pod/coredns-74ff55c5b-9nq69                   1/1     Running   0          47m
kube-system       pod/coredns-74ff55c5b-xdzxr                   1/1     Running   0          47m
kube-system       pod/etcd-kubeadm-node1                        1/1     Running   0          47m
kube-system       pod/kube-apiserver-kubeadm-node1              1/1     Running   0          47m
kube-system       pod/kube-controller-manager-kubeadm-node1     1/1     Running   0          47m
kube-system       pod/kube-proxy-4lxs4                          1/1     Running   0          4m41s
kube-system       pod/kube-proxy-nn29f                          1/1     Running   0          47m
kube-system       pod/kube-scheduler-kubeadm-node1              1/1     Running   0          47m
tigera-operator   pod/tigera-operator-657cc89589-5rl4z          1/1     Running   0          29m

NAMESPACE       NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
calico-system   service/calico-typha   ClusterIP   10.32.42.78   <none>        5473/TCP                 28m
default         service/kubernetes     ClusterIP   10.32.0.1     <none>        443/TCP                  47m
kube-system     service/kube-dns       ClusterIP   10.32.0.10    <none>        53/UDP,53/TCP,9153/TCP   47m

NAMESPACE       NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
calico-system   daemonset.apps/calico-node   2         2         2       2            2           kubernetes.io/os=linux   28m
kube-system     daemonset.apps/kube-proxy    2         2         2       2            2           kubernetes.io/os=linux   47m

NAMESPACE         NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
calico-system     deployment.apps/calico-kube-controllers   1/1     1            1           28m
calico-system     deployment.apps/calico-typha              2/2     2            2           28m
kube-system       deployment.apps/coredns                   2/2     2            2           47m
tigera-operator   deployment.apps/tigera-operator           1/1     1            1           29m

NAMESPACE         NAME                                                DESIRED   CURRENT   READY   AGE
calico-system     replicaset.apps/calico-kube-controllers-b87bd7f6f   1         1         1       28m
calico-system     replicaset.apps/calico-typha-588c975bc5             2         2         2       28m
kube-system       replicaset.apps/coredns-74ff55c5b                   2         2         2       47m
tigera-operator   replicaset.apps/tigera-operator-657cc89589          1         1         1       29m
[root@kubeadm-node1 ~]# 
```

------

# Accessing your cluster from machines other than the master node:

So far, we are able to talk to our Kubernetes cluster from node1, as user student. It is because that is where we have configured our `.kube/config` which is used by `kubectl` commands. To be able to access the cluster from some other computer, such as your work computer, etc, you need to copy the administrator kubeconfig file from your master node to your computer like this:

```
mkdir ~/.kube
scp student@<master-node-ip>:/home/student/.kube/config  ~/.kube/kubeadm-cluster.conf
kubectl --kubeconfig ~/.kube/kubeadm-cluster.conf get nodes
```

It might be a hassle to include `--kubeconfig ~/.kube/kubeadm-cluster.conf` in every kubectl command. You can setup a shell alias, or you can simply save it as `~/.kube/config` on your work computer. **WARNING** However, it is possible that you already have `~/.kube/config` file, which might contain configuration to one or more existing kubernetes clusters, and you don't want to accidently lose access to those clusters by over-writing it with this `.kube/config` . To be able to retain access to those clusters, and still also be able to use this kubeadm cluster, all from single kubectl, without specifying `--kubeconfig` everytime, you can use the KUBECONFIG environment variable. 

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

Notice that my default .kube/config lists two clusters, one of them is production cluster for a client. I don't want to lose this configuration. 

```
[kamran@kworkhorse ~]$ kubectl config get-contexts
CURRENT   NAME                    CLUSTER                 AUTHINFO   NAMESPACE
*         minikube                minikube                minikube   
          client-gce.k8s.local   client-gce.k8s.local   admin      
[kamran@kworkhorse ~]$ 
```

So I have verified that without using `--kubeconfig` with `kubectl`, I can access my previous clusters. By using a specific configuration with `--kubeconfig` , I can access my kubeadm cluster. Is it possible to merge these two configurations? No. Not directly, which is actually a safety feature. Check this article for more detail: [https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable](https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#set-the-kubeconfig-environment-variable) .

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


**Note:**
The `admin.conf` file (`/etc/kubernetes/admin.conf` on master node, copied as `/home/student/.kube/config`) gives the user superuser privileges over the cluster. This file should be used very carefully. For normal users, it’s recommended to generate an unique credential, to which you whitelist privileges. You can do this with the `kubeadm alpha phase kubeconfig user --client-name <client-name>` command. This command will print out a KubeConfig file to STDOUT which you should save to a file and distribute to your user. After that, whitelist privileges by using `kubectl create (cluster)rolebinding`.
