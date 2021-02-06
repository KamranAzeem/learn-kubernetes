# Install specific version of Kubernets in your cluster:

**Note:** This document acts as a supplement to the main installation guides in this repository, and applies to RedHat and derivatives.

This may come in handy, when you are preparing for an exam, such as CKA, or you want to upgrade a Kubernetes cluster from one version (v1.19) to a later version (v1.20). In that case you want to setup a test cluster with an older version and perform the test upgrade.



## Install specific version of `kubelet`, `kubeadm` and `kubectl`:
So, if you want to install a specific version of Kubernetes in your cluster - say `1.19.7` -  then you need a matching version of **kubelet** - and bit less importantly - matching version of **`kubeadm`** and **`kubeadm`**. *"How to do it?"* is shown below. Then, *"how to install a specific version of Kubernetes?"* is demonstrated later.
 


```
[root@kubeadm-test ~]# yum list kubelet kubeadm kubectl --disableexcludes=kubernetes --showduplicates
. . . 

kubelet.x86_64                                                   1.19.3-0                                                     kubernetes
kubelet.x86_64                                                   1.19.4-0                                                     kubernetes
kubelet.x86_64                                                   1.19.5-0                                                     kubernetes
kubelet.x86_64                                                   1.19.6-0                                                     kubernetes
kubelet.x86_64                                                   1.19.7-0                                                     kubernetes
kubelet.x86_64                                                   1.20.0-0                                                     kubernetes
kubelet.x86_64                                                   1.20.1-0                                                     kubernetes
kubelet.x86_64                                                   1.20.2-0                                                     kubernetes
[root@kubeadm-test ~]# 
```


```
[root@kubeadm-test ~]# yum list kubelet-1.19.7-0 kubeadm-1.19.7-0 kubectl-1.19.7-0 --disableexcludes=kubernetes

Last metadata expiration check: 0:22:41 ago on Sat 06 Feb 2021 08:28:03 PM CET.
Available Packages
kubeadm.x86_64                                                    1.19.7-0                                                    kubernetes
kubectl.x86_64                                                    1.19.7-0                                                    kubernetes
kubelet.x86_64                                                    1.19.7-0                                                    kubernetes
[root@kubeadm-test ~]# 
```


Install version `1.19.7-0` of these three components by appending the yum version to the package name, i.e. `<packagename>-<version>`:


```
yum -y install --disableexcludes=kubernetes \
  kubelet-1.19.7-0 \
  kubeadm-1.19.7-0 \
  kubectl-1.19.7-0
```  


```
[root@kubeadm-test ~]# yum -y install --disableexcludes=kubernetes \
>   kubelet-1.19.7-0 \
>   kubeadm-1.19.7-0 \
>   kubectl-1.19.7-0

Last metadata expiration check: 0:23:49 ago on Sat 06 Feb 2021 08:28:03 PM CET.
Dependencies resolved.
========================================================================================================================================
 Package                                      Architecture            Version                         Repository                   Size
========================================================================================================================================
Installing:
 kubeadm                                      x86_64                  1.19.7-0                        kubernetes                  8.3 M
 kubectl                                      x86_64                  1.19.7-0                        kubernetes                  9.0 M
 kubelet                                      x86_64                  1.19.7-0                        kubernetes                   20 M
Installing dependencies:
 conntrack-tools                              x86_64                  1.4.5-6.fc33                    fedora                      207 k
 containernetworking-plugins                  x86_64                  0.9.0-1.fc33                    updates                     9.3 M
 cri-tools                                    x86_64                  1.13.0-0                        kubernetes                  5.1 M
 libnetfilter_cthelper                        x86_64                  1.0.0-18.fc33                   fedora                       22 k
 libnetfilter_cttimeout                       x86_64                  1.0.0-16.fc33                   fedora                       22 k
 libnetfilter_queue                           x86_64                  1.0.2-16.fc33                   fedora                       27 k
 socat                                        x86_64                  1.7.4.1-1.fc33                  updates                     307 k

Transaction Summary
========================================================================================================================================
Install  10 Packages

Total download size: 52 M
Installed size: 266 M
Downloading Packages:
(1/10): socat-1.7.4.1-1.fc33.x86_64.rpm                                                                 742 kB/s | 307 kB     00:00    
(2/10): conntrack-tools-1.4.5-6.fc33.x86_64.rpm                                                         397 kB/s | 207 kB     00:00    
(3/10): libnetfilter_cttimeout-1.0.0-16.fc33.x86_64.rpm                                                 218 kB/s |  22 kB     00:00    
(4/10): libnetfilter_cthelper-1.0.0-18.fc33.x86_64.rpm                                                  105 kB/s |  22 kB     00:00    
(5/10): libnetfilter_queue-1.0.2-16.fc33.x86_64.rpm                                                     303 kB/s |  27 kB     00:00    
(6/10): 14bfe6e75a9efc8eca3f638eb22c7e2ce759c67f95b43b16fae4ebabde1549f3-cri-tools-1.13.0-0.x86_64.rpm  1.7 MB/s | 5.1 MB     00:02    
(7/10): containernetworking-plugins-0.9.0-1.fc33.x86_64.rpm                                             2.5 MB/s | 9.3 MB     00:03    
(8/10): 5d5cf3bfda5f20ae144562586823529247b8b7292c5003182318451418904d3e-kubeadm-1.19.7-0.x86_64.rpm    1.7 MB/s | 8.3 MB     00:05    
(9/10): 67eef704a2de59c72b6cc2cc278c831a195e818a58cfd39dc44a2164efd8fc81-kubectl-1.19.7-0.x86_64.rpm    2.1 MB/s | 9.0 MB     00:04    
(10/10): 55d54ddf43404e0eb64741059220d13d4f139319f9a2065b8660af83fd48ec8e-kubelet-1.19.7-0.x86_64.rpm   2.9 MB/s |  20 MB     00:06    
----------------------------------------------------------------------------------------------------------------------------------------
Total                                                                                                   4.4 MB/s |  52 MB     00:11     
Running transaction check
Transaction check succeeded.
Running transaction test
Transaction test succeeded.
Running transaction
  Preparing        :                                                                                                                1/1 
  Installing       : containernetworking-plugins-0.9.0-1.fc33.x86_64                                                               1/10 
  Installing       : kubectl-1.19.7-0.x86_64                                                                                       2/10 
  Installing       : cri-tools-1.13.0-0.x86_64                                                                                     3/10 
  Installing       : libnetfilter_queue-1.0.2-16.fc33.x86_64                                                                       4/10 
  Installing       : libnetfilter_cttimeout-1.0.0-16.fc33.x86_64                                                                   5/10 
  Installing       : libnetfilter_cthelper-1.0.0-18.fc33.x86_64                                                                    6/10 
  Installing       : conntrack-tools-1.4.5-6.fc33.x86_64                                                                           7/10 
  Running scriptlet: conntrack-tools-1.4.5-6.fc33.x86_64                                                                           7/10 
  Installing       : socat-1.7.4.1-1.fc33.x86_64                                                                                   8/10 
  Installing       : kubelet-1.19.7-0.x86_64                                                                                       9/10 
  Installing       : kubeadm-1.19.7-0.x86_64                                                                                      10/10 
  Running scriptlet: kubeadm-1.19.7-0.x86_64                                                                                      10/10 
  Verifying        : containernetworking-plugins-0.9.0-1.fc33.x86_64                                                               1/10 
  Verifying        : socat-1.7.4.1-1.fc33.x86_64                                                                                   2/10 
  Verifying        : conntrack-tools-1.4.5-6.fc33.x86_64                                                                           3/10 
  Verifying        : libnetfilter_cthelper-1.0.0-18.fc33.x86_64                                                                    4/10 
  Verifying        : libnetfilter_cttimeout-1.0.0-16.fc33.x86_64                                                                   5/10 
  Verifying        : libnetfilter_queue-1.0.2-16.fc33.x86_64                                                                       6/10 
  Verifying        : cri-tools-1.13.0-0.x86_64                                                                                     7/10 
  Verifying        : kubeadm-1.19.7-0.x86_64                                                                                       8/10 
  Verifying        : kubectl-1.19.7-0.x86_64                                                                                       9/10 
  Verifying        : kubelet-1.19.7-0.x86_64                                                                                      10/10 

Installed:
  conntrack-tools-1.4.5-6.fc33.x86_64         containernetworking-plugins-0.9.0-1.fc33.x86_64  cri-tools-1.13.0-0.x86_64               
  kubeadm-1.19.7-0.x86_64                     kubectl-1.19.7-0.x86_64                          kubelet-1.19.7-0.x86_64                 
  libnetfilter_cthelper-1.0.0-18.fc33.x86_64  libnetfilter_cttimeout-1.0.0-16.fc33.x86_64      libnetfilter_queue-1.0.2-16.fc33.x86_64 
  socat-1.7.4.1-1.fc33.x86_64                

Complete!
[root@kubeadm-test ~]#
```


By this time, `kubeadm` is only *installed* - and is not *running*. 

**Note:** kubelet is now set to start. Kubelet will continuously try to start and will fail (crash-loop), because it will wait for kubeadm to tell it what to do. This *"crash-loop"* is expected and normal. After you initialize your master (using kubeadm), the kubelet will run normally.

## Install specific version of Kubernetes in the cluster:

Later, when you reach a point to install specific version of **`Kubernetes`**, you use the following command: 

```
kubeadm init  \
  --pod-network-cidr "10.200.0.0/16" \
  --service-cidr "10.32.0.0/16" \
  --kubernetes-version "v1.19.7" \
  --ignore-preflight-errors="SystemVerification"
```

**Notes:** 
* If you want a particular version of Kubernetes to be installed, you have to use additional switch: `--kubernetes-version string` with the `kubeadm init` command. (Shown next)
* The `--kubernetes-version` switch expects a semantic version, and is different in *appearance* from it's `yum` counterpart, even though it looks the same. 
* The exact (semantic) version of Kubernetes can be found from the URL: [https://github.com/kubernetes/kubernetes/tags](https://github.com/kubernetes/kubernetes/tags)

```
kubeadm init \
  --pod-network-cidr "10.200.0.0/16" \
  --service-cidr "10.32.0.0/16" \
  --kubernetes-version "v1.19.7" \
  --ignore-preflight-errors="SystemVerification"
```


**Note:** If you do not use `--kubernetes-version string`, then the **latest** version of Kubernetes will be installed:
