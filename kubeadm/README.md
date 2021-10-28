# KubeAdm based kubernetes cluster

Reference documentation: [https://kubernetes.io/docs/setup/independent/install-kubeadm/](https://kubernetes.io/docs/setup/independent/install-kubeadm/)

Kubeadm helps you setup/bootstrap a minimum viable/usable Kubernetes cluster that just works. Kubeadm also supports cluster expansion, upgrades, downgrade, and managing bootstrap tokens, which are extra features, if you are comparing it with minikube.

A video showing kubeadm cluster setup, (in Urdu language), using this guide, is available at: [https://youtu.be/FRvkSlIUimM](https://youtu.be/FRvkSlIUimM)

Below are different flavors of this guide:
* Kubeadm on RedHat/CentOS/Fedora (33, 34) - using Flannel for pod network: [kubeadm-fedora-flannel.md](kubeadm-fedora-flannel.md)
* Kubeadm on RedHat/CentOS/Fedora (33) - using Calico for pod network: [kubeadm-fedora-calico.md](kubeadm-fedora-calico.md)
* Kubeadm on Debian/Ubuntu: [kubeadm-ubuntu-flannel.md](kubeadm-ubuntu-flannel.md)

Supplementary documents:
* [kubeadm-install-specific-version.md](kubeadm-install-specific-version.md)
